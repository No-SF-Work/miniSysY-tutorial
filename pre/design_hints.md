# LLVM IR 中最重要的概念，以及编译器设计的提示

本节假设你已经阅读过本章节中的其他所有章节，并且具有一定的面向对象知识。

你第一次看到本节的时间应该是 lab2 刚开始的时候，如果你已经通过了 lab1——无论是递归下降还是使用工具分析。  那么你应该已经对这个实验具体要做什么有了较为直观的感受。
之前的几节介绍的是以文本形式存储的`.ll`形式的 LLVM IR, 这节我们将介绍 LLVM IR 在内存中的存储方式——也就是在程序运行时 LLVM 是怎么保存和维护 LLVM IR 的。并籍此给出一些实现编译器的建议。~~重构你的代码的时候到了（笑）~~

> 注：因为我们的实验只需要完成最简单的功能，所以你大可以自己设计自己的代码架构，只要最后通过测试点即可。

我们  
***强烈建议***  
在向下看之前先大致浏览一遍
[LLVM 中核心类的层次结构参考](https://www.llvm.org/docs/ProgrammersManual.html#the-core-llvm-class-hierarchy-reference)

## 最重要的概念：`Value`, `Use`, `User`
这是我们学习并设计自己的翻译到 LLVM IR 的编译器时需要认识的最重要的概念之一。

**一切皆 `Value`**  

这是个夸张的说法，不过在 LLVM IR 中，的确几乎所有的东西都是一个 `Value`，这里有张继承关系图。（src:https://llvm.org/doxygen/classllvm_1_1Value.html）
![](../files/Value.png)

我们重点关注这么几项
- `BasicBlock`，`Argument`，`User` 都继承了 `Value` 类
- `Constant` 和 `Instruction` 继承了 `User`
- 图中没有 `Function` 类，但实际上 `Function` 类通过多重继承继承了 `Constant` 类，所以 `Function` 也是 `Value` 和 `User`

`BasicBlock` 表示的是基本块类，`Arugument` 表示的是函数的形参，`Constant` 表示的是形如 `i32 4` 的常量，`Instruction` 表示的是形如 `add i32 %a,%b` 的指令

`Value` 是一个非常基础的基类，一个继承于 `Value` 的子类表示它的结果可以被其他地方使用。
一个继承于 `User` 的类表示它会使用一个或多个 `Value` 对象
根据 `Value` 与 `User` 之间的关系，还可以引申出 use-def 链和 def-use 链这两个概念。use-def 链是指被某个 `User` 使用的 `Value` 列表，def-use 链是使用某个 `Value` 的 `User` 列表。实际上，LLVM 中还定义了一个 `Use` 类，`Use` 就是上述的使用关系中的一个边。
下面是从 LLVM 2.0.0 中节选的代码（新版本的太复杂了，增加了理解难度）

```c++
class Value {
    // 无关代码private:
    Use *UseList;
}
```
```c++
class User : public Value {
    // 无关代码
  protected:
    Use *OperandList;
    unsigned NumOperands;
}
```
```c++
class Use {
    // 无关代码
    Use *Next, **Prev;
    Value *Val;
    User *U;
}
```
- `class Value` 中的 `UseList` 保存了**使用了这个 `Value` 的 `User` 列表**，这对应着 def-use 关系。
- `class User` 中的 `OperandList` 保存了**这个 `User` 使用的 `Value` 的列表**，这对应着 use-def 关系。
- `class Use` 中的 `Value, User` 的引用，维护了这条边的两个结点以及使用和被使用的关系，从 `User` 能够通过 `OperandList` 找到这个 `User` 使用的 `Value`, 从 `Value` 也能找到对应的使用这个 `Value` 的 `User`。

以一段手写的 `.ll` 代码为例

```llvm
define dso_local i32 @main(){
    %x0 = add i32 5, 0
    %x1 = add i32 5, %x0
    ret i32 %x1
}
```

其在内存中的存储形式大概是这样的

- `%x0` 是一个 `Instruction` 实例，它的 `OperandList` 里有两个值，一个是 `Constant` 的实例 `5`，另一个是 `Constant` 的实例 `0`；
- `%x1` 是一个 `Instruction` 实例，它的 `OperandList` 里有两个值，一个是 `Constant` 的实例 `5`，另一个是 `Instruction` 的实例 `%x0`；
- `ret` 是一个 `Instruction` 实例，它的 `OperandList` 里有一个值，是 `Instruction` 的实例 `%x0`。

需要注意的是：clang 默认生成的虚拟寄存器是按数字顺序命名的，LLVM 限制了所有数字命名的虚拟寄存器必须严格地从 0 开始递增，且每个函数参数和基本块都会占用一个编号。如果你不能确定怎样用数字命名虚拟寄存器，请使用字符串命名虚拟寄存器。

下面的 LLVM IR 中给出了一些 lli 解释执行成功或失败的情况：

```llvm
; 解释执行成功
define dso_local i32 @main(){
    %1 = sub i32 0, 15
    %2 = sub i32 0, %1
    %3 = add i32 0, %2
    ret i32 %3
}

; 解释执行成功
define dso_local i32 @main(){
    %1 = sub i32 0, 15
    %x = sub i32 0, %1
    %2 = add i32 0, %x
    ret i32 %2
}

; 解释执行失败
; lli: test.ll:2:5: error: instruction expected to be numbered '%1'
;     %0 = sub i32 0, 15
define dso_local i32 @main(){
    %0 = sub i32 0, 15
    %1 = sub i32 0, %0
    %2 = add i32 0, %1
    ret i32 %2
}

; 解释执行成功
define dso_local i32 @main(){
  _entry:
  ; 显式地给基本块指定一个名称
    %0 = sub i32 0, 15
    %1 = sub i32 0, %0
    %2 = add i32 0, %1
    ret i32 %2
}
```

