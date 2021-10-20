# LLVM IR 中最重要的概念，以及编译器设计的提示

本节假设你已经阅读过本章节中的其他所有章节，并且具有一定的面向对象知识。

你第一次看到本节的时间应该是 lab2 刚开始的时候，如果你已经通过了 lab1——无论是递归下降还是使用工具分析。  那么你应该已经对这个实验具体要做什么有了较为直观的感受。
之前的几节介绍的是以文本形式存储的`.ll`形式的LLVM IR,这节我们将介绍 LLVM IR在内存中的存储方式——也就是在程序运行时，LLVM IR在内存中的存储方式。并籍此给出一些实现编译器的建议。~~重构你的代码的时候到了（笑）~~

我们  
***强烈建议***  
在向下看之前先浏览一遍
[LLVM 中核心类的层次结构参考](https://www.llvm.org/docs/ProgrammersManual.html#the-core-llvm-class-hierarchy-reference)

## 最重要的概念：`Value`,`Use`,`User`
这是我们学习并设计自己的翻译到LLVM IR的编译器时需要认识的最重要的概念之一。

**一切皆`Value`**  

这是个比较夸张的说法，不过在LLVM IR中，的确几乎所有的东西都是一个`Value`

[这里有张震撼人心的继承关系图，这里的地方太小放不下](https://llvm.org/doxygen/classllvm_1_1Value.html)

//todo haven't finish