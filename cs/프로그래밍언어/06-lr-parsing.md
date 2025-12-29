# LR 파싱 (LR Parsing)

## 개요

LR 파싱은 Bottom-Up 파싱의 대표적인 기법으로, 프로그래밍 언어 컴파일러에서 가장 널리 사용되는 구문 분석 방법이다. LL 파싱보다 더 넓은 범위의 문법을 처리할 수 있으며, Yacc/Bison과 같은 파서 생성기의 기반이 된다.

LR의 의미는 "입력을 왼쪽에서 오른쪽으로(Left-to-right) 읽고, 최우단 유도(Rightmost derivation)의 역순을 생성한다"는 것이다.

## 핵심 개념

### LR 파싱의 기본 원리

```
Bottom-Up 파싱: 입력에서 시작하여 시작 기호로 축약

문법:
E → E + T | T
T → T * F | F
F → ( E ) | id

입력: id + id * id

축약 과정 (최우단 유도의 역순):
id + id * id
F + id * id      (F → id)
T + id * id      (T → F)
E + id * id      (E → T)
E + F * id       (F → id)
E + T * id       (T → F)
E + T * F        (F → id)
E + T            (T → T * F)
E                (E → E + T)

핵심 동작:
- Shift: 입력 토큰을 스택으로 이동
- Reduce: 스택 상단의 핸들을 비단말로 대체
```

### Shift-Reduce 파싱

```
스택과 입력을 사용한 Shift-Reduce 파싱:

┌─────────────────────────────────────────────────────────────┐
│ 스택              │ 입력                │ 동작              │
├───────────────────┼─────────────────────┼───────────────────┤
│ $                 │ id + id * id $      │ Shift             │
│ $ id              │ + id * id $         │ Reduce F → id     │
│ $ F               │ + id * id $         │ Reduce T → F      │
│ $ T               │ + id * id $         │ Reduce E → T      │
│ $ E               │ + id * id $         │ Shift             │
│ $ E +             │ id * id $           │ Shift             │
│ $ E + id          │ * id $              │ Reduce F → id     │
│ $ E + F           │ * id $              │ Reduce T → F      │
│ $ E + T           │ * id $              │ Shift (T*F 위해)  │
│ $ E + T *         │ id $                │ Shift             │
│ $ E + T * id      │ $                   │ Reduce F → id     │
│ $ E + T * F       │ $                   │ Reduce T → T*F    │
│ $ E + T           │ $                   │ Reduce E → E+T    │
│ $ E               │ $                   │ Accept            │
└───────────────────┴─────────────────────┴───────────────────┘

핵심 결정 문제:
- 언제 Shift하고 언제 Reduce할 것인가?
- Reduce 시 어떤 규칙을 적용할 것인가?
→ LR 파싱 테이블이 이를 결정
```

### 핸들 (Handle)

```
핸들: 현재 시점에서 축약해야 할 부분 문자열

정의:
- 핸들 = 생성 규칙의 우변과 일치하는 스택 상단 부분
- 핸들을 축약하면 최우단 유도의 역순이 생성됨

예시:
우측 문장 형식: E + T * F
핸들: T * F (T → T * F의 우변)
축약 후: E + T

핸들 찾기:
- 핸들은 항상 스택 상단에 있음 (모두 들어와야 함)
- LR 파서는 상태(state)를 사용하여 핸들 인식
```

### LR(0) 아이템

LR 파싱의 기본 단위로, 생성 규칙 내 "점(dot)"의 위치를 표시.

```
생성 규칙: E → E + T

LR(0) 아이템들:
E → . E + T    (아직 아무것도 본 적 없음)
E → E . + T    (E를 봤음)
E → E + . T    (E와 +를 봤음)
E → E + T .    (모두 봤음 - 완료 아이템)

점(dot)의 의미:
- 점 왼쪽: 이미 본 기호들 (스택에 있음)
- 점 오른쪽: 아직 볼 기호들 (입력에서 올 것)
```

