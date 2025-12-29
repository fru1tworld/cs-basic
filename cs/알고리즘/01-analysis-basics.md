# 알고리즘 분석 기초 (Algorithm Analysis Basics)

## 개요

알고리즘 분석은 알고리즘의 **자원 사용량**(시간, 공간)을 입력 크기의 함수로 표현하는 것이다. 이를 통해 알고리즘의 효율성을 객관적으로 비교하고, 입력 크기가 커질 때의 동작을 예측할 수 있다.

## 핵심 개념

### 1. RAM 모델 (Random Access Machine)

알고리즘 분석을 위한 **이론적 계산 모델**이다.

**RAM 모델의 가정**:
```
1. 단순 연산은 상수 시간: O(1)
   - 산술 연산: +, -, *, /, %
   - 비교 연산: <, >, ==, !=
   - 대입 연산: =
   - 배열 인덱스 접근: A[i]

2. 메모리 접근은 상수 시간: O(1)
   - 주소를 알면 즉시 접근 가능
   - 캐시 계층 구조 무시

3. 무한한 메모리 가정
   - 필요한 만큼의 메모리 사용 가능
```

**RAM 모델의 한계**:
```python
# 실제로는 다음 연산들의 비용이 다름

# 1. 캐시 효과 (연속 메모리 접근이 더 빠름)
for i in range(n):
    sum += arr[i]      # 캐시 친화적

for i in random_order:
    sum += arr[i]      # 캐시 비친화적

# 2. 큰 정수 연산 (자릿수에 비례)
x = 2 ** 1000         # O(1)이 아님
y = x * x             # 실제로는 O(k²) where k = 자릿수

# 3. 분기 예측 실패 비용
if random_condition:  # 분기 예측 실패 시 파이프라인 지연
    do_something()
```

### 2. 입력 크기 정의

알고리즘에 따라 **입력 크기**의 정의가 달라진다.

```
문제 유형              | 입력 크기 n
--------------------|-------------------
배열 정렬             | 원소의 개수
행렬 곱셈             | 행/열의 크기 (n×n)
그래프 알고리즘        | 정점 수 V, 간선 수 E
문자열 매칭           | 텍스트 길이 n, 패턴 길이 m
정수 연산             | 비트 수 (log n)
```

**입력 크기 선택의 중요성**:
```python
# 예: 소수 판정
def is_prime(n):
    """O(√n) vs O(√2^k) where k = bit length of n"""
    for i in range(2, int(n**0.5) + 1):
        if n % i == 0:
            return False
    return True

# n = 10^18이면:
# - 입력 크기를 n으로 보면: O(√n) = O(10^9)
# - 입력 크기를 비트 수 k=60으로 보면: O(2^(k/2)) = O(2^30) - 지수 시간!
#
# 암호학에서는 비트 수 기준이 중요 (다항 시간 vs 지수 시간)
```

### 3. 최선/평균/최악 케이스 분석

```python
def linear_search(arr, target):
    """선형 탐색의 세 가지 케이스 분석"""
    for i, x in enumerate(arr):
        if x == target:
            return i
    return -1
```

**세 가지 케이스**:
```
┌─────────────┬─────────────┬─────────────────────────────┐
│   케이스    │  시간 복잡도  │            조건              │
├─────────────┼─────────────┼─────────────────────────────┤
│ 최선 (Best) │    O(1)     │ target이 첫 번째 위치        │
│ 평균 (Avg)  │    O(n)     │ target이 균등 분포로 존재    │
│ 최악 (Worst)│    O(n)     │ target이 마지막이거나 없음   │
└─────────────┴─────────────┴─────────────────────────────┘
```

**Quick Sort 예시**:
```python
def quicksort(arr):
    if len(arr) <= 1:
        return arr
    pivot = arr[0]  # 첫 원소를 피벗으로
    left = [x for x in arr[1:] if x < pivot]
    right = [x for x in arr[1:] if x >= pivot]
    return quicksort(left) + [pivot] + quicksort(right)
```

