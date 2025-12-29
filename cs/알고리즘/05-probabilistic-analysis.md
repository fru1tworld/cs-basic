# 확률적 분석 (Probabilistic Analysis)

## 개요

**확률적 분석**(Probabilistic Analysis)은 입력에 대한 확률 분포를 가정하여 알고리즘의 **평균적인 성능**을 분석하는 기법이다. 최악 케이스 분석과 달리, 실제 운용 환경에서의 성능을 더 정확하게 예측할 수 있다.

## 핵심 개념

### 1. 기대값 (Expected Value)

**정의**:
```
이산 확률 변수 X에 대해:
E[X] = Σ x × P(X = x)

연속 확률 변수 X에 대해:
E[X] = ∫ x × f(x) dx
```

**기대값의 선형성** (Linearity of Expectation):
```
E[X + Y] = E[X] + E[Y]  (항상 성립, 독립 여부와 무관)

E[cX] = c × E[X]  (c는 상수)
```

**곱의 기대값** (독립인 경우):
```
X, Y가 독립이면:
E[X × Y] = E[X] × E[Y]
```

### 2. Indicator Random Variable (지시 확률 변수)

**정의**:
```
사건 A에 대한 지시 확률 변수:

I{A} = 1  if A occurs
       0  if A does not occur

E[I{A}] = 1 × P(A) + 0 × P(not A) = P(A)
```

**활용의 핵심**:
- 복잡한 확률 변수를 간단한 지시 변수들의 합으로 분해
- 기대값의 선형성을 적용하여 계산 단순화

### 3. 예제: Hiring Problem

```python
def hire_assistant(candidates):
    """
    n명의 후보자를 순차적으로 면접
    현재 최고보다 나은 후보가 나타나면 고용

    비용: 면접 비용 cᵢ (작음), 고용 비용 cₕ (큼)
    """
    best = -float('inf')
    hire_count = 0

    for candidate in candidates:
        # 면접 비용: cᵢ
        if candidate > best:
            best = candidate
            # 고용 비용: cₕ
            hire_count += 1

    return hire_count
```

**분석**:
```
Xᵢ = I{i번째 후보가 고용됨}

E[Xᵢ] = P(i번째 후보가 처음 i명 중 최고)

모든 순열이 동등하게 발생한다고 가정하면:
E[Xᵢ] = 1/i

총 고용 횟수 X = Σ(i=1 to n) Xᵢ

E[X] = E[Σ Xᵢ]
     = Σ E[Xᵢ]        (선형성)
     = Σ(i=1 to n) 1/i
     = Hₙ             (n번째 조화급수)
     ≈ ln n + γ       (γ ≈ 0.5772, 오일러 상수)
     = Θ(log n)

총 비용:
E[cost] = n × cᵢ + E[X] × cₕ
        = n × cᵢ + Θ(log n) × cₕ
```

### 4. 예제: Quicksort 평균 분석

```python
def quicksort(arr, lo, hi):
    if lo < hi:
        p = partition(arr, lo, hi)  # pivot 위치
        quicksort(arr, lo, p - 1)
        quicksort(arr, p + 1, hi)

def partition(arr, lo, hi):
    pivot = arr[hi]  # 마지막 원소를 pivot으로
    i = lo - 1
    for j in range(lo, hi):
        if arr[j] <= pivot:
            i += 1
            arr[i], arr[j] = arr[j], arr[i]
    arr[i + 1], arr[hi] = arr[hi], arr[i + 1]
    return i + 1
```

**지시 변수를 이용한 분석**:
```
정렬된 순서로 원소를 z₁ < z₂ < ... < zₙ라 하자

Xᵢⱼ = I{zᵢ와 zⱼ가 비교됨}  (i < j)

총 비교 횟수 X = Σ(i=1 to n-1) Σ(j=i+1 to n) Xᵢⱼ

E[X] = Σᵢ Σⱼ E[Xᵢⱼ]
     = Σᵢ Σⱼ P(zᵢ와 zⱼ가 비교됨)

핵심 관찰:
zᵢ와 zⱼ가 비교되려면, {zᵢ, zᵢ₊₁, ..., zⱼ} 중에서
zᵢ 또는 zⱼ가 먼저 pivot으로 선택되어야 함

P(zᵢ와 zⱼ 비교) = 2 / (j - i + 1)

E[X] = Σ(i=1 to n-1) Σ(j=i+1 to n) 2/(j-i+1)

k = j - i로 치환:
= Σ(i=1 to n-1) Σ(k=1 to n-i) 2/(k+1)
≤ Σ(i=1 to n-1) Σ(k=1 to n) 2/k
= (n-1) × 2Hₙ
≈ 2n ln n

따라서 E[X] = O(n log n)
```

