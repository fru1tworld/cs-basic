# 보조 인덱스 (Secondary Indexes)

## 개요

보조 인덱스(Secondary Index)는 **기본키가 아닌 컬럼에 대한 인덱스**로, 다양한 검색 조건을 효율적으로 처리한다. Clustered Index와 Non-Clustered Index의 차이, Covering Index를 통한 Index-Only Scan 최적화, Include Columns 등을 이해하면 쿼리 성능을 크게 개선할 수 있다.

## 핵심 개념

### 1. Clustered vs Non-Clustered Index

```
┌─────────────────────────────────────────────────────────────┐
│              Clustered Index (클러스터형 인덱스)              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ - 테이블 데이터 자체가 인덱스 순서로 정렬                     │
│ - 테이블당 하나만 가능 (데이터 물리적 순서)                   │
│ - 리프 노드 = 실제 데이터 행                                 │
│                                                             │
│              ┌─────────────────┐                            │
│              │   [30] [70]     │  내부 노드                  │
│              └─────────────────┘                            │
│             /        |         \                            │
│    ┌────────┐  ┌────────┐  ┌────────┐                      │
│    │[10,김] │  │[30,이] │  │[70,박] │  리프 = 실제 데이터   │
│    │[20,최] │  │[50,정] │  │[90,한] │                      │
│    └────────┘  └────────┘  └────────┘                      │
│                                                             │
│ MySQL InnoDB: PK가 자동으로 클러스터형 인덱스               │
│ SQL Server: CREATE CLUSTERED INDEX                         │
│ PostgreSQL: 기본적으로 힙 (CLUSTER 명령으로 재배열 가능)     │
│                                                             │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│            Non-Clustered Index (비클러스터형 인덱스)          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ - 별도의 인덱스 구조, 데이터와 분리                          │
│ - 테이블당 여러 개 가능                                      │
│ - 리프 노드 = 인덱스 키 + 행 위치 (TID/RID/PK)              │
│                                                             │
│     인덱스 (name 컬럼)           테이블 (힙)                │
│     ┌───────────────┐          ┌───────────────┐          │
│     │ 김 → (1,0)    │ ------→  │ id=10, 김     │          │
│     │ 박 → (2,0)    │ ------→  │ id=70, 박     │          │
│     │ 이 → (1,1)    │ ------→  │ id=30, 이     │          │
│     └───────────────┘          └───────────────┘          │
│                                                             │
│ PostgreSQL: 모든 인덱스가 비클러스터형                       │
│ MySQL InnoDB: 보조 인덱스는 PK를 포함                       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 2. MySQL InnoDB의 보조 인덱스 구조

```
┌─────────────────────────────────────────────────────────────┐
│           InnoDB Secondary Index                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ 보조 인덱스 리프 노드 = [인덱스 키 | Primary Key]           │
│                                                             │
│ 검색 과정 (보조 인덱스 → PK 인덱스 → 데이터):                │
│                                                             │
│ 1. 보조 인덱스에서 조건에 맞는 리프 찾기                     │
│ 2. 리프에서 PK 값 추출                                      │
│ 3. PK 인덱스(클러스터)에서 실제 데이터 찾기                  │
│                                                             │
│ SELECT * FROM users WHERE name = '김';                      │
│                                                             │
│     name 인덱스            PK(id) 인덱스                    │
│     ┌───────────┐          ┌───────────────┐               │
│     │ 김 → id=10│ ------→  │ id=10, 모든 데이터│            │
│     └───────────┘          └───────────────┘               │
│            ↓                      ↓                        │
│     인덱스 검색 (1)        클러스터 검색 (2)                │
│                                                             │
│ → 보조 인덱스 검색은 최소 2번의 인덱스 탐색 필요            │
│   (이를 "더블 룩업" 또는 "북마크 룩업"이라 함)               │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 3. Covering Index (커버링 인덱스)

쿼리에 필요한 모든 컬럼이 인덱스에 포함되어 **테이블 접근 없이** 결과 반환:

