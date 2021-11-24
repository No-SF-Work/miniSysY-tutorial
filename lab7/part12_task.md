# Part 12 一维数组、二维数组

在 Part 12 中，你的编译器需要支持一维数组和二维数组。

你需要支持的语法规则如下（以 `CompUnit` 为开始符号）：

```rust
CompUnit     -> Decl* FuncDef
Decl         -> ConstDecl | VarDecl
ConstDecl    -> 'const' BType ConstDef { ',' ConstDef } ';'
BType        -> 'int'
ConstDef     -> Ident { '[' ConstExp ']' } '=' ConstInitVal // [changed]
ConstInitVal -> ConstExp
                | '{' [ ConstInitVal { ',' ConstInitVal } ] '}' // [changed]
ConstExp     -> AddExp
VarDecl      -> BType VarDef { ',' VarDef } ';'
VarDef       -> Ident { '[' ConstExp ']' }
                | Ident { '[' ConstExp ']' } '=' InitVal // [changed]
InitVal      -> Exp
                | '{' [ InitVal { ',' InitVal } ] '}' // [changed]
FuncDef      -> FuncType Ident '(' ')' Block // 保证当前 Ident 只为 "main"
FuncType     -> 'int'
Block        -> '{' { BlockItem } '}'
BlockItem    -> Decl | Stmt
Stmt         -> LVal '=' Exp ';'
                | Block
                | [Exp] ';'
                | 'if' '(' Cond ')' Stmt [ 'else' Stmt ]
                | 'while' '(' Cond ')' Stmt
                | 'break' ';'
                | 'continue' ';'
                | 'return' Exp ';'
Exp          -> AddExp
Cond         -> LOrExp
LVal         -> Ident {'[' Exp ']'} // [changed]
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

### `ConstDef`

- `ConstDef` 的数组维度和各维长度的定义部分存在时，表示定义数组。其语义和 C 语言一致。比如 `[2 * 3][8 / 2]` 表示二维数组，第一和第二维长度分别为 6 和 4，每维的下界从 0 编号。 `ConstDef` 中表示各维长度的 `ConstExp` 必须是能在编译时求值到非负整数的常量表达式。在声明数组时各维长度都需要显式给出，而不允许是未知的。
- 当 `ConstDef` 定义的是数组时，`=` 右边的 `ConstInitVal` 表示常量初始化器。全局常量数组的 `ConstInitVal` 中的 `ConstExp` 必须是常量表达式。局部常量数组的 `ConstInitVal` 中的 `ConstExp` 必须是能在编译时求值的 `int` 型表达式。
- `ConstInitVal` 初始化器必须是以下三种情况之一：
  - 一对花括号 `{}`，表示所有元素初始为 0；
  - 数组维数和各维长度完全对应的初始值，如 `int a[3] = {1, 2, 3};`、`int a[3][2] = { {1, 2}, {3, 4}, {5, 6} };`；
  - 花括号中初始值少于对应维度元素个数，该维其余部分将被隐式初始化为 0，如 `int a[5] = {1, 2};`、`int a[4][4] = { {1, 2, 3}, {4, 5}, {} };`。

### `VarDef`

- `VarDef` 的数组维度和各维长度的定义部分存在时，表示定义数组。其语义和 C 语言一致。`VarDef` 中表示各维长度的 `ConstExp` 必须能在编译时求值到非负整数的常量表达式。在声明数组时各维长度都需要显式给出，而不允许是未知的。
- 全局变量数组的 `InitVal` 中的 `Exp` 必须是常量表达式。局部变量数组 `InitVal` 中的 `Exp` 可以是任何符合语义的表达式。

### `LVal`

当 `LVal` 表示数组时，方括号个数必须和数组变量的维数相同（即定位到元素）。

### 初值的常量/可求值总结

> “编译时可求值”约束为常数和 `int` 类型变量/常量所构成的表达式，且不包括数组元素、函数返回值。
> 在评测时只会针对表达式是否满足**常量**要求进行评测。

- 全局 `int` 类型变量/常量的初值必须是编译时可求值的常量表达式。
- 局部 `int` 类型常量的初值必须是编译时可求值的表达式。（和 C 语言略有不同）
- 数组的各维长度必须是编译时可求值的非负常量表达式。
- 全局数组的 `ConstInitVal/InitVal` 中的 `ConstExp/Exp` 必须是编译时可求值的常量表达式。
- 局部常量数组的 `ConstInitVal` 中的 `ConstExp` 必须是编译时可求值的表达式。（和 C 语言略有不同）

### 其他

数组下标越界是未定义行为，不作为编译错误考察。

## LLVM IR 中数组的初始化

> 你可以不采用以下介绍的实现方式，自己编写测试代码并使用 clang 编译到 LLVM IR，模仿实现 clang 生成的 LLVM IR 中的数组初始化方式。

- 局部数组：调用 C 语言库函数 `memset(pointer, 0, size * sizeof(int))` 将数组元素全部置为 0，其中 `pointer` 为指向数组基址的指针，`size` 为数组容量，`sizeof(int)` 为 4。然后使用 `store` 指令将初始化器中的元素存入数组的对应位置。
  - 评测机已对 `memset` 提供支持，你可以在 IR 中声明后直接调用。
- 全局数组：在全局区进行相应的声明和初始化。你可以使用如下的语法对一维数组进行初始化：`@arr = dso_local global [3 x i32] [i32 1, i32 2, i32 3]`。对于二维数组，你需要对其中包含的每一个一维数组进行初始化：`@arr = dso_local global [2 x [2 x i32]] [[2 x i32] [i32 1, i32 2], [2 x i32] [i32 3, i32 0]]`。你可以使用 `zeroinitializer` 将一个数组中的元素全部置为 0。

## 示例

### 样例 1

样例程序 1：

```cpp
int main() {
    int a[2][2] = {{1}, {2, 3}};
    int e[2][2] = {{a[0][0], a[1][1]}, {5, 6}};
    putint(e[1][1] + a[1][0]);
    return 0;
}
```

示例 IR 1：

```llvm
declare void @putint(i32)
declare void @memset(i32*, i32, i32)
define dso_local i32 @main() {
    %1 = alloca [2 x [2 x i32]]
    %2 = alloca [2 x [2 x i32]]
    %3 = getelementptr [2 x [2 x i32]], [2 x [2 x i32]]* %2, i32 0, i32 0
    %4 = getelementptr [2 x i32], [2 x i32]* %3, i32 0, i32 0
    call void @memset(i32* %4, i32 0, i32 16)
    store i32 1, i32* %4
    %5 = getelementptr i32, i32* %4, i32 2
    store i32 2, i32* %5
    %6 = getelementptr i32, i32* %4, i32 3
    store i32 3, i32* %6
    %7 = getelementptr [2 x [2 x i32]], [2 x [2 x i32]]* %2, i32 0, i32 0
    %8 = add i32 0, 0
    %9 = mul i32 %8, 2
    %10 = getelementptr [2 x i32], [2 x i32]* %7, i32 0, i32 0
    %11 = add i32 %9, 0
    %12 = getelementptr i32, i32* %10, i32 %11
    %13 = load i32, i32* %12
    %14 = getelementptr [2 x [2 x i32]], [2 x [2 x i32]]* %2, i32 0, i32 0
    %15 = add i32 0, 1
    %16 = mul i32 %15, 2
    %17 = getelementptr [2 x i32], [2 x i32]* %14, i32 0, i32 0
    %18 = add i32 %16, 1
    %19 = getelementptr i32, i32* %17, i32 %18
    %20 = load i32, i32* %19
    %21 = getelementptr [2 x [2 x i32]], [2 x [2 x i32]]* %1, i32 0, i32 0
    %22 = getelementptr [2 x i32], [2 x i32]* %21, i32 0, i32 0
    call void @memset(i32* %22, i32 0, i32 16)
    store i32 %13, i32* %22
    %23 = getelementptr i32, i32* %22, i32 1
    store i32 %20, i32* %23
    %24 = getelementptr i32, i32* %22, i32 2
    store i32 5, i32* %24
    %25 = getelementptr i32, i32* %22, i32 3
    store i32 6, i32* %25
    %26 = getelementptr [2 x [2 x i32]], [2 x [2 x i32]]* %1, i32 0, i32 0
    %27 = add i32 0, 1
    %28 = mul i32 %27, 2
    %29 = getelementptr [2 x i32], [2 x i32]* %26, i32 0, i32 0
    %30 = add i32 %28, 1
    %31 = getelementptr i32, i32* %29, i32 %30
    %32 = load i32, i32* %31
    %33 = getelementptr [2 x [2 x i32]], [2 x [2 x i32]]* %2, i32 0, i32 0
    %34 = add i32 0, 1
    %35 = mul i32 %34, 2
    %36 = getelementptr [2 x i32], [2 x i32]* %33, i32 0, i32 0
    %37 = add i32 %35, 0
    %38 = getelementptr i32, i32* %36, i32 %37
    %39 = load i32, i32* %38
    %40 = add i32 %32, %39
    call void @putint(i32 %40)
    ret i32 0
}
```

输出样例 1：

```c
8
```

### 样例 2

样例程序 2：

```cpp
const int c[2][1] = {{1}, {3}};
int b[2][3] = {{1}}, e[4][4];
int d[5], a[3] = {1, 2};
int main() {
    putint(c[1][0] + b[0][0] + c[0][0] + a[1] + d[4]);
    return 0;
}
```

示例 IR 2：

```llvm
declare void @memset(i32*  ,i32 ,i32 )
declare void @putint(i32 )

