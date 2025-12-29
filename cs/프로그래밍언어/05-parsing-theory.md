# 파싱 이론 (Parsing Theory)

## 개요

파싱(Parsing)은 토큰 스트림을 문법 규칙에 따라 분석하여 구문 트리를 생성하는 과정이다. 컴파일러의 핵심 단계로, 프로그램의 구조적 정확성을 검증하고 후속 처리를 위한 트리 구조를 만든다.

파싱의 핵심 질문은 "주어진 토큰 스트림이 문법에 맞는지, 그리고 어떤 구조를 가지는가?"이다.

## 핵심 개념

### 파싱의 두 가지 방향

```
┌─────────────────────────────────────────────────────────────┐
│                    파싱 전략 비교                            │
├────────────────────────┬────────────────────────────────────┤
│     Top-Down 파싱      │         Bottom-Up 파싱             │
├────────────────────────┼────────────────────────────────────┤
│ • 시작 기호에서 출발   │ • 입력에서 출발                     │
│ • 입력 방향으로 유도   │ • 시작 기호 방향으로 축약           │
│ • 최좌단 유도 생성     │ • 최우단 유도의 역순 생성           │
│ • LL 파서 (Left-Left)  │ • LR 파서 (Left-Right)             │
│ • 예측적/재귀 하강     │ • Shift-Reduce 파서                │
├────────────────────────┼────────────────────────────────────┤
│ 구현이 직관적          │ 더 강력 (더 많은 문법 처리)         │
│ 좌재귀 처리 불가       │ 좌재귀 처리 가능                    │
│ 수동 구현 용이         │ 자동 생성 도구 사용 (Yacc, Bison)   │
└────────────────────────┴────────────────────────────────────┘
```

### Top-Down 파싱

시작 기호에서 출발하여 입력 문자열을 "예측"하며 유도하는 방식.

```
문법:
E → T E'
E' → + T E' | ε
T → F T'
T' → * F T' | ε
F → ( E ) | id

입력: id + id * id

Top-Down 파싱 과정:
┌────────────────────────────────────────────────────────────┐
│ 스택           │ 입력              │ 동작                   │
├────────────────┼───────────────────┼────────────────────────┤
│ E $            │ id + id * id $    │ E → T E' 적용          │
│ T E' $         │ id + id * id $    │ T → F T' 적용          │
│ F T' E' $      │ id + id * id $    │ F → id 적용            │
│ id T' E' $     │ id + id * id $    │ id 매칭                │
│ T' E' $        │ + id * id $       │ T' → ε 적용            │
│ E' $           │ + id * id $       │ E' → + T E' 적용       │
│ + T E' $       │ + id * id $       │ + 매칭                 │
│ T E' $         │ id * id $         │ T → F T' 적용          │
│ F T' E' $      │ id * id $         │ F → id 적용            │
│ id T' E' $     │ id * id $         │ id 매칭                │
│ T' E' $        │ * id $            │ T' → * F T' 적용       │
│ * F T' E' $    │ * id $            │ * 매칭                 │
│ F T' E' $      │ id $              │ F → id 적용            │
│ id T' E' $     │ id $              │ id 매칭                │
│ T' E' $        │ $                 │ T' → ε 적용            │
│ E' $           │ $                 │ E' → ε 적용            │
│ $              │ $                 │ 수락!                  │
└────────────────┴───────────────────┴────────────────────────┘
```

### Recursive Descent Parser (재귀 하강 파서)

각 비단말에 대해 함수를 작성하는 Top-Down 파싱 기법.

