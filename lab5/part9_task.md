# Part ⑨ 全局变量

在 Part ⑨ 中，你的编译器需要支持全局变量。

你需要支持的语法规则如下（以 `CompUnit` 为开始符号）：

```rust
CompUnit     -> Decl* FuncDef // [changed]
Decl         -> ConstDecl | VarDecl
ConstDecl    -> 'const' BType ConstDef { ',' ConstDef } ';'
BType        -> 'int'
ConstDef     -> Ident '=' ConstInitVal
ConstInitVal -> ConstExp
ConstExp     -> AddExp
VarDecl      -> BType VarDef { ',' VarDef } ';'
VarDef       -> Ident
                | Ident '=' InitVal
InitVal      -> Exp
FuncDef      -> FuncType Ident '(' ')' Block // 保证当前 Ident 只为 "main"
FuncType     -> 'int'
Block        -> '{' { BlockItem } '}'
BlockItem    -> Decl | Stmt
Stmt         -> LVal '=' Exp ';'
                | Block
                | [Exp] ';'
                | 'if' '(' Cond ')' Stmt [ 'else' Stmt ]
                | 'return' Exp ';'
Exp          -> AddExp
Cond         -> LOrExp
LVal         -> Ident
PrimaryExp   -> '(' Exp ')' | LVal | Number
UnaryExp     -> PrimaryExp
                | Ident '(' [FuncRParams] ')'
                | UnaryOp UnaryExp
UnaryOp      -> '+' | '-' | '!'  // 保证 '!' 只出现在 Cond 中
FuncRParams  -> Exp { ',' Exp }
MulExp       -> UnaryExp
                | MulExp ('*' | '/' | '%') UnaryExp
AddExp       -> MulExp
                | AddExp ('+' | '-') MulExp
RelExp       -> AddExp
                | RelExp ('<' | '>' | '<=' | '>=') AddExp
EqExp        -> RelExp
                | EqExp ('==' | '!=') RelExp
LAndExp      -> EqExp
                | LAndExp '&&' EqExp
LOrExp       -> LAndExp
                | LOrExp '||' LAndExp
```

## 语义约束

- 全局变量声明中指定的初值表达式必须是**常量表达式**；
- 全局变量之间不允许同名；
- 局部变量可以和全局变量同名，在局部变量作用域内，局部变量隐藏了同名全局变量的定义；
- 未显式初始化的局部变量，其值是不确定的；而未显式初始化的全局变量，其值均被初始化为 0。

## 示例

### 样例 1

样例程序 1：

```c
int a = 5;
int main() {
    int b = getint();
    putint(a + b);
    return 0;
}
```

示例 IR 1：

```llvm
@a = dso_local global i32 5
declare i32 @getint()
declare void @putint(i32)
define dso_local i32 @main() {
    %1 = alloca i32
    %2 = call i32 @getint()
    store i32 %2, i32* %1
    %3 = load i32, i32* @a
    %4 = load i32, i32* %1
    %5 = add i32 %3, %4
    call void @putint(i32 %5)
    ret i32 0
}
```

输入样例 1：

```c
4
```

输出样例 1：

```c
9
```

### 样例 2

样例程序 2：

```c
const int a = 6;
int b = a + 1;
int main() {
    int c = b;
    int b = 8;
    putint(b + c);
    return 0;
}
```

示例 IR 2：

```llvm
@b = dso_local global i32 7
declare i32 @getint()
declare void @putint(i32)
define dso_local i32 @main() {
    %1 = alloca i32
    %2 = alloca i32
    %3 = load i32, i32* @b
    store i32 %3, i32* %2
    store i32 8, i32* %1
    %4 = load i32, i32* %1
    %5 = load i32, i32* %2
    %6 = add i32 %4, %5
    call void @putint(i32 %6)
    ret i32 0
}
```


输出样例 2：

```c
15
```

### 样例 3

样例程序 3：

```c
int a = 6;
int b = a + 1;
int main() {
    int c = b;
    int b = 8;
    putint(b + c);
    return 0;
}
```

输出样例 3：

编译器直接以**非 0 的返回值**退出。
