# 추상 구문 트리 (Abstract Syntax Tree, AST)

## 개요

추상 구문 트리(AST)는 프로그램의 구문 구조를 표현하는 트리 형태의 자료구조이다. 파스 트리(Parse Tree)와 달리 구문 분석에 필요했던 세부 사항을 제거하고 프로그램의 의미에 필수적인 요소만 남긴다. 컴파일러, 인터프리터, 정적 분석 도구, IDE의 핵심 데이터 구조이다.

AST의 핵심 역할은 "프로그램의 구조적 표현을 제공하여 후속 처리(타입 검사, 최적화, 코드 생성)를 가능하게 하는 것"이다.

## 핵심 개념

### Parse Tree vs AST

```
입력: 1 + 2 * 3

Parse Tree (구체적 구문 트리):
문법의 모든 세부 사항 포함

          E
        / | \
       E  +  T
       |    /|\
       T   T * F
       |   |   |
       F   F   3
       |   |
       1   2

AST (추상 구문 트리):
의미적으로 중요한 것만 포함

        +
       / \
      1   *
         / \
        2   3

차이점:
┌────────────────────────┬────────────────────────────────────┐
│     Parse Tree         │            AST                     │
├────────────────────────┼────────────────────────────────────┤
│ 모든 문법 규칙 반영     │ 의미적 구조만 반영                   │
│ 비단말 노드 모두 포함   │ 필요한 노드만 포함                   │
│ 연산자 우선순위 암묵적  │ 트리 구조가 우선순위 명시            │
│ 괄호, 구분자 포함       │ 괄호, 구분자 제거                   │
│ 파싱 규칙에 종속        │ 언어 의미에 종속                    │
└────────────────────────┴────────────────────────────────────┘
```

### AST 노드 설계

```java
// 기본 AST 노드 인터페이스
public abstract class ASTNode {
    private SourceLocation location;  // 원본 위치 정보

    public abstract <T> T accept(ASTVisitor<T> visitor);
}

// 표현식 노드들
public abstract class Expression extends ASTNode {}

public class BinaryExpression extends Expression {
    private Expression left;
    private BinaryOperator operator;
    private Expression right;

    @Override
    public <T> T accept(ASTVisitor<T> visitor) {
        return visitor.visitBinaryExpression(this);
    }
}

public class UnaryExpression extends Expression {
    private UnaryOperator operator;
    private Expression operand;
}

public class Literal extends Expression {
    private Object value;
    private LiteralType type;  // INTEGER, FLOAT, STRING, BOOLEAN
}

public class Identifier extends Expression {
    private String name;
}

public class CallExpression extends Expression {
    private Expression callee;  // 함수 표현식
    private List<Expression> arguments;
}

public class MemberExpression extends Expression {
    private Expression object;
    private String property;
    private boolean computed;  // obj.prop vs obj["prop"]
}

// 문장 노드들
public abstract class Statement extends ASTNode {}

public class VariableDeclaration extends Statement {
    private String name;
    private TypeAnnotation type;  // nullable for type inference
    private Expression initializer;  // nullable
    private boolean isFinal;
}

public class IfStatement extends Statement {
    private Expression condition;
    private Statement thenBranch;
    private Statement elseBranch;  // nullable
}

public class WhileStatement extends Statement {
    private Expression condition;
    private Statement body;
}

public class ForStatement extends Statement {
    private Statement initializer;
    private Expression condition;
    private Expression update;
    private Statement body;
}

public class ReturnStatement extends Statement {
    private Expression value;  // nullable for void return
}

public class BlockStatement extends Statement {
    private List<Statement> statements;
}

// 선언 노드들
public class FunctionDeclaration extends ASTNode {
    private String name;
    private List<Parameter> parameters;
    private TypeAnnotation returnType;
    private BlockStatement body;
}

public class ClassDeclaration extends ASTNode {
    private String name;
    private Identifier superclass;  // nullable
    private List<MethodDeclaration> methods;
    private List<FieldDeclaration> fields;
}
```

### AST 구축 (파서에서)