### 클로저 연산 (Closure)

```
아이템 집합의 클로저 계산:

규칙:
1. I의 모든 아이템은 CLOSURE(I)에 포함
2. A → α.Bβ가 CLOSURE(I)에 있고 B → γ가 문법에 있으면
   B → .γ를 CLOSURE(I)에 추가
3. 새 아이템이 추가되지 않을 때까지 반복

예시:
문법:
E' → E
E → E + T | T
T → T * F | F
F → ( E ) | id

CLOSURE({E' → .E}):
E' → .E
E → .E + T    (E 앞에 점이므로 E의 규칙 추가)
E → .T        (E 앞에 점이므로 E의 규칙 추가)
T → .T * F    (T 앞에 점이므로 T의 규칙 추가)
T → .F        (T 앞에 점이므로 T의 규칙 추가)
F → .( E )    (F 앞에 점이므로 F의 규칙 추가)
F → .id       (F 앞에 점이므로 F의 규칙 추가)
```

### GOTO 연산

```
GOTO(I, X) = CLOSURE({A → αX.β | A → α.Xβ ∈ I})

의미: 상태 I에서 기호 X를 보고 이동한 후의 상태

예시:
I₀ = CLOSURE({E' → .E})

GOTO(I₀, E) = CLOSURE({E' → E., E → E.+T})
= {E' → E., E → E.+T}

GOTO(I₀, T) = CLOSURE({E → T.})
= {E → T., T → T.*F}

GOTO(I₀, F) = CLOSURE({T → F.})
= {T → F.}

GOTO(I₀, id) = CLOSURE({F → id.})
= {F → id.}
```

### LR(0) 오토마톤 구성

```
문법:
E' → E
E → E + T | T
T → T * F | F
F → ( E ) | id

LR(0) 상태들:

I₀ (시작):
E' → .E
E → .E + T
E → .T
T → .T * F
T → .F
F → .( E )
F → .id

I₁ = GOTO(I₀, E):
E' → E.
E → E. + T

I₂ = GOTO(I₀, T):
E → T.
T → T. * F

I₃ = GOTO(I₀, F):
T → F.

I₄ = GOTO(I₀, ( ):
F → (. E )
E → .E + T
E → .T
T → .T * F
T → .F
F → .( E )
F → .id

I₅ = GOTO(I₀, id):
F → id.

... 나머지 상태들
```

### LR(0) 파싱 테이블

```
ACTION 테이블: 현재 상태와 입력 토큰에 따른 동작
- sN: Shift하고 상태 N으로 전이
- rK: 규칙 K로 Reduce
- acc: Accept (파싱 성공)
- 빈칸: 에러

GOTO 테이블: 현재 상태와 비단말에 따른 다음 상태

규칙 번호:
(1) E' → E
(2) E → E + T
(3) E → T
(4) T → T * F
(5) T → F
(6) F → ( E )
(7) F → id

LR(0) 파싱 테이블 (충돌 있음):
┌───────┬──────┬──────┬──────┬──────┬──────┬──────┬──────┬──────┬──────┐
│ 상태  │  id  │  +   │  *   │  (   │  )   │  $   │  E   │  T   │  F   │
├───────┼──────┼──────┼──────┼──────┼──────┼──────┼──────┼──────┼──────┤
│   0   │  s5  │      │      │  s4  │      │      │  1   │  2   │  3   │
│   1   │      │  s6  │      │      │      │ acc  │      │      │      │
│   2   │      │ r3   │ s7   │      │ r3   │ r3   │      │      │      │
│   3   │      │ r5   │ r5   │      │ r5   │ r5   │      │      │      │
│   4   │  s5  │      │      │  s4  │      │      │  8   │  2   │  3   │
│   5   │      │ r7   │ r7   │      │ r7   │ r7   │      │      │      │
│   6   │  s5  │      │      │  s4  │      │      │      │  9   │  3   │
│   7   │  s5  │      │      │  s4  │      │      │      │      │ 10   │
│   8   │      │  s6  │      │      │ s11  │      │      │      │      │
│   9   │      │ r2   │ s7   │      │ r2   │ r2   │      │      │      │
│  10   │      │ r4   │ r4   │      │ r4   │ r4   │      │      │      │
│  11   │      │ r6   │ r6   │      │ r6   │ r6   │      │      │      │
└───────┴──────┴──────┴──────┴──────┴──────┴──────┴──────┴──────┴──────┘
```

