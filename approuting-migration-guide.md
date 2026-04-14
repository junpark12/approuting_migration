# Ingress → Gateway API 마이그레이션 가이드

## 1. App Routing Istio + Gateway API 활성화

### 사전 준비: Preview Feature Flag 등록

```bash
# aks-preview 확장 설치/업데이트 (19.0.0b26 이상 필요)
az extension add --name aks-preview --upgrade
az extension show --name aks-preview --query version

# Feature Flag 2개 모두 등록해야 함
az feature register \
  --namespace Microsoft.ContainerService \
  --name ManagedGatewayAPIPreview

az feature register \
  --namespace Microsoft.ContainerService \
  --name AppRoutingIstioGatewayAPIPreview

# 두 flag 모두 Registered 상태 확인 (수 분 소요)
az feature show \
  --namespace Microsoft.ContainerService \
  --name ManagedGatewayAPIPreview \
  --output table

az feature show \
  --namespace Microsoft.ContainerService \
  --name AppRoutingIstioGatewayAPIPreview \
  --output table

az provider register \
  --namespace Microsoft.ContainerService
```

> ⚠️ `ManagedGatewayAPIPreview`가 누락되면 `--enable-gateway-api` 실행 시
> `PreviewFeatureNotRegistered` 에러가 발생한다. 반드시 두 개 모두 등록해야 함.

### Gateway API CRD 설치 여부 확인

```bash
kubectl get crds | grep gateway.networking.k8s.io
```

- **출력 없음** → CRD 미설치. 아래 `az aks update` 명령으로 CRD + Istio 한 번에 설치
- **출력 있음** → CRD 이미 설치됨. `az aks approuting gateway istio enable`만 실행

### Case 1. Gateway API CRD 미설치 (대부분의 경우)

CRD 설치와 Istio 활성화를 동시에 수행한다.

```bash
RG="AKSKorea"
CLUSTER="akskor"

az aks update \
  --resource-group "$RG" \
  --name "$CLUSTER" \
  --enable-gateway-api \
  --enable-app-routing-istio
```

### Case 2. Gateway API CRD가 이미 설치된 경우

`--enable-gateway-api`로 CRD를 설치할 경우 이미 존재한다는 에러가 발생할 수 있다.
이 경우 Istio만 별도로 활성화한다.

```bash
az aks approuting gateway istio enable \
  --resource-group "$RG" \
  --name "$CLUSTER"
```

### 설치 확인

```bash
# GatewayClass 확인 (approuting-istio가 보여야 함)
kubectl get gatewayclass

# Istio 관련 pod 확인
kubectl get pods --all-namespaces | grep istio

# Gateway API CRD 확인
kubectl get crds | grep gateway.networking.k8s.io
```

---

## 2. Gateway + HTTPRoute Internal LB 배포

### 배포 매니페스트 (tobe_env/gateway.yaml)

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: pixel-istio-gw
  namespace: demo
  annotations:
    service.beta.kubernetes.io/azure-load-balancer-internal: "true"
spec:
  gatewayClassName: approuting-istio
  listeners:
  - hostname: echo.internal.demo
    name: http
    protocol: HTTP
    port: 80
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: echo-route
  namespace: demo
spec:
  parentRefs:
  - name: pixel-istio-gw
  rules:
  - backendRefs:
    - name: echo
      port: 80
```

> ⚠️ `service.beta.kubernetes.io/azure-load-balancer-internal: "true"` 를 반드시 Gateway에 설정해야
> Internal LB로 생성된다. 누락 시 Public IP가 할당됨.

### 배포 및 확인

```bash
kubectl apply -f gateway.yaml

# Gateway 상태 확인 (PROGRAMMED=True, Internal IP 할당 확인)
kubectl get gateway -n demo

# HTTPRoute 확인
kubectl get httproute -n demo
```

**실행 결과 (2026-03-19):**
```
NAME             CLASS              ADDRESS      PROGRAMMED   AGE
pixel-istio-gw   approuting-istio   10.224.0.8   True         11s
```
> ✅ Internal LB IP `10.224.0.8` 할당, PROGRAMMED=True 확인

### Istio 자동 생성 리소스 확인

Gateway를 배포하면 Istio가 자동으로 Deployment, Service, HPA, PDB를 생성한다.
NGINX 방식과 달리 `app-routing-system`이 아닌 **Gateway가 있는 namespace(demo)에 생성**된다.

```bash
# 한 번에 전부 확인
kubectl get deployment,svc,hpa,pdb -n demo \
  -l gateway.networking.k8s.io/gateway-name=pixel-istio-gw
```

**실행 결과 (2026-03-19):**
```
NAME                                              READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/pixel-istio-gw-approuting-istio   2/2     2            2           3m33s

