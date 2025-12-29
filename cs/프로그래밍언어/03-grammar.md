# 문법 (Grammar)

## 개요

문법(Grammar)은 언어의 구조를 정의하는 형식적인 규칙 체계이다. 프로그래밍 언어의 구문(Syntax)을 명세하고, 컴파일러의 파서를 자동 생성하는 기반이 된다. BNF, EBNF 같은 표기법으로 문법을 기술하며, 문맥 자유 문법(CFG)이 가장 널리 사용된다.

문법 설계의 핵심 과제는 "어떻게 명확하고 파싱 가능한 언어를 정의하는가?"이다.

## 핵심 개념

### 문법의 형식적 정의

**문맥 자유 문법(Context-Free Grammar, CFG)**은 4-튜플 G = (V, T, P, S)로 정의된다:

```
V: 비단말 기호(Non-terminal) 집합 - 문법 변수
T: 단말 기호(Terminal) 집합 - 실제 토큰
P: 생성 규칙(Production Rules) 집합
S: 시작 기호(Start Symbol) - S ∈ V

예시: 간단한 산술 표현식 문법
V = {E, T, F}
T = {+, *, (, ), id}
S = E
P:
  E → E + T | T
  T → T * F | F
  F → ( E ) | id
```

### BNF (Backus-Naur Form)

John Backus와 Peter Naur가 ALGOL 60을 정의하기 위해 개발한 표기법.

```bnf
<expression> ::= <term> | <expression> "+" <term>
<term>       ::= <factor> | <term> "*" <factor>
<factor>     ::= <number> | "(" <expression> ")"
<number>     ::= <digit> | <number> <digit>
<digit>      ::= "0" | "1" | "2" | "3" | "4" | "5" | "6" | "7" | "8" | "9"

표기법:
<...>  : 비단말 기호
"..."  : 단말 기호 (리터럴)
::=    : 정의됨
|      : 선택 (OR)
```

**BNF의 한계**:
- 반복을 표현하려면 재귀 사용 필요
- 선택적 요소 표현이 장황함

### EBNF (Extended BNF)

BNF를 확장하여 반복, 선택 등을 간결하게 표현.

```ebnf
expression = term , { ("+" | "-") , term } ;
term       = factor , { ("*" | "/") , factor } ;
factor     = number | "(" , expression , ")" ;
number     = digit , { digit } ;
digit      = "0" | "1" | "2" | "3" | "4" | "5" | "6" | "7" | "8" | "9" ;

EBNF 확장 표기법:
{ ... }    : 0회 이상 반복 (Kleene closure)
[ ... ]    : 선택적 (0 또는 1회)
( ... )    : 그룹화
,          : 연결 (concatenation)
;          : 규칙 종료
```

**EBNF 변형들**:
```
ISO EBNF:
  repetition = { "a" } ;        (* 0회 이상 *)
  optional   = [ "a" ] ;        (* 선택적 *)

W3C EBNF (XML, JSON 등에서 사용):
  repetition = "a"* ;           (* 0회 이상 *)
  oneOrMore  = "a"+ ;           (* 1회 이상 *)
  optional   = "a"? ;           (* 선택적 *)

ABNF (RFC 5234, 인터넷 프로토콜에서 사용):
  repetition = *"a"             ; 0회 이상
  oneOrMore  = 1*"a"            ; 1회 이상
  exactly3   = 3"a"             ; 정확히 3회
  range      = 2*4"a"           ; 2~4회
```

### 유도 (Derivation)

시작 기호에서 문자열을 생성하는 과정.

```
문법:
E → E + T | T
T → T * F | F
F → ( E ) | id

"id + id * id" 유도:

최좌단 유도 (Leftmost Derivation):
E ⟹ E + T              (E → E + T)
  ⟹ T + T              (E → T)
  ⟹ F + T              (T → F)
  ⟹ id + T             (F → id)
  ⟹ id + T * F         (T → T * F)
  ⟹ id + F * F         (T → F)
  ⟹ id + id * F        (F → id)
  ⟹ id + id * id       (F → id)

최우단 유도 (Rightmost Derivation):
E ⟹ E + T
  ⟹ E + T * F
  ⟹ E + T * id
  ⟹ E + F * id
  ⟹ E + id * id
  ⟹ T + id * id
  ⟹ F + id * id
  ⟹ id + id * id
```

### Parse Tree (파스 트리)

유도 과정을 트리로 표현한 것.

```
"id + id * id"의 파스 트리:

              E
           /  |  \
          E   +   T
          |      /|\
          T     T * F
          |     |   |
          F     F  id
          |     |
         id    id

트리 특성:
- 루트: 시작 기호
- 내부 노드: 비단말 기호
- 리프 노드: 단말 기호
- 리프를 왼쪽에서 오른쪽으로 읽으면 원래 문자열
```

### 모호한 문법 (Ambiguous Grammar)

하나의 문자열에 대해 두 개 이상의 파스 트리가 존재하는 문법.

