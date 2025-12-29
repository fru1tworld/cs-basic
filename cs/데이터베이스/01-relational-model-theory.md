# 관계형 모델 이론 (Relational Model Theory)

## 개요

관계형 모델은 1970년 Edgar F. Codd가 제안한 데이터베이스의 수학적 기반이다. 집합 이론과 1차 술어 논리에 기반하여, 데이터를 릴레이션(테이블)으로 표현하고 관계 대수를 통해 조작한다. 이 모델이 중요한 이유는 **데이터의 논리적 표현과 물리적 저장을 분리**하여, 데이터 독립성을 달성했기 때문이다.

## 핵심 개념

### 1. 릴레이션의 수학적 정의

릴레이션은 집합론적으로 다음과 같이 정의된다:

```
R ⊆ D₁ × D₂ × ... × Dₙ
```

- **도메인(Domain)**: 각 속성이 가질 수 있는 값들의 집합
- **튜플(Tuple)**: 릴레이션의 한 행, n개의 도메인에서 하나씩 선택한 값들의 순서쌍
- **속성(Attribute)**: 릴레이션의 열, 도메인에 이름을 부여한 것
- **차수(Degree)**: 속성의 개수
- **카디널리티(Cardinality)**: 튜플의 개수

```sql
-- 릴레이션 예시
CREATE TABLE Employee (
    emp_id INTEGER,      -- 도메인: 정수
    name VARCHAR(100),   -- 도메인: 문자열
    salary DECIMAL(10,2) -- 도메인: 실수
);
-- 차수(Degree): 3
-- 카디널리티: 튜플 수에 따라 변함
```

### 2. Codd의 12규칙

Codd는 1985년 "진정한" 관계형 DBMS가 갖춰야 할 12가지 규칙을 제시했다:

| 규칙 | 이름                    | 설명                                                 |
| ---- | ----------------------- | ---------------------------------------------------- |
| 0    | 기본 규칙               | 관계형 DBMS는 관계적 기능만으로 데이터를 관리해야 함 |
| 1    | 정보 규칙               | 모든 정보는 테이블 형태로 표현                       |
| 2    | 보장된 접근 규칙        | 테이블명 + 기본키 + 컬럼명으로 모든 데이터 접근 가능 |
| 3    | NULL 체계적 처리        | NULL은 "알 수 없음"을 일관되게 표현                  |
| 4    | 관계형 온라인 카탈로그  | 메타데이터도 테이블로 저장                           |
| 5    | 포괄적 데이터 하위 언어 | 모든 데이터 조작을 지원하는 언어 필요                |
| 6    | 뷰 갱신 규칙            | 이론적으로 갱신 가능한 뷰는 갱신 가능해야 함         |
| 7    | 고수준 삽입/갱신/삭제   | 집합 단위 연산 지원                                  |
| 8    | 물리적 데이터 독립성    | 물리적 구조 변경이 논리적 구조에 영향 없음           |
| 9    | 논리적 데이터 독립성    | 논리적 구조 변경이 응용에 영향 없음                  |
| 10   | 무결성 독립성           | 무결성 제약이 응용 프로그램과 독립                   |
| 11   | 분산 독립성             | 분산 환경에서도 동일하게 동작                        |
| 12   | 비전복 규칙             | 하위 수준 접근이 무결성을 우회할 수 없음             |

**실무적 의미**: 현대 RDBMS도 12규칙을 완전히 만족하지 못한다. 특히 규칙 6(뷰 갱신)과 규칙 9(논리적 독립성)는 여전히 제한적이다.

### 3. 관계 대수 (Relational Algebra)

관계 대수는 릴레이션을 입력받아 새로운 릴레이션을 출력하는 **절차적** 쿼리 언어다.

#### 기본 연산자

```
┌─────────────────────────────────────────────────────────────┐
│                    관계 대수 연산자                           │
├──────────────┬──────────────────────────────────────────────┤
│ 선택 (σ)     │ σ_condition(R) - 조건 만족하는 튜플 선택      │
│ 투영 (π)     │ π_attributes(R) - 특정 속성만 선택            │
│ 합집합 (∪)   │ R ∪ S - 두 릴레이션의 합집합                  │
│ 차집합 (−)   │ R − S - R에만 있는 튜플                       │
│ 카티션 곱(×) │ R × S - 모든 조합                             │
│ 이름변경 (ρ) │ ρ_name(R) - 릴레이션/속성명 변경              │
├──────────────┼──────────────────────────────────────────────┤
│ 유도 연산자                                                  │
├──────────────┼──────────────────────────────────────────────┤
│ 교집합 (∩)   │ R ∩ S = R − (R − S)                          │
│ 조인 (⋈)    │ R ⋈_cond S = σ_cond(R × S)                   │
│ 자연조인     │ R ⋈ S - 동일 속성으로 조인                    │
│ 나눗셈 (÷)   │ R ÷ S - S의 모든 값과 연결된 R의 값           │
└──────────────┴──────────────────────────────────────────────┘
```

#### SQL과 관계 대수의 매핑

