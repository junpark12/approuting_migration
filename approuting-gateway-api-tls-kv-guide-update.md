# App Routing Gateway API — Azure Key Vault TLS 연동 가이드 (Operator 턴키 방식, GA)

> 본 문서는 [`approuting-gateway-api-tls-kv-guide.md`](./approuting-gateway-api-tls-kv-guide.md)(수동 방식)의 **개정판**이다.
> 기존 문서는 *관리자가 SecretProviderClass·더미 Pod·certificateRefs 를 직접 구성*하는 수동 체인을 다뤘다.
> 본 문서는 **Application Routing operator** 가 그 체인을 자동으로 reconcile 하는 **턴키 방식**을 정리한다 (더미 Pod 불필요).
>
> 공식 출처: [Configure Azure DNS and TLS with the Application Routing Gateway API Implementation](https://learn.microsoft.com/en-us/azure/aks/app-routing-gateway-api-dns-tls#configure-tls-termination-on-a-gateway)
>
> ⚠️ 본 문서는 **TLS 종료(Key Vault 인증서 동기화)** 만 다룬다. DNS 자동화(`ClusterExternalDNS`/`ExternalDNS`)는 범위에서 제외한다.

---

## 0. 무엇이 바뀌었나 (수동 → 자동)

기존 수동 가이드의 핵심 전제는 *"Gateway API GA 에는 KV→Secret 턴키 컨트롤러가 없다 → 더미 Pod 으로 CSI polling 을 살려둬야 한다"* 였다.

**이제는 그 턴키 컨트롤러가 제공된다.** `--enable-app-routing` 으로 배포되는 **Application Routing operator** 가 listener 의 TLS option 두 줄을 보고 SecretProviderClass·Secret·certificateRefs 를 **자동으로 reconcile** 한다. 따라서 **더미 Pod 이 구조적으로 불필요**해졌다.

### 관리자가 직접 하던 일 vs operator 가 자동화하는 일

| 단계 | 기존 수동 방식 (구 문서 §8~10) | Operator 턴키 방식 (본 문서) |
|---|---|---|
| SecretProviderClass 작성 | 관리자가 직접 YAML 작성 (`useVMManagedIdentity`) | **operator 가 자동 생성** (`kv-gw-cert-<gw>-<listener>`, Workload Identity) |
| KV→Secret 동기화 트리거 | **더미 Pod(Deployment, replicas 2)** 으로 SPC mount → CSI polling 유지 | **operator 가 트리거** → 더미 Pod **불필요** |
| Kubernetes TLS Secret 생성 | 더미 Pod 이 mount 할 때 `secretObjects` 로 생성 | operator 가 관장, CSI Driver 가 동기화 |
| listener `tls.certificateRefs` 연결 | 관리자가 Secret 이름을 수동 지정 | **operator 가 자동 패치** |
| 인증 모델 | KV CSI Provider add-on 의 **자동생성 MI** (`useVMManagedIdentity: "true"`) | 사용자 생성 **UAMI + Workload Identity(FIC + ServiceAccount)** |
| 인증서 회전(rotation) | 더미 Pod 상주 + `--enable-secret-rotation` 필요 | operator + unversioned KV URI 로 자동 반영 |

> 💡 **핵심 차이 한 줄**: 기존엔 listener 에 `certificateRefs`(이미 존재하는 Secret 이름)를 직접 적었지만, 턴키 방식은 listener 의 `tls.options` 에 **KV URI + ServiceAccount 두 줄**만 적으면 operator 가 나머지를 만든다.

---

## 1. Operator 가 하는 일 (TLS integration 동작 원리)

`Gateway` 가 `approuting-istio` GatewayClass 를 쓰고, listener 가 아래 **두 개의 TLS option** 을 가지면 Application Routing operator 가 동작한다.

| TLS option key | 값 |
|---|---|
| `kubernetes.azure.com/tls-cert-keyvault-uri` | TLS 인증서를 가져올 Azure Key Vault 인증서 URI. **unversioned URI**(예: `https://<vault>.vault.azure.net/certificates/<cert>`)를 쓰면 operator 가 KV 회전을 자동으로 따라간다. |
| `kubernetes.azure.com/tls-cert-service-account` | `Gateway` 와 같은 namespace 의 Kubernetes ServiceAccount 이름. 이 SA 는 Workload Identity 로 UAMI 에 바인딩되어야 하고, 그 UAMI 는 대상 KV 에 **`Key Vault Secrets User`** role 을 가져야 한다. |

operator 는 두 option 을 가진 listener 마다 다음을 수행한다:

1. **`SecretProviderClass` 자동 생성** — `kv-gw-cert-<gateway-name>-<listener-name>` 이름으로 Gateway namespace 에 생성 (Workload Identity 인증 설정 포함).
2. **CSI Driver 가 인증서를 동기화하도록 트리거** — KV 인증서를 동일 이름의 `kubernetes.io/tls` Secret 으로 sync.
3. **listener 의 `tls.certificateRefs` 자동 패치** — 동기화된 Secret 을 참조하도록 연결.

> ℹ️ 동기화 자체는 여전히 **Azure Key Vault provider for Secrets Store CSI Driver** 가 수행한다. operator 는 그 위에서 SPC 생성·트리거·certificateRefs 패치를 **자동화**할 뿐이다. (그래서 KV CSI add-on 은 여전히 prerequisite)

---

## 2. 사전 요구사항

KV TLS 턴키를 쓰려면 클러스터에 아래가 모두 활성화되어야 한다.

- **Application Routing add-on** (`--enable-app-routing`)
  → DNS/TLS 통합을 reconcile 하는 **Application Routing operator** 를 배포한다. 이 플래그가 NGINX 시절 이름이라 오해를 주지만, **이 기능에 필요한 건 operator** 다.
  기존 클러스터: `az aks approuting enable`
- **Application Routing Gateway API 구현** (`--enable-app-routing-istio`)
  → Gateway API 컨트롤플레인(`approuting-istio` GatewayClass) 활성화.
  기존 클러스터: `az aks approuting gateway istio enable`
  > ⚠️ 두 플래그 **모두** 필요. 하나만 켜면 통합이 동작하지 않는다.
- **Managed Gateway API installation** 활성화.
- **Microsoft Entra Workload Identity + OIDC issuer** (`--enable-oidc-issuer --enable-workload-identity`).
- **Azure Key Vault provider for Secrets Store CSI Driver** add-on (`--addons azure-keyvault-secrets-provider`, 또는 `az aks approuting enable --enable-kv`).
- **Azure CLI 2.86.0 이상** (`az --version` 확인 후 `az upgrade`).
- 본인 identity 에 role assignment / FIC 생성 권한 (`Owner`, 또는 `Role Based Access Control Administrator` + `Managed Identity Contributor`).

> 📌 **기존 NGINX(`--enable-app-routing`) 사용 고객**은 operator 가 이미 떠 있으므로 `az aks approuting gateway istio enable` 로 **istio 만 추가**하면 된다. 기존 NGINX Ingress 는 그대로 병행 동작한다. 단, **Workload Identity/OIDC** 는 신규 인증 모델이라 별도로 켜야 할 수 있다.

> ⚠️ `az aks approuting update` 의 `--attach-kv` / `--attach-zones` 플래그는 **레거시 NGINX 경험용**(add-on 자체 MI 에 KV/DNS 권한 부여)이다. **Gateway API 통합은 이를 사용하지 않으며**, 대신 사용자 UAMI + Workload Identity + FIC 로 동작한다.

환경 변수:

```bash
export RESOURCE_GROUP=<resource-group-name>
export CLUSTER=<cluster-name>
export LOCATION=<azure-region>
```

`kubectl` 자격 증명:

```azurecli
az aks get-credentials --resource-group $RESOURCE_GROUP --name $CLUSTER
```

---

## 3. Azure Key Vault 와 인증서 생성

KV 생성 (RBAC 권한 모델 — 권장):

```azurecli
export KV_NAME=<key-vault-name>
az keyvault create \
  --name $KV_NAME \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION \
  --enable-rbac-authorization true
```

> 다음 단계에서 인증서를 만들려면 본인 identity 에 KV 의 **`Key Vault Certificates Officer`**(또는 `Key Vault Administrator`) role 이 있어야 한다. 진행 전에 부여할 것.

self-signed 와일드카드 인증서 생성 (운영 환경은 `az keyvault certificate import` 로 CA 서명 인증서 import):

```azurecli
cat > cert-policy.json <<EOF
{
  "issuerParameters": { "name": "Self" },
  "keyProperties": { "exportable": true, "keyType": "RSA", "keySize": 2048, "reuseKey": false },
  "secretProperties": { "contentType": "application/x-pkcs12" },
  "x509CertificateProperties": {
    "subject": "CN=*.example.com",
    "subjectAlternativeNames": { "dnsNames": ["*.example.com", "example.com"] },
    "validityInMonths": 12,
    "keyUsage": ["digitalSignature", "keyEncipherment"]
  }
}
EOF

az keyvault certificate create \
  --vault-name $KV_NAME \
  --name approuting-demo-cert \
  --policy @cert-policy.json
```

**unversioned 인증서 URI** 캡처 (operator 가 이 URI 로 SPC 를 구성. unversioned 라야 회전 자동 반영):

```azurecli
export CERT_URI=$(az keyvault certificate show \
  --vault-name $KV_NAME \
  --name approuting-demo-cert \
  --query id -o tsv | sed 's|/[^/]*$||')
echo "Cert URI: $CERT_URI"
```

---

## 4. User-Assigned Managed Identity 생성 및 RBAC 부여

listener TLS sync 가 KV 인증에 사용할 UAMI 생성:

```azurecli
export UAMI_NAME=<managed-identity-name>
az identity create --resource-group $RESOURCE_GROUP --name $UAMI_NAME --location $LOCATION
export UAMI_CLIENT_ID=$(az identity show --resource-group $RESOURCE_GROUP --name $UAMI_NAME --query clientId -o tsv)
export UAMI_PRINCIPAL_ID=$(az identity show --resource-group $RESOURCE_GROUP --name $UAMI_NAME --query principalId -o tsv)
```

UAMI 에 KV `Key Vault Secrets User` role 부여:

```azurecli
az role assignment create \
  --assignee-object-id $UAMI_PRINCIPAL_ID \
  --assignee-principal-type ServicePrincipal \
  --role "Key Vault Secrets User" \
  --scope $(az keyvault show --name $KV_NAME --query id -o tsv)
```

> ⚠️ KV 인증서를 `cert` 로 import 했더라도 CSI Driver 는 내부적으로 **secret API** 로 `.pfx`/`.pem` 본문을 가져오므로 **`Key Vault Secrets User`** 가 필수다.

---

## 5. ServiceAccount + Federated Identity Credential(FIC) 생성

operator 의 TLS 통합은 **ServiceAccount ↔ UAMI** 를 FIC 로 바인딩해 Azure 에 인증한다. **`(namespace, ServiceAccount)` 쌍마다 FIC 1개**가 필요하다.

OIDC issuer URL 캡처:

```azurecli
export OIDC_ISSUER=$(az aks show --resource-group $RESOURCE_GROUP --name $CLUSTER --query oidcIssuerProfile.issuerUrl -o tsv)
```

TLS 통합을 쓸 `Gateway` 가 들어갈 각 namespace 에 대해 namespace + FIC + ServiceAccount 생성. 아래 예시는 `app-a`, `app-b` 두 namespace 에 `approuting-demo-sa` SA 를 만든다:

```azurecli
export SA_NAME=approuting-demo-sa
for ns in app-a app-b; do
  kubectl create namespace $ns

  az identity federated-credential create \
    --identity-name $UAMI_NAME \
    --resource-group $RESOURCE_GROUP \
    --name approuting-demo-fic-$ns \
    --issuer $OIDC_ISSUER \
    --subject "system:serviceaccount:$ns:$SA_NAME" \
    --audiences "api://AzureADTokenExchange"

  kubectl apply -n $ns -f - <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: $SA_NAME
  annotations:
    azure.workload.identity/client-id: $UAMI_CLIENT_ID
  labels:
    azure.workload.identity/use: "true"
EOF
done
```

> `azure.workload.identity/client-id` annotation 이 SA 를 UAMI 와 연결하고, `azure.workload.identity/use: "true"` label 이 Workload Identity webhook 으로 하여금 federated token 을 pod 에 주입하게 한다. **둘 다 필수**.

---

## 6. Gateway 에서 TLS 종료 구성 (핵심)

샘플 `httpbin` 워크로드 배포:

```bash
for ns in app-a app-b; do
  kubectl apply -n $ns -f https://raw.githubusercontent.com/istio/istio/release-1.27/samples/httpbin/httpbin.yaml
done
```

각 namespace 에 HTTPS listener 를 가진 `Gateway` 생성. **listener `tls.options` 에 KV URI + SA 두 줄**만 넣는다 (SecretProviderClass·Secret·certificateRefs 는 operator 가 자동 생성):

```bash
for pair in "app-a:a" "app-b:b"; do
  ns=${pair%%:*}
  sub=${pair##*:}
  fqdn=${sub}.example.com
  kubectl apply -n $ns -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: ${sub}-gateway
  labels:
    app: approuting-demo
    zone: ${sub}
spec:
  gatewayClassName: approuting-istio
  listeners:
  - name: https
    hostname: $fqdn
    port: 443
    protocol: HTTPS
    tls:
      mode: Terminate
      options:
        kubernetes.azure.com/tls-cert-keyvault-uri: $CERT_URI
        kubernetes.azure.com/tls-cert-service-account: $SA_NAME
    allowedRoutes:
      namespaces:
        from: Same
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: ${sub}-route
spec:
  parentRefs:
  - name: ${sub}-gateway
  hostnames: ["$fqdn"]
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /get
    backendRefs:
    - name: httpbin
      port: 8000
EOF
done
```

> 💡 구 문서(수동)와의 차이: 수동 방식은 listener 에 `tls.certificateRefs: [{ name: <기존 Secret> }]` 를 직접 적고 그 Secret 을 더미 Pod 으로 미리 만들어야 했다. 턴키 방식은 `tls.options` 두 줄로 끝이며 operator 가 certificateRefs 까지 채운다.

각 `Gateway` 가 `Programmed` 조건에 도달할 때까지 대기:

```bash
kubectl wait -n app-a --for=condition=programmed gateway a-gateway --timeout=300s
kubectl wait -n app-b --for=condition=programmed gateway b-gateway --timeout=300s
```

---

## 7. 동작 확인

operator 가 `SecretProviderClass` 를 만들고 CSI Driver 가 `kubernetes.io/tls` Secret 을 동기화했는지 확인:

```bash
kubectl get secretproviderclass,secret -n app-a
kubectl get secretproviderclass,secret -n app-b
```

예시 출력 (한 namespace):

```output
NAME                                                                        AGE
secretproviderclass.secrets-store.csi.x-k8s.io/kv-gw-cert-a-gateway-https   2m

NAME                                TYPE                DATA   AGE
secret/kv-gw-cert-a-gateway-https   kubernetes.io/tls   2      2m
```

> SPC/Secret 이름은 `kv-gw-cert-<gateway>-<listener>` 패턴으로 **operator 가 자동 명명**한다 (사용자가 정하지 않음).

### TLS 종료된 HTTPS 트래픽 검증

Gateway 의 외부 IP 로 직접 요청 (DNS 위임 전 테스트는 `curl --resolve` 로 로컬 DNS 우회):

```bash
GATEWAY_IP=$(kubectl get -n app-a gateway a-gateway -o jsonpath='{.status.addresses[0].value}')
curl -k -I --resolve "a.example.com:443:${GATEWAY_IP}" "https://a.example.com/get"
```

`HTTP/2 200` 응답이 보이면 성공이며, gateway 가 제시하는 인증서는 KV 에서 동기화된 인증서다. CA 서명 인증서를 import 했다면 `-k` 대신 `--cacert <ca-chain-경로>` 로 체인 검증.

---

## 8. 인증서 회전(Rotation)

- KV 인증서 URI 를 **unversioned** 로 지정(§3)하면 operator 가 새 버전을 자동으로 따라간다.
- CSI Driver 의 자동 회전을 켜려면(권장):

```azurecli
az aks update -g $RESOURCE_GROUP -n $CLUSTER \
  --enable-secret-rotation \
  --rotation-poll-interval 2m
```

- 새 인증서 버전 업로드:

```azurecli
az keyvault certificate create \
  --vault-name $KV_NAME \
  --name approuting-demo-cert \
  --policy @cert-policy.json
```

다음 polling cycle 에 Secret 이 갱신되고 Envoy 에 반영된다.

### 회전 검증 (thumbprint 비교)

```bash
# KV 측 thumbprint
az keyvault certificate show --vault-name $KV_NAME --name approuting-demo-cert \
  --query 'x509ThumbprintHex' -o tsv

# 클러스터 Secret 측 fingerprint
kubectl -n app-a get secret kv-gw-cert-a-gateway-https -o jsonpath='{.data.tls\.crt}' \
  | base64 -d | openssl x509 -noout -fingerprint -sha1
```

두 값이 일치하면 회전 정상.

---

## 9. 트러블슈팅

| 증상 | 원인 | 조치 |
|---|---|---|
| `Gateway` 가 `Programmed: False` / Secret 미생성 | TLS option 오타, operator 미배포 | `--enable-app-routing`(operator) 활성 확인, option 키 2개 정확히 확인 |
| `MountVolume.SetUp failed ... permission denied` 류 | UAMI 에 KV 권한 없음 | `Key Vault Secrets User` 부여(§4) + 토큰 반영 1~3분 대기 |
| `SecretProviderClass` 는 있는데 Secret 없음 | FIC/SA 바인딩 누락 | §5 의 FIC + SA annotation/label 확인 |
| 인증서 회전했는데 옛 인증서 사용 | `--enable-secret-rotation` 미활성 / KV URI 가 versioned | `--enable-secret-rotation` + unversioned URI 확인 |
| 다른 GatewayClass 에서 동작 안 함 | TLS 통합은 `approuting-istio` 전용 | GatewayClass 를 `approuting-istio` 로 |

---

## 10. 제약 사항 (공식)

- TLS 통합은 `gatewayClassName: approuting-istio` 인 `Gateway` 에만 적용된다. Istio service mesh add-on 의 GatewayClass 등 다른 GatewayClass 는 아직 미지원.

---

## 부록. 기존 수동 방식이 여전히 유효한 경우

operator 턴키 방식이 기본 권장 경로지만, 아래 시나리오는 기존 수동 가이드([`approuting-gateway-api-tls-kv-guide.md`](./approuting-gateway-api-tls-kv-guide.md))가 여전히 참고 가치가 있다:

- **멀티 namespace 인증서 공유 + 테넌트 격리** — `cert-store` ns + `ReferenceGrant` 패턴, ReferenceGrant 의 `from × to` 카테시안 곱 주의 등.
- **operator 를 켤 수 없는 제약 환경** — 더미 Pod(Deployment replicas 2 + podAntiAffinity) 기반 수동 동기화.
- **동작 원리 학습** — Envoy 가 Secret 을 mount 하지 않고 API 로 읽는 구조, CSI polling/mount 메커니즘 이해.
