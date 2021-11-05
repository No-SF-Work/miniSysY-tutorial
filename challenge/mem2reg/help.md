# mem2reg 实验指导

## 简介

在 [LLVM 中的 SSA](../pre/llvm_ir_ssa.md) 中，我们简单地介绍了 SSA 的相关知识。LLVM IR 借助只要求虚拟寄存器是 SSA 形式，而内存不要求是 SSA 形式的特点开了个后门。编译器前端在生成 LLVM IR 时，可以选择不生成真正的 SSA 形式，而是把局部变量生成为 `alloca/load/store` 形式。

LLVM 以 pass（遍）的形式管理对 LLVM IR 的转换、分析或优化等行为，mem2reg 就是其中的一个 pass。在 mem2reg 中，LLVM 会识别出局部变量的 `alloca` 指令，将对应的局部变量改为虚拟寄存器中的 SSA 形式的变量，将该变量的 `store/load` 修改为虚拟寄存器之间的 def-use/use-def 关系，并在适当的地方加入 `phi` 指令和进行变量的重命名。

比如对于以下的代码：

```cpp
int main() {
    int x, cond = 1;
    if (cond > 0)
        x = 1;
    else
        x = -1;
    return x;
}
```

对应的 `alloca/load/store` 形式的 IR 如下：

```llvm
define dso_local i32 @main() {
    %1 = alloca i32
    %2 = alloca i32
    store i32 1, i32* %1
    %3 = load i32, i32* %1
    %4 = icmp sgt i32 %3, 0
    br i1 %4, label %5, label %8

5:
    store i32 1, i32* %2
    br label %6

6:
    %7 = load i32, i32* %2
    ret i32 %7

8:
    %9 = sub i32 0, 1
    store i32 %9, i32* %2
    br label %6
}

```

进行 mem2reg 后，对应的 IR 如下：

```llvm
define dso_local i32 @main() {
    %1 = icmp sgt i32 1, 0
    br i1 %1, label %2, label %5

2:
    br label %3

3:
    %4 = phi i32 [ 1, %2 ], [ %6, %5 ]
    ret i32 %4

5:
    %6 = sub i32 0, 1
    br label %3
}
```

对比发现，`alloca/load/store` 形式的 IR 中的 `%1, %2` 分别对应 `x` 和 `cond`，所有对它们的 `load` 指令的值（如 `%3, %7`）的使用被替换成了对应的 `store` 所写入的值。而在 mem2reg 后的 IR 的 `3` 基本块中，加入了一条 `phi` 指令，该指令根据在当前基本块执行之前执行的是哪一个基本块取不同的值。如果控制流是从 `2` 跳转到了 `3`，`%4` 取 `1`；如果控制流是从 `5` 跳转到了 `3`，`%4` 取 `%6`。需要注意一点：一个基本块中的所有 `phi` 指令都必须在该基本块的开头，且 `phi` 指令是并行取值的，请看下面的例子：

```llvm
define dso_local i32 @main() {
    %1 = icmp slt i32 0, 5
    br i1 %1, label %6, label %2

2:
    %3 = phi i32 [ 0, %0 ], [ %10, %6 ]
    %4 = phi i32 [ 0, %0 ], [ %7, %6 ]
    %5 = phi i32 [ 0, %0 ], [ %11, %6 ]
    ret i32 %5

6:
    %7 = phi i32 [ 0, %0 ], [ %10, %6 ]
    %8 = phi i32 [ 0, %0 ], [ %7, %6 ]
    %9 = phi i32 [ 0, %0 ], [ %11, %6 ]
    %10 = add i32 %7, 1
    %11 = add i32 %9, 1
    %12 = icmp slt i32 %7, 5
    br i1 %12, label %6, label %2
}
```

这段 IR 解释执行后返回值为 `6`，而将 `%10 = add i32 %7, 1` 改为 `%10 = add i32 %8, 1` 后，返回值却变成了 `10`，造成这样区别的原因就在于 `phi` 指令的并行取值：一个基本块里的 `phi` 指令是同时取值的，每次从 `6` 跳转到 `6` 块时，`%8` 取的 `%7` 的值是跳转之前的 `%7` 的值，而非 `%7` 所取的 `%10` 的值。

## 前置概念

> 以下概念仅供 mem2reg 挑战实验使用

