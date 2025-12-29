# 고급 인덱스 (Advanced Indexes)

## 개요

기본적인 B+Tree 인덱스 외에도, 특수한 쿼리 패턴을 효율적으로 처리하기 위한 다양한 인덱스 유형이 존재한다. Bitmap Index, Partial Index, Expression Index, Full-Text Index 등 각각의 인덱스는 특정 사용 사례에 최적화되어 있다. 이 문서에서는 각 인덱스의 내부 구조와 적합한 사용 시나리오를 다룬다.

## 핵심 개념

### 1. Bitmap Index

낮은 카디널리티(값의 종류가 적은) 컬럼에 효과적:

```
┌─────────────────────────────────────────────────────────────┐
│                    Bitmap Index                              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ 테이블: employees                                            │
│ ┌─────┬────────┬────────┬──────────┐                       │
│ │ RID │ gender │ status │ dept_id  │                       │
│ ├─────┼────────┼────────┼──────────┤                       │
│ │  0  │   M    │ active │    10    │                       │
│ │  1  │   F    │ active │    20    │                       │
│ │  2  │   M    │ inactive │  10    │                       │
│ │  3  │   F    │ active │    10    │                       │
│ │  4  │   M    │ active │    30    │                       │
│ └─────┴────────┴────────┴──────────┘                       │
│                                                             │
│ Bitmap Index (gender):                                      │
│ M: 1 0 1 0 1  (RID 0,2,4)                                   │
│ F: 0 1 0 1 0  (RID 1,3)                                     │
│                                                             │
│ Bitmap Index (status):                                       │
│ active:   1 1 0 1 1  (RID 0,1,3,4)                          │
│ inactive: 0 0 1 0 0  (RID 2)                                │
│                                                             │
│ 복합 조건: WHERE gender='M' AND status='active'             │
│ M AND active = 1 0 1 0 1 AND 1 1 0 1 1 = 1 0 0 0 1         │
│ → RID 0, 4                                                  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**장점:**
- 비트 연산으로 복합 조건 매우 빠름
- 압축 효율적 (Run-Length Encoding)
- OLAP/DW에서 유용

**단점:**
- 높은 카디널리티에서 비효율
- 쓰기 시 전체 비트맵 갱신
- OLTP에 부적합

```sql
-- Oracle에서 Bitmap Index 생성
CREATE BITMAP INDEX idx_gender ON employees (gender);

-- PostgreSQL에서는 Bitmap Index Scan만 지원
-- (런타임에 비트맵 생성, 저장하진 않음)
EXPLAIN SELECT * FROM employees WHERE gender = 'M' AND status = 'active';
-- Bitmap Heap Scan
--   -> BitmapAnd
--       -> Bitmap Index Scan on idx_gender
--       -> Bitmap Index Scan on idx_status
```

### 2. Partial Index (부분 인덱스)

테이블의 일부 행에만 인덱스를 생성:

```sql
-- PostgreSQL: 부분 인덱스
CREATE INDEX idx_active_orders ON orders (user_id, created_at)
WHERE status = 'active';

-- 활성 주문만 인덱스에 포함
-- 비활성 주문이 90%라면 인덱스 크기 90% 감소

-- 인덱스가 사용되는 쿼리
SELECT * FROM orders
WHERE user_id = 123 AND status = 'active';  -- ✓

-- 인덱스가 사용되지 않는 쿼리
SELECT * FROM orders
WHERE user_id = 123 AND status = 'pending';  -- ✗
SELECT * FROM orders WHERE user_id = 123;    -- ✗ (status 조건 없음)
```

**사용 시나리오:**
- 특정 상태의 행만 자주 조회
- NULL이 아닌 값만 인덱싱
- 최근 데이터만 인덱싱

```sql
-- NULL 제외
CREATE INDEX idx_email ON users (email) WHERE email IS NOT NULL;

-- 최근 데이터만
CREATE INDEX idx_recent_orders ON orders (created_at)
WHERE created_at > '2024-01-01';
```

### 3. Expression Index (표현식 인덱스)

함수나 표현식의 결과에 인덱스를 생성:

```sql
-- PostgreSQL: 함수 인덱스
CREATE INDEX idx_lower_email ON users (lower(email));

-- 인덱스 사용
SELECT * FROM users WHERE lower(email) = 'test@example.com';