```
┌─────────────┬─────────────┬─────────────────────────────┐
│   케이스    │  시간 복잡도  │            조건              │
├─────────────┼─────────────┼─────────────────────────────┤
│ 최선        │ O(n log n)  │ 피벗이 항상 중앙값           │
│ 평균        │ O(n log n)  │ 랜덤 입력                   │
│ 최악        │ O(n²)       │ 이미 정렬된 배열             │
└─────────────┴─────────────┴─────────────────────────────┘
```

### 4. 평균 케이스 분석

평균 케이스 분석은 **입력의 확률 분포**를 가정해야 한다.

```python
def average_case_linear_search(n):
    """
    가정: target이 배열에 있고, 모든 위치에 균등하게 분포

    기대 비교 횟수:
    E[비교 횟수] = Σ(i=1 to n) i × P(target이 i번째 위치)
                = Σ(i=1 to n) i × (1/n)
                = (1/n) × n(n+1)/2
                = (n+1)/2
                = O(n)
    """
    return (n + 1) / 2
```

**Insert Sort 평균 케이스**:
```python
def insertion_sort_analysis(n):
    """
    가정: 모든 순열이 균등하게 발생

    i번째 삽입 시 평균 비교 횟수:
    - 최선: 1번 (이미 정렬된 위치)
    - 최악: i번 (맨 앞으로 이동)
    - 평균: (1 + 2 + ... + i) / i = (i+1)/2

    전체 평균:
    E[총 비교] = Σ(i=1 to n-1) (i+1)/2
              = (1/2) × Σ(i=2 to n) i
              = (1/2) × (n(n+1)/2 - 1)
              ≈ n²/4
              = O(n²)
    """
    return n * n / 4
```

### 5. Adversarial Analysis (적대적 분석)

**적대자**(adversary)가 알고리즘에 가장 불리한 입력을 선택한다고 가정한다.

```python
def find_max_adversarial():
    """
    문제: n개 원소 중 최댓값 찾기

    적대자 전략:
    - 알고리즘이 어떤 비교를 하든, 적대자는 결과를 선택
    - 아직 '패배'하지 않은 원소를 최대한 유지

    정리: 최댓값을 찾으려면 최소 n-1번의 비교가 필요

    증명:
    - 최댓값은 한 번도 비교에서 진 적이 없어야 함
    - n-1개의 원소가 최소 1번씩 져야 함
    - 각 비교에서 최대 1개 원소가 짐
    → 최소 n-1번 비교 필요
    """
    pass

def find_min_max_adversarial(n):
    """
    문제: n개 원소 중 최솟값과 최댓값 동시에 찾기

    순진한 방법: 2n - 3번 비교
    - 최댓값 찾기: n-1번
    - 최솟값 찾기: n-2번 (최댓값 제외)

    최적 알고리즘: ⌈3n/2⌉ - 2번 비교
    - 쌍으로 비교하여 후보 분류
    """
    # 쌍 비교 알고리즘
    pairs = n // 2
    comparisons_for_pairs = pairs  # 각 쌍당 1번

    # 작은 것들 중 min, 큰 것들 중 max
    smaller_group = (pairs + n % 2)  # 홀수면 하나 추가
    larger_group = pairs

    min_comparisons = smaller_group - 1
    max_comparisons = larger_group - 1

    total = comparisons_for_pairs + min_comparisons + max_comparisons
    # = n//2 + (n//2 + n%2 - 1) + (n//2 - 1)
    # = 3n//2 - 2 (n이 짝수일 때)
    # = 3(n-1)//2 (n이 홀수일 때)
    return total
```

### 6. 점근적 표기법 (Asymptotic Notation)

**Big-O, Big-Ω, Big-Θ**의 엄밀한 정의:

