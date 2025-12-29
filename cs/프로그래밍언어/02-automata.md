# 오토마타 이론 (Automata Theory)

## 개요

오토마타(Automata)는 입력을 받아 상태를 전이하며 특정 언어를 인식하는 추상적인 기계 모델이다. 형식 언어 이론의 핵심으로, 컴파일러의 어휘 분석기(Lexer), 정규 표현식 엔진, 프로토콜 검증 등 다양한 분야에서 활용된다.

오토마타 이론의 핵심 질문은 "주어진 문자열이 특정 언어에 속하는지 어떻게 판단하는가?"이다.

## 핵심 개념

### 유한 오토마타 (Finite Automata, FA)

유한 오토마타는 유한한 상태 집합과 상태 전이 규칙으로 구성되며, 정규 언어를 인식한다.

#### DFA (Deterministic Finite Automaton)

**정의**: 5-튜플 M = (Q, Σ, δ, q₀, F)
- Q: 유한한 상태 집합
- Σ: 입력 알파벳
- δ: Q × Σ → Q (전이 함수, 결정적)
- q₀: 시작 상태 (q₀ ∈ Q)
- F: 수락 상태 집합 (F ⊆ Q)

**특징**:
- 각 상태에서 각 입력 기호에 대해 **정확히 하나의** 전이만 존재
- 구현이 단순하고 실행 시간이 O(n) (n = 입력 길이)

```
예시: 짝수 개의 0을 포함하는 이진 문자열

상태: {q_even, q_odd}
알파벳: {0, 1}
시작 상태: q_even
수락 상태: {q_even}

전이 함수:
δ(q_even, 0) = q_odd
δ(q_even, 1) = q_even
δ(q_odd, 0) = q_even
δ(q_odd, 1) = q_odd

상태 전이 다이어그램:
        1                  1
    ┌───────┐          ┌───────┐
    ▼       │          ▼       │
  ┌───────────┐   0  ┌───────────┐
→ │  q_even   │ ───→ │   q_odd   │
  │   (수락)   │ ←─── │           │
  └───────────┘   0  └───────────┘

입력 "1001" 처리:
q_even →(1)→ q_even →(0)→ q_odd →(0)→ q_even →(1)→ q_even (수락!)
```

#### NFA (Nondeterministic Finite Automaton)

**정의**: 5-튜플 M = (Q, Σ, δ, q₀, F)
- δ: Q × Σ → P(Q) (전이 함수, 상태의 **집합** 반환)

**특징**:
- 각 상태에서 각 입력에 대해 **0개 이상**의 전이 가능
- 여러 경로 중 하나라도 수락 상태에 도달하면 수락
- DFA보다 설계가 직관적이지만 실행은 시뮬레이션 필요

```
예시: "ab"로 끝나는 문자열

상태: {q0, q1, q2}
알파벳: {a, b}
시작 상태: q0
수락 상태: {q2}

전이 함수:
δ(q0, a) = {q0, q1}  // 비결정적: 두 상태로 전이 가능
δ(q0, b) = {q0}
δ(q1, a) = ∅
δ(q1, b) = {q2}
δ(q2, a) = ∅
δ(q2, b) = ∅

상태 전이 다이어그램:
           a, b
         ┌─────┐
         ▼     │
       ┌───┐  a  ┌───┐  b  ┌─────┐
    →  │ q0│ ──→ │ q1│ ──→ │ q2  │
       └───┘     └───┘     │(수락)│
                           └─────┘

입력 "aab" 처리 (모든 경로 추적):
q0 →(a)→ {q0, q1}
  q0 →(a)→ {q0, q1}
    q0 →(b)→ {q0} (수락 X)
    q1 →(b)→ {q2} (수락 O!)
  q1 →(a)→ ∅ (죽은 경로)
```

#### ε-NFA (Epsilon-NFA)

**정의**: 입력 없이 상태 전이 가능한 NFA
- δ: Q × (Σ ∪ {ε}) → P(Q)

**특징**:
- ε-전이: 입력을 소비하지 않고 상태 이동
- 정규 표현식을 NFA로 변환할 때 유용

```
예시: (a|b)*c를 인식하는 ε-NFA

       ε           ε
    ┌─────→ q1 ─a→ q2 ─────┐
    │              ε       │
    │       ┌──────────────┤
    ▼       ▼              │
→ q0 ──ε─→ q3 ─b→ q4 ──ε──┘
    │       ▲
    │       └──────────────┐
    │              ε       │
    └─────→ q5 ─c→ (q6)────┘
              수락

ε-클로저(q0) = {q0, q1, q3, q5}
```

