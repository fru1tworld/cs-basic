# Kubernetes

## K8s 아키텍처

### 전체 아키텍처 개요

- Kubernetes는 분산 시스템으로, Control Plane(마스터)과 Worker Node로 구성됨.

- Kubernetes Cluster
- Control Plane
- API Server, ←, etcd, Controller Manager
- (kube-, (Key-Value, - Node Controller
- apiserver), Store), - Replication Ctrl
- Endpoint Ctrl
- Service Ctrl
- Scheduler, Cloud Controller Mgr
- (kube-, (cloud-specific)
- scheduler)
- kubelet ← API Server와 통신
- Worker Nodes
- Node 1
- kubelet, kube-
- proxy
- Container Runtime
- (containerd/CRI-O)
- Pod A, Pod B, Pod C
- Node 2
- (동일 구조)

### Control Plane 컴포넌트

- 컴포넌트: kube-apiserver:
  - 역할: API 게이트웨이
  - 상세 설명: 모든 통신의 중심점, REST API 노출, 인증/인가 처리
- 컴포넌트: etcd:
  - 역할: 분산 저장소
  - 상세 설명: 클러스터 상태 저장 (Key-Value), SSOT (Single Source of Truth)
- 컴포넌트: kube-scheduler:
  - 역할: Pod 배치 결정
  - 상세 설명: 리소스, Affinity, Taint/Toleration 고려하여 노드 선택
- 컴포넌트: kube-controller-manager:
  - 역할: 상태 관리
  - 상세 설명: Desired State와 Current State 일치시킴
- 컴포넌트: cloud-controller-manager:
  - 역할: 클라우드 통합
  - 상세 설명: LoadBalancer, Volume 등 클라우드 리소스 관리

### Worker Node 컴포넌트

```yaml
# Node의 구성 요소와 역할

kubelet:
  역할: Pod 생명주기 관리
  책임:
    - API Server로부터 PodSpec 수신
    - Container Runtime에 컨테이너 생성 요청
    - Pod 상태 모니터링 및 보고
    - Liveness/Readiness Probe 실행

kube-proxy:
  역할: 네트워크 프록시
  모드:
    - iptables (기본): iptables 규칙으로 서비스 라우팅
    - IPVS: 고성능 L4 로드밸런서 (대규모 클러스터 권장)
    - eBPF: Cilium 등 CNI에서 kube-proxy 대체

Container Runtime:
  역할: 컨테이너 실행
  종류:
    - containerd (K8s 1.24+ 기본)
    - CRI-O (Red Hat)
    - Docker (cri-dockerd 통해 지원)
```

### etcd 심층 이해

```bash
# etcd 클러스터 상태 확인 (Control Plane 노드에서)
etcdctl member list
etcdctl endpoint health

# Kubernetes 키 예: /registry/pods/default/my-pod
# 다른 키 접두사: /registry/deployments/, /registry/services/
# 다른 키 접두사: /registry/secrets/, /registry/configmaps/

# etcd 백업 (중요!)
ETCDCTL_API=3 etcdctl snapshot save snapshot.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
  --key=/etc/kubernetes/pki/etcd/healthcheck-client.key
```

### API Server 요청 흐름

- kubectl apply -f deployment.yaml
- API Server
- Authentication (인증)
- Client Certificate
- Bearer Token
- OIDC
- Authorization (인가)
- RBAC (Role-Based Access Control)
- ABAC
- Webhook
- Admission Control
- Mutating Admission (리소스 수정)
- Validating Admission (리소스 검증)
- ex) LimitRange, ResourceQuota, PodSecurityPolicy
- Validation
- OpenAPI Schema 검증
- Persist to etcd
- Object 저장

## Pod, ReplicaSet, Deployment

### Pod

Kubernetes에서 배포하는 최소 단위는 개별 컨테이너가 아니라 Pod다. 하나의 Pod에는 하나 이상의 컨테이너를 넣을 수 있으며, 함께 배치하고 관리해야 하는 컨테이너를 묶는 데 사용한다.

