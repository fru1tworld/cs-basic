# Google Cloud Platform (GCP)

## 목차
1. [개요](#개요)
2. [Compute Engine](#compute-engine)
3. [Google Kubernetes Engine (GKE)](#google-kubernetes-engine-gke)
4. [Cloud Run](#cloud-run)
5. [BigQuery](#bigquery)
6. [Cloud Functions](#cloud-functions)
7. [데이터베이스 서비스](#데이터베이스-서비스)
8. [네트워킹](#네트워킹)
---

## 개요

Google Cloud Platform(GCP)은 Google이 제공하는 클라우드 서비스 플랫폼으로, Google 검색, Gmail, YouTube 등에 사용되는 동일한 인프라에서 실행됩니다.

### GCP 시장 현황 (2025)
- 글로벌 클라우드 시장 점유율: 약 12%
- 2025년 2분기 클라우드 매출: $136억 (전년 대비 32% 성장)
- Gartner Magic Quadrant 2025 컨테이너 관리 부문 리더

### GCP의 핵심 강점
```
┌─────────────────────────────────────────────────────────────────┐
│                     GCP Core Strengths                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. 데이터 분석 & 머신러닝                                        │
│     - BigQuery: 페타바이트급 분석                                │
│     - Vertex AI: 통합 ML 플랫폼                                  │
│     - TensorFlow 네이티브 지원                                   │
│                                                                  │
│  2. 컨테이너 & Kubernetes                                        │
│     - GKE: 업계 최초 완전관리형 K8s                              │
│     - Cloud Run: 서버리스 컨테이너                               │
│     - Anthos: 하이브리드/멀티클라우드                            │
│                                                                  │
│  3. 글로벌 네트워크                                              │
│     - Premium Tier: Google 글로벌 백본 사용                      │
│     - 낮은 레이턴시, 높은 처리량                                 │
│     - 30+ 리전, 100+ 존                                         │
│                                                                  │
│  4. 개발자 경험                                                  │
│     - Cloud Shell: 브라우저 기반 CLI                             │
│     - Cloud Code: IDE 통합                                       │
│     - Firebase: 모바일/웹 개발 플랫폼                            │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### GCP 리전 구조
```
┌─────────────────────────────────────────────────────────────────┐
│                    GCP Global Infrastructure                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Region (리전)                                                   │
│  └── Zone (영역) - 독립적인 데이터센터                           │
│      ├── Zone A (asia-northeast3-a) ← 서울                      │
│      ├── Zone B (asia-northeast3-b)                             │
│      └── Zone C (asia-northeast3-c)                             │
│                                                                  │
│  Multi-Region (멀티 리전)                                        │
│  └── asia, us, eu 등 대륙 단위                                  │
│      - Cloud Storage, BigQuery 등에서 사용                       │
│      - 자동 복제 및 고가용성                                     │
│                                                                  │
│  Edge PoP (Point of Presence)                                    │
│  └── 전 세계 200+ 지점                                          │
│      - Cloud CDN                                                 │
│      - Media CDN                                                 │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Compute Engine

Compute Engine은 GCP의 IaaS(Infrastructure as a Service) 서비스로, 확장 가능한 가상 머신을 제공합니다.

### 머신 유형
| 머신 패밀리 | 용도 | 사용 사례 |
|------------|------|-----------|
| **E2** | 비용 효율적 | 개발/테스트, 소규모 워크로드 |
| **N2/N2D** | 범용 균형 | 웹 서버, 앱 서버, 중간 규모 DB |
| **C2/C2D** | 컴퓨팅 최적화 | 게임 서버, 과학 연산, HPC |
| **M2/M3** | 메모리 최적화 | 대규모 인메모리 DB, SAP HANA |
| **A2/A3** | 가속기 최적화 | ML 학습, 추론, HPC |
| **T2D** | Arm 기반 | 비용 효율적 스케일아웃 |

### 가격 모델
```
┌─────────────────────────────────────────────────────────────────┐
│                  Compute Engine 가격 모델                        │
├─────────────────┬───────────────┬───────────────────────────────┤
│      모델       │    할인율      │            특징               │
├─────────────────┼───────────────┼───────────────────────────────┤
│ On-Demand       │      0%       │ 초 단위 과금, 최소 1분        │
│ Committed Use   │   최대 57%    │ 1년 또는 3년 약정             │
│ Spot VMs        │   최대 91%    │ 선점 가능, 배치 작업에 적합   │
│ Sustained Use   │   최대 30%    │ 월 사용량에 따라 자동 할인    │
└─────────────────┴───────────────┴───────────────────────────────┘

특징:
- Sustained Use Discount: 자동 적용, 설정 불필요
- Committed Use Discount: 리소스 유형별 약정
- Spot VMs: 최대 24시간, 30초 경고 후 종료
```

### Compute Engine 생성 (gcloud CLI)
```bash
# 기본 인스턴스 생성
gcloud compute instances create my-web-server \
    --zone=asia-northeast3-a \
    --machine-type=e2-medium \
    --image-family=debian-12 \
    --image-project=debian-cloud \
    --boot-disk-size=20GB \
    --boot-disk-type=pd-balanced \
    --network-interface=network=my-vpc,subnet=my-subnet \
    --tags=http-server,https-server \
    --metadata=startup-script='#!/bin/bash
        apt-get update
        apt-get install -y nginx
        systemctl start nginx'

# 커스텀 머신 타입 (세밀한 리소스 조정)
gcloud compute instances create my-custom-vm \
    --zone=asia-northeast3-a \
    --custom-cpu=6 \
    --custom-memory=15GB \
    --custom-vm-type=n2
```

### Compute Engine with Terraform
```hcl
# 프로젝트 및 리전 설정
provider "google" {
  project = "my-gcp-project"
  region  = "asia-northeast3"
}

# VPC 네트워크
resource "google_compute_network" "main" {
  name                    = "main-vpc"
  auto_create_subnetworks = false
}

# 서브넷
resource "google_compute_subnetwork" "private" {
  name          = "private-subnet"
  ip_cidr_range = "10.0.1.0/24"
  region        = "asia-northeast3"
  network       = google_compute_network.main.id

  private_ip_google_access = true

  secondary_ip_range {
    range_name    = "pods"
    ip_cidr_range = "10.1.0.0/16"
  }

  secondary_ip_range {
    range_name    = "services"
    ip_cidr_range = "10.2.0.0/20"
  }
}

# 인스턴스 템플릿
resource "google_compute_instance_template" "web" {
  name_prefix  = "web-template-"
  machine_type = "e2-medium"
  region       = "asia-northeast3"

  disk {
    source_image = "debian-cloud/debian-12"
    auto_delete  = true
    boot         = true
    disk_type    = "pd-balanced"
    disk_size_gb = 20
  }

  network_interface {
    subnetwork = google_compute_subnetwork.private.id
    # Public IP 없음 (NAT 사용)
  }

  metadata = {
    startup-script = <<-EOF
      #!/bin/bash
      apt-get update
      apt-get install -y nginx
      systemctl enable nginx
      systemctl start nginx
    EOF
  }

  service_account {
    email  = google_service_account.web.email
    scopes = ["cloud-platform"]
  }

  shielded_instance_config {
    enable_secure_boot          = true
    enable_vtpm                 = true
    enable_integrity_monitoring = true
  }

  lifecycle {
    create_before_destroy = true
  }
}

# Managed Instance Group (MIG)
resource "google_compute_instance_group_manager" "web" {
  name               = "web-mig"
  base_instance_name = "web"
  zone               = "asia-northeast3-a"

  version {
    instance_template = google_compute_instance_template.web.id
  }

  target_size = 3

  named_port {
    name = "http"
    port = 80
  }

  auto_healing_policies {
    health_check      = google_compute_health_check.web.id
    initial_delay_sec = 300
  }
}

# 오토스케일러
resource "google_compute_autoscaler" "web" {
  name   = "web-autoscaler"
  zone   = "asia-northeast3-a"
  target = google_compute_instance_group_manager.web.id

  autoscaling_policy {
    max_replicas    = 10
    min_replicas    = 2
    cooldown_period = 60

    cpu_utilization {
      target = 0.6
    }
  }
}
```

---

## Google Kubernetes Engine (GKE)

GKE는 업계 최초의 완전 관리형 Kubernetes 서비스로, Google이 Kubernetes를 개발했기 때문에 가장 최신 기능을 빠르게 제공합니다.

### GKE 모드 비교
```
┌─────────────────────────────────────────────────────────────────┐
│                      GKE Operation Modes                         │
├────────────────────┬────────────────────────────────────────────┤
│     Standard       │              Autopilot                      │
├────────────────────┼────────────────────────────────────────────┤
│ 노드 직접 관리     │ 노드 자동 관리 (서버리스)                   │
│ 노드 풀 커스터마이즈│ Pod 사양만 정의                            │
│ GPU/TPU 완전 지원  │ GPU 지원 (제한적)                          │
│ DaemonSet 자유롭게 │ 시스템 DaemonSet만                         │
│ 노드 SSH 접근 가능 │ 노드 접근 불가                              │
│ 노드 단위 과금     │ Pod 리소스 단위 과금                        │
│ 최적화 직접 수행   │ 최적화 자동 적용                            │
└────────────────────┴────────────────────────────────────────────┘

권장 사항:
- Autopilot: 신규 프로젝트, 운영 부담 최소화
- Standard: GPU 집약적 워크로드, 세밀한 제어 필요
```

### GKE Autopilot 클러스터 생성
```bash
# Autopilot 클러스터 생성
gcloud container clusters create-auto my-autopilot-cluster \
    --region=asia-northeast3 \
    --release-channel=regular \
    --enable-master-authorized-networks \
    --master-authorized-networks=10.0.0.0/8 \
    --enable-private-nodes \
    --enable-private-endpoint \
    --master-ipv4-cidr=172.16.0.0/28 \
    --network=my-vpc \
    --subnetwork=my-subnet \
    --cluster-secondary-range-name=pods \
    --services-secondary-range-name=services
```

### GKE Standard 클러스터 (Terraform)
```hcl
resource "google_container_cluster" "primary" {
  name     = "my-gke-cluster"
  location = "asia-northeast3"

  # Autopilot 비활성화 (Standard 모드)
  enable_autopilot = false

  # VPC Native 클러스터
  network    = google_compute_network.main.name
  subnetwork = google_compute_subnetwork.private.name

  ip_allocation_policy {
    cluster_secondary_range_name  = "pods"
    services_secondary_range_name = "services"
  }

  # 초기 노드 풀 제거 (별도 관리)
  remove_default_node_pool = true
  initial_node_count       = 1

  # Private 클러스터 설정
  private_cluster_config {
    enable_private_nodes    = true
    enable_private_endpoint = false
    master_ipv4_cidr_block  = "172.16.0.0/28"
  }

  # 마스터 접근 제어
  master_authorized_networks_config {
    cidr_blocks {
      cidr_block   = "10.0.0.0/8"
      display_name = "VPC Internal"
    }
  }

  # 워크로드 아이덴티티 활성화 (권장)
  workload_identity_config {
    workload_pool = "${var.project_id}.svc.id.goog"
  }

  # 보안 설정
  binary_authorization {
    evaluation_mode = "PROJECT_SINGLETON_POLICY_ENFORCE"
  }

  # 릴리스 채널
  release_channel {
    channel = "REGULAR"
  }

  # 유지보수 정책
  maintenance_policy {
    recurring_window {
      start_time = "2025-01-01T09:00:00Z"
      end_time   = "2025-01-01T17:00:00Z"
      recurrence = "FREQ=WEEKLY;BYDAY=SA,SU"
    }
  }

  # 로깅 및 모니터링
  logging_config {
    enable_components = ["SYSTEM_COMPONENTS", "WORKLOADS"]
  }

  monitoring_config {
    enable_components = ["SYSTEM_COMPONENTS"]
    managed_prometheus {
      enabled = true
    }
  }

  # 네트워크 정책
  network_policy {
    enabled = true
  }
}

# 노드 풀
resource "google_container_node_pool" "primary_nodes" {
  name       = "primary-node-pool"
  location   = "asia-northeast3"
  cluster    = google_container_cluster.primary.name

  node_count = 2

  autoscaling {
    min_node_count = 1
    max_node_count = 10
  }

  management {
    auto_repair  = true
    auto_upgrade = true
  }

  node_config {
    preemptible  = false
    machine_type = "e2-standard-4"
    disk_size_gb = 100
    disk_type    = "pd-balanced"

    # Workload Identity 설정
    workload_metadata_config {
      mode = "GKE_METADATA"
    }

    # Shielded VM
    shielded_instance_config {
      enable_secure_boot          = true
      enable_integrity_monitoring = true
    }

    oauth_scopes = [
      "https://www.googleapis.com/auth/cloud-platform"
    ]

    labels = {
      env = "production"
    }

    taint {
      key    = "dedicated"
      value  = "app"
      effect = "NO_SCHEDULE"
    }
  }
}

# Spot 노드 풀 (비용 절감)
resource "google_container_node_pool" "spot_nodes" {
  name     = "spot-node-pool"
  location = "asia-northeast3"
  cluster  = google_container_cluster.primary.name

  autoscaling {
    min_node_count = 0
    max_node_count = 20
  }

  node_config {
    spot         = true
    machine_type = "e2-standard-4"

    taint {
      key    = "cloud.google.com/gke-spot"
      value  = "true"
      effect = "NO_SCHEDULE"
    }

    labels = {
      "cloud.google.com/gke-spot" = "true"
    }
  }
}
```

### Workload Identity 설정
```bash
# Kubernetes Service Account 생성
kubectl create serviceaccount my-app-sa -n my-namespace

# GCP Service Account 생성
gcloud iam service-accounts create my-app-gsa \
    --display-name="My App Service Account"

# IAM 바인딩 (KSA -> GSA)
gcloud iam service-accounts add-iam-policy-binding \
    my-app-gsa@my-project.iam.gserviceaccount.com \
    --role roles/iam.workloadIdentityUser \
    --member "serviceAccount:my-project.svc.id.goog[my-namespace/my-app-sa]"

# KSA 어노테이션 추가
kubectl annotate serviceaccount my-app-sa \
    -n my-namespace \
    iam.gke.io/gcp-service-account=my-app-gsa@my-project.iam.gserviceaccount.com
```

---

## Cloud Run

Cloud Run은 완전 관리형 서버리스 컨테이너 플랫폼으로, 컨테이너를 즉시 프로덕션에 배포할 수 있습니다.

### Cloud Run 특징
```
┌─────────────────────────────────────────────────────────────────┐
│                     Cloud Run Features                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  확장성:                                                         │
│  - 0에서 수천 개 인스턴스까지 자동 확장                          │
│  - 요청 기반 스케일링                                            │
│  - 최소 인스턴스 설정 가능 (Cold Start 방지)                     │
│                                                                  │
│  유연성:                                                         │
│  - 모든 언어/프레임워크 지원                                     │
│  - Docker 컨테이너 또는 소스 코드에서 직접 배포                   │
│  - 최대 32 vCPU, 32GB 메모리                                    │
│                                                                  │
│  네트워킹:                                                       │
│  - HTTPS 자동 제공                                               │
│  - 커스텀 도메인 지원                                            │
│  - VPC 커넥터로 Private 리소스 접근                              │
│                                                                  │
│  가격:                                                           │
│  - 요청당 과금 (100밀리초 단위)                                  │
│  - 월 200만 요청 무료                                            │
│  - CPU 항상 할당 옵션 (배경 작업용)                              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Cloud Run 배포
```bash
# 소스에서 직접 배포 (Buildpacks 사용)
gcloud run deploy my-api \
    --source . \
    --region=asia-northeast3 \
    --platform=managed \
    --allow-unauthenticated \
    --min-instances=1 \
    --max-instances=100 \
    --memory=512Mi \
    --cpu=1 \
    --timeout=60s \
    --concurrency=80 \
    --set-env-vars="NODE_ENV=production"

# 컨테이너 이미지에서 배포
gcloud run deploy my-api \
    --image=asia-northeast3-docker.pkg.dev/my-project/my-repo/my-api:v1.0.0 \
    --region=asia-northeast3 \
    --service-account=my-service-account@my-project.iam.gserviceaccount.com \
    --vpc-connector=my-vpc-connector \
    --vpc-egress=private-ranges-only \
    --set-secrets="DB_PASSWORD=db-password:latest" \
    --cpu-boost \
    --execution-environment=gen2
```

### Cloud Run with Terraform
```hcl
# Artifact Registry (컨테이너 저장소)
resource "google_artifact_registry_repository" "main" {
  location      = "asia-northeast3"
  repository_id = "my-app-repo"
  format        = "DOCKER"
}

# VPC Connector (Private 리소스 접근용)
resource "google_vpc_access_connector" "connector" {
  name          = "my-vpc-connector"
  region        = "asia-northeast3"
  network       = google_compute_network.main.name
  ip_cidr_range = "10.8.0.0/28"

  min_throughput = 200
  max_throughput = 1000
}

# Cloud Run 서비스
resource "google_cloud_run_v2_service" "api" {
  name     = "my-api"
  location = "asia-northeast3"

  template {
    service_account = google_service_account.cloudrun.email

    scaling {
      min_instance_count = 1
      max_instance_count = 100
    }

    vpc_access {
      connector = google_vpc_access_connector.connector.id
      egress    = "PRIVATE_RANGES_ONLY"
    }

    containers {
      image = "asia-northeast3-docker.pkg.dev/${var.project_id}/${google_artifact_registry_repository.main.repository_id}/my-api:latest"

      resources {
        limits = {
          cpu    = "2"
          memory = "1Gi"
        }
        cpu_idle          = false  # CPU 항상 할당
        startup_cpu_boost = true   # 시작 시 CPU 부스트
      }

      ports {
        container_port = 8080
      }

      env {
        name  = "PROJECT_ID"
        value = var.project_id
      }

      # Secret Manager에서 환경 변수 로드
      env {
        name = "DB_PASSWORD"
        value_source {
          secret_key_ref {
            secret  = google_secret_manager_secret.db_password.secret_id
            version = "latest"
          }
        }
      }

      # Health Check
      startup_probe {
        http_get {
          path = "/health"
          port = 8080
        }
        initial_delay_seconds = 5
        timeout_seconds       = 5
        period_seconds        = 10
        failure_threshold     = 3
      }

      liveness_probe {
        http_get {
          path = "/health"
          port = 8080
        }
        period_seconds    = 30
        timeout_seconds   = 5
        failure_threshold = 3
      }
    }

    timeout = "60s"

    max_instance_request_concurrency = 80
  }

  traffic {
    type    = "TRAFFIC_TARGET_ALLOCATION_TYPE_LATEST"
    percent = 100
  }
}

# IAM - 공개 접근 허용
resource "google_cloud_run_v2_service_iam_member" "public" {
  location = google_cloud_run_v2_service.api.location
  name     = google_cloud_run_v2_service.api.name
  role     = "roles/run.invoker"
  member   = "allUsers"
}

# Cloud Run Job (배치 작업용)
resource "google_cloud_run_v2_job" "batch" {
  name     = "my-batch-job"
  location = "asia-northeast3"

  template {
    template {
      service_account = google_service_account.cloudrun.email

      containers {
        image = "asia-northeast3-docker.pkg.dev/${var.project_id}/${google_artifact_registry_repository.main.repository_id}/my-batch:latest"

        resources {
          limits = {
            cpu    = "4"
            memory = "8Gi"
          }
        }
      }

      max_retries = 3
      timeout     = "3600s"
    }

    parallelism = 10
    task_count  = 100
  }
}
```

---

## BigQuery

BigQuery는 페타바이트 규모의 완전 관리형 서버리스 데이터 웨어하우스입니다.

### BigQuery 아키텍처
```
┌─────────────────────────────────────────────────────────────────┐
│                    BigQuery Architecture                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                    Dremel Execution Engine                 │  │
│  │  - 수천 개 노드에서 병렬 쿼리 실행                          │  │
│  │  - 컬럼 기반 처리                                          │  │
│  │  - 셔플 없는 조인 최적화                                   │  │
│  └───────────────────────────────────────────────────────────┘  │
│                              ↑                                   │
│                              │                                   │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                   Colossus Storage                         │  │
│  │  - 분산 파일 시스템                                        │  │
│  │  - 자동 복제 및 암호화                                     │  │
│  │  - 컬럼 지향 포맷 (Capacitor)                              │  │
│  └───────────────────────────────────────────────────────────┘  │
│                              ↑                                   │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                   Jupiter Network                          │  │
│  │  - 1 Petabit/sec 대역폭                                    │  │
│  │  - 스토리지와 컴퓨팅 분리                                   │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### BigQuery 핵심 기능
```
┌─────────────────────────────────────────────────────────────────┐
│                    BigQuery Key Features                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. 서버리스                                                     │
│     - 인프라 관리 불필요                                         │
│     - 자동 확장                                                  │
│     - On-demand 또는 Flat-rate 가격                             │
│                                                                  │
│  2. 분석 기능                                                    │
│     - SQL 기반 (GoogleSQL)                                      │
│     - ML 내장 (BigQuery ML)                                     │
│     - 지리공간 분석 (BigQuery GIS)                              │
│                                                                  │
│  3. 실시간 처리                                                  │
│     - Streaming Insert                                          │
│     - BigQuery BI Engine (인메모리)                             │
│     - Materialized Views                                        │
│                                                                  │
│  4. 연동                                                         │
│     - Cloud Storage (Federated Query)                           │
│     - Dataflow, Dataproc, Looker                                │
│     - BigQuery Omni (멀티클라우드)                              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### BigQuery 사용 예시
```sql
-- 1. 테이블 생성 (파티셔닝 + 클러스터링)
CREATE TABLE my_dataset.events (
    event_id STRING NOT NULL,
    user_id STRING NOT NULL,
    event_name STRING NOT NULL,
    event_timestamp TIMESTAMP NOT NULL,
    event_params JSON,
    device_info STRUCT<
        platform STRING,
        os_version STRING,
        app_version STRING
    >
)
PARTITION BY DATE(event_timestamp)
CLUSTER BY user_id, event_name
OPTIONS (
    partition_expiration_days = 365,
    require_partition_filter = TRUE,
    description = "앱 이벤트 로그 테이블"
);

-- 2. 스트리밍 인서트
INSERT INTO my_dataset.events
VALUES (
    GENERATE_UUID(),
    'user_123',
    'purchase',
    CURRENT_TIMESTAMP(),
    JSON '{"amount": 99.99, "currency": "KRW"}',
    STRUCT('ios', '17.0', '2.1.0')
);

-- 3. 분석 쿼리 (파티션 필터 필수)
WITH daily_revenue AS (
    SELECT
        DATE(event_timestamp) as date,
        COUNT(DISTINCT user_id) as unique_users,
        SUM(CAST(JSON_VALUE(event_params, '$.amount') AS FLOAT64)) as revenue
    FROM my_dataset.events
    WHERE
        DATE(event_timestamp) BETWEEN DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY) AND CURRENT_DATE()
        AND event_name = 'purchase'
    GROUP BY 1
)
SELECT
    date,
    unique_users,
    revenue,
    revenue / unique_users as arpu
FROM daily_revenue
ORDER BY date DESC;

-- 4. BigQuery ML - 이탈 예측 모델
CREATE OR REPLACE MODEL my_dataset.churn_prediction
OPTIONS (
    model_type = 'LOGISTIC_REG',
    input_label_cols = ['churned'],
    auto_class_weights = TRUE,
    enable_global_explain = TRUE
) AS
SELECT
    user_id,
    days_since_last_login,
    total_sessions,
    avg_session_duration,
    purchase_count,
    churned
FROM my_dataset.user_features
WHERE training_split = 'train';

-- 5. 모델 예측
SELECT
    user_id,
    predicted_churned,
    predicted_churned_probs[OFFSET(1)].prob as churn_probability
FROM ML.PREDICT(MODEL my_dataset.churn_prediction,
    (SELECT * FROM my_dataset.user_features WHERE training_split = 'predict')
)
WHERE predicted_churned_probs[OFFSET(1)].prob > 0.7
ORDER BY churn_probability DESC;
```

### BigQuery Python SDK
```python
from google.cloud import bigquery
from google.cloud.bigquery import SchemaField, TimePartitioning

client = bigquery.Client()

# 1. 데이터셋 생성
dataset_id = f"{client.project}.my_dataset"
dataset = bigquery.Dataset(dataset_id)
dataset.location = "asia-northeast3"
dataset = client.create_dataset(dataset, exists_ok=True)

# 2. 테이블 생성
schema = [
    SchemaField("event_id", "STRING", mode="REQUIRED"),
    SchemaField("user_id", "STRING", mode="REQUIRED"),
    SchemaField("event_name", "STRING", mode="REQUIRED"),
    SchemaField("event_timestamp", "TIMESTAMP", mode="REQUIRED"),
    SchemaField("event_params", "JSON"),
]

table_id = f"{dataset_id}.events"
table = bigquery.Table(table_id, schema=schema)
table.time_partitioning = TimePartitioning(
    type_=bigquery.TimePartitioningType.DAY,
    field="event_timestamp",
    expiration_ms=365 * 24 * 60 * 60 * 1000  # 365일
)
table.clustering_fields = ["user_id", "event_name"]
table.require_partition_filter = True

table = client.create_table(table, exists_ok=True)

# 3. 쿼리 실행
query = """
    SELECT
        DATE(event_timestamp) as date,
        event_name,
        COUNT(*) as count
    FROM `my_project.my_dataset.events`
    WHERE DATE(event_timestamp) = CURRENT_DATE()
    GROUP BY 1, 2
    ORDER BY count DESC
"""

query_job = client.query(query)
results = query_job.result()

for row in results:
    print(f"{row.date}: {row.event_name} - {row.count}")

# 4. 스트리밍 인서트
import json
from datetime import datetime

rows_to_insert = [
    {
        "event_id": "evt_001",
        "user_id": "user_123",
        "event_name": "page_view",
        "event_timestamp": datetime.utcnow().isoformat(),
        "event_params": json.dumps({"page": "/home"})
    }
]

errors = client.insert_rows_json(table_id, rows_to_insert)
if errors:
    print(f"Errors: {errors}")
```

---

## Cloud Functions

Cloud Functions는 이벤트 기반 서버리스 함수 플랫폼입니다.

### Cloud Functions vs Cloud Run
```
┌────────────────────┬─────────────────────┬─────────────────────┐
│       항목         │   Cloud Functions   │     Cloud Run       │
├────────────────────┼─────────────────────┼─────────────────────┤
│ 배포 단위          │ 함수               │ 컨테이너            │
│ 런타임             │ 제한된 런타임       │ 모든 컨테이너       │
│ 실행 시간          │ 최대 60분           │ 최대 60분           │
│ 메모리             │ 최대 32GB           │ 최대 32GB           │
│ vCPU               │ 최대 8              │ 최대 8              │
│ 동시성             │ 1 요청/인스턴스     │ 최대 1000/인스턴스  │
│ 트리거             │ HTTP, Events        │ HTTP 중심           │
│ 사용 사례          │ 간단한 이벤트 처리  │ 마이크로서비스      │
└────────────────────┴─────────────────────┴─────────────────────┘
```

### Cloud Functions 2nd Gen (권장)
```python
# main.py - Cloud Functions 2nd Gen
import functions_framework
from google.cloud import firestore
from google.cloud import pubsub_v1
import json

db = firestore.Client()
publisher = pubsub_v1.PublisherClient()

@functions_framework.http
def process_order(request):
    """HTTP 트리거 함수"""
    request_json = request.get_json(silent=True)

    if not request_json or 'order_id' not in request_json:
        return {'error': 'order_id is required'}, 400

    order_id = request_json['order_id']

    # Firestore에서 주문 조회
    order_ref = db.collection('orders').document(order_id)
    order = order_ref.get()

    if not order.exists:
        return {'error': 'Order not found'}, 404

    order_data = order.to_dict()

    # Pub/Sub으로 이벤트 발행
    topic_path = publisher.topic_path('my-project', 'order-events')
    message = json.dumps({
        'order_id': order_id,
        'status': 'processed',
        'amount': order_data['amount']
    }).encode('utf-8')

    future = publisher.publish(topic_path, message)
    future.result()

    # 주문 상태 업데이트
    order_ref.update({'status': 'processed'})

    return {'status': 'success', 'order_id': order_id}


@functions_framework.cloud_event
def on_firestore_update(cloud_event):
    """Firestore 트리거 함수"""
    from cloudevents.http import CloudEvent

    # 이벤트 데이터 추출
    data = cloud_event.data

    old_value = data.get('oldValue', {})
    new_value = data.get('value', {})

    # 문서 경로 추출
    resource = cloud_event['source']
    doc_path = resource.split('/documents/')[-1]

    print(f"Document updated: {doc_path}")
    print(f"Old value: {old_value}")
    print(f"New value: {new_value}")

    # 비즈니스 로직 처리
    if new_value.get('fields', {}).get('status', {}).get('stringValue') == 'completed':
        # 완료 처리 로직
        pass


@functions_framework.cloud_event
def on_pubsub_message(cloud_event):
    """Pub/Sub 트리거 함수"""
    import base64

    data = cloud_event.data
    message_data = base64.b64decode(data['message']['data']).decode('utf-8')
    message = json.loads(message_data)

    print(f"Received message: {message}")

    # 메시지 처리 로직
    order_id = message.get('order_id')
    if order_id:
        # 처리 로직
        pass
```

### Cloud Functions 배포
```bash
# HTTP 함수 배포 (2nd Gen)
gcloud functions deploy process-order \
    --gen2 \
    --runtime=python312 \
    --region=asia-northeast3 \
    --source=. \
    --entry-point=process_order \
    --trigger-http \
    --allow-unauthenticated \
    --memory=512MB \
    --timeout=60s \
    --min-instances=1 \
    --max-instances=100 \
    --service-account=my-function-sa@my-project.iam.gserviceaccount.com \
    --set-env-vars="PROJECT_ID=my-project" \
    --set-secrets="API_KEY=api-key:latest"

# Pub/Sub 트리거 함수 배포
gcloud functions deploy on-pubsub-message \
    --gen2 \
    --runtime=python312 \
    --region=asia-northeast3 \
    --source=. \
    --entry-point=on_pubsub_message \
    --trigger-topic=order-events \
    --memory=256MB \
    --timeout=540s

# Firestore 트리거 함수 배포
gcloud functions deploy on-firestore-update \
    --gen2 \
    --runtime=python312 \
    --region=asia-northeast3 \
    --source=. \
    --entry-point=on_firestore_update \
    --trigger-event-filters="type=google.cloud.firestore.document.v1.updated" \
    --trigger-event-filters="database=(default)" \
    --trigger-event-filters-path-pattern="document=orders/{orderId}" \
    --memory=256MB
```

---

## 데이터베이스 서비스

### Cloud SQL
```
┌─────────────────────────────────────────────────────────────────┐
│                        Cloud SQL                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  지원 엔진:                                                      │
│  - MySQL 5.6, 5.7, 8.0                                          │
│  - PostgreSQL 11, 12, 13, 14, 15, 16                           │
│  - SQL Server 2017, 2019, 2022                                  │
│                                                                  │
│  특징:                                                           │
│  - 완전 관리형 (패치, 백업, 복제 자동화)                         │
│  - 고가용성: 리전 간 페일오버                                    │
│  - 읽기 복제본 지원                                              │
│  - Cloud SQL Auth Proxy로 안전한 연결                           │
│  - Private IP 연결 지원                                          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

#### Cloud SQL 구성 (Terraform)
```hcl
# Cloud SQL (PostgreSQL)
resource "google_sql_database_instance" "main" {
  name             = "my-postgres-instance"
  database_version = "POSTGRES_15"
  region           = "asia-northeast3"

  deletion_protection = true

  settings {
    tier = "db-custom-4-15360"  # 4 vCPU, 15GB RAM

    ip_configuration {
      ipv4_enabled    = false
      private_network = google_compute_network.main.id
      require_ssl     = true
    }

    backup_configuration {
      enabled                        = true
      start_time                     = "03:00"
      point_in_time_recovery_enabled = true
      backup_retention_settings {
        retained_backups = 7
      }
    }

    availability_type = "REGIONAL"  # 고가용성

    maintenance_window {
      day          = 7  # Sunday
      hour         = 3
      update_track = "stable"
    }

    database_flags {
      name  = "max_connections"
      value = "500"
    }

    disk_autoresize       = true
    disk_autoresize_limit = 500
    disk_size             = 100
    disk_type             = "PD_SSD"

    insights_config {
      query_insights_enabled  = true
      record_application_tags = true
      record_client_address   = true
    }
  }
}

# 읽기 복제본
resource "google_sql_database_instance" "read_replica" {
  name                 = "my-postgres-replica"
  master_instance_name = google_sql_database_instance.main.name
  region               = "asia-northeast3"
  database_version     = "POSTGRES_15"

  replica_configuration {
    failover_target = false
  }

  settings {
    tier              = "db-custom-2-7680"
    availability_type = "ZONAL"

    ip_configuration {
      ipv4_enabled    = false
      private_network = google_compute_network.main.id
    }
  }
}
```

### Firestore (NoSQL)
```python
from google.cloud import firestore
from google.cloud.firestore_v1.base_query import FieldFilter

db = firestore.Client()

# 1. 문서 생성/업데이트
def create_user(user_id: str, user_data: dict):
    doc_ref = db.collection('users').document(user_id)
    doc_ref.set({
        **user_data,
        'created_at': firestore.SERVER_TIMESTAMP,
        'updated_at': firestore.SERVER_TIMESTAMP
    })

# 2. 트랜잭션
def transfer_points(from_user_id: str, to_user_id: str, points: int):
    from_ref = db.collection('users').document(from_user_id)
    to_ref = db.collection('users').document(to_user_id)

    @firestore.transactional
    def update_in_transaction(transaction):
        from_snapshot = from_ref.get(transaction=transaction)
        to_snapshot = to_ref.get(transaction=transaction)

        from_points = from_snapshot.get('points')
        if from_points < points:
            raise ValueError('Insufficient points')

        transaction.update(from_ref, {
            'points': from_points - points,
            'updated_at': firestore.SERVER_TIMESTAMP
        })
        transaction.update(to_ref, {
            'points': to_snapshot.get('points') + points,
            'updated_at': firestore.SERVER_TIMESTAMP
        })

    transaction = db.transaction()
    update_in_transaction(transaction)

# 3. 복합 쿼리
def get_active_premium_users(min_points: int = 1000):
    users_ref = db.collection('users')

    query = (users_ref
        .where(filter=FieldFilter('status', '==', 'active'))
        .where(filter=FieldFilter('membership', '==', 'premium'))
        .where(filter=FieldFilter('points', '>=', min_points))
        .order_by('points', direction=firestore.Query.DESCENDING)
        .limit(100))

    return [doc.to_dict() for doc in query.stream()]

# 4. 실시간 리스너
def watch_user_changes(user_id: str, callback):
    doc_ref = db.collection('users').document(user_id)

    def on_snapshot(doc_snapshot, changes, read_time):
        for doc in doc_snapshot:
            callback(doc.to_dict())

    return doc_ref.on_snapshot(on_snapshot)
```

---

## 네트워킹

### VPC 네트워크 구조
```
┌─────────────────────────────────────────────────────────────────────┐
│                      GCP VPC Network                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  특징:                                                               │
│  - 글로벌 리소스 (AWS VPC는 리전별)                                  │
│  - 서브넷은 리전별                                                   │
│  - 방화벽 규칙은 네트워크 전체에 적용                                │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                    VPC Network (Global)                      │    │
│  │                                                              │    │
│  │  asia-northeast3 (Seoul)                                    │    │
│  │  ┌────────────────────────────────────────────────────────┐ │    │
│  │  │ Subnet: 10.0.0.0/24                                    │ │    │
│  │  │ ┌─────────┐  ┌─────────┐  ┌─────────┐                 │ │    │
│  │  │ │   VM    │  │   GKE   │  │ Cloud   │                 │ │    │
│  │  │ │Instance │  │  Nodes  │  │   SQL   │                 │ │    │
│  │  │ └─────────┘  └─────────┘  └─────────┘                 │ │    │
│  │  └────────────────────────────────────────────────────────┘ │    │
│  │                                                              │    │
│  │  us-central1 (Iowa)                                         │    │
│  │  ┌────────────────────────────────────────────────────────┐ │    │
│  │  │ Subnet: 10.1.0.0/24                                    │ │    │
│  │  │ ┌─────────┐  ┌─────────┐                               │ │    │
│  │  │ │   VM    │  │  Cloud  │                               │ │    │
│  │  │ │Instance │  │   Run   │                               │ │    │
│  │  │ └─────────┘  └─────────┘                               │ │    │
│  │  └────────────────────────────────────────────────────────┘ │    │
│  │                                                              │    │
│  │  ← 글로벌 VPC 내 자유로운 통신 (같은 VPC) →                  │    │
│  │                                                              │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 방화벽 규칙
```hcl
# 모든 인스턴스에 SSH 허용 (IAP 터널 사용)
resource "google_compute_firewall" "allow_iap_ssh" {
  name    = "allow-iap-ssh"
  network = google_compute_network.main.name

  allow {
    protocol = "tcp"
    ports    = ["22"]
  }

  source_ranges = ["35.235.240.0/20"]  # IAP 범위
  target_tags   = ["ssh-enabled"]
}

# 웹 서버 HTTP/HTTPS 허용
resource "google_compute_firewall" "allow_http_https" {
  name    = "allow-http-https"
  network = google_compute_network.main.name

  allow {
    protocol = "tcp"
    ports    = ["80", "443"]
  }

  source_ranges = ["0.0.0.0/0"]
  target_tags   = ["http-server"]
}

# 내부 통신 허용
resource "google_compute_firewall" "allow_internal" {
  name    = "allow-internal"
  network = google_compute_network.main.name

  allow {
    protocol = "tcp"
    ports    = ["0-65535"]
  }

  allow {
    protocol = "udp"
    ports    = ["0-65535"]
  }

  allow {
    protocol = "icmp"
  }

  source_ranges = ["10.0.0.0/8"]
}

# Health Check 허용
resource "google_compute_firewall" "allow_health_check" {
  name    = "allow-health-check"
  network = google_compute_network.main.name

  allow {
    protocol = "tcp"
  }

  source_ranges = [
    "35.191.0.0/16",    # Health Check 범위
    "130.211.0.0/22"
  ]

  target_tags = ["lb-target"]
}
```

### Cloud Load Balancing
```
┌─────────────────────────────────────────────────────────────────────┐
│                    Cloud Load Balancing Types                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Global Load Balancers:                                              │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ HTTP(S) Load Balancer                                        │    │
│  │ - Layer 7, 글로벌 Anycast IP                                 │    │
│  │ - URL 기반 라우팅, Cloud CDN 통합                            │    │
│  │ - Cloud Armor (WAF/DDoS) 통합                                │    │
│  │ - 관리형 SSL 인증서                                          │    │
│  └─────────────────────────────────────────────────────────────┘    │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ SSL Proxy / TCP Proxy Load Balancer                          │    │
│  │ - Layer 4, 글로벌                                            │    │
│  │ - SSL 오프로드, TCP 프록시                                   │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  Regional Load Balancers:                                            │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ Internal HTTP(S) Load Balancer                               │    │
│  │ - Layer 7, 리전별                                            │    │
│  │ - 내부 마이크로서비스용                                      │    │
│  └─────────────────────────────────────────────────────────────┘    │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ Internal/External TCP/UDP Load Balancer                      │    │
│  │ - Layer 4, 리전별                                            │    │
│  │ - 패스스루 (DSR 지원)                                        │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### Global HTTP(S) Load Balancer (Terraform)
```hcl
# Reserved IP
resource "google_compute_global_address" "default" {
  name = "lb-ip"
}

# SSL Certificate (관리형)
resource "google_compute_managed_ssl_certificate" "default" {
  name = "my-cert"

  managed {
    domains = ["api.example.com"]
  }
}

# Backend Service
resource "google_compute_backend_service" "api" {
  name                  = "api-backend"
  protocol              = "HTTP"
  port_name             = "http"
  timeout_sec           = 30
  health_checks         = [google_compute_health_check.api.id]
  load_balancing_scheme = "EXTERNAL_MANAGED"

  backend {
    group           = google_compute_instance_group_manager.api.instance_group
    balancing_mode  = "UTILIZATION"
    capacity_scaler = 1.0
  }

  cdn_policy {
    cache_mode = "CACHE_ALL_STATIC"
    default_ttl = 3600
    max_ttl     = 86400
  }

  log_config {
    enable      = true
    sample_rate = 1.0
  }
}

# Cloud Run Backend (Serverless NEG)
resource "google_compute_region_network_endpoint_group" "cloudrun" {
  name                  = "cloudrun-neg"
  network_endpoint_type = "SERVERLESS"
  region                = "asia-northeast3"

  cloud_run {
    service = google_cloud_run_v2_service.api.name
  }
}

resource "google_compute_backend_service" "cloudrun" {
  name                  = "cloudrun-backend"
  protocol              = "HTTPS"
  load_balancing_scheme = "EXTERNAL_MANAGED"

  backend {
    group = google_compute_region_network_endpoint_group.cloudrun.id
  }
}

# URL Map
resource "google_compute_url_map" "default" {
  name            = "lb-url-map"
  default_service = google_compute_backend_service.api.id

  host_rule {
    hosts        = ["api.example.com"]
    path_matcher = "api-paths"
  }

  path_matcher {
    name            = "api-paths"
    default_service = google_compute_backend_service.api.id

    path_rule {
      paths   = ["/v2/*"]
      service = google_compute_backend_service.cloudrun.id
    }
  }
}

# HTTPS Proxy
resource "google_compute_target_https_proxy" "default" {
  name             = "https-proxy"
  url_map          = google_compute_url_map.default.id
  ssl_certificates = [google_compute_managed_ssl_certificate.default.id]
}

# Forwarding Rule
resource "google_compute_global_forwarding_rule" "default" {
  name                  = "https-forwarding-rule"
  ip_protocol           = "TCP"
  load_balancing_scheme = "EXTERNAL_MANAGED"
  port_range            = "443"
  target                = google_compute_target_https_proxy.default.id
  ip_address            = google_compute_global_address.default.id
}

# Cloud Armor (WAF)
resource "google_compute_security_policy" "default" {
  name = "waf-policy"

  rule {
    action   = "deny(403)"
    priority = "1000"
    match {
      expr {
        expression = "evaluatePreconfiguredExpr('xss-stable')"
      }
    }
    description = "Block XSS attacks"
  }

  rule {
    action   = "deny(403)"
    priority = "1001"
    match {
      expr {
        expression = "evaluatePreconfiguredExpr('sqli-stable')"
      }
    }
    description = "Block SQL injection"
  }

  rule {
    action   = "allow"
    priority = "2147483647"
    match {
      versioned_expr = "SRC_IPS_V1"
      config {
        src_ip_ranges = ["*"]
      }
    }
    description = "Default allow rule"
  }
}
```

---

## 참고 자료

- [Google Cloud 공식 문서](https://cloud.google.com/docs)
- [Google Cloud Architecture Center](https://cloud.google.com/architecture)
- [GKE 모범 사례](https://cloud.google.com/kubernetes-engine/docs/best-practices)
- [BigQuery 문서](https://cloud.google.com/bigquery/docs)
- [Cloud Run 문서](https://cloud.google.com/run/docs)
- [Cloud Functions 문서](https://cloud.google.com/functions/docs)
