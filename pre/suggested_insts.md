# 推荐使用的指令

本节介绍了一些我们认为有用的 LLVM IR 的指令

这些指令**不是必须的**，你可以自己在 [LLVM Lang Ref](https://llvm.org/docs/LangRef.html)里选择指令并且生成代码

只要通过了测试点，我们就认为你所编写的编译器是正确的


### ops

| 名称     | llvm ir  | usage                                                        | intro |
| -------- | -------- | ------------------------------------------------------------ | ----- |
| 加       | add      | ` <result> = add <ty> <op1>, <op2> ; yields ty:result`       |       |
| 减       | sub      | `<result> = sub `<ty`> `<op1`>, `<op2`> ; yields ty:result`  |       |
| 乘       | mul      | `<result> = mul <ty> <op1>, <op2> `                          |       |
| 除       | sdiv     | `<result> = sdiv <ty> <op1>, <op2>  `                        |       |
| 少于     | icmp slt | `<result> = icmp <cond> <ty> <op1>, <op2>   ; yields i1 or <N x i1>:result` |       |
| 少于等于 | icmp sle | 同上                                                         |       |
| 大于等于 | icmp sge | 同上                                                         |       |
| 大于     | icmp sgt | 同上                                                         |       |
| 等于     | icmp sq  | 同上                                                         |       |
| 不等     | icmp ne  | 同上                                                         |       |
| 与       | and      | `<result> = and <ty> <op1>, <op2>   ; yields ty:result`      |       |
| 或       | or       | `<result> = or <ty> <op1>, <op2>   ; yields ty:result`       |       |
| 调用函数 | call     | `<result> = [tail |musttail |notail ] call [fast-math flags] [cconv] [ret attrs] [addrspace(<num>)]            <ty>|<fnty> <fnptrval>(<function args>) [fn attrs] [ operand bundles ]` |       |

### terminator insts

| TAG  | llvm ir | usage                                                        | intro                                                        |
| ---- | ------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 跳转 | br      | `br i1 <cond>, label <iftrue>, label <iffalse>`       `br label <dest>  ` | cause control flow to transfer to a different basic block **|
| 返回 | ret     | `ret <type> <value> `  ,`ret void  `                | return control flow(optionally a value)                      |



| TAG    | llvm ir       | usage                                                        | intro                                                       |
| ------ | ------------- | ------------------------------------------------------------ | ----------------------------------------------------------- |
| Alloca | alloca        | `  <result> = alloca [inalloca] <type> [, <ty> <NumElements>] [, align <alignment>] [, addrspace(<num>)] ; yields type addrspace(num)*:result` | allocate  memory in current stack frame                     |
| Load   | load          | `<result> = load [volatile] <ty>, <ty>* <pointer>[, align <alignment>][, !nontemporal !][, !invariant.load !<empty_node>][, !invariant.group !][, !nonnull !<empty_node>][, !dereferenceable !][, !dereferenceable_or_null !<deref_bytes_node>][, !align !][, !noundef !<empty_node>]` | read memory                                                 |
| Store  | store         | `store [volatile] <ty> <value>, <ty>* <pointer>[, align <alignment>][, !nontemporal !<nontemp_node>][, !invariant.group !<empty_node>] ; yields void` | write memory                                                |
| GEP    | getelementptr | `<result> = getelementptr <ty>, * {, [inrange] <ty> <idx>}*`                                                                                                 `<result> = getelementptr inbounds <ty>, <ty>* <ptrval>{, [inrange] <ty> <idx>}*`                                                                                 `<result> = getelementptr <ty>, <ptr vector> <ptrval>, [inrange] <vector index type> <idx>` | this inst only calculate  memory,do not read or load memory |
| Phi    | phi           | `<result> = phi [fast-math-flags] <ty> [ <val0>, <label0>], ...` |                                                             |
| zext   | zext..to      | <result> = zext <ty> <value> to <ty2>             ; yields ty2 | zext                                                        |