```yaml
# pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
  labels:
    app: my-app
    version: v1
spec:
  containers:
  - name: main-container
    image: nginx:1.25
    ports:
    - containerPort: 80
    resources:
      requests:
        memory: "128Mi"
        cpu: "250m"
      limits:
        memory: "256Mi"
        cpu: "500m"
    livenessProbe:
      httpGet:
        path: /healthz
        port: 80
      initialDelaySeconds: 5
      periodSeconds: 10
    readinessProbe:
      httpGet:
        path: /ready
        port: 80
      initialDelaySeconds: 3
      periodSeconds: 5
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config

  # Sidecar Container
  - name: log-collector
    image: fluentd:v1.16
    volumeMounts:
    - name: logs
      mountPath: /var/log

  volumes:
  - name: config-volume
    configMap:
      name: my-config
  - name: logs
    emptyDir: {}

  # Pod 설정
  restartPolicy: Always
  terminationGracePeriodSeconds: 30
```

### Pod 생명주기

- Pod Lifecycle
- Pending, → Running, → Succeeded
- → Failed
- → 스케줄링 대기, 이미지 Pull 중
- Container States:
- Waiting: 컨테이너 시작 준비 중
- Running: 컨테이너 실행 중
- Terminated: 컨테이너 종료됨
- Pod Conditions:
- PodScheduled: 노드에 스케줄됨
- ContainersReady: 모든 컨테이너 Ready
- Initialized: 모든 Init Container 완료
- Ready: Pod가 트래픽 받을 준비됨

### Probe 종류

```yaml
# Liveness Probe: 컨테이너가 살아있는지 확인
# 실패 시 컨테이너 재시작
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 15
  periodSeconds: 20
  failureThreshold: 3

# Readiness Probe: 트래픽 받을 준비가 되었는지 확인
# 실패 시 Service에서 제외
readinessProbe:
  tcpSocket:
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 10

# Startup Probe: 애플리케이션 시작 확인
# 시작 느린 앱에 유용, 성공 전까지 다른 Probe 비활성화
startupProbe:
  httpGet:
    path: /startup
    port: 8080
  failureThreshold: 30
  periodSeconds: 10
```

### ReplicaSet

Pod를 실행한 뒤에는 장애나 종료로 개수가 줄어들 수 있다. ReplicaSet은 지정된 수의 Pod 복제본을 유지해, 실제 실행 상태가 원하는 상태에 맞도록 관리한다.

```yaml
# replicaset.yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: my-replicaset
  labels:
    app: my-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
```

### Deployment

Pod 수를 유지하는 것에 더해 애플리케이션 버전을 교체하려면 업데이트 과정도 관리해야 한다. Deployment는 ReplicaSet을 관리하면서 선언적 업데이트와 롤백을 제공한다.

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-deployment
  labels:
    app: my-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app

  # 업데이트 전략
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # 최대 추가 Pod 수 (기본 25%)
      maxUnavailable: 0  # 최대 제거 가능 Pod 수 (기본 25%)

  # 롤백을 위한 히스토리 유지
  revisionHistoryLimit: 10

  # 배포 진행 타임아웃
  progressDeadlineSeconds: 600

  template:
    metadata:
      labels:
        app: my-app
      annotations:
        prometheus.io/scrape: "true"
    spec:
      containers:
      - name: app
        image: my-app:v1.2.0
        ports:
        - containerPort: 8080
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: "production"
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: password
```

### Deployment 관리 명령어

```bash
# 배포
kubectl apply -f deployment.yaml

# 상태 확인
kubectl rollout status deployment/my-deployment

# 배포 히스토리
kubectl rollout history deployment/my-deployment

# 롤백
kubectl rollout undo deployment/my-deployment

# 특정 리비전으로 롤백
kubectl rollout undo deployment/my-deployment --to-revision=2

# 스케일 조정
kubectl scale deployment/my-deployment --replicas=5

