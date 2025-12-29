# 인덱스 (Index)

## 개요

인덱스는 데이터베이스에서 검색 속도를 향상시키기 위한 자료구조입니다. 책의 목차나 색인처럼 원하는 데이터를 빠르게 찾을 수 있게 해줍니다.

**핵심 질문**: 왜 인덱스가 필요한가?
- 테이블 Full Scan은 O(n) 시간 복잡도
- 인덱스를 사용하면 O(log n) 시간 복잡도로 검색 가능

## 핵심 개념

### 1. 인덱스의 원리

인덱스는 **정렬된 데이터 구조**입니다. 정렬되어 있기 때문에 이진 탐색이 가능합니다.

```
테이블 (정렬되지 않음):
+----+--------+
| id | name   |
+----+--------+
| 3  | Charlie|
| 1  | Alice  |
| 4  | David  |
| 2  | Bob    |
+----+--------+

인덱스 (name 기준 정렬):
+--------+-----+
| name   | ptr |
+--------+-----+
| Alice  | → 2 |
| Bob    | → 4 |
| Charlie| → 1 |
| David  | → 3 |
+--------+-----+
```

### 2. B-Tree vs B+Tree

#### B-Tree
- 모든 노드에 데이터 저장
- 검색 시 중간 노드에서도 찾을 수 있음
- 범위 검색에 비효율적

```
        [30, 60]
       /   |   \
   [10,20] [40,50] [70,80]
```

#### B+Tree (대부분의 RDBMS 사용)
- **리프 노드에만 실제 데이터** 저장
- 리프 노드끼리 **Linked List**로 연결 → 범위 검색에 유리
- 중간 노드는 인덱스 역할만

```
        [30, 60]              <- 인덱스 노드
       /   |   \
   [10,20]-[40,50]-[70,80]    <- 리프 노드 (데이터 + 링크)
```

**왜 B+Tree인가?**
1. 리프 노드가 연결되어 범위 검색(`BETWEEN`, `>`, `<`) 효율적
2. 중간 노드에 데이터가 없어 더 많은 키 저장 가능 → 트리 높이 낮음
3. 디스크 I/O 최소화

### 3. 클러스터드 인덱스 vs 논클러스터드 인덱스

#### 클러스터드 인덱스 (Clustered Index)
- **테이블당 1개만** 존재 가능
- 실제 데이터가 인덱스 순서대로 **물리적 정렬**
- 보통 Primary Key가 클러스터드 인덱스
- 검색 빠름, 삽입/수정 느림 (재정렬 필요)

```sql
-- MySQL에서 PK는 자동으로 클러스터드 인덱스
CREATE TABLE users (
    id INT PRIMARY KEY,  -- 클러스터드 인덱스
    name VARCHAR(100)
);
```

#### 논클러스터드 인덱스 (Non-Clustered Index)
- **테이블당 여러 개** 가능
- 별도의 인덱스 페이지에 저장
- 실제 데이터 위치를 가리키는 포인터 보유

```sql
-- 논클러스터드 인덱스 생성
CREATE INDEX idx_name ON users(name);
```

### 4. 복합 인덱스 (Composite Index)

여러 컬럼을 하나의 인덱스로 구성합니다.

```sql
CREATE INDEX idx_name_age ON users(name, age);
```

**중요: 컬럼 순서가 매우 중요**

```sql
-- 인덱스: (name, age)

-- O 인덱스 사용
SELECT * FROM users WHERE name = 'Alice';
SELECT * FROM users WHERE name = 'Alice' AND age = 25;

-- X 인덱스 미사용 (선행 컬럼 없음)
SELECT * FROM users WHERE age = 25;
```

**왜 순서가 중요한가?**
- B+Tree는 첫 번째 컬럼 기준으로 정렬
- 두 번째 컬럼은 첫 번째 컬럼 값이 같을 때만 정렬됨

### 5. 커버링 인덱스 (Covering Index)

쿼리에 필요한 모든 컬럼이 인덱스에 포함된 경우, 테이블 접근 없이 인덱스만으로 결과 반환.

```sql
-- 인덱스: (name, age)
SELECT name, age FROM users WHERE name = 'Alice';
-- 테이블 접근 불필요 → 매우 빠름
```

### 6. 인덱스가 사용되지 않는 경우

```sql
-- 1. 인덱스 컬럼에 함수나 연산 적용
SELECT * FROM users WHERE YEAR(created_at) = 2024;  -- X
SELECT * FROM users WHERE created_at >= '2024-01-01';  -- O

-- 2. 부정 조건
SELECT * FROM users WHERE status != 'active';  -- X

-- 3. LIKE 앞에 와일드카드
SELECT * FROM users WHERE name LIKE '%Alice';  -- X
SELECT * FROM users WHERE name LIKE 'Alice%';  -- O

-- 4. OR 조건 (각 컬럼에 인덱스가 없으면)
SELECT * FROM users WHERE name = 'Alice' OR age = 25;

-- 5. 암묵적 타입 변환
SELECT * FROM users WHERE phone = 01012345678;  -- phone이 VARCHAR면 X
```

### 7. 인덱스의 비용

인덱스는 **공짜가 아닙니다**.

| 장점 | 단점 |
|------|------|
| SELECT 성능 향상 | INSERT/UPDATE/DELETE 성능 저하 |
| 정렬 비용 감소 | 추가 저장 공간 필요 |
| | 인덱스 관리 오버헤드 |

**인덱스가 많으면?**
- 쓰기 작업마다 모든 인덱스 갱신
- 스토리지 낭비
- 옵티마이저 선택 복잡도 증가

## 실무 적용

### 인덱스 설계 가이드라인

1. **카디널리티(Cardinality)가 높은 컬럼 선택**
   - 카디널리티: 컬럼의 고유 값 개수
   - 성별(M/F)보다 주민번호가 인덱스에 적합

2. **WHERE, JOIN, ORDER BY에 자주 사용되는 컬럼**

3. **쓰기 빈도가 낮은 컬럼**

4. **복합 인덱스 순서**: 자주 사용되는 컬럼을 앞에

### 인덱스 모니터링

```sql
-- MySQL: 인덱스 사용 현황
SHOW INDEX FROM users;

-- 실행 계획 확인
EXPLAIN SELECT * FROM users WHERE name = 'Alice';
```

## 참고 자료

- [MySQL 공식 문서 - Optimization and Indexes](https://dev.mysql.com/doc/refman/8.0/en/optimization-indexes.html)
- [Use The Index, Luke](https://use-the-index-luke.com/) - SQL 인덱싱 튜토리얼
