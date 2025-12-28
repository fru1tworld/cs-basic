# Neo4j 그래프 데이터베이스

## 목차
1. [그래프 데이터베이스 개요](#그래프-데이터베이스-개요)
2. [Neo4j 아키텍처](#neo4j-아키텍처)
3. [그래프 모델](#그래프-모델)
4. [Cypher 쿼리 언어](#cypher-쿼리-언어)
5. [인덱싱과 성능](#인덱싱과-성능)
6. [사용 사례](#사용-사례)
---

## 그래프 데이터베이스 개요

### 그래프 DB가 필요한 이유

관계형 데이터베이스에서 복잡한 관계 탐색은 비용이 높습니다:

```sql
-- RDBMS에서 친구의 친구 찾기 (3단계)
SELECT DISTINCT f3.user_id
FROM friendships f1
JOIN friendships f2 ON f1.friend_id = f2.user_id
JOIN friendships f3 ON f2.friend_id = f3.user_id
WHERE f1.user_id = 1
AND f3.user_id != 1;

-- 조인이 늘어날수록 성능 급격히 저하
-- 6단계 관계: 수십 초 ~ 수 분
```

```cypher
// Neo4j에서 친구의 친구의 친구 찾기
MATCH (u:User {id: 1})-[:FRIEND*1..3]-(friend)
RETURN DISTINCT friend

// 관계 깊이가 늘어나도 성능 저하 적음
// Index-Free Adjacency: 관계를 직접 포인터로 저장
```

### RDBMS vs Graph DB

| 특성 | RDBMS | Graph DB |
|------|-------|----------|
| 데이터 모델 | 테이블, 행, 열 | 노드, 관계, 속성 |
| 관계 표현 | Foreign Key, JOIN | 직접 연결 |
| 조인 성능 | O(n log n) ~ O(n²) | O(k) - 관계 수에 비례 |
| 스키마 | 고정 스키마 | 유연한 스키마 |
| 적합한 쿼리 | 집계, 범위 검색 | 경로 탐색, 패턴 매칭 |

---

## Neo4j 아키텍처

### 저장 구조

```
┌─────────────────────────────────────────────────────────────────┐
│                      Neo4j Storage                               │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                    Node Store                            │    │
│  │  ┌──────────────────────────────────────────────────┐   │    │
│  │  │ Node Record:                                      │   │    │
│  │  │ [in_use][first_rel_id][first_prop_id][labels]    │   │    │
│  │  └──────────────────────────────────────────────────┘   │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                 Relationship Store                       │    │
│  │  ┌──────────────────────────────────────────────────┐   │    │
│  │  │ Relationship Record:                              │   │    │
│  │  │ [in_use][start_node][end_node][type]             │   │    │
│  │  │ [start_prev][start_next][end_prev][end_next]     │   │    │
│  │  └──────────────────────────────────────────────────┘   │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                  Property Store                          │    │
│  │  ┌──────────────────────────────────────────────────┐   │    │
│  │  │ Property Record:                                  │   │    │
│  │  │ [type][key_id][value][next_prop_id]              │   │    │
│  │  └──────────────────────────────────────────────────┘   │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  Index-Free Adjacency:                                          │
│  - 노드가 관계를 직접 포인터로 참조                             │
│  - 탐색 시 인덱스 조회 불필요                                    │
│  - O(1) 관계 탐색                                               │
└─────────────────────────────────────────────────────────────────┘
```

### 메모리 구조

```
┌─────────────────────────────────────────────────────────────────┐
│                    Neo4j Memory                                  │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                    JVM Heap                              │    │
│  │  - Query Execution                                       │    │
│  │  - Transaction State                                     │    │
│  │  - Object Allocation                                     │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                  Page Cache (Off-Heap)                   │    │
│  │  - Node Store Pages                                      │    │
│  │  - Relationship Store Pages                              │    │
│  │  - Property Store Pages                                  │    │
│  │  - Index Pages                                           │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  설정 예시 (32GB RAM):                                          │
│  dbms.memory.heap.initial_size=8G                               │
│  dbms.memory.heap.max_size=8G                                   │
│  dbms.memory.pagecache.size=16G                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 그래프 모델

### 기본 구성요소

```
┌─────────────────────────────────────────────────────────────────┐
│                    Property Graph Model                          │
│                                                                  │
│     ┌─────────────────┐                ┌─────────────────┐      │
│     │   Node (정점)    │                │   Node (정점)    │      │
│     │ ─────────────── │                │ ─────────────── │      │
│     │ Label: Person   │                │ Label: Company  │      │
│     │                 │                │                 │      │
│     │ Properties:     │                │ Properties:     │      │
│     │  - name: "Kim"  │                │  - name: "ABC"  │      │
│     │  - age: 30      │   WORKS_AT     │  - industry:    │      │
│     │                 │───────────────►│    "Tech"       │      │
│     │                 │  since: 2020   │                 │      │
│     └─────────────────┘                └─────────────────┘      │
│                                                                  │
│     Relationship (간선):                                         │
│     - 시작 노드 → 끝 노드 (방향성)                               │
│     - 타입: WORKS_AT, FRIEND, PURCHASED 등                       │
│     - 속성: since, weight 등                                     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 데이터 모델링 예시

```cypher
// 소셜 네트워크 모델
// 노드 생성
CREATE (alice:User {name: 'Alice', age: 28, email: 'alice@example.com'})
CREATE (bob:User {name: 'Bob', age: 32, email: 'bob@example.com'})
CREATE (tech:Interest {name: 'Technology'})
CREATE (music:Interest {name: 'Music'})
CREATE (post1:Post {content: 'Hello World', created_at: datetime()})

// 관계 생성
CREATE (alice)-[:FRIEND {since: date('2020-01-15')}]->(bob)
CREATE (alice)-[:INTERESTED_IN {level: 'high'}]->(tech)
CREATE (bob)-[:INTERESTED_IN {level: 'medium'}]->(music)
CREATE (alice)-[:POSTED]->(post1)
CREATE (bob)-[:LIKED {at: datetime()}]->(post1)

// 결과 그래프:
//
//   [Alice] ──FRIEND──► [Bob]
//      │                  │
//  INTERESTED_IN      INTERESTED_IN
//      │                  │
//      ▼                  ▼
//   [Tech]            [Music]
//      │
//   POSTED
//      │
//      ▼
//   [Post1] ◄──LIKED── [Bob]
```

---

## Cypher 쿼리 언어

### 기본 문법

```cypher
// 노드 매칭
MATCH (n:Person)
WHERE n.age > 25
RETURN n.name, n.age

// 관계 매칭
MATCH (a:Person)-[r:FRIEND]->(b:Person)
RETURN a.name, b.name, r.since

// 패턴 매칭 (ASCII Art 스타일)
// (a)-->(b)   방향성 관계
// (a)--(b)    무방향 관계
// (a)-[:TYPE]->(b)   타입 지정
// (a)-[*1..3]->(b)   1~3단계 관계
```

### CRUD 연산

```cypher
// CREATE - 생성
CREATE (n:Person {name: 'John', age: 25})
RETURN n

// MERGE - 있으면 매칭, 없으면 생성 (Upsert)
MERGE (n:Person {email: 'john@example.com'})
ON CREATE SET n.created_at = datetime()
ON MATCH SET n.updated_at = datetime()
RETURN n

// SET - 속성 수정
MATCH (n:Person {name: 'John'})
SET n.age = 26, n.city = 'Seoul'
RETURN n

// DELETE - 삭제
MATCH (n:Person {name: 'John'})
DETACH DELETE n  // 관계도 함께 삭제

// 관계 생성
MATCH (a:Person {name: 'Alice'}), (b:Person {name: 'Bob'})
CREATE (a)-[:FRIEND {since: date()}]->(b)
```

### 고급 쿼리

```cypher
// 최단 경로 찾기
MATCH path = shortestPath(
    (a:Person {name: 'Alice'})-[*..6]-(b:Person {name: 'Charlie'})
)
RETURN path, length(path)

// 모든 최단 경로
MATCH path = allShortestPaths(
    (a:Person {name: 'Alice'})-[*..6]-(b:Person {name: 'Charlie'})
)
RETURN path

// 가변 길이 관계
MATCH (a:Person {name: 'Alice'})-[:FRIEND*1..3]-(friend)
RETURN DISTINCT friend.name

// 경로 탐색 + 조건
MATCH path = (a:Person)-[:FRIEND*1..4]->(b:Person)
WHERE a.name = 'Alice'
AND ALL(r IN relationships(path) WHERE r.since > date('2020-01-01'))
RETURN path

// 집계
MATCH (p:Person)-[:POSTED]->(post:Post)
RETURN p.name, COUNT(post) as post_count
ORDER BY post_count DESC
LIMIT 10

// 공통 친구 찾기
MATCH (a:Person {name: 'Alice'})-[:FRIEND]-(mutual)-[:FRIEND]-(b:Person {name: 'Bob'})
RETURN mutual.name

// 추천: 친구의 친구 (내 친구가 아닌)
MATCH (me:Person {name: 'Alice'})-[:FRIEND]-()-[:FRIEND]-(suggested)
WHERE NOT (me)-[:FRIEND]-(suggested) AND me <> suggested
RETURN DISTINCT suggested.name, COUNT(*) as common_friends
ORDER BY common_friends DESC
```

### 서브쿼리와 UNWIND

```cypher
// UNWIND - 리스트를 행으로 펼치기
WITH ['Alice', 'Bob', 'Charlie'] as names
UNWIND names as name
MATCH (p:Person {name: name})
RETURN p

// 서브쿼리 (CALL)
MATCH (p:Person)
CALL {
    WITH p
    MATCH (p)-[:POSTED]->(post)
    RETURN COUNT(post) as postCount
}
RETURN p.name, postCount

// FOREACH - 반복 처리
MATCH path = (a)-[*]->(b)
FOREACH (node IN nodes(path) | SET node.visited = true)

// CASE 표현식
MATCH (p:Person)
RETURN p.name,
    CASE
        WHEN p.age < 20 THEN 'Teen'
        WHEN p.age < 30 THEN 'Twenties'
        WHEN p.age < 40 THEN 'Thirties'
        ELSE 'Older'
    END as age_group
```

---

## 인덱싱과 성능

### 인덱스 유형

```cypher
// 단일 속성 인덱스
CREATE INDEX person_name FOR (p:Person) ON (p.name)

// 복합 인덱스
CREATE INDEX person_name_age FOR (p:Person) ON (p.name, p.age)

// 유니크 제약조건 (자동으로 인덱스 생성)
CREATE CONSTRAINT person_email_unique FOR (p:Person) REQUIRE p.email IS UNIQUE

// 존재 제약조건
CREATE CONSTRAINT person_name_exists FOR (p:Person) REQUIRE p.name IS NOT NULL

// 전문 검색 인덱스
CREATE FULLTEXT INDEX post_content FOR (p:Post) ON EACH [p.title, p.content]

// 전문 검색 쿼리
CALL db.index.fulltext.queryNodes('post_content', 'graph database')
YIELD node, score
RETURN node.title, score
ORDER BY score DESC

// 인덱스 목록 확인
SHOW INDEXES

// 인덱스 삭제
DROP INDEX person_name
```

### 쿼리 최적화

```cypher
// 실행 계획 확인
EXPLAIN MATCH (p:Person {name: 'Alice'})-[:FRIEND]->(f)
RETURN f.name

// 상세 프로파일
PROFILE MATCH (p:Person {name: 'Alice'})-[:FRIEND]->(f)
RETURN f.name

// 주요 연산자:
// - NodeByLabelScan: 레이블로 모든 노드 스캔 (느림)
// - NodeIndexSeek: 인덱스 사용 (빠름)
// - Expand: 관계 탐색
// - Filter: 조건 필터링

// 최적화 팁

// 1. 시작점 명확히 (인덱스 사용)
// Bad:
MATCH (a)-[:FRIEND]-(b)
WHERE a.name = 'Alice'
RETURN b

// Good:
MATCH (a:Person {name: 'Alice'})-[:FRIEND]-(b)
RETURN b

// 2. LIMIT 일찍 적용
// Bad:
MATCH (p:Person)-[:POSTED]->(post)
RETURN p, post
ORDER BY post.created_at DESC
LIMIT 10

// Good:
MATCH (p:Person)-[:POSTED]->(post)
WITH p, post
ORDER BY post.created_at DESC
LIMIT 10
RETURN p, post

// 3. 필요한 속성만 RETURN
// Bad:
MATCH (p:Person)
RETURN p

// Good:
MATCH (p:Person)
RETURN p.name, p.email
```

---

## 사용 사례

### 소셜 네트워크

```cypher
// 6단계 분리 (Six Degrees of Separation)
MATCH path = shortestPath(
    (a:Person {name: 'Alice'})-[*]-(b:Person {name: 'Kevin Bacon'})
)
RETURN length(path), [n IN nodes(path) | n.name]

// 영향력 있는 사용자 (PageRank 스타일)
CALL gds.pageRank.stream('social-graph')
YIELD nodeId, score
RETURN gds.util.asNode(nodeId).name, score
ORDER BY score DESC
LIMIT 10
```

### 추천 시스템

```cypher
// 협업 필터링: 비슷한 취향의 사용자가 좋아한 상품
MATCH (me:User {id: 123})-[:PURCHASED]->(product)<-[:PURCHASED]-(similar)
MATCH (similar)-[:PURCHASED]->(recommended)
WHERE NOT (me)-[:PURCHASED]->(recommended)
RETURN recommended.name, COUNT(DISTINCT similar) as score
ORDER BY score DESC
LIMIT 10

// 콘텐츠 기반: 같은 카테고리, 같은 태그
MATCH (me:User {id: 123})-[:PURCHASED]->(p:Product)
MATCH (p)-[:IN_CATEGORY]->(cat)<-[:IN_CATEGORY]-(recommended)
WHERE NOT (me)-[:PURCHASED]->(recommended)
RETURN recommended.name, COUNT(*) as score
ORDER BY score DESC
LIMIT 10
```

### 사기 탐지

```cypher
// 공유 정보가 많은 계정 클러스터 찾기
MATCH (a:Account)-[:SHARES_PHONE|SHARES_IP|SHARES_DEVICE]-(b:Account)
WHERE a <> b
WITH a, b, COUNT(*) as shared_attributes
WHERE shared_attributes >= 2
RETURN a.id, b.id, shared_attributes

// 순환 거래 탐지
MATCH path = (a:Account)-[:TRANSFER*3..6]->(a)
WHERE ALL(t IN relationships(path) WHERE t.amount > 10000)
RETURN path
```

### 지식 그래프

```cypher
// 엔티티 관계 탐색
MATCH (c:Company {name: 'Tesla'})-[r]-(related)
RETURN type(r), labels(related), related.name

// 간접 관계 추론
MATCH (person:Person)-[:WORKS_AT]->(company:Company)-[:LOCATED_IN]->(city:City)
CREATE (person)-[:LIVES_NEAR]->(city)
```

---

## 참고 자료

- [Neo4j 공식 문서](https://neo4j.com/docs/)
- [Cypher Manual](https://neo4j.com/docs/cypher-manual/current/)
- [Graph Data Science Library](https://neo4j.com/docs/graph-data-science/current/)
- Graph Databases by Ian Robinson, Jim Webber, Emil Eifrem
