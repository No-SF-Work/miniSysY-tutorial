# flex

flex（the Fast Lexical Analyzer Generator）是 Lawrence Berkeley 实验室研发的一款**词法分析程序的生成工具**，它的前身是我们在课程中学到的 Lex。

## 安装 flex

flex 的最新版本是 2.6.4，发布于 2017 年 5 月 6 日。

### Ubuntu

- [ ] TODO: Ubuntu 16.04 和 18.04

```shell
$ sudo apt install flex
```

### MacOS

- [ ] TODO

### Windows

自行探索

## flex 的使用

flex 源文件的扩展名为 `.l`，分为「声明」、「规则」、「用户子程序」三部分。下面我们以生成一个用于统计文本中单词和字母个数的词法分析器的例子来介绍 flex 的使用。

### 生成 C 代码

flex 会读取一个 `*.l` 或者 `*.lex` 文件里的词法规则，并将生成的词法分析器代码写入 `lex.yy.c`。

「声明」部分的代码用`%{` 和 `%}` 来包裹，这些代码会被 flex 原样复制到 `lex.yy.c` 中，你可以在这里书写声明和定义。如下是一个声明部分的示例：

```c
/* word_char_counter.l */
/* 在这里我们要统计字符数和单词数，因此需要声明这两个统计变量 */
%{
#include <string.h>
int chars = 0;
int words = 0;
%}
```

「规则」部分的代码用 `%%` 和 `%%` 来包裹，你可以使用正则表达式来编写模式，在正则表达式后面编写一段 C 代码，指明匹配到相应的模式后所要完成的动作。如下是一个规则部分的示例：

```c
/* word_char_counter.l */
/* 遇到匹配的模式则累加对应的统计变量 */
%%
[a-zA-Z]+ { chars += strlen(yytext); words++; }
. { }
%%
```

在规则部分中，标识符 `yytext` 是一个指针，指向匹配到的输入字符串。

如果要将 flex 与 bison（后面会介绍的用来生成语法解析器的工具）或其他你自行编写的程序配合使用，那么你可以不写「用户子程序」部分；否则，你就需要通过编写「用户子程序」来使用词法解析得到的信息。

```c
/* word_char_counter.l */
int main(int argc, char **argv) {
    yylex();
    printf("I found %d words of %d chars.\n", words, chars);
    return 0;
}
```

其中 `yylex()` 是 flex 生成的函数，它会对输入进行词法分析并完成制定的动作（默认读取 stdin）。

写完词法规则后，可以使用 flex 生成对应的 C 代码文件：

```shell
$ flex word_char_counter.l
$ gcc lex.yy.c -o word_char_counter
```

但是如果你像这样直接进行编译，会出现下面这样的报错信息：

```shell
/usr/bin/ld: /tmp/cc1qil64.o: in function `yylex':
lex.yy.c:(.text+0x4b8): undefined reference to `yywrap'
/usr/bin/ld: /tmp/cc1qil64.o: in function `input':
lex.yy.c:(.text+0x10c7): undefined reference to `yywrap'
collect2: error: ld returned 1 exit status
```

观察报错信息会发现它说找不到 `yywrap()` 这个函数的位置。这是因为在 flex 2.5.4 版本之后，当程序扫描到 `EOF` 时会调用 `yywrap()` 函数来判断是否还有其他的输入，如果 `yywrap()` 返回 0，则程序用来读取输入的指针会在 `yywrap()` 被重定向到另一个输入并且继续读取；反之，如果 `yywrap()` 返回 1，则说明没有其他输入。

在这里，你可以链接 `fl` 库，调用其中默认会返回 1 的 `yywrap()` 函数（只需要读取一个输入文件或 stdin），即使用 `gcc lex.yy.c -o word_char_counter -lfl` 。或者也可以在源文件的开头加上一行 `%option noyywrap`，表示不调用 `yywrap()`，从而解决链接错误。

```shell
$ ./word_car_counter
Hello, flex.
^D
I found 2 words of 9 chars.
```

### 生成 C++ 代码

- [ ] TODO

### 生成 Java 代码

- [ ] TODO
