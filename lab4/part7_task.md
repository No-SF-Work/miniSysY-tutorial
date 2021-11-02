# Part 7 `if` 语句和条件表达式

本次实验中，你的编译器需要支持 `if`、`else` 条件分支语句以及条件表达式。

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
                | 'return' Exp ';' // [changed]
Exp          -> AddExp
Cond         -> LOrExp // [new]
LVal         -> Ident
PrimaryExp   -> '(' Exp ')' | LVal | Number
UnaryExp     -> PrimaryExp
                | Ident '(' [FuncRParams] ')'
                | UnaryOp UnaryExp
UnaryOp      -> '+' | '-' | '!'  // 保证 '!' 只出现在 Cond 中 [changed]
FuncRParams  -> Exp { ',' Exp }
MulExp       -> UnaryExp
                | MulExp ('*' | '/' | '%') UnaryExp
AddExp       -> MulExp
                | AddExp ('+' | '-') MulExp
RelExp       -> AddExp
                | RelExp ('<' | '>' | '<=' | '>=') AddExp  // [new]
EqExp        -> RelExp
                | EqExp ('==' | '!=') RelExp  // [new]
LAndExp      -> EqExp
                | LAndExp '&&' EqExp  // [new]
LOrExp       -> LAndExp
                | LOrExp '||' LAndExp  // [new]
```

**注：**

- `Cond` 中的短路求值在基础实验中不作要求，挑战实验中会有相关内容。
- 本部分的测试用例中不会出现对 `Stmt -> Block` 的变量作用域、生命周期等相关的考察。
- `Stmt` 中的 `if` 语句遵循就近匹配，即 `if if else` 等同于 `if { if else }`。

## 示例

~~示例 IR 中的基本块跳转有点乱，是历史遗留问题~~

### 样例 1

样例程序 1：

```c
int main() {
    int a = getint();
    int b = getint();
    if (a <= b) {
        putint(1);
    }
    else {
        putint(0);
    }
    return 0;
}
```

示例 IR 1：

```llvm
declare i32 @getint()
declare void @putint(i32)
define dso_local i32 @main() {
    %1 = alloca i32
    %2 = alloca i32
    %3 = call i32 @getint()
    store i32 %3, i32* %2
    %4 = call i32 @getint()
    store i32 %4, i32* %1
    %5 = load i32, i32* %2
    %6 = load i32, i32* %1
    %7 = icmp slt i32 %5, %6
    br i1 %7,label %8, label %10
    
8:
    call void @putint(i32 1)
    br label %9

9:
    ret i32 0

10:
    call void @putint(i32 0)
    br label %9
}
```

输入样例 1：

```c
9 12
```

输出样例 1：

```
1
```

### 样例 2

样例程序 2：

```c
int main() {
    int a, b, c = 1, d;
    int result;
    a = 5;
    b = 5;
    d = -2;
    result = 2;
    if (a + b + c + d == 10) {
        result = result + 1;
    } else if (a + b + c + d == 8) {
        result = result + 2;
    } else {
        result = result + 4;
    }
    putint(result);
    return 0;
}
```

示例 IR 2：

```llvm
declare void @putint(i32)
define dso_local i32 @main(){
    %1 = alloca i32
    %2 = alloca i32
    %3 = alloca i32
    %4 = alloca i32
    %5 = alloca i32
    store i32 1, i32* %3
    store i32 5, i32* %5
    store i32 5, i32* %4
    %6 = sub i32 0, 2
    store i32 %6, i32* %2
    store i32 2, i32* %1
    %7 = load i32, i32* %5
    %8 = load i32, i32* %4
    %9 = add i32 %7, %8
    %10 = load i32, i32* %3
    %11 = add i32 %9, %10
    %12 = load i32, i32* %2
    %13 = add i32 %11, %12
    %14 = icmp eq i32 %13, 10
    br i1 %14, label %29, label %20 

15:
    %16 = load i32, i32* %1
    %17 = add i32 %16, 1
    store i32 %17, i32* %1
    br label %18

18:
    %19 = load i32, i32* %1
    call void @putint(i32 %19)
    ret i32 0

20:
    %21 = load i32, i32* %5
    %22 = load i32, i32* %4
    %23 = add i32 %21, %22
    %24 = load i32, i32* %3
    %25 = add i32 %23, %24
    %26 = load i32, i32* %2
    %27 = add i32 %25, %26
    %28 = icmp eq i32 %27, 8
    br i1 %28, label %37, label %34 

29:
    br label %15

30:
    %31 = load i32, i32* %1
    %32 = add i32 %31, 2
    store i32 %32, i32* %1
    br label %33

33:
    br label %18

34:
    %35 = load i32, i32* %1
    %36 = add i32 %35, 4
    store i32 %36, i32* %1
    br label %33

37:
    br label %30
}

