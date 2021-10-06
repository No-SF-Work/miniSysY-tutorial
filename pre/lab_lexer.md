# 词法分析小实验

完成本次实验后，你需要提交本次实验的 pdf 格式的[实验报告](../report.md)，上传到 `pre/词法分析实验/` 中的对应班级目录中，命名规则为 `学号_姓名_labLexer.pdf`。

评测和提交实验报告的截止时间为 2021 年 10 月 10 日 23:59。

## 实验内容

在本次实验中，你需要**手工编写**一个词法分析程序（不允许使用 flex/ANTLR 等自动生成），从输入文件中读入字符串，根据 Token 对照表将识别到的对应 token 输出到标准输出（stdout），每行输出一个 token。

Token 对照表如下：

| Token 名称 | 对应字符串                      | 输出格式          | 备注                                  |
| ---------- | ------------------------------- | ----------------- | ------------------------------------- |
| 标识符     | （定义见下）                    | `Ident($name)`    | 将 `$name` 替换成标识符对应的字符串   |
| 无符号整数 | （定义见下）                    | `Number($number)` | 将 `$number` 替换成标识符对应的字符串 |
| if         | `if`                            | `If`              |                                       |
| else       | `else`                          | `Else`            |                                       |
| while      | `while`                         | `While`           |                                       |
| break      | `break`                         | `Break`           |                                       |
| continue   | `continue`                      | `Continue`        |                                       |
| return     | `return`                        | `Return`          |                                       |
| 赋值符号   | `=`                             | `Assign`          |                                       |
| 分号       | `;`                             | `Semicolon`       |                                       |
| 左括号     | `(`                             | `LPar`            |                                       |
| 右括号     | `)`                             | `RPar`            |                                       |
| 左大括号   | `{`                             | `LBrace`          |                                       |
| 右大括号   | `}`                             | `RBrace`          |                                       |
| 加号       | `+`                             | `Plus`            |                                       |
| 乘号       | `*`                             | `Mult`            |                                       |
| 除号       | `/`                             | `Div`             |                                       |
| 小于号     | `<`                             | `Lt`              |                                       |
| 大于号     | `>`                             | `Gt`              |                                       |
| 等于号     | `==`                            | `Eq`              |                                       |
| 错误       | 不能符合上述 token 规则的字符串 | `Err`             | 程序应输出 `Err` 后终止               |

标识符和无符号整数的文法定义如下：

```
Letter -> 'a' | 'b' | 'c' | 'd' | 'e' | 'f' | 'g' | 'h' | 'i' | 'j' | 'k' | 'l' | 'm' | 'n' | 'o' | 'p' | 'q' | 'r' | 's'
    | 't' | 'u' | 'v' | 'w' | 'x' | 'y' | 'z' | 'A' | 'B' | 'C' | 'D' | 'E' | 'F' | 'G' | 'H' | 'I' | 'J' | 'K' | 'L'
    | 'M' | 'N' | 'O' | 'P' | 'Q' | 'R' | 'S' | 'T' | 'U' | 'V' | 'W' | 'X' | 'Y' | 'Z'

Digit -> '0' | '1' | '2' | '3' | '4' | '5' | '6' | '7' | '8' | '9'

Underline -> '_'

Nondigit -> Letter | Underline

<标识符> -> Nondigit | <标识符> Nondigit | <标识符> Digit

<无符号整数> -> Digit | <无符号整数> Digit
```

注意事项：

- 程序的关键字应区分大小写，程序中所有完全匹配上关键字的字符串不应被识别为标识符；
- 保证无符号整数的范围为 `0 <= number <= 2147483647`，不会出现范围之外的数字；
- 空格、制表符、换行符有分割 token 的作用。在现代编译器中，通常不会直接暴力地去除输入中的空白字符。gcc 和 clang 遇到 `a = 1   0;` 时，都会报语法错误，而不是按 `a=10;` 处理，说明词法分析时保留了空白符，到语法分析时发现存在错误。对于我们的词法分析程序，不需要将空白符作为一个 token，但需要其发挥分割的作用，遇到 `a = 1   0;` 时应当分别识别出 `Number(1)` 和 `Number(0)`；
- 在遇到文法中存在二义性的情况时（如 `===` 可以被识别成 `Assign\nEq`、`Eq\nAssign` 或 `Assign\nAssign\nAssign`），默认遵循最长匹配原则，即要尽可能多地识别一个 token 可以接受的字符。对于 `===`，应识别成 `Eq\nAssign`。

## 示例

输入样例 1：

```c
a = 10;
c = a * 2 + 3;
return c;
```

输出样例 1：

```
Ident(a)
Assign
Number(10)
Semicolon
Ident(c)
Assign
Ident(a)
Mult
Number(2)
Plus
Number(3)
Semicolon
Return
Ident(c)
Semicolon
```

输入样例 2：

```c
a = 10;
:c = a * 2 + 3;
return c;
```

输出样例 2：

```
Ident(a)
Assign
Number(10)
Semicolon
Err
```

输入样例 3：

```c
a = 3;
If = 0
while (a < 4396) {
    if (a == 010) {
        ybb = 233;
        a = a + ybb;
        continue;
    } else {
        a = a + 7;
    }
    If = If + a * 2;
}
```

输出样例 3：

```
Ident(a)
Assign
Number(3)
Semicolon
Ident(If)
Assign
Number(0)
While
LPar
Ident(a)
Lt
Number(4396)
RPar
LBrace
If
LPar
Ident(a)
Eq
Number(010)
RPar
LBrace
Ident(ybb)
Assign
Number(233)
Semicolon
Ident(a)
Assign
Ident(a)
Plus
Ident(ybb)
Semicolon
Continue
Semicolon
RBrace
Else
LBrace
Ident(a)
Assign
Ident(a)
Plus
Number(7)
Semicolon
RBrace
Ident(If)
Assign
Ident(If)
Plus
Ident(a)
Mult
Number(2)
Semicolon
RBrace
```

## 评测

评测在 10 月 1 日开放，10 月 10 日 23:59 截止评测。你的词法分析器可以通过命令行参数指定文件路径或从标准输入读入输入文件，你需要输出结果到标准输出中。

`Dockerfile` 和 `judge.toml` 的编写格式见 [提交作业的格式与方法](https://github.com/BUAA-SE-Compiling/rurikawa/blob/master/docs/manual/submit.md)。

一个配置文件的示例如下：

```dockerfile
# -- Dockerfile --
# 这个文件负责构建包含你的程序的 Docker 容器

# 使用 Java 12
FROM openjdk:12-alpine
# 向容器内复制文件
COPY ./* /app/
# 编译程序
WORKDIR /app/
RUN javac -d ./output ./my/path/MyClass.java
# 将当前目录设为 /app/output
WORKDIR /app/output
```

```toml
# -- judge.toml --
# 这个文件负责告诉评测姬你需要怎么评测你的程序

# 我们的评测标识符是 lexer
[jobs.lexer]
# 使用 Dockerfile 来源，路径就是当前文件夹
image = { source = "dockerfile", path = "." }

# 假如你用的是 Java
run = [
  # 运行程序
  "java my.path.MyClass $input",
]
```

评测机提供的输入是一个文件路径 `$input`，你的词法分析器可以选择将文件作为命令行参数传入后读取文件，也可以选择使用 `cat $input | <你的程序>` 从标准输入读入。
