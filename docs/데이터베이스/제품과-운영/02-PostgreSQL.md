# PostgreSQL

## PostgreSQL 아키텍처

### 프로세스 구조

PostgreSQL은 클라이언트 연결마다 Backend Process를 두는 멀티프로세스 구조다. 메인 프로세스인 Postmaster가 연결 요청을 받아 Backend를 생성하고 Background Worker를 관리한다.

여러 프로세스는 Shared Buffer Pool, WAL Buffer, CLOG Buffer 등의 공유 메모리를 사용한다. 쿼리를 처리하는 Backend 외에 BG Writer, WAL Writer, Autovacuum Launcher도 각자의 역할을 맡는다. 다음 목록에서 이 역할을 구분한다.

### 주요 프로세스

- 프로세스: Postmaster:
  - 역할: 메인 프로세스, 연결 관리 및 다른 프로세스 생성
- 프로세스: Backend:
  - 역할: 클라이언트 연결당 하나씩 생성, SQL 처리
- 프로세스: Background Writer:
  - 역할: Shared Buffer의 Dirty Page를 주기적으로 디스크에 기록
- 프로세스: WAL Writer:
  - 역할: WAL Buffer를 WAL 파일로 flush
- 프로세스: Checkpointer:
  - 역할: 체크포인트 수행, 모든 dirty page를 디스크에 기록
- 프로세스: Autovacuum:
  - 역할: 자동으로 VACUUM 및 ANALYZE 수행
- 프로세스: Stats Collector:
  - 역할: 통계 정보 수집
- 프로세스: Archiver:
  - 역할: WAL 파일을 아카이브 위치로 복사

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

`Seq Scan on users (cost=0.00..1234.00 rows=10000 width=100)`은 users를 순차 스캔하는 계획이다. `cost`의 첫 값 0.00은 첫 행을 반환하기까지의 시작 비용이고, 1234.00은 모든 행을 반환하기까지의 총 비용이다. `rows=10000`은 예상 반환 행 수, `width=100`은 행당 평균 바이트를 나타낸다.

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

## 참고 자료

- [PostgreSQL 공식 문서](https://www.postgresql.org/docs/current/)
- [PostgreSQL Internals - InterDB](https://www.interdb.jp/pg/)
- [Bruce Momjian Presentations](https://momjian.us/main/presentations/internals.html)
- [PostgreSQL Wiki - MVCC](https://wiki.postgresql.org/wiki/MVCC)
- Designing Data-Intensive Applications by Martin Kleppmann

## 관련 학습

- [문서DB MongoDB](01-문서DB-MongoDB.md)
- [MySQL](03-MySQL.md)
- [MVCC](../트랜잭션/04-MVCC.md)
