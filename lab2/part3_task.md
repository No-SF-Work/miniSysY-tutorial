# Part 3 实现正号、负号

在 Part 3 中，你的编译器需要支持正号、负号以及表达式中可能存在的括号。

你需要支持的语法规则如下（以 `CompUnit` 为开始符号）：

```
CompUnit   -> FuncDef
FuncDef    -> FuncType Ident '(' ')' Block
FuncType   -> 'int'
Ident      -> 'main'
Block      -> '{' Stmt '}'
Stmt       -> 'return' Exp ';'
Exp        -> AddExp
AddExp     -> MulExp
MulExp     -> UnaryExp
UnaryExp   -> PrimaryExp | UnaryOp UnaryExp
PrimaryExp -> '(' Exp ')' | Number
UnaryOp    -> '+' | '-'
```

## 示例

样例程序 1：

```c
int main() {
    return ---(-1);
}
```

示例 IR 1：

```llvm
define dso_local i32 @main() {
    %1 = sub i32 0, 1
    %2 = sub i32 0, %1
    %3 = sub i32 0, %2
    %4 = sub i32 0, %3
    ret i32 %4
}
```

示例返回值 1（lli 解释执行后）：

```c
1
```

样例程序 2：

```c
int main() {
    return +-+-010;
}
```

示例 IR 2：

```llvm
define dso_local i32 @main() {
    %1 = sub i32 0, 8
    %2 = sub i32 0, %1
    ret i32 %2
}
```

示例返回值 2（lli 解释执行后）:

```c
8
```

样例程序 3：

```c
int main() {
    return -+(+-((-+(-+(1))));
}
```

示例 IR 3：

编译器直接以**非 0 的返回值**退出。