```
Big-O (상한):
f(n) = O(g(n)) ⟺ ∃c > 0, n₀ > 0 : ∀n ≥ n₀, f(n) ≤ c·g(n)

Big-Ω (하한):
f(n) = Ω(g(n)) ⟺ ∃c > 0, n₀ > 0 : ∀n ≥ n₀, f(n) ≥ c·g(n)

Big-Θ (타이트 바운드):
f(n) = Θ(g(n)) ⟺ f(n) = O(g(n)) ∧ f(n) = Ω(g(n))
                ⟺ ∃c₁, c₂ > 0, n₀ > 0 : ∀n ≥ n₀, c₁·g(n) ≤ f(n) ≤ c₂·g(n)
```

**Little-o, Little-ω** (엄격한 바운드):
```
f(n) = o(g(n)) ⟺ lim(n→∞) f(n)/g(n) = 0
f(n) = ω(g(n)) ⟺ lim(n→∞) f(n)/g(n) = ∞
```

**예시와 증명**:
```python
def prove_big_o():
    """
    증명: 3n² + 2n + 1 = O(n²)

    c = 6, n₀ = 1로 설정

    n ≥ 1일 때:
    3n² + 2n + 1 ≤ 3n² + 2n² + n²  (∵ n ≥ 1이면 1 ≤ n²)
                = 6n²

    따라서 3n² + 2n + 1 ≤ 6·n² for all n ≥ 1 ✓
    """
    pass

def prove_big_theta():
    """
    증명: 3n² + 2n + 1 = Θ(n²)

    상한 (O): 위에서 증명

    하한 (Ω): c = 3, n₀ = 1
    3n² + 2n + 1 ≥ 3n² for all n ≥ 1 ✓

    따라서 3·n² ≤ 3n² + 2n + 1 ≤ 6·n² for all n ≥ 1
    → Θ(n²)
    """
    pass
```

**점근적 표기법의 성질**:
```
1. 전이성 (Transitivity):
   f(n) = O(g(n)) ∧ g(n) = O(h(n)) → f(n) = O(h(n))

2. 반사성 (Reflexivity):
   f(n) = O(f(n)), f(n) = Θ(f(n)), f(n) = Ω(f(n))

3. 대칭성 (Symmetry):
   f(n) = Θ(g(n)) ⟺ g(n) = Θ(f(n))

4. 전치 대칭성 (Transpose Symmetry):
   f(n) = O(g(n)) ⟺ g(n) = Ω(f(n))

5. 덧셈:
   O(f(n)) + O(g(n)) = O(max(f(n), g(n)))

6. 곱셈:
   O(f(n)) × O(g(n)) = O(f(n) × g(n))
```

### 7. 일반적인 복잡도 클래스

```
┌─────────────────┬──────────────┬─────────────────────────┐
│    복잡도       │     명칭     │         예시            │
├─────────────────┼──────────────┼─────────────────────────┤
│ O(1)           │ 상수         │ 배열 인덱스 접근         │
│ O(log n)       │ 로그         │ 이진 탐색               │
│ O(n)           │ 선형         │ 선형 탐색               │
│ O(n log n)     │ 선형 로그    │ 병합 정렬               │
│ O(n²)          │ 이차         │ 버블 정렬               │
│ O(n³)          │ 삼차         │ 행렬 곱셈 (naive)       │
│ O(2ⁿ)          │ 지수         │ 부분집합 열거           │
│ O(n!)          │ 팩토리얼     │ 순열 열거               │
└─────────────────┴──────────────┴─────────────────────────┘
```

**성장률 비교**:
```python
import math

def growth_comparison(n):
    """n이 커질 때 각 함수의 성장"""
    return {
        'log n': math.log2(n),
        'n': n,
        'n log n': n * math.log2(n),
        'n²': n ** 2,
        'n³': n ** 3,
        '2^n': 2 ** n if n <= 30 else float('inf'),
        'n!': math.factorial(n) if n <= 20 else float('inf')
    }

# n = 10
# log n ≈ 3.3, n = 10, n log n ≈ 33, n² = 100, n³ = 1000, 2^n ≈ 1024, n! ≈ 3.6M

# n = 20
# log n ≈ 4.3, n = 20, n log n ≈ 86, n² = 400, n³ = 8000, 2^n ≈ 1M, n! ≈ 2.4×10^18
```

