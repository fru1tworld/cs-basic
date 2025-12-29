# 마스터 정리 (Master Theorem)

## 개요

**마스터 정리**(Master Theorem)는 분할 정복 알고리즘의 점화식 T(n) = aT(n/b) + f(n)을 닫힌 형태로 빠르게 풀 수 있는 정리이다. 1980년 Bentley, Haken, Saxe에 의해 제시되었다.

## 핵심 개념

### 1. 마스터 정리의 형태

**분할 정복 점화식**:
```
T(n) = aT(n/b) + f(n)

where:
- a ≥ 1: 부분 문제의 개수
- b > 1: 문제 크기 축소 비율
- f(n): 분할과 병합 비용 (점근적으로 양수)
```

**핵심 비교값**:
```
n^(log_b a) = 재귀 트리 리프 노드의 개수

이 값과 f(n)을 비교하여 어느 쪽이 지배적인지 결정
```

### 2. 세 가지 케이스

```
┌──────────┬────────────────────────────────┬─────────────────────┐
│  케이스  │            조건                │       해           │
├──────────┼────────────────────────────────┼─────────────────────┤
│ Case 1   │ f(n) = O(n^(log_b a - ε))     │ T(n) = Θ(n^(log_b a)) │
│ (리프)   │ for some ε > 0               │                     │
├──────────┼────────────────────────────────┼─────────────────────┤
│ Case 2   │ f(n) = Θ(n^(log_b a))         │ T(n) = Θ(n^(log_b a) log n) │
│ (균형)   │                               │                     │
├──────────┼────────────────────────────────┼─────────────────────┤
│ Case 3   │ f(n) = Ω(n^(log_b a + ε))     │ T(n) = Θ(f(n))      │
│ (루트)   │ for some ε > 0               │                     │
│          │ + regularity condition        │                     │
└──────────┴────────────────────────────────┴─────────────────────┘
```

**정규 조건 (Regularity Condition)**:
```
Case 3에서 추가로 필요:
af(n/b) ≤ cf(n) for some c < 1 and all sufficiently large n

이는 f(n)이 "규칙적으로 증가"함을 보장
대부분의 다항식 f(n)은 이 조건을 만족
```

### 3. 직관적 이해

```
재귀 트리 관점:

                    f(n)                    ← Level 0: f(n)
                   / | \
        f(n/b)  f(n/b) ... f(n/b)           ← Level 1: a × f(n/b)
          /|\     /|\       /|\
         ...     ...       ...              ← Level 2: a² × f(n/b²)
         ...     ...       ...
        T(1) ... T(1) ... T(1)              ← Level log_b n: a^(log_b n) = n^(log_b a) leaves

Case 1: 리프 비용 지배
- f(n)이 상대적으로 작음
- 총 비용 ≈ 리프 개수 = n^(log_b a)

Case 2: 각 레벨 비용 동일
- f(n) ≈ n^(log_b a)
- 총 비용 ≈ (레벨 수) × (레벨당 비용) = Θ(n^(log_b a) log n)

Case 3: 루트 비용 지배
- f(n)이 상대적으로 큼
- 총 비용 ≈ f(n)
```

### 4. 예제별 적용

**Case 1 예제: T(n) = 9T(n/3) + n**
```
a = 9, b = 3, f(n) = n
n^(log_b a) = n^(log_3 9) = n^2

f(n) = n vs n^2
n = O(n^(2-ε)) for ε = 1

→ Case 1 적용
→ T(n) = Θ(n²)
```

**Case 2 예제: T(n) = 2T(n/2) + n (Merge Sort)**
```
a = 2, b = 2, f(n) = n
n^(log_b a) = n^(log_2 2) = n^1 = n

f(n) = n = Θ(n^1) = Θ(n^(log_b a))

→ Case 2 적용
→ T(n) = Θ(n log n)
```

**Case 2 예제: T(n) = T(n/2) + 1 (Binary Search)**
```
a = 1, b = 2, f(n) = 1
n^(log_b a) = n^(log_2 1) = n^0 = 1

f(n) = 1 = Θ(1) = Θ(n^(log_b a))

→ Case 2 적용
→ T(n) = Θ(log n)
```

**Case 3 예제: T(n) = 3T(n/4) + n log n**
```
a = 3, b = 4, f(n) = n log n
n^(log_b a) = n^(log_4 3) ≈ n^0.79

f(n) = n log n vs n^0.79
n log n = Ω(n^(0.79+ε)) for ε = 0.2 (예: n^0.99도 가능)

정규 조건 확인:
af(n/b) = 3 × (n/4) log(n/4)
        = (3n/4)(log n - log 4)
        ≤ (3/4) n log n
        = cf(n) where c = 3/4 < 1 ✓

→ Case 3 적용
→ T(n) = Θ(n log n)
```