NAME                              REFERENCE                                    TARGETS       MINPODS   MAXPODS   REPLICAS   AGE
hpa/pixel-istio-gw-approuting-istio   Deployment/pixel-istio-gw-approuting-istio   cpu: 2%/80%   2         5         2          4m45s
```

> 이름 패턴: `{gateway이름}-{gatewayClassName}`
> Gateway를 삭제하면 이 리소스들도 같이 정리된다.

### NGINX vs Gateway API 리소스 관리 비교

| 항목 | NGINX (App Routing) | Istio Gateway API |
|---|---|---|
| 컨트롤러 위치 | `app-routing-system` (중앙 집중) | 각 namespace에 Gateway별 생성 |
| LB 설정 | `NginxIngressController` CRD (cluster-scoped) | `Gateway` 리소스 annotation (namespace-scoped) |
| IP 관리 | 컨트롤러 1개 = LB IP 1개, 모든 Ingress가 공유 | Gateway마다 별도 LB IP 가능 |
| 자동 관리 | Pod만 | Deployment + Service + HPA + PDB 자동 생성 |

### 접근 테스트

```bash
# 클러스터 내부에서 curl 테스트
kubectl run curl-test --rm -it --restart=Never \
  --image=curlimages/curl -- \
  curl -H "Host: echo.internal.demo" http://10.224.0.8/
```

---

## 3. 기존 NGINX Ingress 정리

Gateway API를 통한 접근이 확인되면, 기존 NGINX 기반 Ingress 리소스와 컨트롤러를 정리한다.

### 3-1. Ingress 리소스 삭제

```bash
# 현재 Ingress 리소스 확인
kubectl get ingress --all-namespaces

# Ingress 삭제
kubectl delete ingress echo-ingress -n demo
```

### 3-2. Internal NGINX 컨트롤러 삭제

```bash
# NginxIngressController 확인
kubectl get NginxIngressController

# Internal 컨트롤러 삭제
kubectl delete NginxIngressController nginx-internal
```

> 삭제하면 `app-routing-system`의 `nginx-internal-0` pod/service가 함께 제거되고,
> Internal LB IP `10.224.0.7`이 해제된다.

### 3-3. (선택) Default Public NGINX 컨트롤러 삭제

Internal만 사용 중이었다면 기존 Public 컨트롤러도 정리한다.

```bash
kubectl delete NginxIngressController default
```

### 3-4. 정리 확인

```bash
# IngressClass 확인 (NGINX 관련 class가 제거되었는지)
kubectl get ingressclass

# app-routing-system에 NGINX pod/service가 남아있지 않은지
kubectl get pods -n app-routing-system
kubectl get svc -n app-routing-system
```

---

## 4. 기존 NGINX Internal IP 이관 (선택)

NGINX Internal 컨트롤러 삭제 후 해제된 IP를 Gateway에서 그대로 사용하려면,
Gateway에 고정 IP annotation을 추가한다.

```yaml
annotations:
  service.beta.kubernetes.io/azure-load-balancer-internal: "true"
  service.beta.kubernetes.io/azure-load-balancer-ipv4: "10.224.0.7"
```

> 동일 IP를 두 서비스가 동시에 사용할 수 없으므로, 반드시 Step 3에서 NGINX 삭제 후 진행해야 한다.
> 전환 시 순간 단절이 발생할 수 있음에 유의.

### 이관 확인 (2026-03-19)

```bash
kubectl run curl-test --rm -it --restart=Never \
  --image=curlimages/curl -- \
  curl -H "Host: echo.internal.demo" http://10.224.0.7/
```

```json
{
  "host": {"hostname": "echo.internal.demo"},
  "request": {
    "headers": {
      "host": "echo.internal.demo",
      "x-forwarded-for": "192.168.0.127",
      "x-forwarded-proto": "http",
      "x-envoy-external-address": "192.168.0.127",
      "x-envoy-peer-metadata-id": "router~192.168.2.241~pixel-istio-gw-approuting-istio-55445cbf66-rpvjm.demo~demo.svc.cluster.local",
      "x-envoy-attempt-count": "1"
    }
  }
}
```

> ✅ 기존 NGINX Internal IP(`10.224.0.7`)로 요청 → Istio Gateway(Envoy)가 처리 확인
> - `x-envoy-*` 헤더가 응답에 포함됨 (NGINX `x-real-ip` 대신 Envoy 헤더)
> - `x-envoy-peer-metadata-id`에서 `pixel-istio-gw-approuting-istio` pod이 처리한 것 확인
> - 동일 IP로 NGINX → Istio Gateway 이관 완료

---

## 5. 비활성화 (원복)

테스트 후 Istio만 비활성화하면 된다. Gateway API CRDs는 GA 기능이므로 남겨두는 것을 권장한다.

```bash
# 1. Gateway/HTTPRoute 리소스 삭제
kubectl delete gateway,httproute --all --all-namespaces

# 2. Istio ingress 비활성화
az aks approuting gateway istio disable \
  --resource-group "$RG" \
  --name "$CLUSTER"
```

> `--enable-gateway-api`는 GA 기능이라 CRD가 남아있어도 리소스 소비나 기존 워크로드 영향 없음.
> 나중에 Istio GA 시점에 CRD가 이미 설치되어 있으므로 `az aks approuting gateway istio enable`만 실행하면 된다.

---

## 6. 마이그레이션 시 권장 절차

1. `ingress2gateway`로 기본 구조 변환
2. 원본 Ingress의 annotation 목록 추출
3. 각 annotation의 Gateway API 대응 방안 정리 (아래 참고)
4. 변환된 Gateway/HTTPRoute에 filter, policy 수동 추가
5. 스테이징 환경에서 기능 검증 (특히 rewrite, timeout, rate-limit)
6. `configuration-snippet` 사용 부분은 기능 단위로 분해하여 재설계

---
---

# 참고 자료

## 참고 A. ingress2gateway를 활용한 변환 테스트

### 설치

```bash
# macOS
brew install ingress2gateway

