# bison

bison 是一款**语法分析程序的生成工具**，其前身为 yacc。yacc 由贝尔实验室的 S.C.Johnson 基于 LR 分析技术开发的解析器，于 1975 ～ 1978 年写成。大约在 1985 年，UC Berkeley 的研究生 Bob Corbett 使用改进的内部算法实现了伯克利 yacc，接着来自 FSF 的 Richard Stallman 改写了伯克利 yacc，向其中添加了很多特性并将其用于 GNU 项目，这便形成了今天的 GNU bison。

## 安装 bison

bison 的最新版本是 3.7.90，发布于 2021 年 8 月 13 日。

### Ubuntu

Ubuntu 20.04 官方源中的 bison 的最新版本是 3.5.1，Ubuntu 18.04 官方源中的 bison 的最新版本是 3.0.4，如果你需要安装最新版本的 bison，请自行探索。

```shell
$ sudo apt install bison
```

### MacOS

```shell
$ brew install bison
```

Homebrew 中 bison 的最新版本是 3.8.1，MacOS 中自带的 bison 版本是 2.3，需要手动指定环境变量以使用较新的 bison。

```shell
$ echo 'export PATH="/usr/local/opt/bison/bin:$PATH"' >> ~/.bash_profile
```

### Windows & other Linux

自行探索

## bison 的使用

bison 通常与 flex 配合使用（flex 负责解析词法，bison 负责解析语法）。通常的做法是先使用 flex 对输入文本进行词法分析并生成 token 流，然后由 bison 读取用户提供的语法规则，生成用于解析 token 流的代码，再由这部分生成的代码来解析 token 流。

bison 源文件的扩展名为 `.y`，分为「声明」、「定义」、「规则」、「用户子程序」四部分。下面以一个四则运算计算器为示例介绍 bison 和 flex 的配合使用，其文法如下（与本章开头的文法相同）：

```c
expr -> term | expr '+' term | expr '-' term
term -> factor | term '*' factor | term '/' factor
factor -> '(' expr ')' | number
number -> [0-9]+ | [0-9]+ '.' [0-9]* | [0-9]* '.' [0-9]+
```

### 生成 C 代码

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
"+"|"-" { yylval.op = yytext[0]; return ADDOP; }
"*"|"/" { yylval.op = yytext[0]; return MULOP; }
[0-9]+|[0-9]+\.[0-9]*|[0-9]*\.[0-9]+ { yylval.num = atof(yytext); return NUMBER; }
" "|\t {  }
\r\n|\n|\r { return RET; }
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
%union {
    char   op;
    double num;
}

%token RET
%token <num> NUMBER
%token <op> ADDOP MULOP LPAREN RPAREN
%type <num> line expr term factor
```

这里包括了对 token 的定义、对部分 token 类型的定义、对非终结符类型的定义以及 对 `yylval` 类型的定义。

「规则」部分的代码用 `%%` 和 `%%` 包裹，在这里你可以编写一系列语法规则。你需要使用 `:` 代表一个 BNF 文法中的 `->` 或 `::=`；同一非终结符的不同规则使用 `|` 分隔；使用 `;` 表示一个非终结符的规则的结束。每条规则的后面可以插入一段 C 代码，当该规则被应用时，这段代码会被执行。

bison 会将「规则」部分中第一个规则左部的非终结符作为语法的起始符号。

「规则」中的非终结符都不需要预先定义，因为一个符号是否为非终结符可以由 bison 通过所有规则的左部推断出来。对于终结符，单字符的终结符可以直接使用单引号包裹，多字符的终结符则需要在定义部分中定义，由 bison 为其分配一个编号。

下面将已有的文法翻译成 bison 源文件：

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
void yyerror(const char *s) {
    fprintf(stderr, "%s\n", s);
}
```

代码全部编写完成后，分别使用 flex 和 bison 生成词法分析程序和语法分析程序。flex 生成的词法分析程序文件是 `lex.yy.c`，bison 生成的语法分析程序文件包括 `calc.tab.c, calc.tab.h`。

```shell
$ flex calc.l
$ bison -d calc.y
# -d 选项表示同时生成头文件，方便和 flex 联动
```

下面编写一个驱动程序，调用 bison 生成的 `yyparse()` 函数来解析输入的字符串。

```c
/* driver.c */
int yyparse();

int main() {
    yyparse();
    return 0;
}
```

将这些文件放到一起编译并生成二进制文件：

```shell
$ gcc lex.yy.c calc.tab.c driver.c -o calc
```

调用二进制文件就可以得到一个简单的计算器应用：

```shell
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

### 生成 C++ 代码

> 详见 [官方文档](https://www.gnu.org/software/bison/manual/)
