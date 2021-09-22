# 推荐使用的指令

本节介绍了一些我们认为有用的 LLVM IR 的指令以及其简化的用法（比如`load`指令的完整用法为``<result> = load [volatile] <ty>, <ty>* <pointer>[, align <alignment>][, !nontemporal !][, !invariant.load !<empty_node>][, !invariant.group !][, !nonnull !<empty_node>][, !dereferenceable !][, !dereferenceable_or_null !<deref_bytes_node>][, !align !][, !noundef !<empty_node>]``）

这些指令**不一定是必须的**，你可以自己在 [LLVM Lang Ref](https://llvm.org/docs/LangRef.html) 里选择指令并且生成代码

只要通过了测试点，我们就认为你所编写的编译器是正确的

### instructions 

| llvm ir | usage                                                        | intro |
| ------- | ------------------------------------------------------------ | ----- |
| add     | ` <result> = add <ty> <op1>, <op2>`                          |       |
| sub     | `<result> = sub <ty> <op1>, <op2>`                           |       |
| mul     | `<result> = mul <ty> <op1>, <op2> `                          |       |
| sdiv    | `<result> = sdiv <ty> <op1>, <op2>  `                        |       |
| icmp    | `<result> = icmp <cond> <ty> <op1>, <op2>   `                |       |
| and     | `<result> = and <ty> <op1>, <op2>  `                         |       |
| or      | `<result> = or <ty> <op1>, <op2>   `                         |       |
| call    | `<result> =  call  [ret attrs]  <ty> <fnptrval>(<function args>)` |       |
| alloca        | `  <result> = alloca [inalloca] <type> [, <ty> <NumElements>] [, align <alignment>] [, addrspace(<num>)] ; yields type addrspace(num)*:result` | allocate  memory in current stack frame                     |
| load          | `<result> = load [volatile] <ty>, <ty>* <pointer>[, align <alignment>][, !nontemporal !][, !invariant.load !<empty_node>][, !invariant.group !][, !nonnull !<empty_node>][, !dereferenceable !][, !dereferenceable_or_null !<deref_bytes_node>][, !align !][, !noundef !<empty_node>]` | read memory                                                 |
| store         | `store [volatile] <ty> <value>, <ty>* <pointer>[, align <alignment>][, !nontemporal !<nontemp_node>][, !invariant.group !<empty_node>] ; yields void` | write memory                                                |
| getelementptr | `<result> = getelementptr <ty>, * {, [inrange] <ty> <idx>}*`                                                                                                 `<result> = getelementptr inbounds <ty>, <ty>* <ptrval>{, [inrange] <ty> <idx>}*`                                                                                 `<result> = getelementptr <ty>, <ptr vector> <ptrval>, [inrange] <vector index type> <idx>` | this inst only calculate  memory,do not read or load memory |
| phi           | `<result> = phi [fast-math-flags] <ty> [ <val0>, <label0>], ...` |                                                             |
| zext..to      | <result> = zext <ty> <value> to <ty2>             ; yields ty2 | zext                                                        |

### terminator insts

| llvm ir | usage                                                        | intro                                                        |
| ------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| br      | `br i1 <cond>, label <iftrue>, label <iffalse>`       `br label <dest>  ` | cause control flow to transfer to a different basic block ** |
| ret     | `ret <type> <value> `  ,`ret void  `                         | return control flow(optionally a value)                      |
