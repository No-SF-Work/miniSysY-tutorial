# LLVM IR 中最重要的概念，以及编译器设计的提示

本节假设你已经阅读过本章节中的其他所有章节，并且具有一定的面向对象知识。

你第一次看到本节的时间应该是 lab2 刚开始的时候，如果你已经通过了 lab1——无论是递归下降还是使用工具分析。  那么你应该已经对这个实验具体要做什么有了较为直观的感受。
之前的几节介绍的是以文本形式存储的`.ll`形式的 LLVM IR, 这节我们将介绍 LLVM IR 在内存中的存储方式——也就是在程序运行时 LLVM 是怎么保存和维护 LLVM IR 的。并籍此给出一些实现编译器的建议。~~重构你的代码的时候到了（笑）~~
>注：因为我们的实验只需要完成最简单的功能，所以你大可以自己设计自己的代码架构，只要最后通过测试点即可。

我们  
***强烈建议***  
在向下看之前先大致浏览一遍
[LLVM 中核心类的层次结构参考](https://www.llvm.org/docs/ProgrammersManual.html#the-core-llvm-class-hierarchy-reference)

## 最重要的概念：`Value`,`Use`,`User`
这是我们学习并设计自己的翻译到 LLVM IR 的编译器时需要认识的最重要的概念之一。

**一切皆`Value`**  

这是个夸张的说法，不过在 LLVM IR 中，的确几乎所有的东西都是一个`Value`，这里有张继承关系图。（src:https://llvm.org/doxygen/classllvm_1_1Value.html）
![](../files/Value.png)

我们重点关注这么几项
- `BasicBlock`，`Argument`，`User`都继承了`Value`类
- `Constant`和`Instruction`继承了`User`
- 图中没有`Function`类，但实际上`Function`类通过多重继承继承了`Constant`类，所以`Function`也是`Value`和`User`

`BasicBlock`表示的是基本块类，`Arugument`表示的是函数的形参，`Constant`表示的是形如`i32 4`的常量，`Instruction`表示的是形如`add i32 %a,%b`的指令

`Value`是一个非常基础的基类，一个继承于`Value`的子类表示它的结果可以被其他地方使用。
一个继承于`User`的类表示它会使用一个或多个`Value`对象
根据 Value 与 User 之间的关系，还可以引申出 use-def 链和 def-use 链这两个概念。use-def 链是指被某个 User 使用的 Value 列表，def-use 链是使用某个 Value 的 User 列表。实际上，LLVM 中还定义了一个 Use 类，Use 就是上述的使用关系中的一个边。
下面是从 llvm2.0.0 中节选的代码（新版本的太复杂了，增加了理解难度）
``` c++
class Value{
    ......//无关代码
private:
    Use *UseList;
}
```
```c++
class User : public Value{
.....//无关代码
protected:
    Use *OperandList;
    unsigned NumOperands;
}
```
```c++
class Use{
    ......//无关代码
    Use *Next,**Prev;
    Value *Val;
    User *U;
}
```
- `class Value`中的 `UseList`保存了**使用了这个`Value`的`User`列表**，这对应着 def-use 关系。
- 看`class User`中的`OperandList`保存了**这个 User 使用的 Value 的列表**，这对应着 use-def 关系。
- `class Use`中的`Value,User`的引用，维护了这条边的两个结点以及使用和被使用的关系，从`User`能够通过`OperandList`找到这个`User`使用的`Value`, 从`Value`也能找到对应的使用这个`Value`的`User`

以一段手写的`.ll`代码为例
```llvm
define dso_local i32 @main(){
%x0 =add i32 5,0
%x1 =add i32 5,%x0
ret i32 %x1
}
```
其在内存中的存储形式大概是这样的

- `%x0`是一个`Instruction`实例，他的 OperandList 里面有两个值，一个是`Constant`的实例`5`，另一个是`Constant`的实例`0`
- `%x1`是一个`Instruction`实例，他的 OperandList 里有两个值，一个是`Constant`的实例`5`，另一个是`Instruction`的实例`%x0`
- `ret`是一个`Instruction`实例，他的 OperandList 里有一个值，是`Instruction`的实例`%x0`

由于 LLVM IR 是 SSA 形式的，所以我们通常能够将名字和需要名字的指令绑定——比如`add i32 5,0`是一个名字叫`%x0`的`Instrucion`。并且将对指令的命名推迟到需要导出`.ll`格式的文件的时候。

在导出 LLVM IR 的时候，我们能够通过查询某条指令的`Operand`的名字来输出正确的名字，比如对上述代码而言，
1. 我们先遍历到了`%x0`所代表的这条指令，而这个时候它还没有`%x0`这个名字，我们按照自己设定的顺序，命名它为`%x0`
2. 然后继续向下遍历到了`%x1`(同样的，刚遍历到的时候它还没有`%x1`这个名字)然后发现他的`Operand`一个是常量`5`，另一个`Operand`是一条指令，我们再去查询这个`Operand`的名字，发现它叫`%x0`，于是，我们给了这条指令一个新名字`%x1`，并且导出了`%x1 =add i32 5,%x0`。

这部分的内容比较抽象，你可能看晕了~~我也写麻了~~，不过，我们的示例编译器也会按照类似的方法实验，我们会在适当的时候放出示例编译器的代码，你可以到时候阅读并参考示例编译器的实现。