# 또는 Go로 직접 설치
go install github.com/kubernetes-sigs/ingress2gateway@latest
```

### 실행 방법

```bash
# 로컬 yaml 파일 기준 변환
ingress2gateway print --providers=ingress-nginx \
  --input-file=internal-ingress.yaml

# 클러스터에서 직접 읽어서 변환 (특정 네임스페이스)
ingress2gateway print --providers=ingress-nginx --namespace=demo

# 전체 네임스페이스
ingress2gateway print --providers=ingress-nginx --all-namespaces

# 결과를 파일로 저장
ingress2gateway print --providers=ingress-nginx \
  --input-file=internal-ingress.yaml > converted-gateway.yaml
```

> ⚠️ **주의:** `ingress-nginx` provider는 `ingressClassName: nginx`만 인식한다.
> `nginx-internal` 등 커스텀 IngressClass를 사용하는 경우, 파일을 복사해서
> `ingressClassName: nginx`로 변경 후 테스트해야 한다.

### 변환 입력 (internal-ingress.yaml)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: echo-ingress
  namespace: demo
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
    nginx.ingress.kubernetes.io/limit-rps: "50"
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "30"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "120"
    nginx.ingress.kubernetes.io/custom-http-errors: "404,503"
    nginx.ingress.kubernetes.io/configuration-snippet: |
      more_set_headers "X-Custom-Header: my-value";
spec:
  ingressClassName: nginx
  rules:
  - host: echo.internal.demo
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: echo
            port:
              number: 80
```

### 변환 출력 (converted-gateway.yaml)

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  annotations:
    gateway.networking.k8s.io/generator: ingress2gateway-0.5.0
  name: nginx
  namespace: demo
spec:
  gatewayClassName: nginx
  listeners:
  - hostname: echo.internal.demo
    name: echo-internal-demo-http
    port: 80
    protocol: HTTP
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  annotations:
    gateway.networking.k8s.io/generator: ingress2gateway-0.5.0
  name: echo-ingress-echo-internal-demo
  namespace: demo
spec:
  hostnames:
  - echo.internal.demo
  parentRefs:
  - name: nginx
  rules:
  - backendRefs:
    - name: echo
      port: 80
    matches:
    - path:
        type: PathPrefix
        value: /
```

> ⚠️ **기본 라우팅 구조(host, path, backend)만 변환됨. NGINX annotation은 전부 무시되며 경고도 없음.**

---

## 참고 B. Annotation → Gateway API 변환 결과 (ingress2gateway 기준)

### 변환 결과 요약

| 구분 | Annotation | ingress2gateway 결과 | Gateway API 대안 |
|---|---|---|---|
| **✅ 변환됨** | `rewrite-target` | URLRewrite filter로 변환 | — |
| **⚠️ best-effort** | `proxy-read-timeout` + `proxy-send-timeout` | `timeouts.request`로 합산 변환 | HTTPRoute `timeouts` (v1.1+) |
| **⚠️ best-effort** | `proxy-body-size` | 경고 후 무시 ("구현체 기본값 사용") | 구현체별 Policy |
| **❌ 미지원** | `ssl-redirect` | Unsupported | HTTPRoute `RequestRedirect` filter |
| **❌ 미지원** | `force-ssl-redirect` | Unsupported | HTTPRoute `RequestRedirect` filter |
| **❌ 미지원** | `ssl-protocols` | Unsupported | Gateway listener TLS 설정 |
| **❌ 미지원** | `ssl-ciphers` | Unsupported | Gateway listener TLS 설정 |
| **❌ 미지원** | `ssl-prefer-server-ciphers` | Unsupported | Gateway listener TLS 설정 |
| **❌ 미지원** | `hsts` | Unsupported | HTTPRoute `ResponseHeaderModifier` filter |
| **❌ 미지원** | `hsts-max-age` | Unsupported | HTTPRoute `ResponseHeaderModifier` filter |
| **❌ 미지원** | `hsts-include-subdomains` | Unsupported | HTTPRoute `ResponseHeaderModifier` filter |
| **❌ 미지원** | `backend-protocol` | Unsupported | HTTPRoute backendRef (HTTP/HTTPS) |
| **❌ 미지원** | `use-http2` | Unsupported | Gateway API는 HTTP/2 기본 지원 |
| **❌ 미지원** | `proxy-connect-timeout` | Unsupported | HTTPRoute `timeouts` |
| **❌ 미지원** | `proxy-send-timeout` | Unsupported | HTTPRoute `timeouts` |
| **❌ 미지원** | `send-timeout` | Unsupported | HTTPRoute `timeouts` |
| **❌ 미지원** | `cors-allow-origin` | Unsupported | 수동 HTTPRoute filter 또는 앱 레벨 |
| **❌ 미지원** | `cors-allow-methods` | Unsupported | 수동 HTTPRoute filter 또는 앱 레벨 |
| **❌ 미지원** | `cors-allow-headers` | Unsupported | 수동 HTTPRoute filter 또는 앱 레벨 |
| **❌ 미지원** | `affinity: cookie` | "Session affinity is not supported" | HTTPRoute `sessionPersistence` (v1.1+) |
| **❌ 미지원** | `session-cookie-name` | Unsupported | HTTPRoute `sessionPersistence.sessionName` |
| **❌ 미지원** | `session-cookie-secure` | Unsupported | 대안 없음 (앱 레벨 처리) |
| **❌ 미지원** | `session-cookie-path` | Unsupported | 대안 없음 (앱 레벨 처리) |
| **❌ 미지원** | `session-cookie-expires` | Unsupported | 대안 없음 (앱 레벨 처리) |
| **❌ 미지원** | `session-cookie-max-age` | Unsupported | HTTPRoute `sessionPersistence` (v1.1+) |
| **❌ 미지원** | `custom-http-errors` | Unsupported | **대안 없음** (App GW도 backend 오류 통과) |
| **❌ 미지원** | `default-backend` | Unsupported | catch-all HTTPRoute (부분 대체만 가능) |
| **❌ 미지원** | `client-max-body-size` | Unsupported | 구현체별 Policy |
| **❌ 미지원** | `proxy-buffer-size` | Unsupported | 구현체별 Policy |
| **❌ 미지원** | `proxy-buffering` | Unsupported | 구현체별 Policy |
| **❌ 미지원** | `proxy-request-buffering` | Unsupported | 구현체별 Policy |
| **❌ 미지원** | `proxy-charset` | Unsupported | 대안 없음 (앱 레벨 처리) |
| **❌ 미지원** | `client-header-buffer-size` | Unsupported | 대안 없음 |
| **❌ 미지원** | `large-client-header-buffers` | Unsupported | 대안 없음 |
| **❌ 미지원** | `keepalive` | Unsupported | keepalive 기본 활성화. 연결 수/timeout 세부 튜닝은 EnvoyFilter 필요 (App Routing Istio 불가) |
| **❌ 미지원** | `proxy-http-version` | Unsupported | 대안 없음 |

---

### 대안 상세

#### 1. SSL Redirect → RequestRedirect filter

```yaml
# HTTPRoute에 filter 추가
rules:
- filters:
  - type: RequestRedirect
    requestRedirect:
      scheme: https
      statusCode: 301