```sql
-- 테이블: users(id PK, name, email, age, created_at)

-- 커버링 인덱스 생성
CREATE INDEX idx_name_email ON users (name, email);

-- 커버링 인덱스 사용 (Index-Only Scan)
SELECT name, email FROM users WHERE name = '김';
-- 인덱스만으로 결과 반환 가능

-- 커버링 안 됨 (테이블 접근 필요)
SELECT name, email, age FROM users WHERE name = '김';
-- age가 인덱스에 없으므로 테이블 접근 필요

-- MySQL: EXPLAIN에서 "Using index" 표시
EXPLAIN SELECT name, email FROM users WHERE name = '김';
-- Extra: Using index
```

### 4. Index-Only Scan

```
┌─────────────────────────────────────────────────────────────┐
│              Index-Only Scan                                 │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ 일반 인덱스 스캔:                                            │
│ 인덱스 → TID → 테이블 페이지 → 데이터                       │
│                                                             │
│ Index-Only Scan:                                            │
│ 인덱스 → 데이터 (테이블 접근 없음)                          │
│                                                             │
│ 장점:                                                        │
│ - 테이블 Random I/O 제거                                    │
│ - 인덱스가 테이블보다 작음 → 캐시 효율                      │
│ - 대폭적인 성능 향상                                         │
│                                                             │
│ PostgreSQL 특수 조건:                                        │
│ - Visibility Map에서 페이지가 all-visible이어야 함           │
│ - VACUUM이 VM을 갱신                                        │
│ - MVCC 가시성 확인을 위해 힙 방문이 필요할 수 있음           │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

```sql
-- PostgreSQL: Index-Only Scan 확인
EXPLAIN (ANALYZE) SELECT name, email FROM users WHERE name = '김';

-- Index Only Scan using idx_name_email on users
--   Heap Fetches: 0  ← 힙 접근 없음 = 완전한 Index-Only

-- Heap Fetches > 0이면 일부 페이지가 all-visible 아님
-- VACUUM으로 해결
VACUUM users;
```

### 5. Include Columns (INCLUDE 절)

인덱스 키가 아닌 컬럼을 리프 노드에만 저장:

```sql
-- PostgreSQL 11+, SQL Server 2005+

-- 일반 복합 인덱스
CREATE INDEX idx_name_age ON users (name, age);
-- name으로 검색, age로 정렬 모두 가능

-- Include 컬럼 사용
CREATE INDEX idx_name_include_age ON users (name) INCLUDE (age);
-- name으로만 검색 가능, age는 리프에만 저장

-- 차이점:
-- 복합 인덱스: [name, age]가 키 → 키 크기 증가, 정렬에 age 사용 가능
-- Include: [name]만 키 → 키 작음, age로 정렬 불가
```

```
┌─────────────────────────────────────────────────────────────┐
│              Include Columns                                 │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ 복합 인덱스 (name, age, email):                             │
│ 내부 노드: [name, age, email]                               │
│ 리프 노드: [name, age, email, TID]                          │
│ → 키 크기 큼, 팬아웃 낮음                                   │
│ → ORDER BY name, age 가능                                   │
│                                                             │
│ Include 사용 (name) INCLUDE (age, email):                   │
│ 내부 노드: [name]                                           │
│ 리프 노드: [name, TID, age, email]                          │
│ → 키 크기 작음, 팬아웃 높음                                 │
│ → ORDER BY name만 가능                                      │
│ → 커버링 인덱스로는 사용 가능                               │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**사용 시기:**
- 커버링 인덱스가 필요하지만
- 해당 컬럼으로 검색/정렬은 불필요할 때
- 키 크기를 줄여 인덱스 효율을 높이고 싶을 때

### 6. Clustered Index 선택 (MySQL InnoDB)

```sql
-- InnoDB에서 클러스터형 인덱스 선택:

-- 1. 명시적 PK가 있으면 PK가 클러스터형 인덱스
CREATE TABLE users (
    id INT PRIMARY KEY,  -- 클러스터형 인덱스
    name VARCHAR(100)
);

-- 2. PK 없고 UNIQUE NOT NULL이 있으면 첫 번째가 선택
CREATE TABLE users (
    id INT NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,  -- 클러스터형 인덱스
    name VARCHAR(100)
);

-- 3. 둘 다 없으면 InnoDB가 숨겨진 ROW_ID 생성
CREATE TABLE logs (
    message TEXT,
    created_at DATETIME
);
-- 숨겨진 6바이트 ROW_ID가 클러스터형 인덱스
```