```java
// 재귀 하강 파서에서 AST 구축
public class Parser {
    private Lexer lexer;
    private Token currentToken;

    // expr → term (('+' | '-') term)*
    public Expression parseExpression() {
        Expression left = parseTerm();

        while (match(PLUS, MINUS)) {
            Token operator = previous();
            Expression right = parseTerm();
            left = new BinaryExpression(left, toBinaryOp(operator), right);
        }

        return left;
    }

    // term → factor (('*' | '/') factor)*
    private Expression parseTerm() {
        Expression left = parseFactor();

        while (match(STAR, SLASH)) {
            Token operator = previous();
            Expression right = parseFactor();
            left = new BinaryExpression(left, toBinaryOp(operator), right);
        }

        return left;
    }

    // if statement
    public Statement parseIfStatement() {
        expect(IF);
        expect(LPAREN);
        Expression condition = parseExpression();
        expect(RPAREN);

        Statement thenBranch = parseStatement();

        Statement elseBranch = null;
        if (match(ELSE)) {
            elseBranch = parseStatement();
        }

        return new IfStatement(condition, thenBranch, elseBranch);
    }

    // function declaration
    public FunctionDeclaration parseFunctionDeclaration() {
        expect(FUNCTION);
        Token name = expect(IDENTIFIER);
        expect(LPAREN);

        List<Parameter> params = new ArrayList<>();
        if (!check(RPAREN)) {
            do {
                Token paramName = expect(IDENTIFIER);
                expect(COLON);
                TypeAnnotation paramType = parseType();
                params.add(new Parameter(paramName.value, paramType));
            } while (match(COMMA));
        }
        expect(RPAREN);

        TypeAnnotation returnType = null;
        if (match(COLON)) {
            returnType = parseType();
        }

        BlockStatement body = parseBlock();

        return new FunctionDeclaration(name.value, params, returnType, body);
    }
}
```

### Visitor Pattern

AST 순회의 표준 패턴으로, 노드 타입별 처리를 깔끔하게 분리.

```java
// Visitor 인터페이스
public interface ASTVisitor<T> {
    // 표현식
    T visitBinaryExpression(BinaryExpression expr);
    T visitUnaryExpression(UnaryExpression expr);
    T visitLiteral(Literal expr);
    T visitIdentifier(Identifier expr);
    T visitCallExpression(CallExpression expr);
    T visitMemberExpression(MemberExpression expr);

    // 문장
    T visitVariableDeclaration(VariableDeclaration stmt);
    T visitIfStatement(IfStatement stmt);
    T visitWhileStatement(WhileStatement stmt);
    T visitForStatement(ForStatement stmt);
    T visitReturnStatement(ReturnStatement stmt);
    T visitBlockStatement(BlockStatement stmt);

    // 선언
    T visitFunctionDeclaration(FunctionDeclaration decl);
    T visitClassDeclaration(ClassDeclaration decl);
}

// 인터프리터 구현
public class Interpreter implements ASTVisitor<Object> {
    private Environment environment = new Environment();

    @Override
    public Object visitBinaryExpression(BinaryExpression expr) {
        Object left = expr.getLeft().accept(this);
        Object right = expr.getRight().accept(this);

        return switch (expr.getOperator()) {
            case PLUS -> {
                if (left instanceof Number && right instanceof Number) {
                    yield ((Number) left).doubleValue() +
                          ((Number) right).doubleValue();
                }
                if (left instanceof String || right instanceof String) {
                    yield String.valueOf(left) + String.valueOf(right);
                }
                throw new RuntimeError("Invalid operands for +");
            }
            case MINUS -> checkNumbers(left, right,
                (l, r) -> l.doubleValue() - r.doubleValue());
            case STAR -> checkNumbers(left, right,
                (l, r) -> l.doubleValue() * r.doubleValue());
            case SLASH -> checkNumbers(left, right,
                (l, r) -> l.doubleValue() / r.doubleValue());
            case EQUAL_EQUAL -> isEqual(left, right);
            case BANG_EQUAL -> !isEqual(left, right);
            case LESS -> checkNumbers(left, right,
                (l, r) -> l.doubleValue() < r.doubleValue());
            // ... 기타 연산자
        };
    }

    @Override
    public Object visitIfStatement(IfStatement stmt) {
        if (isTruthy(stmt.getCondition().accept(this))) {
            return stmt.getThenBranch().accept(this);
        } else if (stmt.getElseBranch() != null) {
            return stmt.getElseBranch().accept(this);
        }
        return null;
    }

    @Override
    public Object visitCallExpression(CallExpression expr) {
        Object callee = expr.getCallee().accept(this);

        List<Object> arguments = new ArrayList<>();
        for (Expression arg : expr.getArguments()) {
            arguments.add(arg.accept(this));
        }

        if (!(callee instanceof Callable)) {
            throw new RuntimeError("Can only call functions and classes.");
        }

        Callable function = (Callable) callee;
        if (arguments.size() != function.arity()) {
            throw new RuntimeError("Expected " + function.arity() +
                " arguments but got " + arguments.size() + ".");
        }

        return function.call(this, arguments);
    }
}
```