### NFA → DFA 변환 (Subset Construction)

모든 NFA는 동등한 DFA로 변환 가능하다. 이 변환을 **부분집합 구성법(Subset Construction)** 또는 **멱집합 구성법(Powerset Construction)**이라 한다.

**알고리즘**:
1. DFA의 각 상태는 NFA 상태들의 **집합**
2. DFA 시작 상태 = ε-closure(NFA 시작 상태)
3. 각 DFA 상태와 각 입력 기호에 대해 전이 계산
4. NFA 수락 상태를 포함하는 DFA 상태는 수락 상태

```
예시: "ab"로 끝나는 문자열 NFA를 DFA로 변환

NFA 전이:
δ(q0, a) = {q0, q1}
δ(q0, b) = {q0}
δ(q1, b) = {q2}

DFA 상태 = NFA 상태의 부분집합
{q0}         // 시작 상태
{q0, q1}     // q0에서 a 입력
{q0, q2}     // {q0,q1}에서 b 입력 = δ(q0,b) ∪ δ(q1,b) = {q0} ∪ {q2}

DFA 전이 테이블:
| DFA 상태   | a        | b        |
|------------|----------|----------|
| {q0}       | {q0,q1}  | {q0}     |
| {q0,q1}    | {q0,q1}  | {q0,q2}  |
| {q0,q2}    | {q0,q1}  | {q0}     |

수락 상태: {q0,q2} (q2 포함)

상태 수: NFA는 3개, DFA는 3개
최악의 경우 DFA 상태 수는 2ⁿ (n = NFA 상태 수)
```

### DFA 최소화

동등한 상태를 합쳐 최소 상태 수의 DFA를 만드는 과정.

**Hopcroft's Algorithm** (O(n log n)):
1. 상태를 수락/비수락 두 그룹으로 분할
2. 같은 그룹 내에서 다른 그룹으로 전이하는 상태가 있으면 분리
3. 더 이상 분리할 수 없을 때까지 반복

```
예시:
원본 DFA (5개 상태):
q0 ─a→ q1 ─b→ q2 (수락)
q0 ─b→ q3 ─a→ q4 ─b→ q2

분할 과정:
초기: {q0, q1, q3, q4}, {q2}
q1과 q4는 b로 수락 상태로 가므로 분리 필요
결과: {q0, q3}, {q1, q4}, {q2}

q0과 q3: a 입력 시 둘 다 {q1,q4} 그룹으로
         b 입력 시 q0→{q1,q4}, q3→{q0,q3} (다름!)
분리: {q0}, {q3}, {q1, q4}, {q2}

최소화된 DFA (4개 상태)
```

### 정규표현식과 FA의 관계

정규 표현식과 유한 오토마타는 동치이다 (같은 언어 클래스를 표현).

```
┌───────────────────┐         Thompson's         ┌───────────┐
│   정규 표현식      │ ─────────────────────────→ │    NFA    │
│   (a|b)*abb       │      Construction          │           │
└───────────────────┘                            └───────────┘
        ▲                                              │
        │                                              │ Subset
        │ State Elimination                            │ Construction
        │                                              ▼
┌───────────────────┐                            ┌───────────┐
│       DFA         │ ←────────────────────────  │    DFA    │
│   (최소화됨)       │        Minimization        │ (최소화전) │
└───────────────────┘                            └───────────┘
```

#### Thompson's Construction (RE → NFA)

정규 표현식의 각 연산에 대해 ε-NFA를 구성:

```
기본 케이스:
ε:  → (q0) ─ε→ ((q1))

a:  → (q0) ─a→ ((q1))

연결 (AB):
→ (s1) ──A──→ (f1) ─ε→ (s2) ──B──→ ((f2))

선택 (A|B):
         ε    ┌──A──→ (f1)─┐ ε
    → (s)  ───┤            ├───→ ((f))
         ε    └──B──→ (f2)─┘ ε

클로저 (A*):
              ε
           ┌────────────────┐
           │                ▼
    → (s) ─┴─ε→ (s1) ─A→ (f1) ─ε→ ((f))
           │                      ▲
           └──────────ε───────────┘
```

**예시: (a|b)*abb의 NFA**

```
Thompson's Construction 적용:

1. a를 위한 NFA:  q1 ─a→ q2
2. b를 위한 NFA:  q3 ─b→ q4
3. (a|b):        q0 ─ε→ q1 ─a→ q2 ─ε→ q5
                 q0 ─ε→ q3 ─b→ q4 ─ε→ q5
4. (a|b)*:       q6 ─ε→ q0 ... q5 ─ε→ q7
                 q6 ─ε→ q7
                 q5 ─ε→ q0 (루프)
5. abb 추가:     ... ─ε→ q8 ─a→ q9 ─b→ q10 ─b→ q11 (수락)
```

