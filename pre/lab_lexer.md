# 词法分析小实验

完成本次实验后，你需要提交本次实验的 pdf 格式的[实验报告](../report.md)，上传到 `pre/词法分析实验/` 中的对应班级目录中，命名规则为 `学号_姓名_labLexer.pdf`。

提交实验报告的截止时间为 2021 年 X 月 X 日 23:59。

## 实验内容

你需要手工编写一个词法分析程序，从输入文件中读入字符串，根据 Token 对照表将识别到的对应 token 输出到标准输出（stdout），每行输出一个 token。

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
- 空白字符有分隔 token 的作用，在现代编译器中，通常不会直接暴力地去除输入中的空白字符。gcc 和 clang 遇到 `a = 1   0;` 时，都会报语法错误，而不是按 `a=10;` 处理，我们的词法分析程序应当分别识别出 `Number(1)` 和 `Number(0)`；
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

TODO