# 일시 중지/재개 (여러 변경 사항 배치)
kubectl rollout pause deployment/my-deployment
kubectl set image deployment/my-deployment app=my-app:v2.0.0
kubectl set resources deployment/my-deployment -c=app --limits=memory=1Gi
kubectl rollout resume deployment/my-deployment
```

### Deployment vs ReplicaSet vs Pod 관계

Deployment가 v1에서 v2로 업데이트되면, 활성 상태의 v2 ReplicaSet이 Selector를 기준으로 Pod 1, 2, 3의 수를 유지한다. 이전 v1 ReplicaSet은 `replicas=0`인 상태로 남아 롤백에 사용된다. Deployment는 버전 전환을, ReplicaSet은 해당 버전의 Pod 수를 관리하는 구조다.

## Service, Ingress

### Service 개념

Pod의 IP는 바뀔 수 있으므로 클라이언트가 개별 Pod 주소에 의존하면 연결을 유지하기 어렵다. Service는 Pod 집합 앞에 안정적인 IP와 DNS 이름을 제공해 이 문제를 해결한다.

예를 들어 클라이언트는 `my-svc.default.svc` 또는 ClusterIP `10.96.0.10`으로 요청한다. Service 뒤의 Pod 1, 2, 3은 `10.244.x.x` 대역에서 IP가 바뀌더라도, 클라이언트는 같은 Service 주소를 사용할 수 있다.

### Service 유형

- 유형: ClusterIP:
  - 접근 범위: 클러스터 내부
  - 사용 사례: 내부 서비스 통신 (기본값)
- 유형: NodePort:
  - 접근 범위: 노드 IP:Port
  - 사용 사례: 개발/테스트, 외부 접근
- 유형: LoadBalancer:
  - 접근 범위: 외부 LB IP
  - 사용 사례: 프로덕션 외부 노출
- 유형: ExternalName:
  - 접근 범위: 외부 DNS
  - 사용 사례: 외부 서비스 참조

### ClusterIP Service

```yaml
# clusterip-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  type: ClusterIP  # 기본값
  selector:
    app: my-app
  ports:
  - name: http
    port: 80        # Service 포트
    targetPort: 8080  # Pod 포트
    protocol: TCP
```

```bash
# 클러스터 내부에서 접근
curl http://my-service.default.svc.cluster.local:80
curl http://my-service.default:80
curl http://my-service:80  # 같은 네임스페이스
```

### NodePort Service

```yaml
# nodeport-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: my-nodeport-service
spec:
  type: NodePort
  selector:
    app: my-app
  ports:
  - port: 80
    targetPort: 8080
    nodePort: 30080  # 30000-32767 범위, 생략 시 자동 할당
```

```bash
# 외부에서 접근
curl http://<NODE_IP>:30080
```

### LoadBalancer Service

```yaml
# loadbalancer-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: my-loadbalancer
  annotations:
    # AWS NLB 사용
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
    service.beta.kubernetes.io/aws-load-balancer-internal: "false"
spec:
  type: LoadBalancer
  selector:
    app: my-app
  ports:
  - port: 443
    targetPort: 8080
  # 특정 IP 지정 (선택)
  loadBalancerIP: 1.2.3.4
```

### Headless Service

```yaml
# headless-service.yaml
# StatefulSet과 함께 사용, 개별 Pod DNS 제공
apiVersion: v1
kind: Service
metadata:
  name: my-headless-service
spec:
  clusterIP: None  # Headless 설정
  selector:
    app: my-statefulset
  ports:
  - port: 5432
```

```bash
# 개별 Pod DNS
# <pod-name>.<service-name>.<namespace>.svc.cluster.local
nslookup my-statefulset-0.my-headless-service.default.svc.cluster.local
```

### Ingress

- Ingress는 HTTP/HTTPS 트래픽을 클러스터 내부 Service로 라우팅함.

```yaml
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  ingressClassName: nginx

  tls:
  - hosts:
    - api.example.com
    - www.example.com
    secretName: example-tls

  rules:
  - host: api.example.com
    http:
      paths:
      - path: /v1
        pathType: Prefix
        backend:
          service:
            name: api-v1-service
            port:
              number: 80
      - path: /v2
        pathType: Prefix
        backend:
          service:
            name: api-v2-service
            port:
              number: 80

  - host: www.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
```

### Ingress 트래픽 흐름

외부 요청은 Cloud Load Balancer를 거쳐 Ingress Controller로 들어온다. nginx, traefik, AWS ALB 등의 컨트롤러는 Ingress에 선언된 호스트와 경로 규칙에 따라 요청을 Service로 보내고, 각 Service가 뒤의 Pod로 연결한다.

예를 들어 `api.example.com`의 `/v1`은 `api-v1-service`로, `/v2`는 `api-v2-service`로 보낸다. `www.example.com`의 `/`는 `web-service`로 연결해 하나의 진입점에서 여러 서비스를 구분할 수 있다.

### Gateway API (Ingress 차세대)

```yaml
# Gateway API - Ingress의 진화
# 2025년 Ingress NGINX 지원 종료 예정, Gateway API 권장

apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: my-gateway
spec:
  gatewayClassName: nginx
  listeners:
  - name: http
    port: 80
    protocol: HTTP
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: my-route
spec:
  parentRefs:
  - name: my-gateway
  hostnames:
  - "api.example.com"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /v1
    backendRefs:
    - name: api-v1-service
      port: 80
```

## ConfigMap, Secret

### ConfigMap

- ConfigMap은 설정 데이터를 Key-Value 형태로 저장함.

```yaml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  # 단순 Key-Value
  APP_ENV: "production"
  LOG_LEVEL: "info"

  # 설정 파일 전체
  application.yml: |
    server:
      port: 8080
    spring:
      profiles:
        active: production
    logging:
      level:
        root: INFO

  # JSON 설정
  config.json: |
    {
      "database": {
        "host": "postgres",
        "port": 5432
      }
    }
```

### ConfigMap 사용 방법

```yaml
# Pod에서 ConfigMap 사용
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
  - name: app
    image: my-app:v1

    # 1. 환경변수로 사용 (개별 키)
    env:
    - name: APP_ENVIRONMENT
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: APP_ENV

    # 2. 환경변수로 사용 (전체)
    envFrom:
    - configMapRef:
        name: app-config

    # 3. 볼륨으로 마운트 (파일)
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config
      readOnly: true

    # 4. 특정 키만 마운트
    - name: specific-config
      mountPath: /app/application.yml
      subPath: application.yml

  volumes:
  - name: config-volume
    configMap:
      name: app-config
  - name: specific-config
    configMap:
      name: app-config
      items:
      - key: application.yml
        path: application.yml
```

### Secret

- Secret은 민감한 데이터(비밀번호, 토큰, 인증서)를 저장함.

```yaml
# secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
data:
  # Base64 인코딩 필수
  username: YWRtaW4=        # echo -n "admin" | base64
  password: cGFzc3dvcmQxMjM= # echo -n "password123" | base64

---
# stringData 사용 시 자동 인코딩
apiVersion: v1
kind: Secret
metadata:
  name: db-secret-string
type: Opaque
stringData:
  username: admin
  password: password123
```

### Secret 유형

```yaml
# 1. Opaque (기본, 임의 데이터)
type: Opaque

# 2. TLS 인증서
apiVersion: v1
kind: Secret
metadata:
  name: tls-secret
type: kubernetes.io/tls
data:
  tls.crt: <base64-encoded-cert>
  tls.key: <base64-encoded-key>

# 3. Docker Registry 인증
apiVersion: v1
kind: Secret
metadata:
  name: registry-secret
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: <base64-encoded-docker-config>

# 4. Basic Auth
type: kubernetes.io/basic-auth

# 5. SSH Auth
type: kubernetes.io/ssh-auth
```

### Secret 사용 방법

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
  - name: app
    image: my-app:v1

    # 환경변수로 사용
    env:
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: password

    # 볼륨으로 마운트
    volumeMounts:
    - name: secret-volume
      mountPath: /etc/secrets
      readOnly: true

  volumes:
  - name: secret-volume
    secret:
      secretName: db-secret
      defaultMode: 0400  # 파일 권한 설정

  # Private Registry 이미지 Pull
  imagePullSecrets:
  - name: registry-secret
```

### Secret 보안 모범 사례

```yaml
# 1. RBAC으로 Secret 접근 제한
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: secret-reader
rules:
- apiGroups: [""]
  resources: ["secrets"]
  resourceNames: ["db-secret"]  # 특정 Secret만 허용
  verbs: ["get"]

# 2. etcd 암호화 설정
# /etc/kubernetes/manifests/kube-apiserver.yaml
# --encryption-provider-config=/etc/kubernetes/enc/enc.yaml

# 3. External Secret Manager 사용 권장
# - AWS Secrets Manager
# - HashiCorp Vault
# - Azure Key Vault
```

