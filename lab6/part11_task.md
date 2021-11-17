# Part 11 continue、break 与代码回填

在 Part 11 中，你的编译器需要支持循环中的 `continue`、`break` 语句。

`continue`、`break` 不会出现在循环外。

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
                | 'break' ';'
                | 'continue' ';'
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

```c
int main() {
    int n = getint();
    int i = 0, sum = 0;
    while (i < n) {
        if (i % 2 == 0) {
            i = i + 1;
            continue;
        }
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
declare void @putint(i32 )
declare void @putch(i32 )
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
    br i1 %8, label %14, label %13

9:
    %10 = load i32, i32* %2
    %11 = srem i32 %10, 2
    %12 = icmp eq i32 %11, 0
    br i1 %12, label %25, label %18

13:
    ret i32 0
14:
    br label %9

15:
    %16 = load i32, i32* %2
    %17 = add i32 %16, 1
    store i32 %17, i32* %2
    br label %5

18:
    %19 = load i32, i32* %2
    %20 = add i32 %19, 1
    store i32 %20, i32* %2
    %21 = load i32, i32* %1
    %22 = load i32, i32* %2
    %23 = add i32 %21, %22
    store i32 %23, i32* %1
    %24 = load i32, i32* %1
    call void @putint(i32 %24)
    call void @putch(i32 10)
    br label %5

25:
    br label %15
}
```

输入样例 1：

```c
10
```

输出样例 1：

```c
2
6
12
20
30
```

### 样例 2

样例程序 2：

```c
int main() {
    const int ch = 48;
    int i = 1;
    while (i < 12) {
        int j = 0;
        while (1 == 1) {
            if (j % 3 == 1) {
                putch(ch + 1);
            } else {
                putch(ch);
            }
            j = j + 1;
            if (j >= 2 * i - 1)
                break;
        }
        putch(10);
        i = i + 1;
        continue; // something meaningless
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
    %10 = icmp eq i32 1, 1
    br i1 %10, label %18, label %15

11:
    %12 = load i32, i32* %1
    %13 = srem i32 %12, 3
    %14 = icmp eq i32 %13, 1
    br i1 %14, label %30, label %29

15:
    call void @putch(i32 10)
    %16 = load i32, i32* %2
    %17 = add i32 %16, 1
    store i32 %17, i32* %2
    br label %3

18:
    br label %11

19:
    %20 = add i32 48, 1
    call void @putch(i32 %20)
    br label %21

21:
    %22 = load i32, i32* %1
    %23 = add i32 %22, 1
    store i32 %23, i32* %1
    %24 = load i32, i32* %1
    %25 = load i32, i32* %2
    %26 = mul i32 2, %25
    %27 = sub i32 %26, 1
    %28 = icmp sge i32 %24, %27
    br i1 %28, label %33, label %32

29:
    call void @putch(i32 48)
    br label %21

30:
    br label %19

31:
    br label %15

32:
    br label %9

33:
    br label %31
}
```

输出样例 2：

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
