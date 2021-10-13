# 剩下的旅程：miniSysY

miniSysY 语言是在 [SysY 语言](https://gitlab.eduxiji.net/nscscc/compiler2021/-/blob/master/SysY%E8%AF%AD%E8%A8%80%E5%AE%9A%E4%B9%89.pdf) 基础上进行一些修改得到的 C 语言子集。编译原理实验课剩下的工作就是从编译一个简单的 `main()` 函数开始，逐渐扩充对 miniSysY 语言文法的支持，实现一个较为完整的编译器。由于我们没有系统地学习过汇编语言，我们只需要将 miniSysY 代码编译到 LLVM IR。

在后续的实验中，你可以使用自动生成工具（如 ANTLR、flex/bison 等）来生成你的编译器前端代码，也可以选择手工实现前端的词法分析和语法分析代码。

**关于评测**：你需要从 `$input` 读取输入文件，将编译生成的文本格式的 LLVM IR 输出到 `$ir` 中，评测机会使用 `lli` 解释执行该文件，并评测生成的 IR 代码是否正确。请在 `judge.toml` 的 `run` 中使用 `$input` 代替输入文件路径，`$ir` 代替输出文件路径。如：`./compiler < $input > $ir` 或 `./compiler $input $ir` 等。

## 实验概览

miniSysY 实验共有 8 个 lab 和一个挑战实验。

8 个 lab 会以每周一个的速度放出，lab 1 ~ lab 4 的评测和提交实验报告持续时间为五周，后续实验的评测和提交实验报告截止时间为期末统计成绩之前。每个 lab 进行到第三周时会放出示例编译器。部分 lab 包括多个 part，只是为了方便你迭代实现和调试，在评测时你可以选择一部分已经实现的 part 进行评测，以降低等待时间。需要注意的是，你需要在**一次评测**中通过所有 part 的测试点才算完成一个 lab。

挑战实验选做其中一部分即可，得分不会溢出。为了便于你设计架构，挑战实验会提前放出内容。

- 仅有 `main` 函数的编译器及注释（10%）
- 常量表达式运算（10%）
- 局部变量定义与赋值（10%）
- `if` 语句与条件表达式（10%）
- 作用域与全局变量（10%）
- 循环（10%）
- 数组（10%）
- 函数（10%）
- 挑战实验（20%）
  - `mem2reg`（20%）
  - 死代码删除（10%）
  - 函数内联（10%）
  - 短路求值（10%）
  - 常量折叠（10%）


## miniSysY 文法

为了便于你设计架构，现给出 miniSysY 的全部文法如下：

```rust
CompUnit     -> [CompUnit] (Decl | FuncDef)
Decl         -> ConstDecl | VarDecl
ConstDecl    -> 'const' BType ConstDef { ',' ConstDef } ';'
BType        -> 'int'
ConstDef     -> Ident { '[' ConstExp ']' } '=' ConstInitVal
ConstInitVal -> ConstExp 
                | '{' [ ConstInitVal { ',' ConstInitVal } ] '}'
VarDecl      -> BType VarDef { ',' VarDef } ';'
VarDef       -> Ident { '[' ConstExp ']' }
                | Ident { '[' ConstExp ']' } '=' InitVal
InitVal      -> Exp 
                | '{' [ InitVal { ',' InitVal } ] '}'
FuncDef      -> FuncType Ident '(' [FuncFParams] ')' Block
FuncType     -> 'void' | 'int'  
FuncFParams  -> FuncFParam { ',' FuncFParam }
FuncFParam   -> BType Ident ['[' ']' { '[' Exp ']' }]
Block        -> '{' { BlockItem } '}'
BlockItem    -> Decl | Stmt
Stmt         -> LVal '=' Exp ';' 
                | [Exp] ';' 
                | Block
                | 'if' '(' Cond ')' Stmt [ 'else' Stmt ]
                | 'while' '(' Cond ')' Stmt
                | 'break' ';' 
                | 'continue' ';'
                | 'return' [Exp] ';'
Exp          -> AddExp
Cond         -> LOrExp
LVal         -> Ident {'[' Exp ']'}
PrimaryExp   -> '(' Exp ')' | LVal | Number
UnaryExp     -> PrimaryExp 
                | Ident '(' [FuncRParams] ')'
                | UnaryOp UnaryExp
UnaryOp      -> '+' | '−' | '!'  // 注：保证 '!' 仅出现在 Cond 中
FuncRParams  -> Exp { ',' Exp }
MulExp       -> UnaryExp 
                | MulExp ('*' | '/' | '%') UnaryExp
AddExp       -> MulExp 
                | AddExp ('+' | '−') MulExp
RelExp       -> AddExp 
                | RelExp ('<' | '>' | '<=' | '>=') AddExp
EqExp        -> RelExp 
                | EqExp ('==' | '!=') RelExp
LAndExp      -> EqExp 
                | LAndExp '&&' EqExp
LOrExp       -> LAndExp 
                | LOrExp '||' LAndExp
ConstExp     -> AddExp  // 在语义上额外约束这里的 AddExp 必须是一个可以在编译期求出值的常量
```

其中 `Ident` 和 `Number` 的详细定义见下。

## 文法的细节补充

### 注释

miniSysY 语言中有两种注释，包括以 `//` 开头的单行注释和包裹在 `/*`、`*/` 中的多行注释。

- 单行注释：以 `//` 开始，直到换行符结束，不包括换行符。
- 多行注释：以 `/*` 开始，直到第一次出现 `*/` 时结束，包括 `*/`。

### 标识符 `Ident` 的定义

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

### 数字 `Number` 的定义

Number 可以表示八进制、十进制、十六进制数字，文法如下：

```rust
Number             -> decimal-const | octal-const | hexadecimal-const
decimal-const      -> nonzero-digit | decimal-const digit
octal-const        -> '0' | octal-const octal-digit
hexadecimal-const  -> hexadecimal-prefix hexadecimal-digit 
                      | hexadecimal-const hexadecimal-digit
hexadecimal-prefix -> '0x' | '0X'
nonzero-digit      -> '1' | '2' | ... | '9'
octal-digit        -> '0' | '1' | ... | '7'
hexadecimal-digit  -> '0' | '1' | ... | '9'
                      | 'a' | 'b' | 'c' | 'd' | 'e' | 'f'
                      | 'A' | 'B' | 'C' | 'D' | 'E' | 'F'
```

在将 `Number` 翻译成 LLVM IR 中的常量数字时，你需要注意进制的转换。输入保证 `Number` 转换成十进制后范围为 `0 <= Number <= 2147483647`，不会出现范围之外的数字。

## 语义约束

### `CompUnit`

- 一个 miniSysY 程序由单个文件组成，对应 EBNF 表示中的一个 `CompUnit`。在该 `CompUnit` 中，必须存在且仅存在一个标识为 `main` 、无参数、返回类型为 `int` 的 `FuncDef`。
- `CompUnit` 的**顶层**变量/常量声明语句（对应 `Decl`）、函数定义（对应 `FuncDef`）都不可以重复定义同名标识符（`Ident`），即便标识符的类型不同也不允许。

### `ConstInitVal` 和 `InitVal`

- 全局变量声明中指定的初值表达式必须是常量表达式。
- 常量或变量声明中指定的初值要与该常量或变量的类型一致。
- 未显式初始化的局部变量，其值是不确定的；而未显式初始化的全局变量，其值均被初始化为 0。

### `ConstDef`

- `ConstDef` 用于定义常量。在 `Ident` 后、`=` 之前是可选的数组维度和各维长度的定义部分，在 `=` 之后是初始值。
- `ConstDef` 的数组维度和各维长度的定义部分不存在时，表示定义单个变量。此时 `=` 右边必须是单个初始数值。
- `ConstDef` 的数组维度和各维长度的定义部分存在时，表示定义数组。其语义和 C 语言一致，**miniSysY 只要求支持一维数组和二维数组**。比如[2*3][8/2]表示二维数组，第一和第二维长度分别为 6 和 4，每维的下界从 0 编号。 `ConstDef` 中表示各维长度的 `ConstExp` 都必须能在编译时求值到非负整数。在声明数组时各维长度都需要显式给出，而不允许是未知的。
- 当 `ConstDef` 定义的是数组时，`=` 右边的 `ConstInitVal` 表示常量初始化器。`ConstInitVal` 中的 `ConstExp` 是能在编译时求值的 `int` 型表达式，其中可以引用已定义的符号常量。
- `ConstInitVal` 初始化器必须是以下三种情况之一：
  - 一对花括号 `{}`，表示所有元素初始为 0；
  - 数组维数和各维长度完全对应的初始值，如 `int a[3] = {1, 2, 3};`、`int a[3][2] = { {1, 2}, {3, 4}, {5, 6} };`；
  - 花括号中初始值少于对应维度元素个数，该维其余部分将被隐式初始化为 0，如 `int a[5] = {1, 2};`、`int a[4][3] = { {1, 2, 3}, {4, 5}, {} };`。

### `VarDef`

- `VarDef` 用于定义变量。当不含有 `=` 和初始值时，其运行时实际初值未定义。
- `VarDef` 的数组维度和各维长度的定义部分不存在时，表示定义单个变量。存在时，和 `ConstDef` 类似，表示定义数组。
- 当 `VarDef` 含有 `=` 和初始值时， `=` 右边的 `InitVal` 和 `ConstInitVal` 的结构要求相同，唯一的不同是 `ConstInitVal` 中的表达式是 `ConstExp` 常量表达式，而 `InitVal` 中的表达式可以是当前上下文合法的任何 `Exp`。
- `VarDef` 中表示各维长度的 `ConstExp` 必须能求值到**非负整数**，`InitVal` 中的初始值为 `Exp`，其中可以引用变量。

### `FuncFParam`

- `FuncFParam` 定义一个函数的一个形式参数。当 `Ident` 后面的可选部分存在时，表示该形式参数为一个数组。
- 当 `FuncFParam` 为数组定义时，其第一维的长度省去（用方括号 `[]` 表示），而第二维则需要用表达式指明长度，长度是常量。
- 函数实参的语法是 `Exp`。对于 `int` 类型的参数，遵循按值传递；对于数组类型的参数，则形参接收的是实参数组的地址，并通过地址间接访问实参数组中的元素。
- 可以将二维数组的一部分传到形参数组中，如定义了 `int a[4][3]`，可以将 `a[1]` 作为一个包含三个元素的一维数组传递给类型为 `int[]` 的形参。

### `FuncDef`

- `FuncDef` 表示函数定义。其中的 `FuncType` 指明返回类型。
    - 当返回类型为 `int` 时，函数内所有分支都应当含有带有 `Exp` 的 `return` 语句。不含有 `return` 语句的分支的返回值未定义；
    - 当返回值类型为 `void` 时，函数内只能出现不带返回值的 `return` 语句。

### `Block`

- `Block` 表示语句块。语句块会创建作用域，语句块内声明的变量的生命周期在该语句块内。
- 语句块内可以再次定义与语句块外同名的变量或常量（通过 `Decl` 语句)，其作用域从定义处开始到该语句块尾结束，它隐藏语句块外的同名变量或常量。

