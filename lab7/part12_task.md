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
  - 花括号中初始值少于对应维度元素个数，该维其余部分将被隐式初始化为 0，如 `int a[5] = {1, 2};`、`int a[4][3] = { {1, 2, 3}, {4, 5}, {} };`。

### `VarDef`

- `VarDef` 的数组维度和各维长度的定义部分存在时，表示定义数组。其语义和 C 语言一致。`VarDef` 中表示各维长度的 `ConstExp` 必须能在编译时求值到非负整数的常量表达式。在声明数组时各维长度都需要显式给出，而不允许是未知的。
- 全局变量数组的 `InitVal` 中的 `Exp` 必须是常量表达式。局部变量数组 `InitVal` 中的 `Exp` 可以是任何符合语义的表达式。

### 初值的常量/可求值总结

> “编译时可求值”约束为常数和 `int` 类型变量/常量所构成的表达式，且不包括数组元素、函数返回值。
> 在评测时只会针对表达式是否满足**常量**要求进行评测。

- 全局 `int` 类型变量/常量的初值必须是编译时可求值的常量表达式。
- 局部 `int` 类型常量的初值必须是编译时可求值的表达式。（和 C 语言略有不同）
- 数组的各维长度必须是编译时可求值的非负常量表达式。
- 全局数组的 `ConstInitVal/InitVal` 中的 `ConstExp/Exp` 必须是编译时可求值的常量表达式。
- 局部常量数组的 `ConstInitVal` 中的 `ConstExp` 必须是编译时可求值的表达式。（和 C 语言略有不同）

## 示例

### 样例 1

样例程序 1：

```cpp
int main() {
    int a[2][2] = {{1}, {2, 3}};
    int e[2][2] = {{a[0][0], a[2][1]}, {5, 6}};
    putint(e[1][1] + a[1][0]);
    return 0;
}
```

示例 IR 1：

```llvm

```

输出样例 1：

```c
8
```

### 样例 2

样例程序 2：

```cpp
const int c[2][2] = {{1, 2}, {3}};
int b[2][1] = {{1}};
int main() {
    putint(c[1][1] + b[0][0] + c[0][1] + b[1][0]);
    return 0;
}
```

示例 IR 2：

```llvm

```

输出样例 2：

```c
3
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
int arr[5] = {1, 1, 4, 5, 1};
int main() {
    putint(arr[5]);
    return 0;
}
```

输出样例 4：

编译器直接以**非 0 的返回值**退出。