```
모호한 문법:
E → E + E | E * E | id

"id + id * id"의 두 가지 해석:

해석 1: (id + id) * id          해석 2: id + (id * id)

        E                               E
      / | \                           / | \
     E  *  E                         E  +  E
    /|\    |                         |    /|\
   E + E  id                        id  E * E
   |   |                                |   |
  id  id                               id  id

문제점:
- 컴파일러가 어떤 해석을 선택할지 결정 불가
- 연산 결과가 달라질 수 있음
  - (2 + 3) * 4 = 20
  - 2 + (3 * 4) = 14
```

### 모호성 제거

**방법 1: 우선순위와 결합성 반영**

```
모호한 문법:
E → E + E | E * E | id

모호성 제거된 문법 (연산자 우선순위 반영):
E → E + T | T           // 덧셈은 낮은 우선순위
T → T * F | F           // 곱셈은 높은 우선순위
F → ( E ) | id

이제 "id + id * id"는 오직 id + (id * id)로만 파싱됨:

        E
      / | \
     E  +  T
     |    /|\
     T   T * F
     |   |   |
     F   F  id
     |   |
    id  id
```

**방법 2: 결합성(Associativity) 정의**

```
좌결합(Left Associative): a - b - c = (a - b) - c
E → E - T | T

우결합(Right Associative): a = b = c → a = (b = c)
E → T = E | T

"a - b - c"의 파싱 (좌결합):
        E
      / | \
     E  -  T
    /|\    |
   E - T   c
   |   |
   a   b
```

### Left Recursion (좌재귀)

비단말 기호가 자기 자신으로 시작하는 생성 규칙.

```
직접 좌재귀:
A → Aα | β

간접 좌재귀:
A → Bα
B → Aβ

문제점:
- Top-Down 파서(LL 파서)에서 무한 루프 발생
- A를 파싱하려면 먼저 A를 파싱해야 함 → 무한 재귀
```

### Left Recursion 제거

```
원본 (좌재귀):
A → Aα₁ | Aα₂ | ... | Aαₙ | β₁ | β₂ | ... | βₘ

변환 후:
A  → β₁A' | β₂A' | ... | βₘA'
A' → α₁A' | α₂A' | ... | αₙA' | ε

예시:
원본: E → E + T | T

변환 후:
E  → T E'
E' → + T E' | ε

유도 비교:
원본:    E → E + T → T + T → id + id
변환 후: E → T E' → id E' → id + T E' → id + id E' → id + id
```

**간접 좌재귀 제거**:
```
원본:
A → Bα | a
B → Aβ | b

1단계: B를 A에 대입
A → Aβα | bα | a

2단계: 직접 좌재귀 제거
A  → bαA' | aA'
A' → βαA' | ε
```

### Left Factoring (좌인수분해)

공통 접두사를 가진 생성 규칙을 분리하여 LL 파싱 가능하게 만듦.

```
문제 상황:
stmt → if expr then stmt else stmt
     | if expr then stmt

파서가 "if"를 보면 어떤 규칙을 선택할지 결정할 수 없음
(else가 나올 때까지 알 수 없음)

좌인수분해 적용:
stmt      → if expr then stmt stmt_tail
stmt_tail → else stmt | ε

이제 파서는:
1. "if"를 보고 stmt 규칙 적용
2. expr, then, stmt를 파싱
3. 다음 토큰이 "else"면 else stmt 파싱, 아니면 ε
```

**일반적인 좌인수분해**:
```
원본:
A → αβ₁ | αβ₂ | ... | αβₙ | γ

변환 후:
A  → αA' | γ
A' → β₁ | β₂ | ... | βₙ
```

### Dangling Else 문제

```
문법:
stmt → if expr then stmt
     | if expr then stmt else stmt
     | other

입력: if a then if b then s1 else s2

두 가지 해석:
1. if a then (if b then s1 else s2)  // else가 안쪽 if에 매칭
2. if a then (if b then s1) else s2  // else가 바깥쪽 if에 매칭

해결책 1: 문법 재작성
stmt         → matched | unmatched
matched      → if expr then matched else matched | other
unmatched    → if expr then stmt
             | if expr then matched else unmatched

해결책 2: 언어 설계 변경
- Python: 들여쓰기로 블록 구분
- 대부분 언어: else는 가장 가까운 if에 매칭 (관례)
- 일부 언어: if-then-else-end (명시적 종료)
```

### 문맥 자유 문법의 한계

CFG로 표현할 수 없는 언어 특성들:

```
1. 변수 선언과 사용
   "int x; x = 5;"에서 두 x가 같은 변수인지

2. 타입 일치
   "f(a, b)"에서 a, b의 타입이 f의 매개변수 타입과 맞는지

3. 개수 일치
   함수 정의의 매개변수 개수와 호출 시 인자 개수

4. 유일성 제약
   같은 스코프에서 중복 선언 금지

해결: 의미 분석(Semantic Analysis) 단계에서 처리
- 심볼 테이블 사용
- 속성 문법(Attribute Grammar)
- 타입 시스템
```