-- 인덱스 미사용 (표현식이 다름)
SELECT * FROM users WHERE email = 'TEST@example.com';
SELECT * FROM users WHERE upper(email) = 'TEST@EXAMPLE.COM';

-- 날짜 추출
CREATE INDEX idx_order_year ON orders (EXTRACT(YEAR FROM created_at));
SELECT * FROM orders WHERE EXTRACT(YEAR FROM created_at) = 2024;

-- JSON 필드
CREATE INDEX idx_user_data ON users ((data->>'name'));
SELECT * FROM users WHERE data->>'name' = '김';
```

### 4. GiST Index (Generalized Search Tree)

다양한 데이터 타입의 검색을 지원하는 범용 인덱스:

```sql
-- PostgreSQL GiST

-- 기하학적 데이터 (위치 검색)
CREATE INDEX idx_location ON stores USING gist (location);
SELECT * FROM stores WHERE location <@ box '((0,0),(10,10))';

-- 범위 타입
CREATE INDEX idx_time_range ON events USING gist (duration);
SELECT * FROM events WHERE duration && '[2024-01-01, 2024-01-31]';

-- Full-Text Search
CREATE INDEX idx_content ON articles USING gist (to_tsvector('english', content));
SELECT * FROM articles
WHERE to_tsvector('english', content) @@ to_tsquery('database & index');
```

### 5. GIN Index (Generalized Inverted Index)

배열, JSONB, Full-Text 등 다중 값 컬럼에 적합:

```
┌─────────────────────────────────────────────────────────────┐
│                    GIN Index Structure                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ 배열 데이터:                                                 │
│ Row 1: tags = ['java', 'spring']                            │
│ Row 2: tags = ['python', 'django']                          │
│ Row 3: tags = ['java', 'hibernate']                         │
│                                                             │
│ GIN 인덱스 (역색인):                                         │
│ ┌────────────┬───────────┐                                  │
│ │ Key        │ Posting List │                               │
│ ├────────────┼───────────┤                                  │
│ │ django     │ [2]          │                               │
│ │ hibernate  │ [3]          │                               │
│ │ java       │ [1, 3]       │                               │
│ │ python     │ [2]          │                               │
│ │ spring     │ [1]          │                               │
│ └────────────┴───────────┘                                  │
│                                                             │
│ WHERE tags @> '{java}' → java의 posting list [1, 3] 반환    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

```sql
-- PostgreSQL GIN

-- 배열
CREATE INDEX idx_tags ON articles USING gin (tags);
SELECT * FROM articles WHERE tags @> ARRAY['java', 'spring'];

-- JSONB
CREATE INDEX idx_data ON users USING gin (data);
SELECT * FROM users WHERE data @> '{"role": "admin"}';

-- JSONB 특정 경로
CREATE INDEX idx_data_role ON users USING gin ((data -> 'role'));

-- Full-Text Search
CREATE INDEX idx_content ON articles USING gin (to_tsvector('english', content));
```

### 6. Full-Text Index 내부 구조

```
┌─────────────────────────────────────────────────────────────┐
│              Full-Text Index (역색인)                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ 문서:                                                        │
│ Doc 1: "The quick brown fox"                                │
│ Doc 2: "Quick brown dog"                                    │
│ Doc 3: "The lazy fox"                                       │
│                                                             │
│ 토큰화 및 정규화:                                            │
│ Doc 1: [the, quick, brown, fox]                             │
│ Doc 2: [quick, brown, dog]                                  │
│ Doc 3: [the, lazy, fox]                                     │
│                                                             │
│ 역색인:                                                      │
│ ┌────────┬───────────────────────┐                         │
│ │ Term   │ Posting List          │                         │
│ ├────────┼───────────────────────┤                         │
│ │ brown  │ [1:3, 2:2]            │ (doc:pos)              │
│ │ dog    │ [2:3]                 │                         │
│ │ fox    │ [1:4, 3:3]            │                         │
│ │ lazy   │ [3:2]                 │                         │
│ │ quick  │ [1:2, 2:1]            │                         │
│ │ the    │ [1:1, 3:1]            │                         │
│ └────────┴───────────────────────┘                         │
│                                                             │
│ 검색 "fox AND quick":                                       │
│ fox: [1, 3], quick: [1, 2]                                  │
│ 교집합: [1] → Doc 1                                         │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

```sql
-- PostgreSQL Full-Text Search
-- tsvector: 검색 가능한 문서
-- tsquery: 검색 쿼리

