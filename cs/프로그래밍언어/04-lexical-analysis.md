# 어휘 분석 (Lexical Analysis)

## 개요

어휘 분석(Lexical Analysis)은 컴파일러의 첫 번째 단계로, 소스 코드의 문자 스트림을 의미 있는 단위인 토큰(Token)으로 변환하는 과정이다. 이 과정을 수행하는 모듈을 Lexer(또는 Scanner, Tokenizer)라 한다.

어휘 분석의 핵심 역할은 "문자 스트림에서 패턴을 인식하고 토큰을 생성하는 것"이다.

## 핵심 개념

### 컴파일러 전단부에서의 위치

```
┌─────────────────────────────────────────────────────────────┐
│                     컴파일러 전단부                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  소스 코드     어휘 분석      토큰       구문 분석      AST    │
│  ─────────→  (Lexer)  ─────────→  (Parser)  ─────────→     │
│  문자 스트림                 토큰 스트림              구문 트리  │
│                                                             │
│  "if (x > 0)"  →  [IF][LPAREN][ID:x][GT][NUM:0][RPAREN]    │
│                                                             │
└─────────────────────────────────────────────────────────────┘

역할 분리의 이유:
1. 관심사 분리: 문자 처리 vs 구조 분석
2. 효율성: 정규 언어(빠름)와 문맥 자유 언어(복잡함) 분리
3. 이식성: 문자 인코딩 처리를 한 곳에 집중
```

### 토큰 (Token)

토큰은 어휘 분석의 출력 단위로, 토큰 타입과 속성값으로 구성된다.

```
토큰 구조:
┌─────────────┬─────────────────────┐
│  토큰 타입   │    속성값(옵션)      │
├─────────────┼─────────────────────┤
│ IDENTIFIER  │ "count"             │
│ NUMBER      │ 42                  │
│ STRING      │ "hello"             │
│ KEYWORD     │ (if, while 등)      │
│ OPERATOR    │ +, -, *, /          │
│ DELIMITER   │ (, ), {, }, ;       │
└─────────────┴─────────────────────┘

토큰 vs 렉심(Lexeme) vs 패턴:
- 패턴: 토큰을 정의하는 규칙 (정규 표현식)
- 렉심: 패턴에 매칭되는 실제 문자열
- 토큰: (타입, 속성값) 쌍

예시:
  패턴: [a-zA-Z_][a-zA-Z0-9_]*
  렉심: "count", "total", "MAX_SIZE"
  토큰: (IDENTIFIER, "count"), (IDENTIFIER, "total")
```

### 토큰 종류

```java
public enum TokenType {
    // 키워드
    IF, ELSE, WHILE, FOR, RETURN, CLASS, PUBLIC, PRIVATE,

    // 리터럴
    INTEGER_LITERAL,    // 42, 0xFF, 0b1010
    FLOAT_LITERAL,      // 3.14, 1.0e-10
    STRING_LITERAL,     // "hello"
    CHAR_LITERAL,       // 'a'
    BOOLEAN_LITERAL,    // true, false

    // 식별자
    IDENTIFIER,         // variable, functionName

    // 연산자
    PLUS, MINUS, STAR, SLASH,           // + - * /
    EQ, NE, LT, GT, LE, GE,             // == != < > <= >=
    AND, OR, NOT,                        // && || !
    ASSIGN,                              // =
    PLUS_ASSIGN, MINUS_ASSIGN,          // += -=

    // 구분자
    LPAREN, RPAREN,     // ( )
    LBRACE, RBRACE,     // { }
    LBRACKET, RBRACKET, // [ ]
    SEMICOLON,          // ;
    COMMA,              // ,
    DOT,                // .

    // 특수
    EOF,                // 파일 끝
    NEWLINE,            // 줄바꿈 (Python 등)
    INDENT, DEDENT,     // 들여쓰기 (Python)
}
```

### Lexer 동작 원리

