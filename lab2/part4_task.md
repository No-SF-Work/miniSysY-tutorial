# Part 4 实现四则运算及模运算

在 Part 4 中，你的编译器需要实现四则运算以及模运算。

miniSysY 算符的优先级与结合性与 C 语言一致，文法中已经体现出了优先级定义，同一优先级的运算符运算顺序为从左到右运算。

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

## 示例

输入样例 1：

```c
int main() {
    return 1 + (-2) * (3 / (4 - 5));
}
```

输出样例 1：

```llvm
```

输入样例 2：

```c
int main() {
    return 1 +-+ (- - - - - - - - -1);
}
```

输出样例 2：

```llvm
```