# 형식 언어 (Formal Languages)

## 개요

형식 언어(Formal Language)는 특정 알파벳(기호 집합)으로 구성된 문자열의 집합으로, 수학적으로 엄밀하게 정의된 언어이다. 프로그래밍 언어의 구문(Syntax)을 정의하고 분석하는 이론적 기반이 되며, 컴파일러 설계, 자연어 처리, 패턴 매칭 등 다양한 분야에서 활용된다.

형식 언어 이론의 핵심 질문은 "어떤 종류의 언어가 있으며, 각각을 인식하려면 어떤 계산 능력이 필요한가?"이다.

## 핵심 개념

### 기본 정의

```
알파벳(Alphabet) Σ: 유한한 기호들의 집합
  예: Σ = {0, 1}, Σ = {a, b, c}, Σ = {if, else, while, ...}

문자열(String): 알파벳의 기호들을 유한하게 나열한 것
  예: "0110", "abc", "if while"
  빈 문자열: ε (epsilon)

Σ* (Kleene Closure): Σ로 만들 수 있는 모든 문자열의 집합 (ε 포함)
  예: {0,1}* = {ε, 0, 1, 00, 01, 10, 11, 000, ...}

언어(Language) L: Σ*의 부분집합
  예: L = {0ⁿ1ⁿ | n ≥ 0} = {ε, 01, 0011, 000111, ...}
```

### Chomsky Hierarchy (촘스키 위계)

Noam Chomsky가 1956년에 제안한 형식 언어의 분류 체계로, 문법의 생성 규칙(Production Rules)에 대한 제약에 따라 4가지 타입으로 분류한다.

```
┌─────────────────────────────────────────────────────────┐
│                    Type 0                               │
│              Recursively Enumerable                     │
│                  (튜링 기계)                             │
│    ┌───────────────────────────────────────────────┐   │
│    │              Type 1                           │   │
│    │         Context-Sensitive                     │   │
│    │        (선형 유계 오토마타)                      │   │
│    │    ┌───────────────────────────────────┐     │   │
│    │    │          Type 2                   │     │   │
│    │    │      Context-Free                 │     │   │
│    │    │      (푸시다운 오토마타)              │     │   │
│    │    │    ┌───────────────────────┐     │     │   │
│    │    │    │      Type 3           │     │     │   │
│    │    │    │     Regular           │     │     │   │
│    │    │    │   (유한 오토마타)       │     │     │   │
│    │    │    └───────────────────────┘     │     │   │
│    │    └───────────────────────────────────┘     │   │
│    └───────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

### Type 3: 정규 언어 (Regular Language)

**문법 형태**: A → aB 또는 A → a (우선형) 또는 A → Ba 또는 A → a (좌선형)
- A, B: 비단말 기호 (Non-terminal)
- a: 단말 기호 (Terminal)

**인식 기계**: 유한 오토마타 (Finite Automata, FA)

**특징**:
- 메모리 없이 현재 상태만으로 판단
- 정규 표현식(Regular Expression)과 동치
- 컴파일러의 어휘 분석(Lexical Analysis)에 사용

**예시**:
```
L = {aⁿ | n ≥ 0}           // a의 임의 개수: 정규 언어 ✓
L = {ab, aabb, aaabbb, ...} // aⁿbⁿ: 정규 언어 ✗ (짝을 맞춰야 함)
L = {w | w는 짝수 개의 0을 포함} // 정규 언어 ✓
```

**정규 표현식 예시**:
```
(a|b)*       // {ε, a, b, aa, ab, ba, bb, aaa, ...}
a*b*         // {ε, a, b, aa, ab, bb, aab, abb, ...}
(ab)*        // {ε, ab, abab, ababab, ...}
[0-9]+       // 하나 이상의 숫자
[a-zA-Z_][a-zA-Z0-9_]*  // 식별자 패턴
```

### Type 2: 문맥 자유 언어 (Context-Free Language)

**문법 형태**: A → α
- A: 단일 비단말 기호
- α: 단말/비단말의 임의 조합 (빈 문자열 ε 포함 가능)

**인식 기계**: 푸시다운 오토마타 (Pushdown Automata, PDA)

**특징**:
- 스택을 사용하여 중첩 구조 처리 가능
- 프로그래밍 언어의 구문 분석에 핵심
- 괄호 매칭, 중첩 구조 표현 가능

**예시**:
```
// 괄호 매칭
S → (S) | SS | ε

