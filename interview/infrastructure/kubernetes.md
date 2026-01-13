# Kubernetes / 쿠버네티스

> 카테고리: 컨테이너 오케스트레이션
> [← 면접 질문 목록으로 돌아가기](../interview.md)

---

## 📌 Kubernetes 아키텍처 - Control Plane

### K8S-001
Kubernetes의 전체 아키텍처를 설명하고, Control Plane과 Worker Node의 역할 차이를 설명해주세요.

### K8S-002
kube-apiserver의 역할과 동작 방식에 대해 설명해주세요. 다른 컴포넌트들과 어떻게 통신하나요?

### K8S-003
etcd의 역할과 중요성에 대해 설명해주세요. 왜 etcd의 백업이 중요한가요?

### K8S-004
kube-scheduler의 스케줄링 과정을 단계별로 설명해주세요. Filtering과 Scoring 단계는 무엇인가요?

### K8S-005
kube-controller-manager에 포함된 주요 컨트롤러들과 각각의 역할에 대해 설명해주세요.

### K8S-006
cloud-controller-manager의 역할과 클라우드 프로바이더와의 통합 방식에 대해 설명해주세요.

---

## 📌 Kubernetes 아키텍처 - Node 컴포넌트

### K8S-007
kubelet의 역할과 동작 방식에 대해 설명해주세요. Pod의 상태를 어떻게 관리하나요?

### K8S-008
kube-proxy의 역할과 iptables/IPVS 모드의 차이점에 대해 설명해주세요.

### K8S-009
Container Runtime Interface(CRI)란 무엇이며, containerd와 CRI-O의 차이점은 무엇인가요?

### K8S-010
Kubernetes에서 사용되는 CNI(Container Network Interface)란 무엇이며, 주요 CNI 플러그인들을 비교해주세요.

---

## 📌 Pod 기본 개념과 생명주기

### K8S-011
Pod란 무엇이며, 왜 컨테이너 대신 Pod 단위로 관리하나요?

### K8S-012
Pod의 생명주기(Lifecycle) 단계(Pending, Running, Succeeded, Failed, Unknown)에 대해 각각 설명해주세요.

### K8S-013
Pod의 재시작 정책(restartPolicy)인 Always, OnFailure, Never의 차이점과 사용 시나리오를 설명해주세요.

### K8S-014
Pod가 Pending 상태에 머무는 원인들과 해결 방법을 설명해주세요.

### K8S-015
Pod Phase와 Container State의 차이점은 무엇인가요?

---

## 📌 Pod 다중 컨테이너 패턴

### K8S-016
Sidecar 패턴이란 무엇이며, 어떤 상황에서 사용하나요? 구체적인 예시를 들어주세요.

### K8S-017
Ambassador 패턴이란 무엇이며, 프록시 역할을 하는 컨테이너의 활용 사례를 설명해주세요.

### K8S-018
Adapter 패턴이란 무엇이며, 로그 포맷 변환 등의 활용 사례를 설명해주세요.

### K8S-019
Init Container의 역할과 일반 컨테이너와의 차이점을 설명해주세요.

### K8S-020
Init Container의 실행 순서와 실패 시 동작에 대해 설명해주세요.

---

## 📌 워크로드 리소스 - Deployment & ReplicaSet

### K8S-021
Deployment의 역할과 ReplicaSet과의 관계에 대해 설명해주세요.

### K8S-022
Deployment의 배포 전략(RollingUpdate, Recreate)을 비교하고, 각각의 사용 시나리오를 설명해주세요.

### K8S-023
RollingUpdate 전략에서 maxSurge와 maxUnavailable 설정의 의미와 적절한 값 설정 방법을 설명해주세요.

### K8S-024
Deployment의 롤백(rollback) 방법과 revision history 관리에 대해 설명해주세요.

### K8S-025
Blue-Green 배포와 Canary 배포를 Kubernetes에서 구현하는 방법을 설명해주세요.

---

## 📌 워크로드 리소스 - StatefulSet

