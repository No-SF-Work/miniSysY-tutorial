# Part 13 函数

在 Part 13 中，你的编译器需要支持定义函数。

如果你的 IR 指令采用数字编号，需要注意的是：**函数的每个参数会分别占用一个编号**。

你需要支持的语法规则如下（以 `CompUnit` 为开始符号）：

```rust
CompUnit     -> [CompUnit] (Decl | FuncDef) // [changed]
Decl         -> ConstDecl | VarDecl
ConstDecl    -> 'const' BType ConstDef { ',' ConstDef } ';'
BType        -> 'int'
ConstDef     -> Ident { '[' ConstExp ']' } '=' ConstInitVal
ConstInitVal -> ConstExp
                | '{' [ ConstInitVal { ',' ConstInitVal } ] '}'
ConstExp     -> AddExp
VarDecl      -> BType VarDef { ',' VarDef } ';'
VarDef       -> Ident { '[' ConstExp ']' }
                | Ident { '[' ConstExp ']' } '=' InitVal
InitVal      -> Exp
                | '{' [ InitVal { ',' InitVal } ] '}'
FuncDef      -> FuncType Ident '(' [FuncFParams] ')' Block // [changed]
FuncType     -> 'void' | 'int' // [changed]
FuncFParams  -> FuncFParam { ',' FuncFParam } // [new]
FuncFParam   -> BType Ident ['[' ']' { '[' Exp ']' }] // [new]
Block        -> '{' { BlockItem } '}'
BlockItem    -> Decl | Stmt
Stmt         -> LVal '=' Exp ';'
                | Block
                | [Exp] ';'
                | 'if' '(' Cond ')' Stmt [ 'else' Stmt ]
                | 'while' '(' Cond ')' Stmt
                | 'break' ';'
                | 'continue' ';'
                | 'return' [Exp] ';' // [changed]
Exp          -> AddExp
Cond         -> LOrExp
LVal         -> Ident {'[' Exp ']'}
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

### `CompUnit`

- 程序中必须存在且仅存在一个标识为 `main`、无参数、返回类型为 `int` 的 `FuncDef`，`main` 函数是程序的入口点。
- `CompUnit` 的**顶层**变量/常量声明语句（对应 `Decl`）、函数定义（对应 `FuncDef`）都不可以重复定义同名标识符（`Ident`），即便标识符的类型不同也不允许。
- `CompUnit` 的变量/常量/函数声明的作用域从该声明处开始到文件结尾。
  - 这意味着一个函数中不能调用在它的定义之后定义的函数。

### `FuncFParam` 与实参

- `FuncFParam` 定义一个函数的一个形式参数。当 `Ident` 后面的可选部分存在时，表示该形式参数为一个数组。
- 当 `FuncFParam` 为数组定义时，其第一维的长度省去（用方括号 `[]` 表示），而第二维则需要用表达式指明长度，长度是编译时可求值的常量表达式。
- 函数实参的语法是 `Exp`。对于 `int` 类型的参数，遵循按值传递；对于数组类型的参数，则形参接收的是实参数组的地址，并通过地址间接访问实参数组中的元素。
  - 为了简化情况，我们的测试用例中将**不会出现将常量数组的地址作为实参传给函数**的情况。
- 可以将二维数组的一部分传到形参数组中，如定义了 `int a[4][3]`，可以将 `a[1]` 作为一个包含三个元素的一维数组传递给类型为 `int[]` 的形参。
- 函数调用时，实际参数的类型和个数必须与 `Ident` 对应的函数定义的形参完全匹配。

### `FuncDef`

- `FuncDef` 表示函数定义。其中的 `FuncType` 指明返回类型。
  - 当返回类型为 `int` 时，函数内所有分支都应当含有带有 `Exp` 的 `return` 语句。不含有 `return` 语句的分支的返回值未定义；
  - 当返回值类型为 `void` 时，函数内只能出现不带返回值的 `return` 语句。

## 运行时库的函数

你还需要支持数组相关的库函数的调用：

1. `int getarray(int []);`：输入一串整数，第 1 个整数代表后续要输入的整数个数，该个数通过返回值返回；后续的整数通过传入的数组参数返回。`getarray()` 不会检查调用者提供的数组是否有足够的空间容纳输入的一串整数。
   ```cpp
   int a[10][10];
   int n;
   n = getarray(a[0]);
   ```
2. `void putarray(int, int[]);`
   第 1 个参数表示要输出的整数个数（假设为 N），后面应该跟上要输出的 N 个整数的数组。`putarray()` 在输出时会在整数之间安插空格。
   ```cpp
   int n = 2;
   int a[2] = {2, 3};
   putarray(n, a);
   ```

## 示例

### 样例 1

样例程序 1：

```cpp
int func1() {
    return 555;
}

