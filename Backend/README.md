# Backend 개발자 로드맵

> AI(Claude)를 활용하여 작성되었습니다.
> [roadmap.sh](https://roadmap.sh/backend) 백엔드 로드맵을 기준으로 구성했습니다.

---

## 목차

### 01. 인터넷과 웹 기초
- [인터넷 작동 원리](./01-인터넷과-웹-기초/01-인터넷-작동-원리/README.md)
- [HTTP/HTTPS](./01-인터넷과-웹-기초/02-HTTP-HTTPS/README.md)
- [DNS와 도메인](./01-인터넷과-웹-기초/03-DNS와-도메인/README.md)
- [호스팅과 브라우저 동작](./01-인터넷과-웹-기초/04-호스팅과-브라우저-동작/README.md)

### 02. 운영체제
- [프로세스와 스레드](./02-운영체제/01-프로세스와-스레드/README.md)
- [메모리 관리](./02-운영체제/02-메모리-관리/README.md)
- [프로세스간 통신](./02-운영체제/03-프로세스간-통신/README.md)
- [입출력 관리](./02-운영체제/04-입출력-관리/README.md)
- [POSIX 기초](./02-운영체제/05-POSIX-기초/README.md)
- [터미널과 쉘 명령어](./02-운영체제/06-터미널과-쉘-명령어/README.md)

### 03. 프로그래밍 언어
- [언어 기초 문법](./03-프로그래밍-언어/01-언어-기초-문법/README.md)
- [자료구조와 알고리즘](./03-프로그래밍-언어/02-자료구조와-알고리즘/README.md)
- [객체지향 프로그래밍](./03-프로그래밍-언어/03-객체지향-프로그래밍/README.md)
- [함수형 프로그래밍](./03-프로그래밍-언어/04-함수형-프로그래밍/README.md)
- [동시성 프로그래밍](./03-프로그래밍-언어/05-동시성-프로그래밍/README.md)

### 04. 버전 관리
- [Git 기초](./04-버전-관리/01-Git-기초/README.md)
- [브랜치 전략](./04-버전-관리/02-브랜치-전략/README.md)
- [GitHub/GitLab](./04-버전-관리/03-GitHub-GitLab/README.md)

### 05. 데이터베이스

#### 관계형 데이터베이스
- [SQL 기초](./05-데이터베이스/01-관계형-데이터베이스/01-SQL-기초/README.md)
- [PostgreSQL](./05-데이터베이스/01-관계형-데이터베이스/02-PostgreSQL/README.md)
- [MySQL](./05-데이터베이스/01-관계형-데이터베이스/03-MySQL/README.md)
- [인덱싱과 최적화](./05-데이터베이스/01-관계형-데이터베이스/04-인덱싱과-최적화/README.md)
- [정규화](./05-데이터베이스/01-관계형-데이터베이스/05-정규화/README.md)
- [트랜잭션과 ACID](./05-데이터베이스/01-관계형-데이터베이스/06-트랜잭션과-ACID/README.md)

#### NoSQL
- [문서DB (MongoDB)](./05-데이터베이스/02-NoSQL/01-문서DB-MongoDB/README.md)
- [Key-Value (Redis)](./05-데이터베이스/02-NoSQL/02-Key-Value-Redis/README.md)
- [그래프DB (Neo4j)](./05-데이터베이스/02-NoSQL/03-그래프DB-Neo4j/README.md)
- [시계열DB](./05-데이터베이스/02-NoSQL/04-시계열DB/README.md)

#### 고급 주제
- [ORM과 데이터 모델링](./05-데이터베이스/03-ORM과-데이터-모델링/README.md)
- [데이터 복제와 샤딩](./05-데이터베이스/04-데이터-복제와-샤딩/README.md)

### 06. API 설계
- [REST API](./06-API-설계/01-REST-API/README.md)
- [GraphQL](./06-API-설계/02-GraphQL/README.md)
- [gRPC](./06-API-설계/03-gRPC/README.md)
- [WebSocket](./06-API-설계/04-WebSocket/README.md)
- [API 인증/인가](./06-API-설계/05-API-인증-인가/README.md)
- [OpenAPI/Swagger](./06-API-설계/06-OpenAPI-Swagger/README.md)

### 07. 캐싱
- [캐싱 전략](./07-캐싱/01-캐싱-전략/README.md)
- [Redis](./07-캐싱/02-Redis/README.md)
- [Memcached](./07-캐싱/03-Memcached/README.md)
- [CDN](./07-캐싱/04-CDN/README.md)

### 08. 웹 보안
- [OWASP Top 10](./08-웹-보안/01-OWASP-Top-10/README.md)
- [HTTPS/SSL/TLS](./08-웹-보안/02-HTTPS-SSL-TLS/README.md)
- [인증과 인가](./08-웹-보안/03-인증과-인가/)
  - [OAuth 2.0](./08-웹-보안/03-인증과-인가/01-OAuth-2.0/README.md)
  - [JWT](./08-웹-보안/03-인증과-인가/02-JWT/README.md)
  - [세션 기반 인증](./08-웹-보안/03-인증과-인가/03-세션-기반-인증/README.md)
- [CORS](./08-웹-보안/04-CORS/README.md)
- [해싱과 암호화](./08-웹-보안/05-해싱과-암호화/README.md)
- [SQL Injection, XSS, CSRF](./08-웹-보안/06-SQL-Injection-XSS-CSRF/README.md)

### 09. 테스팅
- [단위 테스트](./09-테스팅/01-단위-테스트/README.md)
- [통합 테스트](./09-테스팅/02-통합-테스트/README.md)
- [E2E 테스트](./09-테스팅/03-E2E-테스트/README.md)
- [TDD/BDD](./09-테스팅/04-TDD-BDD/README.md)

### 10. 컨테이너와 가상화
- [Docker](./10-컨테이너와-가상화/01-Docker/README.md)
- [Kubernetes](./10-컨테이너와-가상화/02-Kubernetes/README.md)
- [컨테이너 오케스트레이션](./10-컨테이너와-가상화/03-컨테이너-오케스트레이션/README.md)

### 11. CI/CD
- [CI 개념과 도구](./11-CI-CD/01-CI-개념과-도구/README.md)
- [CD 배포 전략](./11-CI-CD/02-CD-배포-전략/README.md)
- [GitHub Actions](./11-CI-CD/03-GitHub-Actions/README.md)
- [Jenkins/ArgoCD](./11-CI-CD/04-Jenkins-ArgoCD/README.md)

### 12. 웹서버
- [Nginx](./12-웹서버/01-Nginx/README.md)
- [Apache](./12-웹서버/02-Apache/README.md)
- [리버스 프록시](./12-웹서버/03-리버스-프록시/README.md)
- [로드 밸런싱](./12-웹서버/04-로드-밸런싱/README.md)

### 13. 클라우드
- [AWS](./13-클라우드/01-AWS/README.md)
- [GCP](./13-클라우드/02-GCP/README.md)
- [Azure](./13-클라우드/03-Azure/README.md)
- [서버리스](./13-클라우드/04-서버리스/README.md)

### 14. 메시지 브로커
- [RabbitMQ](./14-메시지-브로커/01-RabbitMQ/README.md)
- [Kafka](./14-메시지-브로커/02-Kafka/README.md)
- [이벤트 드리븐 아키텍처](./14-메시지-브로커/03-이벤트-드리븐-아키텍처/README.md)

### 15. 아키텍처 패턴
- [모놀리식 vs 마이크로서비스](./15-아키텍처-패턴/01-모놀리식-vs-마이크로서비스/README.md)
- [클린 아키텍처](./15-아키텍처-패턴/02-클린-아키텍처/README.md)
- [도메인 주도 설계 (DDD)](./15-아키텍처-패턴/03-도메인-주도-설계-DDD/README.md)
- [CQRS와 이벤트 소싱](./15-아키텍처-패턴/04-CQRS와-이벤트-소싱/README.md)
- [서비스 메시](./15-아키텍처-패턴/05-서비스-메시/README.md)

### 16. 확장성과 성능
- [수평/수직 확장](./16-확장성과-성능/01-수평-수직-확장/README.md)
- [CAP 정리](./16-확장성과-성능/02-CAP-정리/README.md)
- [성능 프로파일링](./16-확장성과-성능/03-성능-프로파일링/README.md)
- [병목 현상 해결](./16-확장성과-성능/04-병목-현상-해결/README.md)
- [서킷 브레이커 패턴](./16-확장성과-성능/05-서킷-브레이커-패턴/README.md)

### 17. 모니터링과 관찰성
- [로깅](./17-모니터링과-관찰성/01-로깅/README.md)
- [메트릭](./17-모니터링과-관찰성/02-메트릭/README.md)
- [분산 추적](./17-모니터링과-관찰성/03-분산-추적/README.md)
- [Prometheus/Grafana](./17-모니터링과-관찰성/04-Prometheus-Grafana/README.md)

### 18. 네트워크
- [OSI 7계층](./18-네트워크/01-OSI-7계층/README.md)
- [TCP/UDP](./18-네트워크/02-TCP-UDP/README.md)
- [소켓 프로그래밍](./18-네트워크/03-소켓-프로그래밍/README.md)
- [프록시와 VPN](./18-네트워크/04-프록시와-VPN/README.md)

### 19. 시스템 디자인
- [시스템 디자인 기초](./19-시스템-디자인/01-시스템-디자인-기초/README.md)
- [대규모 시스템 설계 사례](./19-시스템-디자인/02-대규모-시스템-설계-사례/README.md)
- [분산 시스템](./19-시스템-디자인/03-분산-시스템/README.md)

---

## 학습 가이드

### 입문자 추천 순서
1. 인터넷과 웹 기초 → 2. 프로그래밍 언어 → 3. 버전 관리 → 4. 데이터베이스 (SQL 기초) → 5. API 설계 (REST)

### 중급자 추천 순서
1. 테스팅 → 2. 웹 보안 → 3. Docker → 4. CI/CD → 5. 캐싱

### 고급자 추천 순서
1. Kubernetes → 2. 아키텍처 패턴 → 3. 시스템 디자인 → 4. 분산 시스템 → 5. 모니터링

---

## 참고 자료
- [roadmap.sh - Backend Developer](https://roadmap.sh/backend)
- [The Twelve-Factor App](https://12factor.net/ko/)
- [System Design Primer](https://github.com/donnemartin/system-design-primer)