## 실무 적용

### 프로그래밍 언어 문법 예시

```ebnf
// JavaScript 함수 정의 (간략화)
FunctionDeclaration
  = "function" Identifier "(" FormalParameterList? ")" "{" FunctionBody "}"

FormalParameterList
  = Identifier ("," Identifier)*

FunctionBody
  = SourceElements?

SourceElements
  = SourceElement+

SourceElement
  = Statement
  | FunctionDeclaration
```

```ebnf
// SQL SELECT 문 (간략화)
SelectStatement
  = "SELECT" SelectList
    "FROM" TableReference
    WhereClause?
    GroupByClause?
    OrderByClause?
    LimitClause?

SelectList
  = "*" | SelectItem ("," SelectItem)*

SelectItem
  = Expression ("AS" Identifier)?

WhereClause
  = "WHERE" SearchCondition
```

### 재귀 하강 파서와 문법

```java
// 문법:
// expr   → term (('+' | '-') term)*
// term   → factor (('*' | '/') factor)*
// factor → NUMBER | '(' expr ')'

public class RecursiveDescentParser {
    private Lexer lexer;
    private Token currentToken;

    // expr → term (('+' | '-') term)*
    public ASTNode parseExpr() {
        ASTNode left = parseTerm();

        while (currentToken.type == PLUS || currentToken.type == MINUS) {
            Token op = currentToken;
            advance();
            ASTNode right = parseTerm();
            left = new BinaryOpNode(op, left, right);
        }

        return left;
    }

    // term → factor (('*' | '/') factor)*
    public ASTNode parseTerm() {
        ASTNode left = parseFactor();

        while (currentToken.type == STAR || currentToken.type == SLASH) {
            Token op = currentToken;
            advance();
            ASTNode right = parseFactor();
            left = new BinaryOpNode(op, left, right);
        }

        return left;
    }

    // factor → NUMBER | '(' expr ')'
    public ASTNode parseFactor() {
        if (currentToken.type == NUMBER) {
            ASTNode node = new NumberNode(currentToken.value);
            advance();
            return node;
        } else if (currentToken.type == LPAREN) {
            advance();  // '(' 소비
            ASTNode node = parseExpr();
            expect(RPAREN);  // ')' 확인 및 소비
            return node;
        } else {
            throw new SyntaxError("Unexpected token: " + currentToken);
        }
    }
}
```

### 파서 생성기 사용

```yacc
/* Yacc/Bison 문법 파일 예시 */
%token NUMBER IDENTIFIER
%token PLUS MINUS TIMES DIVIDE
%token LPAREN RPAREN

%left PLUS MINUS      /* 낮은 우선순위, 좌결합 */
%left TIMES DIVIDE    /* 높은 우선순위, 좌결합 */

%%

expr:
    expr PLUS expr      { $$ = make_binop('+', $1, $3); }
  | expr MINUS expr     { $$ = make_binop('-', $1, $3); }
  | expr TIMES expr     { $$ = make_binop('*', $1, $3); }
  | expr DIVIDE expr    { $$ = make_binop('/', $1, $3); }
  | LPAREN expr RPAREN  { $$ = $2; }
  | NUMBER              { $$ = make_number($1); }
  | IDENTIFIER          { $$ = make_ident($1); }
  ;

%%

/* %left, %right로 모호성 해결 */
/* 우선순위: 나중에 선언된 것이 높음 */
```

### JSON 문법

```ebnf
// JSON 완전한 문법 (RFC 8259)
json
  = element

element
  = ws value ws

value
  = object
  | array
  | string
  | number
  | "true"
  | "false"
  | "null"

object
  = "{" ws "}"
  | "{" member ("," member)* "}"

member
  = ws string ws ":" element

array
  = "[" ws "]"
  | "[" element ("," element)* "]"

string
  = '"' character* '"'

character
  = unescaped
  | "\" escape

escape
  = '"' | "\" | "/" | "b" | "f" | "n" | "r" | "t"
  | "u" hex hex hex hex

number
  = integer fraction? exponent?

integer
  = digit
  | onenine digits
  | "-" digit
  | "-" onenine digits

ws
  = (" " | "\t" | "\n" | "\r")*
```

## 참고 자료

- Aho, Lam, Sethi, Ullman - "Compilers: Principles, Techniques, and Tools" (Dragon Book)
- Sebesta, Robert - "Concepts of Programming Languages"
- ISO/IEC 14977 - Information technology — Syntactic metalanguage — Extended BNF
- [Yacc/Bison Manual](https://www.gnu.org/software/bison/manual/)
- [ANTLR](https://www.antlr.org/) - Parser Generator
- [PEG (Parsing Expression Grammar)](https://en.wikipedia.org/wiki/Parsing_expression_grammar)