```

#### 2. HSTS → ResponseHeaderModifier filter

```yaml
rules:
- filters:
  - type: ResponseHeaderModifier
    responseHeaderModifier:
      add:
      - name: Strict-Transport-Security
        value: "max-age=31536000; includeSubDomains"
```

#### 3. TLS 설정 (ssl-protocols, ssl-ciphers) → Gateway listener

```yaml
# Gateway listener에서 직접 설정
listeners:
- name: https
  port: 443
  protocol: HTTPS
  tls:
    mode: Terminate
    options:
      gateway.istio.io/tls-min-protocol-version: TLSV1_2
```

> ⚠️ ssl-ciphers는 Gateway API 표준 스펙 외 영역. App Routing Istio에서 지원 여부 별도 확인 필요.

#### 4. Session Affinity → sessionPersistence (Gateway API v1.1+)

```yaml
# HTTPRoute backendRef에 추가
rules:
- backendRefs:
  - name: echo
    port: 80
  sessionPersistence:
    sessionName: INGRESSCOOKIE
    type: Cookie
    absoluteTimeout: 3600s
```

> ⚠️ **sessionPersistence는 Gateway API v1.1 실험적(Experimental) 기능이다.**
> App Routing Istio가 이 스펙을 실제로 구현했는지 별도 확인이 필요하다.
> 동작하지 않을 경우 아래 대안을 검토해야 한다.

**동작하지 않을 경우 대안:**

| 대안 | 가능 여부 | 비고 |
|---|---|---|
| Istio `DestinationRule` (consistentHash) | ❌ | App Routing Istio는 Istio CRDs 미설치 — 사용 불가 |
| Full Istio Mesh + `DestinationRule` | ✅ | App Routing Istio와 동시 사용 불가 — 마이그레이션 방향 변경 필요 |
| **앱 레벨 sticky session** | ✅ | 가장 확실한 대안. 앱에서 직접 Set-Cookie 처리 |

> session-cookie-secure, session-cookie-path는 Gateway API 스펙에 없음 → 앱 레벨에서 Set-Cookie 헤더로 처리.

#### 5. Timeouts → HTTPRoute timeouts (v1.1+)

```yaml
rules:
- backendRefs:
  - name: echo
    port: 80
  timeouts:
    request: 120s        # proxy-read-timeout 대응
    backendRequest: 30s  # proxy-connect-timeout 대응
```

#### 6. CORS → ResponseHeaderModifier filter

Gateway API에 CORS 전용 filter가 없으므로 헤더를 직접 추가:

```yaml
rules:
- filters:
  - type: ResponseHeaderModifier
    responseHeaderModifier:
      add:
      - name: Access-Control-Allow-Origin
        value: "https://app.contoso.com"
      - name: Access-Control-Allow-Methods
        value: "GET, POST, PUT, DELETE, OPTIONS"
      - name: Access-Control-Allow-Headers
        value: "Authorization, Content-Type, X-Custom-Header"