### SLR(1) 파서

Simple LR - FOLLOW 집합을 사용하여 Reduce 충돌 해결

```
SLR(1) 개선:
- LR(0)의 Reduce는 모든 단말에 대해 적용
- SLR(1)은 FOLLOW 집합에 있는 단말에만 Reduce 적용

예시:
상태 2: E → T. 에서
- LR(0): 모든 토큰에 r3 적용
- SLR(1): FOLLOW(E) = {+, ), $}에만 r3 적용

SLR(1) 파싱 테이블 구성:
1. LR(0) 아이템 집합과 상태 구성
2. Shift 액션은 동일
3. Reduce: A → α.가 상태 i에 있으면
   FOLLOW(A)의 모든 토큰에 대해 reduce

한계:
- 여전히 일부 문법에서 충돌 발생
- FOLLOW가 너무 큰 집합일 수 있음
```

### LALR(1) 파서

Look-Ahead LR - 같은 코어를 가진 상태를 병합

```
LALR(1) 특징:
- LR(1)의 상태 수를 줄임
- 같은 LR(0) 코어를 가진 상태를 병합
- Lookahead만 결합

LR(1) 아이템: [A → α.β, a]
- A → α.β: LR(0) 아이템 (코어)
- a: Lookahead 토큰

LR(1) 클로저:
A → α.Bβ, a 가 있으면
B → .γ, b 를 추가 (b ∈ FIRST(βa))

LALR(1) 병합:
LR(1) 상태 {[A → α., a], [B → β., b]}와
              {[A → α., c], [B → β., d]}는
같은 코어 [A → α.], [B → β.]를 가짐
→ 병합: {[A → α., a/c], [B → β., b/d]}

장점:
- LR(1)보다 훨씬 적은 상태 수
- 대부분의 프로그래밍 언어 문법 처리 가능
- Yacc/Bison의 기본 알고리즘
```

### LR(1) 파서

Canonical LR - 가장 강력하지만 상태가 많음

```
LR(1) 아이템: [A → α.β, a]
- Lookahead a는 해당 규칙으로 Reduce할 때 참조

LR(1) CLOSURE({[S' → .S, $]}):
[S' → .S, $]
[S → .CC, $]         // FIRST(ε$) = {$}
[C → .cC, c]         // FIRST(C$) = {c}
[C → .d, c]          // FIRST(C$) = {c}
[C → .cC, d]
[C → .d, d]

LR(1) vs LALR(1) 상태 수:
- C 언어 문법: LR(1) ~수천 상태, LALR(1) ~수백 상태
- 대부분 LALR(1)로 충분

LR(1)이 필요한 문법 (LALR(1)에서 충돌):
S → aAd | bBd | aBe | bAe
A → c
B → c
```

### Shift-Reduce 충돌

```
문제: 같은 상태에서 Shift와 Reduce 모두 가능

예시 (Dangling Else):
stmt → if expr then stmt
     | if expr then stmt else stmt
     | other

상태: if expr then stmt 까지 봤을 때
- else가 오면: Shift (else stmt 계속 읽기)
- 다른 것이 오면: Reduce (if expr then stmt 완료)
- else도 Reduce 가능 → 충돌!

해결:
1. 우선순위 규칙: else는 Shift 선호 (대부분 언어)
2. 문법 재작성
3. 파서 생성기 지시자 (%prec 등)
```

