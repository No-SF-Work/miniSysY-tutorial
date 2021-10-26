# 评测说明 

本次实验的评测标识符为 `lab3`，即你的 `judge.toml` 第一行应改为 `[jobs.lab3]`。

你需要从 `$input` 读取输入文件，将编译生成的文本格式的 LLVM IR 输出到 `$ir` 中，评测机会使用 `lli` 解释执行该文件，并评测生成的 IR 代码是否正确。请在 `judge.toml` 的 `run` 中使用 `$input` 代替输入文件路径，`$ir` 代替输出文件路径。如：`./compiler < $input > $ir` 或 `./compiler $input $ir` 等。

```toml
# 一个示例
[jobs.lab3]

image = { source = "dockerfile", path = "." }

run = [
    "./compiler $input $ir",
]
```

如果编译过程中出现了错误（语法、语义、编译过程错误等），你的**编译器**应当以非 0 的返回值退出（或抛出异常）。否则如果一切正常，你的编译器应当正常退出。