```java
// 문법:
// expr   → term (('+' | '-') term)*
// term   → factor (('*' | '/') factor)*
// factor → NUMBER | '(' expr ')'

public class RecursiveDescentParser {
    private Lexer lexer;
    private Token currentToken;

    public ASTNode parse() {
        currentToken = lexer.nextToken();
        ASTNode result = expr();
        expect(EOF);
        return result;
    }

    // expr → term (('+' | '-') term)*
    private ASTNode expr() {
        ASTNode node = term();

        while (currentToken.type == PLUS || currentToken.type == MINUS) {
            Token op = currentToken;
            advance();
            ASTNode right = term();
            node = new BinaryOp(op, node, right);
        }

        return node;
    }

    // term → factor (('*' | '/') factor)*
    private ASTNode term() {
        ASTNode node = factor();

        while (currentToken.type == STAR || currentToken.type == SLASH) {
            Token op = currentToken;
            advance();
            ASTNode right = factor();
            node = new BinaryOp(op, node, right);
        }

        return node;
    }

    // factor → NUMBER | '(' expr ')'
    private ASTNode factor() {
        if (currentToken.type == NUMBER) {
            ASTNode node = new NumberLiteral(currentToken.value);
            advance();
            return node;
        } else if (currentToken.type == LPAREN) {
            advance();  // '(' 소비
            ASTNode node = expr();
            expect(RPAREN);
            return node;
        } else {
            throw new ParseError("Unexpected token: " + currentToken);
        }
    }

    private void advance() {
        currentToken = lexer.nextToken();
    }

    private void expect(TokenType type) {
        if (currentToken.type != type) {
            throw new ParseError("Expected " + type + ", got " + currentToken.type);
        }
        advance();
    }
}
```

### LL(1) 파서

**LL(1)의 의미**:
- **L**: 입력을 왼쪽에서 오른쪽으로 스캔 (Left-to-right)
- **L**: 최좌단 유도 생성 (Leftmost derivation)
- **(1)**: 1개의 lookahead 토큰 사용

```
LL(1) 파싱 테이블 구성:

문법:
E  → T E'
E' → + T E' | ε
T  → F T'
T' → * F T' | ε
F  → ( E ) | id

FIRST와 FOLLOW 집합 계산:
FIRST(E) = FIRST(T) = FIRST(F) = { (, id }
FIRST(E') = { +, ε }
FIRST(T') = { *, ε }

FOLLOW(E) = { ), $ }
FOLLOW(E') = { ), $ }
FOLLOW(T) = { +, ), $ }
FOLLOW(T') = { +, ), $ }
FOLLOW(F) = { *, +, ), $ }

파싱 테이블 M[A, a]:
┌──────┬──────────────┬───────────────┬──────────────┬──────────┬──────────┐
│      │     id       │      +        │      *       │    (     │    )     │    $     │
├──────┼──────────────┼───────────────┼──────────────┼──────────┼──────────┤
│  E   │  E → T E'    │               │              │ E → T E' │          │          │
│  E'  │              │ E' → + T E'   │              │          │ E' → ε   │ E' → ε   │
│  T   │  T → F T'    │               │              │ T → F T' │          │          │
│  T'  │              │ T' → ε        │ T' → * F T'  │          │ T' → ε   │ T' → ε   │
│  F   │  F → id      │               │              │ F → (E)  │          │          │
└──────┴──────────────┴───────────────┴──────────────┴──────────┴──────────┘
```

### First 집합 계산

**FIRST(α)**: α에서 유도될 수 있는 첫 번째 단말 기호의 집합

```
FIRST 계산 규칙:

1. FIRST(a) = {a}  (a가 단말)

2. FIRST(A) 계산 (A가 비단말):
   - A → aβ 이면 FIRST(A)에 a 추가
   - A → Bβ 이면 FIRST(B) - {ε}를 FIRST(A)에 추가
     - FIRST(B)에 ε가 있으면 FIRST(β)도 추가
   - A → ε 이면 FIRST(A)에 ε 추가

3. FIRST(X₁X₂...Xₙ):
   - FIRST(X₁) - {ε}로 시작
   - X₁이 ε를 유도하면 FIRST(X₂) - {ε} 추가
   - ...계속
   - 모든 Xᵢ가 ε를 유도하면 ε 추가

예시:
문법:
S → ABC
A → a | ε
B → b | ε
C → c

FIRST(A) = {a, ε}
FIRST(B) = {b, ε}
FIRST(C) = {c}
FIRST(S) = FIRST(ABC)
         = {a} ∪ {b} ∪ {c}  (A와 B가 ε 유도 가능)
         = {a, b, c}
```

