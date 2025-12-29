# 점화식 (Recurrence Relations)

## 개요

**점화식**(Recurrence Relation)은 재귀 알고리즘의 시간 복잡도를 표현하는 수학적 도구이다. 큰 문제의 해를 작은 부분 문제의 해로 표현하며, 이를 풀어 닫힌 형태(closed form)의 해를 구한다.

## 핵심 개념

### 1. 점화식의 형태

**분할 정복 점화식**:
```
T(n) = aT(n/b) + f(n)

- a: 재귀 호출 횟수
- n/b: 부분 문제의 크기
- f(n): 분할과 병합 비용
```

**일반 선형 점화식**:
```
T(n) = c₁T(n-1) + c₂T(n-2) + ... + cₖT(n-k) + f(n)
```

**예시**:
```python
# Merge Sort: T(n) = 2T(n/2) + O(n)
def merge_sort(arr):
    if len(arr) <= 1:
        return arr
    mid = len(arr) // 2
    left = merge_sort(arr[:mid])    # T(n/2)
    right = merge_sort(arr[mid:])   # T(n/2)
    return merge(left, right)        # O(n)

# Binary Search: T(n) = T(n/2) + O(1)
def binary_search(arr, target, lo, hi):
    if lo > hi:
        return -1
    mid = (lo + hi) // 2
    if arr[mid] == target:
        return mid
    elif arr[mid] < target:
        return binary_search(arr, target, mid + 1, hi)  # T(n/2)
    else:
        return binary_search(arr, target, lo, mid - 1)  # T(n/2)
    # 비교 비용: O(1)

# Fibonacci (naive): T(n) = T(n-1) + T(n-2) + O(1)
def fib(n):
    if n <= 1:
        return n
    return fib(n-1) + fib(n-2)  # T(n-1) + T(n-2)
```

### 2. Substitution Method (대입법)

**추측 후 증명**하는 방법이다.

**예제 1: Merge Sort**
```
점화식: T(n) = 2T(n/2) + n, T(1) = 1

추측: T(n) = O(n log n), 즉 T(n) ≤ cn log n for some c > 0

귀납적 증명:
기저: T(1) = 1 ≤ c·1·log(1) = 0 (실패)
     → 기저 조건 수정 필요

기저 수정: T(n) ≤ cn log n + n for n ≥ 1

귀납 단계: n/2에서 성립한다고 가정
T(n) = 2T(n/2) + n
     ≤ 2(c(n/2)log(n/2) + n/2) + n
     = cn log(n/2) + n + n
     = cn log n - cn log 2 + 2n
     = cn log n - cn + 2n
     ≤ cn log n (if c ≥ 2)

따라서 T(n) = O(n log n) ✓
```

**예제 2: T(n) = T(n/3) + T(2n/3) + n**
```
추측: T(n) = O(n log n)

증명:
T(n) ≤ c(n/3)log(n/3) + c(2n/3)log(2n/3) + n
     = cn/3·(log n - log 3) + 2cn/3·(log n - log(3/2)) + n
     = cn log n - cn/3·log 3 - 2cn/3·log(3/2) + n
     = cn log n - cn(log 3/3 + 2log(3/2)/3) + n
     ≤ cn log n (if c is large enough)

∵ log 3/3 + 2log(3/2)/3 ≈ 0.53 + 0.39 = 0.92 > 0
```

**흔한 실수와 해결**:
```
잘못된 추측: T(n) = 2T(n/2) + n, 추측 T(n) ≤ cn

T(n) = 2T(n/2) + n
     ≤ 2·c(n/2) + n
     = cn + n
     = (c+1)n  (cn보다 큼!)

문제: 상수가 n에 추가됨
해결: 더 정밀한 추측 (예: cn log n) 또는 하한 빼기

개선된 추측: T(n) ≤ cn log n - bn

T(n) = 2T(n/2) + n
     ≤ 2(c(n/2)log(n/2) - b(n/2)) + n
     = cn log(n/2) - bn + n
     = cn log n - cn - bn + n
     = cn log n - bn - (c-1)n
     ≤ cn log n - bn (if c ≥ 1) ✓
```

### 3. Recursion Tree Method (재귀 트리)

점화식을 트리로 시각화하여 각 레벨의 비용을 합산한다.

