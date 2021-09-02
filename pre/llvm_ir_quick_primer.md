# LLVM IR 快速上手
## 写在前面
**如果你对 LLVM IR 比较熟悉，可以跳过本节**

本节中的内容较多较难，可能需要多看几遍才能消化，由于篇幅与助教能力所限，部分地方可能讲得并不全面或者有所错误，实际以 [LLVM Lang Ref](https://llvm.org/docs/LangRef.html) 为准。

***

## LLVM IR 简介

LLVM IR 是 LLVM 编译器框架使用的一种中间表达形式。你可以把其想象为一种拥有无限多个寄存器的平台无关的汇编语言。

在开发编译器的时候，通常的做法是将源代码编译到某种中间表示（IR），再将这种中间表示翻译为目标体系架构的汇编（比如 MIPS，比如 X86），这种做法相对于直接将源代码翻译为目标体系架构的好处主要有两个。

首先，有很多优化技术是跨平台通用的（比如我们会作为挑战实验的死代码删除和常量折叠），我们只需要在 IR 上做一次这些优化，就能够在所有支持的体系架构上获得效果，这大大的减少了开发的工作量。

其次，假设我们有`m`种源语言和`n`中目标体系架构，如果我们直接将源代码翻译为目标体系架构的代码，那么我们就需要编写`m*n`个编译器才能满足每一种源语言都在所有的目标体系架构上适用。
而如果我们采用一种中间表达形式作为所有源语言编译到的目标语言，再将这种中间表达形式翻译到不同的目标体系架构上，我们就只需要实现`m+n`个编译器。

因此，常见的编译器都分为了三个部分，前端（front-end），中端（middle-end）以及后端（back-end）。每一部分都承担了不同的功能。

- 前端：将源语言编译到 IR
- 中端：对 IR 进行优化
- 后端：将 IR 翻译为目标语言

LLVM 也是按照这一结构设计的。
![](./../pic/llvm_compiler_pipeline.png)

LLVM IR 具有三种表示形式，一种是在内存中的编译中间语言（我们无法通过文件的形式得到）；一种是硬盘上存储的二进制中间语言（以`.bc`结尾），最后一种是可读的中间格式（以`.ll`结尾）。这三种中间格式是完全相等的。本次实验要求的输出内容是`.ll`形式的 LLVM IR。

## LLVM IR 示例程序
让我们通过一个小示例来快速熟悉 LLVM IR 的一些特性。

在学习这部分的时候，你可能需要和推荐的 LLVM IR 指令 一节对照。

我们将下面的这个 c 程序作为输入导出`.ll`形式的 LLVM IR
```c
//main.c
int foo(int first, int second) {
    return first + second;
}

int a = 5;

int main() {
    int b = 4;
    return foo(a, b);
}
```
我们在终端中输入`clang -emit-llvm -S main.c -o main.ll -O0`（如果你不知道为什么要输入这些内容，请先看 LLVM 工具链下载一节）以后，打开同目录下的`main.ll`文件，会看到完整的内容如下所示
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
根据个人使用的硬件与系统不同，部分内容会出现较小区别，比如`target triple`与`target datalayout`这些内容是程序的标签属性说明，在实验中不要求生成生成。程序中的`align`字段表示的是对齐字节，`dso_local`表示是变量和函数的的运行时抢占说明符，以`;`开头的字符串是程序生成的注释，这些内容在实验里也不要求生成。

将他们都删除以后我们的`.ll`文件依然是符合格式要求的，我们将其删去后留下我们需要关注的正文内容，并加上了方便同学们理解的注释
```llvm
; 全局变量以 `@` 为前缀，global 表明了其是全局变量
@a = global i32 5 ; 注意，@a 的类型是 i32* ，后面会详细说明
; 函数定义以 `define` 开头，i32 标明了函数的返回类型，`foo`是函数的名字，`@`是其前缀
;(i32 %0, i32 %1) 分别标明了第一个参数与第二个参数的类型以及他们的名字
define i32 @foo(i32 %0, i32 %1)  { ; %0 的类型是 i32 ，%1 的类型是 i32
  %3 = alloca i32 ; 申请了一个大小为 i32 类型所占大小的空间，%3 的类型是 i32*
  %4 = alloca i32 ; 申请了一个大小为 i32 类型所占大小的空间，%4 的类型是 i32*
  store i32 %0, i32* %3 ; 将 %0:i32 存入 %3:i32*
  store i32 %1, i32* %4 ; 将 %1:i32 存入 %4:i32*
  %5 = load i32, i32* %3 ; 从 %3:i32* 里 load 出一个 i32 类型的值，这个值的名字为 %5
  %6 = load i32, i32* %4 ; 从 %4:i32* 里 load 出一个 i32 类型的值，这个值的名字为 %6
  %7 = add nsw i32 %5, %6 ; 将 %5:i32 与 %6:i32 相加，和的名字为 %7 ,nsw 是 "No Signed Wrap" 的缩写，标识了无符号值运算
  ret i32 %7 ; 将 %7:i32 返回
}

define i32 @main() {
  %1 = alloca i32 
  %2 = alloca i32 
  store i32 0, i32* %1
  store i32 4, i32* %2
  %3 = load i32, i32* @a ; 从 @a:i32* 中 load 出一个 i32 类型的值，给 load 出来这个值的名字命名为 %3
  %4 = load i32, i32* %2
  %5 = call i32 @foo(i32 %3, i32 %4) 
  ; 调用函数 @foo ，i32 表示函数的返回值是 i32 类型的 
  ; 第一个参数是 %3:i32 ，第二个参数是 %4:i32 ，给函数的返回值命名为 %5 ，
  ret i32 %5 
}
```
虽然上面这个文件并没有包含本实验中可能使用到的所有特性与指令，但是已经展现出了很多值得注意的地方，比如

- 注释以`;`开头
- LLVM IR 是静态类型的（比如 32 位 Integer 的值拥有 i32 type）
- 局部变量的作用域是单个函数（比如 @main 中的 %1 是一个 i32* 类型的地址 而 @foo 中的 %1 是一个 i32 类型的值）
- 临时寄存器（或者说临时变量）拥有升序的名字（比如 @main 函数中的 %1,%2,%3）
- 全局变量与局部变量由前缀区分，全局变量和函数名以`@`为前缀，局部变量以`%`为前缀
- 大多数指令按照字面上的意思运行（alloca 申请地址，load 提取值，store 存值，add 做加法等）

## LLVM IR 的结构
### 总体结构
1. LLVM IR 汇编文件的基本单位称为`module`（本实验中涉及到的部分均为单 module )
2. 一个 `module` 中可以拥有多个顶层实体，比如`function`和`global variavle`。
3. 一个`function define`中至少有一个`basicblock`
4. 每个`basicblock`中有若干`instruction`，并以`terminator instruction`结尾