## PersistentVolume, PersistentVolumeClaim

### 스토리지 개념

- Storage in Kubernetes
- Developer, Administrator
- (PVC 생성), (PV 프로비저닝)
- PersistentVolume ←, StorageClass
- Claim, 동적, (Dynamic Prov.)
- "10Gi 필요", 생성
- 바인딩, 프로비저닝
- PersistentVolume
- (실제 스토리지 추상화)
- Actual Storage
- (AWS EBS, GCE PD, NFS, Ceph, Local, etc.)

### PersistentVolume (PV)

```yaml
# pv.yaml - 관리자가 생성
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
spec:
  capacity:
    storage: 10Gi

  accessModes:
  - ReadWriteOnce   # RWO: 단일 노드에서 읽기/쓰기
  # - ReadOnlyMany  # ROX: 여러 노드에서 읽기
  # - ReadWriteMany # RWX: 여러 노드에서 읽기/쓰기

  persistentVolumeReclaimPolicy: Retain
  # Retain: PVC 삭제 시 PV 유지 (데이터 보존)
  # Delete: PVC 삭제 시 PV도 삭제
  # Recycle: 데이터 삭제 후 재사용 (deprecated)

  storageClassName: standard

  # 스토리지 백엔드 설정
  # AWS EBS
  awsElasticBlockStore:
    volumeID: vol-12345
    fsType: ext4

  # 또는 NFS
  # nfs:
  #   server: nfs-server.example.com
  #   path: /exports/data

  # 또는 Local Volume
  # local:
  #   path: /mnt/disks/ssd1
  # nodeAffinity:
  #   required:
  #     nodeSelectorTerms:
  #     - matchExpressions:
  #       - key: kubernetes.io/hostname
  #         operator: In
  #         values:
  #         - node1
```

### PersistentVolumeClaim (PVC)

```yaml
# pvc.yaml - 개발자가 생성
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
  - ReadWriteOnce

  resources:
    requests:
      storage: 5Gi

  storageClassName: standard

  # 특정 PV 지정 (선택)
  # volumeName: my-pv

  # Label Selector로 PV 선택 (선택)
  # selector:
  #   matchLabels:
  #     type: ssd
```

### StorageClass (동적 프로비저닝)

```yaml
# storageclass.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
  annotations:
    storageclass.kubernetes.io/is-default-class: "false"
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
  encrypted: "true"
reclaimPolicy: Delete
allowVolumeExpansion: true  # 볼륨 확장 허용
volumeBindingMode: WaitForFirstConsumer  # Pod 스케줄 시 바인딩

---
# GCP 예시
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ssd
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-ssd
  replication-type: regional-pd
```

### Pod에서 PVC 사용

```yaml
# statefulset with pvc
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:16
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
        env:
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: password

  # StatefulSet 전용: 각 Pod에 개별 PVC 생성
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: fast-ssd
      resources:
        requests:
          storage: 100Gi
```

### 볼륨 확장

```bash
# PVC 스토리지 확장 (StorageClass에서 allowVolumeExpansion: true 필요)
kubectl patch pvc my-pvc -p '{"spec":{"resources":{"requests":{"storage":"20Gi"}}}}'

# 확장 상태 확인
kubectl get pvc my-pvc -o yaml | grep -A 5 status
```

## Helm 패키지 매니저

### Helm 개요

- Helm은 Kubernetes의 패키지 매니저로, 복잡한 애플리케이션 배포를 단순화함.

- Helm Concepts
- Chart: Kubernetes 리소스 패키지
- templates/, # 템플릿 파일들
- deployment.yaml
- service.yaml
- _helpers.tpl
- Chart.yaml, # 차트 메타데이터
- values.yaml, # 기본 설정값
- charts/, # 의존성 차트
- Release: 차트의 설치 인스턴스
- Repository: 차트 저장소

### Helm 4.0 (2025년 릴리스)

```bash
# Helm 4.0 주요 변경사항
# - WebAssembly 기반 플러그인 시스템
# - 로컬 콘텐츠 기반 캐싱
# - slog를 통한 현대적 로깅
# - SDK 개선 (다른 앱에 Helm 기능 임베딩 가능)

# 설치
brew install helm  # macOS
choco install kubernetes-helm  # Windows
```