@c = dso_local constant [2 x [1 x i32]] [[1 x i32] [i32 1], [1 x i32] [i32 3]]
@b = dso_local global [2 x [3 x i32]] [[3 x i32] [i32 1, i32 0, i32 0], [3 x i32] zeroinitializer]
@e = dso_local global [4 x [4 x i32]] zeroinitializer 
@d = dso_local global [5 x i32] zeroinitializer 
@a = dso_local global [3 x i32] [i32 1, i32 2, i32 0]

define dso_local i32 @main() {
    %1 = getelementptr [2 x [1 x i32]], [2 x [1 x i32]]* @c, i32 0, i32 0
    %2 = add i32 0, 1
    %3 = mul i32 %2, 1
    %4 = getelementptr [1 x i32], [1 x i32]* %1, i32 0, i32 0
    %5 = add i32 %3, 0
    %6 = getelementptr i32, i32* %4, i32 %5
    %7 = load i32, i32* %6
    %8 = getelementptr [2 x [3 x i32]], [2 x [3 x i32]]* @b, i32 0, i32 0
    %9 = add i32 0, 0
    %10 = mul i32 %9, 3
    %11 = getelementptr [3 x i32], [3 x i32]* %8, i32 0, i32 0
    %12 = add i32 %10, 0
    %13 = getelementptr i32, i32* %11, i32 %12
    %14 = load i32, i32* %13
    %15 = add i32 %7, %14
    %16 = getelementptr [2 x [1 x i32]], [2 x [1 x i32]]* @c, i32 0, i32 0
    %17 = add i32 0, 0
    %18 = mul i32 %17, 1
    %19 = getelementptr [1 x i32], [1 x i32]* %16, i32 0, i32 0
    %20 = add i32 %18, 0
    %21 = getelementptr i32, i32* %19, i32 %20
    %22 = load i32, i32* %21
    %23 = add i32 %15, %22
    %24 = getelementptr [3 x i32], [3 x i32]* @a, i32 0, i32 0
    %25 = add i32 0, 1
    %26 = getelementptr i32, i32* %24, i32 %25
    %27 = load i32, i32* %26
    %28 = add i32 %23, %27
    %29 = getelementptr [5 x i32], [5 x i32]* @d, i32 0, i32 0
    %30 = add i32 0, 4
    %31 = getelementptr i32, i32* %29, i32 %30
    %32 = load i32, i32* %31
    %33 = add i32 %28, %32
    call void @putint(i32 %33)
    ret i32 0
}
```

输出样例 2：

```c
7
```

### 样例 3

样例程序 3：

```cpp
int a = 1;
int b[2] = {1, a};
int main() {
    putint(b[1]);
    return 0;
}
```

输出样例 3：

编译器直接以**非 0 的返回值**退出。

### 样例 4

样例程序 4：

```cpp
int arr[2][2] = {{1, 1}, {4, 5}};
int main() {
    arr[1] = 2;
    putint(arr[1][0]);
    return 0;
}
```

输出样例 4：

编译器直接以**非 0 的返回值**退出。
