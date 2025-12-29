# 힙 파일 조직 (Heap File Organization)

## 개요

힙 파일(Heap File)은 **튜플을 순서 없이 저장**하는 가장 기본적인 파일 조직 방식이다. 새 레코드는 빈 공간이 있는 아무 페이지에나 삽입된다. PostgreSQL의 기본 테이블 저장 방식이며, MySQL InnoDB는 클러스터링 인덱스를 기본으로 하지만 내부적으로 힙과 유사한 페이지 관리를 한다.

## 핵심 개념

### 1. 힙 파일 vs 정렬 파일

```
┌─────────────────────────────────────────────────────────────┐
│                    Heap File                                 │
├─────────────────────────────────────────────────────────────┤
│ ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐                            │
│ │ 5   │ │ 2   │ │ 8   │ │ 1   │  순서 없이 저장            │
│ │ 3   │ │ 9   │ │ 4   │ │ 7   │                            │
│ └─────┘ └─────┘ └─────┘ └─────┘                            │
│                                                             │
│ 삽입: O(1) - 빈 공간에 바로 삽입                            │
│ 검색: O(N) - 전체 스캔 필요                                  │
│ 삭제: O(N) + O(1) - 찾고 삭제 표시                          │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                    Sorted File                               │
├─────────────────────────────────────────────────────────────┤
│ ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐                            │
│ │ 1   │ │ 3   │ │ 5   │ │ 8   │  정렬 순서로 저장          │
│ │ 2   │ │ 4   │ │ 6   │ │ 9   │                            │
│ └─────┘ └─────┘ └─────┘ └─────┘                            │
│                                                             │
│ 삽입: O(N) - 정렬 위치 찾고 이동                            │
│ 검색: O(log N) - 이진 탐색 가능                              │
│ 범위 검색: 효율적 (연속 페이지)                              │
└─────────────────────────────────────────────────────────────┘
```

### 2. 힙 파일 관리 구조

```
┌─────────────────────────────────────────────────────────────┐
│                 Heap File Structure                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │              Header Page (Page 0)                     │  │
│  │  - Total pages                                        │  │
│  │  - Free page list pointer                            │  │
│  │  - Metadata                                          │  │
│  └──────────────────────────────────────────────────────┘  │
│           │                                                 │
│           ▼                                                 │
│  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐              │
│  │Page 1  │→│Page 2  │→│Page 3  │→│Page 4  │→ ...         │
│  │(full)  │ │(space) │ │(full)  │ │(space) │              │
│  └────────┘ └────────┘ └────────┘ └────────┘              │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 3. 빈 공간 관리 (Free Space Management)

**방법 1: Free List (연결 리스트)**

```
┌─────────────────────────────────────────────────────────────┐
│                    Free List                                 │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Header → Page 2 → Page 5 → Page 8 → NULL                   │
│            ↓         ↓         ↓                            │
│         [empty]   [empty]   [empty]                         │
│                                                             │
│ 삽입 시: Free list에서 페이지 가져옴                         │
│ 삭제 시: 빈 페이지를 Free list에 추가                        │
│                                                             │
│ 단점: 부분적으로 찬 페이지 관리 어려움                       │
└─────────────────────────────────────────────────────────────┘
```

**방법 2: Free Space Map (FSM)**

```
┌─────────────────────────────────────────────────────────────┐
│               Free Space Map (FSM)                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Page ID │ Free Space (bytes)                               │
│  ────────┼──────────────────                                │
│     0    │     200                                          │
│     1    │    4000                                          │
│     2    │       0  (full)                                  │
│     3    │    1500                                          │
│     4    │    8000  (empty)                                 │
│    ...   │    ...                                           │
│                                                             │
│ 100B 튜플 삽입: FSM에서 100B 이상 빈 페이지 찾기            │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 4. PostgreSQL Free Space Map

```
┌─────────────────────────────────────────────────────────────┐
│              PostgreSQL FSM Structure                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ FSM은 별도 파일 (relation_fsm)                              │
│                                                             │
│ 구조: 3-level tree                                          │
│                                                             │
│           [Root FSM Page]                                   │
│                  │                                          │
│     ┌────────────┼────────────┐                            │
│     │            │            │                            │
│ [Upper Page] [Upper Page] [Upper Page]                     │
│     │            │            │                            │
│ [Leaf Pages] [Leaf Pages] [Leaf Pages]                     │
│                                                             │
│ 각 페이지의 빈 공간을 1 byte (256 카테고리)로 표현          │
│ 0 = full, 255 = empty                                       │
│                                                             │
│ 조회: Root부터 내려가며 충분한 공간 가진 페이지 찾기        │
│ 갱신: 페이지 변경 시 FSM 갱신                               │
└─────────────────────────────────────────────────────────────┘
```

```sql
-- PostgreSQL: FSM 정보 확인
CREATE EXTENSION pg_freespacemap;

SELECT * FROM pg_freespace('mytable');
-- blkno | avail
-- ------+-------
--     0 |  2048
--     1 |  4096
--     2 |     0
```

### 5. Visibility Map (VM)

```
┌─────────────────────────────────────────────────────────────┐
│              PostgreSQL Visibility Map                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ 각 페이지당 2 bits:                                         │
│ - Bit 1: All-Visible (모든 튜플이 모든 트랜잭션에 보임)     │
│ - Bit 2: All-Frozen (모든 튜플이 Frozen 상태)               │
│                                                             │
│ 용도:                                                        │
│ 1. Index-Only Scan: VM 체크로 힙 방문 회피                  │
│ 2. VACUUM: All-visible 페이지 스킵                          │
│ 3. Anti-Wraparound: All-frozen 페이지 스킵                  │
│                                                             │
│ Page 0: [1,0] - All-visible, not frozen                     │
│ Page 1: [0,0] - Has dead tuples                             │
│ Page 2: [1,1] - All-visible and frozen                      │
│ Page 3: [1,0] - All-visible, not frozen                     │
└─────────────────────────────────────────────────────────────┘
```