### 5. 적용 불가 사례

**케이스 사이에 떨어지는 경우**:
```
T(n) = 2T(n/2) + n/log n

a = 2, b = 2
n^(log_b a) = n

f(n) = n/log n

Case 1 확인: f(n) = O(n^(1-ε))?
n/log n = O(n^0.99)? → lim(n/log n)/(n^0.99) = lim n^0.01/log n → ∞
→ Case 1 아님

Case 2 확인: f(n) = Θ(n)?
n/log n = Θ(n)? → No, n/log n = o(n)
→ Case 2 아님

Case 3: f(n) = Ω(n^(1+ε))?
n/log n = Ω(n^1.01)? → No
→ Case 3 아님

→ 마스터 정리 적용 불가!
→ 다른 방법 필요 (재귀 트리, 대입법 등)
```

**해결책: 확장 마스터 정리 (Extended Master Theorem)**
```
Case 2의 확장:
f(n) = Θ(n^(log_b a) × (log n)^k) for k ≥ 0

→ T(n) = Θ(n^(log_b a) × (log n)^(k+1))

예: T(n) = 2T(n/2) + n log n
a = 2, b = 2, f(n) = n log n = n × (log n)^1
k = 1

→ T(n) = Θ(n × (log n)^2)
```

### 6. 증명 스케치

**재귀 트리를 이용한 증명**:
```
총 비용 = (비분할 비용의 합) + (리프 비용)
       = Σ(j=0 to log_b n - 1) a^j × f(n/b^j) + Θ(n^(log_b a))

g(n) = Σ(j=0 to log_b n - 1) a^j × f(n/b^j)

Case 1: g(n) = O(n^(log_b a))
→ 리프가 지배 → T(n) = Θ(n^(log_b a))

Case 2: g(n) = Θ(n^(log_b a) × log n)
→ 각 레벨 동일 → T(n) = Θ(n^(log_b a) log n)

Case 3: g(n) = Θ(f(n))
→ 루트가 지배 → T(n) = Θ(f(n))
```

## 실무 적용

### 1. 빠른 판별 코드

```python
import math

def master_theorem(a, b, f_n_degree, f_n_log_factor=0):
    """
    마스터 정리 적용

    T(n) = aT(n/b) + n^d × (log n)^k

    Parameters:
    - a: 부분 문제 개수
    - b: 축소 비율
    - f_n_degree: f(n)의 n의 차수 (d)
    - f_n_log_factor: f(n)의 log n의 차수 (k)

    Returns:
    - (case_number, complexity_string)
    """
    log_b_a = math.log(a) / math.log(b)

    if f_n_degree < log_b_a:
        # Case 1
        return (1, f"Θ(n^{log_b_a:.3f})")

    elif abs(f_n_degree - log_b_a) < 1e-9:
        # Case 2 (확장 포함)
        if f_n_log_factor == 0:
            return (2, f"Θ(n^{log_b_a:.3f} log n)")
        else:
            return (2, f"Θ(n^{log_b_a:.3f} (log n)^{f_n_log_factor + 1})")

    else:  # f_n_degree > log_b_a
        # Case 3 (정규 조건 가정)
        if f_n_log_factor == 0:
            return (3, f"Θ(n^{f_n_degree})")
        else:
            return (3, f"Θ(n^{f_n_degree} (log n)^{f_n_log_factor})")

# 테스트
print(master_theorem(2, 2, 1, 0))  # Merge Sort: Case 2, Θ(n log n)
print(master_theorem(1, 2, 0, 0))  # Binary Search: Case 2, Θ(log n)
print(master_theorem(9, 3, 1, 0))  # Case 1, Θ(n²)
print(master_theorem(3, 4, 1, 1))  # Case 3, Θ(n log n)
print(master_theorem(4, 2, 2, 0))  # T(n) = 4T(n/2) + n²: Case 2, Θ(n² log n)
```

### 2. 알고리즘 복잡도 분석표

