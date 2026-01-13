# Docker 면접 질문지

> **카테고리**: DevOps / 컨테이너 기술
> **난이도**: 초급 ~ 고급
> **총 문항**: 60문항

[목록으로 돌아가기](../README.md)

---

## 📌 Docker 기본 개념

### DOCKER-001
Docker란 무엇이며, 컨테이너 기술이 등장하게 된 배경을 설명해 주세요.

### DOCKER-002
컨테이너와 가상 머신(VM)의 차이점을 아키텍처 관점에서 설명해 주세요.

### DOCKER-003
Docker 이미지와 컨테이너의 차이점을 설명해 주세요.

### DOCKER-004
Docker의 레이어(Layer) 시스템이란 무엇이며, 어떤 장점이 있나요?

### DOCKER-005
Docker가 사용하는 Linux 커널 기술인 namespace와 cgroups에 대해 설명해 주세요.

### DOCKER-006
Docker 데몬(Docker Daemon)의 역할과 Docker 클라이언트와의 통신 방식을 설명해 주세요.

### DOCKER-007
Union File System(UnionFS)이란 무엇이며, Docker에서 어떻게 활용되나요?

### DOCKER-008
Docker Hub와 프라이빗 레지스트리의 차이점과 각각의 사용 시나리오를 설명해 주세요.

### DOCKER-009
OCI(Open Container Initiative)란 무엇이며, Docker와의 관계를 설명해 주세요.

### DOCKER-010
containerd와 runc의 역할과 Docker 아키텍처에서의 위치를 설명해 주세요.

---

## 📌 Dockerfile

### DOCKER-011
Dockerfile의 주요 명령어(FROM, RUN, CMD, ENTRYPOINT, COPY, ADD)의 역할과 차이점을 설명해 주세요.

### DOCKER-012
CMD와 ENTRYPOINT의 차이점을 설명하고, 언제 어떤 것을 사용해야 하는지 예시를 들어 설명해 주세요.

### DOCKER-013
COPY와 ADD 명령어의 차이점은 무엇이며, 어떤 상황에서 각각을 사용해야 하나요?

### DOCKER-014
멀티스테이지 빌드(Multi-stage Build)란 무엇이며, 어떤 장점이 있나요?

### DOCKER-015
멀티스테이지 빌드를 사용하여 Go 또는 Java 애플리케이션의 이미지 크기를 줄이는 방법을 설명해 주세요.

### DOCKER-016
.dockerignore 파일의 역할과 사용법을 설명해 주세요.

### DOCKER-017
Dockerfile에서 ENV와 ARG의 차이점을 설명해 주세요.

### DOCKER-018
Dockerfile의 WORKDIR 명령어의 역할과 사용 시 주의점을 설명해 주세요.

### DOCKER-019
Dockerfile에서 USER 명령어를 사용하는 이유와 보안상의 이점을 설명해 주세요.

### DOCKER-020
HEALTHCHECK 명령어의 역할과 설정 옵션에 대해 설명해 주세요.

### DOCKER-021
Dockerfile 작성 시 레이어 수를 줄이고 이미지 크기를 최적화하는 방법을 설명해 주세요.

### DOCKER-022
Dockerfile의 빌드 캐시가 무효화되는 조건과 캐시를 효율적으로 활용하는 방법을 설명해 주세요.

---

## 📌 Docker 이미지 관리

### DOCKER-023
Docker 이미지를 빌드하는 과정과 주요 옵션들을 설명해 주세요.

### DOCKER-024
Docker 이미지 태깅 전략과 버전 관리 방법에 대해 설명해 주세요.

### DOCKER-025
Docker 이미지를 프라이빗 레지스트리에 푸시하고 풀하는 과정을 설명해 주세요.

### DOCKER-026
dangling 이미지란 무엇이며, 이를 정리하는 방법을 설명해 주세요.

### DOCKER-027
Docker 이미지의 히스토리를 확인하고 각 레이어의 크기를 분석하는 방법을 설명해 주세요.

### DOCKER-028
base 이미지를 선택할 때 고려해야 할 요소들을 설명해 주세요. (alpine, slim, scratch 등)

### DOCKER-029
Docker 이미지를 파일로 저장(save)하고 로드(load)하는 방법과 사용 시나리오를 설명해 주세요.

### DOCKER-030
Docker Content Trust란 무엇이며, 이미지 서명의 중요성을 설명해 주세요.

---

## 📌 Docker 네트워크

### DOCKER-031
Docker의 기본 네트워크 드라이버 종류(bridge, host, none, overlay)와 각각의 특징을 설명해 주세요.

### DOCKER-032
Docker bridge 네트워크의 동작 원리와 컨테이너 간 통신 방식을 설명해 주세요.