-- 인덱스 생성
CREATE INDEX idx_fts ON articles
USING gin (to_tsvector('english', title || ' ' || content));

-- 검색
SELECT * FROM articles
WHERE to_tsvector('english', title || ' ' || content)
    @@ to_tsquery('english', 'database & (index | optimization)');

-- 랭킹
SELECT title, ts_rank(
    to_tsvector('english', content),
    to_tsquery('database')
) as rank
FROM articles
ORDER BY rank DESC;
```

### 7. BRIN Index (Block Range Index)

물리적으로 정렬된 데이터에 효율적인 소형 인덱스:

```
┌─────────────────────────────────────────────────────────────┐
│                    BRIN Index                                │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ 테이블이 created_at 순서로 물리적 정렬된 경우:               │
│                                                             │
│ ┌────────────────────────────────────────────────┐         │
│ │ Block 0-127   │ min: 2024-01-01, max: 2024-01-31│         │
│ │ Block 128-255 │ min: 2024-02-01, max: 2024-02-28│         │
│ │ Block 256-383 │ min: 2024-03-01, max: 2024-03-31│         │
│ │ ...                                            │         │
│ └────────────────────────────────────────────────┘         │
│                                                             │
│ 검색: WHERE created_at BETWEEN '2024-02-01' AND '2024-02-15'│
│ → Block 128-255만 스캔                                      │
│                                                             │
│ 크기: B+Tree의 1/100 ~ 1/1000                               │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

```sql
-- PostgreSQL BRIN
CREATE INDEX idx_brin_date ON logs USING brin (created_at);

-- 적합한 경우:
-- 1. 데이터가 자연스럽게 정렬된 경우 (시계열)
-- 2. 매우 큰 테이블
-- 3. 범위 검색이 주된 패턴

-- 부적합한 경우:
-- 1. 데이터가 무작위로 삽입된 경우
-- 2. 정확한 값 검색이 필요한 경우
```

### 8. 인덱스 유형 비교

```
┌─────────────────────────────────────────────────────────────┐
│              인덱스 유형별 특성 비교                          │
├────────────────┬────────────────────────────────────────────┤
│ 유형           │ 적합한 사용 사례                            │
├────────────────┼────────────────────────────────────────────┤
│ B+Tree         │ 범용, 등호/범위 검색, 정렬                  │
│ Hash           │ 등호 검색만 (PostgreSQL 10+)               │
│ Bitmap         │ 낮은 카디널리티, OLAP, 복합 조건            │
│ GiST           │ 기하학적 데이터, 범위 타입, 근접 검색       │
│ GIN            │ 배열, JSONB, Full-Text                     │
│ BRIN           │ 물리적 정렬된 대용량 데이터                 │
│ Partial        │ 특정 조건의 행만 인덱싱                     │
│ Expression     │ 함수/표현식 결과 인덱싱                     │
└────────────────┴────────────────────────────────────────────┘
```

## 실무 적용

### JSONB 인덱스 전략

```sql
-- 전체 JSONB 인덱스 (모든 키-값)
CREATE INDEX idx_data ON users USING gin (data);
-- 크기가 클 수 있음, 모든 경로 검색 지원

-- 특정 경로만 인덱스
CREATE INDEX idx_data_email ON users USING btree ((data->>'email'));
-- 작은 크기, 해당 경로만 지원

-- jsonb_path_ops (포함 검색 최적화)
CREATE INDEX idx_data ON users USING gin (data jsonb_path_ops);
-- @> 연산자에 최적화, 키 존재 검사는 지원 안 함
```

### Full-Text Search 최적화

```sql
-- 저장된 tsvector 컬럼 사용 (미리 계산)
ALTER TABLE articles ADD COLUMN search_vector tsvector;
UPDATE articles SET search_vector = to_tsvector('english', title || ' ' || content);
CREATE INDEX idx_search ON articles USING gin (search_vector);

-- 트리거로 자동 갱신
CREATE TRIGGER update_search_vector
BEFORE INSERT OR UPDATE ON articles
FOR EACH ROW EXECUTE FUNCTION
  tsvector_update_trigger(search_vector, 'pg_catalog.english', title, content);
```

## 참고 자료

- PostgreSQL Documentation: Index Types
- "Database Internals" (Petrov) - Chapter 6
- PostgreSQL Wiki: GiST and GIN
- Oracle Documentation: Bitmap Indexes
