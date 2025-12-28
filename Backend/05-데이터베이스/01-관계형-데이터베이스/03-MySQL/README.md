# MySQL 심화

## 목차
1. [MySQL 아키텍처](#mysql-아키텍처)
2. [InnoDB vs MyISAM](#innodb-vs-myisam)
3. [복제(Replication)](#복제replication)
4. [파티셔닝](#파티셔닝)
5. [성능 최적화](#성능-최적화)
---

## MySQL 아키텍처

### 전체 구조

```
┌─────────────────────────────────────────────────────────────────┐
│                    Client Connections                            │
│              (mysql, JDBC, PHP, Python, etc.)                   │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                     Connection Layer                             │
│    ┌───────────────┐  ┌───────────────┐  ┌───────────────┐     │
│    │ Connection    │  │ Thread Pool   │  │ Authentication │     │
│    │ Handler       │  │               │  │ & Security     │     │
│    └───────────────┘  └───────────────┘  └───────────────┘     │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      SQL Layer (MySQL Server)                    │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐           │
│  │  Parser  │→│Preprocessor│→│ Optimizer│→│ Executor │           │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘           │
│                    ┌──────────────┐                              │
│                    │ Query Cache  │ (8.0에서 제거됨)              │
│                    └──────────────┘                              │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Storage Engine Layer                          │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐        │
│  │  InnoDB  │  │  MyISAM  │  │  Memory  │  │  Archive │        │
│  │ (Default)│  │          │  │          │  │          │        │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘        │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      File System                                 │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐        │
│  │ Data     │  │ Redo Log │  │ Undo Log │  │ Binary   │        │
│  │ Files    │  │ (ib_log) │  │          │  │ Log      │        │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘        │
└─────────────────────────────────────────────────────────────────┘
```

### InnoDB 아키텍처

```
┌─────────────────────────────────────────────────────────────────┐
│                    InnoDB Memory Structures                      │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                    Buffer Pool                           │    │
│  │  ┌────────────┐ ┌────────────┐ ┌────────────┐           │    │
│  │  │ Data Pages │ │Index Pages │ │ Change     │           │    │
│  │  │            │ │            │ │ Buffer     │           │    │
│  │  └────────────┘ └────────────┘ └────────────┘           │    │
│  │  ┌────────────┐ ┌────────────┐                          │    │
│  │  │Adaptive    │ │ Lock Info  │                          │    │
│  │  │Hash Index  │ │            │                          │    │
│  │  └────────────┘ └────────────┘                          │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐    │
│  │   Redo Log     │  │   Undo Log     │  │   Doublewrite  │    │
│  │   Buffer       │  │   Buffer       │  │   Buffer       │    │
│  └────────────────┘  └────────────────┘  └────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
```

---

## InnoDB vs MyISAM

### 핵심 차이점

| 특성 | InnoDB | MyISAM |
|------|--------|--------|
| **트랜잭션** | ACID 완벽 지원 | 지원 안함 |
| **락킹** | Row-level Locking | Table-level Locking |
| **외래키** | 지원 | 지원 안함 |
| **Crash Recovery** | 자동 복구 | 손상 위험 |
| **MVCC** | 지원 | 지원 안함 |
| **Full-text Search** | MySQL 5.6+ 지원 | 지원 |
| **COUNT(*)** | 느림 (Row scan) | 매우 빠름 (메타데이터) |
| **저장 방식** | Clustered Index | Heap + Index |
| **기본 엔진** | MySQL 5.5+ 기본 | 이전 버전 기본 |

### InnoDB 상세

```sql
-- Clustered Index (Primary Key 기준 물리적 정렬)
CREATE TABLE users (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,  -- Clustered Index
    email VARCHAR(255) UNIQUE,              -- Secondary Index
    name VARCHAR(100),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_created (created_at)          -- Secondary Index
) ENGINE=InnoDB;

-- Secondary Index는 Primary Key 값을 포함
-- SELECT * FROM users WHERE email = 'test@example.com';
-- 실행 순서:
-- 1. Secondary Index (email)에서 PK 찾기
-- 2. Clustered Index에서 PK로 실제 데이터 접근 (Double Lookup)
```

**InnoDB 파일 구조:**
```
┌────────────────────────────────────────────────────────────────┐
│ ibdata1 (System Tablespace)                                    │
│ - Data Dictionary                                               │
│ - Doublewrite Buffer                                           │
│ - Change Buffer                                                │
│ - Undo Logs (5.6 이전)                                         │
└────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────┐
│ *.ibd (File-Per-Table Tablespace)                              │
│ - Table Data                                                    │
│ - Indexes                                                       │
└────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────┐
│ ib_logfile0, ib_logfile1 (Redo Log)                            │
│ - Crash Recovery용                                              │
│ - Write-Ahead Logging                                          │
└────────────────────────────────────────────────────────────────┘
```

### MyISAM 특성

```sql
CREATE TABLE logs (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    message TEXT,
    created_at TIMESTAMP
) ENGINE=MyISAM;

-- MyISAM 파일 구조
-- logs.frm - 테이블 구조
-- logs.MYD - 데이터 (MyISAM Data)
-- logs.MYI - 인덱스 (MyISAM Index)

-- MyISAM의 장점: COUNT(*) 매우 빠름
SELECT COUNT(*) FROM logs;  -- 메타데이터에서 즉시 반환

-- 테이블 복구
REPAIR TABLE logs;
```

### 언제 어떤 엔진을 사용할까?

**InnoDB 사용 (대부분의 경우):**
- 트랜잭션이 필요한 경우
- 데이터 무결성이 중요한 경우
- 동시 쓰기가 많은 경우
- 외래키 제약이 필요한 경우

**MyISAM 고려 (특수한 경우):**
- 읽기 전용 데이터 (로그 아카이브)
- 단순 COUNT(*) 성능이 중요한 경우
- 레거시 시스템 호환성

```sql
-- 엔진 혼합 사용 예시
CREATE TABLE orders (
    id BIGINT PRIMARY KEY
) ENGINE=InnoDB;  -- 트랜잭션 필요

CREATE TABLE order_logs (
    id BIGINT PRIMARY KEY
) ENGINE=MyISAM;  -- 읽기 전용 로그
```

---

## 복제(Replication)

### 복제 유형

```
┌──────────────────────────────────────────────────────────────────┐
│                    Asynchronous Replication                       │
│                                                                   │
│   Master ─────[Binary Log]─────► Slave                           │
│     │                              │                              │
│   Write ◄──── Immediate ────►  Read                              │
│            (No wait for slave)                                    │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│                 Semi-Synchronous Replication                      │
│                                                                   │
│   Master ─────[Binary Log]─────► Slave                           │
│     │                              │                              │
│   Write ◄── Wait for ACK ──────►  ACK                            │
│         (At least 1 slave receives)                               │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│                   Group Replication (MGR)                         │
│                                                                   │
│   ┌────────┐    ┌────────┐    ┌────────┐                         │
│   │ Node 1 │◄──►│ Node 2 │◄──►│ Node 3 │                         │
│   │(Primary)│    │(Secondary)  │(Secondary)                       │
│   └────────┘    └────────┘    └────────┘                         │
│         Multi-Master / Single-Primary                             │
└──────────────────────────────────────────────────────────────────┘
```

### 기본 복제 설정

**Master 설정:**
```ini
# my.cnf (Master)
[mysqld]
server-id = 1
log_bin = mysql-bin
binlog_format = ROW
binlog_row_image = FULL
expire_logs_days = 7
sync_binlog = 1
innodb_flush_log_at_trx_commit = 1
```

```sql
-- Master에서 복제 사용자 생성
CREATE USER 'repl_user'@'%' IDENTIFIED BY 'password';
GRANT REPLICATION SLAVE ON *.* TO 'repl_user'@'%';

-- Master 상태 확인
SHOW MASTER STATUS;
-- +------------------+----------+
-- | File             | Position |
-- +------------------+----------+
-- | mysql-bin.000001 | 154      |
-- +------------------+----------+
```

**Slave 설정:**
```ini
# my.cnf (Slave)
[mysqld]
server-id = 2
relay_log = relay-bin
read_only = 1
log_slave_updates = 1
```

```sql
-- Slave에서 복제 설정
CHANGE MASTER TO
    MASTER_HOST = 'master_ip',
    MASTER_USER = 'repl_user',
    MASTER_PASSWORD = 'password',
    MASTER_LOG_FILE = 'mysql-bin.000001',
    MASTER_LOG_POS = 154;

-- 복제 시작
START SLAVE;

-- 복제 상태 확인
SHOW SLAVE STATUS\G
-- Slave_IO_Running: Yes
-- Slave_SQL_Running: Yes
-- Seconds_Behind_Master: 0
```

### GTID 기반 복제 (권장)

```ini
# my.cnf (Master & Slave)
[mysqld]
gtid_mode = ON
enforce_gtid_consistency = ON
```

```sql
-- GTID 기반 복제 설정
CHANGE MASTER TO
    MASTER_HOST = 'master_ip',
    MASTER_USER = 'repl_user',
    MASTER_PASSWORD = 'password',
    MASTER_AUTO_POSITION = 1;

-- GTID 확인
SELECT @@GLOBAL.GTID_EXECUTED;
```

### 복제 지연 모니터링

```sql
-- 복제 지연 확인
SHOW SLAVE STATUS\G

-- pt-heartbeat 활용 (더 정확)
-- Master에서:
-- pt-heartbeat --update --database mydb --create-table

-- Slave에서:
-- pt-heartbeat --monitor --database mydb
```

---

## 파티셔닝

### 파티션 유형

```sql
-- 1. RANGE 파티셔닝 (범위 기반)
CREATE TABLE orders (
    id BIGINT NOT NULL AUTO_INCREMENT,
    order_date DATE NOT NULL,
    customer_id INT,
    amount DECIMAL(10,2),
    PRIMARY KEY (id, order_date)
) PARTITION BY RANGE (YEAR(order_date)) (
    PARTITION p2022 VALUES LESS THAN (2023),
    PARTITION p2023 VALUES LESS THAN (2024),
    PARTITION p2024 VALUES LESS THAN (2025),
    PARTITION pmax VALUES LESS THAN MAXVALUE
);

-- 2. LIST 파티셔닝 (값 목록 기반)
CREATE TABLE users (
    id BIGINT NOT NULL,
    name VARCHAR(100),
    country_code CHAR(2),
    PRIMARY KEY (id, country_code)
) PARTITION BY LIST COLUMNS (country_code) (
    PARTITION p_asia VALUES IN ('KR', 'JP', 'CN', 'TW'),
    PARTITION p_europe VALUES IN ('DE', 'FR', 'UK', 'IT'),
    PARTITION p_america VALUES IN ('US', 'CA', 'MX', 'BR'),
    PARTITION p_other VALUES IN ('AU', 'NZ', 'IN')
);

-- 3. HASH 파티셔닝 (균등 분배)
CREATE TABLE sessions (
    id BIGINT NOT NULL AUTO_INCREMENT,
    user_id INT NOT NULL,
    data TEXT,
    PRIMARY KEY (id, user_id)
) PARTITION BY HASH (user_id)
PARTITIONS 8;

-- 4. KEY 파티셔닝 (MySQL 내부 해시)
CREATE TABLE logs (
    id BIGINT NOT NULL AUTO_INCREMENT,
    log_key VARCHAR(100),
    message TEXT,
    PRIMARY KEY (id, log_key)
) PARTITION BY KEY (log_key)
PARTITIONS 4;
```

### 파티션 관리

```sql
-- 파티션 추가
ALTER TABLE orders ADD PARTITION (
    PARTITION p2025 VALUES LESS THAN (2026)
);

-- 파티션 삭제 (데이터도 삭제됨 - 빠름!)
ALTER TABLE orders DROP PARTITION p2022;

-- 파티션 재구성
ALTER TABLE orders REORGANIZE PARTITION pmax INTO (
    PARTITION p2025 VALUES LESS THAN (2026),
    PARTITION pmax VALUES LESS THAN MAXVALUE
);

-- 파티션 정보 확인
SELECT
    PARTITION_NAME,
    PARTITION_METHOD,
    PARTITION_EXPRESSION,
    TABLE_ROWS
FROM INFORMATION_SCHEMA.PARTITIONS
WHERE TABLE_NAME = 'orders';

-- 파티션 프루닝 확인
EXPLAIN PARTITIONS SELECT * FROM orders
WHERE order_date BETWEEN '2024-01-01' AND '2024-12-31';
-- partitions: p2024 (해당 파티션만 스캔)
```

### 파티셔닝 주의사항

```sql
-- 1. Primary Key는 파티션 키를 포함해야 함
-- 잘못된 예:
CREATE TABLE bad_example (
    id BIGINT PRIMARY KEY,
    order_date DATE
) PARTITION BY RANGE (YEAR(order_date));  -- 에러!

-- 올바른 예:
CREATE TABLE good_example (
    id BIGINT,
    order_date DATE,
    PRIMARY KEY (id, order_date)
) PARTITION BY RANGE (YEAR(order_date));

-- 2. 외래키 제약 사용 불가

-- 3. 파티션 수 제한: 8192개 (5.7+)
```

---

## 성능 최적화

### Buffer Pool 최적화

```sql
-- Buffer Pool 상태 확인
SHOW ENGINE INNODB STATUS\G

-- Buffer Pool 크기 설정 (RAM의 70-80%)
SET GLOBAL innodb_buffer_pool_size = 8589934592;  -- 8GB

-- 여러 Buffer Pool 인스턴스 (멀티코어 활용)
-- my.cnf
-- innodb_buffer_pool_instances = 8

-- Buffer Pool 히트율 확인
SELECT
    (1 - (
        (SELECT VARIABLE_VALUE FROM performance_schema.global_status
         WHERE VARIABLE_NAME = 'Innodb_buffer_pool_reads') /
        (SELECT VARIABLE_VALUE FROM performance_schema.global_status
         WHERE VARIABLE_NAME = 'Innodb_buffer_pool_read_requests')
    )) * 100 as buffer_pool_hit_ratio;
-- 99% 이상이 이상적
```

### 쿼리 분석

```sql
-- Slow Query Log 활성화
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 1;
SET GLOBAL log_queries_not_using_indexes = 'ON';

-- EXPLAIN ANALYZE (8.0.18+)
EXPLAIN ANALYZE
SELECT u.name, COUNT(o.id)
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
WHERE u.created_at > '2024-01-01'
GROUP BY u.id;

-- 쿼리 프로파일링
SET profiling = 1;
SELECT * FROM users WHERE email = 'test@example.com';
SHOW PROFILES;
SHOW PROFILE FOR QUERY 1;

-- Performance Schema 활용
SELECT
    DIGEST_TEXT,
    COUNT_STAR,
    AVG_TIMER_WAIT/1000000000000 as avg_time_sec,
    SUM_ROWS_EXAMINED,
    SUM_ROWS_SENT
FROM performance_schema.events_statements_summary_by_digest
ORDER BY AVG_TIMER_WAIT DESC
LIMIT 10;
```

### 인덱스 최적화

```sql
-- 미사용 인덱스 확인
SELECT
    object_schema,
    object_name,
    index_name
FROM performance_schema.table_io_waits_summary_by_index_usage
WHERE index_name IS NOT NULL
AND count_star = 0
AND object_schema NOT IN ('mysql', 'performance_schema');

-- 중복 인덱스 확인
SELECT
    a.table_schema,
    a.table_name,
    a.index_name,
    b.index_name as redundant_index
FROM information_schema.statistics a
JOIN information_schema.statistics b
ON a.table_schema = b.table_schema
AND a.table_name = b.table_name
AND a.index_name != b.index_name
AND a.seq_in_index = b.seq_in_index
AND a.column_name = b.column_name;

-- Covering Index 활용
CREATE INDEX idx_covering ON orders(user_id, order_date, total_amount);
-- SELECT user_id, order_date, total_amount FROM orders WHERE user_id = ?
-- Index-Only Scan 가능
```

---

## 참고 자료

- [MySQL 공식 문서](https://dev.mysql.com/doc/)
- [High Performance MySQL](https://www.oreilly.com/library/view/high-performance-mysql/9781449332471/)
- [Percona Blog](https://www.percona.com/blog/)
- Designing Data-Intensive Applications by Martin Kleppmann