### DOCKER-033
Docker host 네트워크 모드의 특징과 사용 시 주의점을 설명해 주세요.

### DOCKER-034
Docker overlay 네트워크란 무엇이며, 어떤 상황에서 사용하나요?

### DOCKER-035
Docker의 내장 DNS 서비스는 어떻게 동작하며, 컨테이너 이름으로 통신하는 원리를 설명해 주세요.

### DOCKER-036
Docker 컨테이너의 포트 매핑(-p 옵션)과 포트 노출(EXPOSE)의 차이점을 설명해 주세요.

### DOCKER-037
사용자 정의 네트워크(user-defined network)를 생성하고 사용하는 이유와 장점을 설명해 주세요.

### DOCKER-038
Docker 네트워크에서 컨테이너 간 통신을 제한하는 방법을 설명해 주세요.

---

## 📌 Docker 볼륨

### DOCKER-039
Docker에서 데이터를 영속화하는 세 가지 방법(volumes, bind mounts, tmpfs)의 차이점을 설명해 주세요.

### DOCKER-040
Docker 볼륨(named volume)의 장점과 바인드 마운트 대비 선호되는 이유를 설명해 주세요.

### DOCKER-041
바인드 마운트(bind mount)의 특징과 개발 환경에서의 활용 방법을 설명해 주세요.

### DOCKER-042
tmpfs 마운트란 무엇이며, 어떤 상황에서 사용하나요?

### DOCKER-043
Docker 볼륨의 생명주기와 컨테이너 삭제 시 볼륨 처리 방법을 설명해 주세요.

### DOCKER-044
Docker 볼륨 드라이버란 무엇이며, 외부 스토리지와 연동하는 방법을 설명해 주세요.

### DOCKER-045
읽기 전용 볼륨 마운트의 사용 시나리오와 설정 방법을 설명해 주세요.

---

## 📌 Docker Compose

### DOCKER-046
Docker Compose란 무엇이며, 단일 docker run 명령어 대비 장점을 설명해 주세요.

### DOCKER-047
docker-compose.yml 파일의 주요 구성 요소(version, services, networks, volumes)를 설명해 주세요.

### DOCKER-048
Docker Compose에서 서비스 간 의존성(depends_on)을 설정하는 방법과 한계점을 설명해 주세요.

### DOCKER-049
Docker Compose에서 환경 변수를 관리하는 방법(.env 파일, environment, env_file)을 설명해 주세요.

### DOCKER-050
Docker Compose에서 서비스를 스케일링하는 방법과 주의점을 설명해 주세요.

### DOCKER-051
Docker Compose의 네트워크 구성과 서비스 간 통신 방식을 설명해 주세요.

### DOCKER-052
Docker Compose에서 여러 compose 파일을 오버라이드하여 사용하는 방법을 설명해 주세요.

### DOCKER-053
Docker Compose의 healthcheck 설정과 서비스 시작 순서 제어 방법을 설명해 주세요.

### DOCKER-054
Docker Compose에서 볼륨을 공유하는 방법과 volumes_from의 대안을 설명해 주세요.

---

## 📌 Docker 보안

### DOCKER-055
Docker 컨테이너의 보안 위험 요소와 이를 완화하는 방법을 설명해 주세요.

### DOCKER-056
Docker에서 루트리스(rootless) 모드란 무엇이며, 어떤 보안상의 이점이 있나요?

### DOCKER-057
Docker 컨테이너를 non-root 사용자로 실행해야 하는 이유를 설명해 주세요.

### DOCKER-058
Docker secrets를 사용하여 민감한 정보를 관리하는 방법을 설명해 주세요.

### DOCKER-059
Docker 이미지 취약점 스캐닝의 중요성과 사용 가능한 도구들을 설명해 주세요.

### DOCKER-060
Docker의 seccomp, AppArmor, SELinux 프로필의 역할과 설정 방법을 설명해 주세요.

### DOCKER-061
Docker 컨테이너에서 capabilities를 제한하는 방법과 이유를 설명해 주세요.

### DOCKER-062
신뢰할 수 있는 base 이미지를 선택하고 관리하는 방법을 설명해 주세요.

---

## 📌 Docker 리소스 관리

### DOCKER-063
Docker 컨테이너의 CPU 사용량을 제한하는 방법과 옵션들을 설명해 주세요.

### DOCKER-064
Docker 컨테이너의 메모리 사용량을 제한하는 방법과 OOM(Out of Memory) 처리 방식을 설명해 주세요.

### DOCKER-065
cgroups란 무엇이며, Docker에서 리소스 제한에 어떻게 활용되나요?

### DOCKER-066
Docker 컨테이너의 I/O 성능을 제한하는 방법을 설명해 주세요.

### DOCKER-067
Docker 컨테이너의 PID 제한과 fork bomb 방지 방법을 설명해 주세요.

