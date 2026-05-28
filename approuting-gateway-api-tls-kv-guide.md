# App Routing Gateway API — Azure Key Vault TLS 연동 가이드 (GA 기준)

> 본 문서는 [`approuting-gateway-api-ga-guide.md`](./approuting-gateway-api-ga-guide.md)의 후속 가이드로, Gateway API 리스너 TLS 종료를 위해 **Azure Key Vault에 보관된 인증서**를 Kubernetes Secret으로 동기화하는 절차를 다룬다.
>
> 공식 출처: [Configure TLS termination with Gateway API and the application routing add-on](https://learn.microsoft.com/en-us/azure/aks/app-routing-gateway-api-tls)
>
> 검증 환경: AKS K8s 1.34.7 / East US / asm-1-29 (Istio 1.29.2-1) / Gateway API CRD v1.3.0 standard.

---

## 1. 무엇을 하는가 / NGINX와의 차이

NGINX Ingress 기반 App Routing에서는 `nginx.ingress.kubernetes.io/...keyvault-uri` annotation 한 줄이면 add-on이 KV→Secret 동기화를 모두 알아서 처리했다(턴키).

**Gateway API GA에서는 그 턴키 컨트롤러가 없다.** 대신 다음 5-step 수동 체인을 직접 구성해야 한다:

```
Azure Key Vault ──▶ (CSI Secrets Store Driver) ──▶ Kubernetes Secret ──▶ Gateway listener.tls.certificateRefs
                            ▲
                            └── 더미 Pod이 마운트할 때 fetch 트리거
```

### 왜 "더미 Pod"이 필요한가?

**핵심 한 줄**: Gateway/Envoy 는 SPC 를 **mount 하지 않고 K8s Secret API 로만 읽으므로**, CSI driver 가 polling 을 시작할 "mount 신호" 를 만들어주지 못한다 → **누군가 대신 mount 해줘야 함** → 더미 Pod.

좀 더 풀어 설명하면:
- CSI Secrets Store driver 는 노드에서 **mount 된 SPC volume 만** polling 대상으로 삼는다 (mount 가 0개면 그 SPC 는 polling 루프에 아예 들어가지 않음 → KV 호출도 0).
- SPC (`SecretProviderClass`) 는 단순 정의일 뿐 자체 reconciler 가 없어, 누가 mount 해주기 전까지는 Secret 도 만들어지지 않고 회전도 안 된다.
- Gateway 가 만드는 Envoy Pod 은 Secret 을 **파일 마운트로 쓰지 않고 K8s Secret API 로 받는** 구조라 mount 신호를 만들지 못한다.

→ 회피책: "**아무 Pod이나 SPC 를 마운트해서 polling trigger 를 유지**". 그 역할의 빈 Pod 이 더미 Pod 이다. (회전이 계속 돌아야 하므로 단발 Pod 이 아니라 §9 의 Deployment 형태로 상주시킨다.)

#### NGINX Ingress에서는 왜 불필요했나?
- **NGINX Pod 자체가 TLS 종료자** → 인증서 파일이 필요하므로 NGINX Pod이 CSI 볼륨을 직접 마운트 → fetch 자동 트리거 → 더미 불필요
- 게다가 App Routing NGINX는 **add-on이 자체 KV 동기화 컨트롤러를 내장**하고 있어서 사용자는 CSI/SPC조차 다룰 필요 없었음

#### Gateway API에서 사라진 두 가지
| | NGINX (Ingress) | Gateway API (App Routing GA) |
|---|---|---|
| TLS 종료 주체 | NGINX Pod (데이터 플레인) | Envoy (Gateway가 생성하는 별도 Pod) — Secret을 **마운트하지 않고 API로 읽음** |
| KV→Secret 턴키 컨트롤러 | 있음 (annotation 한 줄) | **없음** — 사용자가 CSI+SPC+더미 Pod 직접 구성 |

> 👀 관련 Issue: [Azure/AKS#5312 — "Secret sync without dummy pod when using CSI driver for AKV"](https://github.com/Azure/AKS/issues/5312) (2025-10 open, 8개월째 stalled).

---

## 2. 사전 요구사항

- [`approuting-gateway-api-ga-guide.md`](./approuting-gateway-api-ga-guide.md) 까지 완료된 클러스터 (`--enable-app-routing-istio` + `--enable-gateway-api`)
- `azure-cli` 2.86.0 이상 (`az upgrade`)
- KV는 신규 생성 전제 (본 가이드 §4에서 생성)
- 인증서 신뢰 모델: 본 가이드는 **개발/검증용 self-signed**를 사용. 운영 환경은 사내 CA 또는 공인 CA 발급 PFX/PEM을 KV에 import 하는 것을 전제로 함.

---

## 3. 환경 변수

```bash
export RESOURCE_GROUP=ApproutingTest
export CLUSTER=aks-approuting
export LOCATION=eastus
export AKV_NAME=akvapprouting$RANDOM   # 전 세계 유일해야 함
export APP_NS=demo                     # Gateway/HTTPRoute가 들어갈 ns (예시)
```

---

## 4. Azure Key Vault 생성

```bash
az keyvault create \
  --name $AKV_NAME \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION \
  --enable-rbac-authorization true \
  --enable-purge-protection true
```

옵션 의미:
- `--enable-rbac-authorization true`: **RBAC 권한 모델** 사용 (권장). Access Policy 모델을 쓰려면 `false`로.
- `--enable-purge-protection true`: 운영 환경 권장. 비활성 KV를 영구 삭제하기 전 retention 강제.

생성 확인:
```bash
az keyvault show --name $AKV_NAME --resource-group $RESOURCE_GROUP --query 'properties.{rbac:enableRbacAuthorization, uri:vaultUri}'
```

---

## 5. KV CSI Provider add-on 활성화

```bash
az aks enable-addons \
  --addons azure-keyvault-secrets-provider \
  --resource-group $RESOURCE_GROUP \
  --name $CLUSTER
```

> 신규 클러스터 생성 시점에 이미 켰다면 skip 가능. `--enable-secret-rotation true`는 §11에서 별도 다룸.

활성화 확인:
```bash
kubectl get pods -n kube-system -l app=secrets-store-csi-driver
kubectl get pods -n kube-system -l app=secrets-store-provider-azure
```

---

## 6. AKS Managed Identity → KV 권한 부여

KV CSI Provider add-on은 **별도의 user-assigned managed identity**를 자동 생성한다(=`azureKeyvaultSecretsProvider.identity`). 이 identity가 KV에 접근할 수 있어야 한다.

### 6.1 ID 추출
```bash
CLIENT_ID=$(az aks show -g $RESOURCE_GROUP -n $CLUSTER \
  --query 'addonProfiles.azureKeyvaultSecretsProvider.identity.clientId' -o tsv | tr -d '\r')
OBJECT_ID=$(az aks show -g $RESOURCE_GROUP -n $CLUSTER \
  --query 'addonProfiles.azureKeyvaultSecretsProvider.identity.objectId' -o tsv | tr -d '\r')
TENANT_ID=$(az keyvault show -g $RESOURCE_GROUP -n $AKV_NAME \
  --query 'properties.tenantId' -o tsv | tr -d '\r')
AKV_SCOPE=$(az keyvault show -g $RESOURCE_GROUP -n $AKV_NAME --query id -o tsv | tr -d '\r')

echo "CLIENT_ID=$CLIENT_ID"
echo "OBJECT_ID=$OBJECT_ID"
echo "TENANT_ID=$TENANT_ID"
```

### 6.2 권한 부여 (RBAC 모델 — 권장)
`§4`에서 `--enable-rbac-authorization true`로 만들었다면:

```bash
# Secret 읽기 (objectType: secret 또는 cert 모두 secret 형태로 가져오므로 필수)
az role assignment create \
  --role "Key Vault Secrets User" \
  --assignee-object-id $OBJECT_ID \
  --assignee-principal-type ServicePrincipal \
  --scope $AKV_SCOPE

# (선택) 인증서 메타데이터 조회까지 하려면
az role assignment create \
  --role "Key Vault Certificate User" \
  --assignee-object-id $OBJECT_ID \
  --assignee-principal-type ServicePrincipal \
  --scope $AKV_SCOPE
```

> ⚠️ CSI Driver의 `objectType: cert`도 내부적으로 KV의 **secret API**로 `.pfx`/`.pem` 본문을 가져오므로 **Key Vault Secrets User** 권한이 필수. Certificate User는 메타데이터(체인/속성) 조회용 옵션.

### 6.3 권한 부여 (Access Policy 모델)
`§4`에서 RBAC를 끄고 Access Policy 모델로 만들었다면:

```bash
az keyvault set-policy --name $AKV_NAME \
  --object-id $OBJECT_ID \
  --secret-permissions get list \
  --certificate-permissions get list
```

### 6.4 권한 부여를 진행한 본인 계정용 (인증서 업로드 위해)
```bash
ME_OBJECT_ID=$(az ad signed-in-user show --query id -o tsv)
az role assignment create \
  --role "Key Vault Certificates Officer" \
  --assignee-object-id $ME_OBJECT_ID \
  --assignee-principal-type User \
  --scope $AKV_SCOPE
az role assignment create \
  --role "Key Vault Secrets Officer" \
  --assignee-object-id $ME_OBJECT_ID \
  --assignee-principal-type User \
  --scope $AKV_SCOPE
```

> 본인 계정도 RBAC 모델에서는 별도 role이 필요. role 부여 후 토큰 반영까지 1-3분 걸릴 수 있음.

---

## 7. 인증서 생성 및 KV 업로드

### 7.1 (개발용) self-signed 인증서 생성
```bash
mkdir -p httpbin_certs && cd httpbin_certs

# Root CA
openssl req -x509 -sha256 -nodes -days 365 -newkey rsa:2048 \
  -subj '/O=example Inc./CN=example.com' \
  -keyout example.com.key -out example.com.crt

# Server cert (CN: httpbin.example.com)
openssl req -out httpbin.example.com.csr -newkey rsa:2048 -nodes \
  -keyout httpbin.example.com.key \
  -subj "/CN=httpbin.example.com/O=httpbin organization"

openssl x509 -req -sha256 -days 365 \
  -CA example.com.crt -CAkey example.com.key -set_serial 0 \
  -in httpbin.example.com.csr \
  -out httpbin.example.com.crt

cd ..
```

### 7.2 KV에 업로드 — 두 가지 방식

#### 방식 A: secret 객체로 업로드 (가장 단순)
```bash
az keyvault secret set --vault-name $AKV_NAME --name test-httpbin-key --file httpbin_certs/httpbin.example.com.key
az keyvault secret set --vault-name $AKV_NAME --name test-httpbin-crt --file httpbin_certs/httpbin.example.com.crt
```
→ SPC에서 `objectType: secret`로 참조. crt/key를 **각각 별개 secret으로** 보관.

#### 방식 B: certificate 객체로 업로드 (운영 권장)
PFX 통합본을 KV `certificate` 객체로 import (체인 검증/만료 알림/회전 메타데이터 등 KV 인증서 기능 활용 가능):
```bash
# PEM → PFX 변환 (운영에서 발급받은 PFX가 이미 있으면 그대로 사용)
openssl pkcs12 -export \
  -inkey httpbin_certs/httpbin.example.com.key \
  -in httpbin_certs/httpbin.example.com.crt \
  -out httpbin_certs/httpbin.example.com.pfx \
  -password pass:

az keyvault certificate import \
  --vault-name $AKV_NAME \
  --name test-httpbin-cert-pfx \
  --file httpbin_certs/httpbin.example.com.pfx \
  --password ""
```
→ SPC에서 `objectType: cert`(crt 추출) + `objectType: secret`(key 추출) 두 항목으로 참조.

---

## 8. SecretProviderClass (SPC) 작성

App namespace를 미리 생성:
```bash
kubectl create namespace $APP_NS --dry-run=client -o yaml | kubectl apply -f -
```

### 8.1 방식 A에 대응 (secret 객체 2개를 합쳐 TLS Secret 생성)
```yaml
cat <<EOF | kubectl apply -n $APP_NS -f -
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: httpbin-credential-spc
spec:
  provider: azure
  secretObjects:
  - secretName: httpbin-credential        # ← Gateway가 참조할 K8s Secret 이름
    type: kubernetes.io/tls
    data:
    - objectName: test-httpbin-key
      key: tls.key
    - objectName: test-httpbin-crt
      key: tls.crt
  parameters:
    useVMManagedIdentity: "true"
    userAssignedIdentityID: $CLIENT_ID
    keyvaultName: $AKV_NAME
    cloudName: ""
    objects: |
      array:
        - |
          objectName: test-httpbin-key
          objectType: secret
          objectAlias: "test-httpbin-key"
        - |
          objectName: test-httpbin-crt
          objectType: secret
          objectAlias: "test-httpbin-crt"
    tenantId: $TENANT_ID
EOF
```

### 8.2 방식 B에 대응 (KV cert 1개에서 crt/key 분리 추출)
```yaml
cat <<EOF | kubectl apply -n $APP_NS -f -
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: httpbin-credential-spc
spec:
  provider: azure
  secretObjects:
  - secretName: httpbin-credential
    type: kubernetes.io/tls
    data:
    - objectName: test-httpbin-key
      key: tls.key
    - objectName: test-httpbin-crt
      key: tls.crt
  parameters:
    useVMManagedIdentity: "true"
    userAssignedIdentityID: $CLIENT_ID
    keyvaultName: $AKV_NAME
    cloudName: ""
    objects: |
      array:
        - |
          objectName: test-httpbin-cert-pfx
          objectType: secret              # ← PFX 본문을 가져와 key 추출
          objectAlias: "test-httpbin-key"
        - |
          objectName: test-httpbin-cert-pfx
          objectType: cert                # ← 인증서(crt) 추출
          objectAlias: "test-httpbin-crt"
    tenantId: $TENANT_ID
EOF
```

> ⚠️ SPC에 `objectVersion`을 명시하지 않으면 **항상 최신 버전을 fetch** (회전 시 유리). 명시 시 그 버전에 고정.

---

## 9. 더미 워크로드로 fetch 트리거 (Secret 생성)

> 📌 **왜 Pod가 아니라 Deployment(replicas 2)인가**
> CSI Secrets Store driver 는 **현재 노드에 mount 된 SPC volume 만** 주기적으로 polling 해서 회전을 반영한다 (§12). Gateway/Envoy 는 Secret 을 API 로만 읽고 mount 하지 않으므로, polling 을 *살아있게* 해주는 mount 홀더가 필요하다 — 그게 더미 워크로드.
> - 단일 Pod 으로 두면 노드 재시작 · drain · evict · 자동 업그레이드 시 mount 가 끊겨 polling 도 중단된다 (단독 Pod 은 controller 가 없어 다시 안 살아남).
> - Deployment + replicas 2 + podAntiAffinity 로 두면 노드 단위 장애에도 최소 1개의 mount 가 살아있어 rotation 루프가 무중단.
> - 비용 부담은 거의 없다 (`pause` 컨테이너 메모리 수십 MB).

```bash
cat <<EOF | kubectl apply -n $APP_NS -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: secrets-store-sync-httpbin
  labels:
    app: secrets-store-sync-httpbin
spec:
  replicas: 2
  selector:
    matchLabels:
      app: secrets-store-sync-httpbin
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0     # 항상 1개 이상 마운트 유지 → rotation 끊김 방지
      maxSurge: 1
  template:
    metadata:
      labels:
        app: secrets-store-sync-httpbin
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  app: secrets-store-sync-httpbin
              topologyKey: kubernetes.io/hostname
      containers:
      - name: pause
        image: mcr.microsoft.com/oss/kubernetes/pause:3.6
        resources:
          requests: { cpu: "10m",  memory: "16Mi" }
          limits:   { cpu: "50m",  memory: "32Mi" }
        volumeMounts:
        - name: secrets-store01-inline
          mountPath: "/mnt/secrets-store"
          readOnly: true
      volumes:
      - name: secrets-store01-inline
        csi:
          driver: secrets-store.csi.k8s.io
          readOnly: true
          volumeAttributes:
            secretProviderClass: "httpbin-credential-spc"
EOF
```

### 9.1 Secret 및 Pod 생성 확인
```bash
kubectl -n $APP_NS get deploy secrets-store-sync-httpbin
kubectl -n $APP_NS get pod -l app=secrets-store-sync-httpbin -o wide   # 두 Pod 가 서로 다른 노드에 있는지 확인
kubectl -n $APP_NS describe secret httpbin-credential
# Type: kubernetes.io/tls, tls.crt/tls.key 두 항목 존재 확인
```

> ℹ️ **공식 문서는 `Pod` + `sleep 10` 예시**를 보여주는데, 이는 "Secret 만 한 번 만들면 끝"인 일회성 시나리오이다. Pod 가 종료되어도 `secretObjects`로 만들어진 K8s Secret 자체는 남으므로 Gateway 가 정지하지는 않지만, **그 시점부터 KV rotation 이 반영되지 않는다**. 운영 환경에서는 위와 같이 Deployment 형태로 항상 띄워 두는 것을 권장.

---

## 10. TLS Gateway + HTTPRoute

샘플 앱 `httpbin` 배포(이미 있다면 skip):
```bash
export ISTIO_RELEASE="release-1.27"
kubectl apply -n $APP_NS -f https://raw.githubusercontent.com/istio/istio/$ISTIO_RELEASE/samples/httpbin/httpbin.yaml
```

Gateway + HTTPRoute:
```bash
cat <<EOF | kubectl apply -n $APP_NS -f -
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: httpbin-gateway
spec:
  gatewayClassName: approuting-istio
  listeners:
  - name: https
    hostname: "httpbin.example.com"
    port: 443
    protocol: HTTPS
    tls:
      mode: Terminate
      certificateRefs:
      - name: httpbin-credential          # ← §9에서 생성된 Secret
    allowedRoutes:
      namespaces:
        from: Selector
        selector:
          matchLabels:
            kubernetes.io/metadata.name: $APP_NS
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
    - path: { type: PathPrefix, value: /status }
    - path: { type: PathPrefix, value: /delay }
    backendRefs:
    - name: httpbin
      port: 8000
EOF
```

> `tls.certificateRefs[].name`은 반드시 SPC의 `secretObjects[].secretName`과 동일해야 함.

---

## 11. 동작 확인

```bash
kubectl wait -n $APP_NS --for=condition=programmed gateways.gateway.networking.k8s.io httpbin-gateway

export INGRESS_HOST=$(kubectl get -n $APP_NS gateways.gateway.networking.k8s.io httpbin-gateway \
  -o jsonpath='{.status.addresses[0].value}')
export SECURE_INGRESS_PORT=$(kubectl get -n $APP_NS gateways.gateway.networking.k8s.io httpbin-gateway \
  -o jsonpath='{.spec.listeners[?(@.name=="https")].port}')

curl -v -HHost:httpbin.example.com \
  --resolve "httpbin.example.com:$SECURE_INGRESS_PORT:$INGRESS_HOST" \
  --cacert httpbin_certs/example.com.crt \
  "https://httpbin.example.com:$SECURE_INGRESS_PORT/status/418"
# HTTP/2 418   ← I'm a Teapot
```

---

## 12. 인증서 Rotation 운영

### 12.1 자동 rotation 활성화 (CSI driver add-on 옵션)
```bash
az aks update -g $RESOURCE_GROUP -n $CLUSTER \
  --enable-secret-rotation \
  --rotation-poll-interval 2m
```
→ 기본 polling 간격 2분 (조정 가능). 너무 짧게 잡으면 KV throttling 위험.

### 12.2 자동 rotation 전제 조건
- SPC에 `objectVersion` **비워두기** (§8 참고)
- **더미 워크로드(§9 Deployment)가 살아있어야 함** — mount된 상태에서만 CSI가 polling 수행. replicas≥2 + `maxUnavailable: 0` 로 두면 노드 drain/롤링 중에도 polling 무중단
- KV의 `--enable-secret-rotation` 옵션은 클러스터 차원에서 활성화

### 12.3 새 버전 업로드
```bash
# secret 방식
az keyvault secret set --vault-name $AKV_NAME --name test-httpbin-crt --file new-cert.crt
# certificate 방식
az keyvault certificate import --vault-name $AKV_NAME --name test-httpbin-cert-pfx --file new.pfx
```
→ 다음 polling cycle에 더미 Pod들의 mount path가 갱신되고, Secret도 동시에 update됨.

### 12.4 검증 (fingerprint 비교)
```bash
# KV 측 fingerprint
az keyvault certificate show --vault-name $AKV_NAME --name test-httpbin-cert-pfx \
  --query 'x509ThumbprintHex' -o tsv

# 클러스터 Secret 측 fingerprint
kubectl -n $APP_NS get secret httpbin-credential -o jsonpath='{.data.tls\.crt}' \
  | base64 -d | openssl x509 -noout -fingerprint -sha1
```
두 값이 동일하면 rotation 정상.

### 12.5 Envoy 측 즉시 반영 확인
```bash
# Gateway Pod 액세스 로그에서 새 인증서 적용 여부 (TLS handshake)
kubectl logs -n $APP_NS <gateway-pod-name> --tail=20
```

---

## 13. 멀티 namespace 시나리오

App Routing은 **ns 별 Gateway** 모델이므로, 인증서를 사용하는 ns가 늘면 SPC도 ns 별로 필요 → 더미 Pod도 ns 별로 필요해진다. 회피 패턴:

### 13.1 ReferenceGrant + 집중 cert-store ns (권장)

#### 왜 ReferenceGrant 가 필수인가
Gateway API 는 **다른 ns 의 리소스 참조를 기본적으로 거부**(fail-closed)한다. ns A 의 운영자가 임의로 ns B 의 Secret(인증서/개인키)을 listener 에 묶어 노출시키지 못하도록, **참조 대상 ns 의 소유자가 명시적으로 허용**해야 하며 이 허용 선언이 `ReferenceGrant` 다.

ReferenceGrant 가 없을 때 동작:
- Gateway controller 가 cross-ns 참조 발견 → 거부
- Gateway status condition 에 명시적 에러:
  ```
  type: ResolvedRefs   status: False   reason: RefNotPermitted
  type: Programmed     status: False
  ```
- Envoy 에 해당 listener config **푸시 안 됨** → 443 포트는 떠 있어도 TLS handshake 실패. Gateway/Pod 가 죽지는 않고 **그 listener 만 동작 불가**.
- 같은 ns 안에서의 참조 / 같은 Gateway 의 다른 listener 는 영향 없음.

확인:
```bash
kubectl describe gateway <gw> -n <app-ns>
kubectl get gateway <gw> -n <app-ns> \
  -o jsonpath='{.status.listeners[*].conditions[?(@.type=="ResolvedRefs")]}'
```

#### 권장 구성
- `cert-store` 라는 ns 1개 생성 → SPC, 더미 Deployment, Secret 모두 여기 1세트로 집중
- 다른 ns 의 Gateway 가 cert-store ns 의 Secret 을 참조할 수 있도록 **ReferenceGrant** 부여
- KV 1 + 인증서 N + 멀티 ns 시 가장 단순한 패턴 — SPC 1개(인증서 N 묶음) + Deployment 1세트(replicas 2) + Secret N개 + ReferenceGrant N(또는 ns 수만큼)

```yaml
# cert-store ns 쪽
apiVersion: gateway.networking.k8s.io/v1beta1
kind: ReferenceGrant
metadata:
  name: gw-can-read-tls-secret
  namespace: cert-store
spec:
  from:
  - group: gateway.networking.k8s.io
    kind: Gateway
    namespace: tenant-a
  - group: gateway.networking.k8s.io
    kind: Gateway
    namespace: tenant-b
  to:
  - group: ""
    kind: Secret
    name: httpbin-credential
```

각 ns의 Gateway는 cross-ns 참조:
```yaml
spec:
  listeners:
  - name: https
    tls:
      mode: Terminate
      certificateRefs:
      - kind: Secret
        namespace: cert-store     # ← 다른 ns 참조
        name: httpbin-credential
```

→ 더미 Deployment **1세트(replicas 2)** 로 모든 ns의 Gateway가 같은 인증서를 공유.

### 13.2 단일 SPC 에 인증서 여러 개 묶기 (KV 1개 + 인증서 N개)

같은 KV 에 있는 여러 인증서는 **SPC 1개**로 한꺼번에 처리할 수 있다 (`objects` / `secretObjects` 모두 배열). 인증서가 늘어도 SPC · 더미 Deployment · ReferenceGrant 수는 그대로 1세트.

#### SPC — 인증서 2개 묶음 예시 (KV cert / 방식 B 기준)

KV 에 PFX 로 import 한 인증서(`tenant-a-cert-pfx`, `tenant-b-cert-pfx`) 2 개가 있는 상황. 각각 `objectType: secret`(PFX 본문에서 key 추출) + `objectType: cert`(crt 추출) 두 항목으로 분해해 K8s TLS Secret 으로 합친다.

```yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: all-tls-spc
  namespace: cert-store
spec:
  provider: azure
  parameters:
    useVMManagedIdentity: "true"
    userAssignedIdentityID: $KV_PROVIDER_CLIENT_ID   # §6 에서 확인한 KV CSI provider MI
    keyvaultName: $AKV_NAME
    tenantId: $TENANT_ID
    cloudName: ""
    objects: |
      array:
        # ── 인증서 A (KV cert: tenant-a-cert-pfx)
        - |
          objectName: tenant-a-cert-pfx
          objectType: secret              # PFX 본문 → key 추출
          objectAlias: "tenant-a-key"
        - |
          objectName: tenant-a-cert-pfx
          objectType: cert                # 인증서(crt) 추출
          objectAlias: "tenant-a-crt"
        # ── 인증서 B (KV cert: tenant-b-cert-pfx)
        - |
          objectName: tenant-b-cert-pfx
          objectType: secret
          objectAlias: "tenant-b-key"
        - |
          objectName: tenant-b-cert-pfx
          objectType: cert
          objectAlias: "tenant-b-crt"
  secretObjects:
    - secretName: tls-tenant-a            # ← K8s Secret #1 (kubernetes.io/tls)
      type: kubernetes.io/tls
      data:
        - objectName: tenant-a-crt
          key: tls.crt
        - objectName: tenant-a-key
          key: tls.key
    - secretName: tls-tenant-b            # ← K8s Secret #2
      type: kubernetes.io/tls
      data:
        - objectName: tenant-b-crt
          key: tls.crt
        - objectName: tenant-b-key
          key: tls.key
```
→ `cert-store` ns 에 `tls-tenant-a`, `tls-tenant-b` 두 K8s Secret 자동 생성.

> ℹ️ `objectType: cert` 도 내부적으로 KV **secret API** 를 호출하므로 KV CSI provider MI 에는 `Key Vault Secrets User` 권한이 있어야 한다 (§6.2 참고).

#### 더미 Deployment — volume 도 1개로 충분
SPC 가 1개이므로 §9 의 Deployment 를 그대로 쓰되 `secretProviderClass` 만 `all-tls-spc` 로 바꾸면 끝 (volume / volumeMount 도 1개만):
```yaml
volumes:
- name: secrets-store
  csi:
    driver: secrets-store.csi.k8s.io
    readOnly: true
    volumeAttributes:
      secretProviderClass: "all-tls-spc"   # ← 단일 SPC
```

#### ReferenceGrant — 테넌트 격리는 쌍 별로 분리

> ⚠️ ReferenceGrant 는 `from[] × to[]` **카테시안 곱**으로 허용한다. 한 ReferenceGrant 에 `from: [tenant-a, tenant-b]` + `to: [tls-tenant-a, tls-tenant-b]` 를 같이 적으면 tenant-a 가 `tls-tenant-b` 도 참조할 수 있게 된다.
> 테넌트 별로 자기 인증서만 참조하도록 제한하려면 **(from-ns, to-secret) 쌍 별 ReferenceGrant 1 개씩** 만든다.

```yaml
# tenant-a 만 tls-tenant-a 참조 허용
apiVersion: gateway.networking.k8s.io/v1beta1
kind: ReferenceGrant
metadata:
  name: gw-tenant-a-can-read-tls-tenant-a
  namespace: cert-store
spec:
  from:
  - group: gateway.networking.k8s.io
    kind: Gateway
    namespace: tenant-a
  to:
  - group: ""
    kind: Secret
    name: tls-tenant-a
---
# tenant-b 만 tls-tenant-b 참조 허용
apiVersion: gateway.networking.k8s.io/v1beta1
kind: ReferenceGrant
metadata:
  name: gw-tenant-b-can-read-tls-tenant-b
  namespace: cert-store
spec:
  from:
  - group: gateway.networking.k8s.io
    kind: Gateway
    namespace: tenant-b
  to:
  - group: ""
    kind: Secret
    name: tls-tenant-b
```

> 💡 격리가 필요 없고 모든 테넌트가 모든 인증서를 자유롭게 참조해도 무방하다면(예: 단일 팀 운영) 위처럼 쌍 별로 나누지 않고 ReferenceGrant 1 개에 `from`/`to` 를 모두 나열해도 된다 — 운영 단순함과 격리 사이의 트레이드오프.

#### 각 ns 의 Gateway 에서 참조
```yaml
# tenant-a ns
spec:
  listeners:
  - name: https
    tls:
      mode: Terminate
      certificateRefs:
      - kind: Secret
        namespace: cert-store
        name: tls-tenant-a        # ← SPC 가 만들어준 Secret 중 A

# tenant-b ns
spec:
  listeners:
  - name: https
    tls:
      mode: Terminate
      certificateRefs:
      - kind: Secret
        namespace: cert-store
        name: tls-tenant-b        # ← Secret B
```

> 💡 인증서가 늘어나면 SPC 의 `objects` / `secretObjects` 배열에 항목을 추가하고, **테넌트 격리가 필요하면 (from-ns, to-secret) 쌍 별 ReferenceGrant 도 1개씩 추가**. Deployment 는 손댈 필요 없음.

> ⚠️ **언제 분리할까** — ① 인증서가 서로 **다른 KV** 에 있으면 SPC 도 분리 (KV name 이 SPC 단위 속성) / ② 인증서 수십~수백개로 커지면 한 번의 poll 에 KV 호출 폭주 → throttling 위험으로 SPC 분리 검토 / ③ 인증서 ownership·변경주기 차이가 크면 blast radius 줄이려고 분리.

### 13.3 인증서마다 SPC 가 분리되어 있는 경우

(예: KV 가 여러 개라 한 SPC 로 묶을 수 없는 경우)
1 개 Deployment 의 Pod 템플릿에 SPC 별 volume 을 동시 마운트:
```yaml
volumes:
- name: cert-a
  csi: { driver: secrets-store.csi.k8s.io, readOnly: true, volumeAttributes: { secretProviderClass: spc-cert-a } }
- name: cert-b
  csi: { driver: secrets-store.csi.k8s.io, readOnly: true, volumeAttributes: { secretProviderClass: spc-cert-b } }
```
→ SPC 가 N개라도 더미 Deployment 는 여전히 1세트로 모두 처리 가능.

---

## 14. 트러블슈팅

| 증상 | 원인 | 조치 |
|---|---|---|
| `Secret 'xxx' not found on the cluster` (Gateway status) | 더미 워크로드 미배포 → SPC가 fetch 되지 않음 | §9 Deployment 배포 |
| `MountVolume.SetUp failed ... permission denied` (더미 Pod) | KV CSI Provider MI에 KV 권한 없음 | §6 RBAC role 부여 + 토큰 반영 1-3분 대기 |
| Secret은 있으나 Gateway가 `not accepted` | `certificateRefs[].name` ≠ `secretObjects[].secretName` | 두 이름 일치 확인 |
| 인증서 회전했는데 Envoy가 옛것 사용 | `--enable-secret-rotation` 미활성 / 더미 Deployment Pod 0개 / `objectVersion` 고정 | §12 조건 모두 충족 확인 |
| 한쪽 노드 drain 시 회전 잠시 멈춤 | 더미 replicas=1 | replicas≥2 + podAntiAffinity (§9) |
| `objectType: cert` 사용 시 권한 오류 | `Key Vault Secrets User` 미부여 (cert도 secret API 사용) | §6.2 ⚠️ 참고 |
| 다른 ns에서 Secret 못 읽음 | ReferenceGrant 없음 | §13.1 |

### 14.1 관련 공개 이슈
- [Azure/AKS#5312 — Secret sync without dummy pod](https://github.com/Azure/AKS/issues/5312): 8개월 stalled. 컨트롤러 내장 요청은 로드맵 미공개 상태로 봄.

---

## 15. 운영 체크리스트

- [ ] KV는 `--enable-rbac-authorization true` + `--enable-purge-protection true`
- [ ] KV CSI Provider add-on의 MI에 **Key Vault Secrets User** role 부여
- [ ] `--enable-secret-rotation`로 회전 자동화 + SPC `objectVersion` **공란**
- [ ] 더미 워크로드는 **Deployment + replicas 2 + podAntiAffinity + maxUnavailable 0** (§9) — 단일 Pod 금지
- [ ] 멀티 ns 환경은 **cert-store ns + ReferenceGrant** 패턴
- [ ] 회전 검증을 위한 thumbprint 비교 스크립트 사전 작성
- [ ] 만료 알림 — KV의 Event Grid → Webhook/Logic App 권장

---

## 부록 A. cert-manager 대안 (KV 미사용)

KV가 SoT가 아니어도 되면 cert-manager + Azure DNS 솔버로 발급/회전 자동화 가능. Gateway API의 [GatewayHTTPRoute Solver](https://cert-manager.io/docs/usage/gateway/)를 사용. 본 가이드 범위 외이며 별도 구성.

## 부록 B. 정리

```bash
kubectl delete -n $APP_NS gateways.gateway.networking.k8s.io httpbin-gateway
kubectl delete -n $APP_NS httproute httpbin
kubectl delete -n $APP_NS pod secrets-store-sync-httpbin
kubectl delete -n $APP_NS secretproviderclass httpbin-credential-spc
kubectl delete -n $APP_NS secret httpbin-credential
# KV 자체
az keyvault delete --name $AKV_NAME --resource-group $RESOURCE_GROUP
az keyvault purge --name $AKV_NAME --location $LOCATION   # purge protection이 켜져 있으면 retention 후
```