```
입력: "if (count >= 10) {"

┌─────────────────────────────────────────────────────────┐
│ Lexer 처리 과정                                         │
├─────────────────────────────────────────────────────────┤
│ 위치  │ 현재 문자  │ 동작                    │ 결과     │
├───────┼───────────┼─────────────────────────┼──────────┤
│  0    │ 'i'       │ 식별자/키워드 시작      │          │
│  1    │ 'f'       │ 계속 읽기              │          │
│  2    │ ' '       │ 끝 → 키워드 "if" 확인  │ IF       │
│  3    │ '('       │ 단일 문자 토큰          │ LPAREN   │
│  4    │ 'c'       │ 식별자 시작            │          │
│  ...  │ ...       │ "count" 읽기           │          │
│  9    │ ' '       │ 끝                     │ ID:count │
│ 10    │ '>'       │ 연산자 시작            │          │
│ 11    │ '='       │ '>=' 확인              │ GE       │
│ 12    │ ' '       │ 공백 스킵              │          │
│ 13    │ '1'       │ 숫자 시작              │          │
│ 14    │ '0'       │ 계속 읽기              │          │
│ 15    │ ')'       │ 끝                     │ NUM:10   │
│ 16    │ ')'       │ 단일 문자 토큰          │ RPAREN   │
│ 17    │ ' '       │ 공백 스킵              │          │
│ 18    │ '{'       │ 단일 문자 토큰          │ LBRACE   │
└───────┴───────────┴─────────────────────────┴──────────┘

출력 토큰 스트림:
[IF] [LPAREN] [IDENTIFIER:"count"] [GE] [INTEGER:10] [RPAREN] [LBRACE]
```

### Maximal Munch (최대 일치 원칙)

가능한 가장 긴 토큰을 선택하는 원칙.

```
입력: "+++++x"

Maximal Munch 적용:
1. "++" 인식 → INCREMENT
2. "++" 인식 → INCREMENT
3. "+" 인식 → PLUS
4. "x" 인식 → IDENTIFIER

결과: ++ ++ + x
의미: ((++)(++))(+x) → 전위 증가 두 번 후 x에 더함

Maximal Munch 없이 (첫 번째 매치 선택):
결과: + + + + + x
의미: 완전히 다른 해석

실제 예시들:
">>>" → UNSIGNED_RIGHT_SHIFT (Java)
        vs ">" ">" ">" (제네릭과 섞일 때 문제)

"<:" → SUBTYPE 연산자 (Scala)
       vs "<" ":" (비교 후 타입 지정)
```

### 예약어 처리

키워드(예약어)는 식별자와 같은 패턴을 가지므로 특별 처리 필요.

```
방법 1: 키워드 테이블 조회
┌──────────────────────────────────────┐
│ 1. [a-zA-Z_][a-zA-Z0-9_]* 패턴 매치  │
│ 2. 키워드 해시 테이블 조회            │
│ 3. 매치되면 KEYWORD, 아니면 ID       │
└──────────────────────────────────────┘

public Token scanIdentifierOrKeyword() {
    String lexeme = scanWhile(c -> isAlphanumeric(c) || c == '_');

    // 키워드 테이블 조회 (해시맵 O(1))
    TokenType keyword = KEYWORDS.get(lexeme);
    if (keyword != null) {
        return new Token(keyword, lexeme);
    }
    return new Token(IDENTIFIER, lexeme);
}

private static final Map<String, TokenType> KEYWORDS = Map.of(
    "if", IF,
    "else", ELSE,
    "while", WHILE,
    "for", FOR,
    "return", RETURN
    // ...
);

방법 2: 별도 DFA 상태 (Flex/Lex 방식)
- 각 키워드에 대해 명시적 패턴
- 키워드 패턴이 식별자보다 먼저 매칭되도록 우선순위 지정
```

### 정규 표현식 기반 토큰 정의

```
// 토큰 패턴 정의 (정규 표현식)
WHITESPACE    = [ \t\n\r]+
COMMENT       = "//" [^\n]* | "/*" ([^*] | "*"[^/])* "*/"

INTEGER       = [0-9]+
              | 0[xX][0-9a-fA-F]+       // 16진수
              | 0[bB][01]+              // 2진수
              | 0[0-7]+                 // 8진수

FLOAT         = [0-9]+ "." [0-9]* ([eE][+-]?[0-9]+)?
              | [0-9]* "." [0-9]+ ([eE][+-]?[0-9]+)?
              | [0-9]+ [eE][+-]?[0-9]+

STRING        = '"' ([^"\\] | '\\' .)* '"'
              | "'" ([^'\\] | '\\' .)* "'"

IDENTIFIER    = [a-zA-Z_][a-zA-Z0-9_]*

// 연산자 (긴 것부터 매칭 - Maximal Munch)
OPERATOR      = ">>=" | "<<=" | ">>>" | ">>" | "<<"
              | "+=" | "-=" | "*=" | "/=" | "%="
              | "==" | "!=" | "<=" | ">=" | "&&" | "||"
              | "++" | "--"
              | "+" | "-" | "*" | "/" | "%" | "="
              | "<" | ">" | "!" | "&" | "|" | "^"
```

