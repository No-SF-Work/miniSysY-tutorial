# Part 10 循环语句

在 Part 10 中，你的编译器需要支持 `while` 循环。

保证测试用例在正确的控制流下不会出现死循环。

你需要支持的语法规则如下（以 `CompUnit` 为开始符号）：

```rust
CompUnit     -> Decl* FuncDef
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
                | 'while' '(' Cond ')' Stmt
                | 'return' Exp ';' // [changed]
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

## 示例

~~示例 IR 中的基本块跳转有点乱，是历史遗留问题~~

### 样例 1

样例程序 1：

```cpp
int main() {
    int n = getint();
    int i = 0, sum = 0;
    while (i < n) {
        i = i + 1;
        sum = sum + i;
        putint(sum);
        putch(10);
    }
    return 0;
}
```

示例 IR 1：

```llvm
declare i32 @getint()
declare void @putint(i32)
declare void @putch(i32)
define dso_local i32 @main() {
    %1 = alloca i32
    %2 = alloca i32
    %3 = alloca i32
    %4 = call i32 @getint()
    store i32 %4, i32* %3
    store i32 0, i32* %2
    store i32 0, i32* %1
    br label %5

5:
    %6 = load i32, i32* %2
    %7 = load i32, i32* %3
    %8 = icmp slt i32 %6, %7
    br i1 %8, label %9, label %16

9:
    %10 = load i32, i32* %2
    %11 = add i32 %10, 1
    store i32 %11, i32* %2
    %12 = load i32, i32* %1
    %13 = load i32, i32* %2
    %14 = add i32 %12, %13
    store i32 %14, i32* %1
    %15 = load i32, i32* %1
    call void @putint(i32 %15)
    call void @putch(i32 10)
    br label %5

16:
    ret i32 0
}
```

输入样例 1：

```c
5
```

输出样例 1：

```c
1
3
6
10
15

```

### 样例 2

样例程序 2：

```cpp
int main() {
    const int ch = 48;
    int i = 1;
    while (i < 12) {
        int j = 0;
        while (j < 2 * i - 1) {
            if (j % 3 == 1) {
                putch(ch + 1);
            } else {
                putch(ch);
            }
            j = j + 1;
        }
        putch(10);
        i = i + 1;
    }
    return 0;
}
```

示例 IR 2：

```llvm
declare void @putch(i32 )
define dso_local i32 @main() {
    %1 = alloca i32
    %2 = alloca i32
    store i32 1, i32* %2
    br label %3

3:
    %4 = load i32, i32* %2
    %5 = icmp slt i32 %4, 12
    br i1 %5, label %8, label %7

6:
    store i32 0, i32* %1
    br label %9

7:
    ret i32 0
8:
    br label %6

9:
    %10 = load i32, i32* %1
    %11 = load i32, i32* %2
    %12 = mul i32 2, %11
    %13 = sub i32 %12, 1
    %14 = icmp slt i32 %10, %13
    br i1 %14, label %24, label %21

15:
    %16 = load i32, i32* %1
    %17 = sdiv i32 %16, 3
    %18 = mul i32 %17, 3
    %19 = sub i32 %16, %18
    %20 = icmp eq i32 %19, 1
    br i1 %20, label %31, label %30

21:
    call void @putch(i32 10)
    %22 = load i32, i32* %2
    %23 = add i32 %22, 1
    store i32 %23, i32* %2
    br label %3

24:
    br label %15

25:
    %26 = add i32 48, 1
    call void @putch(i32 %26)
    br label %27

27:
    %28 = load i32, i32* %1
    %29 = add i32 %28, 1
    store i32 %29, i32* %1
    br label %9

30:
    call void @putch(i32 48)
    br label %27

31:
    br label %25
}
```

输出样例 2:

```c
0
010
01001
0100100
010010010
01001001001
0100100100100
010010010010010
01001001001001001
0100100100100100100
010010010010010010010
```