**예제: T(n) = 3T(n/4) + cn²**
```
Level 0:           cn²                        비용: cn²
                 /  |  \
Level 1:     c(n/4)² × 3                      비용: 3c(n/4)² = (3/16)cn²
             /|\   /|\   /|\
Level 2:  c(n/16)² × 9                        비용: 9c(n/16)² = (3/16)²cn²
           ...

레벨 i의 비용: (3/16)ⁱ × cn²
총 레벨 수: log₄n (n/4^i = 1일 때 i = log₄n)
리프 개수: 3^(log₄n) = n^(log₄3)

총 비용:
T(n) = Σ(i=0 to log₄n) (3/16)ⁱ × cn² + Θ(n^(log₄3))
     = cn² × Σ(i=0 to ∞) (3/16)ⁱ + Θ(n^0.79...)
     = cn² × 1/(1 - 3/16) + Θ(n^0.79...)
     = cn² × 16/13 + Θ(n^0.79...)
     = Θ(n²)  (∵ n² >> n^0.79)
```

**시각화**:
```
                    cn²                    ← Level 0: cn²
                  / | \
         c(n/4)²  ...  c(n/4)²             ← Level 1: 3×c(n/4)² = 3cn²/16
           /|\         /|\
          ...         ...                   ← Level 2: 9×c(n/16)² = 9cn²/256
          ...         ...
         T(1) ... T(1) ... T(1)            ← Level log₄n: n^(log₄3) leaves
```

**예제: T(n) = T(n/3) + T(2n/3) + n**
```
Level 0:              n                     비용: n
                    /   \
Level 1:         n/3   2n/3                 비용: n/3 + 2n/3 = n
                / \    / \
Level 2:     n/9 2n/9 2n/9 4n/9            비용: n

모든 레벨의 비용이 n

최소 깊이: log₃n (n/3만 따라가는 경로)
최대 깊이: log(3/2)n (2n/3만 따라가는 경로)

총 비용:
Ω(n × log₃n) ≤ T(n) ≤ O(n × log(3/2)n)
→ T(n) = Θ(n log n)
```

### 4. 선형 점화식 풀이

**형태**: T(n) = c₁T(n-1) + c₂T(n-2) + ... + cₖT(n-k)

**특성 방정식**을 이용한다:
```
x^k = c₁x^(k-1) + c₂x^(k-2) + ... + cₖ
```

**예제: 피보나치 수열**
```
점화식: F(n) = F(n-1) + F(n-2), F(0) = 0, F(1) = 1

특성 방정식: x² = x + 1
           x² - x - 1 = 0

근: x = (1 ± √5) / 2
   φ = (1 + √5) / 2 ≈ 1.618 (황금비)
   ψ = (1 - √5) / 2 ≈ -0.618

일반해: F(n) = Aφⁿ + Bψⁿ

초기 조건으로 A, B 결정:
F(0) = A + B = 0
F(1) = Aφ + Bψ = 1

풀면: A = 1/√5, B = -1/√5

닫힌 형태:
F(n) = (φⁿ - ψⁿ) / √5
     = (φⁿ - ψⁿ) / √5
     ≈ φⁿ / √5 (∵ |ψ| < 1)
     = Θ(1.618ⁿ)
```

**중복근이 있는 경우**:
```
점화식: T(n) = 4T(n-1) - 4T(n-2)

특성 방정식: x² - 4x + 4 = 0
           (x - 2)² = 0
           x = 2 (중복근)

일반해: T(n) = (A + Bn) × 2ⁿ

초기 조건으로 A, B 결정
```

### 5. 비동차 점화식

**형태**: T(n) = aT(n-1) + f(n)

```
예제: T(n) = 2T(n-1) + n, T(0) = 0

특수해 찾기: f(n) = n
추측: T_p(n) = An + B

T_p(n) = 2T_p(n-1) + n
An + B = 2(A(n-1) + B) + n
An + B = 2An - 2A + 2B + n
An + B = (2A + 1)n + (2B - 2A)

계수 비교:
A = 2A + 1 → A = -1
B = 2B - 2A → B = 2A = -2

특수해: T_p(n) = -n - 2

동차해: T_h(n) = c × 2ⁿ

일반해: T(n) = c × 2ⁿ - n - 2

초기 조건: T(0) = c - 0 - 2 = 0 → c = 2

최종: T(n) = 2^(n+1) - n - 2 = Θ(2ⁿ)
```

## 실무 적용

### 1. 점화식 설정 연습