### Reduce-Reduce 충돌

```
문제: 같은 상태에서 두 가지 Reduce 가능

예시:
S → aABe
A → Abc | b
B → d

상태에서 입력 d를 봤을 때:
- A → b. 로 Reduce?
- B → d. 로 Reduce?

원인: 보통 문법 설계 오류

해결:
1. 문법 재작성 (권장)
2. 우선순위 지정 (첫 번째 규칙 선택 등)
```

### 연산자 우선순위와 결합성

```yacc
/* Yacc/Bison에서 연산자 우선순위 지정 */

%left '+' '-'       /* 낮은 우선순위, 좌결합 */
%left '*' '/'       /* 높은 우선순위, 좌결합 */
%right '^'          /* 더 높은 우선순위, 우결합 */
%nonassoc UMINUS    /* 단항 마이너스, 비결합 */

%%

expr:
    expr '+' expr       { $$ = $1 + $3; }
  | expr '-' expr       { $$ = $1 - $3; }
  | expr '*' expr       { $$ = $1 * $3; }
  | expr '/' expr       { $$ = $1 / $3; }
  | expr '^' expr       { $$ = pow($1, $3); }
  | '-' expr %prec UMINUS  { $$ = -$2; }  /* UMINUS 우선순위 적용 */
  | '(' expr ')'        { $$ = $2; }
  | NUMBER              { $$ = $1; }
  ;

/* 우선순위/결합성 처리:
   - %left: 같은 우선순위면 좌결합 (Reduce 선호)
   - %right: 같은 우선순위면 우결합 (Shift 선호)
   - %nonassoc: 같은 우선순위 연속 불허
   - %prec: 특정 규칙에 다른 우선순위 부여 */
```

## 실무 적용

### Yacc/Bison 사용

```yacc
/* calculator.y */
%{
#include <stdio.h>
#include <stdlib.h>
void yyerror(const char *s);
int yylex();
%}

%union {
    int ival;
    double dval;
    char* sval;
}

%token <ival> INTEGER
%token <dval> FLOAT
%token <sval> IDENTIFIER
%token IF ELSE WHILE

%type <dval> expr term factor

%left '+' '-'
%left '*' '/'
%right '^'

%%

program:
    statements
    ;

statements:
    statement
  | statements statement
    ;

statement:
    IDENTIFIER '=' expr ';'   { printf("%s = %f\n", $1, $3); }
  | expr ';'                  { printf("= %f\n", $1); }
  | IF '(' expr ')' statement { /* if 처리 */ }
  | IF '(' expr ')' statement ELSE statement
    ;

expr:
    expr '+' term   { $$ = $1 + $3; }
  | expr '-' term   { $$ = $1 - $3; }
  | term            { $$ = $1; }
    ;

term:
    term '*' factor { $$ = $1 * $3; }
  | term '/' factor { $$ = $1 / $3; }
  | factor          { $$ = $1; }
    ;

factor:
    INTEGER         { $$ = (double)$1; }
  | FLOAT           { $$ = $1; }
  | '(' expr ')'    { $$ = $2; }
  | '-' factor      { $$ = -$2; }
    ;

%%

void yyerror(const char *s) {
    fprintf(stderr, "Error: %s\n", s);
}

int main() {
    return yyparse();
}
```

### LR 파싱 테이블 기반 파서 구현

