# 컨테이너 오케스트레이션

## 목차
1. [스케줄링 전략](#1-스케줄링-전략)
2. [서비스 디스커버리](#2-서비스-디스커버리)
3. [롤링 업데이트와 롤백](#3-롤링-업데이트와-롤백)
4. [Auto-scaling (HPA, VPA)](#4-auto-scaling-hpa-vpa)
5. [Kubernetes vs Docker Swarm](#5-kubernetes-vs-docker-swarm)
---

## 1. 스케줄링 전략

### 1.1 Kubernetes 스케줄러 개요

kube-scheduler는 새로 생성된 Pod를 어떤 노드에 배치할지 결정합니다.

```
┌─────────────────────────────────────────────────────────────────┐
│                    Scheduling Process                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   1. Watch: API Server에서 스케줄 대기 Pod 감시                  │
│                                                                  │
│   2. Filtering (Predicates): 실행 가능한 노드 필터링            │
│      ├── 리소스 충분한가? (CPU, Memory)                         │
│      ├── Node Selector 만족하는가?                              │
│      ├── Affinity/Anti-Affinity 조건 만족하는가?                │
│      ├── Taint/Toleration 충족하는가?                           │
│      └── Pod가 요청한 볼륨 마운트 가능한가?                     │
│                                                                  │
│   3. Scoring (Priorities): 최적 노드에 점수 부여                │
│      ├── 리소스 균형 (LeastRequestedPriority)                   │
│      ├── Pod 분산 (SelectorSpreadPriority)                      │
│      ├── 선호도 가중치 (NodeAffinityPriority)                   │
│      └── 이미지 로컬리티 (ImageLocalityPriority)                │
│                                                                  │
│   4. Binding: 선택된 노드에 Pod 할당                            │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 Node Selector

가장 간단한 노드 선택 방법입니다.

```yaml
# Node에 Label 추가
# kubectl label nodes node1 disktype=ssd

apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx
  nodeSelector:
    disktype: ssd  # disktype=ssd 라벨이 있는 노드에만 스케줄
```

### 1.3 Node Affinity

nodeSelector보다 유연한 노드 선택 규칙을 정의합니다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: with-node-affinity
spec:
  affinity:
    nodeAffinity:
      # 필수 조건 (Hard)
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: topology.kubernetes.io/zone
            operator: In
            values:
            - ap-northeast-2a
            - ap-northeast-2b

      # 선호 조건 (Soft)
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        preference:
          matchExpressions:
          - key: disktype
            operator: In
            values:
            - ssd
      - weight: 50
        preference:
          matchExpressions:
          - key: cpu
            operator: In
            values:
            - high-performance

  containers:
  - name: nginx
    image: nginx
```

### 1.4 Pod Affinity / Anti-Affinity

Pod 간의 배치 관계를 정의합니다.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-server
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      affinity:
        # Pod Affinity: 특정 Pod와 같은 노드/존에 배치
        podAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchLabels:
                app: cache  # cache Pod와 같은 노드에
            topologyKey: kubernetes.io/hostname

        # Pod Anti-Affinity: 특정 Pod와 다른 노드/존에 배치
        podAntiAffinity:
          # 같은 Deployment의 Pod를 다른 노드에 분산
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  app: web
              topologyKey: kubernetes.io/hostname

          # 필수: 같은 Zone에 web Pod가 없어야 함
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchLabels:
                app: web
            topologyKey: topology.kubernetes.io/zone

      containers:
      - name: web
        image: nginx
```

### 1.5 Taints와 Tolerations

특정 노드에 Pod 스케줄링을 제한합니다.

```bash
# Node에 Taint 추가
kubectl taint nodes node1 dedicated=gpu:NoSchedule
kubectl taint nodes node2 env=production:NoExecute

# Taint 효과
# NoSchedule: Toleration 없는 Pod 스케줄 거부
# PreferNoSchedule: 가능하면 스케줄 안 함 (Soft)
# NoExecute: 기존 Pod도 퇴거 (Eviction)

# Taint 제거
kubectl taint nodes node1 dedicated=gpu:NoSchedule-
```

```yaml
# Pod에 Toleration 추가
apiVersion: v1
kind: Pod
metadata:
  name: gpu-pod
spec:
  tolerations:
  - key: "dedicated"
    operator: "Equal"
    value: "gpu"
    effect: "NoSchedule"

  # 또는 모든 Taint 허용
  # - operator: "Exists"

  containers:
  - name: gpu-container
    image: nvidia/cuda
    resources:
      limits:
        nvidia.com/gpu: 1
```

### 1.6 Topology Spread Constraints

Pod를 토폴로지(존, 노드 등)에 균등하게 분산합니다.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 6
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      topologySpreadConstraints:
      # Zone 간 균등 분산
      - maxSkew: 1  # 최대 불균형 허용치
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule  # ScheduleAnyway
        labelSelector:
          matchLabels:
            app: web

      # Node 간 균등 분산
      - maxSkew: 2
        topologyKey: kubernetes.io/hostname
        whenUnsatisfiable: ScheduleAnyway
        labelSelector:
          matchLabels:
            app: web

      containers:
      - name: web
        image: nginx
```

### 1.7 Priority와 Preemption

Pod 우선순위와 선점을 설정합니다.

```yaml
# PriorityClass 정의
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000
globalDefault: false
description: "고우선순위 워크로드용"
preemptionPolicy: PreemptLowerPriority

---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: low-priority
value: 1000
globalDefault: true
description: "기본 우선순위"

---
# Pod에서 사용
apiVersion: v1
kind: Pod
metadata:
  name: critical-pod
spec:
  priorityClassName: high-priority
  containers:
  - name: app
    image: my-app

# 리소스 부족 시 high-priority Pod를 위해
# low-priority Pod가 Eviction될 수 있음
```

---

## 2. 서비스 디스커버리

### 2.1 서비스 디스커버리 개요

Pod의 IP는 동적으로 변경되므로, 서비스 디스커버리가 필수입니다.

```
┌─────────────────────────────────────────────────────────────────┐
│                   Service Discovery Methods                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. DNS 기반 (CoreDNS)                                          │
│     ├── Service: <svc>.<ns>.svc.cluster.local                   │
│     ├── Pod: <pod-ip>.<ns>.pod.cluster.local                    │
│     └── Headless: <pod-name>.<svc>.<ns>.svc.cluster.local      │
│                                                                  │
│  2. 환경변수 기반                                                │
│     ├── <SVC>_SERVICE_HOST                                      │
│     └── <SVC>_SERVICE_PORT                                      │
│                                                                  │
│  3. API 기반                                                    │
│     └── Kubernetes API로 Endpoints 조회                        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 DNS 기반 서비스 디스커버리

CoreDNS는 Kubernetes의 기본 DNS 서버입니다.

```yaml
# Service 생성
apiVersion: v1
kind: Service
metadata:
  name: my-service
  namespace: production
spec:
  selector:
    app: my-app
  ports:
  - port: 80
    targetPort: 8080
```

```bash
# DNS 레코드 형식
# Service: <service-name>.<namespace>.svc.cluster.local

# 같은 네임스페이스에서
curl http://my-service

# 다른 네임스페이스에서
curl http://my-service.production

# 완전한 FQDN
curl http://my-service.production.svc.cluster.local

# SRV 레코드 (포트 정보 포함)
# _<port-name>._<protocol>.<service>.<namespace>.svc.cluster.local
nslookup _http._tcp.my-service.production.svc.cluster.local
```

### 2.3 Headless Service

```yaml
# Headless Service (개별 Pod DNS 제공)
apiVersion: v1
kind: Service
metadata:
  name: db-headless
spec:
  clusterIP: None  # Headless
  selector:
    app: database
  ports:
  - port: 5432

---
# StatefulSet과 함께 사용
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: db-headless
  replicas: 3
  selector:
    matchLabels:
      app: database
  template:
    metadata:
      labels:
        app: database
    spec:
      containers:
      - name: postgres
        image: postgres:16
```

```bash
# 개별 Pod DNS
# <pod-name>.<service-name>.<namespace>.svc.cluster.local
nslookup postgres-0.db-headless.default.svc.cluster.local
nslookup postgres-1.db-headless.default.svc.cluster.local
nslookup postgres-2.db-headless.default.svc.cluster.local

# A 레코드로 모든 Pod IP 반환
dig db-headless.default.svc.cluster.local
```

### 2.4 환경변수 기반 디스커버리

Pod 생성 시 자동으로 환경변수가 주입됩니다.

```bash
# Pod 내에서 환경변수 확인
env | grep MY_SERVICE

# 결과 예시
MY_SERVICE_SERVICE_HOST=10.96.0.10
MY_SERVICE_SERVICE_PORT=80
MY_SERVICE_PORT=tcp://10.96.0.10:80
MY_SERVICE_PORT_80_TCP=tcp://10.96.0.10:80
MY_SERVICE_PORT_80_TCP_PROTO=tcp
MY_SERVICE_PORT_80_TCP_PORT=80
MY_SERVICE_PORT_80_TCP_ADDR=10.96.0.10
```

```yaml
# 애플리케이션에서 사용
apiVersion: v1
kind: Pod
metadata:
  name: client-pod
spec:
  containers:
  - name: client
    image: my-client
    env:
    - name: DATABASE_HOST
      value: $(MY_DB_SERVICE_HOST)  # 환경변수 참조
```

### 2.5 Endpoints와 EndpointSlices

```yaml
# Endpoints: Service에 연결된 Pod IP 목록
kubectl get endpoints my-service

# 결과 예시
NAME         ENDPOINTS                                    AGE
my-service   10.244.0.5:8080,10.244.1.3:8080,10.244.2.7:8080  5m

# EndpointSlices (대규모 클러스터에 더 효율적)
kubectl get endpointslices

# 수동으로 외부 서비스 Endpoint 설정
apiVersion: v1
kind: Service
metadata:
  name: external-db
spec:
  ports:
  - port: 5432
---
apiVersion: v1
kind: Endpoints
metadata:
  name: external-db
subsets:
- addresses:
  - ip: 192.168.1.100
  ports:
  - port: 5432
```

### 2.6 ExternalName Service

```yaml
# 외부 DNS를 클러스터 내부 이름으로 매핑
apiVersion: v1
kind: Service
metadata:
  name: external-api
spec:
  type: ExternalName
  externalName: api.external-service.com
```

```bash
# Pod 내에서
curl http://external-api  # → api.external-service.com으로 리다이렉트
```

### 2.7 CoreDNS 설정

```yaml
# CoreDNS ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health {
           lameduck 5s
        }
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
           pods insecure
           fallthrough in-addr.arpa ip6.arpa
           ttl 30
        }
        prometheus :9153
        forward . /etc/resolv.conf {
           max_concurrent 1000
        }
        cache 30
        loop
        reload
        loadbalance
    }
```

---

## 3. 롤링 업데이트와 롤백

### 3.1 배포 전략 종류

```
┌─────────────────────────────────────────────────────────────────┐
│                   Deployment Strategies                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. Rolling Update (기본)                                       │
│     - 점진적 교체                                                │
│     - 무중단 배포                                                │
│     - Kubernetes 기본 지원                                      │
│                                                                  │
│  2. Recreate                                                    │
│     - 전체 교체                                                  │
│     - 다운타임 발생                                              │
│     - 단순하고 빠름                                              │
│                                                                  │
│  3. Blue-Green                                                  │
│     - 두 환경 전환                                               │
│     - 즉시 롤백 가능                                             │
│     - 리소스 2배 필요                                            │
│                                                                  │
│  4. Canary                                                      │
│     - 점진적 트래픽 이동                                         │
│     - 위험 최소화                                                │
│     - 정교한 모니터링 필요                                       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 Rolling Update

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 4
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1         # 최대 추가 Pod (25% 또는 절대값)
      maxUnavailable: 0   # 최대 제거 가능 Pod (25% 또는 절대값)
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: app
        image: my-app:v1.0.0
        readinessProbe:  # 필수! 트래픽 전환 시점 결정
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 15
          periodSeconds: 10
```

```
Rolling Update 과정 (replicas=4, maxSurge=1, maxUnavailable=0):

시작: [v1] [v1] [v1] [v1]

Step 1: [v1] [v1] [v1] [v1] [v2] ← 새 Pod 추가
        (maxSurge=1이므로 최대 5개 Pod)

Step 2: [v1] [v1] [v1] [v2]       ← v2 Ready 후 v1 하나 제거
        (maxUnavailable=0이므로 항상 4개 Ready)

Step 3: [v1] [v1] [v1] [v2] [v2] ← 새 Pod 추가
        ...

완료: [v2] [v2] [v2] [v2]
```

### 3.3 Recreate 전략

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3
  strategy:
    type: Recreate  # 모든 Pod 종료 후 새 버전 시작
  selector:
    matchLabels:
      app: my-app
  template:
    spec:
      containers:
      - name: app
        image: my-app:v2.0.0
```

```
Recreate 과정:

시작:  [v1] [v1] [v1]
Step 1: [ ] [ ] [ ]    ← 모든 Pod 종료 (다운타임!)
Step 2: [v2] [v2] [v2] ← 새 버전 시작
```

### 3.4 Blue-Green 배포

Kubernetes 기본 기능으로 구현하거나 Argo Rollouts 사용

```yaml
# Blue Deployment (현재 운영)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-blue
  labels:
    version: blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
      version: blue
  template:
    metadata:
      labels:
        app: my-app
        version: blue
    spec:
      containers:
      - name: app
        image: my-app:v1.0.0

---
# Green Deployment (새 버전)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-green
  labels:
    version: green
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
      version: green
  template:
    metadata:
      labels:
        app: my-app
        version: green
    spec:
      containers:
      - name: app
        image: my-app:v2.0.0

---
# Service: Selector 변경으로 트래픽 전환
apiVersion: v1
kind: Service
metadata:
  name: my-app
spec:
  selector:
    app: my-app
    version: blue  # green으로 변경하여 전환
  ports:
  - port: 80
    targetPort: 8080
```

```bash
# 트래픽 전환
kubectl patch service my-app -p '{"spec":{"selector":{"version":"green"}}}'

# 롤백
kubectl patch service my-app -p '{"spec":{"selector":{"version":"blue"}}}'
```

### 3.5 Canary 배포

```yaml
# Stable Deployment (90% 트래픽)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-stable
spec:
  replicas: 9
  selector:
    matchLabels:
      app: my-app
      track: stable
  template:
    metadata:
      labels:
        app: my-app
        track: stable
    spec:
      containers:
      - name: app
        image: my-app:v1.0.0

---
# Canary Deployment (10% 트래픽)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-canary
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-app
      track: canary
  template:
    metadata:
      labels:
        app: my-app
        track: canary
    spec:
      containers:
      - name: app
        image: my-app:v2.0.0

---
# Service: 두 Deployment 모두 선택 (app=my-app)
apiVersion: v1
kind: Service
metadata:
  name: my-app
spec:
  selector:
    app: my-app  # stable과 canary 모두 선택
  ports:
  - port: 80
    targetPort: 8080
```

### 3.6 Argo Rollouts

고급 배포 전략을 위한 Kubernetes 컨트롤러

```yaml
# Argo Rollouts Canary 예시
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: my-app
spec:
  replicas: 10
  strategy:
    canary:
      steps:
      - setWeight: 10       # 10% 트래픽
      - pause: {duration: 5m}
      - setWeight: 30       # 30% 트래픽
      - pause: {duration: 5m}
      - setWeight: 50       # 50% 트래픽
      - pause: {duration: 10m}
      # 분석 후 자동 승인/롤백
      - analysis:
          templates:
          - templateName: success-rate
      - setWeight: 100      # 100% 트래픽

      # 트래픽 관리 (Istio/NGINX Ingress)
      trafficRouting:
        istio:
          virtualService:
            name: my-app-vsvc

  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: app
        image: my-app:v2.0.0

---
# AnalysisTemplate: Canary 성공 기준 정의
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate
spec:
  metrics:
  - name: success-rate
    interval: 1m
    successCondition: result[0] >= 0.95
    provider:
      prometheus:
        address: http://prometheus:9090
        query: |
          sum(rate(http_requests_total{status=~"2.."}[5m]))
          /
          sum(rate(http_requests_total[5m]))
```

### 3.7 롤백

```bash
# 배포 히스토리 확인
kubectl rollout history deployment/my-app

# 특정 리비전 상세 정보
kubectl rollout history deployment/my-app --revision=3

# 이전 버전으로 롤백
kubectl rollout undo deployment/my-app

# 특정 리비전으로 롤백
kubectl rollout undo deployment/my-app --to-revision=2

# 롤백 상태 확인
kubectl rollout status deployment/my-app

# 현재 이미지 확인
kubectl get deployment my-app -o jsonpath='{.spec.template.spec.containers[0].image}'
```

---

## 4. Auto-scaling (HPA, VPA)

### 4.1 오토스케일링 종류

```
┌─────────────────────────────────────────────────────────────────┐
│                  Kubernetes Auto-scaling                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. HPA (Horizontal Pod Autoscaler)                             │
│     - Pod 수 조절                                                │
│     - 수평 스케일링                                              │
│     - Stateless 애플리케이션에 적합                             │
│                                                                  │
│  2. VPA (Vertical Pod Autoscaler)                               │
│     - Pod 리소스(CPU/Memory) 조절                               │
│     - 수직 스케일링                                              │
│     - Stateful/단일 인스턴스에 적합                             │
│                                                                  │
│  3. Cluster Autoscaler                                          │
│     - 노드 수 조절                                               │
│     - 클라우드 환경에서 동작                                    │
│     - HPA와 함께 사용                                           │
│                                                                  │
│  4. KEDA (Kubernetes Event Driven Autoscaler)                   │
│     - 이벤트 기반 스케일링                                      │
│     - 외부 메트릭 지원 (Kafka, RabbitMQ 등)                    │
│     - 0으로 스케일 다운 가능                                    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 4.2 HPA (Horizontal Pod Autoscaler)

```yaml
# metrics-server 필수 (CPU/Memory 메트릭 수집)
# kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app

  minReplicas: 2
  maxReplicas: 10

  metrics:
  # CPU 사용률 기반
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70

  # Memory 사용률 기반
  - type: Resource
    resource:
      name: memory
      target:
        type: AverageValue
        averageValue: 500Mi

  # 커스텀 메트릭 (Prometheus Adapter 필요)
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "100"

  # 외부 메트릭 (External Metrics API)
  - type: External
    external:
      metric:
        name: queue_messages_ready
        selector:
          matchLabels:
            queue: my-queue
      target:
        type: Value
        value: "30"

  behavior:
    # 스케일 업 동작
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 100
        periodSeconds: 15
      - type: Pods
        value: 4
        periodSeconds: 15
      selectPolicy: Max

    # 스케일 다운 동작
    scaleDown:
      stabilizationWindowSeconds: 300  # 5분 안정화
      policies:
      - type: Percent
        value: 10
        periodSeconds: 60
```

```bash
# HPA 생성 (명령어)
kubectl autoscale deployment my-app \
  --min=2 \
  --max=10 \
  --cpu-percent=70

# HPA 상태 확인
kubectl get hpa my-app-hpa

# 상세 정보
kubectl describe hpa my-app-hpa

# 메트릭 확인
kubectl top pods
```

### 4.3 VPA (Vertical Pod Autoscaler)

```bash
# VPA 설치
git clone https://github.com/kubernetes/autoscaler.git
cd autoscaler/vertical-pod-autoscaler
./hack/vpa-up.sh
```

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: my-app-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app

  updatePolicy:
    updateMode: Auto  # Off, Initial, Recreate, Auto
    # Off: 권장 사항만 제공
    # Initial: Pod 생성 시에만 적용
    # Recreate: Pod 재생성하여 적용
    # Auto: Recreate와 동일 (향후 in-place 업데이트 예정)

  resourcePolicy:
    containerPolicies:
    - containerName: app
      minAllowed:
        cpu: 100m
        memory: 128Mi
      maxAllowed:
        cpu: 2
        memory: 4Gi
      controlledResources: ["cpu", "memory"]
      controlledValues: RequestsAndLimits
```

```bash
# VPA 권장 사항 확인
kubectl describe vpa my-app-vpa

# 결과 예시:
# Recommendation:
#   Container Recommendations:
#     Container Name: app
#     Lower Bound:
#       Cpu:     25m
#       Memory:  262144k
#     Target:
#       Cpu:     50m
#       Memory:  524288k
#     Upper Bound:
#       Cpu:     100m
#       Memory:  1048576k
```

### 4.4 HPA와 VPA 함께 사용하기

```yaml
# 주의: 같은 메트릭(CPU/Memory)으로 HPA와 VPA 동시 사용 금지!

# HPA: 커스텀 메트릭 기반 스케일링
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "100"

---
# VPA: CPU/Memory 최적화 (Auto 모드)
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: my-app-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  updatePolicy:
    updateMode: Auto
  resourcePolicy:
    containerPolicies:
    - containerName: app
      controlledResources: ["cpu", "memory"]
```

### 4.5 Cluster Autoscaler

```yaml
# AWS EKS Cluster Autoscaler 예시
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cluster-autoscaler
  namespace: kube-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cluster-autoscaler
  template:
    metadata:
      labels:
        app: cluster-autoscaler
    spec:
      serviceAccountName: cluster-autoscaler
      containers:
      - image: registry.k8s.io/autoscaling/cluster-autoscaler:v1.28.0
        name: cluster-autoscaler
        command:
        - ./cluster-autoscaler
        - --v=4
        - --stderrthreshold=info
        - --cloud-provider=aws
        - --skip-nodes-with-local-storage=false
        - --expander=least-waste
        - --node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/my-cluster
        - --balance-similar-node-groups
        - --scale-down-enabled=true
        - --scale-down-delay-after-add=10m
        - --scale-down-unneeded-time=10m
```

### 4.6 KEDA (Kubernetes Event Driven Autoscaler)

```yaml
# KEDA 설치
# helm repo add kedacore https://kedacore.github.io/charts
# helm install keda kedacore/keda -n keda --create-namespace

# ScaledObject: 이벤트 기반 스케일링
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: kafka-consumer-scaler
spec:
  scaleTargetRef:
    name: kafka-consumer
  minReplicaCount: 0    # 0으로 스케일 다운 가능!
  maxReplicaCount: 100
  cooldownPeriod: 30
  triggers:
  - type: kafka
    metadata:
      bootstrapServers: kafka:9092
      consumerGroup: my-consumer-group
      topic: my-topic
      lagThreshold: "100"

---
# Prometheus 메트릭 기반
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: prometheus-scaler
spec:
  scaleTargetRef:
    name: my-app
  minReplicaCount: 1
  maxReplicaCount: 50
  triggers:
  - type: prometheus
    metadata:
      serverAddress: http://prometheus:9090
      metricName: http_requests_total
      threshold: "100"
      query: sum(rate(http_requests_total{app="my-app"}[2m]))
```

---

## 5. Kubernetes vs Docker Swarm

### 5.1 비교 개요

```
┌────────────────────────────────────────────────────────────────────────┐
│                     Kubernetes vs Docker Swarm                          │
├────────────────────┬───────────────────────┬───────────────────────────┤
│       항목         │      Kubernetes       │      Docker Swarm          │
├────────────────────┼───────────────────────┼───────────────────────────┤
│ 복잡도             │ 높음 (학습 곡선 가파름)│ 낮음 (Docker 사용자 친화) │
│ 설치               │ 복잡 (kubeadm 등)     │ 간단 (docker swarm init)  │
│ 확장성             │ 수천 노드 지원        │ 수백 노드 수준            │
│ 기능               │ 풍부 (RBAC, HPA 등)   │ 기본적인 기능             │
│ 생태계             │ 거대 (CNCF)           │ 제한적                    │
│ 커뮤니티           │ 매우 활발             │ 상대적으로 작음           │
│ 클라우드 지원      │ 모든 주요 클라우드    │ 제한적                    │
│ 로드밸런싱         │ Ingress, Service      │ 내장 (Routing Mesh)       │
│ 롤링 업데이트      │ 상세 설정 가능        │ 기본 지원                 │
│ 스토리지           │ 동적 프로비저닝       │ 볼륨 드라이버             │
│ 네트워킹           │ CNI 플러그인          │ Overlay 네트워크          │
│ 모니터링           │ 다양한 도구 통합      │ 기본 도구                 │
│ 시장 점유율        │ ~90%                  │ ~5%                       │
└────────────────────┴───────────────────────┴───────────────────────────┘
```

### 5.2 아키텍처 비교

```
┌──────────────────────────────────────────────────────────────────────┐
│                       Kubernetes Architecture                         │
├──────────────────────────────────────────────────────────────────────┤
│                                                                       │
│   Control Plane                    Worker Nodes                       │
│   ┌─────────────────┐              ┌─────────────────┐               │
│   │ API Server      │              │ kubelet         │               │
│   │ etcd            │◄────────────►│ kube-proxy      │               │
│   │ Scheduler       │              │ Container Runtime│               │
│   │ Controller Mgr  │              │ Pods            │               │
│   └─────────────────┘              └─────────────────┘               │
│                                                                       │
└──────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────┐
│                       Docker Swarm Architecture                       │
├──────────────────────────────────────────────────────────────────────┤
│                                                                       │
│   Manager Nodes                    Worker Nodes                       │
│   ┌─────────────────┐              ┌─────────────────┐               │
│   │ Swarm Manager   │              │ Docker Engine   │               │
│   │ (Raft Consensus)│◄────────────►│ Tasks           │               │
│   │ Orchestrator    │              │ Containers      │               │
│   └─────────────────┘              └─────────────────┘               │
│                                                                       │
│   * Manager도 Worker 역할 가능                                        │
│   * 단일 Docker Engine으로 동작                                       │
│                                                                       │
└──────────────────────────────────────────────────────────────────────┘
```

### 5.3 주요 개념 비교

| Kubernetes | Docker Swarm | 설명 |
|------------|--------------|------|
| Pod | Task | 최소 배포 단위 |
| Deployment | Service | 배포 및 스케일링 관리 |
| Service | Service (Endpoint) | 네트워크 엔드포인트 |
| ConfigMap | Config | 설정 관리 |
| Secret | Secret | 민감 정보 관리 |
| Namespace | Stack | 리소스 그룹화 |
| HPA | - | 자동 스케일링 (Swarm 미지원) |
| Ingress | Routing Mesh | 외부 트래픽 라우팅 |

### 5.4 Docker Swarm 예시

```bash
# Swarm 초기화
docker swarm init

# 워커 노드 추가
docker swarm join --token <token> <manager-ip>:2377

# Service 배포
docker service create \
  --name my-web \
  --replicas 3 \
  --publish 8080:80 \
  --update-delay 10s \
  --update-parallelism 2 \
  nginx

# 스케일링
docker service scale my-web=5

# 롤링 업데이트
docker service update \
  --image nginx:latest \
  --update-parallelism 1 \
  --update-delay 30s \
  my-web

# 상태 확인
docker service ls
docker service ps my-web
```

### 5.5 Docker Compose (Swarm 모드)

```yaml
# docker-stack.yml
version: '3.8'

services:
  web:
    image: nginx:alpine
    deploy:
      replicas: 3
      update_config:
        parallelism: 1
        delay: 10s
        failure_action: rollback
      rollback_config:
        parallelism: 1
        delay: 10s
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
      resources:
        limits:
          cpus: '0.5'
          memory: 256M
        reservations:
          cpus: '0.25'
          memory: 128M
      placement:
        constraints:
          - node.role == worker
    ports:
      - "8080:80"
    networks:
      - webnet

  visualizer:
    image: dockersamples/visualizer
    ports:
      - "8081:8080"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    deploy:
      placement:
        constraints:
          - node.role == manager

networks:
  webnet:
    driver: overlay
```

```bash
# Stack 배포
docker stack deploy -c docker-stack.yml myapp

# Stack 확인
docker stack ls
docker stack services myapp
docker stack ps myapp

# Stack 제거
docker stack rm myapp
```

### 5.6 언제 어떤 것을 선택할까?

```
┌─────────────────────────────────────────────────────────────────┐
│                    선택 가이드라인                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Kubernetes 선택:                                               │
│  ✓ 대규모 프로덕션 환경                                        │
│  ✓ 복잡한 마이크로서비스 아키텍처                              │
│  ✓ 고급 스케줄링/오토스케일링 필요                             │
│  ✓ 멀티 클라우드/하이브리드 클라우드                           │
│  ✓ 활발한 에코시스템 활용 (Istio, Argo 등)                    │
│  ✓ 전담 DevOps/SRE 팀 보유                                     │
│                                                                  │
│  Docker Swarm 선택:                                             │
│  ✓ 소규모 팀/프로젝트                                          │
│  ✓ 빠른 셋업 필요                                              │
│  ✓ Docker Compose에 익숙                                       │
│  ✓ 기본적인 오케스트레이션만 필요                              │
│  ✓ 엣지 컴퓨팅/IoT 환경                                        │
│  ✓ 학습/테스트 목적                                            │
│                                                                  │
│  2025년 현실:                                                   │
│  - 대부분의 기업이 Kubernetes 채택                              │
│  - 주요 클라우드: EKS, GKE, AKS 모두 K8s 기반                  │
│  - Docker Swarm: 단순성이 필요한 특수 케이스에 사용            │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 참고 자료

- [Kubernetes Scheduling](https://kubernetes.io/docs/concepts/scheduling-eviction/)
- [Kubernetes HPA](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)
- [Kubernetes VPA](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler)
- [Argo Rollouts](https://argoproj.github.io/rollouts/)
- [KEDA](https://keda.sh/)
- [Cluster Autoscaler](https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler)
- [Docker Swarm Documentation](https://docs.docker.com/engine/swarm/)