```

输出样例 2：

```c
6
```

### 样例 3

样例程序 3：

```c
int main() {
    int a, b, c = 1, d;
    int result;
    a = 5;
    b = 5;
    d = -2;
    result = 2;
    if (a + b == 9 || a - b == 0 && result != 4)
        result = result + 3;
    else if (c + d != -1 || (result + 1) % 2 == 1)
        result = result + 4;
    putint(result);
    return 0;
}
```

示例 IR 3：

```llvm
declare void @putint(i32)
define dso_local i32 @main() {
    %1 = alloca i32
    %2 = alloca i32
    %3 = alloca i32
    %4 = alloca i32
    %5 = alloca i32
    store i32 1, i32* %3
    store i32 5, i32* %5
    store i32 5, i32* %4
    %6 = sub i32 0, 2
    store i32 %6, i32* %2
    store i32 2, i32* %1
    %7 = load i32, i32* %5
    %8 = load i32, i32* %4
    %9 = add i32 %7, %8
    %10 = icmp eq i32 %9, 9
    br i1 %10, label %27, label %22

  11:
    %12 = load i32, i32* %1
    %13 = add i32 %12, 3
    store i32 %13, i32* %1
    br label %14

  14:
    %15 = load i32, i32* %1
    call void @putint(i32 %15)
    ret i32 0

  16:
    %17 = load i32, i32* %3
    %18 = load i32, i32* %2
    %19 = add i32 %17, %18
    %20 = sub i32 0, 1
    %21 = icmp ne i32 %19, %20
    br i1 %21, label %41, label %36 

  22:
    %23 = load i32, i32* %5
    %24 = load i32, i32* %4
    %25 = sub i32 %23, %24
    %26 = icmp eq i32 %25, 0
    br i1 %26, label %28, label %16 

  27:
    br label %11

  28:
    %29 = load i32, i32* %1
    %30 = icmp ne i32 %29, 4
    br i1 %30, label %31, label %16 

  31:
    br label %11

  32:
    %33 = load i32, i32* %1
    %34 = add i32 %33, 4
    store i32 %34, i32* %1
    br label %35

  35:
    br label %14

  36:
    %37 = load i32, i32* %1
    %38 = add i32 %37, 1
    %39 = srem i32 %38, 2
    %40 = icmp eq i32 %39, 1
    br i1 %40, label %42, label %35 

  41:
    br label %32

  42:
    br label %32
}
```

输出样例 3：

```c
5
```

### 样例 4

样例程序 4：

```c
int main() {
    int a;
    a = 10;
    if (+-!!!a) {
        a = - - -1;
    }
    else {
        a = 0;
    }
    putint(a);
    return 0;
}
```

示例 IR 4：

```llvm
declare void @putint(i32)
define dso_local i32 @main() {
    %1 = alloca i32
    store i32 10, i32* %1
    %2 = load i32, i32* %1
    %3 = icmp eq i32 %2, 0
    %4 = zext i1 %3 to i32
    %5 = icmp eq i1 %3, 0
    %6 = zext i1 %5 to i32
    %7 = icmp eq i1 %5, 0
    %8 = zext i1 %7 to i32
    %9 = zext i1 %7 to i32
    %10 = sub i32 0, %9
    %11 = icmp ne i32 %10, 0
    br i1 %11, label %12, label %18

12:
    %13 = sub i32 0, 1
    %14 = sub i32 0, %13
    %15 = sub i32 0, %14
    store i32 %15, i32* %1
    br label %16

16:
    %17 = load i32, i32* %1
    call void @putint(i32 %17)
    ret i32 0
18:
    store i32 0, i32* %1
    br label %16
}
```

输出样例 4：

```c
0
```