### 基本块（Basic Block）
一个基本块是包含了若干个指令以及一个终结指令的代码序列。基本块只会从终结指令退出，并且基本块的执行是原子性的，也就是说，如果基本块中的一条指令执行了，那么块内其他所有的指令也都会执行。这个约束**是通过代码的语义实现的**，基本块内部没有控制流，控制流是由多个基本块直接通过跳转指令实现的。

### 指令（Instruction）
指令(Instruction)指的是是LLVM IR中的非分支指令(non-branching Instruction)，通常用来进行某种计算或者是访问内存（比如上面例子中的 `add`,`load`），这些指令并不会改变程序的控制流。

值得一提的是，`call`指令也是非分支指令，因为在使用`call`调用函数时，我们并不关系被调用函数内部的具体情况（即使被调用函数内部存在控制流），只关心我们传入的参数以及被调用函数的返回值。

### 终结指令（Terminator instruction）
终结指令**一定**位于基本块的结尾，每个基本块的末尾也**一定**拥有一条终结指令。终结指令决定了程序的控制流的执行方向。比如，`ret`指令将程序控制流返回到当前函数的调用者，`br`指令根据标识符判断控制流执行的方向。

下面，我们根据两个例子来介绍程序的控制流是如何通过基本块与终结指令实现的
//未完待续
```c
//if.c
int main(){

}
```

```c
//while.c
```