// aⁿbⁿ 언어
S → aSb | ε
생성 과정: S → aSb → aaSbb → aaabbb

// 산술 표현식
E → E + T | T
T → T * F | F
F → (E) | id
```

**왜 정규 언어로 표현 불가능한가?**

정규 언어(유한 오토마타)는 유한한 상태만 가진다. `aⁿbⁿ`을 인식하려면 a의 개수를 "기억"해야 하는데, 유한 상태로는 무한히 다양한 n값을 구분할 수 없다. 이것이 **Pumping Lemma**의 핵심이다.

### Type 1: 문맥 의존 언어 (Context-Sensitive Language)

**문법 형태**: αAβ → αγβ
- |αAβ| ≤ |αγβ| (생성 규칙의 결과가 원본보다 짧아지면 안 됨)
- A가 특정 문맥(α, β) 안에서만 γ로 대체됨

**인식 기계**: 선형 유계 오토마타 (Linear Bounded Automata, LBA)

**특징**:
- 입력 길이에 비례하는 테이프만 사용하는 튜링 기계
- 자연어의 많은 특성을 표현 가능

**예시**:
```
L = {aⁿbⁿcⁿ | n ≥ 1}  // 문맥 의존 언어

// 문맥 자유 문법으로 불가능한 이유:
// 스택 하나로는 세 종류(a, b, c)의 개수를 동시에 맞출 수 없음

문법 규칙 예시:
S → aSBC | aBC
CB → BC        // 문맥 의존: C가 B 앞에 있을 때만 교환
aB → ab
bB → bb
bC → bc
cC → cc
```

### Type 0: 재귀 열거 가능 언어 (Recursively Enumerable Language)

**문법 형태**: α → β (제약 없음)
- α는 최소 하나의 비단말 포함
- β는 어떤 것이든 가능 (ε 포함)

**인식 기계**: 튜링 기계 (Turing Machine)

**특징**:
- 가장 강력한 계산 모델
- 정지 문제(Halting Problem) 등 결정 불가능한 문제 존재
- 언어가 속하면 튜링 기계가 "멈추며 수락", 속하지 않으면 "영원히 실행되거나 거부"

### 언어 타입별 비교표

| 타입 | 언어 종류 | 문법 규칙 | 인식 기계 | 폐쇄 특성 | 예시 |
|------|-----------|-----------|-----------|-----------|------|
| Type 3 | 정규 | A → aB, A → a | DFA/NFA | 합집합, 교집합, 여집합, 연결, * | 식별자, 숫자 리터럴 |
| Type 2 | 문맥 자유 | A → α | PDA | 합집합, 연결, * | 프로그래밍 언어 구문 |
| Type 1 | 문맥 의존 | αAβ → αγβ | LBA | 합집합, 교집합, 연결, * | aⁿbⁿcⁿ |
| Type 0 | 재귀 열거 | α → β | TM | 합집합, 교집합, 연결, * | 정지하는 프로그램 |

### Pumping Lemma

특정 언어가 정규 언어나 문맥 자유 언어가 **아님**을 증명하는 도구.

**정규 언어의 Pumping Lemma**:
```
L이 정규 언어이면, 어떤 pumping length p가 존재하여,
|w| ≥ p인 모든 w ∈ L에 대해 w = xyz로 분할 가능하고:
1. |y| > 0
2. |xy| ≤ p
3. 모든 i ≥ 0에 대해 xyⁱz ∈ L
```

**증명 예시: L = {aⁿbⁿ}이 정규 언어가 아님**
```
가정: L이 정규 언어이고 pumping length가 p
w = aᵖbᵖ 선택 (|w| = 2p ≥ p)

Pumping Lemma에 의해 w = xyz로 분할
조건 2에 의해 |xy| ≤ p이므로 xy는 모두 a로 구성
조건 1에 의해 y = aᵏ (k > 0)

xy²z = aᵖ⁺ᵏbᵖ를 고려하면
p + k ≠ p이므로 xy²z ∉ L