### Pushdown Automata (PDA)

스택을 가진 유한 오토마타로, 문맥 자유 언어를 인식한다.

**정의**: 7-튜플 M = (Q, Σ, Γ, δ, q₀, Z₀, F)
- Q: 상태 집합
- Σ: 입력 알파벳
- Γ: 스택 알파벳
- δ: Q × (Σ ∪ {ε}) × Γ → P(Q × Γ*)
- q₀: 시작 상태
- Z₀: 초기 스택 기호
- F: 수락 상태 집합

```
예시: L = {aⁿbⁿ | n ≥ 0}을 인식하는 PDA

상태: {q0, q1, q2}
입력 알파벳: {a, b}
스택 알파벳: {A, Z}  // Z는 스택 바닥 표시
시작: q0, 스택에 Z

전이:
δ(q0, a, Z) = {(q0, AZ)}    // a 읽고 A를 push
δ(q0, a, A) = {(q0, AA)}    // a 읽고 A를 push
δ(q0, b, A) = {(q1, ε)}     // b 읽고 A를 pop
δ(q1, b, A) = {(q1, ε)}     // b 읽고 A를 pop
δ(q1, ε, Z) = {(q2, Z)}     // 스택 비면 수락

입력 "aabb" 처리:
(q0, aabb, Z) ⊢ (q0, abb, AZ) ⊢ (q0, bb, AAZ)
              ⊢ (q1, b, AZ)   ⊢ (q1, ε, Z)
              ⊢ (q2, ε, Z)    수락!

스택 시각화:
초기:  [Z]
a:     [A,Z]
a:     [A,A,Z]
b:     [A,Z]     (A pop)
b:     [Z]       (A pop)
ε:     수락 상태로 전이
```

### DPDA vs NPDA

- **DPDA (Deterministic PDA)**: 각 구성에서 최대 하나의 전이. LR 파서의 기반.
- **NPDA (Nondeterministic PDA)**: 여러 전이 가능. 모든 CFL 인식 가능.

**중요**: DPDA ⊂ NPDA (DPDA가 인식하는 언어는 NPDA가 인식하는 언어의 진부분집합)

```
예시: L = {wwᴿ | w ∈ {a,b}*} (회문)
- NPDA로 인식 가능 (중간 지점을 "추측")
- DPDA로 인식 불가능 (중간 지점을 결정할 수 없음)

예시: L = {wcwᴿ | w ∈ {a,b}*} (c가 중간 표시)
- DPDA로 인식 가능 (c를 보고 중간임을 앎)
```

## 실무 적용

### 정규 표현식 엔진 구현

```java
// DFA 기반 정규 표현식 매처 (간단한 버전)
public class DFAMatcher {
    private int[][] transitions;  // [state][input] -> nextState
    private boolean[] accepting;
    private int startState;

    public boolean matches(String input) {
        int currentState = startState;

        for (char c : input.toCharArray()) {
            int inputIndex = getInputIndex(c);
            if (inputIndex < 0) return false;  // 알파벳에 없는 문자

            currentState = transitions[currentState][inputIndex];
            if (currentState < 0) return false;  // 죽은 상태
        }

        return accepting[currentState];
    }

    // O(n) 시간 복잡도 - 백트래킹 없음
}

// NFA 기반 정규 표현식 매처 (백트래킹 지원)
public class NFAMatcher {
    private Set<Integer>[][] transitions;
    private Set<Integer>[] epsilonTransitions;
    private Set<Integer> acceptingStates;

    public boolean matches(String input) {
        Set<Integer> currentStates = epsilonClosure(Set.of(0));

        for (char c : input.toCharArray()) {
            Set<Integer> nextStates = new HashSet<>();
            for (int state : currentStates) {
                Set<Integer> targets = transitions[state][getInputIndex(c)];
                if (targets != null) {
                    nextStates.addAll(targets);
                }
            }
            currentStates = epsilonClosure(nextStates);

            if (currentStates.isEmpty()) return false;
        }

        return !Collections.disjoint(currentStates, acceptingStates);
    }

    private Set<Integer> epsilonClosure(Set<Integer> states) {
        Set<Integer> closure = new HashSet<>(states);
        Queue<Integer> worklist = new LinkedList<>(states);

        while (!worklist.isEmpty()) {
            int state = worklist.poll();
            Set<Integer> epsilonTargets = epsilonTransitions[state];
            if (epsilonTargets != null) {
                for (int target : epsilonTargets) {
                    if (closure.add(target)) {
                        worklist.add(target);
                    }
                }
            }
        }

        return closure;
    }
}
```

