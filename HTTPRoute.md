# HTTPRoute 구성 가이드 (App Routing Gateway API)

AKS Application Routing add-on의 Gateway API 구현(`approuting-istio`)에서 자주 쓰는 `HTTPRoute` 라우팅 패턴을 정리한다. 모든 예시는 **Shared Gateway(공용 Gateway 1개 + 서비스별 HTTPRoute)** 토폴로지를 전제로 하며, `HTTPRoute`는 각 서비스 네임스페이스에 두고 `parentRefs`로 `gateway-system`의 공용 Gateway를 cross-namespace 참조한다.

> 예시의 도메인(`*.contoso.io`)·경로·네임스페이스·서비스명은 모두 임의의 샘플이다. 실제 환경에 맞게 치환해서 사용한다.

## 목차

1. [사전 이해](#1-사전-이해)
2. [Host 기반 라우팅 (Multi Domain)](#2-host-기반-라우팅-multi-domain)
3. [Path 기반 라우팅 및 URL Rewrite](#3-path-기반-라우팅-및-url-rewrite)
4. [HTTP Request Timeout 구성](#4-http-request-timeout-구성)
5. [Filter 구성 예시 — Redirect / Rewrite / Header Modifier](#5-filter-구성-예시--redirect--rewrite--header-modifier)
6. [라우팅 매칭 우선순위와 팁](#6-라우팅-매칭-우선순위와-팁)

---

## 1. 사전 이해

- **Gateway**: 트래픽 진입점(Listener, Port, Protocol, TLS). 보통 플랫폼 팀이 `gateway-system`에 1개만 운영한다.
- **HTTPRoute**: Host/Path 기반 라우팅 규칙. 서비스 팀이 자기 네임스페이스에 생성한다.
- **parentRefs**: HTTPRoute가 어느 Gateway(어느 Listener)에 붙는지 지정. `sectionName`으로 특정 Listener를 지정할 수 있다.
- **cross-namespace 참조**: HTTPRoute(서비스 ns)가 다른 ns의 Gateway를 참조하려면, Gateway Listener의 `allowedRoutes.namespaces.from`이 `All` 또는 `Selector`여야 한다.
- **filters**: 라우팅 도중 요청/응답을 변형(헤더 수정, URL Rewrite, Redirect 등)하는 표준 확장 지점.

> App Routing Gateway API는 표준 Gateway API 리소스(`Gateway`, `HTTPRoute`)만 reconcile한다. Istio CRD(`VirtualService`, `EnvoyFilter` 등)는 지원하지 않으므로, 모든 라우팅은 HTTPRoute 표준 스펙으로 구현한다.

---

## 2. Host 기반 라우팅 (Multi Domain)

도메인이 여러 개인 경우, Gateway Listener에 와일드카드(`*`) 또는 도메인별 Listener를 구성하고 HTTPRoute는 Hostname별로 분리한다.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: shared-gateway
  namespace: gateway-system
spec:
  infrastructure:
    annotations:
      service.beta.kubernetes.io/azure-load-balancer-internal: "true"
      service.beta.kubernetes.io/azure-load-balancer-internal-subnet: "sn-ingress-lb"
      service.beta.kubernetes.io/azure-load-balancer-ipv4: 10.20.0.10
  gatewayClassName: approuting-istio
  listeners:
    - name: wildcard-listener
      hostname: "*.contoso.io"
      port: 80
      protocol: HTTP
      allowedRoutes:
        namespaces:
          from: All
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: catalog-route
  namespace: ns-catalog
spec:
  parentRefs:
    - name: shared-gateway
      namespace: gateway-system
  hostnames:
    - catalog.contoso.io
  rules:
    - backendRefs:
        - name: catalog-service
          port: 8080
```

**이 설정이 하는 일**

- Gateway는 `*.contoso.io` 와일드카드 Listener 하나로 **모든 서브도메인 트래픽을 단일 Internal LB IP(10.20.0.10)** 로 받는다. 서비스가 늘어도 IP/LB는 1개로 유지된다.
- `allowedRoutes.namespaces.from: All` 덕분에 다른 네임스페이스(`ns-catalog`)의 HTTPRoute가 이 Gateway에 붙을 수 있다.
- HTTPRoute는 `catalog.contoso.io`로 들어온 요청만 골라 `catalog-service:8080`으로 전달한다. 도메인을 추가하려면 같은 패턴으로 HTTPRoute만 더 만들면 된다(Gateway 수정 불필요).

---

## 3. Path 기반 라우팅 및 URL Rewrite

경로별 라우팅과 URL Rewrite는 `spec.rules.matches.path`와 `spec.rules.filters`의 표준 스펙으로 구현한다.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: console-route
  namespace: ns-console
spec:
  parentRefs:
    - name: shared-gateway
      namespace: gateway-system
  hostnames:
    - console.contoso.io
  rules:
    # /orders/* → orders-service (prefix strip: /orders/list → /list)
    - matches:
        - path:
            type: PathPrefix
            value: "/orders"
      filters:
        - type: URLRewrite
          urlRewrite:
            path:
              type: ReplacePrefixMatch
              replacePrefixMatch: /
      backendRefs:
        - name: orders-service
          port: 8080
    # / → 프론트엔드(catch-all, 반드시 마지막에 배치)
    - matches:
        - path:
            type: PathPrefix
            value: "/"
      backendRefs:
        - name: web-frontend
          port: 80
```

**이 설정이 하는 일**

- `console.contoso.io/orders/...` 요청은 `orders-service`로 보내되, `URLRewrite(ReplacePrefixMatch: /)`로 `/orders` 접두사를 제거한다. 예: 외부 `/orders/list` → 백엔드 수신 `/list`. 백엔드가 `/orders` 경로를 모를 때 유용하다.
- 그 외 모든 경로(`/`)는 `web-frontend`로 보낸다. 이는 **HTTP 상태코드 기반 에러 페이지가 아니라, "매칭되는 경로가 없을 때의 default backend(경로 fallback)"** 다. 백엔드가 직접 반환하는 5xx는 이 규칙으로 가로채지 못한다.
- Gateway API는 **더 구체적인(prefix가 더 긴) 매칭이 우선**이므로 `/orders`가 `/`보다 먼저 평가된다. 규칙 작성 순서와 무관하게 우선순위가 결정되지만, 가독성을 위해 catch-all은 마지막에 둔다.

---

## 4. HTTP Request Timeout 구성

기존 NGINX annotation으로 구성하던 요청 타임아웃은 Gateway API HTTPRoute 스펙(`spec.rules.timeouts`)에 내장되어 있다.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: reports-timeout
  namespace: ns-reports
spec:
  parentRefs:
    - name: shared-gateway
      namespace: gateway-system
      sectionName: https
  hostnames:
    - reports.contoso.io
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /reports
      backendRefs:
        - name: reports-service
          port: 8080
      timeouts:
        request: 800ms
```

**이 설정이 하는 일**

- `reports.contoso.io/reports/...` 요청에 대해 **전체 요청 타임아웃을 800ms로 제한**한다. 백엔드가 800ms 안에 응답을 시작하지 못하면 Gateway가 `504 Gateway Timeout`을 반환한다.
- `timeouts.request`는 클라이언트 요청 수신 ~ 백엔드 응답 완료까지의 상한이다. 별도로 `timeouts.backendRequest`(개별 재시도당 타임아웃)도 지정할 수 있다.
- 값은 Go duration 형식(`300ms`, `2s`, `1m`)을 사용한다. 느린 배치/리포트 API는 넉넉히, 사용자 대면 API는 짧게 잡는다.

---

## 5. Filter 구성 예시 — Redirect / Rewrite / Header Modifier

| Filter Type | 설명 |
|---|---|
| `RequestHeaderModifier` | 요청 헤더 추가(add) / 덮어쓰기(set) / 삭제(remove) |
| `ResponseHeaderModifier` | 응답 헤더 추가(add) / 덮어쓰기(set) / 삭제(remove) |
| `RequestRedirect` | 클라이언트에게 3xx 리다이렉트 응답 반환(scheme/hostname/path/port/statusCode 변경) |
| `URLRewrite` | 업스트림 전달 전 요청 변경(`ReplaceFullPath` / `ReplacePrefixMatch` / hostname) |

### 5.1. RequestHeaderModifier / ResponseHeaderModifier

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: header-modify-demo
  namespace: ns-demo
spec:
  parentRefs:
    - name: shared-gateway
      namespace: gateway-system
  hostnames:
    - echo.contoso.io
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      filters:
        - type: RequestHeaderModifier
          requestHeaderModifier:
            add:
              - name: x-forwarded-by
                value: contoso-gateway
        - type: ResponseHeaderModifier
          responseHeaderModifier:
            add:
              - name: x-served-by
                value: approuting-istio
            set:
              - name: cache-control
                value: no-store
      backendRefs:
        - name: echo-service
          port: 8080
```

**이 설정이 하는 일**

- **요청 측**: 백엔드로 전달되기 전 `x-forwarded-by: contoso-gateway` 헤더를 추가한다. 백엔드가 게이트웨이 경유 여부를 식별하는 용도로 쓸 수 있다.
- **응답 측**: 클라이언트로 나가는 응답에 `x-served-by`를 추가하고, `cache-control`을 `no-store`로 강제 덮어쓴다(`set`은 기존 값을 교체).
- `add`는 기존 헤더에 누적, `set`은 교체, `remove`는 삭제다. 보안 헤더 주입(예: `set`으로 `x-frame-options`)이나 캐시 정책 강제에 자주 쓴다.

### 5.2. RequestRedirect

**Case 1. HTTP → HTTPS 강제 전환**

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: redirect-http-to-https
  namespace: ns-demo
spec:
  parentRefs:
    - name: shared-gateway
      namespace: gateway-system
      sectionName: http
  hostnames:
    - "*.contoso.io"
  rules:
    - filters:
        - type: RequestRedirect
          requestRedirect:
            scheme: https
            statusCode: 301
```

**이 설정이 하는 일**: HTTP Listener(`sectionName: http`)로 들어온 모든 `*.contoso.io` 요청을 동일 경로의 `https://`로 **301 영구 리다이렉트**한다. 백엔드로 전달하지 않고 게이트웨이가 즉시 응답하므로, 전 서비스 공통 HTTPS 강제에 쓴다.

**Case 2. 구 도메인 → 신 도메인 (도메인 이전)**

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: redirect-legacy-domain
  namespace: ns-demo
spec:
  parentRefs:
    - name: shared-gateway
      namespace: gateway-system
      sectionName: https
  hostnames:
    - shop-old.contoso.io
  rules:
    - filters:
        - type: RequestRedirect
          requestRedirect:
            hostname: shop.contoso.io
            statusCode: 301
```

**이 설정이 하는 일**: `shop-old.contoso.io`로 온 요청을 경로는 유지한 채 `shop.contoso.io`로 301 리다이렉트한다. 도메인 통합/이전 시 기존 북마크·검색엔진 인덱스를 새 도메인으로 넘긴다.

**Case 3. API 버전 마이그레이션 `/v1/*` → `/v2/*`**

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: redirect-api-version
  namespace: ns-demo
spec:
  parentRefs:
    - name: shared-gateway
      namespace: gateway-system
      sectionName: https
  hostnames:
    - api.contoso.io
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /v1
      filters:
        - type: RequestRedirect
          requestRedirect:
            path:
              type: ReplacePrefixMatch
              replacePrefixMatch: /v2
            statusCode: 301
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: api-backend
          port: 8080
```

**이 설정이 하는 일**: `api.contoso.io/v1/...` 요청을 `/v2/...`로 301 리다이렉트(클라이언트 URL 자체가 바뀜)하고, 그 외 경로는 정상적으로 백엔드로 보낸다. 클라이언트가 신버전 경로를 쓰도록 유도할 때 사용한다. (URL을 바꾸지 않고 내부적으로만 넘기려면 Redirect 대신 URLRewrite를 쓴다.)

### 5.3. URLRewrite

**Case 1. API Gateway Path Prefix 제거**

```yaml
# 외부: /api/users/* → 백엔드: /users/*
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: rewrite-strip-api-prefix
  namespace: ns-demo
spec:
  parentRefs:
    - name: shared-gateway
      namespace: gateway-system
      sectionName: https
  hostnames:
    - api.contoso.io
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /api
      filters:
        - type: URLRewrite
          urlRewrite:
            path:
              type: ReplacePrefixMatch
              replacePrefixMatch: /
      backendRefs:
        - name: api-backend
          port: 8080
```

**이 설정이 하는 일**: 외부 경로 `/api/users` 를 백엔드에는 `/users` 로 전달한다(`/api` 접두사 제거). RequestRedirect와 달리 **클라이언트 URL은 그대로 유지**되고 게이트웨이 내부에서만 경로가 바뀐다. 백엔드가 `/api` prefix를 모를 때 쓴다.

**Case 2. 마이크로서비스 라우팅 + 버전 경로 제거**

```yaml
# /v1/users/* → /users/*, /v1/orders/* → /orders/*
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: rewrite-microservice-routing
  namespace: ns-demo
spec:
  parentRefs:
    - name: shared-gateway
      namespace: gateway-system
      sectionName: https
  hostnames:
    - app.contoso.io
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /v1/users
      filters:
        - type: URLRewrite
          urlRewrite:
            path:
              type: ReplacePrefixMatch
              replacePrefixMatch: /users
      backendRefs:
        - name: users-service
          port: 8080
    - matches:
        - path:
            type: PathPrefix
            value: /v1/orders
      filters:
        - type: URLRewrite
          urlRewrite:
            path:
              type: ReplacePrefixMatch
              replacePrefixMatch: /orders
      backendRefs:
        - name: orders-service
          port: 8080
```

**이 설정이 하는 일**: 외부에는 버전 경로(`/v1`)를 노출하되, 백엔드별로 버전 없는 경로로 변환해 각 마이크로서비스로 분기한다. 단일 호스트(`app.contoso.io`) 아래에서 경로로 여러 서비스를 묶는 API Gateway 패턴이다.

**Case 3. Host Header Rewrite (레거시 백엔드 연동)**

```yaml
# 클라이언트 → legacy.contoso.io, 백엔드 수신 Host → legacy-app.ns-demo.svc.cluster.local
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: rewrite-host-header
  namespace: ns-demo
spec:
  parentRefs:
    - name: shared-gateway
      namespace: gateway-system
      sectionName: https
  hostnames:
    - legacy.contoso.io
  rules:
    - filters:
        - type: URLRewrite
          urlRewrite:
            hostname: legacy-app.ns-demo.svc.cluster.local
      backendRefs:
        - name: legacy-app
          port: 8080
```

**이 설정이 하는 일**: 클라이언트는 `legacy.contoso.io`로 접속하지만, 백엔드로 전달되는 `Host` 헤더를 내부 서비스 FQDN으로 바꾼다. 특정 Host 헤더만 허용하는 레거시 애플리케이션을 변경 없이 연동할 때 쓴다.

**Case 4. Full Path 교체 (특정 경로를 고정 경로로 매핑)**

```yaml
# /healthz → /actuator/health (외부 경로와 내부 헬스체크 엔드포인트 분리)
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: rewrite-health-endpoint
  namespace: ns-demo
spec:
  parentRefs:
    - name: shared-gateway
      namespace: gateway-system
      sectionName: https
  hostnames:
    - api.contoso.io
  rules:
    - matches:
        - path:
            type: Exact
            value: /healthz
      filters:
        - type: URLRewrite
          urlRewrite:
            path:
              type: ReplaceFullPath
              replaceFullPath: /actuator/health
      backendRefs:
        - name: api-backend
          port: 8080
```

**이 설정이 하는 일**: 외부에 노출하는 헬스체크 경로(`/healthz`)와 실제 백엔드 엔드포인트(`/actuator/health`)를 분리한다. `ReplaceFullPath`는 prefix가 아니라 **경로 전체를 통째로 교체**하므로, `Exact` 매칭과 함께 단일 경로를 고정 경로로 매핑할 때 적합하다.

---

## 6. 라우팅 매칭 우선순위와 팁

- **매칭 우선순위(Gateway API 표준)**: ① `Exact` path > `PathPrefix`(더 긴 prefix 우선) > 와일드카드, ② method/header/query 매칭이 많을수록 우선. 규칙 작성 순서가 아니라 이 규칙으로 결정된다. catch-all(`/`)은 항상 가장 낮은 우선순위라 마지막에 두면 가독성이 좋다.
- **Redirect vs Rewrite**: 클라이언트가 보는 URL을 바꾸려면 `RequestRedirect`(3xx 응답), 클라이언트 URL은 그대로 두고 백엔드 전달 경로만 바꾸려면 `URLRewrite`.
- **cross-namespace 주의**: HTTPRoute가 다른 ns의 Gateway를 참조하려면 Gateway Listener의 `allowedRoutes`가 허용해야 한다. backendRef가 다른 ns의 Service를 가리키는 경우에는 `ReferenceGrant`가 추가로 필요하다(같은 ns면 불필요).
- **App Routing 제약**: Istio CRD(`VirtualService`, `EnvoyFilter`) 미지원. 응답 상태코드 기반 에러 페이지(NGINX `custom-http-errors` 류)와 전역 Default Backend는 표준 HTTPRoute로 구현 불가하다. 경로 fallback(catch-all)으로만 부분 대응한다.
- **TLS**: HTTPS Listener를 사용하는 예시는 `sectionName: https`로 해당 Listener에 바인딩한다. Listener의 TLS/인증서 구성은 Gateway 측에서 별도로 다룬다.
