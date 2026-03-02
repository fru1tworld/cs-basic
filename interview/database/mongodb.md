# MongoDB 면접 질문

## 기본 개념

### MONGO-001
MongoDB란 무엇이며, RDBMS와 어떤 차이가 있나요?

### MONGO-002
NoSQL 데이터베이스의 종류와 MongoDB가 속한 유형은 무엇인가요?

### MONGO-003
MongoDB를 사용하는 이유와 장단점은 무엇인가요?

### MONGO-004
BSON이란 무엇이며, JSON과의 차이점은 무엇인가요?

### MONGO-005
Collection과 Document의 개념을 설명해주세요.

### MONGO-006
MongoDB의 스키마리스(Schema-less) 특성은 무엇을 의미하나요?

---

## 데이터 모델링

### MONGO-007
MongoDB에서 Schema Design의 중요한 원칙은 무엇인가요?

### MONGO-008
Embedding(내장)과 Referencing(참조) 방식의 차이와 각각 언제 사용하나요?

### MONGO-009
One-to-Many 관계를 MongoDB에서 어떻게 설계하나요?

### MONGO-010
Many-to-Many 관계를 MongoDB에서 어떻게 설계하나요?

### MONGO-011
Document의 크기 제한은 얼마이며, 큰 데이터는 어떻게 처리하나요?

### MONGO-012
GridFS란 무엇이며 언제 사용하나요?

---

## 쿼리 & 인덱싱

### MONGO-013
MongoDB의 기본 CRUD 작업을 설명해주세요.

### MONGO-014
find()와 findOne()의 차이는 무엇인가요?

### MONGO-015
MongoDB의 쿼리 연산자($eq, $gt, $in, $and, $or 등)를 설명해주세요.

### MONGO-016
Projection이란 무엇이며 어떻게 사용하나요?

### MONGO-017
MongoDB의 인덱스 종류와 각각의 특징을 설명해주세요.

### MONGO-018
Compound Index(복합 인덱스)란 무엇이며 언제 사용하나요?

### MONGO-019
인덱스의 순서가 쿼리 성능에 어떤 영향을 미치나요?

### MONGO-020
Text Index와 Geospatial Index는 무엇인가요?

### MONGO-021
explain()을 사용하여 쿼리 성능을 분석하는 방법은?

### MONGO-022
Covered Query란 무엇이며 어떻게 활용하나요?

---

## Aggregation Framework

### MONGO-023
Aggregation Pipeline이란 무엇인가요?

### MONGO-024
Aggregation의 주요 Stage($match, $group, $project, $sort 등)를 설명해주세요.

### MONGO-025
$lookup을 사용한 Join 작업을 설명해주세요.

### MONGO-026
$unwind는 언제 사용하며 어떤 역할을 하나요?

### MONGO-027
$facet을 사용하여 여러 Aggregation을 동시에 실행하는 방법은?

### MONGO-028
Aggregation Pipeline과 MapReduce의 차이는 무엇인가요?

---

## Replication

### MONGO-029
MongoDB의 Replication이란 무엇인가요?

### MONGO-030
Replica Set의 구조와 역할을 설명해주세요.

### MONGO-031
Primary, Secondary, Arbiter 노드의 역할은 무엇인가요?

### MONGO-032
Read Preference란 무엇이며 종류는 무엇인가요?

### MONGO-033
Write Concern이란 무엇이며 어떻게 설정하나요?

### MONGO-034
Replica Set의 자동 Failover 과정을 설명해주세요.

### MONGO-035
Oplog란 무엇이며 어떤 역할을 하나요?

### MONGO-036
Secondary 노드에서 읽기 작업을 수행할 때 주의할 점은?

---

## Sharding

### MONGO-037
Sharding이란 무엇이며 왜 필요한가요?

### MONGO-038
MongoDB의 Sharding 아키텍처(Shard, Config Server, mongos)를 설명해주세요.

### MONGO-039
Shard Key란 무엇이며 선택 시 고려사항은?

### MONGO-040
Range-based Sharding과 Hash-based Sharding의 차이는?

### MONGO-041
Zone Sharding이란 무엇인가요?

### MONGO-042
Chunk란 무엇이며 어떻게 분할되나요?

### MONGO-043
Balancer의 역할은 무엇인가요?

### MONGO-044
Sharding 환경에서 발생할 수 있는 문제점과 해결 방법은?

---

## Transaction & Consistency

### MONGO-045
MongoDB의 Transaction 지원에 대해 설명해주세요.

### MONGO-046
Multi-Document Transaction은 언제 사용하나요?

### MONGO-047
MongoDB의 ACID 특성을 설명해주세요.

### MONGO-048
Read Isolation과 Snapshot Isolation에 대해 설명해주세요.

### MONGO-049
Atomicity는 Document 레벨과 Multi-Document 레벨에서 어떻게 다르게 동작하나요?

### MONGO-050
Eventual Consistency란 무엇이며 MongoDB에서 어떻게 관리되나요?

---

## 성능 최적화

### MONGO-051
MongoDB 쿼리 성능을 개선하는 방법은?

### MONGO-052
Working Set이란 무엇이며 메모리와의 관계는?

### MONGO-053
Connection Pool의 개념과 적절한 크기 설정 방법은?

### MONGO-054
WiredTiger Storage Engine의 특징은 무엇인가요?

### MONGO-055
Cache 크기 설정과 메모리 관리 전략은?

### MONGO-056
Profiler를 사용하여 느린 쿼리를 찾는 방법은?

### MONGO-057
Bulk Write Operation의 장점과 사용 방법은?

### MONGO-058
Read/Write 성능을 향상시키기 위한 Best Practice는?

---

## 보안 & 관리

### MONGO-059
MongoDB의 인증과 권한 관리 방법은?

### MONGO-060
Role-Based Access Control(RBAC)이란 무엇인가요?

### MONGO-061
MongoDB에서 데이터 암호화 방법은?

### MONGO-062
Backup과 Restore 전략을 설명해주세요.

### MONGO-063
mongodump와 mongorestore의 차이점과 사용 방법은?

### MONGO-064
Point-in-Time Recovery가 가능한가요?

### MONGO-065
MongoDB 모니터링 시 중요한 메트릭은 무엇인가요?

---

## 고급 주제

### MONGO-066
Change Streams란 무엇이며 어떻게 활용하나요?

### MONGO-067
Time Series Collection은 무엇이며 언제 사용하나요?

### MONGO-068
Capped Collection의 특징과 사용 사례는?

### MONGO-069
TTL Index를 사용한 자동 데이터 만료 처리 방법은?

### MONGO-070
MongoDB Atlas의 주요 기능과 장점은?

### MONGO-071
MongoDB Compass란 무엇인가요?

### MONGO-072
MongoDB와 Elasticsearch를 함께 사용하는 아키텍처는?

### MONGO-073
MongoDB와 Redis를 함께 사용하는 캐싱 전략은?

### MONGO-074
CDC(Change Data Capture)를 MongoDB에서 구현하는 방법은?

### MONGO-075
MongoDB의 버전별 주요 변경사항과 개선점은?

---

## 실전 시나리오

### MONGO-076
대량의 데이터 마이그레이션 시 고려사항은?

### MONGO-077
Hot Shard 문제를 어떻게 해결하나요?

### MONGO-078
N+1 문제가 MongoDB에서도 발생하나요? 해결 방법은?

### MONGO-079
실시간 분석을 위한 MongoDB 설계 방법은?

### MONGO-080
Multi-tenancy 아키텍처를 MongoDB로 구현하는 방법은?

---

[면접 질문 목록으로 돌아가기](../interview.md)
