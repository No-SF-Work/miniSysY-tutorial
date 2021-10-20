# LLVM 中的 SSA

如果看完本节以后还有感到迷惑的地方可以看下面这个视频

[B 站源](https://www.bilibili.com/video/BV1oE411y711)     [油管源](https://www.youtube.com/watch?v=m8G_S5LwlTo)

在阅读本节前我们默认你已经阅读过了 [LLVM IR 快速上手](https://buaa-se-compiling.github.io/miniSysY-tutorial/pre/llvm_ir_quick_primer.html) 一节。

**为什么要介绍这些内容？**

在 LLVM IR 中，变量是以 SSA 形式存在的，为了生成正确的 LLVM IR 并实现我们的实验，对 SSA 的知识是必不可少的，LLVM IR 的代码有两种状态，分别是用内存空间代替`phi`的不完全 SSA 以及以`phi`形式存在的 SSA。因为 lab2 中已经涉及到了表达式相关的内容，所以本小节将对 SSA 进行简单的介绍，并指导大家设计合适的中间代码架构，以免实验到了后期因为设计问题出现大幅度重构。

在基础实验中，你只需要实现较简单的 alloca 形式的 IR，而生成 phi 形式的完全 SSA IR 将被我们作为挑战实验发布。

## SSA 介绍

> 在编译器的设计中，**静态单赋值形式**（static single assignment form，通常简写为** SSA form **或是** SSA**）是中间表示（IR，intermediate representation) 的特性，每个变量仅被赋值一次。——维基百科

在 IR 中，每个变量都在使用前都必须先定义，且每个变量只能被赋值一次（如果套用 C++的术语，就是说每个变量只能被初始化，不能被赋值），所以我们称 IR 是静态单一赋值的。
举一个例子，如果想要返回 `1*2+3`的值，我们下意识地就会像这样写。

```llvm
%0 = mul i32 1, 2
%0 = add i32 %0, 3
ret i32 %0
```

很合理，不是吗？但这样写实际上是错的，因为变量`%0`被赋值了两次。我们需要修改为

```llvm
%0 = mul i32 1, 2
%1 = add i32 %0, 3
ret i32 %1
```

### **SSA 的好处（拓展阅读）**

对人类来说，第一种做法似乎更为直观，但是对于编译器来说，第二种做法带来的好处更多。

SSA 可以简化编译器的优化过程，譬如说，考虑这段代码

``` 
d1: y := 1
一些无关代码
d2: y := 2
一些无关代码
d3: x := y
```

我们很容易可以看出第一次对`y`赋值是不必要的，在对`x`赋值时使用的`y`的值时第二次赋值的结果，但是编译器必须要经过一个定义可达性 (Reaching definition) 分析才能做出判断。编译器是怎么分析呢？首先我们先介绍几个概念（这些概念将会在我们课程的后半部分出现，我们在这里先 look ahead 一下，不完全理解也不影响实验的进行）：

- 定义：对变量`x`进行定义的意思是在某处会/可能给`x`进行赋值，比如上面的`d1`处就是一个对`y`的定义。
- kill：当一个变量有了新的定义后，旧有的定义就会被`kill`掉，在上面的语句中`d2`就`kill`掉了`d1`中对 y 的定义
- 定义到达某点：定义`p`到达某点`q`的意思是存在一条路径，沿着这条路径行进，`p`在到达到点`q`之前不会被`kill`掉。
- reaching definition：`a`是`b`的 reaching definition 的意思是存在一条从`a`到达`b`的路径，沿着这条路径走可以自然得到`a`要赋值的变量的值，而不需要额外的信息。 

按照上面的写法，`d1`便不再是`d3`的 reaching definition, 因为`d2`使它不再可能被到达。

对我们来说，这件事情是一目了然的，但是如果控制流再复杂一点，对于编译器来说，它便无法确切知道`d3`的 reaching definition 是`d1`或者`d2`了，也不清楚`d1`和`d2`到底是谁`kill`了谁。但是，如果我们的代码是 SSA 的，那它就会长成这样。

```
d1: y1 := 1
一些无关代码
d2: y2 := 2
一些无关代码
d3: x := y2
```

编译器很容易就能够发现`x`是由`y2`赋值得到，而`y2`被赋值了 2，且`x`和`y2`都只能被赋值一次，显然得到`x`的值的路径就是唯一确定的，`d2`就是`d3`的 reaching definition。而这样的信息，在编译器想要进行优化时会起到很大的作用。

### **SSA 带来的麻烦事**

假设你想用 IR 写一个用循环实现的 factorial 函数

```c
int factorial(int val) {
  int temp = 1;
  for (int i = 2; i <= val; ++i)
  temp *= i;
  return temp;
}
```

按照 C 语言的思路，我们可能大概想这样写

![preview](https://pic4.zhimg.com/v2-beebcdc30a8eb251c482f9856fb70de7_r.jpg)

然而我们会发现 %temp 和%i 被多次赋值了，这并不合法。

怎么办？

#### plan a —— `phi`

`phi`在基础实验中不要求生成，后面的挑战实验中的`mem2reg`的内容就是将 `Load`，`Store`形式的 SSA 转换为`phi` 形式的 SSA。

在`-O1`选项下生成这个函数的`.ll`格式的文件，我们会发现生成的代码大概长这样

```llvm
define dso_local i32 @factorial(i32 %0) local_unnamed_addr #0 {
  %2 = icmp slt i32 %0, 2
  br i1 %2, label %3, label %5

3:                                                ; preds = %5, %1
  %4 = phi i32 [ 1, %1 ], [ %8, %5 ] 			  ; 如果是从块%1 来的，那么值就是 1，如果是从
  ret i32 %4									  ; 块%5 来的，那么值就是%8 的值

5:                                                ; preds = %1, %5
  %6 = phi i32 [ %9, %5 ], [ 2, %1 ]
  %7 = phi i32 [ %8, %5 ], [ 1, %1 ]
  %8 = mul nsw i32 %6, %7
  %9 = add nuw i32 %6, 1
  %10 = icmp eq i32 %6, %0
  br i1 %10, label %3, label %5
}
```

`phi`指令的语法是

`<result> = phi <ty> [<val0>, <label0>], [<val1>, <label1>] …`

这个指令能够根据进入当前基本块之前执行的是哪一个基本块的代码来选择一个变量的值，有了`phi`以后我们的代码就变成了

![img](https://pic1.zhimg.com/80/v2-40c93aafeca39f560d0d555d8a264f54_1440w.jpg)

这样的话，每个变量就只被赋值一次，并且实现了循环递增的效果。

#### plan b——`alloca`，`load`与`store`

前面铺垫了那么长时间，就是为了介绍`alloca`，`load`与`store`三条指令以及他们的作用的。

在`-O0`选项下生成这个函数的`.ll`格式的文件，我们会发现生成的代码大概长这样

```llvm
define dso_local i32 @factorial(i32 %0) #0 {
  %2 = alloca i32, align 4
  %3 = alloca i32, align 4
  %4 = alloca i32, align 4
  store i32 %0, i32* %2, align 4
  store i32 1, i32* %3, align 4
  store i32 2, i32* %4, align 4
  br label %5

5:                                       
  %6 = load i32, i32* %4, align 4
  %7 = load i32, i32* %2, align 4
  %8 = icmp sle i32 %6, %7
  br i1 %8, label %9, label %16

9:                                             
  %10 = load i32, i32* %4, align 4
  %11 = load i32, i32* %3, align 4
  %12 = mul nsw i32 %11, %10
  store i32 %12, i32* %3, align 4
  br label %13

13:                                           
  %14 = load i32, i32* %4, align 4
  %15 = add nsw i32 %14, 1
  store i32 %15, i32* %4, align 4
  br label %5

16:                                               
  %17 = load i32, i32* %3, align 4
  ret i32 %17
}
```

`alloca`指令的作用是在当前执行的函数的栈帧上分配内存并返回一个指向这片内存的指针，当函数返回时内存会被自动释放（一般是改变栈指针）。

不难看出，此 .ll 文件所展示的 IR 并非完全的 SSA 形式，实际上，这里是使用了一个 LLVM 特有的 trick, 将所有的局部变量通过`alloca`指令进行了分配。这样做的原因是：构造 SSA 的算法比较复杂，而且需要各种复杂的数据结构，这些因素导致程序员想要在前端直接生成 SSA 形式的 IR 时非常麻烦。而这个 trick 避免了这些麻烦，降低了从源语言翻译到 LLVM IR 的难度。

基于上述的 trick, 前端能够直接将变量按照栈的方式分配到内存当中，并且这个内存里的变量不需要遵循 SSA 形式，可以被多次定义，从而避免了构造 phi 函数产生的大量开销。

在 LLVM 中，所有的内存访问都需要显式地调用 load/store 指令。要说明的是，LLVM 中并没有给出“取地址”的操作符。以上面生成的代码中的`%3`为例，我们能在这些地方发现它

```llvm
entry:
	%3 = alloca i32, align 4
...

9:
	...
	%11 = load i32, i32* %3, align 4
	...
	store i32 %12,i32* %3 ,align 4
```

变量`%3`通过 `alloca`声明，分配了`i32`大小的空间，这里`%3`的类型为`i32*`, 也就是说，`%3`代表的是这段空间的地址，`load`将`%3`所指向空间的内容读取出来，而`store`将新的值写入`%3`指向的空间。`alloca`分配的栈变量可以进行多次存取，因此，通过`alloca` ,`load`和`store`，我们避免了`phi`指令的使用。

**划重点**，这种避免产生`phi`指令的方法分为四个步骤：

1. 每个可变变量都变为了栈上分配的空间（每个变量都变成了一条`alloca`指令）
2. 每次对变量的读都变成了从内存空间中的一次读（每次读取一个变量都变成了通过`load`对变量所在内存的读）
3. 每次对变量的写都变成了对内存的一次写（每次更新一个变量的值都变成了通过`store`对变量所在内存的写）
4. 获取变量的地址等价于获取内存的地址

不难发现，这种方法虽然避免了`phi`的出现，但是每次变量的存取都变成了访问内存，这会导致严重的性能问题。所幸，正如我们之前所说的，LLVM 提供了一个叫做`mem2reg`的解决方案，它能够把`alloca`指令分配的栈变量转化为 SSA 寄存器，并且在合适的地方插入`phi`。

LLVM 官方非常支持使用上述`alloca + mem2reg`技术，Clang 默认不开优化生成的就是基于栈分配形式的 IR。alloca 技术可以把前端从繁琐的 SSA 构造工作中解脱出来，而 mem2reg 则可以极其快速地生成 SSA 形式。这两者的结合大大提高了编译的效率。

在基础实验中，你只需要实现较简单的 alloca 形式的 IR，而生成 phi 形式的完全 SSA IR 将被我们作为挑战实验发布。