### DOCKER-068
Docker 리소스 사용량을 실시간으로 모니터링하는 방법(docker stats)을 설명해 주세요.

---

## 📌 Docker 로깅 및 모니터링

### DOCKER-069
Docker의 로깅 드라이버 종류와 각각의 특징을 설명해 주세요.

### DOCKER-070
Docker 컨테이너 로그를 확인하고 관리하는 방법(docker logs)을 설명해 주세요.

### DOCKER-071
Docker 로그 로테이션을 설정하는 방법과 중요성을 설명해 주세요.

### DOCKER-072
Docker 컨테이너의 메트릭을 수집하고 모니터링하는 방법을 설명해 주세요.

### DOCKER-073
Docker 이벤트를 모니터링하는 방법(docker events)과 활용 사례를 설명해 주세요.

### DOCKER-074
Prometheus와 Grafana를 사용하여 Docker 컨테이너를 모니터링하는 방법을 설명해 주세요.

---

## 📌 Docker 운영 및 트러블슈팅

### DOCKER-075
Docker 컨테이너가 시작되지 않을 때 디버깅하는 방법을 설명해 주세요.

### DOCKER-076
실행 중인 Docker 컨테이너에 접속하여 디버깅하는 방법(docker exec)을 설명해 주세요.

### DOCKER-077
Docker 컨테이너의 파일 시스템 변경 사항을 확인하는 방법(docker diff)을 설명해 주세요.

### DOCKER-078
Docker 컨테이너와 호스트 간 파일을 복사하는 방법(docker cp)을 설명해 주세요.

### DOCKER-079
Docker 이미지와 컨테이너를 정리하여 디스크 공간을 확보하는 방법(docker system prune)을 설명해 주세요.

### DOCKER-080
Docker 네트워크 문제를 진단하고 해결하는 방법을 설명해 주세요.

### DOCKER-081
Docker 컨테이너의 재시작 정책(restart policy)과 각 옵션의 차이점을 설명해 주세요.

### DOCKER-082
Docker 컨테이너가 예기치 않게 종료되었을 때 원인을 분석하는 방법을 설명해 주세요.

---

## 📌 Docker 성능 최적화

### DOCKER-083
Docker 이미지 크기를 최소화하기 위한 베스트 프랙티스를 설명해 주세요.

### DOCKER-084
Docker 빌드 시간을 단축하기 위한 캐시 활용 전략을 설명해 주세요.

### DOCKER-085
Docker의 BuildKit이란 무엇이며, 기존 빌더 대비 어떤 장점이 있나요?

### DOCKER-086
Docker 컨테이너의 시작 시간을 최적화하는 방법을 설명해 주세요.

### DOCKER-087
Docker 레이어 최적화를 통해 이미지 풀/푸시 시간을 단축하는 방법을 설명해 주세요.

### DOCKER-088
distroless 이미지란 무엇이며, 보안과 성능 관점에서의 장점을 설명해 주세요.

---

## 📌 Docker와 CI/CD 연동

### DOCKER-089
CI/CD 파이프라인에서 Docker 이미지를 빌드하고 배포하는 일반적인 워크플로우를 설명해 주세요.

### DOCKER-090
GitHub Actions에서 Docker 이미지를 빌드하고 레지스트리에 푸시하는 방법을 설명해 주세요.

### DOCKER-091
Docker 이미지 태깅 전략(semantic versioning, git commit hash)과 CI/CD에서의 적용 방법을 설명해 주세요.

### DOCKER-092
Docker를 사용한 테스트 환경 구성과 테스트 컨테이너(Testcontainers) 활용 방법을 설명해 주세요.

### DOCKER-093
Docker 이미지의 보안 스캔을 CI/CD 파이프라인에 통합하는 방법을 설명해 주세요.

### DOCKER-094
Docker layer caching을 CI/CD 환경에서 활용하여 빌드 시간을 단축하는 방법을 설명해 주세요.

### DOCKER-095
Blue-Green 또는 Rolling 배포 전략에서 Docker의 역할을 설명해 주세요.

---

## 📌 심화 질문

### DOCKER-096
Docker와 Kubernetes의 관계와 각각의 역할을 설명해 주세요.

### DOCKER-097
Docker Swarm과 Kubernetes의 차이점과 각각의 사용 시나리오를 설명해 주세요.

### DOCKER-098
Docker in Docker(DinD)란 무엇이며, 사용 시 고려사항을 설명해 주세요.

### DOCKER-099
Docker의 storage driver 종류(overlay2, aufs, btrfs 등)와 선택 기준을 설명해 주세요.

### DOCKER-100
마이크로서비스 아키텍처에서 Docker를 활용할 때의 장점과 주의점을 설명해 주세요.
