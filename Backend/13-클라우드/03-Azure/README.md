# Microsoft Azure

## 목차
1. [개요](#개요)
2. [Azure Virtual Machines](#azure-virtual-machines)
3. [Azure Kubernetes Service (AKS)](#azure-kubernetes-service-aks)
4. [Azure Cosmos DB](#azure-cosmos-db)
5. [Azure Functions](#azure-functions)
6. [Azure DevOps](#azure-devops)
7. [Azure Active Directory](#azure-active-directory)
---

## 개요

Microsoft Azure는 전 세계에서 두 번째로 큰 클라우드 서비스 제공업체로, 약 24%의 글로벌 시장 점유율을 보유하고 있습니다. 특히 엔터프라이즈 시장에서 강력한 입지를 가지고 있으며, Microsoft 365, Dynamics 365, Power Platform과의 통합이 강점입니다.

### Azure의 핵심 강점
```
┌─────────────────────────────────────────────────────────────────┐
│                     Azure Core Strengths                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. 엔터프라이즈 통합                                            │
│     - Microsoft 365, Active Directory 네이티브 통합             │
│     - 하이브리드 클라우드 (Azure Arc, Azure Stack)               │
│     - Windows Server, SQL Server 라이선스 혜택                  │
│                                                                  │
│  2. 개발자 도구                                                  │
│     - Visual Studio / VS Code 통합                              │
│     - GitHub 통합 (GitHub Actions, Copilot)                     │
│     - Azure DevOps (완전한 CI/CD 파이프라인)                    │
│                                                                  │
│  3. 규정 준수 & 보안                                            │
│     - 100+ 규정 준수 인증                                       │
│     - Microsoft Defender for Cloud                              │
│     - Azure Sentinel (SIEM/SOAR)                                │
│                                                                  │
│  4. AI & 데이터                                                  │
│     - Azure OpenAI Service                                      │
│     - Azure Synapse Analytics                                   │
│     - Power BI 통합                                             │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Azure 리전 구조
```
┌─────────────────────────────────────────────────────────────────┐
│                    Azure Global Infrastructure                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Geography (지역)                                                │
│  └── Region (리전) - 예: Korea Central (서울)                   │
│      ├── Availability Zone 1                                   │
│      ├── Availability Zone 2                                   │
│      └── Availability Zone 3                                   │
│                                                                  │
│  Region Pair (리전 쌍)                                          │
│  └── Korea Central ↔ Korea South                               │
│      - 자동 복제 및 우선 복구                                    │
│      - 동시 업데이트 방지                                        │
│                                                                  │
│  Edge Location                                                   │
│  └── Azure CDN, Azure Front Door                                │
│      - 전 세계 185+ PoP                                         │
│                                                                  │
│  Sovereign Cloud                                                 │
│  └── Azure Government, Azure China (21Vianet)                   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 리소스 계층 구조
```
┌─────────────────────────────────────────────────────────────────┐
│                 Azure Resource Hierarchy                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Azure AD Tenant (테넌트)                                        │
│  └── Management Group (관리 그룹) - 선택적                      │
│      └── Subscription (구독)                                    │
│          ├── Resource Group A (리소스 그룹)                     │
│          │   ├── Virtual Machine                                │
│          │   ├── Storage Account                                │
│          │   └── Virtual Network                                │
│          └── Resource Group B                                   │
│              ├── App Service                                    │
│              └── SQL Database                                   │
│                                                                  │
│  중요:                                                           │
│  - 모든 리소스는 리소스 그룹에 속해야 함                         │
│  - 리소스 그룹은 리전을 가짐 (내부 리소스는 다른 리전 가능)       │
│  - RBAC은 계층별로 상속됨                                        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Azure Virtual Machines

Azure VM은 다양한 운영 체제와 구성을 지원하는 IaaS 컴퓨팅 서비스입니다.

### VM 시리즈
| 시리즈 | 용도 | 사용 사례 |
|--------|------|-----------|
| **B-series** | 버스트 가능 | 개발/테스트, 소규모 웹 서버 |
| **D-series** | 범용 | 엔터프라이즈 앱, 중간 규모 DB |
| **E-series** | 메모리 최적화 | 인메모리 DB, SAP HANA |
| **F-series** | 컴퓨팅 최적화 | 배치 처리, 게임 서버 |
| **N-series** | GPU | ML/AI, 렌더링, HPC |
| **L-series** | 스토리지 최적화 | 대용량 로컬 SSD, NoSQL |
| **M-series** | 대규모 메모리 | SAP HANA (최대 12TB RAM) |

### 가격 모델
```
┌─────────────────────────────────────────────────────────────────┐
│                    Azure VM 가격 모델                            │
├─────────────────┬───────────────┬───────────────────────────────┤
│      모델       │    할인율      │            특징               │
├─────────────────┼───────────────┼───────────────────────────────┤
│ Pay-as-you-go   │      0%       │ 초 단위 과금, 유연성 최대     │
│ Reserved (1yr)  │    최대 40%   │ 1년 약정, 취소 가능 (수수료)  │
│ Reserved (3yr)  │    최대 72%   │ 3년 약정, 최대 할인           │
│ Spot VMs        │    최대 90%   │ 선점 가능, 30초 경고 후 종료  │
│ Savings Plan    │    최대 65%   │ 컴퓨팅 사용량 기반 약정       │
│ Hybrid Benefit  │    최대 85%   │ 기존 Windows/SQL 라이선스 활용│
└─────────────────┴───────────────┴───────────────────────────────┘
```

### Azure VM 생성 (Azure CLI)
```bash
# 리소스 그룹 생성
az group create \
    --name myResourceGroup \
    --location koreacentral

# 가상 네트워크 생성
az network vnet create \
    --resource-group myResourceGroup \
    --name myVNet \
    --address-prefix 10.0.0.0/16 \
    --subnet-name mySubnet \
    --subnet-prefix 10.0.1.0/24

# NSG (Network Security Group) 생성
az network nsg create \
    --resource-group myResourceGroup \
    --name myNSG

# SSH 규칙 추가
az network nsg rule create \
    --resource-group myResourceGroup \
    --nsg-name myNSG \
    --name AllowSSH \
    --priority 1000 \
    --protocol Tcp \
    --destination-port-ranges 22 \
    --source-address-prefixes 10.0.0.0/8 \
    --access Allow

# VM 생성
az vm create \
    --resource-group myResourceGroup \
    --name myVM \
    --image Ubuntu2204 \
    --size Standard_D2s_v5 \
    --vnet-name myVNet \
    --subnet mySubnet \
    --nsg myNSG \
    --public-ip-address "" \
    --admin-username azureuser \
    --ssh-key-values ~/.ssh/id_rsa.pub \
    --os-disk-size-gb 64 \
    --zone 1 \
    --storage-sku Premium_LRS \
    --tags Environment=Production Owner=TeamA
```

### Azure VM with Terraform
```hcl
provider "azurerm" {
  features {}
}

# 리소스 그룹
resource "azurerm_resource_group" "main" {
  name     = "rg-myapp-prod"
  location = "Korea Central"

  tags = {
    Environment = "Production"
    CostCenter  = "IT"
  }
}

# 가상 네트워크
resource "azurerm_virtual_network" "main" {
  name                = "vnet-myapp"
  address_space       = ["10.0.0.0/16"]
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
}

# 서브넷
resource "azurerm_subnet" "app" {
  name                 = "snet-app"
  resource_group_name  = azurerm_resource_group.main.name
  virtual_network_name = azurerm_virtual_network.main.name
  address_prefixes     = ["10.0.1.0/24"]
}

# Network Security Group
resource "azurerm_network_security_group" "app" {
  name                = "nsg-app"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name

  security_rule {
    name                       = "AllowHTTPS"
    priority                   = 100
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_range     = "443"
    source_address_prefix      = "*"
    destination_address_prefix = "*"
  }

  security_rule {
    name                       = "AllowSSHFromBastion"
    priority                   = 110
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_range     = "22"
    source_address_prefix      = "10.0.0.0/24"
    destination_address_prefix = "*"
  }
}

# NSG 연결
resource "azurerm_subnet_network_security_group_association" "app" {
  subnet_id                 = azurerm_subnet.app.id
  network_security_group_id = azurerm_network_security_group.app.id
}

# 네트워크 인터페이스
resource "azurerm_network_interface" "app" {
  count               = 3
  name                = "nic-app-${count.index}"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name

  ip_configuration {
    name                          = "internal"
    subnet_id                     = azurerm_subnet.app.id
    private_ip_address_allocation = "Dynamic"
  }
}

# Virtual Machine Scale Set (VMSS)
resource "azurerm_linux_virtual_machine_scale_set" "app" {
  name                = "vmss-app"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  sku                 = "Standard_D2s_v5"
  instances           = 3
  admin_username      = "azureuser"
  zones               = ["1", "2", "3"]

  admin_ssh_key {
    username   = "azureuser"
    public_key = file("~/.ssh/id_rsa.pub")
  }

  source_image_reference {
    publisher = "Canonical"
    offer     = "0001-com-ubuntu-server-jammy"
    sku       = "22_04-lts-gen2"
    version   = "latest"
  }

  os_disk {
    storage_account_type = "Premium_LRS"
    caching              = "ReadWrite"
    disk_size_gb         = 64
  }

  network_interface {
    name    = "nic-vmss"
    primary = true

    ip_configuration {
      name      = "internal"
      primary   = true
      subnet_id = azurerm_subnet.app.id

      load_balancer_backend_address_pool_ids = [
        azurerm_lb_backend_address_pool.app.id
      ]
    }
  }

  # 자동 스케일링 설정
  automatic_instance_repair {
    enabled      = true
    grace_period = "PT30M"
  }

  # 부팅 진단
  boot_diagnostics {
    storage_account_uri = azurerm_storage_account.diag.primary_blob_endpoint
  }

  # 사용자 데이터 (초기화 스크립트)
  custom_data = base64encode(<<-EOF
    #!/bin/bash
    apt-get update
    apt-get install -y nginx
    systemctl enable nginx
    systemctl start nginx
  EOF
  )

  identity {
    type = "SystemAssigned"
  }

  tags = {
    Environment = "Production"
  }
}

# Autoscale 설정
resource "azurerm_monitor_autoscale_setting" "app" {
  name                = "autoscale-app"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  target_resource_id  = azurerm_linux_virtual_machine_scale_set.app.id

  profile {
    name = "default"

    capacity {
      default = 3
      minimum = 2
      maximum = 10
    }

    rule {
      metric_trigger {
        metric_name        = "Percentage CPU"
        metric_resource_id = azurerm_linux_virtual_machine_scale_set.app.id
        time_grain         = "PT1M"
        statistic          = "Average"
        time_window        = "PT5M"
        time_aggregation   = "Average"
        operator           = "GreaterThan"
        threshold          = 70
      }

      scale_action {
        direction = "Increase"
        type      = "ChangeCount"
        value     = "1"
        cooldown  = "PT5M"
      }
    }

    rule {
      metric_trigger {
        metric_name        = "Percentage CPU"
        metric_resource_id = azurerm_linux_virtual_machine_scale_set.app.id
        time_grain         = "PT1M"
        statistic          = "Average"
        time_window        = "PT5M"
        time_aggregation   = "Average"
        operator           = "LessThan"
        threshold          = 25
      }

      scale_action {
        direction = "Decrease"
        type      = "ChangeCount"
        value     = "1"
        cooldown  = "PT5M"
      }
    }
  }
}
```

---

## Azure Kubernetes Service (AKS)

AKS는 Azure에서 제공하는 완전 관리형 Kubernetes 서비스입니다.

### AKS 특징
```
┌─────────────────────────────────────────────────────────────────┐
│                       AKS Features                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  관리형 컨트롤 플레인:                                           │
│  - Control Plane 무료                                           │
│  - 자동 업그레이드/패치                                          │
│  - Azure AD 통합 RBAC                                           │
│                                                                  │
│  네트워킹:                                                       │
│  - Azure CNI (고급 네트워킹)                                    │
│  - Kubenet (기본)                                               │
│  - Azure CNI Overlay (대규모 클러스터)                          │
│                                                                  │
│  보안:                                                           │
│  - Azure AD Pod Identity → Workload Identity                    │
│  - Azure Policy for Kubernetes                                  │
│  - Microsoft Defender for Containers                            │
│                                                                  │
│  스케일링:                                                       │
│  - Cluster Autoscaler                                           │
│  - KEDA (이벤트 기반)                                           │
│  - Virtual Nodes (ACI 연동)                                     │
│                                                                  │
│  통합:                                                           │
│  - Azure Container Registry (ACR)                               │
│  - Azure Monitor Container Insights                             │
│  - Azure Key Vault                                              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### AKS 클러스터 생성 (Azure CLI)
```bash
# AKS 클러스터 생성 (기본)
az aks create \
    --resource-group myResourceGroup \
    --name myAKSCluster \
    --location koreacentral \
    --node-count 3 \
    --node-vm-size Standard_D4s_v5 \
    --zones 1 2 3 \
    --enable-managed-identity \
    --enable-azure-rbac \
    --enable-aad \
    --enable-defender \
    --network-plugin azure \
    --network-policy azure \
    --vnet-subnet-id /subscriptions/.../subnets/aks-subnet \
    --service-cidr 10.2.0.0/16 \
    --dns-service-ip 10.2.0.10 \
    --kubernetes-version 1.29.0 \
    --auto-upgrade-channel stable \
    --tier standard \
    --enable-cluster-autoscaler \
    --min-count 2 \
    --max-count 10 \
    --enable-secret-rotation \
    --attach-acr myContainerRegistry

# 시스템 노드 풀 업데이트
az aks nodepool update \
    --resource-group myResourceGroup \
    --cluster-name myAKSCluster \
    --name nodepool1 \
    --mode System \
    --node-taints CriticalAddonsOnly=true:NoSchedule

# 사용자 노드 풀 추가
az aks nodepool add \
    --resource-group myResourceGroup \
    --cluster-name myAKSCluster \
    --name userpool \
    --node-count 3 \
    --node-vm-size Standard_D8s_v5 \
    --zones 1 2 3 \
    --mode User \
    --labels workload=app \
    --enable-cluster-autoscaler \
    --min-count 2 \
    --max-count 20

# Spot 노드 풀 추가 (비용 절감)
az aks nodepool add \
    --resource-group myResourceGroup \
    --cluster-name myAKSCluster \
    --name spotpool \
    --node-count 0 \
    --node-vm-size Standard_D4s_v5 \
    --priority Spot \
    --eviction-policy Delete \
    --spot-max-price -1 \
    --enable-cluster-autoscaler \
    --min-count 0 \
    --max-count 50 \
    --labels workload=batch \
    --node-taints kubernetes.azure.com/scalesetpriority=spot:NoSchedule
```

### AKS with Terraform
```hcl
# AKS 클러스터
resource "azurerm_kubernetes_cluster" "main" {
  name                = "aks-myapp-prod"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  dns_prefix          = "myapp"
  kubernetes_version  = "1.29.0"
  sku_tier            = "Standard"

  # 관리형 ID
  identity {
    type = "SystemAssigned"
  }

  # 기본 노드 풀 (시스템)
  default_node_pool {
    name                         = "system"
    node_count                   = 3
    vm_size                      = "Standard_D4s_v5"
    zones                        = ["1", "2", "3"]
    vnet_subnet_id               = azurerm_subnet.aks.id
    only_critical_addons_enabled = true
    temporary_name_for_rotation  = "systemtemp"

    upgrade_settings {
      max_surge = "33%"
    }
  }

  # Azure CNI 네트워킹
  network_profile {
    network_plugin      = "azure"
    network_policy      = "azure"
    service_cidr        = "10.2.0.0/16"
    dns_service_ip      = "10.2.0.10"
    load_balancer_sku   = "standard"
    outbound_type       = "loadBalancer"
  }

  # Azure AD 통합
  azure_active_directory_role_based_access_control {
    managed                = true
    azure_rbac_enabled     = true
    admin_group_object_ids = [var.aks_admin_group_id]
  }

  # 자동 업그레이드
  automatic_channel_upgrade = "stable"

  # Key Vault 통합
  key_vault_secrets_provider {
    secret_rotation_enabled  = true
    secret_rotation_interval = "2m"
  }

  # 모니터링
  oms_agent {
    log_analytics_workspace_id = azurerm_log_analytics_workspace.main.id
  }

  # Microsoft Defender
  microsoft_defender {
    log_analytics_workspace_id = azurerm_log_analytics_workspace.main.id
  }

  # 유지보수 윈도우
  maintenance_window {
    allowed {
      day   = "Sunday"
      hours = [2, 3, 4]
    }
  }

  tags = {
    Environment = "Production"
  }
}

# 사용자 노드 풀
resource "azurerm_kubernetes_cluster_node_pool" "app" {
  name                  = "app"
  kubernetes_cluster_id = azurerm_kubernetes_cluster.main.id
  vm_size               = "Standard_D8s_v5"
  node_count            = 3
  zones                 = ["1", "2", "3"]
  vnet_subnet_id        = azurerm_subnet.aks.id

  enable_auto_scaling = true
  min_count           = 2
  max_count           = 20

  node_labels = {
    "workload" = "app"
  }

  upgrade_settings {
    max_surge = "33%"
  }
}

# Spot 노드 풀
resource "azurerm_kubernetes_cluster_node_pool" "spot" {
  name                  = "spot"
  kubernetes_cluster_id = azurerm_kubernetes_cluster.main.id
  vm_size               = "Standard_D4s_v5"
  node_count            = 0
  zones                 = ["1", "2", "3"]
  vnet_subnet_id        = azurerm_subnet.aks.id
  priority              = "Spot"
  eviction_policy       = "Delete"
  spot_max_price        = -1

  enable_auto_scaling = true
  min_count           = 0
  max_count           = 50

  node_labels = {
    "kubernetes.azure.com/scalesetpriority" = "spot"
  }

  node_taints = [
    "kubernetes.azure.com/scalesetpriority=spot:NoSchedule"
  ]
}

# ACR 연결
resource "azurerm_role_assignment" "acr_pull" {
  principal_id                     = azurerm_kubernetes_cluster.main.kubelet_identity[0].object_id
  role_definition_name             = "AcrPull"
  scope                            = azurerm_container_registry.main.id
  skip_service_principal_aad_check = true
}
```

### AKS Workload Identity 설정
```bash
# 1. 사용자 할당 관리형 ID 생성
az identity create \
    --resource-group myResourceGroup \
    --name myapp-identity

# 2. 서비스 계정 어노테이션이 포함된 YAML
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myapp-sa
  namespace: default
  annotations:
    azure.workload.identity/client-id: <MANAGED_IDENTITY_CLIENT_ID>
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: default
spec:
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
        azure.workload.identity/use: "true"
    spec:
      serviceAccountName: myapp-sa
      containers:
      - name: myapp
        image: myacr.azurecr.io/myapp:latest
        env:
        - name: AZURE_CLIENT_ID
          value: <MANAGED_IDENTITY_CLIENT_ID>
EOF

# 3. Federated Credential 생성
az identity federated-credential create \
    --name myapp-federated \
    --identity-name myapp-identity \
    --resource-group myResourceGroup \
    --issuer $(az aks show -g myResourceGroup -n myAKSCluster --query "oidcIssuerProfile.issuerUrl" -o tsv) \
    --subject system:serviceaccount:default:myapp-sa
```

---

## Azure Cosmos DB

Cosmos DB는 전 세계적으로 분산된 다중 모델 데이터베이스 서비스입니다.

### Cosmos DB 특징
```
┌─────────────────────────────────────────────────────────────────┐
│                     Cosmos DB Features                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  다중 API 지원:                                                  │
│  - NoSQL (네이티브)                                              │
│  - MongoDB                                                       │
│  - Cassandra                                                     │
│  - Gremlin (그래프)                                              │
│  - Table (Azure Table Storage 호환)                             │
│  - PostgreSQL (vCore)                                           │
│                                                                  │
│  글로벌 분산:                                                    │
│  - 다중 리전 쓰기                                                │
│  - 자동 장애 조치                                                │
│  - 5가지 일관성 수준                                             │
│                                                                  │
│  성능:                                                           │
│  - 한 자릿수 밀리초 응답 시간 (SLA 99.99%)                       │
│  - 자동 인덱싱                                                   │
│  - 무제한 처리량 및 스토리지                                     │
│                                                                  │
│  가격 모델:                                                      │
│  - Provisioned Throughput (예약 RU/s)                           │
│  - Autoscale (자동 조절)                                        │
│  - Serverless (요청당 과금)                                     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 일관성 수준 (Consistency Levels)
```
┌─────────────────────────────────────────────────────────────────────┐
│                    Cosmos DB Consistency Levels                      │
├─────────────────┬───────────────────────────────────────────────────┤
│    수준         │                      특징                          │
├─────────────────┼───────────────────────────────────────────────────┤
│ Strong          │ 모든 리전에서 최신 데이터 보장                     │
│ (강력)          │ 높은 레이턴시, 낮은 가용성                        │
├─────────────────┼───────────────────────────────────────────────────┤
│ Bounded         │ 지정된 버전/시간 내 일관성                        │
│ Staleness       │ 예: 최근 100 버전 또는 5초 이내                   │
├─────────────────┼───────────────────────────────────────────────────┤
│ Session         │ 세션 내 일관성 보장 (기본값)                      │
│ (세션)          │ 본인 쓰기는 즉시 읽기 가능                        │
├─────────────────┼───────────────────────────────────────────────────┤
│ Consistent      │ 읽기 순서 보장 (쓰기 순서와 동일)                 │
│ Prefix          │ 낮은 레이턴시                                     │
├─────────────────┼───────────────────────────────────────────────────┤
│ Eventual        │ 최종 일관성, 순서 보장 없음                       │
│ (최종)          │ 가장 낮은 레이턴시, 높은 가용성                   │
└─────────────────┴───────────────────────────────────────────────────┘

권장사항:
- 대부분의 경우: Session (기본값)
- 금융/결제: Strong 또는 Bounded Staleness
- 소셜 피드/로그: Eventual
```

### Cosmos DB 생성 및 사용 (Python SDK)
```python
from azure.cosmos import CosmosClient, PartitionKey, exceptions
import os

# 연결 설정
endpoint = os.environ["COSMOS_ENDPOINT"]
key = os.environ["COSMOS_KEY"]
client = CosmosClient(endpoint, key)

# 데이터베이스 생성
database_name = "myapp-db"
database = client.create_database_if_not_exists(id=database_name)

# 컨테이너 생성 (파티션 키 설정 중요!)
container_name = "orders"
container = database.create_container_if_not_exists(
    id=container_name,
    partition_key=PartitionKey(path="/userId"),
    offer_throughput=400,  # 또는 autoscale 설정
    indexing_policy={
        "indexingMode": "consistent",
        "automatic": True,
        "includedPaths": [
            {"path": "/*"}
        ],
        "excludedPaths": [
            {"path": "/description/?"},  # 인덱싱 제외
            {"path": "/_etag/?"}
        ],
        "compositeIndexes": [
            [
                {"path": "/status", "order": "ascending"},
                {"path": "/createdAt", "order": "descending"}
            ]
        ]
    }
)

# 아이템 생성
def create_order(order_data: dict):
    try:
        container.create_item(body=order_data)
    except exceptions.CosmosResourceExistsError:
        print(f"Order {order_data['id']} already exists")

# 아이템 조회 (포인트 읽기 - 가장 효율적)
def get_order(order_id: str, user_id: str):
    try:
        return container.read_item(
            item=order_id,
            partition_key=user_id
        )
    except exceptions.CosmosResourceNotFoundError:
        return None

# 쿼리 (크로스 파티션 쿼리 피하기)
def get_user_orders(user_id: str, status: str = None):
    query = "SELECT * FROM c WHERE c.userId = @userId"
    params = [{"name": "@userId", "value": user_id}]

    if status:
        query += " AND c.status = @status"
        params.append({"name": "@status", "value": status})

    query += " ORDER BY c.createdAt DESC"

    return list(container.query_items(
        query=query,
        parameters=params,
        partition_key=user_id  # 파티션 키 지정으로 효율적 쿼리
    ))

# 트랜잭션 배치 (같은 파티션 내에서만)
def process_order(user_id: str, order_id: str, items: list):
    operations = [
        ("create", ({"id": f"order-{order_id}", "userId": user_id, "status": "pending"},)),
    ]

    for item in items:
        operations.append((
            "create",
            ({"id": f"item-{item['id']}", "userId": user_id, "orderId": order_id, **item},)
        ))

    container.execute_item_batch(
        batch_operations=operations,
        partition_key=user_id
    )

# Change Feed 처리
def process_change_feed():
    from azure.cosmos import ChangeFeedState

    # 처음부터 읽기
    for item in container.query_items_change_feed(
        start_time="Beginning",
        partition_key_range_id="0"
    ):
        print(f"Changed item: {item['id']}")
        # 이벤트 처리 로직
```

### Cosmos DB with Terraform
```hcl
# Cosmos DB 계정
resource "azurerm_cosmosdb_account" "main" {
  name                = "cosmos-myapp-prod"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  offer_type          = "Standard"
  kind                = "GlobalDocumentDB"

  # 자동 장애 조치
  automatic_failover_enabled = true

  # 일관성 정책
  consistency_policy {
    consistency_level       = "Session"
    max_interval_in_seconds = 5
    max_staleness_prefix    = 100
  }

  # 지역 복제
  geo_location {
    location          = "Korea Central"
    failover_priority = 0
    zone_redundant    = true
  }

  geo_location {
    location          = "Japan East"
    failover_priority = 1
  }

  # 네트워크 설정
  is_virtual_network_filter_enabled = true

  virtual_network_rule {
    id = azurerm_subnet.app.id
  }

  # Private Endpoint 사용 시
  public_network_access_enabled = false

  # 백업 정책
  backup {
    type                = "Continuous"
    tier                = "Continuous30Days"
  }

  # 분석 스토어 (Azure Synapse Link)
  analytical_storage_enabled = true

  tags = {
    Environment = "Production"
  }
}

# SQL 데이터베이스
resource "azurerm_cosmosdb_sql_database" "main" {
  name                = "myapp-db"
  resource_group_name = azurerm_resource_group.main.name
  account_name        = azurerm_cosmosdb_account.main.name
}

# SQL 컨테이너
resource "azurerm_cosmosdb_sql_container" "orders" {
  name                  = "orders"
  resource_group_name   = azurerm_resource_group.main.name
  account_name          = azurerm_cosmosdb_account.main.name
  database_name         = azurerm_cosmosdb_sql_database.main.name
  partition_key_path    = "/userId"
  partition_key_version = 2

  # Autoscale 처리량
  autoscale_settings {
    max_throughput = 10000
  }

  # TTL 설정
  default_ttl = 86400  # 1일

  # 인덱싱 정책
  indexing_policy {
    indexing_mode = "consistent"

    included_path {
      path = "/*"
    }

    excluded_path {
      path = "/description/?"
    }

    composite_index {
      index {
        path  = "/status"
        order = "Ascending"
      }
      index {
        path  = "/createdAt"
        order = "Descending"
      }
    }
  }

  # 유니크 키
  unique_key {
    paths = ["/orderId"]
  }
}
```

---

## Azure Functions

Azure Functions는 이벤트 기반 서버리스 컴퓨팅 플랫폼입니다.

### Azure Functions 호스팅 플랜
```
┌─────────────────────────────────────────────────────────────────────┐
│                    Azure Functions Hosting Plans                     │
├─────────────────┬───────────────────────────────────────────────────┤
│      플랜       │                      특징                          │
├─────────────────┼───────────────────────────────────────────────────┤
│ Consumption     │ - 완전 서버리스, 자동 스케일링                     │
│ (소비)          │ - 실행 시간당 과금                                 │
│                 │ - Cold Start 있음                                  │
│                 │ - 월 100만 실행, 400,000 GB-s 무료                │
├─────────────────┼───────────────────────────────────────────────────┤
│ Premium        │ - 사전 준비된 인스턴스 (Cold Start 없음)           │
│ (프리미엄)      │ - VNET 통합                                       │
│                 │ - 무제한 실행 시간                                 │
│                 │ - 더 강력한 하드웨어                               │
├─────────────────┼───────────────────────────────────────────────────┤
│ Flex           │ - 2024년 신규 (2025년 GA)                          │
│ Consumption    │ - Consumption + Premium 장점 결합                  │
│                 │ - 프라이빗 네트워킹                                │
│                 │ - 빠른 스케일링                                    │
├─────────────────┼───────────────────────────────────────────────────┤
│ Dedicated      │ - App Service 플랜에서 실행                        │
│ (전용)          │ - 기존 App Service 리소스 활용                    │
│                 │ - 예측 가능한 비용                                 │
├─────────────────┼───────────────────────────────────────────────────┤
│ Container      │ - 컨테이너 기반 Functions                          │
│ Apps           │ - Kubernetes 스타일 스케일링                       │
│                 │ - KEDA 기반                                       │
└─────────────────┴───────────────────────────────────────────────────┘
```

### Azure Functions 예시 (Python v2)
```python
import azure.functions as func
import logging
import json
from azure.cosmos import CosmosClient
from azure.identity import DefaultAzureCredential
import os

app = func.FunctionApp()

# HTTP 트리거
@app.route(route="orders/{orderId}", methods=["GET"])
@app.cosmos_db_input(
    arg_name="order",
    database_name="myapp-db",
    container_name="orders",
    id="{orderId}",
    partition_key="{orderId}",
    connection="CosmosDBConnection"
)
def get_order(req: func.HttpRequest, order: func.DocumentList) -> func.HttpResponse:
    logging.info(f"Getting order: {req.route_params.get('orderId')}")

    if not order:
        return func.HttpResponse(
            json.dumps({"error": "Order not found"}),
            status_code=404,
            mimetype="application/json"
        )

    return func.HttpResponse(
        json.dumps(order[0]),
        status_code=200,
        mimetype="application/json"
    )


# HTTP 트리거 + Cosmos DB 출력
@app.route(route="orders", methods=["POST"])
@app.cosmos_db_output(
    arg_name="outputDocument",
    database_name="myapp-db",
    container_name="orders",
    connection="CosmosDBConnection"
)
def create_order(req: func.HttpRequest, outputDocument: func.Out[func.Document]) -> func.HttpResponse:
    try:
        order_data = req.get_json()

        # 유효성 검증
        if not order_data.get("userId"):
            return func.HttpResponse(
                json.dumps({"error": "userId is required"}),
                status_code=400,
                mimetype="application/json"
            )

        # 주문 ID 생성
        import uuid
        order_data["id"] = str(uuid.uuid4())
        order_data["status"] = "pending"
        order_data["createdAt"] = func.datetime.utcnow().isoformat()

        # Cosmos DB에 저장
        outputDocument.set(func.Document.from_dict(order_data))

        return func.HttpResponse(
            json.dumps(order_data),
            status_code=201,
            mimetype="application/json"
        )

    except ValueError as e:
        return func.HttpResponse(
            json.dumps({"error": str(e)}),
            status_code=400,
            mimetype="application/json"
        )


# Cosmos DB Change Feed 트리거
@app.cosmos_db_trigger(
    arg_name="documents",
    database_name="myapp-db",
    container_name="orders",
    connection="CosmosDBConnection",
    lease_container_name="leases",
    create_lease_container_if_not_exists=True
)
@app.service_bus_output(
    arg_name="message",
    queue_name="order-events",
    connection="ServiceBusConnection"
)
def on_order_change(documents: func.DocumentList, message: func.Out[str]):
    for doc in documents:
        logging.info(f"Order changed: {doc['id']}, status: {doc.get('status')}")

        # Service Bus로 이벤트 전송
        event = {
            "eventType": "OrderUpdated",
            "orderId": doc["id"],
            "userId": doc["userId"],
            "status": doc.get("status"),
            "timestamp": func.datetime.utcnow().isoformat()
        }
        message.set(json.dumps(event))


# Service Bus 큐 트리거
@app.service_bus_queue_trigger(
    arg_name="msg",
    queue_name="order-events",
    connection="ServiceBusConnection"
)
def process_order_event(msg: func.ServiceBusMessage):
    message_body = msg.get_body().decode('utf-8')
    event = json.loads(message_body)

    logging.info(f"Processing event: {event['eventType']} for order {event['orderId']}")

    # 이벤트 처리 로직
    if event["eventType"] == "OrderUpdated" and event["status"] == "completed":
        # 완료 처리
        send_notification(event["userId"], event["orderId"])


# Timer 트리거 (스케줄 작업)
@app.timer_trigger(
    schedule="0 0 * * * *",  # 매시 정각
    arg_name="timer"
)
def cleanup_expired_orders(timer: func.TimerRequest):
    if timer.past_due:
        logging.warning("Timer is past due!")

    logging.info("Running expired orders cleanup...")
    # 정리 로직


# Blob 트리거
@app.blob_trigger(
    arg_name="blob",
    path="uploads/{name}",
    connection="StorageConnection"
)
@app.blob_output(
    arg_name="outputBlob",
    path="processed/{name}",
    connection="StorageConnection"
)
def process_upload(blob: func.InputStream, outputBlob: func.Out[bytes]):
    logging.info(f"Processing blob: {blob.name}, size: {blob.length} bytes")

    # 파일 처리
    content = blob.read()
    processed_content = process_file(content)

    # 처리된 파일 저장
    outputBlob.set(processed_content)
```

### Azure Functions 배포 (Azure CLI)
```bash
# Function App 생성 (Consumption Plan)
az functionapp create \
    --resource-group myResourceGroup \
    --consumption-plan-location koreacentral \
    --runtime python \
    --runtime-version 3.11 \
    --functions-version 4 \
    --name myFunctionApp \
    --storage-account mystorageaccount \
    --os-type Linux

# Premium Plan으로 생성
az functionapp plan create \
    --resource-group myResourceGroup \
    --name myPremiumPlan \
    --location koreacentral \
    --sku EP1 \
    --is-linux

az functionapp create \
    --resource-group myResourceGroup \
    --plan myPremiumPlan \
    --runtime python \
    --runtime-version 3.11 \
    --functions-version 4 \
    --name myFunctionApp \
    --storage-account mystorageaccount

# 환경 변수 설정
az functionapp config appsettings set \
    --resource-group myResourceGroup \
    --name myFunctionApp \
    --settings "CosmosDBConnection=@Microsoft.KeyVault(SecretUri=https://myvault.vault.azure.net/secrets/cosmos-connection/)"

# 배포
func azure functionapp publish myFunctionApp
```

---

## Azure DevOps

Azure DevOps는 완전한 DevOps 도구 체인을 제공하는 서비스입니다.

### Azure DevOps 구성 요소
```
┌─────────────────────────────────────────────────────────────────┐
│                    Azure DevOps Services                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Azure Boards                                                    │
│  └── 작업 항목 추적, 스프린트 계획, 칸반 보드                    │
│                                                                  │
│  Azure Repos                                                     │
│  └── Git 저장소 호스팅, 브랜치 정책, Pull Request                │
│                                                                  │
│  Azure Pipelines                                                 │
│  └── CI/CD 파이프라인, YAML 기반, 멀티 플랫폼                   │
│                                                                  │
│  Azure Test Plans                                                │
│  └── 수동/자동 테스트 관리, 탐색적 테스트                        │
│                                                                  │
│  Azure Artifacts                                                 │
│  └── 패키지 관리 (npm, NuGet, Maven, pip)                       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Azure Pipelines YAML 예시
```yaml
# azure-pipelines.yml
trigger:
  branches:
    include:
      - main
      - develop
  paths:
    exclude:
      - docs/*
      - README.md

pr:
  branches:
    include:
      - main

pool:
  vmImage: 'ubuntu-latest'

variables:
  - group: 'production-secrets'
  - name: dockerRegistry
    value: 'myacr.azurecr.io'
  - name: imageName
    value: 'myapp'

stages:
  # 빌드 및 테스트
  - stage: Build
    displayName: 'Build and Test'
    jobs:
      - job: BuildJob
        steps:
          - task: UsePythonVersion@0
            inputs:
              versionSpec: '3.11'

          - script: |
              python -m pip install --upgrade pip
              pip install -r requirements.txt
              pip install -r requirements-dev.txt
            displayName: 'Install dependencies'

          - script: |
              python -m pytest tests/ \
                --cov=src \
                --cov-report=xml \
                --junitxml=test-results.xml
            displayName: 'Run tests'

          - task: PublishTestResults@2
            inputs:
              testResultsFormat: 'JUnit'
              testResultsFiles: 'test-results.xml'

          - task: PublishCodeCoverageResults@1
            inputs:
              codeCoverageTool: 'Cobertura'
              summaryFileLocation: 'coverage.xml'

          - task: Docker@2
            displayName: 'Build Docker image'
            inputs:
              containerRegistry: 'acr-connection'
              repository: '$(imageName)'
              command: 'build'
              Dockerfile: 'Dockerfile'
              tags: |
                $(Build.BuildId)
                latest

          - task: Docker@2
            displayName: 'Push Docker image'
            inputs:
              containerRegistry: 'acr-connection'
              repository: '$(imageName)'
              command: 'push'
              tags: |
                $(Build.BuildId)
                latest

  # 스테이징 배포
  - stage: DeployStaging
    displayName: 'Deploy to Staging'
    dependsOn: Build
    condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/develop'))
    jobs:
      - deployment: DeployToStaging
        displayName: 'Deploy to Staging AKS'
        environment: 'staging'
        strategy:
          runOnce:
            deploy:
              steps:
                - task: KubernetesManifest@0
                  displayName: 'Deploy to AKS'
                  inputs:
                    action: 'deploy'
                    kubernetesServiceConnection: 'aks-staging-connection'
                    namespace: 'default'
                    manifests: |
                      kubernetes/deployment.yaml
                      kubernetes/service.yaml
                    containers: |
                      $(dockerRegistry)/$(imageName):$(Build.BuildId)

  # 프로덕션 배포
  - stage: DeployProduction
    displayName: 'Deploy to Production'
    dependsOn: Build
    condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
    jobs:
      - deployment: DeployToProduction
        displayName: 'Deploy to Production AKS'
        environment: 'production'
        strategy:
          canary:
            increments: [10, 50]
            preDeploy:
              steps:
                - script: echo "Pre-deploy checks"
            deploy:
              steps:
                - task: KubernetesManifest@0
                  displayName: 'Deploy canary'
                  inputs:
                    action: 'deploy'
                    kubernetesServiceConnection: 'aks-prod-connection'
                    namespace: 'default'
                    strategy: 'canary'
                    percentage: $(strategy.increment)
                    manifests: |
                      kubernetes/deployment.yaml
                    containers: |
                      $(dockerRegistry)/$(imageName):$(Build.BuildId)
            postRouteTraffic:
              steps:
                - script: |
                    # 헬스 체크 및 메트릭 검증
                    ./scripts/validate-deployment.sh
                  displayName: 'Validate deployment'
            on:
              failure:
                steps:
                  - task: KubernetesManifest@0
                    displayName: 'Rollback'
                    inputs:
                      action: 'reject'
                      kubernetesServiceConnection: 'aks-prod-connection'
                      namespace: 'default'
                      strategy: 'canary'
              success:
                steps:
                  - task: KubernetesManifest@0
                    displayName: 'Promote canary'
                    inputs:
                      action: 'promote'
                      kubernetesServiceConnection: 'aks-prod-connection'
                      namespace: 'default'
                      strategy: 'canary'
```

### GitHub Actions (Azure 배포)
```yaml
# .github/workflows/deploy.yml
name: Deploy to Azure

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  AZURE_WEBAPP_NAME: myapp
  AZURE_RESOURCE_GROUP: myResourceGroup

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      - name: Install dependencies
        run: |
          pip install -r requirements.txt
          pip install -r requirements-dev.txt

      - name: Run tests
        run: pytest tests/ --cov=src

      - name: Login to Azure Container Registry
        uses: azure/docker-login@v1
        with:
          login-server: ${{ secrets.ACR_LOGIN_SERVER }}
          username: ${{ secrets.ACR_USERNAME }}
          password: ${{ secrets.ACR_PASSWORD }}

      - name: Build and push image
        run: |
          docker build -t ${{ secrets.ACR_LOGIN_SERVER }}/myapp:${{ github.sha }} .
          docker push ${{ secrets.ACR_LOGIN_SERVER }}/myapp:${{ github.sha }}

  deploy:
    needs: build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v4

      - name: Azure Login
        uses: azure/login@v1
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}

      - name: Set AKS Context
        uses: azure/aks-set-context@v3
        with:
          resource-group: ${{ env.AZURE_RESOURCE_GROUP }}
          cluster-name: myAKSCluster

      - name: Deploy to AKS
        uses: azure/k8s-deploy@v4
        with:
          namespace: default
          manifests: |
            kubernetes/deployment.yaml
            kubernetes/service.yaml
          images: |
            ${{ secrets.ACR_LOGIN_SERVER }}/myapp:${{ github.sha }}
```

---

## Azure Active Directory

Azure AD(현재 Microsoft Entra ID)는 Microsoft의 클라우드 기반 ID 및 액세스 관리 서비스입니다.

### Azure AD 핵심 개념
```
┌─────────────────────────────────────────────────────────────────┐
│                    Azure AD / Entra ID                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  테넌트 (Tenant)                                                 │
│  └── 조직을 대표하는 Azure AD 인스턴스                           │
│                                                                  │
│  ID 유형:                                                        │
│  ├── 사용자 (Users) - 사람                                      │
│  ├── 그룹 (Groups) - 사용자 집합                                │
│  ├── 서비스 주체 (Service Principal) - 앱/서비스                │
│  └── 관리형 ID (Managed Identity) - Azure 리소스                │
│                                                                  │
│  인증 방법:                                                      │
│  ├── 비밀번호 + MFA                                             │
│  ├── 인증서                                                     │
│  ├── FIDO2 보안 키                                              │
│  └── Passkey (패스키)                                           │
│                                                                  │
│  권한 부여:                                                      │
│  ├── RBAC (역할 기반 액세스 제어)                                │
│  ├── 조건부 액세스 (Conditional Access)                         │
│  └── PIM (Privileged Identity Management)                       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 관리형 ID (Managed Identity)
```
┌─────────────────────────────────────────────────────────────────────┐
│                    Managed Identity Types                            │
├──────────────────────┬──────────────────────────────────────────────┤
│  시스템 할당         │  사용자 할당                                  │
│  (System-Assigned)   │  (User-Assigned)                             │
├──────────────────────┼──────────────────────────────────────────────┤
│ 리소스와 1:1 연결    │ 독립적으로 생성                              │
│ 리소스 삭제 시 삭제  │ 수동 삭제 필요                               │
│ 공유 불가           │ 여러 리소스에서 공유                          │
│ 빠른 설정           │ 재사용 가능                                   │
└──────────────────────┴──────────────────────────────────────────────┘

사용 사례:
- Key Vault 비밀 접근
- Storage Account 접근
- Azure SQL 인증
- Azure Service Bus/Event Hub 접근
```

### Azure AD 인증 예시 (Python)
```python
from azure.identity import DefaultAzureCredential, ManagedIdentityCredential
from azure.keyvault.secrets import SecretClient
from azure.storage.blob import BlobServiceClient
import os

# 1. DefaultAzureCredential (개발 및 프로덕션 모두 지원)
# 순서: 환경변수 → 관리형 ID → Azure CLI → VS Code → Interactive
credential = DefaultAzureCredential()

# 2. Key Vault에서 비밀 가져오기
vault_url = "https://myvault.vault.azure.net/"
secret_client = SecretClient(vault_url=vault_url, credential=credential)

db_password = secret_client.get_secret("db-password").value

# 3. Storage Account 접근
storage_url = "https://mystorageaccount.blob.core.windows.net/"
blob_service = BlobServiceClient(storage_url, credential=credential)

container_client = blob_service.get_container_client("mycontainer")
blob_list = container_client.list_blobs()

# 4. SQL Database 연결 (Azure AD 인증)
import pyodbc

# 관리형 ID 토큰으로 SQL 연결
token = credential.get_token("https://database.windows.net/.default")
token_bytes = token.token.encode("utf-16-le")

connection_string = (
    "Driver={ODBC Driver 18 for SQL Server};"
    "Server=myserver.database.windows.net;"
    "Database=mydb;"
)

conn = pyodbc.connect(
    connection_string,
    attrs_before={
        1256: token_bytes  # SQL_COPT_SS_ACCESS_TOKEN
    }
)

# 5. Azure Functions에서 관리형 ID 사용
# (함수 앱에서 시스템 할당 ID 활성화 필요)
def main(req):
    credential = ManagedIdentityCredential()
    # 또는 사용자 할당 ID
    # credential = ManagedIdentityCredential(client_id="<USER_ASSIGNED_CLIENT_ID>")

    secret_client = SecretClient(
        vault_url="https://myvault.vault.azure.net/",
        credential=credential
    )
    secret = secret_client.get_secret("api-key")
    return f"Secret retrieved successfully"
```

### RBAC 설정 (Azure CLI)
```bash
# 1. 역할 할당
az role assignment create \
    --assignee <USER_OR_SP_OBJECT_ID> \
    --role "Contributor" \
    --scope /subscriptions/<SUBSCRIPTION_ID>/resourceGroups/myResourceGroup

# 2. 커스텀 역할 생성
az role definition create --role-definition '{
    "Name": "Custom VM Operator",
    "Description": "Can start and stop virtual machines",
    "Actions": [
        "Microsoft.Compute/virtualMachines/start/action",
        "Microsoft.Compute/virtualMachines/powerOff/action",
        "Microsoft.Compute/virtualMachines/restart/action",
        "Microsoft.Compute/virtualMachines/read"
    ],
    "NotActions": [],
    "AssignableScopes": [
        "/subscriptions/<SUBSCRIPTION_ID>"
    ]
}'

# 3. 서비스 주체 생성 (CI/CD용)
az ad sp create-for-rbac \
    --name "github-actions-sp" \
    --role "Contributor" \
    --scopes /subscriptions/<SUBSCRIPTION_ID>/resourceGroups/myResourceGroup \
    --sdk-auth

# 4. 관리형 ID에 역할 할당
az role assignment create \
    --assignee <MANAGED_IDENTITY_PRINCIPAL_ID> \
    --role "Key Vault Secrets User" \
    --scope /subscriptions/<SUBSCRIPTION_ID>/resourceGroups/myResourceGroup/providers/Microsoft.KeyVault/vaults/myvault
```

---

## 참고 자료

- [Azure 공식 문서](https://docs.microsoft.com/azure/)
- [Azure Architecture Center](https://docs.microsoft.com/azure/architecture/)
- [Azure Well-Architected Framework](https://docs.microsoft.com/azure/architecture/framework/)
- [AKS 모범 사례](https://docs.microsoft.com/azure/aks/best-practices)
- [Cosmos DB 문서](https://docs.microsoft.com/azure/cosmos-db/)
- [Azure Functions 문서](https://docs.microsoft.com/azure/azure-functions/)
- [Azure DevOps 문서](https://docs.microsoft.com/azure/devops/)
