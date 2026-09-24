# CS 모의 면접 질문 목록

CS 개념과 기술 스택을 공부하며 정리한 모의 면접 질문 목록입니다. 질문에 직접 답해 보면서 이해한 내용과 더 공부할 내용을 구분하는 데 활용할 수 있습니다. 개인 학습용으로 정리한 자료이므로 참고용으로 봐 주세요.

---

## 카테고리별 질문 목록

### 1. Computer Science 기초

#### 자료구조 (Data Structure)

[자료구조 질문 보기](./cs/ds.md)

- 스택, 큐, 해시, 트리, 힙, 그래프
- 정렬 알고리즘
- MST, Thread Safe
- 이진탐색, 그리디, 동적계획법

#### 컴퓨터 구조 (Computer Architecture)

[컴퓨터 구조 질문 보기](./cs/architecture.md)

- CPU 구조, 파이프라이닝
- 메모리 계층, 캐시 메모리
- 가상 메모리, TLB
- 멀티코어, 병렬 처리
- I/O 시스템, 성능 최적화

#### 네트워크 (Network)

[네트워크 질문 보기](./cs/network.md)

- HTTP/HTTPS, 쿠키/세션
- TCP/UDP, OSI 7계층
- DNS, DHCP, IP 주소
- 3-Way/4-Way Handshake
- 로드밸런서, CORS, SOP

#### 데이터베이스 (Database)

[데이터베이스 질문 보기](./cs/db.md)

- Key, RDB vs NoSQL
- 트랜잭션, ACID, 격리 레벨
- 인덱스, B-Tree/B+Tree
- JOIN, 정규화
- 락(Lock), 레플리케이션, 샤딩

#### 운영체제 (Operating System)

[운영체제 질문 보기](./cs/os.md)

- 시스템 콜, 인터럽트
- 프로세스, 스레드, PCB
- CPU 스케줄링, 컨텍스트 스위칭
- 동기화, 뮤텍스, 세마포어, Deadlock
- 가상 메모리, 페이징, TLB
- 캐시 메모리, 파일 시스템

#### 개발 상식 및 기타

[개발 상식 및 기타 질문 보기](./cs/etc.md)

- 가상화, Docker, CI/CD
- 객체지향, SOLID, 디자인 패턴
- 함수형 프로그래밍, 순수함수
- MVC 패턴, GC
- 인증/인가, OAuth, JWT
- Git, 암호화, 인코딩

---

### 2. 프로그래밍 언어

[프로그래밍 언어 이론(PLT) 질문 보기](./etc/plt.md) - 의미론, 바인딩, 평가 전략, 타입 시스템, 효과, 프로그램 검증

#### Java

[Java 질문 보기](./etc/java.md)

- JVM, GC, 메모리 구조
- Collection Framework
- 동기화, Thread, Executor
- Stream API, Optional
- 리플렉션, Annotation

#### JavaScript / TypeScript

[JavaScript / TypeScript 질문 보기](./etc/javascript.md)

- 실행 컨텍스트, 클로저, this
- Promise, async/await, Event Loop
- TypeScript 타입 시스템
- 제네릭, 유틸리티 타입

#### Python

[Python 질문 보기](./etc/python.md)

- GIL, 메모리 관리
- 데코레이터, 제너레이터
- 동시성 처리 (Threading, Multiprocessing, Asyncio)

#### Go

[Go 질문 보기](./etc/go.md)

- 고루틴, 채널
- 인터페이스, 슬라이스
- defer, panic, recover

---

### 3. 데이터베이스 & 캐시

#### Redis

[Redis 질문 보기](./database/redis.md)

- 데이터 타입, Persistence (RDB, AOF)
- Pub/Sub, 트랜잭션
- Redis Cluster, Sentinel
- 캐시 전략, Eviction 정책
- 지연, 캐시 적중률, 메모리 사용량으로 장애 진단
- ACL과 접근 제어

#### Elasticsearch

[Elasticsearch 질문 보기](./database/elasticsearch.md)

- 아키텍처, Shard, Replica
- Query DSL, Aggregation
- Mapping, Analyzer
- 인덱스 관리, ILM
- 성능 튜닝
- 검색 지연, 색인 요청 거절, 디스크 경보
- 인덱스 접근 권한

---

### 4. 메시징 & 이벤트 스트리밍

#### Kafka

[Kafka 질문 보기](./messaging/kafka.md)

- 아키텍처, Producer, Consumer, Broker
- Partition, Offset, Consumer Group
- 리플리케이션, ISR
- Exactly-Once Semantics
- Kafka Streams, Kafka Connect
- 성능 튜닝, 모니터링
- Producer, Broker, Consumer의 병목 구분
- Lag와 처리 완료, 복제 지연과 장애 진단

---

### 5. 인프라

#### Docker

[Docker 질문 보기](./infrastructure/docker.md)

- 컨테이너 vs VM, 이미지, 레이어
- Dockerfile, 멀티스테이지 빌드, 최적화
- Docker 네트워크 (bridge, host, overlay)
- Docker 볼륨, 바인드 마운트
- Docker Compose, 서비스 정의
- Docker 보안, 루트리스, 시크릿
- 리소스 관리, cgroups
- 로깅, 모니터링, 트러블슈팅
- CI/CD 연동
- CPU throttling, 종료 원인, 로그 관리, Docker 소켓 보안

#### Kubernetes

[Kubernetes 질문 보기](./infrastructure/kubernetes.md)

- 아키텍처, Control Plane, Node 컴포넌트
- Pod, Deployment, StatefulSet, DaemonSet
- Service, Ingress, 네트워킹
- PV, PVC, StorageClass, CSI
- ConfigMap, Secret
- 스케줄링, Taint/Toleration, Affinity
- RBAC, NetworkPolicy, 보안
- HPA, VPA, Cluster Autoscaler
- Helm, Operator, CRD
- 트러블슈팅, 서비스 메시
- 지연과 재시작 원인 진단, 메트릭 수집 범위
- RBAC와 NetworkPolicy

---

### 6. 기타

#### 시스템 설계 (System Design)

[시스템 설계 질문 보기](./etc/system_design.md)

- 이벤트, 메시지, EDA
- 분산 트랜잭션, SAGA, 이벤트 소싱
- CQRS
- 데이터베이스 샤딩
- CAP 이론, Consensus
- 레플리케이션, 리더십
- MSA, API 게이트웨이, 서비스 메시

#### WebSocket

[WebSocket 질문 보기](./etc/websocket.md)

- WebSocket vs HTTP
- Handshake, 메시지 프레이밍
- Ping/Pong, 재연결
- 보안, 부하 분산

---

## 통계

-  총 카테고리: 6개
-  총 질문 파일: 19개

---

## 활용 방법

1. 관심있는 카테고리의 질문 파일을 클릭합니다
2. 각 질문에 대해 스스로 답변을 작성해봅니다
3. 모르는 내용은 학습 후 다시 도전합니다
4. 실제 면접처럼 구두로 설명하는 연습을 합니다

---

[README로 돌아가기](../readme.md)
