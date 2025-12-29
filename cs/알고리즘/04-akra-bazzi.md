# Akra-Bazzi 정리 (Akra-Bazzi Theorem)

## 개요

**Akra-Bazzi 정리**(1998)는 마스터 정리를 일반화한 것으로, 비균등 분할과 다양한 형태의 분할 정복 점화식을 해결할 수 있다. 마스터 정리가 적용되지 않는 경우에도 사용 가능하다.

## 핵심 개념

### 1. Akra-Bazzi 점화식의 형태

```
T(n) = g(n) + Σ(i=1 to k) aᵢT(bᵢn + hᵢ(n))

where:
- aᵢ > 0: 각 부분 문제의 계수
- 0 < bᵢ < 1: 각 부분 문제의 축소 비율
- |hᵢ(n)| = O(n / log²n): 반올림/내림 오차
- g(n): 분할/병합 비용, n^c보다 느리게 증가하지 않음 (polynomial growth)
```

**마스터 정리와의 비교**:
```
마스터 정리: T(n) = aT(n/b) + f(n)
- 모든 부분 문제 크기 동일 (n/b)
- 부분 문제 개수 동일 (a개)

Akra-Bazzi: T(n) = g(n) + Σ aᵢT(bᵢn)
- 부분 문제 크기가 다를 수 있음 (b₁n, b₂n, ...)
- 각 부분 문제에 다른 계수 (a₁, a₂, ...)
```

### 2. Akra-Bazzi 공식

**핵심 방정식**:
```
p를 찾아라: Σ(i=1 to k) aᵢbᵢᵖ = 1

이 p를 "critical exponent"라 함
```

**해의 공식**:
```
T(n) = Θ(nᵖ(1 + ∫₁ⁿ g(u)/u^(p+1) du))
```

### 3. 적용 예제

**예제 1: 비균등 이진 분할**
```
T(n) = T(n/3) + T(2n/3) + n

식별:
- k = 2
- a₁ = 1, b₁ = 1/3
- a₂ = 1, b₂ = 2/3
- g(n) = n

p 찾기:
(1/3)ᵖ + (2/3)ᵖ = 1

수치 해석으로 p ≈ 1을 확인:
(1/3)¹ + (2/3)¹ = 1/3 + 2/3 = 1 ✓

→ p = 1

적분 계산:
∫₁ⁿ u/u^(1+1) du = ∫₁ⁿ 1/u du = ln n

T(n) = Θ(n¹(1 + ln n))
     = Θ(n log n)
```

**예제 2: 3-way 분할**
```
T(n) = T(n/4) + T(n/2) + T(3n/4) + n

p 찾기:
(1/4)ᵖ + (1/2)ᵖ + (3/4)ᵖ = 1

p = 1 확인:
1/4 + 1/2 + 3/4 = 6/4 = 1.5 ≠ 1

p = 2 확인:
1/16 + 1/4 + 9/16 = (1 + 4 + 9)/16 = 14/16 ≠ 1

수치 해석으로 p ≈ 1.42를 찾음

g(n) = n이므로:
∫₁ⁿ u/u^(p+1) du = ∫₁ⁿ u^(-p) du
= [u^(1-p)/(1-p)]₁ⁿ
= (n^(1-p) - 1)/(1-p)
= O(1) (∵ 1-p < 0)

T(n) = Θ(n^1.42)
```

**예제 3: 마스터 정리로 못 푸는 경우**
```
T(n) = 2T(n/2) + n/log n

마스터 정리: f(n) = n/log n은 어떤 케이스에도 해당 안됨
(n^(1-ε)과 n 사이의 "갭"에 위치)

Akra-Bazzi 적용:
- a₁ = 2, b₁ = 1/2
- g(n) = n/log n

p 찾기:
2 × (1/2)ᵖ = 1
(1/2)^(p-1) = 1
p = 1

적분:
∫₁ⁿ (u/log u)/u² du = ∫₁ⁿ 1/(u log u) du
= [ln(log u)]₁ⁿ
= ln(log n) - ln(log 1)  (log 1 = 0이라 특별 처리 필요)

실제로: ∫₂ⁿ 1/(u log u) du = ln(ln n) - ln(ln 2) = Θ(log log n)

T(n) = Θ(n(1 + log log n))
     = Θ(n log log n)
```

### 4. p 값 찾기

**수치적 방법**:
```python
from scipy.optimize import fsolve
import numpy as np

def find_critical_exponent(coefficients, ratios):
    """
    Σ aᵢ × bᵢᵖ = 1을 만족하는 p를 찾음

    Parameters:
    - coefficients: [a₁, a₂, ..., aₖ]
    - ratios: [b₁, b₂, ..., bₖ]
    """
    def equation(p):
        return sum(a * (b ** p) for a, b in zip(coefficients, ratios)) - 1

    # 초기 추정값 1로 시작
    p_solution = fsolve(equation, 1.0)[0]
    return p_solution

# 예제 1: T(n) = T(n/3) + T(2n/3) + n
p1 = find_critical_exponent([1, 1], [1/3, 2/3])
print(f"T(n/3) + T(2n/3): p = {p1}")  # p ≈ 1

# 예제 2: T(n) = T(n/4) + T(n/2) + T(3n/4) + n
p2 = find_critical_exponent([1, 1, 1], [1/4, 1/2, 3/4])
print(f"T(n/4) + T(n/2) + T(3n/4): p = {p2}")  # p ≈ 1.42

# 예제 3: T(n) = 2T(n/2) + n (마스터 정리 확인)
p3 = find_critical_exponent([2], [1/2])
print(f"2T(n/2): p = {p3}")  # p = 1 (마스터 정리 Case 2와 일치)
```