### 기본 명령어

```bash
# Repository 관리
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm repo list
helm search repo postgresql

# Chart 정보 확인
helm show chart bitnami/postgresql
helm show values bitnami/postgresql > values.yaml

# 설치
helm install my-postgres bitnami/postgresql \
  --namespace database \
  --create-namespace \
  --values custom-values.yaml \
  --set auth.postgresPassword=mypassword

# 설치된 릴리스 확인
helm list -A
helm status my-postgres -n database

# 업그레이드
helm upgrade my-postgres bitnami/postgresql \
  -n database \
  --values custom-values.yaml \
  --set image.tag=16.1

# 롤백
helm rollback my-postgres 1 -n database

# 삭제
helm uninstall my-postgres -n database
```

### Chart 구조

- my-chart/
- Chart.yaml, # 차트 정보 (이름, 버전, 의존성)
- Chart.lock, # 의존성 잠금 파일
- values.yaml, # 기본 설정값
- values.schema.json, # values 스키마 (선택)
- templates/
- NOTES.txt, # 설치 후 메시지
- _helpers.tpl, # 재사용 가능한 템플릿 헬퍼
- deployment.yaml
- service.yaml
- configmap.yaml
- secret.yaml
- ingress.yaml
- hpa.yaml
- charts/, # 의존성 차트
- crds/, # CRD 정의

### Chart.yaml

```yaml
# Chart.yaml
apiVersion: v2
name: my-app
description: My Application Helm Chart
type: application  # 또는 library
version: 1.2.0     # Chart 버전
appVersion: "2.0.0"  # 앱 버전

keywords:
  - backend
  - api

maintainers:
  - name: DevTeam
    email: dev@example.com

dependencies:
  - name: postgresql
    version: "13.x.x"
    repository: https://charts.bitnami.com/bitnami
    condition: postgresql.enabled
  - name: redis
    version: "18.x.x"
    repository: https://charts.bitnami.com/bitnami
    condition: redis.enabled
```

### 템플릿 작성

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "my-app.fullname" . }}
  labels:
    {{- include "my-app.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "my-app.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "my-app.selectorLabels" . | nindent 8 }}
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
    spec:
      {{- with .Values.imagePullSecrets }}
      imagePullSecrets:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      containers:
      - name: {{ .Chart.Name }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
        imagePullPolicy: {{ .Values.image.pullPolicy }}
        ports:
        - containerPort: {{ .Values.service.targetPort }}
        {{- if .Values.resources }}
        resources:
          {{- toYaml .Values.resources | nindent 10 }}
        {{- end }}
        env:
        {{- range $key, $value := .Values.env }}
        - name: {{ $key }}
          value: {{ $value | quote }}
        {{- end }}
```

### values.yaml

```yaml
# values.yaml
replicaCount: 3

image:
  repository: my-app
  tag: ""  # Chart.appVersion 사용
  pullPolicy: IfNotPresent

imagePullSecrets: []

service:
  type: ClusterIP
  port: 80
  targetPort: 8080

ingress:
  enabled: true
  className: nginx
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
  hosts:
    - host: api.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: api-tls
      hosts:
        - api.example.com

resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 250m
    memory: 256Mi

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 80

env:
  SPRING_PROFILES_ACTIVE: production
  LOG_LEVEL: info

postgresql:
  enabled: true
  auth:
    database: myapp

redis:
  enabled: true
```

### Helm 개발 워크플로우

```bash
# 차트 생성
helm create my-chart

# 문법 검증
helm lint my-chart

# 템플릿 렌더링 확인 (dry-run)
helm template my-release my-chart --values custom-values.yaml

# 클러스터에 dry-run 설치
helm install my-release my-chart --dry-run --debug

# 의존성 업데이트
helm dependency update my-chart

# 패키징
helm package my-chart

# OCI Registry에 푸시
helm push my-chart-1.0.0.tgz oci://registry.example.com/charts
```

## RBAC

### RBAC 개요

- RBAC(Role-Based Access Control)은 Kubernetes의 권한 관리 시스템임.

- RBAC Components
- Subject, Resource
- (누가?), (무엇을?)
- User, - Pods
- Group, - Services
- ServiceAcc, - Secrets
- →, Binding, ←
- (연결)
- Role
- (어떤 권한?)
- get
- list
- create
- delete
- Namespace 범위: Role + RoleBinding
- Cluster 범위: ClusterRole + ClusterRoleBinding

### Role (네임스페이스 범위)

```yaml
# role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: default
rules:
# Pod 읽기 권한
- apiGroups: [""]  # core API group
  resources: ["pods"]
  verbs: ["get", "watch", "list"]

# Pod 로그 접근
- apiGroups: [""]
  resources: ["pods/log"]
  verbs: ["get"]

# 특정 ConfigMap만 접근 허용
- apiGroups: [""]
  resources: ["configmaps"]
  resourceNames: ["app-config"]  # 특정 리소스만
  verbs: ["get"]

---
# 개발자용 Role
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
  namespace: development
rules:
- apiGroups: ["", "apps", "batch"]
  resources: ["pods", "deployments", "jobs", "services", "configmaps"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]

# exec 권한
- apiGroups: [""]
  resources: ["pods/exec"]
  verbs: ["create"]

# 로그 접근
- apiGroups: [""]
  resources: ["pods/log"]
  verbs: ["get"]
```

### ClusterRole (클러스터 범위)

```yaml
# clusterrole.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: secret-reader
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "watch", "list"]