### Follow 집합 계산

**FOLLOW(A)**: A 다음에 나올 수 있는 단말 기호의 집합

```
FOLLOW 계산 규칙:

1. FOLLOW(S)에 $ 추가 (S는 시작 기호)

2. A → αBβ 형태의 모든 생성 규칙에 대해:
   - FIRST(β) - {ε}를 FOLLOW(B)에 추가
   - β가 ε를 유도하거나 β가 없으면:
     FOLLOW(A)를 FOLLOW(B)에 추가

예시:
문법:
E  → T E'
E' → + T E' | ε
T  → F T'
T' → * F T' | ε
F  → ( E ) | id

단계별 계산:
1. FOLLOW(E) = {$}  (시작 기호)

2. F → ( E ) 에서 E 뒤에 ) 가 옴
   FOLLOW(E) = {$, )}

3. E → T E' 에서:
   - T 뒤에 E'가 옴 → FIRST(E') - {ε} = {+}
   - E'가 ε 유도 → FOLLOW(E) 추가
   FOLLOW(T) = {+, ), $}

4. E' → + T E' 에서:
   - T 뒤에 E'가 옴 → {+}
   - E'가 ε 유도 → FOLLOW(E') 추가
   FOLLOW(E') = FOLLOW(E) = {), $}

5. T → F T' 와 T' → * F T' 에서:
   FOLLOW(F) = FIRST(T') ∪ (T'가 ε면) FOLLOW(T)
             = {*} ∪ {+, ), $}
             = {*, +, ), $}
```

### LL(1) 문법 조건

문법이 LL(1)이 되려면:

```
조건 1: First 집합의 비교집성
A → α | β 형태의 생성 규칙에서:
FIRST(α) ∩ FIRST(β) = ∅

조건 2: First-Follow 조건
A → α | β에서 ε ∈ FIRST(α)이면:
FIRST(α) ∩ FOLLOW(A) = ∅  (α가 ε를 유도할 때)
FIRST(β) ∩ FOLLOW(A) = ∅  (β가 ε를 유도할 때)

LL(1)이 아닌 예시:
1. 좌재귀: E → E + T | T
   FIRST(E + T) = FIRST(T) → 충돌

2. 공통 접두사: S → if E then S else S | if E then S
   둘 다 FIRST가 {if} → 충돌

3. 모호한 문법: E → E + E | id
   한 입력에 여러 유도 가능
```

### LL(1) 파싱 테이블 구성 알고리즘

```python
def build_ll1_table(grammar):
    table = {}  # table[non_terminal][terminal] = production

    for production in grammar.productions:
        A = production.lhs  # 좌변
        alpha = production.rhs  # 우변

        # FIRST(alpha)의 각 단말에 대해 규칙 추가
        for terminal in first(alpha):
            if terminal != EPSILON:
                if (A, terminal) in table:
                    raise LL1ConflictError(f"Conflict at [{A}, {terminal}]")
                table[(A, terminal)] = production

        # alpha가 ε를 유도하면 FOLLOW(A)의 각 단말에 대해 규칙 추가
        if EPSILON in first(alpha):
            for terminal in follow(A):
                if (A, terminal) in table:
                    raise LL1ConflictError(f"Conflict at [{A}, {terminal}]")
                table[(A, terminal)] = production

    return table
```

### 예측적 파싱 알고리즘

```python
def ll1_parse(tokens, table, start_symbol):
    stack = ['$', start_symbol]  # $는 스택 바닥 표시
    tokens.append(Token('$', '$'))  # 입력 끝 표시
    index = 0

    while stack:
        top = stack.pop()
        current = tokens[index]

        if top == '$':
            if current.type == '$':
                return True  # 파싱 성공
            else:
                raise ParseError("Unexpected input after parse")

        elif is_terminal(top):
            if top == current.type:
                index += 1  # 매칭 성공, 다음 토큰으로
            else:
                raise ParseError(f"Expected {top}, got {current.type}")

        else:  # 비단말
            key = (top, current.type)
            if key not in table:
                raise ParseError(f"No rule for [{top}, {current.type}]")

            production = table[key]
            # 생성 규칙의 우변을 역순으로 스택에 푸시
            for symbol in reversed(production.rhs):
                if symbol != EPSILON:
                    stack.append(symbol)

    return index == len(tokens) - 1
```

