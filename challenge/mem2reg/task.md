# 挑战实验：mem2reg

LLVM IR 借助 “memory 不是 SSA value” 的特点开了个后门。编译器前端在生成 LLVM IR 时，可以选择不生成真正的 SSA 形式，而是把局部变量生成为 `alloca/load/store` 形式。在本次实验中，你需要在满足前面基础实验全部要求的前提下实现 mem2reg，将局部 `int` 类型变量 `alloca/load/store` 形式的 LLVM IR 转换成真正的 SSA 形式的 LLVM IR 并输出。进行 mem2reg 后，你输出的 LLVM IR 中不能再有任何局部 `int` 类型变量相关的 `alloca/load/store` 指令，同时根据程序的语义应当会出现一些 `phi` 指令。

> 注意是局部 `int` 类型变量，不包括全局变量、数组等。

具体指导参考 [mem2reg 实验指导](help.md)。

## 评测

本次实验的评测标识符为 `mem2reg`，即你的 `judge.toml` 第一行应改为 `[jobs.mem2reg]`。

你需要从 `$input` 读取输入文件，将编译生成的文本格式的 LLVM IR 输出到 `$ir` 中，评测机会使用 `lli` 解释执行该文件，并评测生成的 IR 代码是否正确。请在 `judge.toml` 的 `run` 中使用 `$input` 代替输入文件路径，`$ir` 代替输出文件路径。如：`./compiler < $input > $ir` 或 `./compiler $input $ir` 等。

```toml
# 一个示例
[jobs.mem2reg]

image = { source = "dockerfile", path = "." }

run = [
    "./compiler $input $ir",
]
```

## 注意事项

你需要**认真编写**挑战实验的实验报告，详细说明你是如何完成本次挑战实验的，你对你的编译器进行了哪些改动，你参考了哪些资料，并尽可能完整地阐述你的编译器完成挑战实验任务的工作流程。如果实验报告的内容含糊不清，无法证明你独立完成本次实验，违反[诚信](../../integrity.md)原则，我们会酌情扣分。当然，你也需要适当地精简语言，我们对实验报告的评定点在于**扣分**，写出特别长的实验报告（如纯文字内容超过 7 页）并不意味着你能得到加分。

- 实验评测截止时间：2022 年 1 月 7 日 23:59
- 实验报告命名格式：`学号_姓名_mem2reg.pdf`
- 实验报告提交：[北航云盘](https://bhpan.buaa.edu.cn:443/link/413EA0802B7A7627A6B5112531C40772) `挑战实验/` 对应班级目录中
- 实验报告提交截止时间：2022 年 1 月 9 日 23:59