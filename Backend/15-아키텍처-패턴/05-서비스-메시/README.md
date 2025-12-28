# 서비스 메시 (Service Mesh)

## 목차
1. [개요](#개요)
2. [Service Mesh 개념](#service-mesh-개념)
3. [Istio 아키텍처](#istio-아키텍처)
4. [Linkerd](#linkerd)
5. [Envoy Proxy](#envoy-proxy)
6. [Sidecar 패턴](#sidecar-패턴)
7. [트래픽 관리](#트래픽-관리)
8. [mTLS (Mutual TLS)](#mtls-mutual-tls)
9. [관측성 (Observability)](#관측성-observability)
10. [Service Mesh vs API Gateway](#service-mesh-vs-api-gateway)
11. [실무 적용 가이드](#실무-적용-가이드)
---

## 개요

마이크로서비스 아키텍처가 복잡해지면서 **서비스 간 통신 관리**가 주요 과제가 되었다. Service Mesh는 이 문제를 **인프라 레이어에서 해결**하는 접근 방식이다.

### 마이크로서비스의 과제

```
┌─────────────────────────────────────────────────────────────────────┐
│                  마이크로서비스 통신의 과제                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  서비스 A ────────?────────► 서비스 B                                │
│                                                                      │
│  해결해야 할 문제들:                                                  │
│  ─────────────────────────────────────────────────────────────────  │
│                                                                      │
│  1. 서비스 디스커버리                                                │
│     → 서비스 B는 어디에 있는가? IP가 바뀌면?                          │
│                                                                      │
│  2. 로드 밸런싱                                                      │
│     → 서비스 B의 여러 인스턴스 중 어디로 보낼 것인가?                  │
│                                                                      │
│  3. 장애 복구                                                        │
│     → 서비스 B가 응답하지 않으면? 재시도? 타임아웃?                   │
│                                                                      │
│  4. 보안                                                             │
│     → 통신이 암호화되어 있는가? 인증은?                               │
│                                                                      │
│  5. 관측성                                                           │
│     → 요청이 어디서 실패했는가? 지연 시간은?                          │
│                                                                      │
│  6. 트래픽 제어                                                      │
│     → 카나리 배포, A/B 테스트, 속도 제한                              │
│                                                                      │
│  기존 해결책: 각 서비스에 라이브러리 추가 (Spring Cloud, Netflix OSS) │
│  문제점: 언어/프레임워크마다 구현 필요, 버전 관리 어려움               │
│                                                                      │
│  Service Mesh: 이 모든 것을 인프라 레이어에서 처리!                   │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Service Mesh 개념

### 정의

Service Mesh는 **마이크로서비스 간 통신을 관리하는 전용 인프라 레이어**이다. 애플리케이션 코드 변경 없이 서비스 간 통신에 대한 관측성, 보안, 신뢰성을 제공한다.

### 아키텍처 구조

```
┌─────────────────────────────────────────────────────────────────────┐
│                   Service Mesh 아키텍처                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                     Control Plane                            │    │
│  │                                                               │    │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │    │
│  │  │  Discovery  │  │   Config    │  │     Certificate     │  │    │
│  │  │   Service   │  │   Server    │  │       Authority     │  │    │
│  │  └─────────────┘  └─────────────┘  └─────────────────────┘  │    │
│  │                                                               │    │
│  │  - 서비스 레지스트리 관리                                     │    │
│  │  - 트래픽 정책 배포                                          │    │
│  │  - 인증서 발급/갱신                                          │    │
│  │                                                               │    │
│  └───────────────────────────┬───────────────────────────────────┘    │
│                              │ (정책, 인증서 배포)                   │
│                              ▼                                       │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                      Data Plane                              │    │
│  │                                                               │    │
│  │  ┌──────────────────────┐    ┌──────────────────────┐       │    │
│  │  │        Pod A         │    │        Pod B         │       │    │
│  │  │  ┌───────┐ ┌───────┐│    │┌───────┐ ┌───────┐  │       │    │
│  │  │  │Service│◄┼►│Sidecar││◄───►││Sidecar│◄┼►│Service│  │       │    │
│  │  │  │   A   │ │ Proxy ││    ││ Proxy │ │   B   │  │       │    │
│  │  │  └───────┘ └───────┘│    │└───────┘ └───────┘  │       │    │
│  │  └──────────────────────┘    └──────────────────────┘       │    │
│  │                                                               │    │
│  │  - 모든 트래픽이 Sidecar Proxy를 통과                         │    │
│  │  - 로드 밸런싱, 재시도, 타임아웃, mTLS 처리                   │    │
│  │  - 메트릭, 로그, 트레이스 수집                                │    │
│  │                                                               │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 주요 Service Mesh 솔루션 비교

```
┌─────────────────────────────────────────────────────────────────────┐
│                  Service Mesh 솔루션 비교                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌────────────────┬───────────────┬───────────────┬─────────────┐  │
│  │                │    Istio      │   Linkerd     │   Consul    │  │
│  ├────────────────┼───────────────┼───────────────┼─────────────┤  │
│  │ Proxy          │ Envoy (C++)   │ linkerd2-proxy│ Envoy       │  │
│  │                │               │ (Rust)        │             │  │
│  ├────────────────┼───────────────┼───────────────┼─────────────┤  │
│  │ 성능 오버헤드   │ 중간~높음     │ 낮음          │ 중간        │  │
│  │ (p50 지연)     │ (~2.5ms)      │ (~0.5ms)      │ (~1.5ms)    │  │
│  ├────────────────┼───────────────┼───────────────┼─────────────┤  │
│  │ 메모리 사용량   │ 40-50MB       │ 10-20MB       │ 30-40MB     │  │
│  │ (per proxy)    │               │               │             │  │
│  ├────────────────┼───────────────┼───────────────┼─────────────┤  │
│  │ 기능           │ 매우 풍부     │ 핵심에 집중    │ 풍부        │  │
│  ├────────────────┼───────────────┼───────────────┼─────────────┤  │
│  │ 복잡도         │ 높음          │ 낮음          │ 중간        │  │
│  ├────────────────┼───────────────┼───────────────┼─────────────┤  │
│  │ 학습 곡선      │ 가파름        │ 완만함        │ 중간        │  │
│  ├────────────────┼───────────────┼───────────────┼─────────────┤  │
│  │ 커뮤니티       │ 매우 큼       │ 성장 중       │ 큼          │  │
│  └────────────────┴───────────────┴───────────────┴─────────────┘  │
│                                                                      │
│  선택 가이드:                                                        │
│  • 다양한 기능, 커스터마이징 필요 → Istio                           │
│  • 단순함, 낮은 오버헤드 → Linkerd                                  │
│  • HashiCorp 스택 사용 중 → Consul Connect                          │
│  • 리소스 제약 환경 → Linkerd                                       │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Istio 아키텍처

### Istio 구성 요소

```
┌─────────────────────────────────────────────────────────────────────┐
│                       Istio 아키텍처                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                    Control Plane (istiod)                    │    │
│  │                                                               │    │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │    │
│  │  │   Pilot     │  │   Citadel   │  │       Galley        │  │    │
│  │  │             │  │             │  │                     │  │    │
│  │  │ • 서비스    │  │ • 인증서    │  │ • 설정 검증         │  │    │
│  │  │   디스커버리│  │   관리      │  │ • 설정 배포         │  │    │
│  │  │ • 트래픽    │  │ • mTLS      │  │                     │  │    │
│  │  │   정책 관리 │  │ • RBAC      │  │                     │  │    │
│  │  └─────────────┘  └─────────────┘  └─────────────────────┘  │    │
│  │                                                               │    │
│  │              istiod (통합 컴포넌트, 1.5+)                     │    │
│  │                                                               │    │
│  └──────────────────────────┬────────────────────────────────────┘    │
│                             │ xDS (gRPC)                             │
│                             │ (Listener, Route, Cluster, Endpoint)  │
│                             ▼                                        │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                     Data Plane                               │    │
│  │                                                               │    │
│  │   ┌─────────────────────────────────────────────────────┐   │    │
│  │   │                    Kubernetes Pod                    │   │    │
│  │   │  ┌─────────────────┐    ┌─────────────────────────┐ │   │    │
│  │   │  │   Application   │    │    Envoy Sidecar        │ │   │    │
│  │   │  │   Container     │◄──►│    Proxy                │ │   │    │
│  │   │  │                 │    │                         │ │   │    │
│  │   │  │ - 비즈니스 로직 │    │ - L7 프록시             │ │   │    │
│  │   │  │ - 메시 인식 X   │    │ - TLS 종료/시작        │ │   │    │
│  │   │  │                 │    │ - 로드 밸런싱          │ │   │    │
│  │   │  │                 │    │ - 트래픽 라우팅        │ │   │    │
│  │   │  │                 │    │ - 메트릭 수집          │ │   │    │
│  │   │  └─────────────────┘    └─────────────────────────┘ │   │    │
│  │   └─────────────────────────────────────────────────────┘   │    │
│  │                                                               │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                 Observability (선택적)                       │    │
│  │   ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │    │
│  │   │ Prometheus│  │  Grafana │  │  Jaeger  │  │   Kiali  │   │    │
│  │   │ (메트릭) │  │(대시보드)│  │(트레이싱)│  │(시각화) │   │    │
│  │   └──────────┘  └──────────┘  └──────────┘  └──────────┘   │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### Istio 설치 및 기본 설정

```yaml
# Istio 설치 (istioctl)
# istioctl install --set profile=demo -y

# 네임스페이스에 Sidecar 자동 주입 활성화
apiVersion: v1
kind: Namespace
metadata:
  name: my-app
  labels:
    istio-injection: enabled  # 이 라벨로 자동 주입

---
# DestinationRule: 서비스별 트래픽 정책
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: order-service-destination
  namespace: my-app
spec:
  host: order-service
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        h2UpgradePolicy: UPGRADE
        http1MaxPendingRequests: 100
        http2MaxRequests: 1000
    loadBalancer:
      simple: LEAST_CONN  # 최소 연결 로드밸런싱
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 10s
      baseEjectionTime: 30s
      maxEjectionPercent: 50

---
# VirtualService: 라우팅 규칙
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: order-service-routing
  namespace: my-app
spec:
  hosts:
    - order-service
  http:
    - match:
        - headers:
            x-version:
              exact: "v2"
      route:
        - destination:
            host: order-service
            subset: v2
    - route:
        - destination:
            host: order-service
            subset: v1
          weight: 90
        - destination:
            host: order-service
            subset: v2
          weight: 10  # 카나리 배포: 10% 트래픽

---
# Gateway: 외부 트래픽 진입점
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: my-app-gateway
  namespace: my-app
spec:
  selector:
    istio: ingressgateway
  servers:
    - port:
        number: 443
        name: https
        protocol: HTTPS
      tls:
        mode: SIMPLE
        credentialName: my-app-tls-secret
      hosts:
        - "api.example.com"
```

### Istio 트래픽 관리 예제

```yaml
# 재시도 정책
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: payment-service
spec:
  hosts:
    - payment-service
  http:
    - route:
        - destination:
            host: payment-service
      retries:
        attempts: 3
        perTryTimeout: 2s
        retryOn: gateway-error,connect-failure,retriable-4xx

---
# 타임아웃 설정
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: inventory-service
spec:
  hosts:
    - inventory-service
  http:
    - route:
        - destination:
            host: inventory-service
      timeout: 10s

---
# Circuit Breaker (Outlier Detection)
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: payment-service-circuit-breaker
spec:
  host: payment-service
  trafficPolicy:
    outlierDetection:
      consecutive5xxErrors: 5      # 5회 연속 5xx 에러 시
      interval: 30s                # 30초 간격으로 체크
      baseEjectionTime: 60s        # 60초 동안 제외
      maxEjectionPercent: 100      # 최대 100% 제외 가능
      minHealthPercent: 30         # 최소 30% 인스턴스 유지

---
# Fault Injection (테스트용)
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: order-service-fault
spec:
  hosts:
    - order-service
  http:
    - fault:
        delay:
          percentage:
            value: 10  # 10% 요청에
          fixedDelay: 5s  # 5초 지연
        abort:
          percentage:
            value: 5  # 5% 요청에
          httpStatus: 500  # 500 에러
      route:
        - destination:
            host: order-service
```

---

## Linkerd

### Linkerd 특징

```
┌─────────────────────────────────────────────────────────────────────┐
│                       Linkerd 특징                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  "Simple, ultralight, security-first service mesh for Kubernetes"  │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  핵심 특징                                                    │    │
│  │                                                               │    │
│  │  1. Rust 기반 Micro-Proxy (linkerd2-proxy)                   │    │
│  │     • 메모리: ~10MB (vs Envoy 40-50MB)                       │    │
│  │     • p50 지연: ~0.5ms (vs Istio ~2.5ms)                     │    │
│  │     • 메모리 안전성 (Rust 언어 특성)                          │    │
│  │                                                               │    │
│  │  2. 단순성                                                    │    │
│  │     • 설치: 5분 이내                                          │    │
│  │     • 설정: 최소 구성으로 동작                                 │    │
│  │     • 문서: 명확하고 따라하기 쉬움                             │    │
│  │                                                               │    │
│  │  3. Security-First                                           │    │
│  │     • 기본 mTLS 활성화                                        │    │
│  │     • 자동 인증서 갱신                                        │    │
│  │     • 제로 트러스트 네트워크                                   │    │
│  │                                                               │    │
│  │  4. CNCF Graduated Project                                   │    │
│  │     • 프로덕션 검증 완료                                      │    │
│  │     • 활발한 커뮤니티                                         │    │
│  │                                                               │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  적합한 사용 사례:                                                   │
│  ─────────────────────────────────────────────────────────────────  │
│  • 리소스가 제한된 환경                                             │
│  • 빠른 도입이 필요한 경우                                          │
│  • 복잡한 기능보다 핵심 기능에 집중                                  │
│  • 학습 곡선을 최소화하고 싶은 경우                                  │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### Linkerd 설치 및 사용

```bash
# Linkerd CLI 설치
curl --proto '=https' --tlsv1.2 -sSfL https://run.linkerd.io/install | sh

# 설치 전 사전 검사
linkerd check --pre

# Control Plane 설치
linkerd install --crds | kubectl apply -f -
linkerd install | kubectl apply -f -

# 설치 확인
linkerd check

# Viz 확장 (대시보드)
linkerd viz install | kubectl apply -f -

# 대시보드 접근
linkerd viz dashboard
```

```yaml
# 네임스페이스에 메시 활성화
apiVersion: v1
kind: Namespace
metadata:
  name: my-app
  annotations:
    linkerd.io/inject: enabled

---
# ServiceProfile: 재시도, 타임아웃 설정
apiVersion: linkerd.io/v1alpha2
kind: ServiceProfile
metadata:
  name: order-service.my-app.svc.cluster.local
  namespace: my-app
spec:
  routes:
    - name: POST /api/orders
      condition:
        method: POST
        pathRegex: /api/orders
      responseClasses:
        - condition:
            status:
              min: 500
              max: 599
          isFailure: true
      # 재시도 설정
      isRetryable: true
    - name: GET /api/orders/{id}
      condition:
        method: GET
        pathRegex: /api/orders/[^/]+
      timeout: 10s

---
# TrafficSplit: 카나리 배포
apiVersion: split.smi-spec.io/v1alpha1
kind: TrafficSplit
metadata:
  name: order-service-split
  namespace: my-app
spec:
  service: order-service
  backends:
    - service: order-service-v1
      weight: 900  # 90%
    - service: order-service-v2
      weight: 100  # 10%
```

---

## Envoy Proxy

### Envoy 개요

```
┌─────────────────────────────────────────────────────────────────────┐
│                       Envoy Proxy 아키텍처                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Envoy는 L7 프록시로, Istio/AWS App Mesh의 데이터 플레인으로 사용됨  │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                      Envoy 구조                               │    │
│  │                                                               │    │
│  │   ┌─────────────────────────────────────────────────────┐   │    │
│  │   │                    Listeners                         │   │    │
│  │   │   • 특정 포트에서 연결 수신                           │   │    │
│  │   │   • 0.0.0.0:15001 (outbound), 0.0.0.0:15006 (inbound)│   │    │
│  │   └─────────────────────────┬───────────────────────────┘   │    │
│  │                             │                                │    │
│  │                             ▼                                │    │
│  │   ┌─────────────────────────────────────────────────────┐   │    │
│  │   │                   Filter Chains                      │   │    │
│  │   │   • HTTP Connection Manager                          │   │    │
│  │   │   • TCP Proxy                                        │   │    │
│  │   │   • Rate Limit, JWT Auth 등                          │   │    │
│  │   └─────────────────────────┬───────────────────────────┘   │    │
│  │                             │                                │    │
│  │                             ▼                                │    │
│  │   ┌─────────────────────────────────────────────────────┐   │    │
│  │   │                     Routes                           │   │    │
│  │   │   • 요청을 어느 클러스터로 보낼지 결정                 │   │    │
│  │   │   • Path, Header 기반 라우팅                          │   │    │
│  │   └─────────────────────────┬───────────────────────────┘   │    │
│  │                             │                                │    │
│  │                             ▼                                │    │
│  │   ┌─────────────────────────────────────────────────────┐   │    │
│  │   │                    Clusters                          │   │    │
│  │   │   • 업스트림 서비스 그룹                               │   │    │
│  │   │   • 로드 밸런싱 정책                                   │   │    │
│  │   │   • 헬스 체크                                         │   │    │
│  │   └─────────────────────────┬───────────────────────────┘   │    │
│  │                             │                                │    │
│  │                             ▼                                │    │
│  │   ┌─────────────────────────────────────────────────────┐   │    │
│  │   │                   Endpoints                          │   │    │
│  │   │   • 실제 서비스 인스턴스 (IP:Port)                    │   │    │
│  │   └─────────────────────────────────────────────────────┘   │    │
│  │                                                               │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  xDS API (동적 설정):                                                │
│  • LDS (Listener Discovery Service)                                 │
│  • RDS (Route Discovery Service)                                    │
│  • CDS (Cluster Discovery Service)                                  │
│  • EDS (Endpoint Discovery Service)                                 │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### Envoy 주요 특징

| 특징 | 설명 |
|------|------|
| **L7 프록시** | HTTP/2, gRPC, WebSocket 지원 |
| **동적 설정** | xDS API를 통한 런타임 설정 변경 |
| **관측성** | 풍부한 메트릭, 분산 추적 지원 |
| **확장성** | Wasm, Lua 필터로 커스텀 로직 추가 |
| **성능** | C++로 작성, 높은 처리량 |

---

## Sidecar 패턴

### Sidecar 패턴 개념

```
┌─────────────────────────────────────────────────────────────────────┐
│                      Sidecar 패턴                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  "메인 애플리케이션 옆에 보조 컨테이너를 배치하여 기능 확장"           │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                    Kubernetes Pod                            │    │
│  │  ┌─────────────────────────────────────────────────────┐    │    │
│  │  │  Shared Resources                                    │    │    │
│  │  │  • Network Namespace (localhost로 통신)              │    │    │
│  │  │  • Volume (공유 파일 시스템)                         │    │    │
│  │  │  • IPC Namespace                                     │    │    │
│  │  └─────────────────────────────────────────────────────┘    │    │
│  │                                                              │    │
│  │  ┌─────────────────────┐    ┌─────────────────────────┐    │    │
│  │  │    Application      │    │       Sidecar           │    │    │
│  │  │     Container       │    │       Container         │    │    │
│  │  │                     │    │                         │    │    │
│  │  │ • 비즈니스 로직     │◄──►│ • 트래픽 프록시         │    │    │
│  │  │ • 포트: 8080        │    │ • 포트: 15001 (inbound) │    │    │
│  │  │                     │    │        15006 (outbound) │    │    │
│  │  │                     │    │                         │    │    │
│  │  └─────────────────────┘    └─────────────────────────┘    │    │
│  │                                                              │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  트래픽 흐름:                                                        │
│  ─────────────────────────────────────────────────────────────────  │
│                                                                      │
│  외부 요청                                                           │
│      │                                                              │
│      ▼                                                              │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ iptables (PREROUTING) ──► Sidecar Inbound (15006)           │   │
│  │                               │                              │   │
│  │                               ▼                              │   │
│  │                          Application (8080)                  │   │
│  │                               │                              │   │
│  │                               ▼ (outbound request)           │   │
│  │ iptables (OUTPUT) ◄── Sidecar Outbound (15001)              │   │
│  │                               │                              │   │
│  │                               ▼                              │   │
│  │                          외부 서비스                          │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### Sidecar 자동 주입

```yaml
# Istio Sidecar 주입 설정
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  namespace: my-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
      annotations:
        # Sidecar 주입 관련 설정
        sidecar.istio.io/inject: "true"
        sidecar.istio.io/proxyCPU: "100m"
        sidecar.istio.io/proxyMemory: "128Mi"
        sidecar.istio.io/proxyMemoryLimit: "256Mi"
        # 트래픽 제외 설정
        traffic.sidecar.istio.io/excludeInboundPorts: "8081"  # 헬스체크 포트 제외
        traffic.sidecar.istio.io/excludeOutboundPorts: "3306"  # MySQL 직접 연결
    spec:
      containers:
        - name: order-service
          image: order-service:v1.0.0
          ports:
            - containerPort: 8080
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"

# 주입 후 Pod 구조 (kubectl get pod order-service-xxx -o yaml)
# containers:
#   - name: order-service        # 애플리케이션
#   - name: istio-proxy          # Sidecar (자동 주입됨)
# initContainers:
#   - name: istio-init           # iptables 규칙 설정
```

### Sidecar-less 아키텍처 (Ambient Mesh)

```
┌─────────────────────────────────────────────────────────────────────┐
│                   Sidecar-less: Istio Ambient Mesh                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  기존 Sidecar 모델의 단점:                                           │
│  • Pod마다 Sidecar → 리소스 오버헤드                                 │
│  • Sidecar 업그레이드 = Pod 재시작 필요                              │
│  • 복잡한 iptables 규칙                                             │
│                                                                      │
│  Ambient Mesh (Sidecar-less):                                       │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                                                              │    │
│  │  Node 1                                                      │    │
│  │  ┌──────────────────────────────────────────────────────┐   │    │
│  │  │  ztunnel (DaemonSet)                                  │   │    │
│  │  │  • L4 보안 (mTLS)                                     │   │    │
│  │  │  • 노드당 하나                                         │   │    │
│  │  └──────────────────────────────────────────────────────┘   │    │
│  │       │                │                │                    │    │
│  │       ▼                ▼                ▼                    │    │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────┐             │    │
│  │  │   Pod A    │  │   Pod B    │  │   Pod C    │             │    │
│  │  │ (No Proxy) │  │ (No Proxy) │  │ (No Proxy) │             │    │
│  │  └────────────┘  └────────────┘  └────────────┘             │    │
│  │                                                              │    │
│  │  Waypoint Proxy (L7 기능 필요 시)                            │    │
│  │  ┌──────────────────────────────────────────────────────┐   │    │
│  │  │  • 라우팅, 정책, 관측성                               │   │    │
│  │  │  • 서비스 어카운트별 배포                              │   │    │
│  │  └──────────────────────────────────────────────────────┘   │    │
│  │                                                              │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  장점:                                                               │
│  • Sidecar 없이 Pod 실행 → 리소스 절약                              │
│  • Mesh 업그레이드가 Pod에 영향 없음                                 │
│  • 점진적 도입 가능                                                  │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 트래픽 관리

### 트래픽 관리 기능

```
┌─────────────────────────────────────────────────────────────────────┐
│                      트래픽 관리 기능                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  1. 로드 밸런싱                                                      │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  • ROUND_ROBIN: 순차적 분배                                  │    │
│  │  • LEAST_CONN: 최소 연결 수                                  │    │
│  │  • RANDOM: 무작위 선택                                       │    │
│  │  • PASSTHROUGH: 클라이언트 IP 해시                           │    │
│  │  • Consistent Hashing: 세션 어피니티 (헤더/쿠키 기반)         │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  2. 트래픽 분할 (Canary, Blue-Green)                                │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                                                              │    │
│  │  Client Request                                              │    │
│  │        │                                                     │    │
│  │        ▼                                                     │    │
│  │  ┌─────────────────────────────────────────┐                │    │
│  │  │         VirtualService                   │                │    │
│  │  │  if header["x-canary"] == "true":       │                │    │
│  │  │      → v2 (100%)                         │                │    │
│  │  │  else:                                   │                │    │
│  │  │      → v1 (90%), v2 (10%)               │                │    │
│  │  └─────────────────────────────────────────┘                │    │
│  │        │                    │                                │    │
│  │        ▼                    ▼                                │    │
│  │  [ Service v1 ]       [ Service v2 ]                        │    │
│  │     (Stable)            (Canary)                            │    │
│  │                                                              │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  3. A/B 테스트                                                       │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  헤더/쿠키 기반 라우팅                                       │    │
│  │  - User-Agent 기반: 모바일 → v2, 데스크톱 → v1              │    │
│  │  - 사용자 ID 기반: 특정 사용자 그룹 → 실험 버전              │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  4. 미러링 (Shadow Traffic)                                         │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                                                              │    │
│  │  Client ─────► v1 (실제 응답)                               │    │
│  │          └───► v2 (미러링, fire-and-forget)                 │    │
│  │                                                              │    │
│  │  용도: 새 버전을 실제 트래픽으로 테스트 (영향 없이)           │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 트래픽 관리 설정 예제

```yaml
# 카나리 배포
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: order-service-canary
spec:
  hosts:
    - order-service
  http:
    # 1. 카나리 헤더가 있으면 무조건 v2로
    - match:
        - headers:
            x-canary:
              exact: "true"
      route:
        - destination:
            host: order-service
            subset: v2
    # 2. 일반 트래픽은 90:10 분할
    - route:
        - destination:
            host: order-service
            subset: v1
          weight: 90
        - destination:
            host: order-service
            subset: v2
          weight: 10

---
# DestinationRule: subset 정의
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: order-service-subsets
spec:
  host: order-service
  subsets:
    - name: v1
      labels:
        version: v1
    - name: v2
      labels:
        version: v2

---
# 트래픽 미러링
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: order-service-mirror
spec:
  hosts:
    - order-service
  http:
    - route:
        - destination:
            host: order-service
            subset: v1
      mirror:
        host: order-service
        subset: v2
      mirrorPercentage:
        value: 100.0  # 100% 트래픽 미러링

---
# Rate Limiting (Envoy Filter)
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: rate-limit-filter
  namespace: istio-system
spec:
  workloadSelector:
    labels:
      app: order-service
  configPatches:
    - applyTo: HTTP_FILTER
      match:
        context: SIDECAR_INBOUND
        listener:
          filterChain:
            filter:
              name: envoy.filters.network.http_connection_manager
      patch:
        operation: INSERT_BEFORE
        value:
          name: envoy.filters.http.local_ratelimit
          typed_config:
            "@type": type.googleapis.com/envoy.extensions.filters.http.local_ratelimit.v3.LocalRateLimit
            stat_prefix: http_local_rate_limiter
            token_bucket:
              max_tokens: 100       # 최대 토큰
              tokens_per_fill: 10   # 충전당 토큰
              fill_interval: 1s     # 충전 간격
```

---

## mTLS (Mutual TLS)

### mTLS 개념

```
┌─────────────────────────────────────────────────────────────────────┐
│                       mTLS (Mutual TLS)                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  TLS vs mTLS:                                                       │
│  ─────────────────────────────────────────────────────────────────  │
│                                                                      │
│  일반 TLS (단방향):                                                  │
│  ┌────────────┐                    ┌────────────┐                   │
│  │   Client   │ ───────────────── │   Server   │                   │
│  │            │ 1. Server 인증서  │            │                   │
│  │            │    검증           │            │                   │
│  │            │ 2. 암호화 통신    │            │                   │
│  └────────────┘                    └────────────┘                   │
│                                                                      │
│  클라이언트만 서버를 검증                                            │
│                                                                      │
│  mTLS (양방향):                                                      │
│  ┌────────────┐                    ┌────────────┐                   │
│  │   Client   │ ◄──────────────► │   Server   │                   │
│  │            │ 1. Client 인증서  │            │                   │
│  │            │    검증           │            │                   │
│  │            │ 2. Server 인증서  │            │                   │
│  │            │    검증           │            │                   │
│  │            │ 3. 암호화 통신    │            │                   │
│  └────────────┘                    └────────────┘                   │
│                                                                      │
│  양쪽 모두 상대방을 검증 → Zero Trust 구현                           │
│                                                                      │
│                                                                      │
│  Service Mesh에서의 mTLS:                                           │
│  ─────────────────────────────────────────────────────────────────  │
│                                                                      │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │                    Control Plane (istiod)                     │  │
│  │                                                                │  │
│  │  ┌─────────────────────────────────────────────────────────┐ │  │
│  │  │                  Certificate Authority                   │ │  │
│  │  │  • SPIFFE ID 기반 인증서 발급                            │ │  │
│  │  │  • 자동 갱신 (기본 24시간 유효)                          │ │  │
│  │  │  • 무중단 인증서 교체                                    │ │  │
│  │  └─────────────────────────────────────────────────────────┘ │  │
│  └──────────────────────────┬────────────────────────────────────┘  │
│                             │                                       │
│                             │ 인증서 배포                           │
│                             ▼                                       │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  Pod A                          Pod B                         │  │
│  │  ┌────────┐ ┌────────┐    ┌────────┐ ┌────────┐              │  │
│  │  │App    │◄┤Sidecar │◄═══►│Sidecar │◄┤App    │              │  │
│  │  │       │ │ (cert) │mTLS │ (cert) │ │       │              │  │
│  │  └────────┘ └────────┘    └────────┘ └────────┘              │  │
│  │                                                                │  │
│  │  • 애플리케이션은 mTLS 인식 불필요                             │  │
│  │  • Sidecar가 투명하게 TLS 종료/시작                           │  │
│  │  • 인증서 갱신 자동화                                         │  │
│  │                                                                │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### mTLS 설정

```yaml
# Istio mTLS 설정

# PeerAuthentication: 네임스페이스 전체에 mTLS 강제
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: my-app
spec:
  mtls:
    mode: STRICT  # STRICT: mTLS 필수, PERMISSIVE: 선택적

---
# 특정 워크로드에만 다른 설정 적용
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: legacy-service
  namespace: my-app
spec:
  selector:
    matchLabels:
      app: legacy-service
  mtls:
    mode: PERMISSIVE  # 레거시 서비스는 일반 트래픽도 허용

---
# DestinationRule: 클라이언트 측 TLS 설정
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: order-service-tls
  namespace: my-app
spec:
  host: order-service
  trafficPolicy:
    tls:
      mode: ISTIO_MUTUAL  # Istio가 관리하는 mTLS 사용

---
# AuthorizationPolicy: 서비스 간 접근 제어
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: order-service-authz
  namespace: my-app
spec:
  selector:
    matchLabels:
      app: order-service
  action: ALLOW
  rules:
    # API Gateway에서만 접근 허용
    - from:
        - source:
            principals: ["cluster.local/ns/istio-system/sa/istio-ingressgateway-service-account"]
    # 같은 네임스페이스의 특정 서비스에서만 접근 허용
    - from:
        - source:
            namespaces: ["my-app"]
            principals: ["cluster.local/ns/my-app/sa/payment-service"]

---
# RequestAuthentication: JWT 검증
apiVersion: security.istio.io/v1beta1
kind: RequestAuthentication
metadata:
  name: jwt-auth
  namespace: my-app
spec:
  selector:
    matchLabels:
      app: order-service
  jwtRules:
    - issuer: "https://auth.example.com"
      jwksUri: "https://auth.example.com/.well-known/jwks.json"
      audiences:
        - "order-service"
      forwardOriginalToken: true

---
# AuthorizationPolicy: JWT 클레임 기반 인가
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: require-jwt
  namespace: my-app
spec:
  selector:
    matchLabels:
      app: order-service
  action: ALLOW
  rules:
    - from:
        - source:
            requestPrincipals: ["https://auth.example.com/*"]
      when:
        - key: request.auth.claims[role]
          values: ["admin", "user"]
```

---

## 관측성 (Observability)

### Service Mesh의 관측성

```
┌─────────────────────────────────────────────────────────────────────┐
│                     관측성 (Observability)                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  3가지 핵심 요소:                                                    │
│  ─────────────────────────────────────────────────────────────────  │
│                                                                      │
│  1. Metrics (메트릭)                                                │
│     ┌─────────────────────────────────────────────────────────┐    │
│     │  • 요청 수, 에러율, 지연 시간 (RED Metrics)              │    │
│     │  • 프로메테우스 형식으로 자동 수집                        │    │
│     │  • 서비스별, 버전별, 경로별 분류                          │    │
│     │                                                          │    │
│     │  예시 메트릭:                                            │    │
│     │  istio_requests_total{                                   │    │
│     │    destination_service="order-service",                  │    │
│     │    response_code="200"                                   │    │
│     │  } 12345                                                 │    │
│     └─────────────────────────────────────────────────────────┘    │
│                                                                      │
│  2. Logs (로그)                                                     │
│     ┌─────────────────────────────────────────────────────────┐    │
│     │  • Envoy 액세스 로그                                     │    │
│     │  • 요청/응답 헤더, 상태 코드, 지연 시간                   │    │
│     │  • 구조화된 JSON 형식                                    │    │
│     │                                                          │    │
│     │  예시 로그:                                              │    │
│     │  {                                                       │    │
│     │    "authority": "order-service:8080",                    │    │
│     │    "method": "POST",                                     │    │
│     │    "path": "/api/orders",                                │    │
│     │    "response_code": 201,                                 │    │
│     │    "duration": "45ms",                                   │    │
│     │    "upstream_cluster": "order-service-v1"                │    │
│     │  }                                                       │    │
│     └─────────────────────────────────────────────────────────┘    │
│                                                                      │
│  3. Traces (분산 추적)                                              │
│     ┌─────────────────────────────────────────────────────────┐    │
│     │                                                          │    │
│     │  Client ───► API GW ───► Order ───► Payment ───► DB     │    │
│     │     │          │          │          │          │        │    │
│     │     └──────────┴──────────┴──────────┴──────────┘        │    │
│     │                    Trace ID: abc-123                      │    │
│     │                                                          │    │
│     │  각 구간(Span)의 시작/종료 시간, 메타데이터 수집          │    │
│     │  → 요청이 어디서 지연되는지, 어디서 실패하는지 파악       │    │
│     │                                                          │    │
│     └─────────────────────────────────────────────────────────┘    │
│                                                                      │
│  관측성 스택:                                                        │
│  ─────────────────────────────────────────────────────────────────  │
│                                                                      │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌────────────┐    │
│  │ Prometheus │  │   Grafana  │  │   Jaeger   │  │   Kiali    │    │
│  │            │  │            │  │            │  │            │    │
│  │  메트릭    │  │  대시보드  │  │ 분산 추적  │  │ 서비스    │    │
│  │  저장/쿼리 │  │  시각화    │  │ 시각화     │  │ 토폴로지  │    │
│  └────────────┘  └────────────┘  └────────────┘  └────────────┘    │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 관측성 설정

```yaml
# Istio 텔레메트리 설정
apiVersion: telemetry.istio.io/v1alpha1
kind: Telemetry
metadata:
  name: mesh-default
  namespace: istio-system
spec:
  # 액세스 로깅
  accessLogging:
    - providers:
        - name: envoy
  # 메트릭 설정
  metrics:
    - providers:
        - name: prometheus
      overrides:
        - match:
            metric: REQUEST_COUNT
          tagOverrides:
            request_path:
              operation: UPSERT
              value: request.url_path

---
# 분산 추적 설정
apiVersion: telemetry.istio.io/v1alpha1
kind: Telemetry
metadata:
  name: tracing-config
  namespace: my-app
spec:
  tracing:
    - providers:
        - name: jaeger
      randomSamplingPercentage: 10  # 10% 샘플링
      customTags:
        environment:
          literal:
            value: "production"

---
# Prometheus ServiceMonitor (메트릭 수집)
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: istio-mesh
  namespace: monitoring
spec:
  selector:
    matchLabels:
      istio: pilot
  endpoints:
    - port: http-monitoring
      interval: 15s
      path: /metrics

---
# Grafana Dashboard ConfigMap (예시)
apiVersion: v1
kind: ConfigMap
metadata:
  name: istio-dashboard
  labels:
    grafana_dashboard: "1"
data:
  istio-dashboard.json: |
    {
      "title": "Istio Service Dashboard",
      "panels": [
        {
          "title": "Request Rate",
          "targets": [
            {
              "expr": "sum(rate(istio_requests_total{reporter=\"destination\"}[5m])) by (destination_service)"
            }
          ]
        },
        {
          "title": "Error Rate",
          "targets": [
            {
              "expr": "sum(rate(istio_requests_total{reporter=\"destination\", response_code=~\"5.*\"}[5m])) by (destination_service) / sum(rate(istio_requests_total{reporter=\"destination\"}[5m])) by (destination_service)"
            }
          ]
        },
        {
          "title": "P99 Latency",
          "targets": [
            {
              "expr": "histogram_quantile(0.99, sum(rate(istio_request_duration_milliseconds_bucket{reporter=\"destination\"}[5m])) by (le, destination_service))"
            }
          ]
        }
      ]
    }
```

### 애플리케이션에서 트레이싱 헤더 전파

```java
// Spring Boot에서 트레이싱 헤더 전파
@Component
public class TracingHeaderPropagator implements ClientHttpRequestInterceptor {

    private static final List<String> TRACE_HEADERS = List.of(
        "x-request-id",
        "x-b3-traceid",
        "x-b3-spanid",
        "x-b3-parentspanid",
        "x-b3-sampled",
        "x-b3-flags",
        "x-ot-span-context",
        "traceparent",
        "tracestate"
    );

    @Override
    public ClientHttpResponse intercept(
            HttpRequest request,
            byte[] body,
            ClientHttpRequestExecution execution) throws IOException {

        // 현재 요청의 헤더에서 트레이스 헤더 추출
        HttpServletRequest currentRequest =
            ((ServletRequestAttributes) RequestContextHolder.getRequestAttributes())
                .getRequest();

        // 외부 호출에 헤더 전파
        TRACE_HEADERS.forEach(header -> {
            String value = currentRequest.getHeader(header);
            if (value != null) {
                request.getHeaders().add(header, value);
            }
        });

        return execution.execute(request, body);
    }
}

// RestTemplate 설정
@Configuration
public class RestTemplateConfig {

    @Bean
    public RestTemplate restTemplate(TracingHeaderPropagator propagator) {
        RestTemplate restTemplate = new RestTemplate();
        restTemplate.getInterceptors().add(propagator);
        return restTemplate;
    }
}

// WebClient 설정 (Reactive)
@Configuration
public class WebClientConfig {

    @Bean
    public WebClient webClient(TracingHeaderPropagator propagator) {
        return WebClient.builder()
            .filter((request, next) -> {
                // 현재 컨텍스트에서 트레이스 헤더 전파
                return next.exchange(
                    ClientRequest.from(request)
                        // 헤더 추가 로직
                        .build()
                );
            })
            .build();
    }
}
```

---

## Service Mesh vs API Gateway

### 개념 비교

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    Service Mesh vs API Gateway 비교                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   API Gateway                              Service Mesh                      │
│   ─────────────────────────────────────   ─────────────────────────────────  │
│                                                                              │
│   ┌─────────────────┐                     ┌─────────────────────────────┐   │
│   │   Client        │                     │   Client                    │   │
│   │   (외부)        │                     │   (외부)                    │   │
│   └────────┬────────┘                     └────────────┬────────────────┘   │
│            │                                           │                     │
│            ▼                                           ▼                     │
│   ┌─────────────────┐                     ┌─────────────────────────────┐   │
│   │   API Gateway   │                     │   Ingress Gateway           │   │
│   │                 │                     │   (Mesh 진입점)              │   │
│   │   • 인증/인가   │                     └────────────┬────────────────┘   │
│   │   • Rate Limit  │                                  │                     │
│   │   • 요청 변환   │                                  ▼                     │
│   │   • API 라우팅  │                     ┌─────────────────────────────┐   │
│   │   • 캐싱       │                     │   ┌─────┐     ┌─────┐       │   │
│   └────────┬────────┘                     │   │Proxy│◄───►│Proxy│       │   │
│            │                              │   └──┬──┘     └──┬──┘       │   │
│            ▼                              │   ┌──▼──┐     ┌──▼──┐       │   │
│   ┌─────────────────┐                     │   │Svc A│     │Svc B│       │   │
│   │   서비스 A      │                     │   └─────┘     └─────┘       │   │
│   └─────────────────┘                     │                             │   │
│                                           │   ← East-West Traffic →    │   │
│   ↑ North-South Traffic                   └─────────────────────────────┘   │
│   (외부 → 내부)                                                              │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 상세 비교표

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         기능별 상세 비교                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌──────────────────┬─────────────────────┬─────────────────────────────┐   │
│  │      항목        │    API Gateway       │      Service Mesh          │   │
│  ├──────────────────┼─────────────────────┼─────────────────────────────┤   │
│  │ 트래픽 방향      │ North-South          │ East-West                  │   │
│  │                  │ (외부 → 내부)        │ (서비스 ↔ 서비스)          │   │
│  ├──────────────────┼─────────────────────┼─────────────────────────────┤   │
│  │ 배포 위치        │ 클러스터 경계 (Edge) │ 각 서비스 옆 (Sidecar)      │   │
│  ├──────────────────┼─────────────────────┼─────────────────────────────┤   │
│  │ 주요 기능        │ • API 인증/인가       │ • mTLS                      │   │
│  │                  │ • Rate Limiting      │ • 분산 추적                 │   │
│  │                  │ • 요청/응답 변환      │ • 서킷 브레이커             │   │
│  │                  │ • API 버저닝         │ • 로드 밸런싱               │   │
│  │                  │ • 캐싱               │ • 재시도/타임아웃           │   │
│  ├──────────────────┼─────────────────────┼─────────────────────────────┤   │
│  │ 보안 범위        │ 외부 접근 제어        │ 내부 Zero Trust            │   │
│  ├──────────────────┼─────────────────────┼─────────────────────────────┤   │
│  │ 관측성           │ API 레벨             │ 서비스 간 통신 레벨          │   │
│  ├──────────────────┼─────────────────────┼─────────────────────────────┤   │
│  │ 복잡도           │ 상대적 단순           │ 높음 (Sidecar 관리)         │   │
│  ├──────────────────┼─────────────────────┼─────────────────────────────┤   │
│  │ 성능 영향        │ 단일 홉 지연          │ 각 홉마다 지연 추가         │   │
│  ├──────────────────┼─────────────────────┼─────────────────────────────┤   │
│  │ 대표 솔루션      │ Kong, AWS API GW     │ Istio, Linkerd             │   │
│  │                  │ NGINX, Apigee        │ Consul Connect             │   │
│  └──────────────────┴─────────────────────┴─────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 함께 사용하는 통합 아키텍처

```
┌─────────────────────────────────────────────────────────────────────────────┐
│              API Gateway + Service Mesh 통합 아키텍처                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   외부 클라이언트                                                            │
│        │                                                                     │
│        │ HTTPS                                                               │
│        ▼                                                                     │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                      API Gateway Layer                               │   │
│   │  ┌───────────────────────────────────────────────────────────────┐  │   │
│   │  │                    Kong / AWS API GW                          │  │   │
│   │  │                                                                │  │   │
│   │  │  담당 기능:                                                    │  │   │
│   │  │  • 외부 인증 (OAuth 2.0, API Key, JWT)                        │  │   │
│   │  │  • API Rate Limiting (Client별, API별)                        │  │   │
│   │  │  • API 버전 관리 (/v1, /v2)                                   │  │   │
│   │  │  • 요청/응답 변환 (프로토콜 변환, 필드 매핑)                    │  │   │
│   │  │  • DDoS 방어                                                   │  │   │
│   │  │  • 캐싱 (응답 캐시)                                            │  │   │
│   │  │                                                                │  │   │
│   │  └───────────────────────────────────────────────────────────────┘  │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                         │                                                    │
│                         ▼                                                    │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                     Kubernetes Cluster                               │   │
│   │  ┌───────────────────────────────────────────────────────────────┐  │   │
│   │  │                 Istio Ingress Gateway                         │  │   │
│   │  │           (API Gateway에서 전달된 요청 수신)                   │  │   │
│   │  │           (Mesh 내부 라우팅 규칙 적용)                        │  │   │
│   │  └───────────────────────────────────────────────────────────────┘  │   │
│   │                         │                                           │   │
│   │  ┌──────────────────────┼──────────────────────────────────────┐   │   │
│   │  │                Service Mesh Layer (Istio)                    │   │   │
│   │  │                                                              │   │   │
│   │  │  담당 기능:                                                  │   │   │
│   │  │  • 서비스 간 mTLS (자동 암호화)                              │   │   │
│   │  │  • 분산 추적 (Jaeger/Zipkin)                                 │   │   │
│   │  │  • 서킷 브레이커                                             │   │   │
│   │  │  • 재시도/타임아웃                                           │   │   │
│   │  │  • 카나리 배포                                               │   │   │
│   │  │  • 서비스 간 인가 (AuthorizationPolicy)                      │   │   │
│   │  │                                                              │   │   │
│   │  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │   │   │
│   │  │  │   Pod + Envoy │  │   Pod + Envoy │  │   Pod + Envoy │       │   │   │
│   │  │  │              │  │              │  │              │       │   │   │
│   │  │  │  User Svc   │◄─►│  Order Svc  │◄─►│ Payment Svc │       │   │   │
│   │  │  │              │  │              │  │              │       │   │   │
│   │  │  └──────────────┘  └──────────────┘  └──────────────┘       │   │   │
│   │  │         ↑                  ↑                 ↑               │   │   │
│   │  │         └───────── mTLS 암호화 통신 ──────────┘               │   │   │
│   │  │                                                              │   │   │
│   │  └──────────────────────────────────────────────────────────────┘   │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│   역할 분담 요약:                                                            │
│   ─────────────────────────────────────────────────────────────────────────  │
│   • API Gateway: 외부 클라이언트 관리, API 수준 정책                         │
│   • Istio Ingress: 클러스터 진입점, 외부-내부 연결                           │
│   • Service Mesh: 내부 서비스 간 통신 관리, Zero Trust                       │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 선택 가이드

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      언제 무엇을 선택할 것인가?                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  API Gateway만 필요한 경우:                                                  │
│  ───────────────────────────────────────────────────────────────────────    │
│  • 외부 API 노출이 주 목적                                                   │
│  • 마이크로서비스 수가 적음 (< 10개)                                         │
│  • 서비스 간 통신 복잡도가 낮음                                              │
│  • 모놀리식 또는 초기 마이크로서비스 단계                                     │
│                                                                              │
│  Service Mesh만 필요한 경우:                                                 │
│  ───────────────────────────────────────────────────────────────────────    │
│  • 내부 서비스 간 통신 관리가 주 목적                                        │
│  • 외부 API 노출이 거의 없음                                                 │
│  • 내부 보안(mTLS)이 중요                                                    │
│                                                                              │
│  둘 다 필요한 경우 (권장):                                                   │
│  ───────────────────────────────────────────────────────────────────────    │
│  • 마이크로서비스 수가 많음 (> 10개)                                         │
│  • 외부 API + 복잡한 내부 통신                                               │
│  • 엔터프라이즈 환경                                                         │
│  • Zero Trust 보안 모델 필요                                                 │
│  • 분산 추적, 관측성 필요                                                    │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                        결정 플로우차트                               │    │
│  │                                                                     │    │
│  │  외부 API 노출? ───Yes──► API Gateway 필요                          │    │
│  │       │                                                             │    │
│  │      No                                                             │    │
│  │       │                                                             │    │
│  │       ▼                                                             │    │
│  │  서비스 > 10개? ───Yes──► Service Mesh 고려                         │    │
│  │       │                                                             │    │
│  │      No                                                             │    │
│  │       │                                                             │    │
│  │       ▼                                                             │    │
│  │  mTLS/관측성 필요? ──Yes──► Service Mesh 필요                       │    │
│  │       │                                                             │    │
│  │      No                                                             │    │
│  │       │                                                             │    │
│  │       ▼                                                             │    │
│  │  단순 로드밸런서로 충분                                              │    │
│  │                                                                     │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 통합 설정 예제

```yaml
# 1. API Gateway (Kong) 설정 - 외부 API 관리
apiVersion: configuration.konghq.com/v1
kind: KongIngress
metadata:
  name: external-api-config
route:
  protocols:
    - https
  strip_path: true
  request_buffering: true

---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-gateway-ingress
  annotations:
    konghq.com/override: external-api-config
    konghq.com/plugins: rate-limiting,jwt-auth,cors
spec:
  ingressClassName: kong
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /v1/orders
            pathType: Prefix
            backend:
              service:
                name: order-service
                port:
                  number: 80

---
# Kong Rate Limiting 플러그인
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: rate-limiting
plugin: rate-limiting
config:
  minute: 100
  policy: local

---
# 2. Istio Service Mesh 설정 - 내부 통신 관리

# VirtualService: 내부 라우팅
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: order-service-internal
  namespace: production
spec:
  hosts:
    - order-service
  http:
    - route:
        - destination:
            host: order-service
            subset: v1
          weight: 90
        - destination:
            host: order-service
            subset: v2
          weight: 10
      retries:
        attempts: 3
        perTryTimeout: 2s
      timeout: 10s

---
# PeerAuthentication: mTLS 강제
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  mtls:
    mode: STRICT

---
# AuthorizationPolicy: 서비스 간 접근 제어
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: order-to-payment
  namespace: production
spec:
  selector:
    matchLabels:
      app: payment-service
  action: ALLOW
  rules:
    - from:
        - source:
            principals:
              - "cluster.local/ns/production/sa/order-service"
      to:
        - operation:
            methods: ["POST"]
            paths: ["/api/payments/*"]
```

---

## 실무 적용 가이드

### 도입 단계별 가이드

```
┌─────────────────────────────────────────────────────────────────────┐
│                   Service Mesh 도입 단계                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Phase 1: 관측성 먼저 (2-4주)                                        │
│  ─────────────────────────────────────────────────────────────────  │
│  • Sidecar 주입만 활성화                                             │
│  • mTLS는 PERMISSIVE 모드 (기존 트래픽 영향 없음)                    │
│  • 메트릭, 로그, 트레이싱 수집                                       │
│  • 서비스 간 의존성 파악                                             │
│                                                                      │
│  Phase 2: 트래픽 관리 (2-4주)                                        │
│  ─────────────────────────────────────────────────────────────────  │
│  • VirtualService, DestinationRule 설정                             │
│  • 타임아웃, 재시도 정책                                             │
│  • 카나리 배포 파이프라인 구축                                        │
│                                                                      │
│  Phase 3: 보안 강화 (2-4주)                                          │
│  ─────────────────────────────────────────────────────────────────  │
│  • mTLS STRICT 모드 전환                                            │
│  • AuthorizationPolicy로 접근 제어                                   │
│  • 외부 인증 (JWT) 통합                                              │
│                                                                      │
│  Phase 4: 고급 기능 (지속)                                           │
│  ─────────────────────────────────────────────────────────────────  │
│  • Circuit Breaker, Rate Limiting                                   │
│  • 멀티 클러스터 메시                                                │
│  • Wasm 확장                                                        │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 성능 최적화

```yaml
# Sidecar 리소스 최적화
apiVersion: apps/v1
kind: Deployment
metadata:
  name: high-traffic-service
spec:
  template:
    metadata:
      annotations:
        # Sidecar 리소스 제한
        sidecar.istio.io/proxyCPU: "100m"
        sidecar.istio.io/proxyMemory: "128Mi"
        sidecar.istio.io/proxyCPULimit: "2000m"
        sidecar.istio.io/proxyMemoryLimit: "1Gi"

        # Envoy 동시성 설정
        proxy.istio.io/config: |
          concurrency: 2

        # 불필요한 트래픽 제외
        traffic.sidecar.istio.io/excludeOutboundPorts: "3306,6379"

---
# Mesh 전역 설정 최적화
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  meshConfig:
    # 액세스 로그 비활성화 (프로덕션에서 성능 향상)
    accessLogFile: ""

    # 프로토콜 자동 감지 비활성화 (명시적 지정 시)
    protocolDetectionTimeout: 0s

    # DNS 프록시 활성화 (성능 향상)
    defaultConfig:
      proxyMetadata:
        ISTIO_META_DNS_CAPTURE: "true"
        ISTIO_META_DNS_AUTO_ALLOCATE: "true"
```

### 트러블슈팅

```bash
# Sidecar 주입 확인
kubectl get pods -n my-app -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[*].name}{"\n"}{end}'

# Envoy 설정 확인
istioctl proxy-config all <pod-name> -n <namespace>
istioctl proxy-config listeners <pod-name> -n <namespace>
istioctl proxy-config routes <pod-name> -n <namespace>
istioctl proxy-config clusters <pod-name> -n <namespace>

# Envoy 상태 확인
kubectl exec <pod-name> -c istio-proxy -n <namespace> -- curl localhost:15000/stats

# mTLS 상태 확인
istioctl authn tls-check <pod-name> -n <namespace>

# 서비스 간 연결 분석
istioctl analyze -n my-app

# Kiali에서 서비스 그래프 확인
istioctl dashboard kiali

# 트래픽 스니핑 (디버깅)
kubectl exec <pod-name> -c istio-proxy -n <namespace> -- \
  curl localhost:15000/config_dump > envoy_config.json
```

---

## 참고 자료

- [Istio Documentation](https://istio.io/latest/docs/)
- [Linkerd Documentation](https://linkerd.io/docs/)
- [Envoy Proxy](https://www.envoyproxy.io/docs/envoy/latest/)
- [Service Mesh Performance Comparison - CNCF](https://deepness-lab.org/wp-content/uploads/2024/05/Service_Mesh_Performance_Project_Report.pdf)
- [Istio vs Linkerd - Buoyant](https://www.buoyant.io/linkerd-vs-istio)
- [Service Mesh Traffic Management Guide 2024 - Endgrate](https://endgrate.com/blog/service-mesh-traffic-management-guide-2024)
- [CNCF Service Mesh Landscape](https://landscape.cncf.io/card-mode?category=service-mesh)
- William Morgan, "The Service Mesh: What Every Software Engineer Needs to Know"

---

*마지막 업데이트: 2025년 1월*