### Flex/Lex 사용

```lex
/* 토큰 정의 - calculator.l */
%{
#include "y.tab.h"  /* 파서가 생성한 토큰 정의 */
%}

%%

[ \t\n]+        { /* 공백 무시 */ }
"//"[^\n]*      { /* 한 줄 주석 무시 */ }

[0-9]+          { yylval.num = atoi(yytext); return NUMBER; }
[0-9]+\.[0-9]+  { yylval.fnum = atof(yytext); return FLOAT; }

"if"            { return IF; }
"else"          { return ELSE; }
"while"         { return WHILE; }
"return"        { return RETURN; }

[a-zA-Z_][a-zA-Z0-9_]* { yylval.str = strdup(yytext); return IDENTIFIER; }

"+"             { return PLUS; }
"-"             { return MINUS; }
"*"             { return STAR; }
"/"             { return SLASH; }
"=="            { return EQ; }
"!="            { return NE; }
"<="            { return LE; }
">="            { return GE; }
"<"             { return LT; }
">"             { return GT; }
"="             { return ASSIGN; }

"("             { return LPAREN; }
")"             { return RPAREN; }
"{"             { return LBRACE; }
"}"             { return RBRACE; }
";"             { return SEMICOLON; }
","             { return COMMA; }

.               { printf("Unknown character: %s\n", yytext); }

%%

int yywrap() { return 1; }
```

### 에러 복구

```java
public class Lexer {
    public Token nextToken() {
        skipWhitespaceAndComments();

        if (isAtEnd()) {
            return new Token(EOF, "", line, column);
        }

        char c = peek();

        // 숫자
        if (isDigit(c)) return scanNumber();

        // 식별자 / 키워드
        if (isAlpha(c) || c == '_') return scanIdentifier();

        // 문자열
        if (c == '"' || c == '\'') return scanString();

        // 연산자와 구분자
        Token op = scanOperator();
        if (op != null) return op;

        // 에러 복구: 알 수 없는 문자
        return errorToken("Unexpected character: " + c);
    }

    private Token errorToken(String message) {
        // 에러 보고
        errors.add(new LexicalError(message, line, column));

        // Panic Mode: 다음 공백이나 알려진 문자까지 스킵
        while (!isAtEnd() && !isWhitespace(peek()) && !isKnownStart(peek())) {
            advance();
        }

        // 에러 토큰 반환 (파서에서 처리)
        return new Token(ERROR, message, line, column);
    }
}
```

## 실무 적용

### 수동 Lexer 구현