### 어휘 분석기 (Lexer) 구현

```python
# DFA 기반 간단한 Lexer
class Token:
    def __init__(self, type, value):
        self.type = type
        self.value = value

class Lexer:
    def __init__(self):
        # 각 토큰 타입에 대한 DFA 정의
        self.token_dfas = {
            'IDENTIFIER': self.build_identifier_dfa(),
            'NUMBER': self.build_number_dfa(),
            'KEYWORD': self.build_keyword_dfa(),
        }

    def tokenize(self, input_str):
        tokens = []
        pos = 0

        while pos < len(input_str):
            if input_str[pos].isspace():
                pos += 1
                continue

            # Maximal Munch: 가장 긴 매치 찾기
            best_match = None
            best_length = 0

            for token_type, dfa in self.token_dfas.items():
                length = dfa.longest_match(input_str, pos)
                if length > best_length:
                    best_length = length
                    best_match = token_type

            if best_match:
                value = input_str[pos:pos + best_length]
                tokens.append(Token(best_match, value))
                pos += best_length
            else:
                raise SyntaxError(f"Unexpected character at position {pos}")

        return tokens

    def build_identifier_dfa(self):
        # [a-zA-Z_][a-zA-Z0-9_]*
        return DFA(
            transitions={
                0: {'letter': 1, '_': 1},
                1: {'letter': 1, 'digit': 1, '_': 1}
            },
            start=0,
            accepting={1}
        )
```

### 프로토콜 상태 기계

```java
// TCP 연결 상태 기계 (간략화)
public enum TCPState {
    CLOSED, LISTEN, SYN_SENT, SYN_RECEIVED,
    ESTABLISHED, FIN_WAIT_1, FIN_WAIT_2,
    CLOSE_WAIT, CLOSING, LAST_ACK, TIME_WAIT
}

public class TCPStateMachine {
    private TCPState currentState = TCPState.CLOSED;

    public void handleEvent(TCPEvent event) {
        TCPState nextState = getNextState(currentState, event);

        if (nextState != null) {
            performAction(currentState, event, nextState);
            currentState = nextState;
        } else {
            throw new IllegalStateException(
                "Invalid transition from " + currentState + " on " + event
            );
        }
    }

    private TCPState getNextState(TCPState state, TCPEvent event) {
        // 상태 전이 테이블
        return switch (state) {
            case CLOSED -> switch (event) {
                case PASSIVE_OPEN -> TCPState.LISTEN;
                case ACTIVE_OPEN -> TCPState.SYN_SENT;
                default -> null;
            };
            case LISTEN -> switch (event) {
                case RECV_SYN -> TCPState.SYN_RECEIVED;
                case CLOSE -> TCPState.CLOSED;
                default -> null;
            };
            case ESTABLISHED -> switch (event) {
                case CLOSE -> TCPState.FIN_WAIT_1;
                case RECV_FIN -> TCPState.CLOSE_WAIT;
                default -> null;
            };
            // ... 기타 상태 전이
            default -> null;
        };
    }
}
```

### ReDoS 취약점 이해

```python
# 백트래킹 정규 표현식의 위험성
import re
import time

# 취약한 패턴: (a+)+
vulnerable_pattern = re.compile(r'(a+)+$')

# 공격 입력: 'a' * n + 'X'
def test_redos(n):
    attack_input = 'a' * n + 'X'
    start = time.time()
    vulnerable_pattern.match(attack_input)
    elapsed = time.time() - start
    print(f"n={n}: {elapsed:.4f}s")

# n=10:  0.0001s
# n=20:  0.1s
# n=25:  3.2s
# n=30:  102s (지수적 증가!)

# 해결책: DFA 기반 엔진 사용 또는 패턴 재작성
safe_pattern = re.compile(r'a+$')  # 동일한 의미, 안전
```

## 참고 자료

- Hopcroft, Motwani, Ullman - "Introduction to Automata Theory, Languages, and Computation"
- Sipser, Michael - "Introduction to the Theory of Computation"
- Thompson, Ken - "Regular Expression Search Algorithm" (1968)
- Cox, Russ - "Regular Expression Matching Can Be Simple And Fast" (RE2 설명)
- [Visualizing NFA/DFA](https://cyberzhg.github.io/toolbox/nfa2dfa)
- [Regular Expression Denial of Service (ReDoS)](https://owasp.org/www-community/attacks/Regular_expression_Denial_of_Service_-_ReDoS)