int func2() {
    return 111;
}

int main() {
    int a = func1();
    putint(a - func2());
    return 0;
}
```

示例 IR 1：

```llvm
declare void @putint(i32)
define dso_local i32 @func1() {
    ret i32 555
}
define dso_local i32 @func2() {
    ret i32 111
}
define dso_local i32 @main() {
    %1 = alloca i32
    %2 = call i32 @func1()
    store i32 %2, i32* %1
    %3 = load i32, i32* %1
    %4 = call i32 @func2()
    %5 = sub i32 %3, %4
    call void @putint(i32 %5)
    ret i32 0
}
```

输出样例 1：

```c
444
```

### 样例 2

样例程序 2：

```cpp
void set1(int pos, int arr[]) {
    arr[pos] = 1;
}

int main() {
    int a[2][5];
    int n;
    n = getarray(a[0]);
    getarray(a[1]);
    int i = 0;
    while (i < n) {
        set1(i, a[i % 2]);
        i = i + 1;
    }
    putarray(n, a[0]);
    putarray(n, a[1]);
    return 0;
}
```

示例 IR 2：

```llvm
declare i32 @getarray(i32*)
declare void @putarray(i32, i32*)

define dso_local void @set1(i32 %0, i32* %1) {
    %3 = alloca i32* 
    %4 = alloca i32
    store i32 %0, i32* %4
    store i32*  %1, i32* * %3
    %5 = load i32* , i32* * %3
    %6 = load i32, i32* %4
    %7 = getelementptr i32, i32* %5, i32 %6
    store i32 1, i32* %7
    ret void 
}

define dso_local i32 @main() {
    %1 = alloca i32
    %2 = alloca i32
    %3 = alloca [2 x [5 x i32]]
    %4 = getelementptr [2 x [5 x i32]], [2 x [5 x i32]]* %3, i32 0, i32 0
    %5 = add i32 0, 0
    %6 = mul i32 %5, 5
    %7 = getelementptr [5 x i32], [5 x i32]* %4, i32 0, i32 %6
    %8 = call i32 @getarray(i32* %7)
    store i32 %8, i32* %2
    %9 = getelementptr [2 x [5 x i32]], [2 x [5 x i32]]* %3, i32 0, i32 0
    %10 = add i32 0, 1
    %11 = mul i32 %10, 5
    %12 = getelementptr [5 x i32], [5 x i32]* %9, i32 0, i32 %11
    %13 = call i32 @getarray(i32* %12)
    store i32 0, i32* %1
    br label %14

14:
    %15 = load i32, i32* %1
    %16 = load i32, i32* %2
    %17 = icmp slt i32 %15, %16
    br i1 %17, label %18, label %28 

18:
    %19 = load i32, i32* %1
    %20 = getelementptr [2 x [5 x i32]], [2 x [5 x i32]]* %3, i32 0, i32 0
    %21 = load i32, i32* %1
    %22 = srem i32 %21, 2
    %23 = add i32 0, %22
    %24 = mul i32 %23, 5
    %25 = getelementptr [5 x i32], [5 x i32]* %20, i32 0, i32 %24
    call void @set1(i32 %19, i32* %25)
    %26 = load i32, i32* %1
    %27 = add i32 %26, 1
    store i32 %27, i32* %1
    br label %14

28:
    %29 = load i32, i32* %2
    %30 = getelementptr [2 x [5 x i32]], [2 x [5 x i32]]* %3, i32 0, i32 0
    %31 = add i32 0, 0
    %32 = mul i32 %31, 5
    %33 = getelementptr [5 x i32], [5 x i32]* %30, i32 0, i32 %32
    call void @putarray(i32 %29, i32* %33)
    %34 = load i32, i32* %2
    %35 = getelementptr [2 x [5 x i32]], [2 x [5 x i32]]* %3, i32 0, i32 0
    %36 = add i32 0, 1
    %37 = mul i32 %36, 5
    %38 = getelementptr [5 x i32], [5 x i32]* %35, i32 0, i32 %37
    call void @putarray(i32 %34, i32* %38)
    ret i32 0
}
```

输入样例 2：

```c
5 1 2 3 4 5
5 6 7 8 9 10
```

输出样例 2：

```c
5: 1 2 1 4 1
5: 6 1 8 1 10
```

### 样例 3

样例程序 3：

```cpp
int gcd(int m, int n) {
    if (n == 0) {
        return m;
    }
    return gcd(n, m % n);
}

int main() {
    int a = 100, b = 48;
    putint(gcd(a, b));
    return 0;
}

