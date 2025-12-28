# CD (Continuous Deployment) 배포 전략

## 목차
1. [Blue-Green Deployment](#1-blue-green-deployment)
2. [Canary Deployment](#2-canary-deployment)
3. [Rolling Update](#3-rolling-update)
4. [A/B Testing](#4-ab-testing)
5. [Feature Flag](#5-feature-flag)
6. [롤백 전략](#6-롤백-전략)
7. [배포 전략 비교 및 선택 가이드](#7-배포-전략-비교-및-선택-가이드)
---

## 1. Blue-Green Deployment

### 1.1 개념

**Blue-Green Deployment**는 두 개의 동일한 프로덕션 환경(Blue와 Green)을 유지하면서, 새로운 버전을 비활성 환경에 배포하고 트래픽을 전환하는 배포 전략이다.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      Blue-Green Deployment 흐름                          │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  Phase 1: 초기 상태                                                      │
│  ┌─────────────┐         ┌─────────────────────────────────────────┐    │
│  │    Users    │────────▶│  Load Balancer                         │    │
│  └─────────────┘         └─────────────┬───────────────────────────┘    │
│                                        │                                 │
│                                        ▼                                 │
│                          ┌─────────────────────────┐                    │
│                          │   Blue Environment      │ ◀── 현재 운영      │
│                          │      (v1.0.0)           │                    │
│                          └─────────────────────────┘                    │
│                          ┌─────────────────────────┐                    │
│                          │   Green Environment     │ ◀── 비활성         │
│                          │      (v1.0.0)           │                    │
│                          └─────────────────────────┘                    │
│                                                                          │
│  Phase 2: 새 버전 배포                                                   │
│                          ┌─────────────────────────┐                    │
│                          │   Blue Environment      │ ◀── 트래픽 처리    │
│                          │      (v1.0.0)           │                    │
│                          └─────────────────────────┘                    │
│                          ┌─────────────────────────┐                    │
│                          │   Green Environment     │ ◀── 새 버전 배포   │
│                          │      (v1.1.0)           │     + 테스트       │
│                          └─────────────────────────┘                    │
│                                                                          │
│  Phase 3: 트래픽 전환                                                    │
│                                        │                                 │
│                                        ▼                                 │
│                          ┌─────────────────────────┐                    │
│                          │   Blue Environment      │ ◀── 대기 (롤백용)  │
│                          │      (v1.0.0)           │                    │
│                          └─────────────────────────┘                    │
│                          ┌─────────────────────────┐                    │
│                          │   Green Environment     │ ◀── 운영 중        │
│                          │      (v1.1.0)           │                    │
│                          └─────────────────────────┘                    │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 1.2 장점과 단점

| 구분 | 내용 |
|------|------|
| **장점** | - **Zero Downtime**: 트래픽 전환 시 다운타임 없음 |
|          | - **즉시 롤백**: 문제 발생 시 이전 환경으로 즉시 전환 |
|          | - **완전한 테스트**: 프로덕션과 동일한 환경에서 테스트 가능 |
|          | - **간단한 트래픽 관리**: 100% 전환으로 복잡한 라우팅 불필요 |
| **단점** | - **비용**: 두 배의 인프라 리소스 필요 |
|          | - **DB 동기화**: 데이터베이스 스키마 변경 시 복잡성 증가 |
|          | - **세션 관리**: Stateful 애플리케이션에서 세션 유실 가능 |

### 1.3 Kubernetes에서 Blue-Green 구현

```yaml
# blue-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-blue
  labels:
    app: myapp
    version: blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: blue
  template:
    metadata:
      labels:
        app: myapp
        version: blue
    spec:
      containers:
      - name: myapp
        image: myregistry/myapp:1.0.0
        ports:
        - containerPort: 8080
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 15
          periodSeconds: 20
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"

---
# green-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-green
  labels:
    app: myapp
    version: green
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: green
  template:
    metadata:
      labels:
        app: myapp
        version: green
    spec:
      containers:
      - name: myapp
        image: myregistry/myapp:1.1.0
        ports:
        - containerPort: 8080
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 10

---
# service.yaml - 트래픽 전환을 위한 Service
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  selector:
    app: myapp
    version: blue   # green으로 변경하여 트래픽 전환
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8080
  type: LoadBalancer
```

### 1.4 트래픽 전환 스크립트

```bash
#!/bin/bash
# blue-green-switch.sh

NAMESPACE=${1:-default}
CURRENT_VERSION=$(kubectl get svc myapp-service -n $NAMESPACE -o jsonpath='{.spec.selector.version}')

if [ "$CURRENT_VERSION" == "blue" ]; then
    NEW_VERSION="green"
else
    NEW_VERSION="blue"
fi

echo "현재 버전: $CURRENT_VERSION"
echo "전환 대상: $NEW_VERSION"

# 새 버전 준비 상태 확인
READY_REPLICAS=$(kubectl get deployment myapp-$NEW_VERSION -n $NAMESPACE -o jsonpath='{.status.readyReplicas}')
DESIRED_REPLICAS=$(kubectl get deployment myapp-$NEW_VERSION -n $NAMESPACE -o jsonpath='{.spec.replicas}')

if [ "$READY_REPLICAS" != "$DESIRED_REPLICAS" ]; then
    echo "오류: $NEW_VERSION 환경이 준비되지 않음 ($READY_REPLICAS/$DESIRED_REPLICAS)"
    exit 1
fi

# 트래픽 전환
kubectl patch svc myapp-service -n $NAMESPACE \
    -p "{\"spec\":{\"selector\":{\"version\":\"$NEW_VERSION\"}}}"

echo "트래픽이 $NEW_VERSION으로 전환되었습니다."
```

---

## 2. Canary Deployment

### 2.1 개념

**Canary Deployment**는 새로운 버전을 일부 사용자(예: 5%)에게만 먼저 배포하고, 모니터링을 통해 문제가 없으면 점진적으로 전체 사용자에게 확대하는 배포 전략이다.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      Canary Deployment 진행 과정                         │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  100% ─┬───────────────────────────────────────────────────────────┬─   │
│        │  ■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■  │    │
│   95%  │  ■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■     │    │
│        │  ■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■     │    │
│   75%  │  ■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■              │    │
│        │  ■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■               │    │
│   50%  │  ■■■■■■■■■■■■■■■■■■■■■■■■■■■                             │    │
│        │  ■■■■■■■■■■■■■■■■■■■■■■■■■■■                              │    │
│   25%  │  ■■■■■■■■■■■■■■                                          │    │
│        │  ■■■■■■■■■■■■■■                                           │    │
│    5%  │  ■■■                                                      │    │
│        │  ■■■                                                       │    │
│    0% ─┴───────────────────────────────────────────────────────────┴─   │
│        Phase 1   Phase 2    Phase 3    Phase 4    Phase 5               │
│                                                                          │
│  ■ v1.0.0 (Stable)     ■ v1.1.0 (Canary)                                │
│                                                                          │
│  각 Phase에서 모니터링:                                                   │
│  • Error Rate                                                            │
│  • Latency (p50, p95, p99)                                              │
│  • CPU/Memory Usage                                                      │
│  • Business Metrics                                                      │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.2 장점과 단점

| 구분 | 내용 |
|------|------|
| **장점** | - **위험 최소화**: 일부 사용자에게만 영향 |
|          | - **실제 트래픽 테스트**: 프로덕션 환경에서 검증 |
|          | - **점진적 롤아웃**: 문제 발생 시 영향 범위 제한 |
|          | - **비용 효율적**: Blue-Green 대비 적은 리소스 |
| **단점** | - **복잡한 모니터링**: 세밀한 관찰 필요 |
|          | - **트래픽 분산 복잡성**: 정교한 라우팅 구현 필요 |
|          | - **롤백 시간**: 즉각적인 롤백이 어려울 수 있음 |

### 2.3 Argo Rollouts를 활용한 Canary 구현

```yaml
# canary-rollout.yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: myapp-rollout
spec:
  replicas: 10
  strategy:
    canary:
      # Canary 단계 정의
      steps:
      - setWeight: 5
      - pause: {duration: 5m}  # 5분 대기 (모니터링)
      - setWeight: 20
      - pause: {duration: 10m}
      - setWeight: 50
      - pause: {duration: 10m}
      - setWeight: 80
      - pause: {duration: 5m}
      # 100%로 자동 전환

      # 트래픽 관리 (Istio)
      trafficRouting:
        istio:
          virtualService:
            name: myapp-vsvc
            routes:
            - primary

      # 자동 롤백 조건
      analysis:
        templates:
        - templateName: success-rate
        startingStep: 1
        args:
        - name: service-name
          value: myapp

  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: myregistry/myapp:1.1.0
        ports:
        - containerPort: 8080

---
# 분석 템플릿 - Prometheus 메트릭 기반
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate
spec:
  args:
  - name: service-name
  metrics:
  - name: success-rate
    interval: 1m
    count: 5
    successCondition: result[0] >= 0.95
    failureCondition: result[0] < 0.90
    failureLimit: 3
    provider:
      prometheus:
        address: http://prometheus.monitoring:9090
        query: |
          sum(rate(
            http_requests_total{
              service="{{args.service-name}}",
              status=~"2.."
            }[5m]
          )) /
          sum(rate(
            http_requests_total{
              service="{{args.service-name}}"
            }[5m]
          ))
```

### 2.4 Nginx Ingress를 활용한 간단한 Canary

```yaml
# stable-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-stable
spec:
  ingressClassName: nginx
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapp-stable
            port:
              number: 80

---
# canary-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-canary
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "10"  # 10% 트래픽
spec:
  ingressClassName: nginx
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapp-canary
            port:
              number: 80
```

---

## 3. Rolling Update

### 3.1 개념

**Rolling Update**는 기존 인스턴스를 점진적으로 새 버전으로 교체하는 배포 전략이다. Kubernetes의 기본 배포 전략이며, 추가 인프라 없이 무중단 배포가 가능하다.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      Rolling Update 진행 과정                            │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  Step 1: 초기 상태 (4 replicas)                                          │
│  ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐                                        │
│  │v1.0 │ │v1.0 │ │v1.0 │ │v1.0 │                                        │
│  └─────┘ └─────┘ └─────┘ └─────┘                                        │
│                                                                          │
│  Step 2: 새 Pod 생성 + 구 Pod 종료                                       │
│  ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐                                        │
│  │v1.1 │ │v1.0 │ │v1.0 │ │v1.0 │  maxSurge: 1                           │
│  └─────┘ └─────┘ └─────┘ └─────┘  maxUnavailable: 1                     │
│     ↑                       ↓                                            │
│   생성중                  종료중                                          │
│                                                                          │
│  Step 3: 계속 진행                                                       │
│  ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐                                        │
│  │v1.1 │ │v1.1 │ │v1.0 │ │v1.0 │                                        │
│  └─────┘ └─────┘ └─────┘ └─────┘                                        │
│                     ↑       ↓                                            │
│                   생성중  종료중                                          │
│                                                                          │
│  Step 4: 완료                                                            │
│  ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐                                        │
│  │v1.1 │ │v1.1 │ │v1.1 │ │v1.1 │                                        │
│  └─────┘ └─────┘ └─────┘ └─────┘                                        │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 3.2 장점과 단점

| 구분 | 내용 |
|------|------|
| **장점** | - **추가 인프라 불필요**: 기존 리소스 내에서 배포 |
|          | - **무중단 배포**: 점진적 교체로 서비스 연속성 유지 |
|          | - **Kubernetes 기본 지원**: 별도 설정 없이 사용 가능 |
|          | - **리소스 효율적**: Blue-Green 대비 저비용 |
| **단점** | - **배포 시간**: 대규모 환경에서 시간 소요 |
|          | - **버전 혼재**: 배포 중 두 버전이 동시 운영 |
|          | - **롤백 복잡**: 중간 실패 시 롤백에 시간 소요 |
|          | - **호환성 필수**: 신/구 버전 간 호환성 필요 |

### 3.3 Kubernetes Rolling Update 설정

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 6
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2        # 최대 추가 Pod 수 (또는 %)
      maxUnavailable: 1  # 최대 비가용 Pod 수 (또는 %)
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: myregistry/myapp:1.1.0
        ports:
        - containerPort: 8080

        # Readiness Probe - 트래픽 수신 준비 확인
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
          failureThreshold: 3

        # Liveness Probe - 컨테이너 생존 확인
        livenessProbe:
          httpGet:
            path: /health/live
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10

        # Startup Probe - 초기 시작 시간이 긴 앱용
        startupProbe:
          httpGet:
            path: /health/ready
            port: 8080
          failureThreshold: 30
          periodSeconds: 10

        # 리소스 관리
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"

      # Graceful Shutdown
      terminationGracePeriodSeconds: 60
```

### 3.4 Rolling Update 제어 명령어

```bash
# 배포 상태 확인
kubectl rollout status deployment/myapp

# 배포 일시 중지
kubectl rollout pause deployment/myapp

# 배포 재개
kubectl rollout resume deployment/myapp

# 이전 버전으로 롤백
kubectl rollout undo deployment/myapp

# 특정 리비전으로 롤백
kubectl rollout undo deployment/myapp --to-revision=2

# 배포 히스토리 확인
kubectl rollout history deployment/myapp
```

---

## 4. A/B Testing

### 4.1 개념

**A/B Testing**은 두 가지(또는 그 이상) 버전의 기능을 서로 다른 사용자 그룹에게 제공하여 성능, 사용성, 전환율 등을 비교 분석하는 실험 기법이다.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          A/B Testing 구조                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│                        ┌─────────────┐                                  │
│                        │   Traffic   │                                  │
│                        │   Router    │                                  │
│                        └──────┬──────┘                                  │
│                               │                                          │
│                    ┌──────────┴──────────┐                              │
│                    │  User Segmentation  │                              │
│                    │  (Cookie, Header,   │                              │
│                    │   User ID, etc.)    │                              │
│                    └──────────┬──────────┘                              │
│                    ┌──────────┴──────────┐                              │
│                    ▼                     ▼                              │
│           ┌─────────────┐        ┌─────────────┐                        │
│           │  Version A  │        │  Version B  │                        │
│           │  (Control)  │        │  (Variant)  │                        │
│           │             │        │             │                        │
│           │ 기존 버튼   │        │ 새 버튼     │                        │
│           │ 색상: 파랑  │        │ 색상: 초록  │                        │
│           └──────┬──────┘        └──────┬──────┘                        │
│                  │                      │                                │
│                  ▼                      ▼                                │
│           ┌─────────────────────────────────────┐                       │
│           │         Analytics Platform          │                       │
│           │  • Conversion Rate                  │                       │
│           │  • Click-through Rate               │                       │
│           │  • Bounce Rate                      │                       │
│           │  • Revenue per User                 │                       │
│           └─────────────────────────────────────┘                       │
│                                                                          │
│  A/B Test와 Canary 배포의 차이:                                          │
│  • Canary: 기술적 안정성 검증 (에러율, 레이턴시)                          │
│  • A/B Test: 비즈니스 지표 측정 (전환율, 매출)                           │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 4.2 A/B Testing vs Canary Deployment

| 구분 | A/B Testing | Canary Deployment |
|------|-------------|-------------------|
| **목적** | 비즈니스 지표 최적화 | 기술적 안정성 검증 |
| **분할 기준** | 사용자 세그먼트 | 트래픽 비율 |
| **측정 지표** | 전환율, CTR, 매출 | 에러율, 레이턴시, 리소스 |
| **기간** | 통계적 유의성 확보까지 | 안정성 확인까지 |
| **롤백** | 실험 종료 (낮은 성과 버전 제거) | 문제 감지 시 즉시 롤백 |

### 4.3 Istio를 활용한 A/B Testing

```yaml
# VirtualService - 헤더 기반 라우팅
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: myapp-ab-test
spec:
  hosts:
  - myapp.example.com
  http:
  # Beta 사용자 그룹 - Version B로 라우팅
  - match:
    - headers:
        x-user-group:
          exact: "beta"
    route:
    - destination:
        host: myapp-v2
        port:
          number: 80
      weight: 100

  # 특정 사용자 ID 기반 라우팅 (해시)
  - match:
    - headers:
        x-user-id:
          regex: ".*[0-4]$"  # 마지막 숫자 0-4: Version A
    route:
    - destination:
        host: myapp-v1
        port:
          number: 80

  - match:
    - headers:
        x-user-id:
          regex: ".*[5-9]$"  # 마지막 숫자 5-9: Version B
    route:
    - destination:
        host: myapp-v2
        port:
          number: 80

  # 기본 라우팅 (Version A)
  - route:
    - destination:
        host: myapp-v1
        port:
          number: 80
```

### 4.4 A/B Testing 구현 예시 (Spring Boot)

```java
@RestController
@RequestMapping("/api/checkout")
public class CheckoutController {

    private final ABTestService abTestService;
    private final AnalyticsService analyticsService;

    @PostMapping
    public ResponseEntity<CheckoutResponse> checkout(
            @RequestHeader("X-User-ID") String userId,
            @RequestBody CheckoutRequest request) {

        // A/B 테스트 그룹 결정
        String variant = abTestService.getVariant(
            "checkout-button-experiment",
            userId
        );

        CheckoutResponse response;

        if ("variant-b".equals(variant)) {
            // Version B: 새로운 결제 플로우
            response = checkoutWithNewFlow(request);
        } else {
            // Version A (Control): 기존 결제 플로우
            response = checkoutWithOriginalFlow(request);
        }

        // 분석 데이터 기록
        analyticsService.trackEvent(AnalyticsEvent.builder()
            .userId(userId)
            .experiment("checkout-button-experiment")
            .variant(variant)
            .action("checkout_completed")
            .revenue(response.getTotalAmount())
            .build());

        return ResponseEntity.ok(response);
    }
}

@Service
public class ABTestService {

    private final FeatureFlagClient featureFlagClient;

    /**
     * 사용자를 일관되게 같은 그룹에 할당
     * (세션 간에도 동일한 경험 보장)
     */
    public String getVariant(String experimentId, String userId) {
        // 해시 기반 그룹 할당
        int hash = Math.abs((experimentId + userId).hashCode());
        int bucket = hash % 100;

        ExperimentConfig config = featureFlagClient
            .getExperiment(experimentId);

        // 예: 50% A, 50% B
        if (bucket < config.getVariantAPercentage()) {
            return "control";
        } else {
            return "variant-b";
        }
    }
}
```

---

## 5. Feature Flag

### 5.1 개념

**Feature Flag (기능 플래그)**는 코드 배포와 기능 릴리스를 분리하여, 런타임에 특정 기능을 켜거나 끌 수 있게 하는 기법이다.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                       Feature Flag 활용 패턴                             │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  1. Release Toggle (릴리스 토글)                                         │
│     ┌────────────────────────────────────────────────────────────────┐  │
│     │  if (featureFlags.isEnabled("new-checkout")) {                 │  │
│     │      return newCheckoutFlow();                                 │  │
│     │  } else {                                                      │  │
│     │      return legacyCheckoutFlow();                              │  │
│     │  }                                                             │  │
│     └────────────────────────────────────────────────────────────────┘  │
│     용도: 미완성 기능 숨기기, 점진적 롤아웃                               │
│                                                                          │
│  2. Experiment Toggle (실험 토글)                                        │
│     ┌────────────────────────────────────────────────────────────────┐  │
│     │  String variant = featureFlags.getVariant("pricing-test",      │  │
│     │                                            userId);            │  │
│     │  switch (variant) {                                            │  │
│     │      case "control" -> showOriginalPrice();                    │  │
│     │      case "variant-a" -> showDiscountedPrice();                │  │
│     │      case "variant-b" -> showSubscriptionPrice();              │  │
│     │  }                                                             │  │
│     └────────────────────────────────────────────────────────────────┘  │
│     용도: A/B 테스트, 다변량 테스트                                       │
│                                                                          │
│  3. Ops Toggle (운영 토글)                                               │
│     ┌────────────────────────────────────────────────────────────────┐  │
│     │  if (featureFlags.isEnabled("external-payment-gateway")) {     │  │
│     │      return externalGateway.process(payment);                  │  │
│     │  } else {                                                      │  │
│     │      return fallbackProcess(payment);                          │  │
│     │  }                                                             │  │
│     └────────────────────────────────────────────────────────────────┘  │
│     용도: 서킷 브레이커, 기능 비활성화 (장애 대응)                         │
│                                                                          │
│  4. Permission Toggle (권한 토글)                                        │
│     ┌────────────────────────────────────────────────────────────────┐  │
│     │  if (featureFlags.isEnabledFor("beta-features", user)) {       │  │
│     │      showBetaFeatures();                                       │  │
│     │  }                                                             │  │
│     └────────────────────────────────────────────────────────────────┘  │
│     용도: 베타 사용자, 유료 기능, 내부 사용자 전용                         │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 5.2 Feature Flag 도구 비교

| 도구 | 타입 | 특징 |
|------|------|------|
| **LaunchDarkly** | SaaS | 실시간 업데이트, 상세 타겟팅, A/B 테스트 |
| **Unleash** | 오픈소스/SaaS | 셀프 호스팅 가능, 활성화 전략 |
| **Flagsmith** | 오픈소스/SaaS | Remote Config 지원, 세그먼트 |
| **ConfigCat** | SaaS | 간단한 설정, 빠른 시작 |
| **Split** | SaaS | Feature Experimentation, 통계 분석 |

### 5.3 Feature Flag 구현 예시

```java
// Feature Flag 인터페이스
public interface FeatureFlagService {

    boolean isEnabled(String flagKey);

    boolean isEnabledForUser(String flagKey, String userId);

    boolean isEnabledForContext(String flagKey, EvaluationContext context);

    String getVariant(String flagKey, String userId);
}

// Unleash 기반 구현
@Service
@RequiredArgsConstructor
public class UnleashFeatureFlagService implements FeatureFlagService {

    private final Unleash unleash;

    @Override
    public boolean isEnabled(String flagKey) {
        return unleash.isEnabled(flagKey);
    }

    @Override
    public boolean isEnabledForUser(String flagKey, String userId) {
        UnleashContext context = UnleashContext.builder()
            .userId(userId)
            .build();
        return unleash.isEnabled(flagKey, context);
    }

    @Override
    public boolean isEnabledForContext(String flagKey, EvaluationContext context) {
        UnleashContext unleashContext = UnleashContext.builder()
            .userId(context.getUserId())
            .sessionId(context.getSessionId())
            .remoteAddress(context.getIpAddress())
            .addProperty("country", context.getCountry())
            .addProperty("subscription", context.getSubscriptionTier())
            .build();
        return unleash.isEnabled(flagKey, unleashContext);
    }

    @Override
    public String getVariant(String flagKey, String userId) {
        UnleashContext context = UnleashContext.builder()
            .userId(userId)
            .build();
        Variant variant = unleash.getVariant(flagKey, context);
        return variant.getName();
    }
}

// 비즈니스 로직에서 사용
@Service
@RequiredArgsConstructor
public class ProductService {

    private final FeatureFlagService featureFlags;
    private final ProductRepository productRepository;

    public List<ProductDto> getProducts(String userId) {
        List<Product> products = productRepository.findAll();

        // Feature Flag: 새로운 추천 알고리즘
        if (featureFlags.isEnabledForUser("new-recommendation-engine", userId)) {
            return applyNewRecommendation(products, userId);
        }

        return convertToDto(products);
    }

    public ProductDetailDto getProductDetail(Long productId, String userId) {
        Product product = productRepository.findById(productId)
            .orElseThrow(() -> new ProductNotFoundException(productId));

        ProductDetailDto dto = ProductDetailDto.from(product);

        // Feature Flag: 리뷰 요약 기능 (AI)
        if (featureFlags.isEnabled("ai-review-summary")) {
            dto.setReviewSummary(aiService.summarizeReviews(productId));
        }

        // Feature Flag: 가격 비교 기능 (점진적 롤아웃)
        EvaluationContext context = EvaluationContext.builder()
            .userId(userId)
            .country(getCountryFromUser(userId))
            .build();

        if (featureFlags.isEnabledForContext("price-comparison", context)) {
            dto.setPriceComparison(getPriceComparison(productId));
        }

        return dto;
    }
}
```

### 5.4 Feature Flag 베스트 프랙티스

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Feature Flag 12 계명 (2025)                           │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  1. 명확한 명명 규칙 사용                                                 │
│     ✓ "enable-new-checkout-flow"                                        │
│     ✗ "flag1", "test123"                                                │
│                                                                          │
│  2. 수명주기 정의                                                        │
│     • Release Toggle: 2주 이내 제거                                      │
│     • Experiment Toggle: 실험 종료 후 제거                               │
│     • Ops Toggle: 필요시 유지                                            │
│                                                                          │
│  3. 기술 부채 관리                                                       │
│     • 정기적으로 사용하지 않는 플래그 정리                                │
│     • 플래그 소유자 지정                                                 │
│                                                                          │
│  4. 기본값은 안전하게                                                    │
│     • 플래그 서버 장애 시 기본값 = 안전한 동작                            │
│     • 새 기능은 기본 OFF                                                 │
│                                                                          │
│  5. 테스트 커버리지                                                      │
│     • 플래그 ON/OFF 양쪽 모두 테스트                                     │
│     • 플래그 조합 테스트                                                 │
│                                                                          │
│  6. 모니터링 및 알림                                                     │
│     • 플래그 변경 시 알림                                                │
│     • 플래그별 성능 메트릭 추적                                          │
│                                                                          │
│  7. 점진적 롤아웃                                                        │
│     • 1% → 5% → 25% → 50% → 100%                                        │
│     • 각 단계에서 메트릭 검증                                            │
│                                                                          │
│  8. 킬 스위치 준비                                                       │
│     • 중요 기능은 언제든 비활성화 가능하도록                              │
│                                                                          │
│  9. 로깅 및 감사                                                         │
│     • 플래그 평가 결과 로깅                                              │
│     • 변경 이력 추적                                                     │
│                                                                          │
│  10. 환경별 관리                                                         │
│      • dev/staging/prod 별도 설정                                        │
│                                                                          │
│  11. CI/CD 통합                                                          │
│      • 플래그 상태를 배포 파이프라인에 통합                               │
│                                                                          │
│  12. 문서화                                                              │
│      • 플래그 목적, 소유자, 만료일 명시                                   │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 6. 롤백 전략

### 6.1 롤백 유형

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          롤백 전략 비교                                   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  1. 즉시 롤백 (Instant Rollback)                                         │
│     ┌─────────┐         ┌─────────┐                                     │
│     │ v1.1.0  │ ──────▶ │ v1.0.0  │  • Blue-Green 전환                  │
│     │ (문제)  │ 즉시    │ (안정)  │  • 트래픽 즉시 이동                  │
│     └─────────┘         └─────────┘  • 다운타임: 수초                   │
│                                                                          │
│  2. 점진적 롤백 (Gradual Rollback)                                       │
│     ┌─────────┐         ┌─────────┐                                     │
│     │ v1.1.0  │ ──────▶ │ v1.0.0  │  • Canary 역방향                    │
│     │  90%    │  ─▶     │  100%   │  • 점진적 트래픽 이동               │
│     └─────────┘         └─────────┘  • 영향 최소화                      │
│                                                                          │
│  3. 재배포 롤백 (Redeploy Rollback)                                      │
│     ┌─────────┐         ┌─────────┐                                     │
│     │ v1.1.0  │ ──────▶ │ v1.0.0  │  • 이전 버전 재빌드/배포            │
│     │ (삭제)  │ 배포    │ (배포)  │  • 시간 소요 (분 단위)              │
│     └─────────┘         └─────────┘  • Rolling Update 방식              │
│                                                                          │
│  4. 기능 롤백 (Feature Rollback)                                         │
│     ┌─────────┐                                                         │
│     │ v1.1.0  │  Feature Flag OFF  • 코드 변경 없음                     │
│     │ 기능OFF │ ◀────────────────  • 즉시 적용                          │
│     └─────────┘                     • 가장 빠른 대응                     │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 6.2 Kubernetes 롤백 명령

```bash
# 롤백 히스토리 확인
kubectl rollout history deployment/myapp

# 출력 예시:
# REVISION  CHANGE-CAUSE
# 1         Initial deployment
# 2         Update to v1.1.0
# 3         Update to v1.2.0 (current)

# 이전 버전으로 롤백
kubectl rollout undo deployment/myapp

# 특정 리비전으로 롤백
kubectl rollout undo deployment/myapp --to-revision=1

# 롤백 상태 확인
kubectl rollout status deployment/myapp

# 현재 이미지 확인
kubectl get deployment myapp -o jsonpath='{.spec.template.spec.containers[0].image}'
```

### 6.3 자동 롤백 설정 (Argo Rollouts)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: myapp
spec:
  replicas: 5
  strategy:
    canary:
      steps:
      - setWeight: 10
      - pause: {duration: 2m}
      - setWeight: 30
      - pause: {duration: 5m}
      - setWeight: 60
      - pause: {duration: 5m}

      # 자동 롤백 분석
      analysis:
        templates:
        - templateName: error-rate-check
        - templateName: latency-check
        startingStep: 1

  # 실패 시 자동 롤백
  progressDeadlineAborted: true

---
# 에러율 검사
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: error-rate-check
spec:
  metrics:
  - name: error-rate
    interval: 30s
    count: 10
    successCondition: result[0] < 0.05  # 5% 미만
    failureCondition: result[0] >= 0.10  # 10% 이상이면 실패
    failureLimit: 3  # 3번 연속 실패 시 롤백
    provider:
      prometheus:
        address: http://prometheus:9090
        query: |
          sum(rate(http_requests_total{status=~"5.*",app="myapp"}[5m])) /
          sum(rate(http_requests_total{app="myapp"}[5m]))

---
# 레이턴시 검사
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: latency-check
spec:
  metrics:
  - name: p99-latency
    interval: 30s
    count: 10
    successCondition: result[0] < 500  # 500ms 미만
    failureCondition: result[0] >= 1000  # 1초 이상이면 실패
    failureLimit: 3
    provider:
      prometheus:
        address: http://prometheus:9090
        query: |
          histogram_quantile(0.99,
            sum(rate(http_request_duration_seconds_bucket{app="myapp"}[5m]))
            by (le)
          ) * 1000
```

### 6.4 데이터베이스 롤백 전략

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    DB 스키마 변경 시 롤백 전략                            │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  1. Expand-Contract 패턴 (안전한 마이그레이션)                            │
│                                                                          │
│     Phase 1: Expand (확장)                                               │
│     ┌─────────────────────────────────────────────────────────────────┐ │
│     │  • 새 컬럼 추가 (nullable or default)                           │ │
│     │  • 기존 코드는 변경 없이 동작                                    │ │
│     │  ALTER TABLE users ADD COLUMN email_verified BOOLEAN DEFAULT    │ │
│     │  FALSE;                                                         │ │
│     └─────────────────────────────────────────────────────────────────┘ │
│                                                                          │
│     Phase 2: Migrate (마이그레이션)                                      │
│     ┌─────────────────────────────────────────────────────────────────┐ │
│     │  • 새 코드 배포 (신/구 컬럼 모두 사용)                           │ │
│     │  • 데이터 백필 작업                                              │ │
│     │  UPDATE users SET email_verified = TRUE WHERE verified_at IS    │ │
│     │  NOT NULL;                                                      │ │
│     └─────────────────────────────────────────────────────────────────┘ │
│                                                                          │
│     Phase 3: Contract (축소)                                             │
│     ┌─────────────────────────────────────────────────────────────────┐ │
│     │  • 구 컬럼 사용 코드 제거                                        │ │
│     │  • 구 컬럼 삭제                                                  │ │
│     │  ALTER TABLE users DROP COLUMN verified_at;                     │ │
│     └─────────────────────────────────────────────────────────────────┘ │
│                                                                          │
│  2. 버전별 호환성 유지                                                   │
│     • v1.0: old_column 사용                                             │
│     • v1.1: old_column + new_column 모두 처리                            │
│     • v1.2: new_column만 사용, old_column 삭제                           │
│                                                                          │
│  3. 롤백 마이그레이션 준비                                                │
│     • 모든 마이그레이션에 대해 rollback 스크립트 작성                     │
│     • Flyway/Liquibase undo 기능 활용                                    │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 7. 배포 전략 비교 및 선택 가이드

### 7.1 종합 비교

| 전략 | 다운타임 | 롤백 속도 | 리소스 비용 | 복잡도 | 리스크 |
|------|----------|----------|------------|--------|--------|
| **Blue-Green** | 없음 | 즉시 | 높음 (2배) | 중간 | 낮음 |
| **Canary** | 없음 | 빠름 | 낮음 | 높음 | 매우 낮음 |
| **Rolling Update** | 없음 | 느림 | 낮음 | 낮음 | 중간 |
| **A/B Testing** | 없음 | 빠름 | 중간 | 높음 | 낮음 |
| **Feature Flag** | 없음 | 즉시 | 낮음 | 중간 | 매우 낮음 |

### 7.2 시나리오별 권장 전략

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     시나리오별 배포 전략 선택                             │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  Q1: 즉시 롤백이 필요한가?                                               │
│      │                                                                   │
│      ├── Yes ─▶ Blue-Green 또는 Feature Flag                            │
│      │                                                                   │
│      └── No ─▶ Q2: 프로덕션 트래픽으로 검증이 필요한가?                   │
│                  │                                                       │
│                  ├── Yes ─▶ Canary Deployment                            │
│                  │                                                       │
│                  └── No ─▶ Rolling Update                                │
│                                                                          │
│  Q3: 비즈니스 지표 측정이 목적인가?                                       │
│      │                                                                   │
│      ├── Yes ─▶ A/B Testing + Feature Flag                              │
│      │                                                                   │
│      └── No ─▶ 기술적 배포 전략 사용                                     │
│                                                                          │
│  ──────────────────────────────────────────────────────────────────────  │
│                                                                          │
│  시나리오 예시:                                                          │
│                                                                          │
│  • 결제 시스템 업데이트                                                  │
│    → Blue-Green (즉시 롤백 필요, 높은 안정성 요구)                        │
│                                                                          │
│  • 새로운 추천 알고리즘                                                  │
│    → Canary + Feature Flag (점진적 롤아웃, 메트릭 검증)                  │
│                                                                          │
│  • UI 버튼 색상 변경                                                     │
│    → A/B Testing (비즈니스 지표 측정)                                    │
│                                                                          │
│  • 일반 버그 수정                                                        │
│    → Rolling Update (간단, 빠른 배포)                                    │
│                                                                          │
│  • 실험적 기능 테스트                                                    │
│    → Feature Flag (빠른 활성화/비활성화)                                 │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 참고 자료

- [Harness - Blue-Green and Canary Deployments](https://www.harness.io/blog/blue-green-canary-deployment-strategies)
- [Unleash - Comparing Deployment Strategies](https://www.getunleash.io/blog/comparing-deployment-strategies-canary-blue-green-and-rolling)
- [Argo Rollouts Documentation](https://argoproj.github.io/argo-rollouts/)
- [Kubernetes Deployment Strategies](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Feature Flag Best Practices 2025](https://octopus.com/devops/feature-flags/feature-flag-best-practices/)
- [Optimizely - Feature Flags vs A/B Testing](https://www.optimizely.com/insights/blog/feature-flags-vs-ab-testing/)
