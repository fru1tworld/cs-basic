# 함수적 종속성 (Functional Dependency)

## 개요

함수적 종속성(FD)은 릴레이션 내 속성들 간의 제약 조건을 형식화한 개념이다. 정규화 이론의 수학적 기초가 되며, 데이터 중복과 이상 현상(Anomaly)을 방지하는 스키마 설계의 핵심 도구다. FD를 이해하면 **왜 특정 테이블 구조가 문제가 되는지**, **어떻게 분해해야 하는지**를 이론적으로 판단할 수 있다.

## 핵심 개념

### 1. 함수적 종속성의 정의

속성 집합 X가 속성 집합 Y를 **함수적으로 결정**한다는 것은:

```
X → Y (X functionally determines Y)

모든 튜플 t1, t2에 대해:
t1[X] = t2[X] ⟹ t1[Y] = t2[Y]
```

X의 값이 같으면 Y의 값도 반드시 같다.

```sql
-- 예: Student(student_id, name, dept_id, dept_name)

-- 유효한 함수적 종속성
student_id → name, dept_id       -- 학번이 같으면 이름, 학과코드 동일
dept_id → dept_name              -- 학과코드가 같으면 학과명 동일

-- 무효한 함수적 종속성
name → student_id                -- 동명이인 존재 가능
dept_name → dept_id              -- 이론상 가능하나, 동명 학과 존재 가능
```

### 2. 함수적 종속성의 유형

```
┌─────────────────────────────────────────────────────────────┐
│                    FD 유형 분류                              │
├─────────────────────────────────────────────────────────────┤
│ 완전 함수적 종속 (Full FD)                                   │
│   X → Y이고, X의 어떤 진부분집합도 Y를 결정하지 못함          │
│   예: {student_id, course_id} → grade                       │
│       student_id만으로는 grade 결정 불가                     │
├─────────────────────────────────────────────────────────────┤
│ 부분 함수적 종속 (Partial FD)                                │
│   X → Y이고, X의 진부분집합도 Y를 결정함                     │
│   예: {student_id, course_id} → student_name                │
│       student_id만으로 student_name 결정 가능                │
├─────────────────────────────────────────────────────────────┤
│ 이행적 함수적 종속 (Transitive FD)                           │
│   X → Y, Y → Z이고, Y → X가 아닐 때, X → Z는 이행적 종속     │
│   예: student_id → dept_id → dept_name                      │
│       student_id → dept_name은 이행적 종속                   │
└─────────────────────────────────────────────────────────────┘
```

### 3. Armstrong's Axioms

FD로부터 새로운 FD를 유도하는 **완전하고 건전한** 추론 규칙:

```
기본 공리 (Sound and Complete)
┌─────────────────────────────────────────────────────────────┐
│ 1. 반사성 (Reflexivity)                                      │
│    Y ⊆ X ⟹ X → Y                                            │
│    예: {A, B} → A                                           │
├─────────────────────────────────────────────────────────────┤
│ 2. 증가성 (Augmentation)                                     │
│    X → Y ⟹ XZ → YZ                                          │
│    예: A → B이면 AC → BC                                    │
├─────────────────────────────────────────────────────────────┤
│ 3. 이행성 (Transitivity)                                     │
│    X → Y, Y → Z ⟹ X → Z                                     │
│    예: A → B, B → C이면 A → C                               │
└─────────────────────────────────────────────────────────────┘

유도 규칙
┌─────────────────────────────────────────────────────────────┐
│ 4. 합집합 (Union)                                            │
│    X → Y, X → Z ⟹ X → YZ                                    │
├─────────────────────────────────────────────────────────────┤
│ 5. 분해 (Decomposition)                                      │
│    X → YZ ⟹ X → Y, X → Z                                    │
├─────────────────────────────────────────────────────────────┤
│ 6. 의사이행 (Pseudo-transitivity)                            │
│    X → Y, WY → Z ⟹ WX → Z                                   │
└─────────────────────────────────────────────────────────────┘
```

**예제: 합집합 규칙 증명**
```
Given: X → Y, X → Z
Prove: X → YZ

1. X → Y                    (given)
2. X → XY                   (augmentation: X로 증가)
3. X → Z                    (given)
4. XY → YZ                  (augmentation: Y로 증가)
5. X → YZ                   (transitivity: 2, 4)
```

### 4. 속성 클로저 (Attribute Closure)

속성 집합 X에 대해, X로부터 FD를 통해 결정할 수 있는 모든 속성의 집합.

```
X⁺ = 속성 클로저 of X

알고리즘:
result = X
repeat:
    for each FD (Y → Z):
        if Y ⊆ result:
            result = result ∪ Z
until result doesn't change
return result
```

**예제:**
```
F = {A → B, B → C, C → D, D → E, BE → F}
A⁺ 계산:

초기: {A}
A → B 적용: {A, B}
B → C 적용: {A, B, C}
C → D 적용: {A, B, C, D}
D → E 적용: {A, B, C, D, E}
BE → F 적용: BE ⊆ {A,B,C,D,E}이므로 {A, B, C, D, E, F}

결과: A⁺ = {A, B, C, D, E, F}
```

**클로저 활용:**
- **슈퍼키 판별**: X⁺가 모든 속성을 포함하면 X는 슈퍼키
- **FD 검증**: X → Y가 성립하는지는 Y ⊆ X⁺로 확인

### 5. 정규 커버 (Canonical Cover)

FD 집합 F와 동등하면서 **최소화된** FD 집합.