### K8S-026
StatefulSet이란 무엇이며, Deployment와의 차이점을 설명해주세요.

### K8S-027
StatefulSet에서 Pod 이름과 네트워크 ID의 안정성(stable identity)은 어떻게 보장되나요?

### K8S-028
StatefulSet의 순차적 배포(ordered deployment)와 병렬 배포(parallel deployment) 방식의 차이를 설명해주세요.

### K8S-029
StatefulSet에서 PersistentVolumeClaim 템플릿의 역할과 동작 방식을 설명해주세요.

### K8S-030
StatefulSet 사용 시 Headless Service가 필요한 이유를 설명해주세요.

---

## 📌 워크로드 리소스 - DaemonSet, Job, CronJob

### K8S-031
DaemonSet의 역할과 사용 사례(로그 수집, 모니터링 에이전트 등)를 설명해주세요.

### K8S-032
DaemonSet에서 특정 노드에만 Pod를 배포하는 방법을 설명해주세요.

### K8S-033
Job의 역할과 completions, parallelism 설정의 의미를 설명해주세요.

### K8S-034
Job의 backoffLimit와 activeDeadlineSeconds 설정의 역할을 설명해주세요.

### K8S-035
CronJob의 역할과 스케줄 표현식, concurrencyPolicy 설정에 대해 설명해주세요.

---

## 📌 서비스 & 네트워킹 - Service 타입

### K8S-036
Kubernetes Service의 역할과 필요성에 대해 설명해주세요.

### K8S-037
ClusterIP 타입 Service의 동작 방식과 사용 시나리오를 설명해주세요.

### K8S-038
NodePort 타입 Service의 동작 방식과 포트 범위 제한에 대해 설명해주세요.

### K8S-039
LoadBalancer 타입 Service의 동작 방식과 클라우드 환경에서의 프로비저닝 과정을 설명해주세요.

### K8S-040
ExternalName 타입 Service의 역할과 사용 사례를 설명해주세요.

### K8S-041
Headless Service란 무엇이며, StatefulSet과 함께 사용되는 이유를 설명해주세요.

---

## 📌 서비스 & 네트워킹 - Ingress

### K8S-042
Ingress의 역할과 Service와의 차이점을 설명해주세요.

### K8S-043
Ingress Controller의 역할과 주요 구현체(NGINX, Traefik, HAProxy 등)를 비교해주세요.

### K8S-044
Ingress에서 호스트 기반 라우팅과 경로 기반 라우팅을 설정하는 방법을 설명해주세요.

### K8S-045
Ingress에서 TLS/SSL 인증서를 설정하는 방법과 cert-manager와의 연동에 대해 설명해주세요.

### K8S-046
Ingress의 annotations을 활용한 설정(rate limiting, rewrites 등) 방법을 설명해주세요.

---

## 📌 스토리지 - PV, PVC, StorageClass

### K8S-047
PersistentVolume(PV)과 PersistentVolumeClaim(PVC)의 개념과 관계를 설명해주세요.

### K8S-048
PV의 접근 모드(ReadWriteOnce, ReadOnlyMany, ReadWriteMany)의 차이점을 설명해주세요.

### K8S-049
PV의 Reclaim Policy(Retain, Delete, Recycle)의 차이점과 사용 시나리오를 설명해주세요.

### K8S-050
StorageClass의 역할과 동적 프로비저닝(Dynamic Provisioning)의 동작 방식을 설명해주세요.

### K8S-051
CSI(Container Storage Interface)의 역할과 주요 CSI 드라이버들에 대해 설명해주세요.

### K8S-052
emptyDir, hostPath, configMap, secret 볼륨 타입의 차이점과 사용 사례를 설명해주세요.

---

## 📌 컨피그 & 시크릿

### K8S-053
ConfigMap의 역할과 생성 방법(literal, file, directory)에 대해 설명해주세요.

### K8S-054
ConfigMap을 Pod에 주입하는 방법(환경변수, 볼륨 마운트)의 차이점을 설명해주세요.

