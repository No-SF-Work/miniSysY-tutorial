# Part 4 实现四则运算及模运算

在 Part 4 中，你的编译器需要实现四则运算以及模运算。

miniSysY 算符的优先级与结合性与 C 语言一致，文法中已经体现出了优先级定义，同一优先级的运算符运算顺序为从左到右运算。

如果你打算手工实现编译器，并且采用了递归下降的语法分析方式，在分析四则运算相关的语法时局部采用算符优先分析法可能效果会更好。

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
              | AddExp ('+' | '−') MulExp
MulExp     -> UnaryExp
              | MulExp ('*' | '/' | '%') UnaryExp
UnaryExp   -> PrimaryExp | UnaryOp UnaryExp
PrimaryExp -> '(' Exp ')' | Number
UnaryOp    -> '+' | '-'
```

其中除法取整和模运算规则与 C 语言 int 类型相同。

## 示例

样例程序 1：

```c
int main() {
    return 1 + (-2) * (3 / (4 - 5));
}
```

示例 IR 1：

```llvm
define dso_local i32 @main() {
    %1 = sub i32 0, 2
    %2 = sub i32 4, 5
    %3 = sdiv i32 3, %2
    %4 = mul i32 %1, %3
    %5 = add i32 1, %4
    ret i32 %5
}
```

示例返回值 1（lli 解释执行后）：

```c
7
```

输入样例 2：

```c
int main() {
    return 1 +-+ (- - - - - - - - -1);
}
```

示例 IR 2：

```llvm
define dso_local i32 @main() {
    %1 = sub i32 0, 1
    %2 = sub i32 0, %1
    %3 = sub i32 0, %2
    %4 = sub i32 0, %3
    %5 = sub i32 0, %4
    %6 = sub i32 0, %5
    %7 = sub i32 0, %6
    %8 = sub i32 0, %7
    %9 = sub i32 0, %8
    %10 = sub i32 0, %9
    %11 = add i32 1, %10
    ret i32 %11
}
```

示例返回值 2（lli 解释执行后）：

```c
2
```

输入样例 3：

```c
int main() {
    return 4 * (1 / 5) - 4 + 1 ** 1;
}
```

示例 IR 3：

编译器直接以**非 0 的返回值**退出。