```

> ⚠️ **ResponseHeaderModifier는 응답에 헤더를 추가할 뿐, OPTIONS preflight 요청에 200 OK를 대신 응답하지 않는다.**
> 브라우저가 보내는 OPTIONS preflight는 별도 매칭 rule로 처리해야 하는데,
> Gateway API 표준에는 고정 응답(Direct Response) filter가 없어 Istio 전용 확장이 필요하다.
> 이 경우 복잡도가 급격히 상승하므로 **앱 레벨에서 CORS를 처리하는 것을 강력히 권장**한다.

#### 7. custom-http-errors / default-backend → 대안 없음

| 접근 | 설명 | 한계 |
|---|---|---|
| catch-all HTTPRoute | `/` PathPrefix로 error-page-svc 연결 | backend가 반환한 404/500은 처리 불가 |
| App GW custom error page | App GW 레벨 설정 | App GW 자체 오류(502, 503)만 처리 가능. backend 오류는 통과 |
| **앱 레벨 처리** | backend가 직접 error page 반환 | ✅ 유일한 완전한 대안 |

#### 8. 버퍼/Body 크기 설정 → 구현체 기본값에 의존

`proxy-body-size`, `proxy-buffer-size`, `proxy-buffering`, `client-max-body-size` 등은
Gateway API 표준 스펙에 없으며, App Routing Istio(Envoy)의 기본값을 사용한다.

Envoy 기본값:
- request body: 제한 없음 (스트리밍)
- buffer: 32KB

대부분의 경우 기본값으로 충분하나, 대용량 파일 업로드 환경에서는 별도 확인 필요.

---

### 마이그레이션 전 체크리스트

| 항목 | 영향도 | 조치 |
|---|---|---|
| ssl-redirect / force-ssl-redirect | 높음 | HTTPRoute RequestRedirect filter 수동 추가 |
| HSTS | 중간 | ResponseHeaderModifier filter 수동 추가 |
| Session affinity | 높음 | sessionPersistence 수동 추가 (v1.1+) |
| Timeouts | 중간 | HTTPRoute timeouts 수동 추가 |
| CORS | 높음 | ResponseHeaderModifier filter 수동 추가 또는 앱 레벨 처리 |
| custom-http-errors | 높음 | **앱 레벨 처리 필요** (Gateway API 대안 없음) |
| 버퍼/Body 크기 | 낮음 | Envoy 기본값 확인 후 필요시 조치 |
| ssl-protocols / ssl-ciphers | 중간 | Gateway listener TLS 설정으로 대체 |
| keepalive / use-http2 | 낮음 | keepalive 기본 활성화. 세부 튜닝 불가 / HTTP2 기본 지원 |

---

## 참고 C. Gateway API Preview 제한사항 — DNS/TLS 자동 관리 미지원

### NGINX App Routing (기존) — 자동 연동 가능

```bash
# Key Vault 연동
az aks approuting update \
  --resource-group myRG --name myCluster \
  --enable-kv \
  --attach-kv /subscriptions/.../vaults/myKeyVault

# Azure DNS Zone 자동 등록
az aks approuting zone add \
  --resource-group myRG --name myCluster \
  --ids /subscriptions/.../dnszones/contoso.com
```

Ingress에 `tls` 섹션과 annotation을 추가하면 Key Vault에서 인증서를 자동으로 가져오고,
host 기반으로 Azure DNS에 A 레코드를 자동 등록한다.

```yaml
# NGINX Ingress — 이것만 쓰면 TLS + DNS 자동
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app
  annotations:
    kubernetes.azure.com/tls-cert-keyvault-uri: https://myKeyVault.vault.azure.net/certificates/my-cert
spec:
  ingressClassName: webapprouting.kubernetes.azure.com
  tls:
  - hosts:
    - app.contoso.com
    secretName: my-cert-secret
  rules:
  - host: app.contoso.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: my-app
            port:
              number: 80
```

### Gateway API (현재 Preview) — 수동 설정 필요

```bash
# 1. 인증서를 직접 Secret으로 생성
kubectl create secret tls my-cert-secret \
  --cert=cert.pem --key=key.pem -n demo

# 2. DNS도 수동 등록
az network dns record-set a add-record \
  --resource-group myRG --zone-name contoso.com \
  --record-set-name app --ipv4-address 10.224.0.7
```

```yaml
# 3. Gateway에 TLS 직접 설정
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
spec:
  listeners:
  - name: https
    protocol: HTTPS
    port: 443
    tls:
      mode: Terminate
      certificateRefs:
      - name: my-cert-secret    # ← 수동으로 만든 Secret 참조
    hostname: app.contoso.com
