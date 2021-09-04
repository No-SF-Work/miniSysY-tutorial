# flex/bison/ANTLR 简介

在后续完成一个完整的编译器的实验中，我们允许你使用解析器生成工具来自动生成编译器的前端词法分析部分和语法分析部分。

如果你打算手动实现词法分析和语法分析，可以跳过此节。

这部分会涉及到后面的知识，看不懂其中的一些地方是正常的。

本节的语法解析部分将会使用生成器生成用于解析算术表达式的代码，其示例文法如下：

```c
expr -> term | expr '+' term | expr '-' term
term -> factor | term '*' factor | term '/' factor
factor -> '(' expr ')' | number
number -> [0-9]+ | [0-9]+ '.' [0-9]* | [0-9]* '.' [0-9]+
```

## flex

flex（the Fast Lexical Analyzer Generator）是 Lawrence Berkeley 实验室研发的一款**词法分析程序的生成工具**，它的前身是我们在课程中学到的 Lex。

### 安装 flex

flex 的最新版本是 2.6.4，发布于 2017 年 5 月 6 日。

#### Ubuntu

- [ ] TODO: Ubuntu 16.04 和 18.04

```bash
$ sudo apt install flex
```

#### MacOS

- [ ] TODO

#### Windows

自行探索

### flex 的使用

flex 源文件的扩展名为 `.l`，分为「声明」、「规则」、「用户子程序」三部分。下面我们以生成一个用于统计文本中单词和字母个数的词法分析器的例子来介绍 flex 的使用。

#### 生成 C 代码

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

在这里，我们可以链接 `fl` 库，调用其中默认会返回 1 的 `yywrap()` 函数（因为我们只会读取一个输入），即使用 `gcc lex.yy.c -o word_char_counter -lfl` 。或者也可以在源文件的开头加上一行 `%option noyywrap`，表示不调用 `yywrap()`，从而解决链接错误。

```bash
$ ./word_car_counter
Hello, flex.
^D
I found 2 words of 9 chars.
```

#### 生成 C++ 代码

- [ ] TODO

#### 生成 Java 代码

- [ ] TODO

## bison

bison 是一款**语法分析程序的生成工具**，其前身为 yacc。yacc 由贝尔实验室的 S.C.Johnson 基于 LR 分析技术开发的解析器，于 1975～1978 年写成。大约在 1985 年，UC Berkeley 的研究生 Bob Corbett 使用改进的内部算法实现了伯克利 yacc，接着来自 FSF 的 Richard Stallman 改写了伯克利 yacc，向其中添加了很多特性并将其用于 GNU 项目，这便形成了今天的 GNU bison。

### 安装 bison

bison 的最新版本是 3.7.90，发布于 2021 年 8 月 13 日。

#### Ubuntu

Ubuntu 官方源中的 bison 的最新版本是 3.5.1，如果你需要安装最新版本的 bison，请自行探索。

- [ ] TODO: Ubuntu 16.04 和 18.04

```bash
$ sudo apt install bison
```

#### MacOS

- [ ] TODO

#### Windows

自行探索

### bison 的使用

bison 通常与 flex 配合使用（flex 负责解析词法，bison 负责解析语法）。通常的做法是先使用 flex 对输入文本进行词法分析并生成 token 流，然后由 bison 读取用户提供的语法规则，生成用于解析 token 流的代码，再由这部分生成的代码来解析 token 流。

bison 源文件的扩展名为 `.y`，分为「声明」、「定义」、「规则」、「用户子程序」四部分。我们以一个四则运算计算器为示例介绍 bison 和 flex 的配合使用，其文法如下（与本章开头的文法相同）：

```c
expr -> term | expr '+' term | expr '-' term
term -> factor | term '*' factor | term '/' factor
factor -> '(' expr ')' | number
number -> [0-9]+ | [0-9]+ '.' [0-9]* | [0-9]* '.' [0-9]+
```

#### 生成 C 语言代码

首先编写 flex 源文件 `calc.l` 如下：

```c
/* calc.l */
%option noyywrap

%{
#include "calc.tab.h"
%}

/* 将不同的符号解析成不同的 token */
%%
\( { return LPAREN; }
\) { return RPAREN; }
"+" | "-" { yylval.op = yytext[0]; return ADDOP; }
"*" | "/" { yylval.op = yytext[0]; return MULOP; }
[0-9]+ | [0-9]+\.[0-9]* | [0-9]*\.[0-9]+ { yylval.num = atof(yytext); return NUMBER; }
" " | \t {  }
\r\n | \n | \r { return RET; }
%%
```

- `LPAREN, RPAREN, ADDOP` 等 token 的名称会定义在 bison 源文件的「定义」部分中。bison 生成语法分析器代码后，这些符号会被写入 `calc.tab.h` 文件。
- `yylval` 是 flex 的全局变量，用于在 flex 和 bison 之间传值，其默认类型为 `int`。这里的 `yylval` 的类型为 `union { char op; double num; }`，表示要么是一个 `char`，要么是一个 `num`，同样定义在 bison 源文件的「定义」部分中。