```sql
-- 선택(σ): WHERE 절
SELECT * FROM Employee WHERE salary > 50000;
-- 관계 대수: σ_salary>50000(Employee)

-- 투영(π): SELECT 절
SELECT name, salary FROM Employee;
-- 관계 대수: π_name,salary(Employee)

-- 조인(⋈): JOIN
SELECT * FROM Employee E JOIN Department D ON E.dept_id = D.id;
-- 관계 대수: Employee ⋈_dept_id=id Department

-- 복합 쿼리
SELECT name FROM Employee WHERE dept_id IN (SELECT id FROM Department WHERE name = 'IT');
-- 관계 대수: π_name(Employee ⋈ σ_name='IT'(Department))
```

### 4. 관계 미적분 (Relational Calculus)

관계 미적분은 **선언적** 쿼리 언어로, "무엇을" 원하는지만 명시한다.

#### 튜플 관계 미적분 (Tuple Relational Calculus)

```
{ t | P(t) }
```

- 술어 P를 만족하는 튜플 t의 집합

```
-- "급여가 50000 이상인 직원의 이름"
{ t.name | Employee(t) ∧ t.salary > 50000 }

-- SQL로 표현
SELECT name FROM Employee WHERE salary > 50000;
```

#### 도메인 관계 미적분 (Domain Relational Calculus)

```
{ <x₁, x₂, ..., xₙ> | P(x₁, x₂, ..., xₙ) }
```

- 각 변수는 도메인 값을 나타냄

```
-- "급여가 50000 이상인 직원의 이름과 급여"
{ <n, s> | ∃i (Employee(i, n, s) ∧ s > 50000) }
```

### 5. 관계 대수와 관계 미적분의 동등성

Codd의 정리에 의해, **관계 대수와 안전한(safe) 관계 미적분은 표현력이 동등**하다.

- 관계 대수: 절차적 - "어떻게" 계산하는지 명시
- 관계 미적분: 선언적 - "무엇을" 원하는지 명시
- SQL: 관계 미적분에 가깝지만, 실행은 관계 대수로 변환됨

```
┌─────────────────────────────────────────────────────────────┐
│     SQL Query                                                │
│         ↓                                                    │
│     Parser (구문 분석)                                        │
│         ↓                                                    │
│     Relational Algebra Expression (관계 대수 표현)            │
│         ↓                                                    │
│     Query Optimizer (최적화)                                  │
│         ↓                                                    │
│     Execution Plan (실행 계획)                               │
└─────────────────────────────────────────────────────────────┘
```

### 6. 키(Key)의 정의

```
슈퍼키(Superkey): 튜플을 유일하게 식별할 수 있는 속성 집합
    └─ 후보키(Candidate Key): 최소 슈퍼키 (불필요한 속성 없음)
        └─ 기본키(Primary Key): 선택된 후보키 (NULL 불가)
외래키(Foreign Key): 다른 릴레이션의 기본키를 참조
```

```sql
-- 예: (emp_id)만으로 직원을 식별 가능
-- {emp_id} - 후보키이자 슈퍼키
-- {emp_id, name} - 슈퍼키 (최소가 아님)
-- {name} - 중복 가능하므로 키가 아님

CREATE TABLE Employee (
    emp_id INTEGER PRIMARY KEY,  -- 기본키
    dept_id INTEGER REFERENCES Department(id)  -- 외래키
);
```

## 실무 적용

### 논리적 vs 물리적 데이터 독립성

```
논리적 데이터 독립성
├── 뷰(View)를 통한 추상화
├── 테이블 구조 변경 시 뷰로 호환성 유지
└── 예: 테이블 분할 후 UNION ALL 뷰 제공

물리적 데이터 독립성
├── 인덱스 추가/삭제가 쿼리에 영향 없음
├── 파티셔닝 변경이 논리적 구조에 영향 없음
└── 스토리지 이동이 투명하게 처리됨
```

### 관계 대수 이해의 실무적 가치

1. **쿼리 최적화 이해**: EXPLAIN으로 보는 실행 계획이 관계 대수 연산의 구현
2. **조인 순서 최적화**: 관계 대수의 교환/결합 법칙 활용
3. **뷰 설계**: 복잡한 쿼리를 관계 대수적으로 분해하여 뷰 계층 설계

```sql
-- 관계 대수적 사고로 쿼리 최적화
-- Before: 카티션 곱 후 선택
SELECT * FROM A, B WHERE A.id = B.a_id AND A.status = 'active';

-- After: 선택 후 조인 (Predicate Pushdown)
SELECT * FROM (SELECT * FROM A WHERE status = 'active') A
JOIN B ON A.id = B.a_id;

-- 옵티마이저가 자동으로 이런 변환을 수행
```

## 참고 자료

- Codd, E.F. "A Relational Model of Data for Large Shared Data Banks" (1970)
- Codd, E.F. "Is Your DBMS Really Relational?" (1985)
- Database System Concepts (Silberschatz) - Chapter 2: Relational Model
- CMU 15-445 Lecture 1: Relational Model & Algebra
