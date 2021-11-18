# Lab 6 实验指导

> 更为正式的语言你可以在龙书等书本上找到，比如龙书的第六章第七节，这个指导的目的是让你理解回填存在的意义和作用。

基本思想：在生成一些跳转指令时，暂时不指定这个指令跳转的目标位置，而是将其放入一个容器中暂时存储，在能确定正确的目标标号以后，再去填充这个容器内的指令的目标标号。

为什么需要这么做呢，举个简单的例子

```cpp
while (cond_a) {
    if (cond_b) {
        break;
    }
    do_some_thing();
}
some_other_things();
```

以上述代码为例，在一趟式的翻译中（边分析边推导边生成代码或者先生成 AST 再遍历 AST 来生成代码），我们必须在处理 `some_other_things();` 之前完成对 `while` 语句中的内容的处理，而我们在分析到 `break` 的时候，并不知道我们生成的 `br` 指令应该跳转到哪个基本块（通常这个块还没有生成）。这时我们就需要一个记录来记录下“此处应有一个跳出去的 `br`”，然后把这个记录存起来，继续向下分析。

当分析结束，我们到达了` some_other_things();` 后，我们就知道了那些未确定的 `br` 指令应该跳转的地方，然后我们按图索骥，按照记录去找到这些需要被更新的指令，至此，我们就完成了一次“回填”。

实现的思路：  
你的编译器中肯定是有类似于 `analyseWhileStmt()` 或者 `visitWhileStmt()` 的函数的，你可以在进入这个函数时生成一个容器，用于记录接下来可能出现的 `break` 和 `continue`，在其出现以后生成一个临时的占位符，并将这个占位符的位置存储到容器中，在退出这个函数前遍历这个容器，将那些位置替换为正确的跳转。

伪代码如下：

```java
Stack<Recorder> stk = new Stack();

visitWhileStmt() {
    stk.push(new Recorder());
    // do some thing
    stk.top.foreach(mark -> {
        update(mark);
    });
    stk.pop();
}

visitBreakStmt() {
    stk.top.record(new Mark("break"));
}

visitContinueStmt() {
    stk.top.record(new Mark("continue"));
}
```

> 使用 `Stack` 是因为循环会出现嵌套。
