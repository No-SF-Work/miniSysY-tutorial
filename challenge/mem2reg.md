# mem2reg

## SSA

静态单赋值（Static Single Assignment, **SSA**）是编译器中间表示中非常重要的一个概念，它是一种变量的命名约定。当程序中的每个变量都有且只有一个赋值语句时，称一个程序是 SSA 形式的。SSA 有一个最基本的属性：引用透明性（referential transparency），程序中的每个变量只有一个定义，变量的值和它在程序的位置无关，只要一个变量被定义了，那么它的值就是固定的。

对于下面一段代码：

```c
x = 1;
y = x + 1;
x = 2;
z = x + 1;
```

`y` 和 `z` 的值都是 `x + 1`，如果对 `y` 和 `z` 的赋值语句之间还有很多其他的语句，你可能很难注意到 `x` 的值发生了变化，`y` 和 `z` 实际的值并不相等。将其转换成 SSA 后，则只有 `x1` 和 `x2` 相等时 `y` 和 `z` 才相等。

```c
x1 = 1;
y  = x1 + 1;
x2 = 2;
z  = x2 + 1;
```

## 背景知识

## mem2reg

## 参考资料

Barry Rosen; Mark N. Wegman; F. Kenneth Zadeck (1988). "Global value numbers and redundant computations" (PDF). Proceedings of the 15th ACM SIGPLAN-SIGACT Symposium on Principles of Programming Languages.

https://en.wikipedia.org/wiki/Static_single_assignment_form

对于LLVM之类的编译器是如何实现在构造 SSA 形式的 IR 的时候，计算出 def-use 链？ - RednaxelaFX的回答 - 知乎
https://www.zhihu.com/question/41999500/answer/93243408

https://llvm-clang-study-notes.readthedocs.io/en/latest/ssa/index.html

Static Single Assignment Book