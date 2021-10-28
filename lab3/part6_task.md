# Part 6 调用函数

在 Part 6 中，你的编译器需支持对 miniSysY 库函数的调用。

在之后的实验中，评测样例程序中会调用 miniSysY 运行时库的函数来进行输入输出。运行时库提供一系列 I/O 函数，用于在程序中表达输入/输出需求，这些库函数不用在程序中声明即可调用，你的编译器需要支持在调用这些库函数时直接翻译成对应的 LLVM IR 形式的调用，但不需要检查这些库函数参数的合法性。评测时评测机会将运行时库链接并进行评测。

在 **Lab 3** 中不要求支持 `getarray` 和 `putarray` 函数。

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
Exp          -> AddExp
LVal         -> Ident
PrimaryExp   -> '(' Exp ')' | LVal | Number
AddExp       -> MulExp
                | AddExp ('+' | '−') MulExp
MulExp       -> UnaryExp
                | MulExp ('*' | '/' | '%') UnaryExp
UnaryExp     -> PrimaryExp
                | Ident '(' [FuncRParams] ')'
                | UnaryOp UnaryExp
FuncRParams  -> Exp { ',' Exp }
UnaryOp      -> '+' | '-'
```

## 语义约束

对库函数以外未定义的函数的调用应当报错，对函数的调用传入的实参列表与函数的形参列表长度或类型不匹配时应当报错。

## 运行时库的函数

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
   int a[2] = {2, 3};
   putarray(n, a);
   ```

## 示例

### 样例 1

样例程序 1：

```c
int main() {
    int n = getint();
    putint(n + 4);
    return 0;
}
```

示例 IR 1：

```llvm
declare i32 @getint()
declare void @putint(i32)
define dso_local i32 @main(){
    %1 = alloca i32
    %2 = call i32 @getint()
    store i32 %2, i32* %1
    %3 = load i32, i32* %1
    %4 = add i32 %3, 4
    call void @putint(i32 %4)
    ret i32 0
}
```

输入样例 1：

```c
4
```

输出样例 1：

```c
8
```

### 样例 2

样例程序 2：

```c
int main() {
    int a = getch(), b;
    b = getch();
    putch(a);
    putch(b);
    putch(10);
    putch(a - 16);
    putch(b + 6);
    return 0;
}
```

示例 IR 2：

```llvm
declare i32 @getch()
declare void @putch(i32)
define dso_local i32 @main(){
    %1 = alloca i32
    %2 = alloca i32
    %3 = call i32 @getch()
    store i32 %3, i32* %2
    %4 = call i32 @getch()
    store i32 %4, i32* %1
    %5 = load i32, i32* %2
    call void @putch(i32 %5)
    %6 = load i32, i32* %1
    call void @putch(i32 %6)
    call void @putch(i32 10)
    %7 = load i32, i32* %2
    %8 = sub i32 %7, 16
    call void @putch(i32 %8)
    %9 = load i32, i32* %1
    %10 = add i32 %9, 6
    call void @putch(i32 %10)
    ret i32 0
}
```

输入样例 2：

```c
tl
```

输出样例 2：

```c
tl
dr
```

### 样例 3

样例程序 3：

```c
int main() {
    int a = getint();
    putint();
    return 0;
}
```

输出样例 3：

编译器直接以**非 0 的返回值**退出。