이는 조건 3에 모순 → L은 정규 언어가 아님
```

### 프로그래밍 언어와 형식 언어

실제 프로그래밍 언어는 여러 계층에서 다른 형식 언어를 사용한다:

```
┌─────────────────────────────────────────────┐
│             프로그래밍 언어                    │
├─────────────────────────────────────────────┤
│  어휘 분석 (Lexical Analysis)                │
│  → 정규 언어 (Regular Language)              │
│  → 토큰 인식: 식별자, 숫자, 키워드 등          │
├─────────────────────────────────────────────┤
│  구문 분석 (Syntax Analysis)                 │
│  → 문맥 자유 언어 (Context-Free Language)    │
│  → 문장 구조: 표현식, 문장, 함수 정의 등        │
├─────────────────────────────────────────────┤
│  의미 분석 (Semantic Analysis)               │
│  → 문맥 의존적 검사                           │
│  → 타입 검사, 변수 선언 확인 등                │
└─────────────────────────────────────────────┘
```

**예시: C 언어 식별자**
```c
// 정규 표현식으로 표현
[a-zA-Z_][a-zA-Z0-9_]*

// Type 3 문법
<identifier> → <letter> <id_tail>
<id_tail> → <letter> <id_tail> | <digit> <id_tail> | ε
<letter> → a | b | ... | z | A | ... | Z | _
<digit> → 0 | 1 | ... | 9
```

**예시: 산술 표현식 (문맥 자유 문법)**
```
<expr> → <expr> + <term> | <term>
<term> → <term> * <factor> | <factor>
<factor> → ( <expr> ) | <number> | <identifier>
```

## 실무 적용

### 컴파일러에서의 활용

```java
// 어휘 분석기 (Lexer) - 정규 언어 기반
public class SimpleLexer {
    private static final Pattern IDENTIFIER = Pattern.compile("[a-zA-Z_][a-zA-Z0-9_]*");
    private static final Pattern NUMBER = Pattern.compile("[0-9]+");
    private static final Pattern WHITESPACE = Pattern.compile("\\s+");

    public List<Token> tokenize(String input) {
        List<Token> tokens = new ArrayList<>();
        int pos = 0;

        while (pos < input.length()) {
            Matcher whitespaceMatcher = WHITESPACE.matcher(input.substring(pos));
            if (whitespaceMatcher.lookingAt()) {
                pos += whitespaceMatcher.end();
                continue;
            }

            Matcher idMatcher = IDENTIFIER.matcher(input.substring(pos));
            if (idMatcher.lookingAt()) {
                tokens.add(new Token(TokenType.IDENTIFIER, idMatcher.group()));
                pos += idMatcher.end();
                continue;
            }

            Matcher numMatcher = NUMBER.matcher(input.substring(pos));
            if (numMatcher.lookingAt()) {
                tokens.add(new Token(TokenType.NUMBER, numMatcher.group()));
                pos += numMatcher.end();
                continue;
            }

            // 연산자 등 처리
            char c = input.charAt(pos);
            tokens.add(new Token(getOperatorType(c), String.valueOf(c)));
            pos++;
        }

        return tokens;
    }
}
```

### 정규 표현식 엔진 이해

```python
# Python에서 정규 표현식 사용
import re

# 이메일 패턴 (실제로는 더 복잡함)
email_pattern = r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'

# 내부적으로 NFA 또는 DFA로 변환되어 실행
# Python의 re 모듈은 NFA 기반 (백트래킹 지원)
# Go의 regexp 패키지는 DFA 기반 (선형 시간 보장)

def validate_email(email):
    return bool(re.match(email_pattern, email))

# 성능 주의: 백트래킹으로 인한 ReDoS 공격 가능
# 취약 패턴 예: (a+)+ 에 'aaaaaaaaaaX' 입력
```

### 문법 설계 시 고려사항

```
// 모호한 문법 (Ambiguous Grammar) - 피해야 함
E → E + E | E * E | id

// 입력 "id + id * id"의 두 가지 해석:
// (id + id) * id
// id + (id * id)

// 모호성 제거: 연산자 우선순위 반영
E → E + T | T           // 덧셈 (낮은 우선순위)
T → T * F | F           // 곱셈 (높은 우선순위)
F → ( E ) | id          // 기본 단위

// 이제 "id + id * id"는 "id + (id * id)"로만 파싱됨
```

## 참고 자료

- Hopcroft, Motwani, Ullman - "Introduction to Automata Theory, Languages, and Computation"
- Sipser, Michael - "Introduction to the Theory of Computation"
- Chomsky, Noam - "Three models for the description of language" (1956)
- [Stanford CS103 - Formal Languages](https://web.stanford.edu/class/cs103/)
- [MIT 18.404J - Theory of Computation](https://ocw.mit.edu/courses/18-404j-theory-of-computation-fall-2020/)