### AST 변환

```java
// AST 변환 (예: 상수 폴딩)
public class ConstantFolder implements ASTVisitor<ASTNode> {

    @Override
    public ASTNode visitBinaryExpression(BinaryExpression expr) {
        Expression left = (Expression) expr.getLeft().accept(this);
        Expression right = (Expression) expr.getRight().accept(this);

        // 둘 다 리터럴이면 계산
        if (left instanceof Literal && right instanceof Literal) {
            Object leftVal = ((Literal) left).getValue();
            Object rightVal = ((Literal) right).getValue();

            if (leftVal instanceof Number && rightVal instanceof Number) {
                double result = evaluate(
                    ((Number) leftVal).doubleValue(),
                    ((Number) rightVal).doubleValue(),
                    expr.getOperator()
                );
                return new Literal(result, LiteralType.NUMBER);
            }
        }

        return new BinaryExpression(left, expr.getOperator(), right);
    }

    // 1 + 2 * 3 → 7 (컴파일 타임에 계산)
}

// 탈당화 (Desugaring) 변환
public class Desugarer implements ASTVisitor<ASTNode> {

    @Override
    public ASTNode visitForStatement(ForStatement stmt) {
        // for (init; cond; update) body
        // → { init; while (cond) { body; update; } }

        List<Statement> bodyStatements = new ArrayList<>();
        bodyStatements.add((Statement) stmt.getBody().accept(this));
        bodyStatements.add(new ExpressionStatement(stmt.getUpdate()));

        WhileStatement whileStmt = new WhileStatement(
            stmt.getCondition(),
            new BlockStatement(bodyStatements)
        );

        List<Statement> outerStatements = new ArrayList<>();
        outerStatements.add(stmt.getInitializer());
        outerStatements.add(whileStmt);

        return new BlockStatement(outerStatements);
    }

    // x += 1 → x = x + 1
    @Override
    public ASTNode visitAssignmentExpression(AssignmentExpression expr) {
        if (expr.getOperator() != ASSIGN) {
            // 복합 할당 연산자를 기본 형태로
            BinaryOperator binOp = compoundToBinary(expr.getOperator());
            Expression newValue = new BinaryExpression(
                expr.getTarget(),
                binOp,
                expr.getValue()
            );
            return new AssignmentExpression(
                expr.getTarget(),
                ASSIGN,
                newValue
            );
        }
        return expr;
    }
}
```

### 소스 위치 정보