### K8S-055
Secret의 역할과 ConfigMap과의 차이점을 설명해주세요. Secret은 정말 안전한가요?

### K8S-056
Secret의 타입(Opaque, kubernetes.io/dockerconfigjson, kubernetes.io/tls 등)에 대해 설명해주세요.

### K8S-057
외부 시크릿 관리 도구(Vault, AWS Secrets Manager)와 Kubernetes Secret의 연동 방법을 설명해주세요.

### K8S-058
ConfigMap/Secret 변경 시 Pod에 자동으로 반영되지 않는 이유와 해결 방법을 설명해주세요.

---

## 📌 스케줄링 - nodeSelector, Affinity

### K8S-059
nodeSelector를 사용한 Pod 스케줄링 방법과 한계점을 설명해주세요.

### K8S-060
Node Affinity와 nodeSelector의 차이점, requiredDuringSchedulingIgnoredDuringExecution와 preferredDuringSchedulingIgnoredDuringExecution의 차이를 설명해주세요.

### K8S-061
Pod Affinity와 Pod Anti-Affinity의 개념과 사용 사례를 설명해주세요.

### K8S-062
topologyKey의 역할과 topology spread constraints의 활용 방법을 설명해주세요.

---

## 📌 스케줄링 - Taint & Toleration

### K8S-063
Taint와 Toleration의 개념과 동작 방식을 설명해주세요.

### K8S-064
Taint의 effect(NoSchedule, PreferNoSchedule, NoExecute)의 차이점을 설명해주세요.

### K8S-065
Master/Control Plane 노드에 Pod가 스케줄되지 않는 이유와 이를 허용하는 방법을 설명해주세요.

### K8S-066
Node에 문제가 생겼을 때 자동으로 적용되는 Taint(node.kubernetes.io/not-ready 등)에 대해 설명해주세요.

---

## 📌 리소스 관리 - Requests & Limits

### K8S-067
컨테이너의 resource requests와 limits의 차이점과 역할을 설명해주세요.

### K8S-068
CPU와 Memory 리소스 단위(millicore, Mi, Gi)에 대해 설명해주세요.

### K8S-069
requests만 설정했을 때와 limits만 설정했을 때의 동작 차이를 설명해주세요.

### K8S-070
Memory limits을 초과했을 때와 CPU limits을 초과했을 때의 동작 차이를 설명해주세요.

---

## 📌 리소스 관리 - QoS, LimitRange, ResourceQuota

### K8S-071
Pod의 QoS(Quality of Service) 클래스(Guaranteed, Burstable, BestEffort)의 결정 기준과 의미를 설명해주세요.

### K8S-072
LimitRange의 역할과 설정 방법(default, defaultRequest, min, max)을 설명해주세요.

### K8S-073
ResourceQuota의 역할과 네임스페이스 단위 리소스 제한 방법을 설명해주세요.

### K8S-074
PriorityClass의 역할과 Pod 우선순위 기반 스케줄링/프리엠션에 대해 설명해주세요.

---

## 📌 오토스케일링 - HPA

### K8S-075
HPA(Horizontal Pod Autoscaler)의 동작 원리와 설정 방법을 설명해주세요.

### K8S-076
HPA에서 CPU/Memory 기반 스케일링과 Custom Metrics 기반 스케일링의 차이를 설명해주세요.

### K8S-077
HPA의 스케일링 알고리즘과 stabilizationWindowSeconds 설정의 역할을 설명해주세요.

### K8S-078
HPA 사용 시 주의사항과 Best Practice를 설명해주세요.

---

## 📌 오토스케일링 - VPA & Cluster Autoscaler

### K8S-079
VPA(Vertical Pod Autoscaler)의 동작 원리와 HPA와의 차이점을 설명해주세요.

### K8S-080
VPA의 updateMode(Off, Initial, Auto)의 차이점을 설명해주세요.

### K8S-081
Cluster Autoscaler의 동작 원리와 노드 추가/삭제 조건을 설명해주세요.

