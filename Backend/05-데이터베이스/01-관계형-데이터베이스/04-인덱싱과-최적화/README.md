# 인덱싱과 최적화

## 목차
1. [인덱스 기초](#인덱스-기초)
2. [B-Tree 인덱스](#b-tree-인덱스)
3. [B+Tree 인덱스](#btree-인덱스)
4. [Hash 인덱스](#hash-인덱스)
5. [기타 인덱스 유형](#기타-인덱스-유형)
6. [EXPLAIN 분석](#explain-분석)
7. [인덱스 설계 전략](#인덱스-설계-전략)
---

## 인덱스 기초

### 인덱스란?

인덱스는 데이터베이스 테이블의 검색 속도를 향상시키는 자료구조입니다. 책의 색인(Index)처럼 원하는 데이터를 빠르게 찾을 수 있게 해줍니다.

```
인덱스 없이 검색 (Full Table Scan):
┌────────────────────────────────────────┐
│ Page 1 → Page 2 → Page 3 → ... → Found │  O(n)
└────────────────────────────────────────┘

인덱스로 검색 (Index Scan):
┌────────────────────────────────────────┐
│      Root                              │
│       ↓                                │
│    Branch                              │  O(log n)
│       ↓                                │
│     Leaf → Data                        │
└────────────────────────────────────────┘
```

### 인덱스의 Trade-off

| 장점 | 단점 |
|------|------|
| SELECT 쿼리 성능 향상 | INSERT/UPDATE/DELETE 성능 저하 |
| ORDER BY, GROUP BY 최적화 | 추가 저장 공간 필요 |
| 고유성 보장 (UNIQUE) | 인덱스 유지 관리 비용 |

### 인덱스 생성과 관리

```sql
-- 단일 컬럼 인덱스
CREATE INDEX idx_users_email ON users(email);

-- 복합 인덱스 (Multi-column Index)
CREATE INDEX idx_orders_user_date ON orders(user_id, order_date);

-- 유니크 인덱스
CREATE UNIQUE INDEX idx_users_email_unique ON users(email);

-- 부분 인덱스 (PostgreSQL)
CREATE INDEX idx_active_users ON users(email) WHERE status = 'active';

-- 인덱스 삭제
DROP INDEX idx_users_email;

-- 인덱스 상태 확인 (MySQL)
SHOW INDEX FROM users;

-- 인덱스 크기 확인 (PostgreSQL)
SELECT
    indexrelname as index_name,
    pg_size_pretty(pg_relation_size(indexrelid)) as index_size
FROM pg_stat_user_indexes
WHERE schemaname = 'public';
```

---

## B-Tree 인덱스

### B-Tree 구조

B-Tree(Balanced Tree)는 자가 균형 트리 구조로, 모든 리프 노드가 같은 깊이에 있습니다.

```
                    ┌─────────────┐
                    │  [30, 60]   │  ← Root Node
                    └─────────────┘
                    /      |      \
                   /       |       \
         ┌─────────┐ ┌─────────┐ ┌─────────┐
         │[10, 20] │ │[40, 50] │ │[70, 80] │  ← Branch Nodes
         └─────────┘ └─────────┘ └─────────┘
          /   |   \    /   |   \    /   |   \
         ↓    ↓    ↓  ↓    ↓    ↓  ↓    ↓    ↓
        [데이터 포인터들]                      ← Leaf Nodes

B-Tree 특성:
- 노드는 여러 키와 자식 포인터를 가짐
- 모든 리프 노드는 같은 레벨
- 루트에서 모든 리프까지 경로 길이가 동일
- 검색, 삽입, 삭제 모두 O(log n)
```

### B-Tree 검색 과정

```sql
-- 예: SELECT * FROM users WHERE id = 45;

검색 과정:
1. Root [30, 60] → 45는 30~60 사이 → 중간 Branch로
2. Branch [40, 50] → 45는 40~50 사이 → 해당 Leaf로
3. Leaf에서 45 찾아서 데이터 포인터 반환

깊이가 3~4 정도면 수백만 행 커버 가능
(branching factor가 100이면: 100^4 = 1억 행)
```

### B-Tree가 적합한 경우

```sql
-- 1. 동등 비교 (=)
SELECT * FROM users WHERE email = 'test@example.com';

-- 2. 범위 검색 (<, >, BETWEEN)
SELECT * FROM orders WHERE order_date BETWEEN '2024-01-01' AND '2024-12-31';

-- 3. 정렬 (ORDER BY)
SELECT * FROM products ORDER BY price DESC;

-- 4. 접두사 검색 (LIKE 'prefix%')
SELECT * FROM users WHERE name LIKE 'Kim%';

-- B-Tree가 부적합한 경우:
-- LIKE '%suffix' → 인덱스 사용 불가
-- NOT IN, != → 대부분의 데이터 스캔
```

---

## B+Tree 인덱스

### B+Tree vs B-Tree 차이

```
B-Tree:
- 모든 노드에 키와 데이터 저장
- 중간 노드에서도 데이터 접근 가능

B+Tree (대부분 DB에서 사용):
- 리프 노드에만 데이터 저장
- 중간 노드는 키와 포인터만 저장
- 리프 노드끼리 연결 리스트로 연결
```

```
B+Tree 구조:
                    ┌─────────────┐
                    │  [30 | 60]  │  ← Root (키 + 포인터만)
                    └─────────────┘
                    /      |      \
         ┌─────────┐ ┌─────────┐ ┌─────────┐
         │[10|20]  │ │[40|50]  │ │[70|80]  │  ← Branch (키 + 포인터만)
         └─────────┘ └─────────┘ └─────────┘
              ↓           ↓           ↓
         ┌─────────┐ ┌─────────┐ ┌─────────┐
         │10→20→   │⟷│40→50→   │⟷│70→80→   │  ← Leaf (키 + 데이터)
         │[data]   │ │[data]   │ │[data]   │     (연결 리스트)
         └─────────┘ └─────────┘ └─────────┘
```

### B+Tree의 장점

```sql
-- 1. 범위 검색에 최적화 (리프 노드가 연결되어 있음)
SELECT * FROM orders WHERE price BETWEEN 100 AND 500;

-- 실행 과정:
-- 1. 인덱스에서 100 위치 찾기 (O(log n))
-- 2. 연결 리스트 따라가며 500까지 순차 스캔 (효율적!)

-- 2. 내부 노드가 작아서 더 많은 키 저장 가능
-- → 트리 깊이가 낮아짐 → 디스크 I/O 감소

-- 3. 전체 스캔도 효율적 (리프 노드만 순회)
SELECT COUNT(*) FROM orders WHERE status = 'completed';
```

### B+Tree 인덱스 동작 예시

```sql
-- 복합 인덱스에서의 B+Tree
CREATE INDEX idx_orders ON orders(user_id, order_date, status);

-- 최좌측 접두사 규칙 (Leftmost Prefix Rule)
-- 인덱스 사용 가능:
SELECT * FROM orders WHERE user_id = 1;  -- ✓
SELECT * FROM orders WHERE user_id = 1 AND order_date = '2024-01-01';  -- ✓
SELECT * FROM orders WHERE user_id = 1 AND order_date = '2024-01-01' AND status = 'completed';  -- ✓

-- 인덱스 사용 불가:
SELECT * FROM orders WHERE order_date = '2024-01-01';  -- ✗ (user_id 없음)
SELECT * FROM orders WHERE status = 'completed';  -- ✗

-- 부분 사용:
SELECT * FROM orders WHERE user_id = 1 AND status = 'completed';  -- △ (user_id만)
```

---

## Hash 인덱스

### Hash 인덱스 구조

```
Hash Function: key → bucket

┌───────────────────────────────────────────────┐
│   Key        │  Hash   │     Bucket           │
├───────────────────────────────────────────────┤
│ "alice"      │  h(key) │  → Bucket 3 → Data   │
│ "bob"        │  h(key) │  → Bucket 7 → Data   │
│ "charlie"    │  h(key) │  → Bucket 2 → Data   │
└───────────────────────────────────────────────┘

장점: O(1) 검색 (해시 충돌이 없는 경우)
단점: 범위 검색 불가, 정렬 불가
```

### Hash 인덱스 사용

```sql
-- PostgreSQL에서 Hash 인덱스 생성
CREATE INDEX idx_users_email_hash ON users USING hash(email);

-- MySQL Memory 엔진에서 Hash 인덱스
CREATE TABLE cache (
    key_name VARCHAR(100) PRIMARY KEY,
    value TEXT
) ENGINE=MEMORY;

-- Hash 인덱스가 적합한 경우
SELECT * FROM users WHERE email = 'exact@match.com';  -- ✓ O(1)

-- Hash 인덱스가 부적합한 경우
SELECT * FROM users WHERE email LIKE 'a%';  -- ✗
SELECT * FROM users WHERE email > 'a';  -- ✗
SELECT * FROM users ORDER BY email;  -- ✗
```

### Hash vs B-Tree 비교

| 특성 | Hash | B-Tree |
|------|------|--------|
| 동등 비교 (=) | O(1) - 매우 빠름 | O(log n) |
| 범위 검색 | 불가능 | 가능 |
| 정렬 | 불가능 | 가능 |
| LIKE 'prefix%' | 불가능 | 가능 |
| 메모리 사용 | 상대적으로 큼 | 효율적 |

---

## 기타 인덱스 유형

### GiST (Generalized Search Tree)

```sql
-- PostgreSQL에서 지리 정보, 전문 검색 등에 사용
CREATE EXTENSION postgis;

CREATE INDEX idx_locations_geom ON locations USING gist(geom);

-- 근처 위치 검색
SELECT * FROM locations
WHERE ST_DWithin(geom, ST_MakePoint(127.0, 37.5)::geography, 1000);
```

### GIN (Generalized Inverted Index)

```sql
-- 배열, JSONB, 전문 검색에 적합
CREATE INDEX idx_posts_tags ON posts USING gin(tags);
CREATE INDEX idx_documents_content ON documents USING gin(to_tsvector('english', content));

-- 태그 포함 검색
SELECT * FROM posts WHERE tags @> ARRAY['tech', 'database'];

-- 전문 검색
SELECT * FROM documents
WHERE to_tsvector('english', content) @@ to_tsquery('database & optimization');
```

### BRIN (Block Range Index)

```sql
-- 대용량 테이블에서 물리적으로 정렬된 데이터에 적합
CREATE INDEX idx_logs_timestamp ON logs USING brin(timestamp);

-- 특성:
-- - 매우 작은 인덱스 크기
-- - 삽입 속도 빠름
-- - 데이터가 물리적 순서와 상관관계가 있을 때 효과적
-- - 시계열 데이터, 로그 테이블에 적합

-- 인덱스 크기 비교
-- B-Tree: 테이블 크기의 2-3%
-- BRIN: 테이블 크기의 0.1% 이하
```

### 전문 검색 인덱스 (Full-Text Index)

```sql
-- MySQL
CREATE FULLTEXT INDEX idx_articles_content ON articles(title, content);

SELECT * FROM articles
WHERE MATCH(title, content) AGAINST('database optimization' IN NATURAL LANGUAGE MODE);

-- PostgreSQL
CREATE INDEX idx_articles_search ON articles
USING gin(to_tsvector('english', title || ' ' || content));

SELECT * FROM articles
WHERE to_tsvector('english', title || ' ' || content)
    @@ plainto_tsquery('english', 'database optimization');
```

---

## EXPLAIN 분석

### EXPLAIN 기본 사용법

```sql
-- 기본 EXPLAIN
EXPLAIN SELECT * FROM users WHERE email = 'test@example.com';

-- 실제 실행 통계 포함 (PostgreSQL)
EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'test@example.com';

-- 상세 옵션 (PostgreSQL)
EXPLAIN (ANALYZE, BUFFERS, TIMING, FORMAT TEXT)
SELECT * FROM orders WHERE user_id = 1;

-- MySQL
EXPLAIN FORMAT=JSON SELECT * FROM users WHERE id = 1;
```

### PostgreSQL EXPLAIN 출력 읽기

```sql
EXPLAIN ANALYZE
SELECT u.name, COUNT(o.id) as order_count
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
WHERE u.created_at > '2024-01-01'
GROUP BY u.id;

-- 출력 예시:
HashAggregate  (cost=1234.56..1234.78 rows=100 width=40)
               (actual time=15.234..15.567 rows=98 loops=1)
  Group Key: u.id
  ->  Hash Left Join  (cost=234.56..1200.00 rows=500 width=36)
                      (actual time=5.123..14.567 rows=450 loops=1)
        Hash Cond: (u.id = o.user_id)
        ->  Seq Scan on users u  (cost=0.00..123.00 rows=100 width=32)
                                 (actual time=0.015..1.234 rows=98 loops=1)
              Filter: (created_at > '2024-01-01')
              Rows Removed by Filter: 902
        ->  Hash  (cost=200.00..200.00 rows=2000 width=8)
                  (actual time=4.567..4.567 rows=1850 loops=1)
              ->  Seq Scan on orders o  (cost=0.00..200.00 rows=2000 width=8)
                                        (actual time=0.010..3.456 rows=1850 loops=1)
Planning Time: 0.234 ms
Execution Time: 15.789 ms
```

### 주요 스캔 유형

```sql
-- 1. Seq Scan (Sequential Scan) - 전체 테이블 스캔
EXPLAIN SELECT * FROM users WHERE status = 'active';
-- Seq Scan on users (cost=0.00..100.00 rows=1000 width=50)

-- 2. Index Scan - 인덱스 사용
EXPLAIN SELECT * FROM users WHERE email = 'test@example.com';
-- Index Scan using idx_users_email on users (cost=0.29..8.30 rows=1 width=50)

-- 3. Index Only Scan - 인덱스만으로 결과 반환 (Covering Index)
EXPLAIN SELECT email FROM users WHERE email = 'test@example.com';
-- Index Only Scan using idx_users_email on users (cost=0.29..4.30 rows=1 width=20)

-- 4. Bitmap Index Scan - 여러 조건을 비트맵으로 결합
EXPLAIN SELECT * FROM users WHERE age > 30 AND city = 'Seoul';
-- Bitmap Heap Scan on users
--   ->  BitmapAnd
--         ->  Bitmap Index Scan on idx_users_age
--         ->  Bitmap Index Scan on idx_users_city
```

### MySQL EXPLAIN 분석

```sql
EXPLAIN SELECT * FROM orders WHERE user_id = 1 AND status = 'completed';

-- 출력:
-- +----+-------------+--------+------+---------------+------+---------+-------+------+-------------+
-- | id | select_type | table  | type | possible_keys | key  | key_len | ref   | rows | Extra       |
-- +----+-------------+--------+------+---------------+------+---------+-------+------+-------------+
-- |  1 | SIMPLE      | orders | ref  | idx_user_stat | idx  | 8       | const |  150 | Using where |
-- +----+-------------+--------+------+---------------+------+---------+-------+------+-------------+

-- type 컬럼 (좋은 순서):
-- system > const > eq_ref > ref > range > index > ALL

-- const: Primary Key나 Unique 인덱스로 단일 행 조회
-- eq_ref: JOIN에서 Primary Key/Unique 인덱스 사용
-- ref: 인덱스를 사용한 동등 비교
-- range: 인덱스 범위 스캔
-- index: 전체 인덱스 스캔
-- ALL: 전체 테이블 스캔 (최악)
```

### 실행 계획 문제 패턴

```sql
-- 1. 인덱스 미사용 패턴
EXPLAIN SELECT * FROM users WHERE YEAR(created_at) = 2024;
-- 함수 적용 시 인덱스 사용 불가
-- 해결: WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01'

-- 2. 타입 불일치
EXPLAIN SELECT * FROM users WHERE phone = 12345678;
-- phone이 VARCHAR인데 숫자로 비교 → 인덱스 사용 불가
-- 해결: WHERE phone = '12345678'

-- 3. 부정 조건
EXPLAIN SELECT * FROM users WHERE status != 'deleted';
-- NOT 조건은 대부분 Full Scan
-- 해결: 긍정 조건으로 변경하거나 파티셔닝 고려

-- 4. OR 조건
EXPLAIN SELECT * FROM users WHERE email = 'a@test.com' OR phone = '123';
-- 개별 인덱스가 있어도 비효율적
-- 해결: UNION으로 분리
SELECT * FROM users WHERE email = 'a@test.com'
UNION
SELECT * FROM users WHERE phone = '123';
```

---

## 인덱스 설계 전략

### 복합 인덱스 설계

```sql
-- 선택도(Selectivity)가 높은 컬럼을 앞에
-- 선택도 = 고유값 수 / 전체 행 수

-- 예: status(5개 값), user_id(10000개 값)
CREATE INDEX idx_orders ON orders(user_id, status);  -- ✓ 좋음
CREATE INDEX idx_orders ON orders(status, user_id);  -- ✗ 나쁨

-- 등호 조건 컬럼을 앞에, 범위 조건 컬럼을 뒤에
-- 쿼리: WHERE user_id = ? AND created_at > ?
CREATE INDEX idx_orders ON orders(user_id, created_at);  -- ✓

-- 정렬 고려
-- 쿼리: WHERE user_id = ? ORDER BY created_at DESC
CREATE INDEX idx_orders ON orders(user_id, created_at DESC);
```

### Covering Index (커버링 인덱스)

```sql
-- 인덱스만으로 쿼리 결과 반환 (테이블 접근 불필요)

-- 쿼리: SELECT email, name FROM users WHERE email = ?
CREATE INDEX idx_users_email_name ON users(email, name);
-- 또는 PostgreSQL INCLUDE 절 사용
CREATE INDEX idx_users_email ON users(email) INCLUDE (name);

-- EXPLAIN에서 "Index Only Scan" 또는 "Using index" 확인
```

### 인덱스 유지 관리

```sql
-- 인덱스 사용률 확인 (PostgreSQL)
SELECT
    schemaname,
    relname as table_name,
    indexrelname as index_name,
    idx_scan as times_used,
    pg_size_pretty(pg_relation_size(indexrelid)) as size
FROM pg_stat_user_indexes
ORDER BY idx_scan DESC;

-- 미사용 인덱스 찾기
SELECT
    indexrelname,
    idx_scan
FROM pg_stat_user_indexes
WHERE idx_scan = 0
AND schemaname = 'public';

-- 인덱스 재구축 (Bloat 해소)
REINDEX INDEX idx_users_email;
REINDEX TABLE users;

-- MySQL에서 인덱스 최적화
ANALYZE TABLE users;
OPTIMIZE TABLE users;
```

---

## 참고 자료

- [Use The Index, Luke](https://use-the-index-luke.com/)
- [PostgreSQL B-Tree Index Explained](https://www.qwertee.io/blog/postgresql-b-tree-index-explained-part-1/)
- [MySQL Indexing Strategies](https://dev.mysql.com/doc/refman/8.0/en/optimization-indexes.html)
- [B-Trees and Database Indexes - PlanetScale](https://planetscale.com/blog/btrees-and-database-indexes)
- High Performance MySQL, 3rd Edition
- Designing Data-Intensive Applications by Martin Kleppmann
