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

输入样例 1：

```c
int main() {
    return ---(-1);
}
```

输出样例 1（lli 解释执行后）：

```c
1
```

输入样例 2：

```c
int main() {
    return +-+-010;
}
```

输出样例 2（lli 解释执行后）:

```c
8
```

输入样例 3：

```c
int main() {
    return -+(+-((-+(-+(1))));
}
```

输出样例 3：

编译器直接以**非 0 的返回值**退出。