```java
public class ManualLexer {
    private final String source;
    private int start = 0;
    private int current = 0;
    private int line = 1;
    private int column = 1;
    private final List<Token> tokens = new ArrayList<>();

    public List<Token> scanTokens() {
        while (!isAtEnd()) {
            start = current;
            scanToken();
        }
        tokens.add(new Token(EOF, "", line, column));
        return tokens;
    }

    private void scanToken() {
        char c = advance();

        switch (c) {
            // 단일 문자 토큰
            case '(': addToken(LPAREN); break;
            case ')': addToken(RPAREN); break;
            case '{': addToken(LBRACE); break;
            case '}': addToken(RBRACE); break;
            case ';': addToken(SEMICOLON); break;
            case ',': addToken(COMMA); break;

            // 1-2문자 토큰
            case '+':
                addToken(match('+') ? INCREMENT : match('=') ? PLUS_ASSIGN : PLUS);
                break;
            case '-':
                addToken(match('-') ? DECREMENT : match('=') ? MINUS_ASSIGN : MINUS);
                break;
            case '=':
                addToken(match('=') ? EQ : ASSIGN);
                break;
            case '!':
                addToken(match('=') ? NE : NOT);
                break;
            case '<':
                addToken(match('=') ? LE : match('<') ? LSHIFT : LT);
                break;
            case '>':
                addToken(match('=') ? GE : match('>') ? RSHIFT : GT);
                break;

            // 슬래시 (나눗셈 또는 주석)
            case '/':
                if (match('/')) {
                    // 한 줄 주석
                    while (peek() != '\n' && !isAtEnd()) advance();
                } else if (match('*')) {
                    // 블록 주석
                    blockComment();
                } else if (match('=')) {
                    addToken(SLASH_ASSIGN);
                } else {
                    addToken(SLASH);
                }
                break;

            // 공백
            case ' ':
            case '\r':
            case '\t':
                break;
            case '\n':
                line++;
                column = 1;
                break;

            // 문자열
            case '"': string(); break;

            default:
                if (isDigit(c)) {
                    number();
                } else if (isAlpha(c)) {
                    identifier();
                } else {
                    error("Unexpected character.");
                }
                break;
        }
    }

    private void number() {
        // 정수 부분
        while (isDigit(peek())) advance();

        // 16진수 체크
        if (current - start == 1 && source.charAt(start) == '0') {
            if (peek() == 'x' || peek() == 'X') {
                advance();
                while (isHexDigit(peek())) advance();
                addToken(INTEGER_LITERAL,
                    Long.parseLong(source.substring(start + 2, current), 16));
                return;
            }
        }

        // 소수점
        if (peek() == '.' && isDigit(peekNext())) {
            advance(); // '.' 소비
            while (isDigit(peek())) advance();
        }

        // 지수
        if (peek() == 'e' || peek() == 'E') {
            advance();
            if (peek() == '+' || peek() == '-') advance();
            while (isDigit(peek())) advance();
            addToken(FLOAT_LITERAL,
                Double.parseDouble(source.substring(start, current)));
        } else if (source.substring(start, current).contains(".")) {
            addToken(FLOAT_LITERAL,
                Double.parseDouble(source.substring(start, current)));
        } else {
            addToken(INTEGER_LITERAL,
                Long.parseLong(source.substring(start, current)));
        }
    }

    private void string() {
        StringBuilder value = new StringBuilder();
        while (peek() != '"' && !isAtEnd()) {
            if (peek() == '\n') {
                line++;
                column = 1;
            }
            if (peek() == '\\') {
                advance();
                value.append(escapeChar(advance()));
            } else {
                value.append(advance());
            }
        }

        if (isAtEnd()) {
            error("Unterminated string.");
            return;
        }

        advance(); // 닫는 따옴표
        addToken(STRING_LITERAL, value.toString());
    }

    private char escapeChar(char c) {
        return switch (c) {
            case 'n' -> '\n';
            case 't' -> '\t';
            case 'r' -> '\r';
            case '\\' -> '\\';
            case '"' -> '"';
            case '0' -> '\0';
            default -> c;
        };
    }

    private void identifier() {
        while (isAlphanumeric(peek())) advance();

        String text = source.substring(start, current);
        TokenType type = keywords.getOrDefault(text, IDENTIFIER);
        addToken(type);
    }

    // 유틸리티 메서드들
    private char advance() {
        column++;
        return source.charAt(current++);
    }

    private boolean match(char expected) {
        if (isAtEnd() || source.charAt(current) != expected) return false;
        current++;
        column++;
        return true;
    }

    private char peek() {
        return isAtEnd() ? '\0' : source.charAt(current);
    }

    private char peekNext() {
        return current + 1 >= source.length() ? '\0' : source.charAt(current + 1);
    }

    private boolean isAtEnd() {
        return current >= source.length();
    }
}
```

### Python 스타일 들여쓰기 처리