**더 정밀한 분석**:
```
E[X] = 2(n+1)Hₙ - 4n
     ≈ 2n ln n + O(n)
     ≈ 1.39 n log₂ n

이는 Merge Sort의 n log n보다 약 39% 더 많은 비교
하지만 상수 요소(이동 횟수, 캐시 효율)로 인해 실제로는 더 빠름
```

### 5. 예제: Birthday Paradox (생일 역설)

```
n명 중 생일이 같은 쌍이 존재할 확률은?

모든 쌍 (i, j)에 대해:
Xᵢⱼ = I{i와 j의 생일이 같음}

P(Xᵢⱼ = 1) = 1/365

같은 생일 쌍의 기대 개수:
E[Σᵢ<ⱼ Xᵢⱼ] = C(n,2) × 1/365 = n(n-1)/(2 × 365)

n = 28이면: 28 × 27 / 730 ≈ 1.04

평균적으로 1쌍 이상 존재!

정확한 확률 (최소 1쌍):
P(충돌) = 1 - (364/365)(363/365)...(365-n+1)/365)
n = 23일 때 P ≈ 50.7%
n = 70일 때 P ≈ 99.9%
```

### 6. Randomized Algorithm 분석

**결정적 vs 무작위 알고리즘**:
```
결정적 알고리즘:
- 입력에 대해 항상 같은 실행 경로
- 최악 케이스 입력이 존재할 수 있음

무작위 알고리즘:
- 내부에서 난수 사용
- 어떤 입력에 대해서도 기대 성능 보장
- 적대자가 최악 케이스 입력을 만들기 어려움
```

**예제: Randomized Quicksort**:
```python
import random

def randomized_quicksort(arr, lo, hi):
    if lo < hi:
        # 무작위로 pivot 선택
        rand_idx = random.randint(lo, hi)
        arr[rand_idx], arr[hi] = arr[hi], arr[rand_idx]

        p = partition(arr, lo, hi)
        randomized_quicksort(arr, lo, p - 1)
        randomized_quicksort(arr, p + 1, hi)
```

```
분석:
- 모든 입력에 대해 E[비교 횟수] = O(n log n)
- 최악 케이스 입력이라는 개념이 없음
- 단, 운이 나쁘면 O(n²)이 될 수 있음 (확률 매우 낮음)

정리: 임의의 입력에 대해
P(T(n) > c × n log n) ≤ 1/n^α for some α > 0
(고확률 경계, High Probability Bound)
```

### 7. 분산과 집중 부등식

**분산 (Variance)**:
```
Var[X] = E[(X - E[X])²]
       = E[X²] - (E[X])²

표준편차: σ = √Var[X]
```

**Markov 부등식**:
```
X ≥ 0인 확률 변수에 대해:
P(X ≥ a) ≤ E[X] / a

예: 기대값이 10이면, X ≥ 100일 확률은 최대 10%
```

**Chebyshev 부등식**:
```
P(|X - E[X]| ≥ k × σ) ≤ 1/k²

예: 평균에서 2σ 벗어날 확률 ≤ 25%
    평균에서 3σ 벗어날 확률 ≤ 11%
```

**Chernoff 경계**:
```
독립적인 Bernoulli 시행 X₁, ..., Xₙ에 대해:
X = Σ Xᵢ, μ = E[X]

P(X ≥ (1+δ)μ) ≤ exp(-δ²μ/3)  for 0 < δ < 1
P(X ≤ (1-δ)μ) ≤ exp(-δ²μ/2)  for 0 < δ < 1

훨씬 더 강한 "꼬리 경계" (tail bound)
```

