# App Routing (NGINX) Internal Ingress 구성 - akskor

## 목적

고객사 환경(App Gateway + App Routing NGINX Internal)과 유사한 구성을 만들기 위해,
akskor 클러스터에서 App Routing add-on의 NGINX Ingress를 **Internal LoadBalancer**로 전환하고
샘플 앱을 배포하여 Ingress를 통한 접근을 확인한다.

> 이후 Gateway API (Istio) 마이그레이션 테스트의 기준 환경으로 사용 예정

## 현재 상태

| 항목 | 값 |
|---|---|
| 클러스터 | akskor (AKSKorea / koreacentral) |
| 노드풀 | aksnew - 3노드 (Standard_D2as_v5) |
| 네트워크 | Azure CNI + Cilium, Pod CIDR 192.168.0.0/16 |
| App Routing | enabled (webAppRouting) |
| NGINX 상태 | default NginxIngressController, **Public** LB (20.249.206.59) |
| Ingress 리소스 | 없음 |
| Gateway API | 미설치 |

---

## Step 1. Internal NGINX Ingress Controller 생성

기존 default NGINX는 Public LB로 동작 중이다.
Internal용 NginxIngressController를 **별도로** 생성한다.

> NginxIngressController CRD를 사용하면 App Routing add-on이 관리하는 NGINX 인스턴스를
> 추가로 만들 수 있고, loadBalancerAnnotations로 Internal LB 설정이 가능하다.

```yaml
# internal-nginx-controller.yaml
apiVersion: approuting.kubernetes.azure.com/v1alpha1
kind: NginxIngressController
metadata:
  name: nginx-internal
spec:
  ingressClassName: nginx-internal
  controllerNamePrefix: nginx-internal
  loadBalancerAnnotations:
    service.beta.kubernetes.io/azure-load-balancer-internal: "true"
```

```bash
kubectl apply -f internal-nginx-controller.yaml
```

### 확인

```bash
kubectl get ingressclass
kubectl get svc -n app-routing-system
kubectl get pods -n app-routing-system
```

### 실행 결과 (2026-03-19)

**IngressClass:**
```
NAME                                 CONTROLLER                                       PARAMETERS   AGE
nginx-internal                       approuting.kubernetes.azure.com/nginx-internal   <none>       17s
webapprouting.kubernetes.azure.com   webapprouting.kubernetes.azure.com/nginx         <none>       356d
```
> 참고: 기존 default는 초기 App Routing 네이밍(`webapprouting.kubernetes.azure.com`)이 적용되어 있음.
> 신규는 `approuting.kubernetes.azure.com`으로 변경됨. 동작에는 차이 없음.

**Service (Internal LB IP 할당 확인):**
```
NAME                       TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)                      AGE
nginx                      LoadBalancer   10.0.193.23    20.249.206.59   80:30840/TCP,443:30165/TCP   356d
nginx-internal-0           LoadBalancer   10.0.137.44    10.224.0.7      80:31703/TCP,443:32469/TCP   3m28s
nginx-internal-0-metrics   ClusterIP      10.0.221.160   <none>          10254/TCP                    3m28s
nginx-metrics              ClusterIP      10.0.241.194   <none>          10254/TCP                    356d
```
> ✅ `nginx-internal-0`이 Internal IP `10.224.0.7` (노드 서브넷 대역)을 할당받음

**Pods:**
```
NAME                                READY   STATUS    RESTARTS   AGE
nginx-54b5b9bd7c-b5x74              1/1     Running   0          26m
nginx-54b5b9bd7c-hs8c8              1/1     Running   0          29m
nginx-internal-0-6f874794b7-2zv9h   1/1     Running   0          4m50s
nginx-internal-0-6f874794b7-wzt6z   1/1     Running   0          5m5s
```
> ✅ Internal NGINX pod 2개 정상 Running

---

## Step 2. 샘플 애플리케이션 배포

간단한 echo server를 demo 네임스페이스에 배포한다.

```yaml
# sample-app.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: demo
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: echo
  namespace: demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: echo
  template:
    metadata:
      labels:
        app: echo
    spec:
      containers:
      - name: echo
        image: ealen/echo-server:latest
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: echo
  namespace: demo
spec:
  selector:
    app: echo
  ports:
  - port: 80
    targetPort: 80
```

```bash
kubectl apply -f sample-app.yaml
kubectl get pods -n demo
kubectl get svc -n demo
```

---

## Step 3. Internal Ingress 리소스 생성

Internal NGINX IngressClass를 사용하는 Ingress를 만든다.

```yaml
# internal-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: echo-ingress
  namespace: demo
spec:
  ingressClassName: nginx-internal
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

```bash
kubectl apply -f internal-ingress.yaml
kubectl get ingress -n demo
```

---

## Step 4. 접근 테스트

Internal LB이므로 클러스터 내부 또는 VNet 내에서만 접근 가능하다.

```bash
# Internal LB IP 확인
INTERNAL_IP=$(kubectl get svc -n app-routing-system nginx-internal \
  -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo "Internal LB IP: $INTERNAL_IP"

# 클러스터 내부에서 curl 테스트 (임시 pod 사용)
kubectl run curl-test --rm -it --restart=Never \
  --image=curlimages/curl -- \
  curl -H "Host: echo.internal.demo" http://$INTERNAL_IP/
```

### 실행 결과 (2026-03-19)

```bash
kubectl run curl-test --rm -it --restart=Never \
  --image=curlimages/curl -- \
  curl -H "Host: echo.internal.demo" http://10.224.0.7/
```

```json
{
  "host": {"hostname": "echo.internal.demo", "ip": "::ffff:192.168.2.163"},
  "http": {"method": "GET", "baseUrl": "", "originalUrl": "/", "protocol": "http"},
  "request": {
    "headers": {
      "host": "echo.internal.demo",
      "x-real-ip": "192.168.2.44",
      "x-forwarded-for": "192.168.2.44",
      "x-forwarded-proto": "http"
    }
  }
}
```
> ✅ Internal LB(10.224.0.7) → NGINX Ingress → echo pod 정상 응답 확인

---

## Step 5. (선택) 기존 Public NGINX 정리

고객사 환경처럼 Internal만 사용한다면, 기존 default Public NGINX를 삭제하거나
그대로 두고 Internal 전용으로만 사용해도 된다.

```bash
# 기존 default(public) controller를 삭제하려면:
# kubectl delete NginxIngressController default

# 또는 그대로 두고 Internal만 사용 (두 개 병렬 운영 가능)
```

---

## 정리 - 파일 목록

| 파일 | 설명 |
|---|---|
| `internal-nginx-controller.yaml` | Internal NginxIngressController 정의 |
| `sample-app.yaml` | echo server Deployment + Service |
| `internal-ingress.yaml` | Internal Ingress 리소스 |

---

## 다음 단계

마이그레이션 관련 내용은 [approuting-migration-guide.md](./approuting-migration-guide.md) 참고.