```

### 비교

| 기능 | NGINX App Routing | Gateway API (Preview) |
|---|---|---|
| Key Vault 인증서 연동 | ✅ `az aks approuting update --enable-kv` | ❌ 수동 Secret 생성 |
| Azure DNS 자동 등록 | ✅ `az aks approuting zone add` | ❌ 수동 DNS 레코드 등록 |
| 인증서 자동 갱신 | ✅ add-on이 관리 | ❌ 직접 관리 (cert-manager 등) |

> GA 시점에 위 기능들이 추가될 예정이지만, 현재 Preview에서는 전부 수동이다.

### 실제 영향도 — App Gateway 연동 환경에서는 제한적

위 자동 기능은 App Routing add-on 자체의 편의 기능이다.
실제 운영 환경에서는 다른 방식으로 TLS/DNS를 관리하는 경우가 많으며, 이 경우 영향이 없다.

| 기능 | App Routing 자동 | 일반적인 운영 방식 | Gateway API 영향 |
|---|---|---|---|
| TLS 인증서 | `--enable-kv`로 add-on 연동 | Workload Identity + CSI Driver로 Key Vault 직접 마운트 | **영향 없음** — 기존 방식 그대로 사용 |
| DNS | `az aks approuting zone add`로 자동 등록 | App Gateway에서 hostname 설정 + 백엔드에 Internal IP 지정 | **영향 없음** — App Gateway가 DNS/호스트 관리 |

> App Routing의 자동 TLS/DNS 기능을 사용하지 않고,
> Workload Identity + Key Vault CSI Driver 또는 App Gateway 기반으로 운영 중이라면
> Gateway API Preview 전환 시 추가 작업이 발생하지 않는다.

---

---

## 참고 D. SNI Passthrough (TLSRoute) 미지원

### SNI Passthrough란

TLS 트래픽을 Gateway에서 **복호화하지 않고**, SNI 헤더만 보고 백엔드로 그대로 전달하는 방식이다.

```
TLS Terminate (일반):
  Client ─TLS→ Gateway (복호화) ─HTTP→ backend

SNI Passthrough:
  Client ─TLS→ Gateway (암호화 유지, SNI만 확인) ─TLS→ backend (여기서 복호화)
```

### NGINX에서는 annotation 하나로 가능

```yaml
annotations:
  nginx.ingress.kubernetes.io/ssl-passthrough: "true"
```

### Gateway API Preview에서는 미지원

`TLSRoute` 리소스로 정의하는 것이 Gateway API 표준이지만, 현재 App Routing Istio Preview에서 미지원이다.

```yaml
# Gateway API 표준에는 있지만, 현재 App Routing Istio에서 사용 불가
apiVersion: gateway.networking.k8s.io/v1alpha2
kind: TLSRoute
spec:
  parentRefs:
  - name: my-gateway
    sectionName: tls-passthrough
  rules:
  - backendRefs:
    - name: my-backend
      port: 443