### `Stmt`

- `Stmt` 中的 `if` 类型语句遵循就近匹配。
- 单个 `Exp` 可以作为 `Stmt`。`Exp` 会被求值，所求的值会被丢弃。

### `LVal`

- `LVal` 表示具有左值的表达式，可以为变量或者某个数组元素。
- 当 `LVal` 表示数组时，方括号个数必须和数组变量的维数相同。

### `Exp` 和 `Cond`

- `Exp` 代表 `int` 类型的表达式，`Cond` 代表条件表达式。单目运算符 `!` 只会在 `Cond` 中出现。
- `LVal` 必须是当前作用域内、该 `Exp` 语句之前有定义的变量或常量；对于赋值号左边的 `LVal` 必须是变量。
- miniSysY 算符的优先级与结合性与 C 语言一致，文法中已经体现出了优先级与结合性的定义。
- `Cond` 中的短路求值在基础实验中不作要求，挑战实验中会有相关内容。

## 运行时库

在 lab3 及之后的 lab 中，评测样例会调用 miniSysY 运行时库的函数来完成输入输出。运行时库提供一系列 I/O 函数，用于在程序中表达输入/输出需求，这些库函数不用在程序中声明即可调用，因此在 lab3 之后，你的编译器需要支持在调用这些库函数时直接翻译成对应的 LLVM IR 形式的调用，但不需要检查这些库函数参数的合法性。评测时评测机会将运行时库链接并进行评测，在本地调试时怎样链接运行时库会在后续实验中说明。