```python
# Python Lexer는 INDENT/DEDENT 토큰 생성
class PythonLexer:
    def __init__(self, source):
        self.source = source
        self.indent_stack = [0]  # 들여쓰기 레벨 스택
        self.tokens = []

    def tokenize(self):
        lines = self.source.split('\n')

        for line in lines:
            if self.is_blank_or_comment(line):
                continue

            # 들여쓰기 계산
            indent = self.count_indent(line)
            current_indent = self.indent_stack[-1]

            if indent > current_indent:
                # 들여쓰기 증가
                self.indent_stack.append(indent)
                self.tokens.append(Token(INDENT))
            else:
                # 들여쓰기 감소 (여러 레벨 가능)
                while indent < self.indent_stack[-1]:
                    self.indent_stack.pop()
                    self.tokens.append(Token(DEDENT))

                if indent != self.indent_stack[-1]:
                    raise IndentationError("unindent does not match")

            # 라인 내용 토큰화
            self.tokenize_line(line.strip())
            self.tokens.append(Token(NEWLINE))

        # 파일 끝에서 남은 DEDENT 처리
        while len(self.indent_stack) > 1:
            self.indent_stack.pop()
            self.tokens.append(Token(DEDENT))

        return self.tokens

# 입력:
# if x > 0:
#     print(x)
#     if y > 0:
#         print(y)
#     print("done")
# print("end")

# 토큰 스트림:
# IF ID:x GT NUM:0 COLON NEWLINE
# INDENT PRINT LPAREN ID:x RPAREN NEWLINE
# IF ID:y GT NUM:0 COLON NEWLINE
# INDENT PRINT LPAREN ID:y RPAREN NEWLINE
# DEDENT PRINT LPAREN STRING:"done" RPAREN NEWLINE
# DEDENT PRINT LPAREN STRING:"end" RPAREN NEWLINE
```

### 숫자 리터럴 파싱

```java
public class NumberLexer {
    // 다양한 숫자 형식 지원
    // 10진수: 123, 123.456, 1.23e10
    // 16진수: 0xFF, 0XFF
    // 8진수: 0o77, 0O77
    // 2진수: 0b1010, 0B1010
    // 구분자: 1_000_000 (Java, Python)

    public Token scanNumber() {
        StringBuilder sb = new StringBuilder();
        int base = 10;

        // 접두사 확인
        if (peek() == '0' && !isAtEnd(1)) {
            char next = peekNext();
            switch (next) {
                case 'x', 'X' -> { advance(); advance(); base = 16; }
                case 'o', 'O' -> { advance(); advance(); base = 8; }
                case 'b', 'B' -> { advance(); advance(); base = 2; }
            }
        }

        // 숫자 읽기 (언더스코어 허용)
        if (base == 16) {
            while (isHexDigit(peek()) || peek() == '_') {
                if (peek() != '_') sb.append(peek());
                advance();
            }
            return makeIntToken(sb.toString(), 16);
        }

        // 10진수 처리
        while (isDigit(peek()) || peek() == '_') {
            if (peek() != '_') sb.append(peek());
            advance();
        }

        // 소수점
        if (peek() == '.' && isDigit(peekNext())) {
            sb.append(advance());
            while (isDigit(peek()) || peek() == '_') {
                if (peek() != '_') sb.append(peek());
                advance();
            }
        }

        // 지수
        if (peek() == 'e' || peek() == 'E') {
            sb.append(advance());
            if (peek() == '+' || peek() == '-') {
                sb.append(advance());
            }
            while (isDigit(peek())) {
                sb.append(advance());
            }
        }

        // 접미사 (L, F, D 등)
        char suffix = peek();
        if (suffix == 'L' || suffix == 'l') {
            advance();
            return makeLongToken(sb.toString());
        } else if (suffix == 'F' || suffix == 'f') {
            advance();
            return makeFloatToken(sb.toString());
        } else if (suffix == 'D' || suffix == 'd') {
            advance();
            return makeDoubleToken(sb.toString());
        }

        // 기본값
        if (sb.toString().contains(".") || sb.toString().contains("e")) {
            return makeDoubleToken(sb.toString());
        }
        return makeIntToken(sb.toString(), base);
    }
}
```

## 참고 자료

- Aho, Lam, Sethi, Ullman - "Compilers: Principles, Techniques, and Tools" (Dragon Book)
- Flex Manual: https://westes.github.io/flex/manual/
- Crafting Interpreters - Chapter 4: Scanning
- [re2c](https://re2c.org/) - Lexer Generator
- Bob Nystrom - "Crafting Interpreters" (무료 온라인 도서)
