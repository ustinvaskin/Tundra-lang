# Formal Grammar and AST Draft for Tundra 0.1

This grammar is based on the current Tundra 0.1 spec.

## Note

This scanner is intentionally small. It recognizes only the syntax needed by Tundra 0.1 and leaves meaning to later phases. It does not know what `origin` or `history` mean. It only recognizes them as identifiers.

That is important because provenance is a runtime/debugging idea, not special syntax.

## Tundra 0.1 includes

- semicolon-terminated statements
- `val` and `var`
- named functions with `def`
- explicit `return`
- parenthesized `if`, `while`, and `for`
- lists
- records
- field access
- indexing
- function calls
- multiline strings
- `origin(value)` and `history(value)` as normal function calls

## Tundra 0.1 does not include

- classes
- `this`
- `super`
- schemas
- anonymous functions
- optional semicolons
- final-expression returns
- field assignment
- index assignment
- escape sequences
- string interpolation

# Grammar — EBNF

```ebnf
program        → statement* EOF ;

// Statements
statement      → varDecl
               | valDecl
               | funDecl
               | ifStmt
               | whileStmt
               | forStmt
               | returnStmt
               | block
               | exprStmt ;

varDecl        → "var" IDENTIFIER "=" expression ";" ;
valDecl        → "val" IDENTIFIER "=" expression ";" ;

funDecl        → "def" IDENTIFIER "(" params? ")" block ;
params         → IDENTIFIER ( "," IDENTIFIER )* ;

ifStmt         → "if" "(" expression ")" block ( "else" block )? ;
whileStmt      → "while" "(" expression ")" block ;
forStmt        → "for" "(" IDENTIFIER "in" expression ")" block ;

returnStmt     → "return" expression? ";" ;
exprStmt       → expression ";" ;

block          → "{" statement* "}" ;

// Expressions, lowest precedence to highest precedence
expression     → assignment ;

assignment     → IDENTIFIER "=" assignment
               | logic_or ;

logic_or       → logic_and ( "or" logic_and )* ;
logic_and      → equality ( "and" equality )* ;

equality       → comparison ( ( "==" | "!=" ) comparison )* ;
comparison     → addition ( ( "<" | "<=" | ">" | ">=" ) addition )* ;
addition       → multiply ( ( "+" | "-" ) multiply )* ;
multiply       → unary ( ( "*" | "/" ) unary )* ;

unary          → ( "!" | "-" ) unary
               | postfix ;

postfix        → primary ( "(" args? ")" | "." IDENTIFIER | "[" expression "]" )* ;
args           → expression ( "," expression )* ;

// Primary expressions
primary        → NUMBER
               | STRING
               | "true"
               | "false"
               | "none"
               | IDENTIFIER
               | "(" expression ")"
               | listLiteral
               | recordLiteral ;

listLiteral    → "[" ( expression ( "," expression )* )? "]" ;
recordLiteral  → "{" ( field ( "," field )* )? "}" ;
field          → IDENTIFIER ":" expression ;
```

# Token Types Needed

Because records use `:`, Tundra needs a `COLON` token.  
Because lists and indexing use `[]`, Tundra needs `LEFT_BRACKET` and `RIGHT_BRACKET`.  
Because strings may span multiple lines, the scanner must keep counting line numbers while scanning a `STRING`.

```java
package tundra;

public enum TokenType {
    // Delimiters
    LEFT_PAREN, RIGHT_PAREN,
    LEFT_BRACE, RIGHT_BRACE,
    LEFT_BRACKET, RIGHT_BRACKET,
    COMMA, DOT, SEMICOLON, COLON,

    // Operators
    PLUS, MINUS, STAR, SLASH,
    BANG, BANG_EQUAL,
    EQUAL, EQUAL_EQUAL,
    GREATER, GREATER_EQUAL,
    LESS, LESS_EQUAL,

    // Literals
    IDENTIFIER, STRING, NUMBER,

    // Keywords
    AND, OR,
    IF, ELSE, WHILE, FOR, IN,
    DEF, RETURN,
    TRUE, FALSE, NONE,
    VAL, VAR,

    EOF
}
```

There is no `CLASS`, `THIS`, or `SUPER` in Tundra 0.1.

# AST Node Definitions

## Program

```text
Program
  statements: Stmt[]
```

A program is just a list of statements ending at `EOF`.

# Statements

## Block

```text
Block
  statements: Stmt[]
```

A block introduces a new lexical scope.

Example:

```tundra
{
  val x = 1;
  print(x);
}
```

## VarDecl

Used for both `val` and `var`.

```text
VarDecl
  name: Token
  mutable: Boolean
  initializer: Expr
```

Examples:

```tundra
val x = 1;
var y = 2;
```

`mutable` is:

- `false` for `val`
- `true` for `var`

## FunDecl

```text
FunDecl
  name: Token
  params: Token[]
  body: Block
```

