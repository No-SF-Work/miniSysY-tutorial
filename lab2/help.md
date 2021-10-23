# Lab 2 实验指导

在你使用 clang 测试生成样例中的部分代码时，需要注意 C 语言中存在 `++` 和 `--` 运算符，连写的 `+` 和 `-` 会首先被词法分析识别成 `++` 和 `--` 运算符，导致编译失败，你需要在连写的 `+` 和 `-` 之间加上空格。miniSysY 的文法中不会出现 `++` 和 `--` 运算符，你在词法分析时不需要为此预留额外的兼容代码。


下面是一个例子：

```c
int main() {
    return 1 +-+ (- - -15) / 0x5;
}
```

```llvm
define dso_local i32 @main(){
    %x0 = sub i32 0, 15
    %x1 = sub i32 0, %x0
    %x2 = sub i32 0, %x1
    %x3 = sub i32 0, %x2
    %x4 = sdiv i32 %x3, 5
    %x5 = add i32 1, %x4
    ret i32 %x5
}
```

关于 LLVM IR 的进一步介绍：[LLVM 中的 SSA](../pre/llvm_ir_ssa.md)、[LLVM IR 中最重要的概念，以及编译器设计的提示](../pre/design_hints.md)。