```sql
-- VM 확인
CREATE EXTENSION pg_visibility;

SELECT * FROM pg_visibility_map('mytable');
-- blkno | all_visible | all_frozen
-- ------+-------------+------------
--     0 | t           | f
--     1 | f           | f
```

### 6. TOAST (The Oversized-Attribute Storage Technique)

```
┌─────────────────────────────────────────────────────────────┐
│                      TOAST                                   │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ 원본 테이블                     TOAST 테이블                │
│ ┌─────────────────────┐        ┌─────────────────────┐     │
│ │ id │ name │ bio_ptr │──────→ │ chunk_id │ chunk_data│     │
│ │ 1  │ Kim  │ (toast) │        │    1     │ [blob...]│     │
│ │ 2  │ Lee  │ (toast) │        │    2     │ [blob...]│     │
│ └─────────────────────┘        └─────────────────────┘     │
│                                                             │
│ TOAST 전략:                                                  │
│ - PLAIN: TOAST 안 함                                        │
│ - EXTENDED: 압축 후 외부 저장 (기본값)                      │
│ - EXTERNAL: 압축 없이 외부 저장                             │
│ - MAIN: 압축만, 외부 저장은 최후                            │
│                                                             │
│ 압축 알고리즘: pglz (기본), lz4 (PostgreSQL 14+)            │
└─────────────────────────────────────────────────────────────┘
```

```sql
-- TOAST 전략 변경
ALTER TABLE mytable ALTER COLUMN bio SET STORAGE EXTERNAL;

-- TOAST 테이블 확인
SELECT relname, reltoastrelid::regclass
FROM pg_class WHERE relname = 'mytable';
```

### 7. Overflow Pages (MySQL InnoDB)

```
┌─────────────────────────────────────────────────────────────┐
│              InnoDB Overflow Pages                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ ROW_FORMAT=COMPACT/REDUNDANT:                               │
│ - VARCHAR/BLOB 768B까지 인라인                              │
│ - 초과분은 overflow page                                    │
│                                                             │
│ ROW_FORMAT=DYNAMIC (기본, 5.7+):                            │
│ - 긴 컬럼은 20B 포인터만 인라인                             │
│ - 전체 데이터는 overflow page                               │
│                                                             │
│ Main Page                    Overflow Pages                 │
│ ┌──────────────────┐        ┌─────────────────┐            │
│ │ id │ name │ ptr  │───────→│ [full data...] │            │
│ │ 1  │ Kim  │ →    │        └─────────────────┘            │
│ └──────────────────┘                                        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 8. 힙 파일의 삭제 처리

```
┌─────────────────────────────────────────────────────────────┐
│              삭제 처리 방식                                   │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ 1. 삭제 표시 (Mark as Deleted)                              │
│    - 튜플 헤더에 삭제 표시                                   │
│    - 물리적 공간은 그대로                                    │
│    - PostgreSQL: xmax 설정                                   │
│    - MySQL: delete mark flag                                │
│                                                             │
│ 2. 공간 재사용                                               │
│    - PostgreSQL: VACUUM이 정리 후 FSM 갱신                  │
│    - MySQL: purge thread가 정리                             │
│                                                             │
│ 3. 페이지 Compaction                                         │
│    - 삭제된 공간을 모아 연속 빈 공간 생성                    │
│    - PostgreSQL: HOT pruning, VACUUM                        │
│    - MySQL: page reorganization                             │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 실무 적용

### 테이블 Bloat 관리

```sql
-- PostgreSQL: 테이블 bloat 확인
SELECT
    schemaname || '.' || relname AS table,
    pg_size_pretty(pg_total_relation_size(relid)) AS total_size,
    pg_size_pretty(pg_relation_size(relid)) AS table_size,
    n_dead_tup,
    n_live_tup,
    round(n_dead_tup::numeric / nullif(n_live_tup, 0) * 100, 2) AS dead_pct
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;

-- Bloat 심할 때 처리
VACUUM FULL mytable;  -- 테이블 잠금, 완전 재작성
-- 또는
CLUSTER mytable USING mytable_pkey;  -- 인덱스 순서로 재배열

-- 온라인 대안: pg_repack
-- sudo apt install postgresql-14-repack
-- pg_repack -d mydb -t mytable
```

### FSM/VM 갱신

```sql
-- VACUUM이 FSM/VM 갱신
VACUUM mytable;

-- 통계와 함께
VACUUM ANALYZE mytable;

-- 자동화
-- postgresql.conf
autovacuum = on
autovacuum_vacuum_threshold = 50
autovacuum_vacuum_scale_factor = 0.2  -- 20% dead tuples
```

### 대용량 컬럼 최적화

```sql
-- PostgreSQL: TOAST 활용
-- 큰 텍스트 컬럼을 압축 없이 외부 저장 (빠른 부분 읽기)
ALTER TABLE logs ALTER COLUMN message SET STORAGE EXTERNAL;

-- 압축 효율 우선
ALTER TABLE logs ALTER COLUMN message SET STORAGE EXTENDED;

-- MySQL: ROW_FORMAT 선택
ALTER TABLE mytable ROW_FORMAT=DYNAMIC;  -- 긴 컬럼 최적화
ALTER TABLE mytable ROW_FORMAT=COMPRESSED;  -- 전체 압축
```

## 참고 자료

- PostgreSQL Documentation: Database Physical Storage
- PostgreSQL Wiki: HOT (Heap-Only Tuples)
- "Database Internals" (Petrov) - Chapter 3, 4
- CMU 15-445: Storage Models & Data Layout
