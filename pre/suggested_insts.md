# 推荐使用的指令
本节默认你已经掌握了一定的LLVM相关的知识。

本节介绍了一些我们认为有用的 LLVM IR 的指令以及其简化的用法（比如`load`指令的完整用法为
``<result> = load [volatile] <ty>, <ty>* <pointer>[, align <alignment>][, !nontemporal !][, !invariant.load !<empty_node>][, !invariant.group !][, !nonnull !<empty_node>][, !dereferenceable !][, !dereferenceable_or_null !<deref_bytes_node>][, !align !][, !noundef !<empty_node>**`**）

当然，**你可以不局限我们所介绍的这些指令**。在实验中，我们只要求你编译到正确的 LLVM IR 即可，因此你可以自己在 [LLVM Lang Ref](https://llvm.org/docs/LangRef.html) 里选择指令自己需要的指令用在生成的代码中。

只要通过了测试点，我们就认为你所编写的编译器是正确的。

### instructions

| llvm ir | usage                                                        | intro |
| ------- | ------------------------------------------------------------ | ----- |
| add     | ` <result> = add <ty> <op1>, <op2>`                          | / |
| sub     | `<result> = sub <ty> <op1>, <op2>`                           | / |
| mul     | `<result> = mul <ty> <op1>, <op2> `                          | / |
| sdiv    | `<result> = sdiv <ty> <op1>, <op2>  `                        | 有符号除法 |
| icmp    | `<result> = icmp <cond> <ty> <op1>, <op2>   `                | 比较指令 |
| and     | `<result> = and <ty> <op1>, <op2>  `                         | 与 |
| or      | `<result> = or <ty> <op1>, <op2>   `                         | 或 |
| call    | `<result> =  call  [ret attrs]  <ty> <fnptrval>(<function args>)` | 函数调用 |
| alloca        | `  <result> = alloca <type> ` | 分配内存                 |
| load          | `<result> = load  <ty>, <ty>* <pointer>` | 读取内存                                               |
| store         | `store  <ty> <value>, <ty>* <pointer>` | 写内存                                              |
| getelementptr | `<result> = getelementptr <ty>, * {, [inrange] <ty> <idx>}*`                                                                                                 `<result> = getelementptr inbounds <ty>, <ty>* <ptrval>{, [inrange] <ty> <idx>}*` | 计算目标元素的位置（仅计算） |
| phi           | `<result> = phi [fast-math-flags] <ty> [ <val0>, <label0>], ...` |  |
| zext..to      | `<result> = zext <ty> <value> to <ty2>  ` | 类型转换，将 `ty`的`value`的type转换为`ty2`                         |

### terminator insts

| llvm ir | usage                                                        | intro                          |
| ------- | ------------------------------------------------------------ | ------------------------------ |
| br      | `br i1 <cond>, label <iftrue>, label <iffalse>`       `br label <dest>  ` | 改变控制流                     |
| ret     | `ret <type> <value> `  ,`ret void  `                         | 退出当前函数，并返回值（可选） |
