# Part 1 仅有 main 函数与 return 的编译器

## 任务

在 Part 1 中，你需要完成一个可以将一个 `main` 函数（仅有一条 `return` 语句）编译成 LLVM IR 的编译器。

你需要支持的语法规则如下（以 `CompUnit` 为开始符号）：

```
CompUnit -> FuncDef
FuncDef  -> FuncType Ident '(' ')' Block
FuncType -> 'int'
Ident    -> 'main'
Block    -> '{' Stmt '}'
Stmt     -> 'return' Number ';'
```

其中 Number 可以表示八进制、十进制、十六进制数字，文法如下：

```
Number             -> decimal-const | octal-const | hexadecimal-const
decimal-const      -> nonzero-digit | decimal-const digit
octal-const        -> '0' | octal-const octal-digit
hexadecimal-const  -> hexadecimal-prefix hexadecimal-digit
                      | hexadecimal-const hexadecimal-digit
hexadecimal-prefix -> '0x' | '0X'
nonzero-digit      -> '1' | '2' | '3' | '4' | '5' | '6' | '7' | '8' | '9'
octal-digit        -> '0' | '1' | '2' | '3' | '4' | '5' | '6' | '7'
hexadecimal-digit  -> '0' | '1' | '2' | '3' | '4' | '5' | '6' | '7' | '8' | '9'
                      | 'a' | 'b' | 'c' | 'd' | 'e' | 'f'
                      | 'A' | 'B' | 'C' | 'D' | 'E' | 'F'
```

在将 `Number` 翻译成 LLVM IR 中的常量数字时，你需要注意进制的转换。输入保证 `Number` 转换成十进制后范围为 `0 <= Number <= 2147483647`，不会出现范围之外的数字。

## 示例

输入样例 1：

```c
int main() {
    return 123;
}
```

输出样例 1：

```llvm
define dso_local i32 @main(){
    ret i32 123
}
```

输入样例 2：

```c
int main() {
    return 0:
} 
```

输出样例 2：

如果编译过程中出现了错误（语法、语义等错误），你的编译器应当直接以**非 0 的返回值**退出。