```

示例 IR 3：

```llvm
declare void @putint(i32 )

define dso_local i32 @gcd(i32 %0, i32 %1) {
    %3 = alloca i32
    %4 = alloca i32
    store i32 %0, i32* %4
    store i32 %1, i32* %3
    %5 = load i32, i32* %3
    %6 = icmp eq i32 %5, 0
    br i1 %6, label %7, label %9 

7:
    %8 = load i32, i32* %4
    ret i32 %8
    
9:
    %10 = load i32, i32* %3
    %11 = load i32, i32* %4
    %12 = load i32, i32* %3
    %13 = srem i32 %11, %12
    %14 = call i32 @gcd(i32 %10, i32 %13)
    ret i32 %14
}

define dso_local i32 @main() {
    %1 = alloca i32
    %2 = alloca i32
    store i32 100, i32* %2
    store i32 48, i32* %1
    %3 = load i32, i32* %2
    %4 = load i32, i32* %1
    %5 = call i32 @gcd(i32 %3, i32 %4)
    call void @putint(i32 %5)
    ret i32 0
}
```

输出样例 3：

```
4
```

### 样例 4

样例程序 4：

```cpp
int sum2d(int a[][3]) {
    int i = 0, sum = 0;
    while (i < 2) {
        int j = 0;
        while (j < 3) {
            sum = sum + a[i][j];
            j = j + 1;
        }
        i = i + 1;
    }
    return sum;
}

int main() {
    int arr[2][3] = {{1, 2, 3}, {4, 5}};
    putint(sum2d(arr));
    return 0;
}
```

示例 IR 4：

```llvm
declare void @memset(i32*, i32, i32)
declare void @putint(i32 )

define dso_local i32 @sum2d([3 x i32]* %0) {
    %2 = alloca i32
    %3 = alloca i32
    %4 = alloca i32
    %5 = alloca [3 x i32]* 
    store [3 x i32]* %0, [3 x i32]* * %5
    store i32 0, i32* %4
    store i32 0, i32* %3
    br label %6

6:
    %7 = load i32, i32* %4
    %8 = icmp slt i32 %7, 2
    br i1 %8, label %12, label %10 

9:
    store i32 0, i32* %2
    br label %13

10:
    %11 = load i32, i32* %3
    ret i32 %11
    
12:
    br label %9

13:
    %14 = load i32, i32* %2
    %15 = icmp slt i32 %14, 3
    br i1 %15, label %16, label %29 

16:
    %17 = load i32, i32* %3
    %18 = load [3 x i32]*, [3 x i32]* * %5
    %19 = load i32, i32* %4
    %20 = getelementptr [3 x i32], [3 x i32]* %18, i32 0
    %21 = mul i32 %19, 3
    %22 = load i32, i32* %2
    %23 = add i32 %21, %22
    %24 = getelementptr [3 x i32], [3 x i32]* %20, i32 0, i32 %23
    %25 = load i32, i32* %24
    %26 = add i32 %17, %25
    store i32 %26, i32* %3
    %27 = load i32, i32* %2
    %28 = add i32 %27, 1
    store i32 %28, i32* %2
    br label %13

29:
    %30 = load i32, i32* %4
    %31 = add i32 %30, 1
    store i32 %31, i32* %4
    br label %6
}

define dso_local i32 @main() {
    %1 = alloca [2 x [3 x i32]]
    %2 = getelementptr [2 x [3 x i32]], [2 x [3 x i32]]* %1, i32 0, i32 0
    %3 = getelementptr [3 x i32], [3 x i32]* %2, i32 0, i32 0
    call void @memset(i32* %3, i32 0, i32 24)
    store i32 1, i32* %3
    %4 = getelementptr i32, i32* %3, i32 1
    store i32 2, i32* %4
    %5 = getelementptr i32, i32* %3, i32 2
    store i32 3, i32* %5
    %6 = getelementptr i32, i32* %3, i32 3
    store i32 4, i32* %6
    %7 = getelementptr i32, i32* %3, i32 4
    store i32 5, i32* %7
    %8 = getelementptr [2 x [3 x i32]], [2 x [3 x i32]]* %1, i32 0, i32 0
    %9 = call i32 @sum2d([3 x i32]* %8)
    call void @putint(i32 %9)
    ret i32 0
}
```

输出样例 4：

```
15
```

### 样例 5

样例程序 5：

```cpp
int foo(int a, int b) {
    int t = a + b;
    a = t - a;
    b = t - b;
    return a - b;
}

int main() {
    putint(foo(1, 2, 3));
    return 0;
}
```

输出样例 5：

编译器直接以**非 0 的返回值**退出。