## 실무 적용

### 1. 시간 제한과 입력 크기 추정

```python
def estimate_complexity(n, time_limit_seconds=1):
    """
    1초에 약 10^8 ~ 10^9 연산 가능 가정

    허용되는 복잡도 추정:
    """
    operations_per_second = 10 ** 8

    complexities = {
        'O(n!)': math.factorial(min(n, 13)) if n <= 13 else float('inf'),
        'O(2^n)': 2 ** n if n <= 27 else float('inf'),
        'O(n³)': n ** 3,
        'O(n²)': n ** 2,
        'O(n log n)': n * math.log2(n) if n > 0 else 0,
        'O(n)': n,
        'O(log n)': math.log2(n) if n > 0 else 0,
        'O(1)': 1
    }

    feasible = []
    for name, ops in complexities.items():
        if ops <= operations_per_second * time_limit_seconds:
            feasible.append(name)

    return feasible

# 입력 크기별 가능한 복잡도
# n = 10: O(n!), O(2^n), O(n³), O(n²), O(n log n), O(n), O(log n), O(1)
# n = 20: O(2^n), O(n³), O(n²), O(n log n), O(n), O(log n), O(1)
# n = 100: O(n³), O(n²), O(n log n), O(n), O(log n), O(1)
# n = 1000: O(n²), O(n log n), O(n), O(log n), O(1)
# n = 10^6: O(n log n), O(n), O(log n), O(1)
# n = 10^9: O(n), O(log n), O(1)
```

### 2. 상수 계수의 중요성

```python
def constant_factor_matters():
    """
    점근적으로 느린 알고리즘이 실제로 더 빠를 수 있음
    """
    # 예: n = 1000

    # Algorithm A: T(n) = 1000n (O(n))
    # Algorithm B: T(n) = n² (O(n²))

    n = 1000
    algo_a = 1000 * n    # 1,000,000
    algo_b = n * n       # 1,000,000

    # n = 1000일 때 같은 시간!
    # n < 1000이면 B가 더 빠름
    # n > 1000이면 A가 더 빠름

    # 실제 예: Insertion Sort vs Merge Sort
    # Insertion Sort: ~c₁n² (c₁ 작음, 캐시 효율 좋음)
    # Merge Sort: ~c₂n log n (c₂ 큼, 추가 메모리)
    # 작은 n에서는 Insertion Sort가 더 빠름
    # → Timsort는 작은 부분에 Insertion Sort 사용
```

### 3. 시간 측정과 분석

```python
import time
import random

def measure_complexity(func, sizes, num_trials=5):
    """알고리즘의 실제 시간 복잡도 추정"""
    results = []

    for n in sizes:
        times = []
        for _ in range(num_trials):
            data = [random.randint(0, n) for _ in range(n)]

            start = time.perf_counter()
            func(data.copy())
            end = time.perf_counter()

            times.append(end - start)

        avg_time = sum(times) / len(times)
        results.append((n, avg_time))

    # 복잡도 추정: T(n)/T(n/2) 비율 분석
    for i in range(1, len(results)):
        n1, t1 = results[i-1]
        n2, t2 = results[i]
        ratio = t2 / t1 if t1 > 0 else 0

        # 비율로 복잡도 추정
        # O(n): ratio ≈ 2
        # O(n log n): ratio ≈ 2.x
        # O(n²): ratio ≈ 4
        print(f"n: {n1} → {n2}, time ratio: {ratio:.2f}")

    return results
```

## 참고 자료

- Cormen, Leiserson, Rivest, Stein - "Introduction to Algorithms" (CLRS) Chapter 2-3
- Kleinberg, Tardos - "Algorithm Design" Chapter 2
- Knuth - "The Art of Computer Programming" Vol.1
- MIT 6.006 Introduction to Algorithms - Lecture 1-2
