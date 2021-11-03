# Lab 4 实验指导

## 基本块

> 基本块相关知识会在课程“代码优化”一章中系统地学习到。

基本块是一段顺序执行的的指令，控制流只能从一个基本块的第一条指令开始执行，从最后一条指令退出基本块，或是跳转到其他基本块（包括自己）的第一条指令，或是退出程序。基本块的最后一条指令必须是一个跳转指令或返回指令，且中间不会出现跳转和返回指令。

之前的实验中，你生成的 LLVM IR 其实就是在一个基本块中的，从第一条指令开始执行，直到最后一条 `ret` 指令退出运行。而在 Lab 4 中，我们引入了 `if` 和 `else`，这使得程序不再是按照 IR 一条一条地顺序执行下去，IR 需要划分成多个不同的基本块，控制流在这些基本块之间跳转。

以下面一段代码为例：

```c
int main() {
    int a = getint();
    if (a == 1) {
        putint(1);
    } else {
        putint(2);
    }
    return 0;
}
```

```llvm
declare i32 @getint()
declare void @putint(i32)
define dso_local i32 @main() {
    %1 = alloca i32
    %2 = call i32 @getint()
    store i32 %2, i32* %1
    %3 = load i32, i32* %1
    %4 = icmp eq i32 %3, 1
    br i1 %4, label %5, label %6

5:
    call void @putint(i32 1)
    br label %7

6:
    call void @putint(i32 2)
    br label %7

7:
    ret i32 0
}
```

代码中共有 4 个基本块，第一个基本块被隐式命名成了 `0`（当然你也可以显式地给它命名），第一个基本块中包括了一条 `icmp` 指令和一条 `br` 指令，`icmp` 指令将 `a` 和 `1` 比较，结果存放在 `%4` 中，`br` 指令是一个条件跳转指令，当 `%4` 为 `true` 时跳转到 `5`，`%4` 为 `false` 时跳转到 `6`。`5` 和 `6` 的 `br` 指令是无条件跳转，即执行到这里后直接跳转到 `7`。

**再次强调**：clang 默认生成的虚拟寄存器是按数字顺序命名的，LLVM 限制了所有数字命名的虚拟寄存器必须严格地从 0 开始递增，且每个函数参数和基本块都会占用一个编号。如果你不能确定怎样用数字命名虚拟寄存器，请使用字符串命名虚拟寄存器。

## LLVM IR 指令指导

在这里 [推荐指令](../pre/suggested_insts.md) 你可以回顾之前介绍到的在本次实验中出现的一些新指令。

本次 lab 中将会出现多个基本块，这意味着你需要配合使用`zext`,`and`,`or`和`icmp`指令来完成控制流在基本块之间的跳转。

LLVM IR 是一个强类型的语言，这意味着你无法进行隐式转换。如果想要将一个 `i1`类型的变量转换为`i32`类型的变量，你必须使用`zext`指令来进行显式的类型转换。

`zext`指令的使用方法是`<result> = zext <ty> <value> to <ty2> `, 下面是一个简单的例子
```llvm
define i32 @main() {
%x = add i1 0,0
%x1 = zext i1 %x to i32
ret i32 %x1
```
`icmp`是比较指令，它的使用方法是`<result> = icmp <cond> <ty> <op1>, <op2>`，需要注意的是，`icmp`的`<result>`是`i1`的。

`and`和`or`是按位与/或指令 `<result> = and/or <ty> <op1>, <op2>`，将被用来实现较复杂条件语句的运算。

`br`是跳转指令，分为无条件和有条件两种情况，`br i1 <cond>, label <iftrue>, label <iffalse>`（有条件跳转），`br label <dest>`（无条件跳转）
下面是一个简单的例子
```llvm
define i32 @main() {
block_a:
    %x=add i32 0,123
    %y=add i32 0,321 ;
    %m=add i32 0,123
    %n=add i32 0,123 ;
    %res_xy = icmp eq i32 %x,%y
    %res_mn = icmp eq i32 %m,%n
    %cond = or i1 %res_xy,%res_mn; 你可以把 and 改成 or 看有什么变化
    br i1 %cond ,label %block_true,label %block_false
block_true:
    ret i32 0
block_false:
    ret i32 1
}
```
用`c`语言的逻辑，这段代码的意思大概是这样的
```c
if((x==y)&&(m==n)){
    return 0;
}
return 1;
```
**短路求值预演**

我们可以通过多个跳转指令和多个比较指令来实现条件语句中的符合条件，并实现短路运算。会在挑战实验中较为详细介绍，在此不再展开，有兴趣可以浏览这个链接。

https://www.zhihu.com/question/53273670
