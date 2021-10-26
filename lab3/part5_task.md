# Part 5 局部变量与赋值

~~终于可以支持一个像样的程序了~~

在 Part 5 中，你的编译器需要增加对局部变量的支持（当然也包括局部常量）。

你需要支持的语法规则如下（以 `CompUnit` 为开始符号）：

```rust
CompUnit     -> FuncDef
Decl         -> ConstDecl | VarDecl
ConstDecl    -> 'const' BType ConstDef { ',' ConstDef } ';'
BType        -> 'int'
ConstDef     -> Ident '=' ConstInitVal
ConstInitVal -> ConstExp
ConstExp     -> AddExp
VarDecl      -> BType VarDef { ',' VarDef } ';'
VarDef       -> Ident 
                | Ident '=' InitVal;
InitVal      -> Exp 
FuncDef      -> FuncType Ident '(' ')' Block
FuncType     -> 'int'
Ident        -> 'main'
Block        -> '{' { BlockItem } '}'
BlockItem    -> Decl | Stmt
Stmt         -> LVal '=' Exp ';' 
                | [Exp] ';'
                | 'return' Exp ';'
LVal         -> Ident
Exp          -> AddExp
AddExp       -> MulExp 
                | AddExp ('+' | '−') MulExp
MulExp       -> UnaryExp
                | MulExp ('*' | '/' | '%') UnaryExp
UnaryExp     -> PrimaryExp | UnaryOp UnaryExp
PrimaryExp   -> '(' Exp ')' | LVal | Number
UnaryOp      -> '+' | '-'
```

- 标识符 `Ident` 的定义

```rust
Ident    -> Nondigit
            | Ident Nondigit
            | Ident Digit
Nondigit -> '_' | 'a' | 'b' | ... | 'z' | 'A' | 'B' | ... | 'Z'
Digit    -> '0' | '1' | ... | '9'
```

**对于同名标识符的规定**：
- 全局变量和局部变量的作用域可以重叠，重叠部分局部变量优先；
- 同名局部变量的作用域不能重叠；
- 变量名可以与函数名相同。

## 语义约束

### `ConstInitVal`

- `ConstInitVal` 中的 `ConstExp` 必须是能在编译时求值的 `int` 型表达式，其中可以引用已定义的**常量**。

### `VarDef`

- `VarDef` 用于定义变量。当不含有 `=` 和初始值时，其运行时实际初值未定义。
- 当 `VarDef` 含有 `=` 和初始值时， `=` 右边的 `InitVal` 和 `ConstInitVal` 的结构要求相同，唯一的不同是 `ConstInitVal` 中的表达式是 `ConstExp` 常量表达式，而 `InitVal` 中的表达式可以是当前上下文合法的任何 `Exp`。

### `Block`

- `Block` 内不能有同名的变量或常量。

### `Stmt`

- 单个 `Exp` 可以作为 `Stmt`。该 `Exp` 会被求值，所求的值会被丢弃。

### `LVal`

- 赋值号左边的 `LVal` 必须是变量；`Exp` 中的 `LVal` 必须是当前作用域内、该 `Exp` 语句之前有定义的变量或常量。

## 示例

### 样例 1

样例程序 1：

```c
int main() {
    int a = 123 - 122;
    return a;
}
```

示例 IR 1：

```llvm
define dso_local i32 @main(){
    %1 = alloca i32
    %2 = sub i32 123, 122
    store i32 %2, i32* %1
    %3 = load i32, i32* %1
    ret i32 %3
}
```

输出样例 1：

```c
1
```

### 样例 2

样例程序 2：

```c
int main() {
    const int Nqn7m1 = 010;
    int yiersan = 456;
    int mAgIc_NuMbEr;
    mAgIc_NuMbEr = 8456;
    int a1a11a11 = (mAgIc_NuMbEr - yiersan) / 1000 - Nqn7m1, _CHAOS_TOKEN;
    _CHAOS_TOKEN = 2;
    a1a11a11 = a1a11a11 + _CHAOS_TOKEN;
    return a1a11a11 - _CHAOS_TOKEN + 000;
}
```

示例 IR 2：

```llvm
define dso_local i32 @main(){
    %1 = alloca i32
    %2 = alloca i32
    %3 = alloca i32
    %4 = alloca i32
    store i32 456, i32* %4
    store i32 8456, i32* %3
    %5 = load i32, i32* %3
    %6 = load i32, i32* %4
    %7 = sub i32 %5, %6
    %8 = sdiv i32 %7, 1000
    %9 = sub i32 %8, 8
    store i32 %9, i32* %2
    store i32 2, i32* %1
    %10 = load i32, i32* %2
    %11 = load i32, i32* %1
    %12 = add i32 %10, %11
    store i32 %12, i32* %2
    %13 = load i32, i32* %2
    %14 = load i32, i32* %1
    %15 = sub i32 %13, %14
    %16 = add i32 %15, 0
    ret i32 %16
}

```

输出样例 2:

```c
0
```

### 样例 3

样例程序 3：

```c
int main() {
    const int sudo = 0;
    int rm = 5, r = 3, home = 5;
    sudo = rm -r /home*       0;
    return sudo;
}
```

输出样例 3：

编译器直接以**非 0 的返回值**退出。（赋值号左边的 `LVal` 必须是变量）