接下来要编写 bison 的源文件 `calc.y`。

「声明」部分的代码用 `%{` 和 `%}` 包裹，会被 bison 原样复制到生成的代码中，你可以在这里书写声明。如下是一个「声明」部分的示例：

```c
/* calc.y */
%{
#include <stdio.h>
    int yylex(void);
    void yyerror(const char *s);
%}
```

这里声明了 `yylex()` 和 `yyerror(const char *)` 两个函数，其中 `yylex` 函数会由 flex 生成，`yyerror` 需要在最后的「用户子程序」部分定义。

「定义」部分用于定义一些 bison 中专有的变量、类型等。

```c
/* calc.y */
%token RET
%token <num> NUMBER
%token <op> ADDOP MULOP LPAREN RPAREN
%type <num> line expr term factor

%union {
    char   op;
    double num;
}
```

这里包括了对 token 的定义、对部分 token 类型的定义、对非终结符类型的定义以及 对 `yylval` 类型的定义。

「规则」部分的代码用 `%%` 和 `%%` 包裹，在这里你可以编写一系列语法规则。你需要使用 `:` 代表一个 BNF 文法中的 `->` 或 `::=`；同一非终结符的不同规则使用 `|` 分隔；使用 `;` 表示一个非终结符的规则的结束。每条规则的后面可以插入一段 C 代码，当该规则被应用时，这段代码会被执行。

bison 会将「规则」部分中第一个规则左部的非终结符作为语法的起始符号。

「规则」中的非终结符都不需要预先定义，因为一个符号是否为非终结符可以由 bison 通过所有规则的左部推断出来。对于终结符，单字符的终结符可以直接使用单引号包裹，多字符的终结符则需要在定义部分中定义，由 bison 为其分配一个编号。

下面我们将已有的文法翻译成 bison 源文件：

```c
/* calc.y */
%%

calculator
: calculator line { }
| { }

line
: expr RET
{
    printf(" = %f\n", $1);
}

expr
: term
{
    $$ = $1;
}
| expr ADDOP term
{
    switch ($2) {
    case '+': $$ = $1 + $3; break;
    case '-': $$ = $1 - $3; break;
    }
}

term
: factor
{
    $$ = $1;
}
| term MULOP factor
{
    switch ($2) {
        case '*': $$ = $1 * $3; break;
        case '/': $$ = $1 / $3; break;
    }
}

factor
: LPAREN expr RPAREN
{
    $$ = $2;
}
| NUMBER
{
    $$ = $1;
}

%%
```

在这里，我们对文法做了些许改动，使得生成的代码可以识别出换行符后便输出这一行的运算结果，从而进行多次运算。

- `$$` 表示规则左部非终结符的值；
- `$1, $2, ..., $n` 表示规则右部每一个符号的值。如果符号是终结符，它的值就是 `calc.l` 中对应的 `yylval` 的值。

「用户子程序」部分由自己实现，这里实现了解析错误时用于报错的 `yyerror` 函数。

```c
/* calc.y */
void yyerror(const char *s)
{
    fprintf(stderr, "%s\n", s);
}
```

代码全部编写完成后，分别使用 flex 和 bison 生成词法分析程序和语法分析程序。flex 生成的词法分析程序文件是 `lex.yy.c`，bison 生成的语法分析程序文件包括 `calc.tab.c, calc.tab.h`。

```bash
$ flex calc.l
$ bison -d calc.y
# -d 选项表示同时生成头文件，方便和 flex 联动
```

我们编写一个驱动程序，调用 bison 生成的 `yyparse()` 函数来解析输入的字符串。

```c
/* driver.c */
int yyparse();

int main() {
    yyparse();
    return 0;
}
```

将这些文件放到一起编译并生成二进制文件：

```bash
$ gcc lex.yy.c calc.tab.c driver.c -o calc
```

调用二进制文件就可以得到一个简单的计算机应用：

```bash
$ ./calc
1919 * 810
 = 1554390.000000
123.456 - 654.321
 = -530.865000
4. * .6
 = 2.400000
1 + 1 * 4
 = 5.000000
(5 - 1) * 4
 = 16.000000
6 * 0 -
syntax error
```

## ANTLR

ANTLR（ANother Tool for Language Recognition）是一款强大的语法分析器生成工具，基于 LL(*) 分析技术。ANTLR 通过解析用户自定义的上下文无关文法，自动生成词法分析器、语法分析器。

ANTLR 支持多种代码生成目标，包括 Java、C++、C#、Python、Go、JavaScript、Swift 等。

### 安装 ANTLR

- [ ] TODO

#### Ubuntu

#### MacOS

#### Windows

自行探索

### ANTLR 的文法文件：`.g4`

- [ ] TODO

### ANTLR 的使用

- [ ] TODO

#### 生成 C++ 代码

#### 生成 Java 代码