## 실무 적용

### 1. 평균 케이스 분석 프레임워크

```python
def average_case_analysis(algorithm_description):
    """
    평균 케이스 분석 단계:

    1. 입력 분포 가정 (보통 균등 분포)
    2. 비용 함수 정의 (비교 횟수, 이동 횟수 등)
    3. 지시 변수로 분해
    4. 각 지시 변수의 기대값 계산
    5. 선형성으로 합산
    """
    pass

# 예: 선형 탐색 평균 분석
def linear_search_average_analysis(n):
    """
    가정: target이 배열에 있고, 균등 분포

    Xᵢ = I{i번째에서 발견}
    P(Xᵢ = 1) = 1/n

    비용 = i (i번째에서 발견 시)

    E[비용] = Σ(i=1 to n) i × (1/n)
            = (1/n) × n(n+1)/2
            = (n+1)/2
    """
    return (n + 1) / 2
```

### 2. 실험적 검증

```python
import random
import time
from collections import defaultdict

def empirical_average_complexity(func, n, trials=1000):
    """알고리즘의 평균 복잡도를 실험적으로 측정"""
    total_comparisons = 0

    for _ in range(trials):
        # 무작위 입력 생성
        arr = list(range(n))
        random.shuffle(arr)

        comparisons = func(arr.copy())
        total_comparisons += comparisons

    avg = total_comparisons / trials
    theoretical = 2 * n * (math.log(n) if n > 0 else 0)  # Quicksort 이론값

    print(f"n={n}: 측정 평균={avg:.1f}, 이론값≈{theoretical:.1f}")
    return avg

def quicksort_count_comparisons(arr):
    """비교 횟수를 세는 Quicksort"""
    comparisons = [0]

    def sort(lo, hi):
        if lo >= hi:
            return

        # Random pivot
        rand_idx = random.randint(lo, hi)
        arr[rand_idx], arr[hi] = arr[hi], arr[rand_idx]

        pivot = arr[hi]
        i = lo - 1

        for j in range(lo, hi):
            comparisons[0] += 1
            if arr[j] <= pivot:
                i += 1
                arr[i], arr[j] = arr[j], arr[i]

        arr[i + 1], arr[hi] = arr[hi], arr[i + 1]

        sort(lo, i)
        sort(i + 2, hi)

    sort(0, len(arr) - 1)
    return comparisons[0]
```

### 3. 해시 테이블 분석

```python
def hash_table_analysis():
    """
    Simple Uniform Hashing Assumption (SUHA):
    - 각 키가 m개의 슬롯에 균등하게 해싱됨

    n개 키, m개 슬롯일 때:
    Load factor α = n/m

    체이닝에서 평균 체인 길이:
    E[체인 길이] = α

    성공적 탐색의 평균 비교 횟수:
    E = 1 + α/2 ≈ 1 + n/(2m)

    실패한 탐색의 평균 비교 횟수:
    E = α ≈ n/m

    α = O(1)이면 (m = Θ(n)):
    탐색, 삽입, 삭제 모두 O(1) 평균
    """
    pass
```

## 고급 주제

### 1. Las Vegas vs Monte Carlo 알고리즘

```
Las Vegas:
- 항상 정확한 결과 반환
- 실행 시간이 확률적
- 예: Randomized Quicksort

Monte Carlo:
- 실행 시간은 결정적
- 결과가 확률적으로 정확
- 예: Miller-Rabin 소수 판정
```

### 2. Yao's Minimax Principle

```
정리 (Yao, 1977):
결정적 알고리즘의 최악 케이스 기대 성능
≥ 무작위 알고리즘의 최악 입력에 대한 기대 성능

응용:
- 무작위 알고리즘의 하한 증명
- 입력 분포에 대해 결정적 알고리즘을 분석하여 증명
```

## 참고 자료

- CLRS Chapter 5: Probabilistic Analysis and Randomized Algorithms
- Motwani, Raghavan - "Randomized Algorithms"
- Mitzenmacher, Upfal - "Probability and Computing"
- MIT 6.046J - Lecture on Randomized Algorithms