```java
public class LRParser {
    private int[][] actionTable;  // [state][terminal] -> action
    private int[][] gotoTable;    // [state][nonterminal] -> state
    private Production[] rules;
    private Lexer lexer;

    // Action 인코딩: shift=양수, reduce=음수, accept=0
    private static final int ACCEPT = 0;

    public ASTNode parse() {
        Stack<Integer> stateStack = new Stack<>();
        Stack<Object> symbolStack = new Stack<>();  // 토큰 또는 AST 노드
        stateStack.push(0);  // 초기 상태

        Token token = lexer.nextToken();

        while (true) {
            int state = stateStack.peek();
            int terminalIndex = token.getIndex();
            int action = actionTable[state][terminalIndex];

            if (action > 0) {
                // Shift
                int nextState = action;
                symbolStack.push(token);
                stateStack.push(nextState);
                token = lexer.nextToken();

            } else if (action < 0) {
                // Reduce
                int ruleIndex = -action;
                Production rule = rules[ruleIndex];

                // 규칙의 우변 길이만큼 pop
                Object[] children = new Object[rule.rhsLength()];
                for (int i = rule.rhsLength() - 1; i >= 0; i--) {
                    stateStack.pop();
                    children[i] = symbolStack.pop();
                }

                // 의미 액션 실행하여 AST 노드 생성
                ASTNode node = rule.semanticAction(children);
                symbolStack.push(node);

                // GOTO 테이블로 다음 상태 결정
                int topState = stateStack.peek();
                int nonterminalIndex = rule.lhs().getIndex();
                int nextState = gotoTable[topState][nonterminalIndex];
                stateStack.push(nextState);

            } else if (action == ACCEPT) {
                // 파싱 성공
                return (ASTNode) symbolStack.pop();

            } else {
                // 에러
                throw new ParseError("Syntax error at " + token);
            }
        }
    }
}
```

### 에러 복구

```java
public class LRParserWithErrorRecovery extends LRParser {

    // Panic Mode 에러 복구
    private void panicModeRecovery(Stack<Integer> stateStack,
                                   Stack<Object> symbolStack,
                                   Token token) {
        reportError(token);

        // 1. 스택에서 복구 가능한 상태 찾기
        while (!stateStack.isEmpty()) {
            int state = stateStack.peek();

            // 에러 토큰에 대한 동작이 있는 상태 찾기
            if (hasErrorProduction(state)) {
                // error 심볼로 shift
                stateStack.push(gotoOnError(state));
                symbolStack.push(new ErrorNode());

                // 동기화 토큰까지 입력 버리기
                while (!isSynchronizingToken(token)) {
                    token = lexer.nextToken();
                }
                return;
            }

            stateStack.pop();
            if (!symbolStack.isEmpty()) symbolStack.pop();
        }

        throw new UnrecoverableParseError();
    }

    // Phrase-Level 에러 복구
    private Token phraseLevelRecovery(Token token, int expected) {
        reportError("Expected " + getTokenName(expected) +
                   ", found " + token);

        // 누락된 토큰 삽입
        if (canInsert(expected)) {
            return createDummyToken(expected);
        }

        // 잘못된 토큰 삭제
        if (canDelete(token)) {
            return lexer.nextToken();
        }

        // 토큰 교체
        return createDummyToken(expected);
    }
}
```

### GLR 파싱 (일반화 LR)

```
모호한 문법도 처리하는 LR 파싱의 확장

원리:
- 충돌 시 여러 파싱 경로를 병렬로 추적
- 파스 포레스트(Parse Forest) 생성
- 나중에 의미 분석으로 모호성 해결

적용:
- 자연어 처리 (모호성이 많음)
- C++ 파싱 (선언과 표현식이 모호할 수 있음)

도구:
- Bison의 %glr-parser 옵션
- Elkhound
- Tree-sitter

예시:
Time flies like an arrow.
→ 여러 해석 가능 (GLR로 모두 생성)
```

## 참고 자료

- Aho, Lam, Sethi, Ullman - "Compilers: Principles, Techniques, and Tools" (Dragon Book)
- Appel, Andrew - "Modern Compiler Implementation"
- [Bison Manual](https://www.gnu.org/software/bison/manual/)
- [Stanford CS143 - LR Parsing](https://web.stanford.edu/class/cs143/)
- DeRemer, Frank - "Practical Translators for LR(k) Languages" (LALR 창시)
- Tomita, Masaru - "Efficient Parsing for Natural Language" (GLR)