### K8S-082
HPA, VPA, Cluster Autoscaler를 함께 사용할 때의 고려사항을 설명해주세요.

---

## 📌 보안 - RBAC

### K8S-083
RBAC(Role-Based Access Control)의 개념과 구성 요소(Role, ClusterRole, RoleBinding, ClusterRoleBinding)를 설명해주세요.

### K8S-084
Role과 ClusterRole의 차이점, RoleBinding과 ClusterRoleBinding의 차이점을 설명해주세요.

### K8S-085
RBAC에서 verbs(get, list, watch, create, update, patch, delete)의 의미를 설명해주세요.

### K8S-086
최소 권한 원칙(Principle of Least Privilege)을 Kubernetes RBAC에서 적용하는 방법을 설명해주세요.

---

## 📌 보안 - ServiceAccount & 인증

### K8S-087
ServiceAccount의 역할과 Pod에서의 사용 방법을 설명해주세요.

### K8S-088
ServiceAccount 토큰의 자동 마운트와 이를 비활성화하는 방법을 설명해주세요.

### K8S-089
Kubernetes API 서버의 인증(Authentication) 방식들(X.509, Bearer Token, OIDC 등)을 설명해주세요.

### K8S-090
kubeconfig 파일의 구조와 contexts, clusters, users 설정에 대해 설명해주세요.

---

## 📌 보안 - NetworkPolicy & Pod Security

### K8S-091
NetworkPolicy의 역할과 Ingress/Egress 규칙 설정 방법을 설명해주세요.

### K8S-092
NetworkPolicy가 적용되지 않는 경우(CNI 미지원 등)와 기본 정책에 대해 설명해주세요.

### K8S-093
Pod Security Standards(Privileged, Baseline, Restricted)의 차이점을 설명해주세요.

### K8S-094
Pod Security Admission Controller의 역할과 enforce, audit, warn 모드의 차이를 설명해주세요.

### K8S-095
컨테이너의 securityContext 설정(runAsUser, runAsNonRoot, readOnlyRootFilesystem 등)에 대해 설명해주세요.

---

## 📌 헬스 체크 - Probe

### K8S-096
Liveness Probe의 역할과 설정 방법(httpGet, tcpSocket, exec)을 설명해주세요.

### K8S-097
Readiness Probe의 역할과 Liveness Probe와의 차이점을 설명해주세요.

### K8S-098
Startup Probe의 역할과 느린 시작 애플리케이션에서의 활용 방법을 설명해주세요.

### K8S-099
Probe 설정값(initialDelaySeconds, periodSeconds, timeoutSeconds, failureThreshold)의 의미와 적절한 설정 방법을 설명해주세요.

### K8S-100
잘못된 Probe 설정으로 인한 문제(CrashLoopBackOff, 서비스 불가 등)와 해결 방법을 설명해주세요.

---

## 📌 로깅 & 모니터링

### K8S-101
kubectl logs 명령어의 다양한 옵션(-f, --previous, -c, --since)을 설명해주세요.

### K8S-102
Kubernetes에서의 로깅 아키텍처와 노드 레벨/클러스터 레벨 로깅의 차이를 설명해주세요.

### K8S-103
metrics-server의 역할과 kubectl top 명령어 사용 방법을 설명해주세요.

### K8S-104
Prometheus를 활용한 Kubernetes 모니터링 구성 방법을 설명해주세요.

### K8S-105
Kubernetes에서 수집해야 하는 주요 메트릭(Node, Pod, Container 레벨)을 설명해주세요.

---

## 📌 Helm & 패키지 관리

### K8S-106
Helm의 역할과 Chart, Release, Repository의 개념을 설명해주세요.

### K8S-107
Helm Chart의 구조(Chart.yaml, values.yaml, templates/)를 설명해주세요.

### K8S-108
Helm의 템플릿 함수와 values.yaml을 통한 설정 오버라이드 방법을 설명해주세요.

### K8S-109
Helm의 Release 관리(install, upgrade, rollback, uninstall)와 Revision에 대해 설명해주세요.