```java
// 에러 메시지와 디버깅을 위한 위치 정보
public class SourceLocation {
    private final String filename;
    private final int line;
    private final int column;
    private final int startOffset;  // 문자 오프셋
    private final int endOffset;

    // 범위 정보 (여러 토큰에 걸친 노드)
    public static SourceLocation span(SourceLocation start, SourceLocation end) {
        return new SourceLocation(
            start.filename,
            start.line,
            start.column,
            start.startOffset,
            end.endOffset
        );
    }
}

// 위치 정보를 포함한 AST 노드
public abstract class ASTNode {
    protected SourceLocation location;

    public void setLocation(SourceLocation location) {
        this.location = location;
    }

    public SourceLocation getLocation() {
        return location;
    }
}

// 파서에서 위치 정보 설정
public Expression parseExpression() {
    Token start = currentToken;
    Expression left = parseTerm();

    while (match(PLUS, MINUS)) {
        Token operator = previous();
        Expression right = parseTerm();
        BinaryExpression expr = new BinaryExpression(left, toBinaryOp(operator), right);

        // 왼쪽 피연산자 시작부터 오른쪽 피연산자 끝까지
        expr.setLocation(SourceLocation.span(
            left.getLocation(),
            right.getLocation()
        ));

        left = expr;
    }

    return left;
}
```

### AST 출력과 시각화

```java
// AST를 문자열로 출력 (S-expression 형태)
public class ASTPrinter implements ASTVisitor<String> {
    private int indent = 0;

    @Override
    public String visitBinaryExpression(BinaryExpression expr) {
        return parenthesize(expr.getOperator().toString(),
            expr.getLeft(), expr.getRight());
    }

    @Override
    public String visitLiteral(Literal expr) {
        return expr.getValue().toString();
    }

    @Override
    public String visitIfStatement(IfStatement stmt) {
        StringBuilder sb = new StringBuilder();
        sb.append(indentation()).append("(if ");
        sb.append(stmt.getCondition().accept(this)).append("\n");
        indent++;
        sb.append(indentation()).append(stmt.getThenBranch().accept(this));
        if (stmt.getElseBranch() != null) {
            sb.append("\n").append(indentation())
              .append(stmt.getElseBranch().accept(this));
        }
        indent--;
        sb.append(")");
        return sb.toString();
    }

    private String parenthesize(String name, ASTNode... nodes) {
        StringBuilder sb = new StringBuilder();
        sb.append("(").append(name);
        for (ASTNode node : nodes) {
            sb.append(" ").append(node.accept(this));
        }
        sb.append(")");
        return sb.toString();
    }

    private String indentation() {
        return "  ".repeat(indent);
    }
}

// 사용
// 1 + 2 * 3 → (+ 1 (* 2 3))
// if (x > 0) { return x; } else { return -x; }
// → (if (> x 0)
//     (return x)
//     (return (- x)))
```

## 실무 적용

### TypeScript 컴파일러의 AST

```typescript
// TypeScript AST 예시 (ts.Node)
import * as ts from "typescript";

// 소스 코드를 AST로 파싱
const sourceFile = ts.createSourceFile(
    "example.ts",
    `function greet(name: string): string {
        return "Hello, " + name;
    }`,
    ts.ScriptTarget.Latest,
    true
);

// AST 순회
function visit(node: ts.Node) {
    if (ts.isFunctionDeclaration(node)) {
        console.log("Function:", node.name?.getText());
        node.parameters.forEach(param => {
            console.log("  Param:", param.name.getText());
            if (param.type) {
                console.log("  Type:", param.type.getText());
            }
        });
    }
    ts.forEachChild(node, visit);
}

visit(sourceFile);

// AST 변환 (Transformer)
const transformer: ts.TransformerFactory<ts.SourceFile> = (context) => {
    return (sourceFile) => {
        const visitor: ts.Visitor = (node) => {
            // console.log 호출을 제거
            if (ts.isCallExpression(node) &&
                ts.isPropertyAccessExpression(node.expression) &&
                node.expression.name.getText() === "log") {
                return ts.factory.createVoidZero(); // void 0으로 대체
            }
            return ts.visitEachChild(node, visitor, context);
        };
        return ts.visitNode(sourceFile, visitor) as ts.SourceFile;
    };
};
```

### Babel의 AST (JavaScript)

