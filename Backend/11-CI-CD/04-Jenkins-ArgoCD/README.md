# Jenkins & ArgoCD

## 목차
1. [Jenkins Pipeline 개요](#1-jenkins-pipeline-개요)
2. [Declarative vs Scripted Pipeline](#2-declarative-vs-scripted-pipeline)
3. [Jenkinsfile 작성](#3-jenkinsfile-작성)
4. [ArgoCD GitOps 개념](#4-argocd-gitops-개념)
5. [ArgoCD 설정 및 사용법](#5-argocd-설정-및-사용법)
6. [Jenkins + ArgoCD 통합](#6-jenkins--argocd-통합)
---

## 1. Jenkins Pipeline 개요

### 1.1 Jenkins Pipeline이란?

**Jenkins Pipeline**은 Jenkins에서 CI/CD 파이프라인을 코드로 정의하는 기능이다. Jenkinsfile이라는 텍스트 파일에 파이프라인을 정의하여 소스 코드와 함께 버전 관리할 수 있다.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      Jenkins Pipeline 구조                               │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────────┐│
│  │                         Pipeline                                    ││
│  │  ┌─────────────────────────────────────────────────────────────────┐││
│  │  │                         Agent                                   │││
│  │  │  (실행 환경: any, docker, kubernetes, label)                    │││
│  │  └─────────────────────────────────────────────────────────────────┘││
│  │  ┌─────────────────────────────────────────────────────────────────┐││
│  │  │                         Stages                                  │││
│  │  │  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐           │││
│  │  │  │  Build  │──│  Test   │──│ Analyze │──│ Deploy  │           │││
│  │  │  └─────────┘  └─────────┘  └─────────┘  └─────────┘           │││
│  │  │       │            │            │            │                  │││
│  │  │    Steps        Steps        Steps        Steps                 │││
│  │  └─────────────────────────────────────────────────────────────────┘││
│  │  ┌─────────────────────────────────────────────────────────────────┐││
│  │  │                     Post (후처리)                                │││
│  │  │  always / success / failure / unstable / changed               │││
│  │  └─────────────────────────────────────────────────────────────────┘││
│  └─────────────────────────────────────────────────────────────────────┘│
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 1.2 Pipeline의 이점

| 이점 | 설명 |
|------|------|
| **코드로 관리 (Pipeline as Code)** | Jenkinsfile을 Git으로 버전 관리, 코드 리뷰 가능 |
| **재사용성** | Shared Library로 공통 로직 재사용 |
| **내구성** | Jenkins 재시작 후에도 파이프라인 상태 유지 |
| **일시 중지 가능** | 수동 승인 대기 등 중간 상태에서 일시 중지 |
| **확장성** | 다양한 플러그인과 통합 |

---

## 2. Declarative vs Scripted Pipeline

### 2.1 비교 개요

```
┌─────────────────────────────────────────────────────────────────────────┐
│                Declarative vs Scripted Pipeline 비교                     │
├────────────────────────────────┬────────────────────────────────────────┤
│      Declarative Pipeline      │        Scripted Pipeline               │
├────────────────────────────────┼────────────────────────────────────────┤
│ • pipeline { } 으로 시작        │ • node { } 으로 시작                    │
│ • 정형화된 구조                 │ • 자유로운 Groovy 스크립트              │
│ • 문법 검증 용이                │ • 복잡한 로직 표현 가능                 │
│ • 초보자 친화적                 │ • Groovy 지식 필요                     │
│ • 가독성 높음                   │ • 유연성 높음                          │
│ • 제한된 Groovy 사용            │ • 전체 Groovy 기능 사용                │
│ • Blue Ocean UI 지원           │ • 레거시 파이프라인                     │
│ • 현재 권장 방식                │ • 복잡한 요구사항에 사용               │
└────────────────────────────────┴────────────────────────────────────────┘
```

### 2.2 Declarative Pipeline 문법

```groovy
// Jenkinsfile (Declarative)
pipeline {
    // 실행 환경 정의
    agent {
        kubernetes {
            yaml '''
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: maven
    image: maven:3.9.6-eclipse-temurin-21
    command:
    - sleep
    args:
    - infinity
  - name: docker
    image: docker:24-dind
    securityContext:
      privileged: true
'''
        }
    }

    // 환경 변수
    environment {
        DOCKER_REGISTRY = 'ghcr.io'
        IMAGE_NAME = 'myorg/myapp'
        MAVEN_OPTS = '-Dmaven.repo.local=.m2/repository'
    }

    // 옵션 설정
    options {
        timeout(time: 30, unit: 'MINUTES')
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timestamps()
    }

    // 파라미터 정의
    parameters {
        string(name: 'BRANCH', defaultValue: 'main', description: 'Branch to build')
        choice(name: 'ENVIRONMENT', choices: ['dev', 'staging', 'prod'], description: 'Target environment')
        booleanParam(name: 'SKIP_TESTS', defaultValue: false, description: 'Skip tests')
    }

    // 트리거 설정
    triggers {
        pollSCM('H/5 * * * *')  // 5분마다 SCM 폴링
        cron('H 2 * * *')       // 매일 새벽 2시 빌드
    }

    // 파이프라인 단계
    stages {
        stage('Checkout') {
            steps {
                checkout scm
                script {
                    env.GIT_COMMIT_SHORT = sh(
                        script: 'git rev-parse --short HEAD',
                        returnStdout: true
                    ).trim()
                }
            }
        }

        stage('Build') {
            steps {
                container('maven') {
                    sh 'mvn clean package -DskipTests'
                }
            }
        }

        stage('Test') {
            when {
                expression { params.SKIP_TESTS == false }
            }
            parallel {
                stage('Unit Tests') {
                    steps {
                        container('maven') {
                            sh 'mvn test'
                        }
                    }
                    post {
                        always {
                            junit '**/target/surefire-reports/*.xml'
                        }
                    }
                }
                stage('Integration Tests') {
                    steps {
                        container('maven') {
                            sh 'mvn verify -DskipUnitTests'
                        }
                    }
                }
            }
        }

        stage('Code Quality') {
            steps {
                container('maven') {
                    withSonarQubeEnv('SonarQube') {
                        sh 'mvn sonar:sonar'
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build Image') {
            steps {
                container('docker') {
                    sh """
                        docker build -t ${DOCKER_REGISTRY}/${IMAGE_NAME}:${GIT_COMMIT_SHORT} .
                        docker tag ${DOCKER_REGISTRY}/${IMAGE_NAME}:${GIT_COMMIT_SHORT} \
                                   ${DOCKER_REGISTRY}/${IMAGE_NAME}:latest
                    """
                }
            }
        }

        stage('Push Image') {
            steps {
                container('docker') {
                    withCredentials([usernamePassword(
                        credentialsId: 'docker-registry',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )]) {
                        sh """
                            echo \$DOCKER_PASS | docker login ${DOCKER_REGISTRY} -u \$DOCKER_USER --password-stdin
                            docker push ${DOCKER_REGISTRY}/${IMAGE_NAME}:${GIT_COMMIT_SHORT}
                            docker push ${DOCKER_REGISTRY}/${IMAGE_NAME}:latest
                        """
                    }
                }
            }
        }

        stage('Deploy') {
            when {
                branch 'main'
            }
            steps {
                input message: 'Deploy to production?', ok: 'Deploy'
                echo "Deploying to ${params.ENVIRONMENT}"
            }
        }
    }

    // 후처리
    post {
        always {
            cleanWs()
        }
        success {
            slackSend(
                color: 'good',
                message: "Build Succeeded: ${env.JOB_NAME} #${env.BUILD_NUMBER}"
            )
        }
        failure {
            slackSend(
                color: 'danger',
                message: "Build Failed: ${env.JOB_NAME} #${env.BUILD_NUMBER}"
            )
        }
    }
}
```

### 2.3 Scripted Pipeline 문법

```groovy
// Jenkinsfile (Scripted)
node('kubernetes') {
    def dockerImage
    def gitCommit

    try {
        stage('Checkout') {
            checkout scm
            gitCommit = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
        }

        stage('Build') {
            // 조건부 로직
            if (env.BRANCH_NAME == 'main') {
                sh 'mvn clean package -Pprod'
            } else {
                sh 'mvn clean package -Pdev'
            }
        }

        stage('Test') {
            // 병렬 실행
            parallel(
                'Unit Tests': {
                    sh 'mvn test'
                },
                'Integration Tests': {
                    sh 'mvn verify -DskipUnitTests'
                }
            )
        }

        // 동적 스테이지 생성
        def environments = ['dev', 'staging', 'prod']
        environments.each { env ->
            stage("Deploy to ${env}") {
                if (shouldDeploy(env, env.BRANCH_NAME)) {
                    deployTo(env, gitCommit)
                } else {
                    echo "Skipping ${env} deployment"
                }
            }
        }

        currentBuild.result = 'SUCCESS'
    } catch (Exception e) {
        currentBuild.result = 'FAILURE'
        throw e
    } finally {
        // 정리 작업
        cleanWs()
        notifyBuildResult()
    }
}

// 헬퍼 함수
def shouldDeploy(environment, branch) {
    switch(environment) {
        case 'dev':
            return true
        case 'staging':
            return branch == 'develop'
        case 'prod':
            return branch == 'main'
        default:
            return false
    }
}

def deployTo(environment, version) {
    echo "Deploying version ${version} to ${environment}"
    // 배포 로직
}

def notifyBuildResult() {
    def color = currentBuild.result == 'SUCCESS' ? 'good' : 'danger'
    slackSend(color: color, message: "${env.JOB_NAME} - ${currentBuild.result}")
}
```

### 2.4 언제 어떤 것을 사용해야 하나?

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        Pipeline 선택 가이드                              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  Declarative Pipeline 선택:                                              │
│  ✓ 신규 프로젝트                                                         │
│  ✓ 표준적인 CI/CD 워크플로우                                              │
│  ✓ 팀에 Jenkins 전문가가 없는 경우                                        │
│  ✓ 코드 리뷰 및 유지보수 용이성 중요                                       │
│  ✓ Blue Ocean UI 사용                                                    │
│                                                                          │
│  Scripted Pipeline 선택:                                                 │
│  ✓ 복잡한 조건부 로직 필요                                               │
│  ✓ 동적 스테이지 생성 필요                                               │
│  ✓ 고급 Groovy 기능 활용                                                 │
│  ✓ 레거시 파이프라인 마이그레이션                                         │
│  ✓ 매우 유연한 워크플로우 필요                                            │
│                                                                          │
│  Hybrid 접근 (권장):                                                      │
│  • Declarative Pipeline 사용                                             │
│  • 복잡한 로직은 script { } 블록 활용                                     │
│  • 공통 로직은 Shared Library로 분리                                      │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Jenkinsfile 작성

### 3.1 Shared Library 활용

Shared Library를 사용하면 여러 파이프라인에서 공통 로직을 재사용할 수 있다.

```
# Shared Library 디렉토리 구조
jenkins-shared-library/
├── vars/
│   ├── buildPipeline.groovy       # 전역 변수/함수
│   ├── dockerBuild.groovy
│   └── notification.groovy
├── src/
│   └── com/
│       └── mycompany/
│           └── jenkins/
│               ├── BuildConfig.groovy
│               └── DeploymentUtils.groovy
└── resources/
    └── templates/
        └── deployment.yaml
```

```groovy
// vars/buildPipeline.groovy
def call(Map config = [:]) {
    pipeline {
        agent any

        environment {
            APP_NAME = config.appName ?: 'default-app'
            DOCKER_REGISTRY = config.registry ?: 'ghcr.io'
        }

        stages {
            stage('Build') {
                steps {
                    script {
                        dockerBuild(
                            imageName: "${DOCKER_REGISTRY}/${APP_NAME}",
                            dockerfile: config.dockerfile ?: 'Dockerfile'
                        )
                    }
                }
            }

            stage('Test') {
                when {
                    expression { config.skipTests != true }
                }
                steps {
                    sh config.testCommand ?: 'npm test'
                }
            }

            stage('Deploy') {
                when {
                    branch 'main'
                }
                steps {
                    script {
                        deployToKubernetes(
                            namespace: config.namespace,
                            deployment: config.deployment
                        )
                    }
                }
            }
        }

        post {
            always {
                notification.send(
                    channel: config.slackChannel ?: '#builds',
                    status: currentBuild.result
                )
            }
        }
    }
}

// vars/dockerBuild.groovy
def call(Map config) {
    def imageName = config.imageName
    def dockerfile = config.dockerfile ?: 'Dockerfile'
    def context = config.context ?: '.'

    sh """
        docker build -f ${dockerfile} -t ${imageName}:${env.BUILD_NUMBER} ${context}
        docker tag ${imageName}:${env.BUILD_NUMBER} ${imageName}:latest
    """

    return "${imageName}:${env.BUILD_NUMBER}"
}

// vars/notification.groovy
def send(Map config) {
    def color = config.status == 'SUCCESS' ? 'good' :
                config.status == 'FAILURE' ? 'danger' : 'warning'

    def message = """
        *Build ${config.status}*
        Job: ${env.JOB_NAME}
        Build: #${env.BUILD_NUMBER}
        Duration: ${currentBuild.durationString}
    """.stripIndent()

    slackSend(channel: config.channel, color: color, message: message)
}
```

### 3.2 Shared Library 사용

```groovy
// Jenkinsfile에서 Shared Library 사용
@Library('my-shared-library@main') _

// 단순 사용
buildPipeline(
    appName: 'my-service',
    registry: 'ghcr.io/myorg',
    testCommand: './gradlew test',
    namespace: 'production',
    slackChannel: '#deployments'
)
```

### 3.3 멀티브랜치 파이프라인

```groovy
// Jenkinsfile - 브랜치별 다른 동작
pipeline {
    agent any

    stages {
        stage('Build') {
            steps {
                sh 'mvn clean package'
            }
        }

        stage('Deploy to Dev') {
            when {
                branch 'develop'
            }
            steps {
                echo 'Deploying to Dev environment'
            }
        }

        stage('Deploy to Staging') {
            when {
                branch pattern: 'release/*', comparator: 'GLOB'
            }
            steps {
                echo 'Deploying to Staging environment'
            }
        }

        stage('Deploy to Production') {
            when {
                allOf {
                    branch 'main'
                    tag pattern: 'v\\d+\\.\\d+\\.\\d+', comparator: 'REGEXP'
                }
            }
            steps {
                input message: 'Deploy to Production?'
                echo 'Deploying to Production environment'
            }
        }
    }
}
```

### 3.4 Credentials 관리

```groovy
pipeline {
    agent any

    stages {
        stage('Deploy') {
            steps {
                // Username/Password
                withCredentials([usernamePassword(
                    credentialsId: 'docker-hub',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh 'docker login -u $DOCKER_USER -p $DOCKER_PASS'
                }

                // Secret File
                withCredentials([file(
                    credentialsId: 'kubeconfig',
                    variable: 'KUBECONFIG'
                )]) {
                    sh 'kubectl apply -f deployment.yaml'
                }

                // SSH Key
                withCredentials([sshUserPrivateKey(
                    credentialsId: 'ssh-key',
                    keyFileVariable: 'SSH_KEY',
                    usernameVariable: 'SSH_USER'
                )]) {
                    sh '''
                        ssh -i $SSH_KEY $SSH_USER@server.example.com \
                            "cd /app && git pull && ./restart.sh"
                    '''
                }

                // Secret Text
                withCredentials([string(
                    credentialsId: 'api-token',
                    variable: 'API_TOKEN'
                )]) {
                    sh 'curl -H "Authorization: Bearer $API_TOKEN" https://api.example.com'
                }
            }
        }
    }
}
```

---

## 4. ArgoCD GitOps 개념

### 4.1 GitOps란?

**GitOps**는 Git을 단일 진실의 원천(Single Source of Truth)으로 사용하여 인프라와 애플리케이션을 선언적으로 관리하는 방식이다.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          GitOps 원칙                                     │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  1. 선언적 (Declarative)                                                 │
│     • 시스템 상태를 선언적으로 정의                                        │
│     • "어떻게"가 아닌 "무엇을" 정의                                        │
│     • Kubernetes YAML, Helm Charts, Kustomize                            │
│                                                                          │
│  2. 버전 관리 (Versioned and Immutable)                                  │
│     • 모든 변경사항은 Git에 기록                                          │
│     • 변경 이력 추적 가능                                                 │
│     • 롤백 용이                                                          │
│                                                                          │
│  3. 자동 동기화 (Pulled Automatically)                                   │
│     • Git 변경사항을 자동으로 클러스터에 적용                              │
│     • Pull 기반 배포 (Push가 아님)                                        │
│                                                                          │
│  4. 지속적 조정 (Continuously Reconciled)                                │
│     • 실제 상태와 원하는 상태의 차이 감지                                  │
│     • 자동으로 원하는 상태로 수렴                                         │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 4.2 ArgoCD 아키텍처

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        ArgoCD 아키텍처                                   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌───────────────────────────────────────────────────────────────┐      │
│  │                         Git Repository                        │      │
│  │  (Application Manifests: YAML, Helm, Kustomize)               │      │
│  └───────────────────────────────────┬───────────────────────────┘      │
│                                      │                                   │
│                                      ▼ Poll / Webhook                    │
│  ┌───────────────────────────────────────────────────────────────┐      │
│  │                         ArgoCD Server                         │      │
│  │  ┌────────────────┐  ┌──────────────┐  ┌────────────────────┐ │      │
│  │  │   API Server   │  │   Repo       │  │   Application      │ │      │
│  │  │   (gRPC/REST)  │  │   Server     │  │   Controller       │ │      │
│  │  └────────────────┘  └──────────────┘  └────────────────────┘ │      │
│  │        │                    │                   │             │      │
│  │        │                    │                   ▼             │      │
│  │        │                    │          ┌────────────────┐     │      │
│  │        │                    │          │  Reconciliation│     │      │
│  │        │                    │          │     Loop       │     │      │
│  │        │                    │          └────────────────┘     │      │
│  └────────┼────────────────────┼─────────────────┼───────────────┘      │
│           │                    │                 │                       │
│           ▼                    ▼                 ▼                       │
│  ┌─────────────┐      ┌─────────────┐    ┌─────────────────────┐        │
│  │  Web UI /   │      │  Manifest   │    │  Kubernetes Cluster │        │
│  │    CLI      │      │   Cache     │    │  (Target)           │        │
│  └─────────────┘      └─────────────┘    └─────────────────────┘        │
│                                                                          │
│  핵심 컴포넌트:                                                           │
│  • API Server: 웹 UI, CLI, CI/CD 도구와의 인터페이스                       │
│  • Repo Server: Git 저장소와 통신, 매니페스트 생성                         │
│  • Application Controller: 실제 상태와 원하는 상태 비교, 동기화            │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 4.3 Push vs Pull 기반 배포

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Push vs Pull 기반 GitOps                              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  Traditional CI/CD (Push)             GitOps (Pull)                     │
│  ─────────────────────────            ──────────────────────            │
│                                                                          │
│  ┌───────┐      ┌───────┐             ┌───────┐      ┌───────┐         │
│  │  Git  │──────│  CI   │             │  Git  │      │ArgoCD │         │
│  └───────┘      └───┬───┘             └───┬───┘      └───┬───┘         │
│                     │                     │              │              │
│                     │ Push                │  Pull        │ Sync        │
│                     ▼                     │              ▼              │
│              ┌───────────┐                └──────▶ ┌───────────┐        │
│              │Kubernetes │                         │Kubernetes │        │
│              └───────────┘                         └───────────┘        │
│                                                                          │
│  • CI에 클러스터 자격증명 필요         • ArgoCD만 클러스터 접근           │
│  • 외부에서 클러스터로 Push           • 클러스터 내에서 Git으로 Pull       │
│  • 보안 위험 높음                     • 보안 강화                         │
│  • 드리프트 감지 어려움               • 지속적 상태 조정                   │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 5. ArgoCD 설정 및 사용법

### 5.1 ArgoCD 설치

```bash
# Namespace 생성 및 ArgoCD 설치
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 초기 admin 비밀번호 확인
kubectl -n argocd get secret argocd-initial-admin-secret \
    -o jsonpath="{.data.password}" | base64 -d

# CLI 로그인
argocd login localhost:8080 --username admin --password <password>

# 비밀번호 변경
argocd account update-password
```

### 5.2 Application 정의 (선언적)

```yaml
# application.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
  # 삭제 시 리소스도 함께 삭제
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default

  # 소스 정의 (Git 저장소)
  source:
    repoURL: https://github.com/myorg/my-app-manifests.git
    targetRevision: main
    path: overlays/production

    # Kustomize 옵션
    kustomize:
      namePrefix: prod-
      images:
        - myregistry/myapp:v1.2.3

    # 또는 Helm 옵션
    # helm:
    #   releaseName: my-app
    #   valueFiles:
    #     - values-prod.yaml
    #   values: |
    #     replicaCount: 3
    #     image:
    #       tag: v1.2.3

  # 대상 클러스터
  destination:
    server: https://kubernetes.default.svc
    namespace: production

  # 동기화 정책
  syncPolicy:
    automated:
      prune: true       # Git에서 삭제된 리소스도 삭제
      selfHeal: true    # 수동 변경 시 자동 복구
      allowEmpty: false # 빈 디렉토리 허용 안함

    syncOptions:
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground
      - PruneLast=true

    # 재시도 정책
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m

  # 무시할 차이
  ignoreDifferences:
    - group: apps
      kind: Deployment
      jsonPointers:
        - /spec/replicas  # HPA가 관리하는 경우
```

### 5.3 AppProject 설정

```yaml
# app-project.yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: production
  namespace: argocd
spec:
  description: Production applications

  # 소스 저장소 제한
  sourceRepos:
    - 'https://github.com/myorg/*'
    - 'https://charts.helm.sh/stable'

  # 대상 클러스터/네임스페이스 제한
  destinations:
    - namespace: 'production'
      server: https://kubernetes.default.svc
    - namespace: 'production-*'
      server: https://kubernetes.default.svc

  # 배포 가능한 리소스 제한
  clusterResourceWhitelist:
    - group: ''
      kind: Namespace
    - group: rbac.authorization.k8s.io
      kind: ClusterRole
    - group: rbac.authorization.k8s.io
      kind: ClusterRoleBinding

  namespaceResourceBlacklist:
    - group: ''
      kind: Secret  # Secret 직접 배포 금지

  # 역할 정의 (RBAC)
  roles:
    - name: developer
      description: Developer role
      policies:
        - p, proj:production:developer, applications, get, production/*, allow
        - p, proj:production:developer, applications, sync, production/*, allow
      groups:
        - my-org:developers

    - name: admin
      description: Admin role
      policies:
        - p, proj:production:admin, applications, *, production/*, allow
      groups:
        - my-org:admins

  # Sync Window (배포 시간 제한)
  syncWindows:
    - kind: allow
      schedule: '0 9-18 * * 1-5'  # 평일 09:00-18:00
      duration: 9h
      applications:
        - '*'
    - kind: deny
      schedule: '0 0 * * 0'       # 일요일 전체
      duration: 24h
      applications:
        - '*-critical'
```

### 5.4 Kustomize와 함께 사용

```
# 디렉토리 구조
my-app-manifests/
├── base/
│   ├── kustomization.yaml
│   ├── deployment.yaml
│   ├── service.yaml
│   └── configmap.yaml
└── overlays/
    ├── development/
    │   ├── kustomization.yaml
    │   └── replica-patch.yaml
    ├── staging/
    │   ├── kustomization.yaml
    │   └── replica-patch.yaml
    └── production/
        ├── kustomization.yaml
        ├── replica-patch.yaml
        └── resource-patch.yaml
```

```yaml
# base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - service.yaml
  - configmap.yaml

commonLabels:
  app: my-app

---
# overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: production

resources:
  - ../../base

namePrefix: prod-

images:
  - name: myregistry/myapp
    newTag: v1.2.3

replicas:
  - name: my-app
    count: 5

patches:
  - path: resource-patch.yaml

---
# overlays/production/resource-patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  template:
    spec:
      containers:
        - name: my-app
          resources:
            requests:
              memory: "512Mi"
              cpu: "500m"
            limits:
              memory: "1Gi"
              cpu: "1000m"
```

### 5.5 Secrets 관리

```
┌─────────────────────────────────────────────────────────────────────────┐
│                   ArgoCD에서 Secrets 관리 방법                           │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  1. Sealed Secrets                                                       │
│     ┌─────────────────────────────────────────────────────────────────┐ │
│     │  # Secret 암호화                                                │ │
│     │  kubeseal --format=yaml < secret.yaml > sealed-secret.yaml      │ │
│     │                                                                 │ │
│     │  # 암호화된 Secret을 Git에 커밋                                  │ │
│     │  git add sealed-secret.yaml && git commit -m "Add secret"       │ │
│     └─────────────────────────────────────────────────────────────────┘ │
│                                                                          │
│  2. External Secrets Operator                                            │
│     ┌─────────────────────────────────────────────────────────────────┐ │
│     │  apiVersion: external-secrets.io/v1beta1                        │ │
│     │  kind: ExternalSecret                                           │ │
│     │  spec:                                                          │ │
│     │    secretStoreRef:                                              │ │
│     │      name: aws-secrets-manager                                  │ │
│     │    target:                                                      │ │
│     │      name: my-app-secrets                                       │ │
│     │    data:                                                        │ │
│     │      - secretKey: database-password                             │ │
│     │        remoteRef:                                               │ │
│     │          key: prod/my-app/database                              │ │
│     │          property: password                                     │ │
│     └─────────────────────────────────────────────────────────────────┘ │
│                                                                          │
│  3. SOPS (Secrets OPerationS)                                            │
│     • 파일 단위 암호화/복호화                                            │
│     • ArgoCD SOPS 플러그인 사용                                          │
│                                                                          │
│  4. HashiCorp Vault                                                      │
│     • ArgoCD Vault 플러그인                                              │
│     • Vault Agent Injector와 함께 사용                                   │
│                                                                          │
│  ⚠️ 주의: 평문 Secret을 Git에 커밋하지 마세요!                            │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 5.6 ArgoCD CLI 사용법

```bash
# 애플리케이션 목록 조회
argocd app list

# 애플리케이션 생성
argocd app create my-app \
    --repo https://github.com/myorg/manifests.git \
    --path overlays/production \
    --dest-server https://kubernetes.default.svc \
    --dest-namespace production \
    --sync-policy automated \
    --auto-prune \
    --self-heal

# 애플리케이션 상태 조회
argocd app get my-app

# 동기화
argocd app sync my-app

# 강제 동기화 (리소스 교체)
argocd app sync my-app --force

# 롤백
argocd app rollback my-app 1  # 리비전 1로 롤백

# 히스토리 조회
argocd app history my-app

# 차이점 조회
argocd app diff my-app

# 애플리케이션 삭제
argocd app delete my-app --cascade  # 리소스도 함께 삭제
```

---

## 6. Jenkins + ArgoCD 통합

### 6.1 CI/CD 파이프라인 아키텍처

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Jenkins + ArgoCD 통합 아키텍처                        │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐   │
│  │ Application     │     │   Manifest      │     │   Kubernetes    │   │
│  │ Source Repo     │     │   Repo          │     │   Cluster       │   │
│  │ (코드)          │     │   (설정)         │     │                 │   │
│  └────────┬────────┘     └────────┬────────┘     └────────┬────────┘   │
│           │                       │                       │             │
│           ▼                       │                       ▲             │
│  ┌─────────────────┐              │                       │             │
│  │    Jenkins      │              │                       │             │
│  │  ┌───────────┐  │              │                       │             │
│  │  │ 1. Build  │  │              │                       │             │
│  │  │ 2. Test   │  │              │                       │             │
│  │  │ 3. Push   │──┼──────────────┘                       │             │
│  │  │    Image  │  │                                      │             │
│  │  │ 4. Update │──┼──▶ (Manifest Repo 업데이트)          │             │
│  │  │   Manifest│  │                                      │             │
│  │  └───────────┘  │                                      │             │
│  └─────────────────┘                                      │             │
│                                                           │             │
│                      ┌─────────────────┐                  │             │
│                      │     ArgoCD      │                  │             │
│                      │  ┌───────────┐  │                  │             │
│                      │  │ 5. Detect │  │                  │             │
│                      │  │    Change │◀─┼── (Git Poll)     │             │
│                      │  │ 6. Sync   │──┼──────────────────┘             │
│                      │  └───────────┘  │                                │
│                      └─────────────────┘                                │
│                                                                          │
│  분리의 이점:                                                            │
│  • CI (Jenkins): 빌드, 테스트, 이미지 푸시                               │
│  • CD (ArgoCD): 배포, 상태 관리, 롤백                                    │
│  • 관심사 분리로 각 도구의 장점 활용                                      │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 6.2 Jenkinsfile: Manifest 업데이트

```groovy
// Jenkinsfile - ArgoCD와 통합
pipeline {
    agent any

    environment {
        DOCKER_REGISTRY = 'ghcr.io'
        IMAGE_NAME = 'myorg/myapp'
        MANIFEST_REPO = 'git@github.com:myorg/app-manifests.git'
        MANIFEST_BRANCH = 'main'
    }

    stages {
        stage('Build') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('Test') {
            steps {
                sh 'mvn test'
            }
        }

        stage('Build & Push Image') {
            steps {
                script {
                    def imageTag = "${env.BUILD_NUMBER}-${env.GIT_COMMIT[0..6]}"

                    docker.withRegistry("https://${DOCKER_REGISTRY}", 'docker-credentials') {
                        def image = docker.build("${IMAGE_NAME}:${imageTag}")
                        image.push()
                        image.push('latest')
                    }

                    env.IMAGE_TAG = imageTag
                }
            }
        }

        stage('Update Manifests') {
            steps {
                withCredentials([sshUserPrivateKey(
                    credentialsId: 'git-ssh-key',
                    keyFileVariable: 'SSH_KEY'
                )]) {
                    sh """
                        # SSH 설정
                        mkdir -p ~/.ssh
                        cp \$SSH_KEY ~/.ssh/id_rsa
                        chmod 600 ~/.ssh/id_rsa
                        ssh-keyscan github.com >> ~/.ssh/known_hosts

                        # Manifest 저장소 클론
                        git clone ${MANIFEST_REPO} manifests
                        cd manifests

                        # 이미지 태그 업데이트 (Kustomize)
                        cd overlays/production
                        kustomize edit set image ${DOCKER_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}

                        # 또는 sed 사용
                        # sed -i "s|image: ${DOCKER_REGISTRY}/${IMAGE_NAME}:.*|image: ${DOCKER_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}|g" deployment.yaml

                        # 변경사항 커밋 및 푸시
                        git config user.email "jenkins@example.com"
                        git config user.name "Jenkins"
                        git add .
                        git commit -m "Update image to ${IMAGE_TAG}

                        Build: ${BUILD_NUMBER}
                        Commit: ${GIT_COMMIT}
                        "
                        git push origin ${MANIFEST_BRANCH}
                    """
                }
            }
        }

        stage('Trigger ArgoCD Sync') {
            steps {
                withCredentials([string(
                    credentialsId: 'argocd-token',
                    variable: 'ARGOCD_TOKEN'
                )]) {
                    sh '''
                        # ArgoCD CLI로 동기화 (선택적)
                        argocd app sync my-app \
                            --server argocd.example.com \
                            --auth-token $ARGOCD_TOKEN \
                            --grpc-web

                        # 동기화 완료 대기
                        argocd app wait my-app \
                            --server argocd.example.com \
                            --auth-token $ARGOCD_TOKEN \
                            --grpc-web \
                            --health
                    '''
                }
            }
        }
    }

    post {
        success {
            slackSend(
                color: 'good',
                message: "Deployment triggered: ${IMAGE_NAME}:${IMAGE_TAG}"
            )
        }
        failure {
            slackSend(
                color: 'danger',
                message: "Build failed: ${env.JOB_NAME} #${env.BUILD_NUMBER}"
            )
        }
    }
}
```

### 6.3 ArgoCD Webhook 통합

```yaml
# ArgoCD Application with Webhook
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
  annotations:
    # 자동 새로고침 간격 (초)
    argocd.argoproj.io/refresh: "30"
spec:
  source:
    repoURL: https://github.com/myorg/app-manifests.git
    targetRevision: main
    path: overlays/production
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

```groovy
// Jenkins에서 ArgoCD Webhook 호출
stage('Trigger ArgoCD') {
    steps {
        // Webhook으로 refresh 트리거
        httpRequest(
            url: 'https://argocd.example.com/api/v1/applications/my-app?refresh=hard',
            httpMode: 'GET',
            customHeaders: [[
                name: 'Authorization',
                value: "Bearer ${env.ARGOCD_TOKEN}"
            ]]
        )
    }
}
```

---

## 참고 자료

- [Jenkins Pipeline Documentation](https://www.jenkins.io/doc/book/pipeline/)
- [Jenkins Pipeline Syntax](https://www.jenkins.io/doc/book/pipeline/syntax/)
- [ArgoCD Official Documentation](https://argo-cd.readthedocs.io/en/stable/)
- [ArgoCD Best Practices](https://argo-cd.readthedocs.io/en/stable/user-guide/best_practices/)
- [GitOps Principles](https://opengitops.dev/)
- [Declarative vs Scripted Pipeline](https://www.baeldung.com/ops/jenkins-scripted-vs-declarative-pipelines)
- [GitOps for Kubernetes Implementation Guide 2025](https://atmosly.com/blog/gitops-for-kubernetes-implementation-guide-2025)