### K8S-110
Helm Hooks의 역할과 pre-install, post-install 등의 사용 사례를 설명해주세요.

---

## 📌 클러스터 관리 - 업그레이드 & 백업

### K8S-111
Kubernetes 클러스터 버전 업그레이드 절차와 주의사항을 설명해주세요.

### K8S-112
Control Plane 업그레이드와 Worker Node 업그레이드의 순서와 방법을 설명해주세요.

### K8S-113
etcd 백업과 복구 방법을 설명해주세요.

### K8S-114
kubectl drain과 cordon 명령어의 역할과 차이점을 설명해주세요.

### K8S-115
노드 유지보수 시 Pod 안전하게 이동시키는 방법과 PodDisruptionBudget의 역할을 설명해주세요.

---

## 📌 트러블슈팅 & 디버깅

### K8S-116
Pod가 CrashLoopBackOff 상태일 때의 원인 분석과 해결 방법을 설명해주세요.

### K8S-117
Pod가 ImagePullBackOff 상태일 때의 원인과 해결 방법을 설명해주세요.

### K8S-118
kubectl describe, kubectl logs, kubectl exec를 활용한 디버깅 방법을 설명해주세요.

### K8S-119
kubectl debug 명령어를 활용한 ephemeral container 디버깅 방법을 설명해주세요.

### K8S-120
Service에 연결되지 않는 Pod 문제 해결 방법(selector, endpoints 확인 등)을 설명해주세요.

### K8S-121
DNS 관련 문제 해결 방법(CoreDNS 확인, nslookup 테스트 등)을 설명해주세요.

---

## 📌 서비스 메시 - Istio & Linkerd

### K8S-122
서비스 메시(Service Mesh)의 개념과 필요성에 대해 설명해주세요.

### K8S-123
Sidecar Proxy 패턴과 서비스 메시에서의 트래픽 제어 방식을 설명해주세요.

### K8S-124
Istio의 아키텍처와 주요 컴포넌트(Envoy, Istiod)에 대해 설명해주세요.

### K8S-125
Istio의 트래픽 관리 기능(VirtualService, DestinationRule)에 대해 설명해주세요.

### K8S-126
Istio의 보안 기능(mTLS, Authorization Policy)에 대해 설명해주세요.

### K8S-127
Linkerd의 특징과 Istio와의 비교를 설명해주세요.

---

## 📌 CRD & Operator 패턴

### K8S-128
CRD(Custom Resource Definition)의 개념과 Kubernetes 확장 방법을 설명해주세요.

### K8S-129
Custom Resource와 Custom Controller의 관계를 설명해주세요.

### K8S-130
Operator 패턴이란 무엇이며, 어떤 상황에서 사용하나요?

### K8S-131
Operator Framework(Operator SDK, Kubebuilder)를 활용한 Operator 개발 방법을 설명해주세요.

### K8S-132
유명한 Operator 사례(Prometheus Operator, MySQL Operator 등)와 그 장점을 설명해주세요.

---

## 📌 고급 네트워킹

### K8S-133
Kubernetes 클러스터 내 Pod 간 통신 원리를 설명해주세요.

### K8S-134
Service의 ClusterIP가 동작하는 원리(kube-proxy, iptables/IPVS)를 설명해주세요.

### K8S-135
Pod에서 외부 서비스로 통신할 때의 네트워크 흐름을 설명해주세요.

### K8S-136
Gateway API의 개념과 Ingress와의 차이점을 설명해주세요.

---

## 📌 멀티 클러스터 & GitOps

### K8S-137
멀티 클러스터 관리의 필요성과 주요 고려사항을 설명해주세요.

### K8S-138
Federation의 개념과 멀티 클러스터 배포 전략을 설명해주세요.

### K8S-139
GitOps의 개념과 ArgoCD를 활용한 Kubernetes 배포 방법을 설명해주세요.

### K8S-140
Flux와 ArgoCD의 비교 및 선택 기준을 설명해주세요.

---