```
정규 커버 조건:
1. F와 동치 (F⁺ = Fc⁺)
2. 좌변에 불필요한 속성 없음 (extraneous attributes 제거)
3. 우변에 불필요한 속성 없음
4. 각 FD의 좌변이 유일함 (동일 좌변 FD를 합침)
```

**알고리즘:**
```
1. 분해: 모든 FD를 우변이 단일 속성이 되도록 분해
   A → BC를 A → B, A → C로

2. 좌변 최소화: 각 FD X → A에서 X의 불필요한 속성 제거
   XY → A에서 X⁺에 A가 포함되면 Y → A로 대체

3. 불필요한 FD 제거: 다른 FD로 유도 가능한 FD 제거

4. 합집합: 동일 좌변 FD 합침
```

**예제:**
```
F = {A → BC, B → C, A → B, AB → C}

1단계: A → B, A → C, B → C, AB → C
2단계: AB → C에서 A⁺ = {A,B,C}이므로 C ∈ A⁺, 즉 A → C가 이미 있음
       AB → C 제거 가능
3단계: A → C는 A → B, B → C로 유도 가능, 제거
4단계: 결과 Fc = {A → B, B → C}
```

### 6. BCNF/3NF 분해 알고리즘

#### BCNF 분해 (무손실 보장)

```
BCNF_Decomposition(R, F):
    result = {R}
    while exists R_i in result that is not in BCNF:
        find X → Y in F⁺ that violates BCNF in R_i
        # X → Y 위반: X가 R_i의 슈퍼키가 아님
        R_i1 = X ∪ Y
        R_i2 = R_i - (Y - X)
        replace R_i with R_i1 and R_i2
    return result
```

**예제:**
```
R(A, B, C, D), F = {A → B, B → C}
키: {A, D}

1. B → C 위반 (B는 슈퍼키 아님)
   분해: R1(B, C), R2(A, B, D)

2. R1: B → C, B가 키 → BCNF
   R2: A → B 위반 (A는 슈퍼키 아님)
   분해: R21(A, B), R22(A, D)

결과: R1(B, C), R21(A, B), R22(A, D)
```

#### 3NF 분해 (무손실 + 종속성 보존)

```
3NF_Decomposition(R, F):
    Fc = canonical_cover(F)
    result = {}
    for each X → Y in Fc:
        result = result ∪ {XY}
    if no schema in result contains a candidate key of R:
        result = result ∪ {any candidate key}
    return result
```

### 7. 무손실 분해 검증

분해 R = R₁ ∪ R₂가 무손실인 조건:

```
(R₁ ∩ R₂) → R₁  또는  (R₁ ∩ R₂) → R₂

즉, 공통 속성이 적어도 한쪽의 슈퍼키여야 함
```

```sql
-- 손실 분해 예시
-- 원본: R(A, B, C) with {A → B}
-- 분해: R1(A, B), R2(B, C)
-- 공통: B, but B → A도 B → C도 아님

-- 데이터
Original:
| A | B | C |
|---|---|---|
| 1 | x | p |
| 2 | x | q |

After decomposition and join:
| A | B | C |
|---|---|---|
| 1 | x | p |
| 1 | x | q |  -- spurious tuple!
| 2 | x | p |  -- spurious tuple!
| 2 | x | q |

-- 가짜 튜플(spurious tuple) 발생
```

## 실무 적용

### FD 분석을 통한 테이블 설계

```sql
-- 문제 있는 설계
CREATE TABLE Orders (
    order_id INT,
    customer_id INT,
    customer_name VARCHAR(100),  -- customer_id → customer_name
    customer_email VARCHAR(100), -- customer_id → customer_email
    product_id INT,
    product_name VARCHAR(100),   -- product_id → product_name
    quantity INT,
    PRIMARY KEY (order_id)
);

-- FD 분석
-- order_id → customer_id, product_id, quantity
-- customer_id → customer_name, customer_email (이행적 종속)
-- product_id → product_name (이행적 종속)

-- 개선된 설계 (3NF)
CREATE TABLE Customers (
    customer_id INT PRIMARY KEY,
    customer_name VARCHAR(100),
    customer_email VARCHAR(100)
);

CREATE TABLE Products (
    product_id INT PRIMARY KEY,
    product_name VARCHAR(100)
);

CREATE TABLE Orders (
    order_id INT PRIMARY KEY,
    customer_id INT REFERENCES Customers,
    product_id INT REFERENCES Products,
    quantity INT
);
```

### 실무에서의 반정규화 결정

```sql
-- 정규화된 설계 (읽기 성능 저하)
SELECT o.*, c.customer_name, p.product_name
FROM Orders o
JOIN Customers c ON o.customer_id = c.customer_id
JOIN Products p ON o.product_id = p.product_id
WHERE o.order_date > '2024-01-01';

-- 반정규화 결정 기준
-- 1. 해당 조인이 자주 발생하는가?
-- 2. 참조 데이터가 거의 변경되지 않는가?
-- 3. 쓰기보다 읽기가 압도적으로 많은가?

-- 반정규화 예시 (읽기 최적화)
CREATE TABLE OrdersDenormalized (
    order_id INT PRIMARY KEY,
    customer_id INT,
    customer_name VARCHAR(100),  -- 중복 저장
    product_id INT,
    product_name VARCHAR(100),   -- 중복 저장
    quantity INT
);
-- 트레이드오프: 저장 공간↑, 갱신 이상 위험, 읽기 성능↑
```

## 참고 자료

- Armstrong, W.W. "Dependency Structures of Data Base Relationships" (1974)
- Database System Concepts (Silberschatz) - Chapter 7: Normalization
- Ramakrishnan, Gehrke - "Database Management Systems" Chapter 19
- CMU 15-445 Lecture: Normalization
