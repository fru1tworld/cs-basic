# PostgreSQL 심화

## 목차
1. [PostgreSQL 아키텍처](#postgresql-아키텍처)
2. [MVCC (Multi-Version Concurrency Control)](#mvcc-multi-version-concurrency-control)
3. [확장 기능](#확장-기능)
4. [성능 최적화](#성능-최적화)
5. [고급 기능](#고급-기능)
---

## PostgreSQL 아키텍처

### 프로세스 구조

PostgreSQL은 멀티프로세스 아키텍처를 사용합니다:

```
┌─────────────────────────────────────────────────────────────┐
│                        Client Connections                    │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                      Postmaster (Main Process)               │
│  - 연결 요청 수신                                             │
│  - Backend Process 생성                                       │
│  - Background Worker 관리                                     │
└─────────────────────────────────────────────────────────────┘
                              │
            ┌─────────────────┼─────────────────┐
            ▼                 ▼                 ▼
     ┌────────────┐    ┌────────────┐    ┌────────────┐
     │  Backend   │    │  Backend   │    │  Backend   │
     │  Process   │    │  Process   │    │  Process   │
     │ (per conn) │    │ (per conn) │    │ (per conn) │
     └────────────┘    └────────────┘    └────────────┘
            │                 │                 │
            └─────────────────┼─────────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    Shared Memory                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │
│  │ Shared Buffer│  │   WAL Buffer │  │  CLOG Buffer │       │
│  │    Pool      │  │              │  │              │       │
│  └──────────────┘  └──────────────┘  └──────────────┘       │
└─────────────────────────────────────────────────────────────┘
                              │
            ┌─────────────────┼─────────────────┐
            ▼                 ▼                 ▼
     ┌────────────┐    ┌────────────┐    ┌────────────┐
     │  BG Writer │    │  WAL Writer│    │  Autovacuum│
     │            │    │            │    │  Launcher  │
     └────────────┘    └────────────┘    └────────────┘
```

### 주요 프로세스

| 프로세스 | 역할 |
|----------|------|
| Postmaster | 메인 프로세스, 연결 관리 및 다른 프로세스 생성 |
| Backend | 클라이언트 연결당 하나씩 생성, SQL 처리 |
| Background Writer | Shared Buffer의 Dirty Page를 주기적으로 디스크에 기록 |
| WAL Writer | WAL Buffer를 WAL 파일로 flush |
| Checkpointer | 체크포인트 수행, 모든 dirty page를 디스크에 기록 |
| Autovacuum | 자동으로 VACUUM 및 ANALYZE 수행 |
| Stats Collector | 통계 정보 수집 |
| Archiver | WAL 파일을 아카이브 위치로 복사 |

### 메모리 구조

```sql
-- 주요 메모리 파라미터 확인
SHOW shared_buffers;        -- 공유 버퍼 크기 (RAM의 25% 권장)
SHOW effective_cache_size;  -- OS 캐시 포함 예상 메모리
SHOW work_mem;              -- 정렬/해시 작업용 메모리 (per operation)
SHOW maintenance_work_mem;  -- VACUUM, CREATE INDEX 등에 사용
SHOW wal_buffers;           -- WAL 버퍼 크기

-- 권장 설정 예시 (32GB RAM 서버 기준)
-- shared_buffers = 8GB
-- effective_cache_size = 24GB
-- work_mem = 256MB
-- maintenance_work_mem = 2GB
```

---

## MVCC (Multi-Version Concurrency Control)

### MVCC 개념

MVCC는 "Reader는 Writer를 블로킹하지 않고, Writer도 Reader를 블로킹하지 않는" 동시성 제어 방식입니다.

```
Transaction Timeline:

Time →  ─────────────────────────────────────────────────────►

T1: ────[BEGIN]────[UPDATE row]────[COMMIT]────
                        │
                        ├── Old Version (xmin=100, xmax=101)
                        └── New Version (xmin=101, xmax=∞)

T2: ──────────[BEGIN]──────────────[SELECT]────[COMMIT]───
                                      │
                                      └── Sees: depends on isolation level
```

### Tuple 구조와 xmin/xmax

```sql
-- 시스템 컬럼 확인 (숨겨진 MVCC 정보)
SELECT xmin, xmax, ctid, * FROM users LIMIT 5;

-- xmin: 튜플을 생성한 트랜잭션 ID
-- xmax: 튜플을 삭제/수정한 트랜잭션 ID (0이면 활성 상태)
-- ctid: 물리적 위치 (page, offset)
```

**MVCC 동작 방식:**

```
┌──────────────────────────────────────────────────────────────┐
│                        Page (8KB)                            │
├──────────────────────────────────────────────────────────────┤
│  Tuple 1: { xmin=100, xmax=0,   data="Alice" }  ← 활성      │
│  Tuple 2: { xmin=100, xmax=105, data="Bob" }    ← 삭제됨    │
│  Tuple 3: { xmin=105, xmax=0,   data="Bobby" }  ← 새 버전   │
└──────────────────────────────────────────────────────────────┘

UPDATE 시:
1. 기존 Tuple의 xmax를 현재 트랜잭션 ID로 설정
2. 새로운 Tuple 생성 (xmin = 현재 트랜잭션 ID)
3. 기존 Tuple은 그대로 유지 (다른 트랜잭션이 읽을 수 있음)
```

### Visibility Check

```sql
-- 트랜잭션 가시성 규칙
/*
튜플이 보이려면:
1. xmin이 커밋된 트랜잭션이어야 함
2. xmin이 현재 스냅샷보다 이전이어야 함
3. xmax가 없거나, 아직 커밋되지 않았거나, 현재 스냅샷 이후여야 함
*/

-- 현재 트랜잭션 ID 확인
SELECT txid_current();

-- 스냅샷 정보 확인
SELECT txid_current_snapshot();
-- 결과: xmin:xmax:xip_list
-- 예: 100:105:102,103
-- 의미: 100~104 범위에서 102,103은 아직 진행 중
```

### VACUUM의 중요성

```sql
-- Dead Tuple 정리
VACUUM users;

-- 통계 정보도 함께 업데이트
VACUUM ANALYZE users;

-- 공간 반환까지 수행 (테이블 락 발생)
VACUUM FULL users;

-- Dead Tuple 비율 확인
SELECT
    schemaname,
    relname,
    n_live_tup,
    n_dead_tup,
    round(n_dead_tup::numeric / nullif(n_live_tup + n_dead_tup, 0) * 100, 2) as dead_ratio
FROM pg_stat_user_tables
WHERE n_dead_tup > 0
ORDER BY n_dead_tup DESC;

-- Autovacuum 설정
SHOW autovacuum_vacuum_threshold;        -- 기본 50
SHOW autovacuum_vacuum_scale_factor;     -- 기본 0.2 (20%)
-- VACUUM 트리거 조건: dead tuples > threshold + scale_factor * table_size
```

### Transaction ID Wraparound 방지

```sql
-- XID는 32비트 (약 42억)
-- Wraparound 방지를 위해 VACUUM이 필수

-- 현재 가장 오래된 XID 확인
SELECT
    datname,
    age(datfrozenxid) as xid_age,
    datfrozenxid
FROM pg_database
ORDER BY age(datfrozenxid) DESC;

-- Freeze 상태 확인
-- 20억에 가까워지면 위험
-- autovacuum_freeze_max_age 기본값: 2억
```

---

## 확장 기능

### 핵심 확장

```sql
-- 설치 가능한 확장 목록
SELECT * FROM pg_available_extensions ORDER BY name;

-- 설치된 확장 확인
SELECT * FROM pg_extension;
```

### pg_stat_statements (쿼리 성능 분석)

```sql
-- 확장 설치
CREATE EXTENSION pg_stat_statements;

-- 가장 느린 쿼리 Top 10
SELECT
    substring(query, 1, 100) as query,
    calls,
    round(total_exec_time::numeric, 2) as total_time_ms,
    round(mean_exec_time::numeric, 2) as mean_time_ms,
    round((100 * total_exec_time / sum(total_exec_time) OVER())::numeric, 2) as percentage
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;

-- 호출 빈도가 높은 쿼리
SELECT
    substring(query, 1, 100) as query,
    calls,
    round(mean_exec_time::numeric, 2) as mean_time_ms
FROM pg_stat_statements
ORDER BY calls DESC
LIMIT 10;
```

### PostGIS (지리 정보)

```sql
CREATE EXTENSION postgis;

-- 위치 데이터 저장
CREATE TABLE locations (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    geom GEOMETRY(Point, 4326)  -- WGS84 좌표계
);

-- 데이터 삽입
INSERT INTO locations (name, geom)
VALUES ('Seoul Tower', ST_SetSRID(ST_MakePoint(126.9882, 37.5512), 4326));

-- 반경 5km 내 위치 검색
SELECT name, ST_Distance(
    geom::geography,
    ST_SetSRID(ST_MakePoint(127.0, 37.5), 4326)::geography
) as distance_m
FROM locations
WHERE ST_DWithin(
    geom::geography,
    ST_SetSRID(ST_MakePoint(127.0, 37.5), 4326)::geography,
    5000  -- 5km
);
```

### pg_trgm (유사 문자열 검색)

```sql
CREATE EXTENSION pg_trgm;

-- 유사도 기반 인덱스
CREATE INDEX idx_products_name_trgm ON products
USING gin (name gin_trgm_ops);

-- 유사 문자열 검색
SELECT name, similarity(name, 'iPhone') as sim
FROM products
WHERE name % 'iPhone'  -- 유사도 임계값 이상
ORDER BY sim DESC;

-- 퍼지 검색
SELECT * FROM products
WHERE name ILIKE '%iphne%'  -- 오타 포함
ORDER BY similarity(name, 'iPhone') DESC;
```

### hstore (키-값 저장)

```sql
CREATE EXTENSION hstore;

-- hstore 컬럼 사용
ALTER TABLE products ADD COLUMN attributes hstore;

-- 데이터 삽입
UPDATE products
SET attributes = 'color => red, size => large, weight => 1.5kg'
WHERE id = 1;

-- 특정 키 조회
SELECT name, attributes -> 'color' as color
FROM products
WHERE attributes ? 'color';

-- GIN 인덱스
CREATE INDEX idx_products_attrs ON products USING gin (attributes);
```

### JSONB 활용

```sql
-- JSONB 컬럼 (바이너리 JSON, 인덱싱 가능)
CREATE TABLE events (
    id SERIAL PRIMARY KEY,
    data JSONB NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 데이터 삽입
INSERT INTO events (data) VALUES
('{"type": "click", "page": "/home", "user": {"id": 1, "name": "John"}}');

-- JSONB 연산자
SELECT
    data->>'type' as event_type,           -- 텍스트로 추출
    data->'user'->>'name' as user_name,    -- 중첩 접근
    data @> '{"type": "click"}' as is_click -- 포함 여부
FROM events;

-- GIN 인덱스
CREATE INDEX idx_events_data ON events USING gin (data);

-- 특정 경로 인덱스
CREATE INDEX idx_events_type ON events ((data->>'type'));

-- JSONB 함수
SELECT jsonb_pretty(data) FROM events;
SELECT jsonb_array_elements(data->'items') FROM events WHERE data ? 'items';
```

---

## 성능 최적화

### 쿼리 실행 계획 분석

```sql
-- 기본 EXPLAIN
EXPLAIN SELECT * FROM users WHERE email = 'test@example.com';

-- 실제 실행 통계 포함
EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'test@example.com';

-- 상세 정보 (버퍼 사용량, 타이밍)
EXPLAIN (ANALYZE, BUFFERS, TIMING, FORMAT TEXT)
SELECT * FROM users WHERE email = 'test@example.com';

-- JSON 형식 출력
EXPLAIN (ANALYZE, FORMAT JSON) SELECT * FROM users WHERE id < 1000;
```

**실행 계획 읽기:**

```
Seq Scan on users  (cost=0.00..1234.00 rows=10000 width=100)
                          │        │       │        │
                          │        │       │        └── 행당 평균 바이트
                          │        │       └── 예상 반환 행 수
                          │        └── 총 비용 (첫 행 ~ 모든 행)
                          └── 시작 비용 (첫 행 반환까지)
```

### 인덱스 전략

```sql
-- B-Tree (기본, 범위 쿼리에 적합)
CREATE INDEX idx_users_created ON users(created_at);

-- Hash (동등 비교에 최적화)
CREATE INDEX idx_users_email_hash ON users USING hash(email);

-- GiST (지리 정보, 범위 타입)
CREATE INDEX idx_locations_geom ON locations USING gist(geom);

-- GIN (배열, JSONB, 전문 검색)
CREATE INDEX idx_posts_tags ON posts USING gin(tags);

-- BRIN (대용량 테이블, 물리적 순서와 상관관계가 있는 컬럼)
CREATE INDEX idx_logs_timestamp ON logs USING brin(timestamp);

-- 커버링 인덱스 (Index-Only Scan 가능)
CREATE INDEX idx_users_email_name ON users(email) INCLUDE (name);

-- 인덱스 사용률 확인
SELECT
    schemaname,
    relname,
    indexrelname,
    idx_scan,
    idx_tup_read,
    idx_tup_fetch
FROM pg_stat_user_indexes
ORDER BY idx_scan DESC;
```

### 파티셔닝

```sql
-- 범위 파티셔닝
CREATE TABLE measurements (
    id BIGSERIAL,
    sensor_id INT NOT NULL,
    measured_at TIMESTAMP NOT NULL,
    value NUMERIC,
    PRIMARY KEY (id, measured_at)
) PARTITION BY RANGE (measured_at);

-- 월별 파티션 생성
CREATE TABLE measurements_2024_01 PARTITION OF measurements
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

CREATE TABLE measurements_2024_02 PARTITION OF measurements
    FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');

-- 파티션 자동 생성 함수
CREATE OR REPLACE FUNCTION create_monthly_partition()
RETURNS void AS $$
DECLARE
    partition_date DATE;
    partition_name TEXT;
    start_date DATE;
    end_date DATE;
BEGIN
    partition_date := DATE_TRUNC('month', CURRENT_DATE + INTERVAL '1 month');
    partition_name := 'measurements_' || TO_CHAR(partition_date, 'YYYY_MM');
    start_date := partition_date;
    end_date := partition_date + INTERVAL '1 month';

    EXECUTE format(
        'CREATE TABLE IF NOT EXISTS %I PARTITION OF measurements
         FOR VALUES FROM (%L) TO (%L)',
        partition_name, start_date, end_date
    );
END;
$$ LANGUAGE plpgsql;

-- 리스트 파티셔닝
CREATE TABLE orders (
    id BIGSERIAL,
    region VARCHAR(20) NOT NULL,
    order_date DATE NOT NULL,
    PRIMARY KEY (id, region)
) PARTITION BY LIST (region);

CREATE TABLE orders_asia PARTITION OF orders
    FOR VALUES IN ('KR', 'JP', 'CN');

CREATE TABLE orders_eu PARTITION OF orders
    FOR VALUES IN ('DE', 'FR', 'UK');
```

---

## 고급 기능

### Logical Replication

```sql
-- 퍼블리셔 (소스)
CREATE PUBLICATION my_pub FOR TABLE users, orders;

-- 구독자 (타겟)
CREATE SUBSCRIPTION my_sub
    CONNECTION 'host=source_host dbname=mydb user=repl_user'
    PUBLICATION my_pub;

-- 복제 상태 확인
SELECT * FROM pg_stat_subscription;
SELECT * FROM pg_replication_slots;
```

### Foreign Data Wrapper

```sql
-- 다른 PostgreSQL 서버 연결
CREATE EXTENSION postgres_fdw;

CREATE SERVER remote_server
    FOREIGN DATA WRAPPER postgres_fdw
    OPTIONS (host 'remote.server.com', port '5432', dbname 'remotedb');

CREATE USER MAPPING FOR current_user
    SERVER remote_server
    OPTIONS (user 'remote_user', password 'password');

-- 외부 테이블 가져오기
IMPORT FOREIGN SCHEMA public
    FROM SERVER remote_server
    INTO local_schema;

-- 또는 수동 정의
CREATE FOREIGN TABLE remote_users (
    id INTEGER,
    name TEXT,
    email TEXT
) SERVER remote_server OPTIONS (table_name 'users');
```

### Connection Pooling (PgBouncer)

```ini
# pgbouncer.ini
[databases]
mydb = host=127.0.0.1 port=5432 dbname=mydb

[pgbouncer]
listen_addr = 0.0.0.0
listen_port = 6432
auth_type = md5
auth_file = /etc/pgbouncer/userlist.txt

# Pool 설정
pool_mode = transaction  # session, transaction, statement
default_pool_size = 20
max_client_conn = 1000
min_pool_size = 5
reserve_pool_size = 5
```

---

## 참고 자료

- [PostgreSQL 공식 문서](https://www.postgresql.org/docs/current/)
- [PostgreSQL Internals - InterDB](https://www.interdb.jp/pg/)
- [Bruce Momjian Presentations](https://momjian.us/main/presentations/internals.html)
- [PostgreSQL Wiki - MVCC](https://wiki.postgresql.org/wiki/MVCC)
- Designing Data-Intensive Applications by Martin Kleppmann