### 运行时库的函数

1. `int getint();`：输入一个整数，返回对应的整数值。
   ```c
   int n;
   n = getint();
   ```
2. `int getch();`：输入一个字符，返回字符对应的 ASCII 码值。
   ```c
   int n;
   n = getch();
   ```
3. `int getarray(int []);`：输入一串整数，第 1 个整数代表后续要输入的整数个数，该个数通过返回值返回；后续的整数通过传入的数组参数返回。`getarray()` 不会检查调用者提供的数组是否有足够的空间容纳输入的一串整数。
   ```c
   int a[10][10]; 
   int n; 
   n = getarray(a[0]);
   ```
4. `void putint(int);`：输出一个整数的值。
   ```c
   int n = 10; 
   putint(n); 
   putint(11);
   ```
5. `void putch(int);`：输出一个 ASCII 码对应的字符。传入的整数参数取值范围为 `0~255`。
   ```c
   int n = 10; 
   putch(n);
   ```
6. `void putarray(int, int[]);`
   第 1 个参数表示要输出的整数个数（假设为 N），后面应该跟上要输出的 N 个整数的数组。`putarray()` 在输出时会在整数之间安插空格。
   ```c
   int n = 2; 
   int a[] = {2, 3}; 
   putarray(n, a);
   ```