**이분 탐색 방법**:
```python
def find_p_bisection(coefficients, ratios, tol=1e-10):
    """이분 탐색으로 p 찾기"""
    def f(p):
        return sum(a * (b ** p) for a, b in zip(coefficients, ratios)) - 1

    # f(0) > 0 (모든 aᵢ > 0이므로 Σaᵢ > 1)
    # f(∞) < 0 (모든 bᵢ < 1이므로 bᵢᵖ → 0)
    # 따라서 해가 존재

    lo, hi = 0, 100
    while hi - lo > tol:
        mid = (lo + hi) / 2
        if f(mid) > 0:
            lo = mid
        else:
            hi = mid
    return (lo + hi) / 2
```

### 5. 적분 계산

**일반적인 적분 패턴**:
```
g(n) = nᶜ일 때:

∫₁ⁿ uᶜ/u^(p+1) du = ∫₁ⁿ u^(c-p-1) du

Case 1: c - p - 1 ≠ -1
= [u^(c-p)/(c-p)]₁ⁿ

  if c > p: = Θ(n^(c-p))
  if c < p: = Θ(1)

Case 2: c - p - 1 = -1, 즉 c = p
= [ln u]₁ⁿ = ln n = Θ(log n)

따라서:
- g(n) = n^c, c < p  → T(n) = Θ(nᵖ)
- g(n) = n^p         → T(n) = Θ(nᵖ log n)
- g(n) = n^c, c > p  → T(n) = Θ(nᶜ)
```

이는 마스터 정리의 세 케이스와 정확히 일치한다!

### 6. 반올림/내림 처리

```
T(n) = T(⌊n/3⌋) + T(⌈2n/3⌉) + n

hᵢ(n) 항이 있어도:
|hᵢ(n)| = O(n / log²n)

조건을 만족하면 점근적 결과에 영향 없음

증명 아이디어:
- 반올림 오차는 최대 O(1)
- log²n으로 나눈 것은 충분히 작음
- 적분에서 무시할 수 있는 항이 됨
```

## 실무 적용

### 1. 복잡한 분할 정복 분석

```python
def analyze_divide_conquer(description):
    """
    분할 정복 알고리즘의 복잡도 분석
    """
    pass

# 예: Quicksort의 최악 케이스와 비슷한 분할
# T(n) = T(n-1) + T(0) + n
# 이것은 Akra-Bazzi 형태가 아님 (b = (n-1)/n ≈ 1로 수렴 안함)
# → 직접 전개: T(n) = Θ(n²)

# 예: Select 알고리즘 (Median of Medians)
# T(n) = T(n/5) + T(7n/10) + n
# a₁ = 1, b₁ = 1/5
# a₂ = 1, b₂ = 7/10
# (1/5)ᵖ + (7/10)ᵖ = 1
# p ≈ 0.84
# → T(n) = Θ(n^0.84)? 아니, g(n) = n이 지배
# 실제로: T(n) = Θ(n)
```

### 2. Akra-Bazzi vs Master Theorem

```python
def when_to_use_akra_bazzi():
    """
    Akra-Bazzi를 사용해야 하는 경우

    1. 비균등 분할
       T(n) = T(αn) + T((1-α)n) + f(n)

    2. 다중 분할
       T(n) = Σ aᵢT(bᵢn) + f(n)

    3. 마스터 정리의 "갭"에 빠지는 경우
       f(n)이 n^(log_b a ± ε)에 해당하지 않을 때

    4. 다른 계수를 가진 분할
       T(n) = 2T(n/4) + 3T(n/2) + f(n)
    """
    examples = [
        ("T(n) = T(n/3) + T(2n/3) + n", "비균등 분할"),
        ("T(n) = 2T(n/2) + n/log n", "마스터 갭"),
        ("T(n) = T(n/2) + T(n/4) + T(n/8) + n", "다중 분할"),
    ]
    return examples
```

### 3. 알고리즘 설계에의 응용

```python
def design_optimal_divide_conquer():
    """
    분할 비율 선택이 복잡도에 미치는 영향

    T(n) = T(αn) + T((1-α)n) + n

    p를 찾으면 α^p + (1-α)^p = 1

    α = 1/2일 때: (1/2)^p + (1/2)^p = 1 → p = 1
    → T(n) = Θ(n log n)

    α = 1/3일 때: (1/3)^p + (2/3)^p = 1 → p = 1
    → T(n) = Θ(n log n)

    결론: 분할 비율과 관계없이 T(n) = Θ(n log n)!
    (단, g(n) = n일 때)

    이유: 두 부분의 합이 n이면, 각 레벨의 총 비용이 n
    """
    pass
```

## 증명 (스케치)

### Akra-Bazzi 정리의 증명 아이디어

```
1. 변환: T(n)을 적분 형태로 표현

2. 특성 방정식 Σ aᵢbᵢᵖ = 1의 의미:
   - 재귀 트리에서 각 레벨의 가중치 합
   - p = 1이면 각 레벨 비용이 보존됨

3. 적분 항 ∫ g(u)/u^(p+1) du:
   - 비균질 항 g(n)의 기여를 누적

4. nᵖ 항:
   - 동차 해의 기여 (리프 노드 비용)
```

## 참고 자료

- Akra, Bazzi - "On the Solution of Linear Recurrence Equations" (1998)
- Leighton - "Notes on Better Master Theorems for Divide-and-Conquer Recurrences"
- CLRS Chapter 4 Problems (4-6: Akra-Bazzi)
- MIT 6.046J - Advanced Recurrence Analysis