---
# Node 관리자 ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-admin
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "list", "watch", "update", "patch"]
- apiGroups: [""]
  resources: ["nodes/status"]
  verbs: ["update", "patch"]

---
# 읽기 전용 ClusterRole (모든 리소스)
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-viewer
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["get", "list", "watch"]
```

### RoleBinding / ClusterRoleBinding

```yaml
# rolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: default
subjects:
# 사용자
- kind: User
  name: jane
  apiGroup: rbac.authorization.k8s.io

# 그룹
- kind: Group
  name: developers
  apiGroup: rbac.authorization.k8s.io

# ServiceAccount
- kind: ServiceAccount
  name: my-service-account
  namespace: default

roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io

---
# ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: read-secrets-global
subjects:
- kind: Group
  name: monitoring
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
```

### ServiceAccount

```yaml
# serviceaccount.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app-sa
  namespace: default
automountServiceAccountToken: false  # 보안: 필요시에만 true

---
# Pod에서 ServiceAccount 사용
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  serviceAccountName: my-app-sa
  automountServiceAccountToken: true
  containers:
  - name: app
    image: my-app
```

### RBAC 모범 사례

```yaml
# 1. 최소 권한 원칙 (Principle of Least Privilege)
# - 와일드카드(*) 사용 금지
# - 필요한 리소스와 동사만 명시

# Bad
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]

# Good
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]

# 2. 기본 ServiceAccount 사용 금지
# 각 애플리케이션에 전용 ServiceAccount 생성

# 3. 민감한 리소스 접근 제한
# secrets, configmaps의 특정 항목만 허용
rules:
- apiGroups: [""]
  resources: ["secrets"]
  resourceNames: ["allowed-secret"]
  verbs: ["get"]

# 4. 읽기와 쓰기 권한 분리
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]  # 읽기만

---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-admin
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch", "create", "update", "delete"]
```

### 권한 확인

```bash
# 현재 사용자 권한 확인
kubectl auth can-i create pods
kubectl auth can-i delete deployments --namespace production

# 다른 사용자/SA 권한 확인
kubectl auth can-i create pods --as=jane
kubectl auth can-i list secrets --as=system:serviceaccount:default:my-sa

# 전체 권한 목록
kubectl auth can-i --list

# 누가 특정 작업 가능한지 확인
kubectl auth reconcile -f rbac.yaml --dry-run=client
```

## 참고 자료

- [Kubernetes 공식 문서](https://kubernetes.io/docs/)
- [Kubernetes Architecture](https://kubernetes.io/docs/concepts/architecture/)
- [Kubernetes Services, Ingress](https://kubernetes.io/docs/concepts/services-networking/)
- [Kubernetes RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
- [Helm 공식 문서](https://helm.sh/docs/)
- [Gateway API](https://gateway-api.sigs.k8s.io/)

## 관련 학습

- [Docker](01-Docker.md)
- [컨테이너 오케스트레이션](03-컨테이너-오케스트레이션.md)