### 파서 에러 복구

```java
// Panic Mode 에러 복구
public class ErrorRecoveryParser {
    private Set<TokenType> synchronizingTokens;  // 동기화 토큰

    public void parse() {
        try {
            program();
        } catch (ParseError e) {
            reportError(e);
            synchronize();
            // 복구 후 계속 파싱 시도
        }
    }

    private void synchronize() {
        // 다음 동기화 토큰까지 스킵
        advance();
        while (!isAtEnd()) {
            // 문장 끝 (세미콜론) 이후 복구
            if (previous().type == SEMICOLON) return;

            // 새 문장/선언 시작점에서 복구
            switch (current().type) {
                case CLASS:
                case FUN:
                case VAR:
                case FOR:
                case IF:
                case WHILE:
                case RETURN:
                    return;
            }

            advance();
        }
    }

    // Phrase-Level 에러 복구
    private ASTNode factor() {
        if (match(NUMBER)) {
            return new NumberLiteral(previous().value);
        }
        if (match(LPAREN)) {
            ASTNode expr = expression();
            if (!match(RPAREN)) {
                // 닫는 괄호 누락 → 에러 보고 후 삽입한 것처럼 처리
                reportError("Expected ')' after expression");
                // 복구: 괄호가 있다고 가정하고 계속
            }
            return expr;
        }

        // 에러: 예상치 못한 토큰
        throw new ParseError("Expected expression");
    }
}
```

### LL(k) 및 LL(*) 파싱

```
LL(k): k개의 lookahead 토큰 사용
- k가 클수록 더 많은 문법 처리 가능
- 파싱 테이블 크기가 지수적으로 증가

LL(*): 필요한 만큼 lookahead 사용 (ANTLR)
- 정규 표현식으로 lookahead 패턴 표현
- 백트래킹 최소화

예시: LL(2) 필요한 문법
S → a b | a c

LL(1)에서:
FIRST(a b) = {a}
FIRST(a c) = {a}
충돌!

LL(2)에서:
FIRST₂(a b) = {ab}
FIRST₂(a c) = {ac}
구분 가능!
```

## 실무 적용

### 표현식 파서 (Pratt Parsing)

연산자 우선순위를 우아하게 처리하는 파싱 기법.

```java
public class PrattParser {
    private Map<TokenType, Integer> precedence = Map.of(
        PLUS, 1, MINUS, 1,
        STAR, 2, SLASH, 2,
        CARET, 3  // 거듭제곱 (우결합)
    );

    private Map<TokenType, Boolean> rightAssoc = Map.of(
        CARET, true
    );

    public ASTNode parseExpression(int minPrecedence) {
        // Prefix 처리 (숫자, 변수, 단항 연산자, 괄호)
        ASTNode left = parsePrefix();

        // Infix 처리 (이항 연산자)
        while (true) {
            TokenType op = current().type;
            int prec = precedence.getOrDefault(op, 0);

            if (prec < minPrecedence) break;

            advance();

            // 결합성 처리: 우결합이면 같은 우선순위도 재귀
            int nextMinPrec = rightAssoc.getOrDefault(op, false) ? prec : prec + 1;
            ASTNode right = parseExpression(nextMinPrec);

            left = new BinaryOp(op, left, right);
        }

        return left;
    }

    private ASTNode parsePrefix() {
        Token token = current();

        if (token.type == NUMBER) {
            advance();
            return new NumberLiteral(token.value);
        }

        if (token.type == IDENTIFIER) {
            advance();
            return new Variable(token.value);
        }

        if (token.type == MINUS) {
            advance();
            ASTNode operand = parseExpression(3);  // 단항 -는 높은 우선순위
            return new UnaryOp(MINUS, operand);
        }

        if (token.type == LPAREN) {
            advance();
            ASTNode expr = parseExpression(0);
            expect(RPAREN);
            return expr;
        }

        throw new ParseError("Unexpected token: " + token);
    }
}

// 입력: 1 + 2 * 3 ^ 4 ^ 5
// 파싱: 1 + (2 * (3 ^ (4 ^ 5)))
```