```

### App Gateway 연동 환경에서의 영향

App Gateway는 L7 로드밸런서로 항상 TLS를 종료한다. backend와의 구간은 아래 두 가지 옵션으로 구성된다.

---

**옵션 1. Re-encrypt (App GW → backend HTTPS)**

App GW가 TLS를 종료한 뒤, backend로 새 TLS 연결을 맺는 방식.
backend가 HTTPS를 호스팅하는 경우 사용하며, App GW backend 설정에 backend의 root CA를 등록해야 한다.

```
Client ─TLS→ App GW (TLS 종료, L7 라우팅) ─새 TLS→ backend (https://api.contoso.com)
              ↑ frontend 인증서 필요           ↑ backend root CA를 App GW에 등록
```

| 위치 | 인증서 | 용도 |
|---|---|---|
| App GW frontend | ✅ 필요 | Client → App GW 구간 TLS |
| App GW backend 설정 | ✅ backend의 root CA 등록 | App GW → backend 신뢰 |
| backend | ✅ 필요 | App GW → backend 구간 TLS |

---

**옵션 2. SSL Offload (App GW → backend HTTP)**

App GW가 TLS를 종료한 뒤, backend로는 HTTP로 전달하는 방식.
backend가 TLS를 처리하지 않아도 되므로 구성이 단순하다.

```
Client ─TLS→ App GW (TLS 종료, L7 라우팅) ─HTTP→ backend
              ↑ frontend 인증서만 필요        ↑ backend는 TLS 불필요
```

| 위치 | 인증서 | 용도 |
|---|---|---|
| App GW frontend | ✅ 필요 | Client → App GW 구간 TLS |
| App GW backend 설정 | ❌ CA 등록 불필요 | HTTP 연결이므로 신뢰 설정 없음 |
| backend | ❌ 불필요 | HTTP 수신 |

---

> **두 옵션 모두** App GW에서 TLS를 종료하므로, Gateway API 레이어(NGINX/Istio)에서는
> **HTTP만 받으면 됨 → 마이그레이션 영향 없음.**

반면, NGINX에서 `ssl-passthrough`를 사용하는 환경은
TLS를 복호화하지 않고 SNI만 보고 라우팅하는 구조인데, 이 패턴은 현재 Gateway API에서 미지원이다.

```
NGINX ssl-passthrough:
  Client ─TLS→ NGINX (복호화 안 함, SNI만 확인) ─TLS 그대로→ backend
```

### 하이브리드 환경 시나리오 — 외부(App GW) + 내부(ExpressRoute) 병행

실제 기업 환경에서는 외부/내부 트래픽 경로가 다르게 구성되는 경우가 있다.
예: 외부 고객은 App GW를 통해, 내부 고객(on-prem)은 ExpressRoute → Internal LB로 직접 접근.

```
외부 고객 (인터넷):
  Client ─TLS→ App GW (TLS 종료, L7 라우팅) ─HTTP→ NGINX Internal → backend
                                                     → Gateway API 전환 가능 ✅

내부 고객 (on-prem → ExpressRoute → VNet):
  Client ─TLS→ Internal LB → NGINX (ssl-passthrough) ─TLS 그대로→ backend
                                ↑ SNI만 보고 라우팅          ↑ backend이 직접 TLS 처리
                                → Gateway API 전환 불가 ❌ (TLSRoute 미지원)
```

이 경우 **같은 클러스터에서 경로에 따라 마이그레이션 가능 여부가 갈린다.**

| 트래픽 경로 | TLS 처리 | Gateway API 전환 |
|---|---|---|
| 외부 → App GW → NGINX Internal | App GW에서 종료 | ✅ 가능 |
| 내부 → ExpressRoute → NGINX Internal (passthrough) | backend에서 종료 | ❌ 불가 (현재) |

### 대응 방안

- 외부 트래픽 경로만 먼저 Gateway API로 전환하고, 내부 경로는 NGINX 유지 (병렬 운영)
- 또는, 내부 경로도 TLS 종료 방식으로 아키텍처 변경 검토 (Gateway에서 TLS Terminate)
- GA 시점에 TLSRoute 지원이 추가되면 내부 경로도 전환



---

## 참고 E. App Routing Istio vs Full Istio Service Mesh

### App Routing Istio (`--enable-app-routing-istio`)

Ingress-only 경량 Istio이다. 외부→클러스터 진입(north-south)만 담당하며,
pod 간 통신(east-west)에는 관여하지 않는다.

```
[외부 트래픽]
      │
      ▼
┌─────────────┐
│  Gateway     │  ← Istio가 여기만 담당 (ingress-only)
│  (L7 라우팅)  │
└──────┬──────┘
       ▼
  frontend ──→ backend ──→ DB
       (이 구간은 Istio 관여 없음: mTLS 없음, 트래픽 제어 없음)
```

- ✅ Gateway + HTTPRoute 기반 L7 라우팅
- ❌ pod 간 mTLS, 카나리 배포, circuit breaker, 분산 추적 등 불가

### Full Istio Service Mesh (`--enable-azure-service-mesh`)

모든 pod에 sidecar(envoy proxy)가 붙어 pod 간 통신 전체를 제어한다.

```
[외부 트래픽]
      │
      ▼
┌─────────────┐
│  Gateway     │
└──────┬──────┘
       ▼
 [front+envoy] ─mTLS→ [back+envoy] ─mTLS→ [DB+envoy]
   ↑ 모든 pod 간 통신을 Istio가 전부 제어
```

- ✅ mTLS 자동 암호화, 트래픽 split, retry policy, 분산 추적 등
- ✅ Gateway API도 자체 지원 (별도 App Routing Istio 불필요)

### 상세 비교

| Feature | App Routing Gateway API | Istio Service Mesh Add-on |
|---|---|---|
| GatewayClass | `approuting-istio` | `istio` |
| 역할 | **Ingress only** (north-south) | **Full mesh** (north-south + east-west) |
| Sidecar injection | 미적용 | 클러스터 전체 적용 |
| Gateway API CRDs | ✅ 설치 | ✅ 설치 |
| Istio CRDs | ❌ **미설치** | ✅ 설치 |
| 업그레이드 방식 | **In-place** (minor/patch 모두) | Canary (minor), In-place (patch) |

### Istio CRDs 미설치 — App Routing Istio의 핵심 차이

App Routing Istio는 "Istio"라는 이름이 붙어있지만, 실제로는 **Envoy 기반 Gateway API 구현체**로만 동작한다.
Inbound 트래픽 라우팅만 담당하므로 Istio mesh 고유 CRDs가 설치되지 않고 사용할 수 없다.

```
App Routing Istio에서 사용 가능한 리소스:
  ✅ Gateway              (gateway.networking.k8s.io)
  ✅ HTTPRoute            (gateway.networking.k8s.io)
  ✅ ReferenceGrant       (gateway.networking.k8s.io)

App Routing Istio에서 사용 불가한 Istio CRDs:
  ❌ VirtualService       (networking.istio.io)   — L7 라우팅, traffic split
  ❌ DestinationRule      (networking.istio.io)   — load balancing, circuit breaker
  ❌ PeerAuthentication   (security.istio.io)     — mTLS 정책
  ❌ AuthorizationPolicy  (security.istio.io)     — 서비스 간 접근 제어
  ❌ ServiceEntry         (networking.istio.io)   — 외부 서비스 등록
  ❌ Sidecar              (networking.istio.io)   — sidecar 범위 설정
```

> **요약:** App Routing Istio = Gateway API 리소스만 사용 가능.
> Istio의 VirtualService, DestinationRule 등 mesh 기능이 필요하면 Full Istio Service Mesh(`--enable-azure-service-mesh`)를 사용해야 한다.

### 업그레이드 정책

- Istio 컨트롤 플레인 버전은 **K8s 버전에 연동**됨 — AKS가 호환되는 Istio 버전을 자동 결정
- Patch 버전: AKS 릴리스에 따라 자동 업그레이드
- Minor 버전: K8s 버전 업그레이드 시 또는 새 Istio minor 버전 릴리스 시 in-place 업그레이드
- 클러스터에 maintenance window가 설정되어 있으면 istiod 업그레이드 시 해당 윈도우를 존중함

> ⚠️ App Routing Istio는 서비스 메시 add-on과 달리 **canary 방식이 아닌 in-place 업그레이드**이다.
> HPA와 PDB가 영향을 최소화하지만, 프로덕션 환경에서는 maintenance window 설정을 권장한다.

### 동시 사용 불가

두 옵션은 **같은 Istio 컨트롤 플레인을 다른 범위로 설치**하는 것이라 동시 사용 시 충돌한다.

```bash
# Full Istio가 켜진 클러스터에서 App Routing Istio 활성화 시도 → 에러 발생
az aks update -g myRG -n myCluster --enable-app-routing-istio
# → 차단됨 (두 Istio 컨트롤 플레인 충돌)
```

### 조합 호환성

| 조합 | 가능 여부 |
|---|---|
| `--enable-azure-service-mesh` + App Routing **NGINX** | ✅ 가능 |
| `--enable-azure-service-mesh` + `--enable-app-routing-istio` | ❌ 충돌 |
| `--enable-azure-service-mesh` + Gateway API (Istio 자체 gateway) | ✅ 가능 |

> **정리:** Full Istio를 이미 사용 중이라면 App Routing Istio는 필요 없고 쓸 수도 없다.
> Full Istio 자체가 Gateway API를 지원하므로 Gateway + HTTPRoute를 직접 만들면 된다.
> App Routing Istio는 **서비스 메시 없이 ingress만 Gateway API로 전환**하려는 환경을 위한 경량 옵션이다.

## 참고 F. allowedRoutes — Gateway의 HTTPRoute 접근 범위 제한

### NGINX와의 차이

NGINX Ingress에서는 IngressClass 하나에 **모든 namespace의 Ingress가 무조건 연결**되었다.
Gateway API에서는 `allowedRoutes` 설정으로 **어떤 namespace의 HTTPRoute가 붙을 수 있는지 제어**할 수 있다.

> ⚠️ **기본값은 `Same` (동일 namespace만 허용)**이다.
> 명시하지 않으면 다른 namespace의 HTTPRoute는 자동 거부된다.

### 설정 옵션

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
spec:
  gatewayClassName: approuting-istio
  listeners:
  - name: http
    port: 80
    protocol: HTTP
    allowedRoutes:
      namespaces:
        from: Same       # Same | All | Selector
```

| 값 | 의미 |
|---|---|
| `Same` (기본값) | Gateway와 **같은 namespace**의 HTTPRoute만 연결 가능 |
| `All` | **모든 namespace**의 HTTPRoute가 연결 가능 |
| `Selector` | **label selector로 지정한 namespace**만 허용 |

### 동작 예시

```
allowedRoutes 미지정 또는 from: Same (기본값):
  demo/gateway ← demo/httproute     ✅ 허용
  demo/gateway ← prod/httproute     ❌ 자동 거부

allowedRoutes.from: All:
  demo/gateway ← demo/httproute     ✅ 허용
  demo/gateway ← prod/httproute     ✅ 허용

allowedRoutes.from: Selector (label: shared=true):
  demo/gateway ← demo/httproute     ✅ (label 있으면)
  demo/gateway ← prod/httproute     ✅ (label 있으면)
  demo/gateway ← test/httproute     ❌ (label 없으면)
```

### 마이그레이션 시 주의사항

NGINX에서는 namespace 구분 없이 모든 Ingress가 붙었기 때문에, Gateway API로 전환 시
**"왜 다른 namespace의 HTTPRoute가 안 붙지?"** 하는 상황이 발생할 수 있다.

**권장 패턴:**

| 패턴 | 구조 | 특징 |
|---|---|---|
| **namespace별 Gateway (권장)** | ns-a/gateway ← ns-a/httproute | namespace 격리, 팀별 독립 관리. Gateway마다 LB+IP 생성 |
| **공유 Gateway** | shared/gateway (`from: All`) ← ns-a, ns-b의 httproute | NGINX처럼 하나의 LB IP 공유. 중앙 관리 |

```
패턴 1 — namespace별 Gateway (권장):
  ns-a/gateway-a ← ns-a/httproute-a    (LB IP: 10.224.0.10)
  ns-b/gateway-b ← ns-b/httproute-b    (LB IP: 10.224.0.11)
  → 격리 ✅, IP/LB 비용 증가 ⚠️

패턴 2 — 공유 Gateway:
  shared/gateway (allowedRoutes: All) ← ns-a/httproute, ns-b/httproute
  → 하나의 IP/LB ✅, namespace 격리 ❌
```

> **멀티 namespace 환경에서 마이그레이션 시**, 기존 NGINX처럼 하나의 LB로 운영하려면
> `allowedRoutes.namespaces.from: All` 을 명시해야 한다.
> namespace별 격리가 필요하면 각 namespace에 Gateway를 생성하는 것이 Gateway API의 설계 원칙이다.