- **控制流图**（Control Flow Graph, CFG）：一个程序中所有基本块执行的可能流向图，图中的每个节点代表一个基本块，有向边代表基本块间的跳转关系. CFG 有一个入口基本块和一/多个出口基本块，分别对应程序的开始和终止.
- **支配**（dominate）：对于 CFG 中的节点 $n_1$ 和 $n_2$，$n_1$ 支配 $n_2$ 当且仅当所有从入口节点到 $n_2$ 的路径中都包含 $n_1$，即 $n_1$ 是从入口节点到 $n_2$ 的必经节点. 每个基本块都支配自身.
- **严格支配**（strictly dominate）：$n_1$ 严格支配 $n_2$ 当且仅当 $n_1$ 支配 $n_2$ 且 $n1\neq n_2$.
- **直接支配者**（immediate dominator, idom）：节点 $n$ 的直接支配者是离 $n$ 最近的严格支配 $n$ 的节点（标准定义是：严格支配 $n$，且不严格支配任何严格支配 $n$ 的节点的节点）. 入口节点以外的节点都有直接支配者. 节点之间的直接支配关系可以形成一棵支配树（dominator tree）.

下图展示了一个 CFG 和对应的支配树：

![CFG 和支配树](../pic/cfg_and_dom_tree.png)

- **支配边界**（dominance frontier）：节点 $n$ 的支配边界是 CFG 中刚好**不**被 $n$ 支配到的节点集合. 形式化一点的定义是：节点 $n$ 的支配边界 $DF(n) = \{x | n 支配 x 的前驱节点，n 不严格支配 x\}$.

在一个基本块 `x` 中对变量 `a` 进行赋值，在不考虑路径中对变量进行重新赋值（kill）的情况下，所有被 `x` 支配的基本块中，`a` 的值一定是 `x` 中所赋的值。而对于 `x` 的支配边界中的基本块，情况则有所不同——它们的控制流不一定来自于 `x`，`a` 的值只是有可能是 `x` 中所赋的值。在支配边界所支配的基本块中，当然也无法确定 `a` 的值，支配边界是恰好不能确定 `a` 是否取 `x` 中所赋的值的分界线。例如上面的图中，`4` 支配 `7` 的前驱节点 `5`，但是 `4` 不支配 `7`，`4` 的支配边界是 `{7}`，如果有个变量 `a` 在 `2` 中被定义，在 `3` 和 `4` 中分别被赋值为 0 和 1，我们不能确定在程序执行到 `7` 时 `a` 的值是 0 还是 1。

计算 CFG 中每个节点的支配边界的算法如下：

```c
for CFG 中的边 a -> b {
    x = a
    while x 不严格支配 b {
        DF(x) = DF(x) ∪ b
        x = idom(x) // x 的直接支配者
    }
}
```

为了简单起见，我们定义一个节点集的支配边界是集合中所有节点的支配边界的并集。

- **迭代支配边界**（iterated dominance frontier）：节点集 $S$ 的迭代支配边界 $DF^{+} (S)$ 是通过迭代地计算支配边界，直到到达一个不动点得到的. 迭代方式为：$DF^{+} (S) = DF_{i\to \infty}(S),~ ~ DF_1(S)=DF(S), ~ ~ DF_{i+1}(S)=DF(S\cup DF_i(S))$.

- [ ] TODO: join set and explain that DF+(S) = J(S\cup {entry})

## SSA 构造算法

下面的 SSA 构造算法假设了变量在被使用时一定已经被定义/赋值了，我们的测试用例中也会保证这一点，学有余力想要了解更多 SSA 相关内容请参考 [SSA Book(http://ssabook.gforge.inria.fr/latest/book.pdf)](http://ssabook.gforge.inria.fr/latest/book.pdf)。

### 插入 phi 函数

### 变量重命名

## SSA 构造算法的应用——LLVM IR 上的 mem2reg

## 参考文献

https://llvm-clang-study-notes.readthedocs.io/en/latest/ssa/index.html

Barry Rosen; Mark N. Wegman; F. Kenneth Zadeck (1988). "Global value numbers and redundant computations" (PDF). Proceedings of the 15th ACM SIGPLAN-SIGACT Symposium on Principles of Programming Languages.

https://en.wikipedia.org/wiki/Static_single_assignment_form

对于 LLVM 之类的编译器是如何实现在构造 SSA 形式的 IR 的时候，计算出 def-use 链？ - RednaxelaFX 的回答 - 知乎
https://www.zhihu.com/question/41999500/answer/93243408

Static Single Assignment Book