### 파서 조합기 (Parser Combinator)

함수형 스타일로 파서를 조합하는 방법.

```scala
// Scala의 Parser Combinator 예시
import scala.util.parsing.combinator._

object ArithmeticParser extends RegexParsers {
  def number: Parser[Int] = """\d+""".r ^^ { _.toInt }

  def factor: Parser[Int] = number | "(" ~> expr <~ ")"

  def term: Parser[Int] = factor ~ rep("*" ~ factor | "/" ~ factor) ^^ {
    case n ~ list => list.foldLeft(n) {
      case (acc, "*" ~ x) => acc * x
      case (acc, "/" ~ x) => acc / x
    }
  }

  def expr: Parser[Int] = term ~ rep("+" ~ term | "-" ~ term) ^^ {
    case n ~ list => list.foldLeft(n) {
      case (acc, "+" ~ x) => acc + x
      case (acc, "-" ~ x) => acc - x
    }
  }

  def apply(input: String): Int = parseAll(expr, input).get
}

// 사용
ArithmeticParser("1 + 2 * 3")  // 7
ArithmeticParser("(1 + 2) * 3")  // 9
```

### ANTLR 사용 예시

```antlr
// Expr.g4 - ANTLR 문법 파일
grammar Expr;

// 파서 규칙
prog: stat+ ;

stat: expr NEWLINE          # printExpr
    | ID '=' expr NEWLINE   # assign
    | NEWLINE               # blank
    ;

expr: expr op=('*'|'/') expr    # MulDiv
    | expr op=('+'|'-') expr    # AddSub
    | INT                        # int
    | ID                         # id
    | '(' expr ')'              # parens
    ;

// 렉서 규칙
MUL : '*' ;
DIV : '/' ;
ADD : '+' ;
SUB : '-' ;
ID  : [a-zA-Z]+ ;
INT : [0-9]+ ;
NEWLINE : '\r'? '\n' ;
WS  : [ \t]+ -> skip ;
```

```java
// ANTLR 생성 파서 사용
public class Calculator extends ExprBaseVisitor<Integer> {
    Map<String, Integer> memory = new HashMap<>();

    @Override
    public Integer visitAssign(ExprParser.AssignContext ctx) {
        String id = ctx.ID().getText();
        int value = visit(ctx.expr());
        memory.put(id, value);
        return value;
    }

    @Override
    public Integer visitMulDiv(ExprParser.MulDivContext ctx) {
        int left = visit(ctx.expr(0));
        int right = visit(ctx.expr(1));
        if (ctx.op.getType() == ExprParser.MUL)
            return left * right;
        return left / right;
    }

    @Override
    public Integer visitAddSub(ExprParser.AddSubContext ctx) {
        int left = visit(ctx.expr(0));
        int right = visit(ctx.expr(1));
        if (ctx.op.getType() == ExprParser.ADD)
            return left + right;
        return left - right;
    }

    @Override
    public Integer visitId(ExprParser.IdContext ctx) {
        String id = ctx.ID().getText();
        return memory.getOrDefault(id, 0);
    }

    @Override
    public Integer visitInt(ExprParser.IntContext ctx) {
        return Integer.valueOf(ctx.INT().getText());
    }

    @Override
    public Integer visitParens(ExprParser.ParensContext ctx) {
        return visit(ctx.expr());
    }
}
```

## 참고 자료

- Aho, Lam, Sethi, Ullman - "Compilers: Principles, Techniques, and Tools" (Dragon Book)
- Appel, Andrew - "Modern Compiler Implementation"
- Grune, Jacobs - "Parsing Techniques: A Practical Guide"
- [ANTLR](https://www.antlr.org/) - Parser Generator
- Bob Nystrom - "Crafting Interpreters" - Chapter 6: Parsing Expressions
- Pratt, Vaughan - "Top Down Operator Precedence" (1973)
