# LLVM IR 快速上手

## 写在前面

**如果你对 LLVM IR 比较熟悉，可以跳过本节**

本节的内容较多较难，可能需要多看几遍才能消化。由于本节篇幅较长且助教精力有限，部分地方可能讲得不全面或存在错误，实际以 [LLVM Lang Ref](https://llvm.org/docs/LangRef.html) 以及 [LLVM Programmer Manual](https://llvm.org/docs/ProgrammersManual.html#the-core-llvm-class-hierarchy-reference) 为准。

## LLVM IR 简介

在开发编译器时，通常的做法是将源代码编译到某种中间表示（Intermediate Representation，一般称为 IR），然后再将 IR 翻译为目标体系结构的汇编（比如 MIPS 或 X86），这种做法相对于直接将源代码翻译为目标体系结构的好处主要有两个：

- 首先，有一些优化技术是目标平台无关的（例如作为我们实验挑战任务的死代码删除和常量折叠），我们只需要在 IR 上做这些优化，再翻译到不同的汇编，这样就能够在所有支持的体系结构上实现这种优化，这大大的减少了开发的工作量。

- 其次，假设我们有 `m` 种源语言和 `n` 种目标平台，如果我们直接将源代码翻译为目标平台的代码，那么我们就需要编写 `m * n` 个不同的编译器。然而，如果我们采用一种 IR 作为中转，先将源语言编译到这种 IR ，再将这种 IR 翻译到不同的目标平台上，那么我们就只需要实现 `m + n` 个编译器。

因此，目前常见的编译器都分为了三个部分，前端（front-end），中端（middle-end）以及后端（back-end），每一部分都承担了不同的功能：

- 前端：将源语言编译到 IR
- 中端：对 IR 进行优化
- 后端：将 IR 翻译为目标语言

同理，LLVM 也是按照这一结构设计的。

![](./../pic/llvm_compiler_pipeline.png)

LLVM IR 具有三种表示形式，这三种中间格式是完全等价的：
- 在内存中的编译中间语言（我们无法通过文件的形式得到）
- 在硬盘上存储的二进制中间语言（格式为 `.bc`）
- 人类可读的代码语言（格式为 `.ll`）

本次实验要求输出 `.ll` 形式的 LLVM IR。

## LLVM IR 示例程序

让我们通过一个小示例来快速熟悉 LLVM IR 的一些特性。在学习这部分的时候，你可能需要和推荐的 LLVM IR 指令 一节对照。

在接下来的任务中，我们会生成下面这个 C 程序的 `.ll` 形式 LLVM IR：

```c
// main.c
int foo(int first, int second) {
    return first + second;
}

int a = 5;

int main() {
    int b = 4;
    return foo(a, b);
}
```

我们在命令行中输入 `clang -emit-llvm -S main.c -o main.ll -O0`（如果你还不知道这个命令中各个选项的含义，请先去看「LLVM 相关工具链下载」一节），然后打开同目录下的 `main.ll` 文件，会看到生成的内容如下所示：

```llvm
; ModuleID = 'main.c'
source_filename = "main.c"
target datalayout = "e-m:e-p270:32:32-p271:32:32-p272:64:64-i64:64-f80:128-n8:16:32:64-S128"
target triple = "x86_64-pc-linux-gnu"

@a = dso_local global i32 5, align 4

; Function Attrs: noinline nounwind optnone sspstrong uwtable
define dso_local i32 @foo(i32 %0, i32 %1) #0 {
  %3 = alloca i32, align 4
  %4 = alloca i32, align 4
  store i32 %0, i32* %3, align 4
  store i32 %1, i32* %4, align 4
  %5 = load i32, i32* %3, align 4
  %6 = load i32, i32* %4, align 4
  %7 = add nsw i32 %5, %6
  ret i32 %7
}

; Function Attrs: noinline nounwind optnone sspstrong uwtable
define dso_local i32 @main() #0 {
  %1 = alloca i32, align 4
  %2 = alloca i32, align 4
  store i32 0, i32* %1, align 4
  store i32 4, i32* %2, align 4
  %3 = load i32, i32* @a, align 4
  %4 = load i32, i32* %2, align 4
  %5 = call i32 @foo(i32 %3, i32 %4)
  ret i32 %5
}

attributes #0 = { noinline nounwind optnone sspstrong uwtable "disable-tail-calls"="false" "frame-pointer"="all" "less-precise-fpmad"="false" "min-legal-vector-width"="0" "no-infs-fp-math"="false" "no-jump-tables"="false" "no-nans-fp-math"="false" "no-signed-zeros-fp-math"="false" "no-trapping-math"="true" "stack-protector-buffer-size"="8" "target-cpu"="x86-64" "target-features"="+cx8,+fxsr,+mmx,+sse,+sse2,+x87" "tune-cpu"="generic" "unsafe-fp-math"="false" "use-soft-float"="false" }

!llvm.module.flags = !{!0, !1, !2}
!llvm.ident = !{!3}

!0 = !{i32 1, !"wchar_size", i32 4}
!1 = !{i32 7, !"PIC Level", i32 2}
!2 = !{i32 7, !"PIE Level", i32 2}
!3 = !{!"clang version 12.0.1"}
```

根据个人使用的硬件与系统不同，部分内容会出现较小区别，例如 `target triple` 与 `target datalayout` 是程序的标签属性说明，在我们的实验中并不要求生成它们；`align` 字段描述了程序的对齐属性；`dso_local` 是变量和函数的的运行时抢占说明符；以 `;` 开头的字符串是 LLVM IR 的注释（这个最好记住）……这些内容在本实验里也不要求生成。

将上面谈到的内容都删除以后，我们的 `.ll` 文件依然是符合格式的。我们将无用的语句删去后，就只留下了需要关注的正文内容。为了方便同学的理解，我们还加上了相关的注释：

```llvm
; 所有的全局变量都以 @ 为前缀，后面的 global 关键字表明了它是一个全局变量
@a = global i32 5 ; 注意，@a 的类型是 i32* ，后面会详细说明

; 函数定义以 `define` 开头，i32 标明了函数的返回类型，其中 `foo`是函数的名字，`@` 是其前缀
; 函数参数 (i32 %0, i32 %1) 分别标明了其第一、第二个参数的类型以及他们的名字
define i32 @foo(i32 %0, i32 %1)  { ; 第一个参数的名字是 %0，类型是 i32；第二个参数的名字是 %1，类型是 i32。
  ; 以 % 开头的符号表示虚拟寄存器，你可以把它当作一个临时变量（与全局变量相区分），或称之为临时寄存器
  %3 = alloca i32 ; 为 %3 分配空间，其大小与一个 i32 类型的大小相同。%3 类型即为 i32*
  %4 = alloca i32 ; 同理，%4 类型为 i32*

  store i32 %0, i32* %3 ; 将 %0（i32）存入 %3（i32*）
  store i32 %1, i32* %4 ; 将 %1（i32）存入 %4（i32*）

  %5 = load i32, i32* %3 ; 从 %3（i32*）中 load 出一个值（类型为 i32），这个值的名字为 %5
  %6 = load i32, i32* %4 ; 同理，从 %4（i32*） 中 load 出一个值给 %6（i32）

  %7 = add nsw i32 %5, %6 ; 将 %5（i32） 与 %6（i32）相加，其和的名字为 %7。nsw 是 "No Signed Wrap" 的缩写，表示无符号值运算

  ret i32 %7 ; 返回 %7（i32）
}

define i32 @main() {
  ; 注意，下面出现的 %1，%2……与上面的无关，即每个函数的临时寄存器是独立的
  %1 = alloca i32
  %2 = alloca i32

  store i32 0, i32* %1
  store i32 4, i32* %2

  %3 = load i32, i32* @a
  %4 = load i32, i32* %2

  ; 调用函数 @foo ，i32 表示函数的返回值类型
  ; 第一个参数是 %3（i32），第二个参数是 %4（i32），给函数的返回值命名为 %5
  %5 = call i32 @foo(i32 %3, i32 %4)

  ret i32 %5
}
```

虽然上面这个文件并没有包含本实验中可能使用到的所有特性与指令，但是已经展现出了很多值得注意的地方，比如：
- 注释以 `;` 开头
- LLVM IR 是静态类型的（即在编写时每个值都有明确的类型）
- 局部变量的作用域是单个函数（比如 `@main` 中的 `%1` 是一个 `i32*` 类型的地址，而 `@foo` 中的 `%1` 是一个 `i32` 类型的值）
- 临时寄存器（或者说临时变量）拥有升序的名字（比如 `@main` 函数中的 `%1`，`%2`，`%3`）
- 全局变量与局部变量由前缀区分，全局变量和函数名以 `@` 为前缀，局部变量以 `%` 为前缀
- 大多数指令与字面含义相同（`alloca` 分配内存并返回地址，`load` 从内存读出值，`store` 向内存存值，`add` 用于加法等）

## LLVM IR 的结构
如果看完下面的内容以后依然对 LLVM IR 的结构不甚了了，[LLVM Programmer Manual](https://llvm.org/docs/ProgrammersManual.html#the-core-llvm-class-hierarchy-reference) 里的内容可能能起到帮助。

### 总体结构

1. LLVM IR 文件的基本单位称为 `module`（本实验中涉及到的部分均为单 `module`，因为本实验只涉及到单文件编译）
2. 一个 `module` 中可以拥有多个顶层实体，比如 `function` 和 `global variavle`
3. 一个 `function define` 中至少有一个 `basicblock`
4. 每个 `basicblock` 中有若干 `instruction`，并且都以 `terminator instruction` 结尾

### 函数定义与函数声明 (Define&Delcare)
LLVM 中

### 基本块（Basic Block）

一个基本块是包含了若干个指令以及一个终结指令的代码序列。

基本块只会从终结指令退出，并且基本块的执行是原子性的，也就是说，如果基本块中的一条指令执行了，那么块内其他所有的指令也都会执行。这个约束**是通过代码的语义实现的**。基本块内部没有控制流，控制流是由多个基本块直接通过跳转指令实现的。

形象地讲，一个基本块中的代码是顺序执行的，且顺序执行的代码都属于一个基本块。

例如你有一份不含跳转（没有分支、循环）也没有函数调用的、只会顺序执行的代码，那么这份代码只有一个基本块。

然而，一旦在中间加入一个 `if-else` 语句，那么代码就会变成四个基本块：`if` 上面的代码仍然是顺序执行的，在一个基本块中；`then` 和 `else` 各自部分的代码也都是顺序执行的，因此各有一个基本块；`if` 之后的代码也是顺序执行的，也在一个基本块中。所以总共四个基本块。

### 指令（Instruction）

指令指的是 LLVM IR 中的非分支指令（non-branching Instruction），通常用来进行某种计算或者是访存（比如上面例子中的 `add`、`load`），这些指令并不会改变程序的控制流。

值得一提的是，`call` 指令也是非分支指令，因为在使用 `call` 调用函数时，我们并不关系被调用函数内部的具体情况（即使被调用函数内部存在的控制流），而是只关心我们传入的参数以及被调用函数的返回值，因此这并不会影响我们当前程序的控制流。

### 终结指令（Terminator instruction）

终结指令**一定**位于某个基本块的末尾（否则中间就改变了基本块内的控制流）；反过来，每个基本块的末尾也**一定**是一条终结指令（否则仍然是顺序执行的，基本块不应该结束）。终结指令决定了程序控制流的执行方向。例如，`ret` 指令会使程序的控制流返回到当前函数的调用者（可以理解为 `return`），`br` 指令表示根据标识符选择一个控制流的方向（可以理解为 `if`）。

下面，我们通过一个例子来介绍程序的控制流是如何通过基本块与终结指令描述的：
```c
//if.c
int main() {
    int a = getint();
    int b = getint();
    int c = 0;
    if (a == 0) {
        c = 5;
    } else {
        c = 10;
    }
    putint(c);
    return 0;
}
```

```c
//while.c
int getint();

int putint(int a);

int main() {
    int a = getint();
    int c = 1;
    while (a != 0) {
        c = c * (c + 1);
        a = a - 1;
    }
    putint(c);
    return 0;
}
```

将 `if.c` 导出为 LLVM IR 并且删去实验无关部分后的代码如下所示
``` llvm
define dso_local i32 @main() #0 {
  %1 = alloca i32
  %2 = alloca i32
  %3 = alloca i32
  %4 = alloca i32
  store i32 0, i32* %1
  %5 = call i32 (...) @getint()
  store i32 %5, i32* %2
  %6 = call i32 (...) @getint()
  store i32 %6, i32* %3
  %7 = load i32, i32* %2
  %8 = load i32, i32* %3
  %9 = icmp eq i32 %7, %8
  br i1 %9, label %10, label %11

10:                                             
  store i32 15, i32* %4
  br label %12

11:                                               
  store i32 30, i32* %4
  br label %12

12:                                               
  %13 = load i32, i32* %4
  %14 = call i32 @putint(i32 %13)
  ret i32 0
}
```
//todo exp

将 `while.c` 导出为 LLVM IR  并且删去实验无关部分后的代码如下所示
```llvm

```
