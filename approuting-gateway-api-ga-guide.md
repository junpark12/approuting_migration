# App Routing Gateway API (GA) 적용 가이드

> 기준 릴리스: AKS `2026-04-28` (v20260428) — App Routing Gateway API **GA**
> 공식 문서: <https://learn.microsoft.com/en-us/azure/aks/app-routing-gateway-api>
> 테스트 환경: `ApproutingTest` RG / `aks-approuting` (East US, K8s 1.34.7)
>
> 본 가이드는 **GA 경로**만 다룹니다. Preview Feature Flag 등록 절차가 필요한 환경(예: RP 롤아웃 미완료 리전)은 별도 가이드(`approuting-migration-guide.md`)를 참고하세요.

---

## 1. 사전 요구사항

### 1.1 azure-cli 버전
GA 사용 시 **`azure-cli` 2.86.0 이상** 필요. Preview 익스텐션 불필요.

```bash
az --version | head -3
# 필요 시 업그레이드
az upgrade
```

### 1.2 리전 롤아웃 상태 확인
RP가 해당 리전에 v20260428 이상으로 배포 완료된 경우에만 GA 기능 사용 가능. 리전별 RP 롤아웃 현황은 [AKS Release Tracker](https://releases.aks.azure.com/)에서 확인할 수 있다.

| 리전 (확인 시점 2026-06) | RP 버전 | 상태 |
|---|---|---|
| East US | v20260428 | ✅ Finished |
| Korea Central | v20260428 | ✅ Finished |

### 1.3 기존 Istio/Gateway API 리소스 충돌 확인
다음이 클러스터에 이미 있으면 사전 정리 필요:

| 항목 | 검사 | 조치 |
|---|---|---|
| Istio service mesh add-on | `az aks show ... --query serviceMeshProfile` | App Routing GW API와 **동시 활성화 불가** → 먼저 비활성화 |
| 기존 Istio CRD (`*.istio.io`) | `kubectl get crd \| grep istio.io` | 잔존 시 App Routing Istio control plane 기동 실패 → 삭제 |
| 기존 `istio` GatewayClass | `kubectl get gatewayclass istio` | 삭제 |
| Experimental 채널 Gateway API CRD | `kubectl get crd gateways.gateway.networking.k8s.io -o yaml \| grep channel` | Standard 채널만 허용 → 제거 후 진행 |

> ⚠️ Istio CRD 삭제는 해당 CR(VirtualService/DestinationRule 등)도 함께 삭제됨.
> ```bash
> kubectl delete crd $(kubectl get crd -o name | grep -E 'istio\.io')
> kubectl delete gatewayclass istio
> ```

---

## 2. 환경 변수 설정

```bash
export RESOURCE_GROUP=ApproutingTest
export CLUSTER=aks-approuting
export LOCATION=eastus
```

---

## 3. App Routing Gateway API (Istio) 활성화

> **사전조건**: [Managed Gateway API installation](https://learn.microsoft.com/en-us/azure/aks/managed-gateway-api)이 활성화되어 있어야 함 (`--enable-gateway-api`).
> Self-managed Gateway API CRD 사용은 미지원.

### 3.1 신규 클러스터 (참고)
Managed Gateway API와 App Routing Istio를 **한 번에** 활성화 가능.
```bash
az aks create -g $RESOURCE_GROUP -n $CLUSTER -l $LOCATION \
  --enable-gateway-api \
  --enable-app-routing-istio \
  --node-count 2 \
  --node-vm-size Standard_D2a_v4 \
  --generate-ssh-keys
```

### 3.2 기존 클러스터 (현재 환경)
순서대로 활성화 (또는 한 번에 묶어도 됨).
```bash
# 1) Managed Gateway API CRD 설치 (사전조건)
az aks update -g $RESOURCE_GROUP -n $CLUSTER --enable-gateway-api

# 2) App Routing Gateway API (Istio) 활성화
az aks update -g $RESOURCE_GROUP -n $CLUSTER --enable-app-routing-istio
```

> 💡 두 명령을 한 번에:
> ```bash
> az aks update -g $RESOURCE_GROUP -n $CLUSTER \
>   --enable-gateway-api --enable-app-routing-istio
> ```
>
> ⚠️ 순서가 바뀌어 `--enable-app-routing-istio`를 먼저 실행해도 명령 자체는 성공하지만, **GatewayClass와 customization ConfigMap(`istio-gateway-class-defaults`)이 생성되지 않아** Gateway 리소스를 사용할 수 없습니다. 이 경우 `--enable-gateway-api`를 나중에 실행해도 자동으로 정상화됩니다.

### 3.3 kubeconfig 가져오기
```bash
az aks get-credentials -g $RESOURCE_GROUP -n $CLUSTER --overwrite-existing
```

---

## 4. 설치 검증

### 4.1 Gateway API CRD
```bash
kubectl get crd | grep gateway.networking.k8s.io
```
예상 출력:
```
gatewayclasses.gateway.networking.k8s.io
gateways.gateway.networking.k8s.io
grpcroutes.gateway.networking.k8s.io
httproutes.gateway.networking.k8s.io
referencegrants.gateway.networking.k8s.io
```

| AKS K8s 버전 | Gateway API Bundle |
|---|---|
| 1.26.x – 1.33.x | v1.2.1 |
| 1.34.x (현재) | **v1.3.0** |
| 1.35.0+ | v1.4.1 |

CRD bundle 버전 확인:
```bash
kubectl get crd gateways.gateway.networking.k8s.io -o jsonpath='{.metadata.annotations}'
```

### 4.2 GatewayClass
```bash
kubectl get gatewayclass
# 예상: approuting-istio
```

### 4.3 Istio control plane (App Routing 전용)
```bash
kubectl get pods -n aks-istio-system
# istiod-xxx 2개가 Running

kubectl get validatingwebhookconfiguration
# azure-service-mesh-ccp-validating-webhook 존재 확인
```

### 4.4 Gateway 기본 설정 ConfigMap
```bash
kubectl get cm -n aks-istio-system istio-gateway-class-defaults
```
이 ConfigMap에서 Gateway 인프라(Deployment/Service/HPA/PDB)의 **기본값**과 **허용 커스터마이즈 항목**이 관리됨.

### 4.5 GA 빌드 검증 (선택)

`aks-preview` 익스텐션이 설치되어 있으면 CLI 경고에 "is in preview"가 나올 수 있습니다. 실제로 클러스터에 배포된 것이 GA 릴리스 빌드인지 확인하려면 다음 두 가지를 비교하세요.

#### a) istiod 이미지 태그가 GA 릴리스 노트와 일치하는지
2026-04-28 릴리스 노트: *"asm-1-27 to `1.27.9-2`, asm-1-28 to `1.28.6-1`, **asm-1-29 to `1.29.2-1`**"*

```bash
kubectl get deploy -n aks-istio-system istiod \
  -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
# 예: mcr.microsoft.com/oss/v2/istio/pilot:v1.29.2-1
```

이미지 태그가 릴리스 노트의 GA 빌드와 일치하면 ✅

#### b) add-on Helm chart 빌드가 GA 릴리스 이후의 것인지
```bash
helm list -A | grep -E 'azure-service-mesh-istio-discovery|aks-managed-overlay'
```

예시 출력:
```
azure-service-mesh-istio-discovery  kube-system  12  2026-05-27 ...
  azure-service-mesh-istio-discovery-addon-1.0.0-v20260520-addon-260521-1
  1.0.0-v20260520-addon-260521-1
```

`v20260520-addon-260521-1`처럼 **2026-04-28 릴리스 이후의 빌드 날짜**(v2026MMDD…)가 박혀 있으면 GA 채널에서 패치된 차트가 적용된 것 ✅

> 두 가지가 모두 GA 릴리스 노트와 정합되면, CLI 경고와 무관하게 **데이터 플레인 자체는 GA 빌드로 동작 중**이라고 판단할 수 있습니다.

---

## 5. 샘플 앱으로 동작 확인 (HTTP, External LB)

### 5.1 샘플 앱 배포
```bash
export ISTIO_RELEASE=release-1.27
kubectl apply -f https://raw.githubusercontent.com/istio/istio/$ISTIO_RELEASE/samples/httpbin/httpbin.yaml
```

### 5.2 Gateway + HTTPRoute
```bash
kubectl apply -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: httpbin-gateway
spec:
  gatewayClassName: approuting-istio
  listeners:
  - name: http
    port: 80
    protocol: HTTP
    allowedRoutes:
      namespaces:
        from: Same
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: httpbin
spec:
  parentRefs:
  - name: httpbin-gateway
  hostnames: ["httpbin.example.com"]
  rules:
  - matches:
    - path: { type: PathPrefix, value: /get }
    backendRefs:
    - name: httpbin
      port: 8000
EOF
```

### 5.3 자동 프로비저닝되는 리소스 확인
Gateway 1개 생성 시 다음이 자동 생성됨 (네이밍: `<gateway-name>-<gatewayClassName>`):

```bash
kubectl get deployment,svc,hpa,pdb -l 'gateway.networking.k8s.io/gateway-name=httpbin-gateway'
```
예상:
- `Deployment` `httpbin-gateway-approuting-istio` (replicas 2)
- `Service` (LoadBalancer)
- `HorizontalPodAutoscaler` (min 2, max 5)
- `PodDisruptionBudget` (minAvailable 1)

### 5.4 트래픽 테스트
```bash
kubectl wait --for=condition=programmed gateways.gateway.networking.k8s.io httpbin-gateway
export INGRESS_HOST=$(kubectl get gateways.gateway.networking.k8s.io httpbin-gateway \
  -o jsonpath='{.status.addresses[0].value}')

curl -s -I -H "Host: httpbin.example.com" "http://$INGRESS_HOST/get"
# HTTP/1.1 200 OK
```

---

## 6. Internal LoadBalancer 패턴 (사내망 노출 시)

기본은 Public LB. Internal LB로 노출하려면 `Gateway`에 annotation 추가:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: pixel-istio-gw
  namespace: demo
  annotations:
    service.beta.kubernetes.io/azure-load-balancer-internal: "true"
    # 특정 IP 고정이 필요한 경우
    service.beta.kubernetes.io/azure-load-balancer-ipv4: "10.224.0.7"
spec:
  gatewayClassName: approuting-istio
  listeners:
  - name: http
    hostname: echo.internal.demo
    port: 80
    protocol: HTTP
    allowedRoutes:
      namespaces:
        from: Same
```

> 허용되는 LB annotation 목록은 [istio-gateway-api#annotation-customizations](https://learn.microsoft.com/en-us/azure/aks/istio-gateway-api#annotation-customizations) 참고. 허용 목록 외의 annotation은 webhook이 차단함.

---

## 7. 액세스 로그 (Gateway/Envoy)

> **Preview 시절 이슈**: 고객 환경에서 Gateway Pod의 액세스 로그가 출력되지 않는 문제가 보고됨 — [Azure/AKS#5728](https://github.com/Azure/AKS/issues/5728).
> **GA 시점 확인 (2026-04-28 릴리스, East US, v1.29.2-1)**: 별도 설정 없이 **기본 활성화**되어 Envoy JSON 액세스 로그가 stdout으로 정상 출력됨.

### 7.1 확인 방법
```bash
# Gateway Pod 확인 (네이밍: <gateway-name>-<gatewayClassName>-...)
kubectl get pods -n demo -l 'gateway.networking.k8s.io/gateway-name=pixel-istio-gw'

# 로그 조회
kubectl logs -n demo <gateway-pod-name> --tail=50
```

### 7.2 출력 예시 (HTTP 요청 1건)
```json
{
  "authority": "echo.internal.demo",
  "method": "GET",
  "path": "/",
  "protocol": "HTTP/1.1",
  "response_code": 200,
  "response_flags": "-",
  "route_name": "demo.echo-route.0",
  "upstream_cluster": "outbound|80||echo.demo.svc.cluster.local",
  "upstream_host": "10.244.0.156:80",
  "upstream_service_time": "10",
  "duration": 10,
  "bytes_received": 0,
  "bytes_sent": 1946,
  "downstream_remote_address": "10.224.0.5:29145",
  "user_agent": "curl/8.20.0",
  "x_forwarded_for": "10.224.0.5",
  "request_id": "8c8bbcbb-42ea-4560-8bc3-aa97cb2173b0",
  "start_time": "2026-05-27T07:09:30.636Z"
}
```

주요 필드:
| 필드 | 의미 |
|---|---|
| `authority` | Host 헤더 (가상 호스트) |
| `method`/`path`/`protocol` | 요청 라인 |
| `response_code` | HTTP 응답 코드 |
| `response_flags` | Envoy 응답 플래그 (`-` 정상, `UH`/`UF` 등 실패 사유 — [참고](https://www.envoyproxy.io/docs/envoy/latest/configuration/observability/access_log/usage#config-access-log-format-response-flags)) |
| `route_name` | 매칭된 HTTPRoute (`<ns>.<name>.<rule-index>`) |
| `upstream_cluster` / `upstream_host` | 백엔드 cluster와 실제 선택된 Pod |
| `upstream_service_time` | 백엔드 처리 시간 (ms) |
| `duration` | 전체 처리 시간 (ms) |
| `x_forwarded_for` | 클라이언트 원본 IP (LB 단계 포함 체인) |
| `request_id` | 트레이싱용 요청 ID |

### 7.3 로그 수집 운영
- **stdout** 출력이므로 AKS 표준 로그 수집 경로(Container Insights / Log Analytics, Fluent Bit/Loki 등) **그대로 수집** 가능
- JSON 포맷이라 KQL/LogQL에서 필드 단위 파싱 용이
- 포맷 자체는 Istio/Envoy 표준 access log이므로 분석 노하우 재사용 가능
- 별도 커스터마이즈가 필요한 경우(필드 추가/제외, 포맷 변경)는 [istio-gateway-api#configmap-customizations](https://learn.microsoft.com/en-us/azure/aks/istio-gateway-api#configmap-customizations)의 허용 항목 확인 필요

> ✅ Preview 시점 고객 우려 사항(액세스 로그 부재)은 GA에서 해소된 것으로 관찰됩니다.

### 7.4 Client IP 보존 (`externalTrafficPolicy: Local`)

기본 설정에서는 access log의 `x_forwarded_for` / `downstream_remote_address`에 **실제 클라이언트 IP가 아닌 노드 IP(예: `10.224.0.5`)** 가 기록된다. 이는 Service의 `externalTrafficPolicy`가 기본값 `Cluster`이어서, kube-proxy가 외부 트래픽을 다른 노드의 Pod로 SNAT하면서 source IP가 노드 IP로 치환되기 때문이다.

GA된 App Routing Gateway API add-on은 자동 생성된 Service의 일부 필드를 **§9 ConfigMap 커스터마이즈**를 통해 변경하는 것을 허용한다 (§9.4 Service allow-list 참고). 그 중 `spec.externalTrafficPolicy: Local`을 적용하면 외부 트래픽을 받은 노드의 로컬 Pod로만 라우팅 → SNAT 미발생 → 원본 client IP 보존된다.

#### 적용 방법 (GatewayClass-level)

`gatewayClassName: approuting-istio`인 모든 Gateway에 일괄 적용:

```bash
kubectl edit cm istio-gateway-class-defaults -n aks-istio-system
```

다음 `data.service` 블록을 추가/병합:

```yaml
data:
  service: |
    spec:
      externalTrafficPolicy: Local
```

#### 적용 방법 (Gateway-level — 특정 Gateway만)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-gateway-overrides
  namespace: demo
data:
  service: |
    spec:
      externalTrafficPolicy: Local
---
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: pixel-istio-gw
  namespace: demo
spec:
  gatewayClassName: approuting-istio
  infrastructure:
    parametersRef:
      group: ""
      kind: ConfigMap
      name: my-gateway-overrides
  listeners:
    - name: http
      port: 80
      protocol: HTTP
```

#### 적용 후 확인

```bash
# Service 정책 확인
kubectl get svc -n demo -l 'gateway.networking.k8s.io/gateway-name=pixel-istio-gw' \
  -o jsonpath='{.items[0].spec.externalTrafficPolicy}'
# → Local
```

실제 access log는 **Container Insights의 `ContainerLogV2` 테이블**에서 확인 가능 (Log Analytics workspace):

```kusto
ContainerLogV2
| where PodNamespace == "demo"
| where ContainerName has "istio-proxy" or PodName has "pixel-istio-gw"
| extend log = parse_json(LogMessage)
| project TimeGenerated,
          x_forwarded_for = tostring(log.x_forwarded_for),
          downstream      = tostring(log.downstream_remote_address),
          method          = tostring(log.method),
          path            = tostring(log.path),
          status          = toint(log.response_code)
| order by TimeGenerated desc
```

→ `x_forwarded_for` / `downstream_remote_address`에 실제 client public IP가 기록되는 것을 확인.

> ⚠️ `externalTrafficPolicy: Local` 사이드 이펙트
> - 외부 트래픽을 수신한 노드에 **해당 Gateway의 Pod가 1개 이상 있어야** 라우팅됨. Pod 없는 노드로 인입되면 패킷 drop → AKS LB의 health probe가 자동으로 Pod 없는 노드를 제외하므로 통상 문제 없으나, **HPA min replicas는 노드 수 대비 충분히** 두는 것이 안전.
> - **트래픽이 노드 간 균등 분산되지 않을 수 있음** (노드별 Pod 수 차이에 따라). 필요 시 §9의 `topologySpreadConstraints`로 분산.

---

## 8. Session Affinity (sticky session) 현황

NGINX Ingress의 cookie 기반 session affinity(`nginx.ingress.kubernetes.io/affinity: cookie`)에 대응하는 Gateway API 스펙은 **HTTPRoute `sessionPersistence` (v1.1+)** 이지만, 2026-05 기준 App Routing Gateway API GA에서는 **사실상 사용 불가**.

### 8.1 현황 요약
| 레이어 | 상태 | 근거 |
|---|---|---|
| Gateway API 스펙 | v1.1에 추가되었으나 **Experimental channel 전용** | 본 가이드 §4.1에서 확인한 AKS App Routing의 CRD bundle은 **Standard channel (v1.3.0)** → `sessionPersistence` 필드 자체가 CRD 스키마에 없음 → apply 시 unknown field 거부 가능성 |
| upstream Istio | **미구현** | [istio/istio#55839 — Implement SessionPersistence in Gateway API](https://github.com/istio/istio/issues/55839) (2026-05 open, 진행 중) |
| App Routing Istio add-on | **미지원** | upstream Istio가 미구현이므로 자동 미지원 |

### 8.2 현실적 대안
**유일한 현실적 대안 = 앱 레벨 sticky session 처리** (앱이 자체적으로 Set-Cookie 발급 후 상태 저장소 / 일관된 라우팅 구현).

다음은 **대안이 아닌 이유**:
- ❌ **Istio service mesh add-on의 VirtualService + DestinationRule(consistentHash)**: App Routing과 동시 활성화 **불가** (§9, §10 참고). 둘 중 하나만 선택 가능하므로 App Routing을 쓰는 한 사용할 수 없음.
- ❌ Gateway 직접 패치: CRD에 필드가 없어 무효.

### 8.3 향후 전망 (참고용, 보장 없음)
- upstream Istio가 [#55839](https://github.com/istio/istio/issues/55839)를 구현하더라도, 그 변경이 **AKS App Routing add-on에 언제(혹은 실제로) 반영될지는 미지수**.
- 구현 → upstream Istio 릴리스 포함 → asm-N-NN 빌드 채택 → AKS 릴리스 트레인 반영의 단계가 모두 필요.
- 따라서 신규 설계 시점에는 **앱 레벨 처리 전제**로 계획 권장.

---

## 9. Gateway 커스터마이즈

### 9.1 무엇을 커스터마이즈하는가
Gateway 리소스 1개를 만들면 AKS가 자동으로 다음 4개 리소스를 생성한다(§5.3 참고):

| 자동 생성 | 역할 |
|---|---|
| **Deployment** | Envoy proxy Pod 실행 |
| **Service** | LoadBalancer (외부 IP) |
| **HorizontalPodAutoscaler** | 자동 스케일 (기본 min 2 / max 5 / CPU 80%) |
| **PodDisruptionBudget** | 중단 보호 (기본 minAvailable 1) |

이 4개 리소스의 운영 파라미터(replicas, 노드 배치, 리소스 limit, HPA min/max 등)를 **ConfigMap을 통해** 조정한다. Gateway API 표준 스펙(`Gateway` yaml)에는 이런 운영 필드가 없기 때문에 Istio 자동 배포 모델이 추가 제공하는 메커니즘이다.

App Routing 공식 문서는 이 커스터마이즈에 대해 [Istio mesh add-on 문서를 그대로 참조하라](https://learn.microsoft.com/en-us/azure/aks/app-routing-gateway-api#gateway-resource-customization)고 명시하고 있다. 즉 **App Routing과 mesh add-on이 동일한 allow-list / 동일한 ConfigMap 스키마**를 공유한다.

### 9.2 두 단계 적용 (Override 관계)

| 단계 | 위치 | 적용 범위 | 비고 |
|---|---|---|---|
| **GatewayClass-level** (기본값) | `aks-istio-system/istio-gateway-class-defaults` | `gatewayClassName: approuting-istio`인 **모든 Gateway** | AKS가 자동 생성·reconcile. GatewayClass당 1개만 허용 |
| **Gateway-level** (개별 override) | 임의 namespace의 ConfigMap + `Gateway.spec.infrastructure.parametersRef`로 연결 | 특정 Gateway 1개 | 동일 필드는 Gateway-level이 GatewayClass-level을 **override** |

### 9.3 ConfigMap data 스키마
data 키는 자동 생성 리소스 종류별로 분리된다 (`deployment`, `service`, `horizontalPodAutoscaler`, `podDisruptionBudget`). 각 값은 해당 리소스의 부분 spec을 yaml string으로 갖는다.

```yaml
data:
  deployment: |
    spec:
      minReadySeconds: 15
  horizontalPodAutoscaler: |
    spec:
      minReplicas: 3
      maxReplicas: 6
  podDisruptionBudget: |
    spec:
      minAvailable: 2
```

### 9.4 Allow-list (커스터마이즈 가능한 필드)
허용 외 필드는 add-on managed webhook이 차단한다. 전체 목록은 [공식 문서](https://learn.microsoft.com/en-us/azure/aks/istio-gateway-api#configmap-customizations) 기준이며, 주요 항목만 발췌:

#### Deployment (`data.deployment`)
| 필드 | 용도 |
|---|---|
| `metadata.labels` / `metadata.annotations` | 라벨/주석 |
| `spec.replicas` | 정적 replica 수 (보통 HPA 사용 권장) |
| `spec.minReadySeconds` ⭐ | 새 Pod이 Ready 후 다음 롤링까지 대기 시간 (**2026-04-28 릴리스 신규**) |
| `spec.template.metadata.labels` / `annotations` | Pod 라벨/주석 |
| `spec.template.spec.nodeSelector` / `nodeName` / `tolerations` | 노드 배치 |
| `spec.template.spec.affinity.*` (node/pod/anti) | 어피니티 |
| `spec.template.spec.topologySpreadConstraints` | 토폴로지 분산 |
| `spec.template.spec.containers.resources.requests` / `limits` | CPU/메모리 |
| `spec.template.spec.containers.resizePolicy` | 리소스 in-place resize |

#### Service (`data.service`)
| 필드 | 용도 |
|---|---|
| `metadata.labels` / `annotations` | LB annotation은 보통 `Gateway.spec.infrastructure.annotations`에서 처리(§6 참고) |
| `spec.type` | Service type |
| `spec.loadBalancerSourceRanges` | LB 인입 CIDR 제한 |
| `spec.loadBalancerClass` | LB 컨트롤러 선택 |
| `spec.externalTrafficPolicy` / `internalTrafficPolicy` | 트래픽 정책 (`Local` 등) |

#### HorizontalPodAutoscaler (`data.horizontalPodAutoscaler`)
| 필드 | 용도 |
|---|---|
| `spec.minReplicas` | 최소 Pod 수. **2 미만 불가** (PDB 충돌 방지) |
| `spec.maxReplicas` | 최대 Pod 수 |
| `spec.metrics` | 커스텀 메트릭 |
| `spec.behavior.scaleUp/scaleDown.{stabilizationWindowSeconds, selectPolicy, policies}` | 스케일 진동/속도 제어 |

#### PodDisruptionBudget (`data.podDisruptionBudget`)
| 필드 | 용도 |
|---|---|
| `spec.minAvailable` | 동시에 죽일 수 없는 최소 가용 Pod 수 |
| `spec.unhealthyPodEvictionPolicy` | unhealthy Pod 강제 eviction 정책 |

> ⚠️ PDB minAvailable / eviction 정책 변경은 노드 업그레이드·삭제 시 `UpgradeFailed`(PodDrainFailure) 원인이 될 수 있음. [PDB troubleshooting 가이드](https://learn.microsoft.com/en-us/troubleshoot/azure/azure-kubernetes/create-upgrade-delete/error-code-poddrainfailure) 참고.

### 9.5 예시 ① — GatewayClass-level (모든 Gateway에 기본값 적용)

```bash
kubectl edit cm istio-gateway-class-defaults -n aks-istio-system
```

```yaml
data:
  deployment: |
    metadata:
      labels:
        owner: platform-team
    spec:
      minReadySeconds: 15
  horizontalPodAutoscaler: |
    spec:
      minReplicas: 3
      maxReplicas: 6
  podDisruptionBudget: |
    spec:
      minAvailable: 1
```

> GatewayClass-level ConfigMap은 AKS가 reconcile 하므로 직접 만들지 말고 `kubectl edit`으로 기존 것을 수정한다. (`gateway.istio.io/defaults-for-class=approuting-istio` 라벨 보존)

### 9.6 예시 ② — Gateway-level (특정 Gateway만 다르게)

1) ConfigMap 작성 (임의 namespace, 보통 Gateway와 같은 ns):
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: httpbin-gw-options
  namespace: default
data:
  horizontalPodAutoscaler: |
    spec:
      minReplicas: 2
      maxReplicas: 4
  deployment: |
    spec:
      template:
        spec:
          nodeSelector:
            workload: ingress
          tolerations:
          - key: ingress-only
            operator: Exists
            effect: NoSchedule
```

2) Gateway에서 `infrastructure.parametersRef`로 ConfigMap 연결:
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: httpbin-gateway
  namespace: default
spec:
  gatewayClassName: approuting-istio
  infrastructure:
    parametersRef:
      group: ""
      kind: ConfigMap
      name: httpbin-gw-options
  listeners:
  - name: http
    port: 80
    protocol: HTTP
    allowedRoutes:
      namespaces:
        from: Same
```

3) 검증:
```bash
kubectl get hpa httpbin-gateway-approuting-istio
# MINPODS 2, MAXPODS 4 로 변경 확인
```

### 9.7 결정 가이드
| 무엇을? | 어디서? |
|---|---|
| LB 종류/서브넷/Health probe 등 **LoadBalancer 동작** | `Gateway.spec.infrastructure.annotations` (§6) |
| Pod 개수, HPA, PDB, 노드 배치, 리소스 limit, minReadySeconds | **본 §9 ConfigMap** |
| 라우팅 규칙, 헤더 변형, 리다이렉트, 타임아웃 | `HTTPRoute` |
| 클러스터 전체 기본값 통일 | GatewayClass-level (§9.5) |
| 특정 Gateway만 다르게 | Gateway-level (§9.6, GatewayClass-level을 override) |

---

## 10. 제약 사항 (GA 시점)

| 항목 | 현황 | 대안 |
|---|---|---|
| Istio service mesh add-on과 동시 활성화 | ❌ 불가 | 둘 중 하나만 선택 |
| Azure DNS / TLS 인증서 자동 통합 (NGINX의 `tls-cert-keyvault-uri` annotation 같은 것) | ❌ 미지원 | CSI Secrets Store + SPC 수동 구성 (별도 가이드 예정) |
| `TLSRoute` (SNI passthrough) | ❌ 미지원 | Istio 1.30 도입 시 지원 예정 |
| Egress 트래픽 관리 | ❌ 미지원 | — |
| Sidecar injection / Istio CRD 사용 | ❌ 미지원 (인프라 전용) | 메시 기능 필요 시 Istio mesh add-on 별도 사용 |

### 10.1 알려진 이슈 — istiod HPA `requests.cpu` 누락 (Bug)

**증상**

App Routing Gateway API 활성화 시 `aks-istio-system` ns에는 고객 요청을 직접 처리하지는 않지만 컨트롤 플레인 `istiod-asm-...` Deployment가 자동 배포되며, 함께 HPA(HorizontalPodAutoscaler)도 생성된다. 그러나 해당 Deployment의 Pod 스펙에 **`resources.requests.cpu`가 누락**되어 HPA가 CPU 사용률을 계산할 기준점을 찾지 못해 이벤트에 오류가 반복 발생한다.

```bash
kubectl get hpa -n aks-istio-system
kubectl describe hpa -n aks-istio-system <istiod-hpa-name>
```

```
Warning  FailedGetResourceMetric        ...  failed to get cpu utilization:
  missing request for cpu in container discovery of Pod istiod-asm-1-29-...
Warning  FailedComputeMetricsReplicas   ...
```

```bash
# Deployment 스펙 — requests 비어있음을 확인
kubectl get deploy -n aks-istio-system -l app=istiod \
  -o jsonpath='{.items[*].spec.template.spec.containers[*].resources}'
```

**영향**

- istiod HPA가 metric 계산 불가 → **자동 스케일 아웃 미작동** (`minReplicas`로 고정 운영)
- 관제/모니터링에서 이벤트 노이즈로 알람 오인식 가능
- 고객이 배포한 **Gateway용 HPA(`<gateway-name>-...`)는 정상 동작** — 본 이슈는 컨트롤 플레인 측에만 해당

**원인**

`aks-istio-system` ns는 **AKS add-on managed** 영역으로, 고객이 Deployment/HPA 스펙을 직접 patch해도 add-on reconciler가 원상복구한다. AKS 측에서 배포하는 istiod manifest에 `requests.cpu`가 누락되어 있어 **고객 측 워크어라운드 불가**.

**대응**

- 본 항목은 **AKS(Microsoft) 쪽에서 관리하는 영역**이므로, **Azure Support 케이스를 통해 bug fix 요청** 예정.
- 임시 회피책은 없음 (고객이 스펙을 patch 해도 reconcile 됨).
- Gateway data plane(Envoy)의 HPA는 정상 동작하므로 서비스 가용성에 직접 영향은 없으나, 관제 이벤트 필터링이 필요할 수 있음.

---

## 11. Istio service mesh add-on과의 차이

| 항목 | App Routing Gateway API | Istio service mesh add-on |
|---|---|---|
| GatewayClass | `approuting-istio` | `istio` |
| Sidecar/CRD | 미지원 (인프라 전용) | 지원 |
| 업그레이드 방식 | **In-place** (minor/patch 모두) | **Revisioned** canary (minor), in-place (patch) |
| 대상 용도 | NGINX Ingress 대체 / GW API 표준 ingress | 풀 메시 (mTLS, 정책, 텔레메트리) |

---

## 12. NGINX Ingress 마이그레이션 시 매핑 요약

| NGINX (current_env) | Gateway API (tobe_env) |
|---|---|
| `IngressClass: nginx` | `Gateway.spec.gatewayClassName: approuting-istio` |
| `Ingress.spec.rules[].host` | `Gateway.spec.listeners[].hostname` 또는 `HTTPRoute.spec.hostnames` |
| `Ingress.spec.rules[].http.paths` | `HTTPRoute.spec.rules[].matches[].path` |
| `service.beta.kubernetes.io/azure-load-balancer-internal: "true"` (controller) | `Gateway` annotation (동일 key) |
| `nginx.ingress.kubernetes.io/*` annotation 다수 | ConfigMap 커스터마이즈 또는 HTTPRoute filter, 일부는 미대응 (별도 매핑 표 작성 필요) |

> 상세 annotation 매핑은 기존 `approuting-migration-guide.md`의 매핑 표 참고.

---

## 13. 운영 체크리스트

- [ ] 대상 리전의 RP 상태 = Finished
- [ ] `az --version >= 2.86.0`
- [ ] 기존 Istio mesh add-on 미사용 또는 사전 비활성화
- [ ] `--enable-gateway-api` + `--enable-app-routing-istio` 적용
- [ ] `aks-istio-system` 네임스페이스의 `istiod` Pod Running
- [ ] `gatewayclass approuting-istio` Accepted
- [ ] 샘플 Gateway 1개 생성 → Deployment/Service/HPA/PDB 자동 생성 확인
- [ ] Internal/External LB 요구사항에 맞춰 annotation 적용
- [ ] (별도 가이드) TLS / Key Vault 인증서 연동
- [ ] (별도 작업) NGINX Ingress → HTTPRoute 마이그레이션 및 트래픽 컷오버

---

## 부록 A. Gateway API 리소스 명명 규칙

기본적으로 Istio control plane이 Gateway 리소스로부터 다음 이름의 리소스를 만듦:
```
<gateway-name>-<gatewayClassName>
예) httpbin-gateway-approuting-istio
```
- DNS 호환(63자 이하, 소문자/숫자/하이픈) 필요
- 이름 오버라이드: `metadata.annotations["gateway.istio.io/name-override"]`

## 부록 B. 정리

테스트 클러스터 정리:
```bash
kubectl delete gateway httpbin-gateway
kubectl delete httproute httpbin
kubectl delete -f https://raw.githubusercontent.com/istio/istio/$ISTIO_RELEASE/samples/httpbin/httpbin.yaml

# 필요 시 add-on 비활성화
az aks update -g $RESOURCE_GROUP -n $CLUSTER --disable-app-routing-istio
# az aks update -g $RESOURCE_GROUP -n $CLUSTER --disable-gateway-api  # CRD 제거 (주의: 기존 Gateway/HTTPRoute 동작에 영향)
```