```python
def analyze_algorithm(code_description):
    """알고리즘의 점화식 도출"""
    pass

# 예제 1: Karatsuba 곱셈
def karatsuba(x, y):
    """
    n자리 수의 곱셈
    x = x₁ × 10^(n/2) + x₀
    y = y₁ × 10^(n/2) + y₀

    xy = x₁y₁ × 10ⁿ + ((x₁+x₀)(y₁+y₀) - x₁y₁ - x₀y₀) × 10^(n/2) + x₀y₀

    점화식: T(n) = 3T(n/2) + O(n)
    - 3번의 재귀 호출 (x₁y₁, x₀y₀, (x₁+x₀)(y₁+y₀))
    - O(n) 덧셈/뺄셈
    """
    pass

# 예제 2: 최근접 점 쌍
def closest_pair(points):
    """
    점화식: T(n) = 2T(n/2) + O(n log n) 또는 O(n)

    O(n log n) 버전: 분할 후 각각 정렬
    O(n) 버전: 미리 정렬 후 유지
    """
    pass
```

### 2. 점화식 풀이 검증

```python
def verify_recurrence_solution(recurrence, solution, n_values):
    """점화식 해의 검증"""
    for n in n_values:
        # 점화식으로 직접 계산
        actual = recurrence(n)
        # 닫힌 형태로 계산
        predicted = solution(n)

        print(f"n={n}: actual={actual}, predicted={predicted}")

# 예: 피보나치
import math

def fib_recurrence(n, memo={}):
    if n in memo:
        return memo[n]
    if n <= 1:
        return n
    memo[n] = fib_recurrence(n-1, memo) + fib_recurrence(n-2, memo)
    return memo[n]

def fib_closed_form(n):
    phi = (1 + math.sqrt(5)) / 2
    psi = (1 - math.sqrt(5)) / 2
    return round((phi**n - psi**n) / math.sqrt(5))

# verify_recurrence_solution(fib_recurrence, fib_closed_form, [5, 10, 20, 30])
```

### 3. 실행 시간 측정과 점화식 검증

```python
import time

def empirical_complexity(func, base_size=1000, ratio=2, trials=5):
    """실험적으로 복잡도 추정"""
    sizes = [base_size * (ratio ** i) for i in range(trials)]
    times = []

    for n in sizes:
        data = list(range(n))
        import random
        random.shuffle(data)

        start = time.perf_counter()
        func(data)
        end = time.perf_counter()

        times.append(end - start)

    # 시간 비율로 복잡도 추정
    print(f"{'n':>10} {'time':>12} {'ratio':>10}")
    for i, (n, t) in enumerate(zip(sizes, times)):
        r = times[i] / times[i-1] if i > 0 else 0
        print(f"{n:>10} {t:>12.6f} {r:>10.2f}")

    # ratio ≈ 2: O(n)
    # ratio ≈ 2.x: O(n log n)
    # ratio ≈ 4: O(n²)
    # ratio ≈ 8: O(n³)
```

## 연습 문제

### 문제 1: 다음 점화식을 풀어라
```
a) T(n) = 2T(n/2) + n²
b) T(n) = T(n-1) + n
c) T(n) = 2T(n-1) + 1, T(1) = 1
d) T(n) = T(n/2) + T(n/4) + n
```

### 문제 2: 다음 알고리즘의 점화식을 세우고 풀어라
```python
def mystery(n):
    if n <= 1:
        return 1
    count = 0
    for i in range(n):
        count += 1
    return mystery(n // 2) + mystery(n // 2) + count
```

### 풀이
```
문제 1:
a) T(n) = 2T(n/2) + n²
   재귀 트리: 레벨 i에서 비용 = 2ⁱ × (n/2ⁱ)² = n²/2ⁱ
   총합: n² × Σ(1/2)ⁱ = n² × 2 = Θ(n²)

b) T(n) = T(n-1) + n
   T(n) = n + (n-1) + ... + 1 = n(n+1)/2 = Θ(n²)

c) T(n) = 2T(n-1) + 1
   특성방정식: x = 2 → T_h = c × 2ⁿ
   특수해: T_p = -1
   일반해: T(n) = c × 2ⁿ - 1
   T(1) = 2c - 1 = 1 → c = 1
   T(n) = 2ⁿ - 1 = Θ(2ⁿ)

d) T(n) = T(n/2) + T(n/4) + n
   추측: T(n) = O(n)
   T(n) ≤ cn/2 + cn/4 + n = 3cn/4 + n
   ≤ cn (if c ≥ 4)
   → T(n) = Θ(n)

문제 2:
점화식: T(n) = 2T(n/2) + n
→ Master Theorem Case 2: T(n) = Θ(n log n)
```

## 참고 자료

- CLRS Chapter 4: Divide-and-Conquer
- Concrete Mathematics (Graham, Knuth, Patashnik) - Chapter 7
- MIT 6.046J Design and Analysis of Algorithms - Recurrences
