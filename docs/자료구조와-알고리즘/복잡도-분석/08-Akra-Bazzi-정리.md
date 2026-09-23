# Akra Bazzi 정리

부분 문제를 n/3과 2n/3처럼 서로 다른 크기로 나누면 기본 마스터 정리의 형태에 맞지 않는다. Akra-Bazzi 정리(1998)는 이런 비균등 분할을 포함하도록 분할 정복 점화식을 일반화한다.

## Akra-Bazzi 점화식의 형태

- T(n) = g(n) + Σ(i=1 to k) aᵢT(bᵢn + hᵢ(n))
- where:
- aᵢ > 0: 각 부분 문제의 계수
- 0 < bᵢ < 1: 각 부분 문제의 축소 비율
- -, hᵢ(n), = O(n / log²n): 반올림/내림 오차
- g(n): 분할/병합 비용, n^c보다 느리게 증가하지 않음 (polynomial growth)

마스터 정리의 `aT(n/b)`는 크기가 같은 부분 문제 a개를 나타낸다. 반면 Akra-Bazzi의 `Σ aᵢT(bᵢn)`은 각 항에 다른 계수 aᵢ와 축소 비율 bᵢ를 사용할 수 있다.

## Akra-Bazzi 공식

먼저 `Σ(i=1 to k) aᵢbᵢᵖ = 1`을 만족하는 지수 p를 구한다. 이 p를 critical exponent라고 하며, 여기에 비재귀 비용 g(n)의 기여를 적분으로 더한다.

`T(n) = Θ(nᵖ(1 + ∫₁ⁿ g(u)/u^(p+1) du))`

## 적용 예제

- 예제 1: 비균등 이진 분할
- T(n) = T(n/3) + T(2n/3) + n
- 식별:
- k = 2
- a₁ = 1, b₁ = 1/3
- a₂ = 1, b₂ = 2/3
- g(n) = n
- p 찾기:
- (1/3)ᵖ + (2/3)ᵖ = 1
- 수치 해석으로 p ≈ 1을 확인:
- (1/3)¹ + (2/3)¹ = 1/3 + 2/3 = 1 가능
- → p = 1
- 적분 계산:
- ∫₁ⁿ u/u^(1+1) du = ∫₁ⁿ 1/u du = ln n
- T(n) = Θ(n¹(1 + ln n))
- = Θ(n log n)

- 예제 2: 3-way 분할
- T(n) = T(n/4) + T(n/2) + T(3n/4) + n
- p 찾기:
- (1/4)ᵖ + (1/2)ᵖ + (3/4)ᵖ = 1
- p = 1 확인:
- 1/4 + 1/2 + 3/4 = 6/4 = 1.5 ≠ 1
- p = 2 확인:
- 1/16 + 1/4 + 9/16 = (1 + 4 + 9)/16 = 14/16 ≠ 1
- 수치 해석으로 p ≈ 1.42를 찾음
- g(n) = n이므로:
- ∫₁ⁿ u/u^(p+1) du = ∫₁ⁿ u^(-p) du
- = [u^(1-p)/(1-p)]₁ⁿ
- = (n^(1-p) - 1)/(1-p)
- = O(1) (∵ 1-p < 0)
- T(n) = Θ(n^1.42)

- 예제 3: 마스터 정리로 못 푸는 경우
- T(n) = 2T(n/2) + n/log n
- 마스터 정리: f(n) = n/log n은 어떤 케이스에도 해당 안됨
- (n^(1-ε)과 n 사이의 "갭"에 위치)
- Akra-Bazzi 적용:
- a₁ = 2, b₁ = 1/2
- g(n) = n/log n
- p 찾기:
- 2 × (1/2)ᵖ = 1
- (1/2)^(p-1) = 1
- p = 1
- 적분:
- ∫₁ⁿ (u/log u)/u² du = ∫₁ⁿ 1/(u log u) du
- = [ln(log u)]₁ⁿ
- = ln(log n) - ln(log 1), (log 1 = 0이라 특별 처리 필요)
- 실제로: ∫₂ⁿ 1/(u log u) du = ln(ln n) - ln(ln 2) = Θ(log log n)
- T(n) = Θ(n(1 + log log n))
- = Θ(n log log n)

## p 값 찾기

계수와 비율만으로 p를 바로 구하기 어렵다면 방정식의 근을 수치적으로 찾을 수 있다. 다음 코드는 `fsolve`를 사용하는 예다.

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

같은 방정식에 이분 탐색을 적용하는 예제는 다음과 같다. 탐색 구간과 부호에 대한 가정을 코드 주석과 함께 확인해야 한다.

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

## 적분 계산

- 일반적인 적분 패턴:
- g(n) = nᶜ일 때:
- ∫₁ⁿ uᶜ/u^(p+1) du = ∫₁ⁿ u^(c-p-1) du
- Case 1: c - p - 1 ≠ -1
- = [u^(c-p)/(c-p)]₁ⁿ
- if c > p: = Θ(n^(c-p))
- if c < p: = Θ(1)
- Case 2: c - p - 1 = -1, 즉 c = p
- = [ln u]₁ⁿ = ln n = Θ(log n)
- 따라서:
- g(n) = n^c, c < p, → T(n) = Θ(nᵖ)
- g(n) = n^p, → T(n) = Θ(nᵖ log n)
- g(n) = n^c, c > p, → T(n) = Θ(nᶜ)

g(n)이 거듭제곱 형태인 이 경우에는 앞서 본 마스터 정리의 세 경우와 같은 구분이 나온다.

## 반올림/내림 처리

- T(n) = T(⌊n/3⌋) + T(⌈2n/3⌉) + n
- hᵢ(n) 항이 있어도:
- hᵢ(n), = O(n / log²n)
- 조건을 만족하면 점근적 결과에 영향 없음
- 증명 아이디어:
- 반올림 오차는 최대 O(1)
- log²n으로 나눈 것은 충분히 작음
- 적분에서 무시할 수 있는 항이 됨

## 증명 (스케치)

### Akra-Bazzi 정리의 증명 아이디어

- 변환: T(n)을 적분 형태로 표현
- 특성 방정식 Σ aᵢbᵢᵖ = 1의 의미:
- 재귀 트리에서 각 레벨의 가중치 합
- p = 1이면 각 레벨 비용이 보존됨
- 적분 항 ∫ g(u)/u^(p+1) du:
- 비균질 항 g(n)의 기여를 누적
- nᵖ 항:
- 동차 해의 기여 (리프 노드 비용)

## 참고 자료

- Akra, Bazzi - "On the Solution of Linear Recurrence Equations" (1998)
- Leighton - "Notes on Better Master Theorems for Divide-and-Conquer Recurrences"
- CLRS Chapter 4 Problems (4-6: Akra-Bazzi)
- MIT 6.046J - Advanced Recurrence Analysis

## 관련 학습

- [마스터 정리](07-마스터-정리.md)
- [확률적 분석](09-확률적-분석.md)