**클러스터형 인덱스 설계 원칙:**
1. **짧은 키**: 보조 인덱스에 PK가 포함되므로 저장 공간 영향
2. **순차 증가**: 랜덤 삽입은 페이지 분할 유발
3. **변경 안 됨**: PK 변경 시 물리적 이동 필요

### 7. 복합 인덱스와 선두 컬럼

```sql
-- 복합 인덱스: (a, b, c)
CREATE INDEX idx_abc ON table (a, b, c);

-- 사용 가능한 조건:
WHERE a = 1                          -- ✓ 선두 컬럼
WHERE a = 1 AND b = 2                -- ✓ 선두부터 연속
WHERE a = 1 AND b = 2 AND c = 3      -- ✓ 전체 사용
WHERE a = 1 AND c = 3                -- △ a만 인덱스 사용

-- 사용 불가능:
WHERE b = 2                          -- ✗ 선두 컬럼 없음
WHERE b = 2 AND c = 3                -- ✗ 선두 컬럼 없음
WHERE c = 3                          -- ✗ 선두 컬럼 없음
```

### 8. 인덱스 선택도 (Selectivity)

```sql
-- 선택도 = 고유 값 수 / 전체 행 수

-- 높은 선택도 (인덱스 효과적)
-- PK, 이메일, 주민번호 등 거의 유일한 값
SELECT COUNT(DISTINCT id) / COUNT(*) FROM users;  -- ≈ 1.0

-- 낮은 선택도 (인덱스 비효율적)
-- 성별, 상태 등 값이 몇 개 안 되는 경우
SELECT COUNT(DISTINCT gender) / COUNT(*) FROM users;  -- ≈ 0.0001

-- 일반적으로 선택도 < 5-10%일 때 인덱스 스캔 고려
-- 너무 낮으면 (10-20% 이상) 풀 스캔이 더 효율적
```

## 실무 적용

### 커버링 인덱스 설계

```sql
-- 자주 실행되는 쿼리 분석
SELECT user_id, order_date, total_amount
FROM orders
WHERE user_id = ? AND status = 'completed'
ORDER BY order_date DESC
LIMIT 10;

-- 최적의 커버링 인덱스
CREATE INDEX idx_user_status_date ON orders
    (user_id, status, order_date DESC)
    INCLUDE (total_amount);

-- 결과:
-- 1. user_id, status로 빠른 필터링
-- 2. order_date로 정렬 (별도 정렬 불필요)
-- 3. total_amount 포함으로 Index-Only Scan
```

### 보조 인덱스 최적화 (MySQL InnoDB)

```sql
-- 문제: 보조 인덱스가 PK를 포함하므로 PK가 크면 비효율

-- 나쁜 예: UUID PK
CREATE TABLE orders (
    id CHAR(36) PRIMARY KEY,  -- 36바이트
    user_id INT,
    amount DECIMAL,
    INDEX idx_user (user_id)  -- 리프: [user_id, id(36B)]
);

-- 좋은 예: 자동 증가 정수 PK
CREATE TABLE orders (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,  -- 8바이트
    uuid CHAR(36) UNIQUE,  -- 외부 참조용
    user_id INT,
    amount DECIMAL,
    INDEX idx_user (user_id)  -- 리프: [user_id, id(8B)]
);
```

### 인덱스 사용 여부 확인

```sql
-- PostgreSQL
EXPLAIN (ANALYZE, BUFFERS)
SELECT name, email FROM users WHERE name = '김';

-- Index Scan vs Index Only Scan 확인
-- Buffers: shared hit vs read 확인

-- MySQL
EXPLAIN SELECT name, email FROM users WHERE name = '김';
-- type: ref, index, ALL 등 확인
-- Extra: Using index (커버링), Using filesort 등 확인
```

## 참고 자료

- "High Performance MySQL" - Chapter 5: Indexing for High Performance
- PostgreSQL Documentation: Indexes
- MySQL Documentation: InnoDB and the ACID Model
- Use The Index, Luke: https://use-the-index-luke.com/
- CMU 15-445: Tree Indexes