Example:

```tundra
def sum(a, b) {
  return a + b;
}
```

## IfStmt

```text
IfStmt
  condition: Expr
  thenBranch: Block
  elseBranch: Block?
```

Example:

```tundra
if (ready) {
  print("ready");
} else {
  print("not ready");
}
```

## WhileStmt

```text
WhileStmt
  condition: Expr
  body: Block
```

Example:

```tundra
while (index < length(items)) {
  print(items[index]);
  index = index + 1;
}
```

## ForStmt

```text
ForStmt
  variable: Token
  iterable: Expr
  body: Block
```

Example:

```tundra
for (line in lines) {
  print(line);
}
```

Later, this can be desugared into a lower-level loop.

## ReturnStmt

```text
ReturnStmt
  keyword: Token
  value: Expr?
```

Examples:

```tundra
return value;
return;
```

A bare `return;` returns `none`.

The `keyword` token is useful for error reporting, especially if `return` appears outside a function.

## ExprStmt

```text
ExprStmt
  expr: Expr
```

Example:

```tundra
print("hello");
```

# Expressions

## Literal

```text
Literal
  value: Number | String | Boolean | None
```

Examples:

```tundra
123
12.34
"hello"
"hello
world"
true
false
none
```

Multiline strings are still just `STRING` literals. The parser does not need special handling for them because the scanner already turns the whole multiline string into one token.

## Variable

```text
Variable
  name: Token
```

Example:

```tundra
price
```

## Assign

```text
Assign
  name: Token
  value: Expr
```

Example:

```tundra
item = "forty-two";
```

For Tundra 0.1, assignment only works on variables.

These are not supported yet:

```tundra
person.name = "Ada";
items[0] = 42;
```

Those can come later.

## Unary

```text
Unary
  operator: Token
  operand: Expr
```

Examples:

```tundra
!done
-price
```

## Binary

```text
Binary
  left: Expr
  operator: Token
  right: Expr
```

Examples:

```tundra
a + b
price <= limit
x == y
```

## Logical

Separate from `Binary` because `and` and `or` short-circuit.

```text
Logical
  left: Expr
  operator: Token
  right: Expr
```

Examples:

```tundra
ready and valid
left or right
```

## Call

```text
Call
  callee: Expr
  paren: Token
  args: Expr[]
```

Example:

```tundra
parseNumber(text)
```

`paren` stores the closing `)` token for better error reporting.

## FieldAccess

```text
FieldAccess
  object: Expr
  name: Token
```

Example:

```tundra
order.price
```

## Index

```text
Index
  object: Expr
  bracket: Token
  index: Expr
```

Example:

```tundra
items[0]
```

`bracket` stores the closing `]` token for better error reporting.

## ListLiteral

```text
ListLiteral
  elements: Expr[]
```

Examples:

```tundra
[1, 2, 3]
["red", "green", "blue"]
[]
```

## RecordLiteral

```text
RecordLiteral
  fields: RecordField[]
```

Example:

```tundra
{
  id: "A12",
  item: "Tea",
  price: 4.50
}
```

## RecordField

```text
RecordField
  name: Token
  value: Expr
```

Example field:

```tundra
price: 4.50
```

# Important Notes

## `origin` and `history`

`origin` and `history` are not special AST nodes. They parse as normal function calls:

```tundra
origin(total);
history(total);
```

AST shape:

```text
Call(
  callee = Variable("history"),
  args = [Variable("total")]
)
```

The interpreter handles them as built-in functions when debug mode is enabled.

## Records and blocks both use `{}`

This is okay. The parser can tell the difference from context.

After `if (...)`, `{ ... }` means a block:

```tundra
if (ready) {
  print("ready");
}
```

After `=`, `{ ... }` means a record expression:

```tundra
val order = {
  id: "A12",
  price: 4.50
};
```

## Assignment is right-associative

This grammar:

```ebnf
assignment → IDENTIFIER "=" assignment
           | logic_or ;
```

means this:

```tundra
a = b = 3;
```

parses as:

```text
a = (b = 3)
```

That is standard assignment behavior.

## `Logical` is separate from `Binary`

This matters because `and` and `or` short-circuit.

For example:

```tundra
ready and expensiveCheck()
```

If `ready` is false, `expensiveCheck()` should not run. That means the interpreter must evaluate logical expressions differently from normal binary expressions.

## `val` and `var` share one AST node

Instead of having separate nodes:

```text
ValDecl
VarDecl
```

Tundra can use one node:

```text
VarDecl
  mutable: Boolean
```

That keeps the interpreter simpler.

## Multiline strings are handled by the scanner

The grammar does not need a special multiline string rule.

This:

```tundra
val text = "hello
world";
```

still parses as:

```text
VAL IDENTIFIER EQUAL STRING SEMICOLON
```

The scanner already produced one `STRING` token containing the newline.