```python
algorithms = [
    # (이름, a, b, f(n) 차수, f(n) log 차수)
    ("Binary Search", 1, 2, 0, 0),
    ("Merge Sort", 2, 2, 1, 0),
    ("Strassen's Matrix Mult", 7, 2, 2, 0),
    ("Naive Matrix Mult (D&C)", 8, 2, 2, 0),
    ("Karatsuba Multiplication", 3, 2, 1, 0),
    ("Closest Pair", 2, 2, 1, 0),
    ("Maximum Subarray (D&C)", 2, 2, 1, 0),
]

print(f"{'Algorithm':<30} {'Recurrence':<25} {'Complexity':<20}")
print("-" * 75)

for name, a, b, d, k in algorithms:
    case, complexity = master_theorem(a, b, d, k)
    recurrence = f"T(n) = {a}T(n/{b}) + n^{d}"
    if k > 0:
        recurrence += f"(log n)^{k}"
    print(f"{name:<30} {recurrence:<25} {complexity:<20}")
```

출력:
```
Algorithm                      Recurrence                Complexity
---------------------------------------------------------------------------
Binary Search                  T(n) = 1T(n/2) + n^0      Θ(n^0.000 log n)
Merge Sort                     T(n) = 2T(n/2) + n^1      Θ(n^1.000 log n)
Strassen's Matrix Mult         T(n) = 7T(n/2) + n^2      Θ(n^2.807)
Naive Matrix Mult (D&C)        T(n) = 8T(n/2) + n^2      Θ(n^3.000)
Karatsuba Multiplication       T(n) = 3T(n/2) + n^1      Θ(n^1.585)
Closest Pair                   T(n) = 2T(n/2) + n^1      Θ(n^1.000 log n)
Maximum Subarray (D&C)         T(n) = 2T(n/2) + n^1      Θ(n^1.000 log n)
```

### 3. 비균등 분할 처리

```python
def non_uniform_recurrence():
    """
    마스터 정리는 T(n) = aT(n/b) + f(n) 형태만 처리

    T(n) = T(n/3) + T(2n/3) + n 같은 비균등 분할은 적용 불가
    → Akra-Bazzi 정리 또는 재귀 트리 사용
    """
    # 재귀 트리로 분석
    # T(n) = T(n/3) + T(2n/3) + n
    #
    # 각 레벨에서 전체 비용의 합 = n
    # 최소 높이: log_3(n) (왼쪽 경로만)
    # 최대 높이: log_(3/2)(n) (오른쪽 경로만)
    #
    # 총 비용: Θ(n log n)
    pass
```

### 4. 정수 분할 처리

```python
def floor_ceil_handling():
    """
    실제 점화식: T(n) = 2T(⌊n/2⌋) + n

    마스터 정리는 T(n) = 2T(n/2) + n에 적용
    floor/ceil은 점근적으로 영향 없음 (CLRS Theorem 4.4)

    증명 아이디어:
    - ⌊n/2⌋ ≤ n/2 ≤ ⌈n/2⌉
    - n/2의 상수 배 범위 내에 있음
    - 점근적 분석에서 상수는 무시됨
    """
    pass
```

## 핵심 정리

### 외워야 할 공식

```
T(n) = aT(n/b) + f(n), c_crit = log_b(a)

1. f(n) = O(n^(c_crit - ε))  →  T(n) = Θ(n^c_crit)
2. f(n) = Θ(n^c_crit)        →  T(n) = Θ(n^c_crit log n)
3. f(n) = Ω(n^(c_crit + ε))  →  T(n) = Θ(f(n))  [+ regularity]
```

### 자주 나오는 패턴

```
T(n) = T(n/2) + O(1)      → Θ(log n)       [Binary Search]
T(n) = T(n/2) + O(n)      → Θ(n)           [Case 3]
T(n) = 2T(n/2) + O(1)     → Θ(n)           [Case 1]
T(n) = 2T(n/2) + O(n)     → Θ(n log n)     [Merge Sort]
T(n) = 2T(n/2) + O(n²)    → Θ(n²)          [Case 3]
T(n) = 3T(n/2) + O(n)     → Θ(n^1.58...)   [Case 1, Karatsuba]
T(n) = 4T(n/2) + O(n)     → Θ(n²)          [Case 1]
T(n) = 4T(n/2) + O(n²)    → Θ(n² log n)    [Case 2]
T(n) = 7T(n/2) + O(n²)    → Θ(n^2.81...)   [Strassen]
T(n) = 8T(n/2) + O(n²)    → Θ(n³)          [Naive Matrix]
```

## 참고 자료

- CLRS Chapter 4: Divide-and-Conquer (Section 4.5: Master Method)
- Bentley, Haken, Saxe - "A General Method for Solving Divide-and-Conquer Recurrences" (1980)
- MIT 6.046J - Master Method and Akra-Bazzi