```javascript
// Babel AST 사용
const parser = require("@babel/parser");
const traverse = require("@babel/traverse").default;
const generate = require("@babel/generator").default;
const t = require("@babel/types");

// 파싱
const code = `const square = (n) => n * n;`;
const ast = parser.parse(code, {
    sourceType: "module",
    plugins: ["jsx", "typescript"]
});

// 순회 및 변환
traverse(ast, {
    // 화살표 함수를 일반 함수로 변환
    ArrowFunctionExpression(path) {
        const { params, body } = path.node;

        // 표현식 바디를 블록으로 변환
        const newBody = t.isBlockStatement(body)
            ? body
            : t.blockStatement([t.returnStatement(body)]);

        // FunctionExpression으로 교체
        path.replaceWith(
            t.functionExpression(null, params, newBody)
        );
    },

    // const를 var로 변환
    VariableDeclaration(path) {
        if (path.node.kind === "const" || path.node.kind === "let") {
            path.node.kind = "var";
        }
    }
});

// 코드 생성
const output = generate(ast, {}, code);
console.log(output.code);
// var square = function (n) { return n * n; };
```

### ESLint 규칙 작성

```javascript
// ESLint 커스텀 규칙 (no-console)
module.exports = {
    meta: {
        type: "suggestion",
        docs: {
            description: "disallow console statements",
            category: "Best Practices"
        },
        fixable: "code",
        schema: []
    },

    create(context) {
        return {
            // console.* 호출 탐지
            CallExpression(node) {
                if (node.callee.type === "MemberExpression" &&
                    node.callee.object.name === "console") {

                    context.report({
                        node,
                        message: "Unexpected console statement.",
                        fix(fixer) {
                            // 자동 수정: 문장 전체 삭제
                            return fixer.remove(node.parent);
                        }
                    });
                }
            },

            // var 사용 금지
            VariableDeclaration(node) {
                if (node.kind === "var") {
                    context.report({
                        node,
                        message: "Use let or const instead of var.",
                        fix(fixer) {
                            return fixer.replaceTextRange(
                                [node.start, node.start + 3],
                                "let"
                            );
                        }
                    });
                }
            }
        };
    }
};
```

### IDE 기능 구현

```java
// 코드 완성 (Autocomplete)
public class AutocompleteProvider {
    public List<Suggestion> getSuggestions(AST ast, Position cursor) {
        // 커서 위치의 노드 찾기
        ASTNode nodeAtCursor = findNodeAtPosition(ast, cursor);

        // 컨텍스트에 따른 제안
        if (nodeAtCursor instanceof MemberExpression) {
            // obj. 뒤에 커서 → obj의 멤버 제안
            MemberExpression member = (MemberExpression) nodeAtCursor;
            Type objType = typeChecker.getType(member.getObject());
            return objType.getMembers().stream()
                .map(m -> new Suggestion(m.getName(), m.getType()))
                .collect(Collectors.toList());
        }

        if (nodeAtCursor instanceof Identifier) {
            // 스코프 내 변수/함수 제안
            Scope scope = scopeAnalyzer.getScopeAt(cursor);
            return scope.getAllSymbols().stream()
                .filter(s -> s.getName().startsWith(((Identifier) nodeAtCursor).getName()))
                .map(s -> new Suggestion(s.getName(), s.getType()))
                .collect(Collectors.toList());
        }

        return Collections.emptyList();
    }
}

// 리팩토링: 이름 변경
public class RenameRefactoring {
    public List<TextEdit> rename(AST ast, Position position, String newName) {
        // 커서 위치의 심볼 찾기
        Symbol symbol = findSymbolAtPosition(ast, position);
        if (symbol == null) return Collections.emptyList();

        List<TextEdit> edits = new ArrayList<>();

        // 모든 참조 찾기
        List<Identifier> references = findAllReferences(ast, symbol);
        for (Identifier ref : references) {
            edits.add(new TextEdit(ref.getLocation(), newName));
        }

        return edits;
    }
}
```

## 참고 자료

- Appel, Andrew - "Modern Compiler Implementation"
- Bob Nystrom - "Crafting Interpreters"
- [AST Explorer](https://astexplorer.net/) - 다양한 파서의 AST 시각화
- [Babel Handbook](https://github.com/jamiebuilds/babel-handbook)
- [TypeScript AST Viewer](https://ts-ast-viewer.com/)
- [ESLint Developer Guide](https://eslint.org/docs/developer-guide/)
