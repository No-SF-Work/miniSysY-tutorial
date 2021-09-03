# flex/bison/ANTLR 简介

在后续完成一个完整的编译器的实验中，我们不反对你使用解析器生成工具来自动生成编译器前端词法分析和语法分析部分的代码。

如果你打算手动实现你的编译器的词法分析和语法分析，可以跳过此节。

这部分会涉及到后面的知识，如果你看不懂一些地方，这是正常的。

本节的语法分析部分所使用的示例文法如下：

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

flex 源文件的扩展名为 `.l`，分为声明、规则、用户子程序三部分。我们以生成一个统计文本中单词和字母个数的词法分析器为示例介绍 flex 的使用。

#### 生成 C 语言代码

flex 读取一个 `*.l` 或者 `*.lex` 文件里的词法规则，生成的词法分析器代码写入 `lex.yy.c`。

声明部分的代码被 `%{` 和 `%}` 包裹，会被 flex 原样复制到 `lex.yy.c` 中，你可以在这里书写声明和定义。如下是一个声明部分的示例：

```c
/* word_char_counter.l */
%{
#include <string.h> 
int chars = 0;
int words = 0;
%}
```

规则部分的代码被 `%%` 和 `%%` 包裹，你可以使用正则表达式来编写模式，在正则表达式后面编写一段 C 代码，指明匹配到相应的模式后所要完成的动作。如下是一个规则部分的示例：

```c
/* word_char_counter.l */
%%
[a-zA-Z]+ { chars += strlen(yytext); words++; }
. { }
%%
```

在规则部分中，标识符 `yytext` 是一个指针，指向匹配到的输入字符串。

如果要与 bison 配合，用户子程序部分不是必需的；如果只是使用 flex，需要编写用户子程序。

```c
/* word_char_counter.l */
int main(int argc, char **argv) {
    yylex();
    printf("I found %d words of %d chars.\n", words, chars);
    return 0;
}
```

其中 `yylex()` 是 flex 的生成的函数，对输入进行词法分析并完成指定的动作，默认读取 stdin。

编写完成词法规则后，可以使用 flex 生成对应的 C 代码文件。

```bash
$ flex word_char_counter.l
$ gcc lex.yy.c -o word_char_counter
```

这样直接编译，会出现链接错误如下：

```
/usr/bin/ld: /tmp/cc1qil64.o: in function `yylex':
lex.yy.c:(.text+0x4b8): undefined reference to `yywrap'
/usr/bin/ld: /tmp/cc1qil64.o: in function `input':
lex.yy.c:(.text+0x10c7): undefined reference to `yywrap'
collect2: error: ld returned 1 exit status
```

原因是在 flex 2.5.4 版本之后，在程序扫描到 EOF 时，会调用 `yywrap()` 函数，判断是不是还有其他的输入，如果 `yywrap()` 返回 0，程序读取输入的指针在 `yywrap()` 被设置到另一个输入，继续读取输入；如果 `yywrap()` 返回 1，则说明没有其他的输入。在这里，我们可以链接 fl 库，调用其中默认的返回 1 的 `yywrap()` 函数，即 `gcc lex.yy.c -o word_char_counter -lfl` ，或者在源文件的开头加上一行 `%option noyywrap`，不调用 `yywrap()`，解决链接错误。

```bash
$ ./word_car_counter
Hello, flex.
^D
I found 2 words of 9 chars.
```

#### 生成 C++ 语言代码

- [ ] TODO

#### 生成 Java 语言代码

- [ ] TODO

## bison

bison 是一款**语法分析程序的生成工具**。bison 的前身是 yacc。yacc 由贝尔实验室的 S.C.Johnson 基于 LR 分析技术，于 1975～1978 年写成。大约1985年，UC Berkeley 的研究生 Bob Corbett 使用改进的内部算法实现了伯克利 yacc，来自 FSF 的 Richard Stallman 改写了伯克利 yacc 并将其用于 GNU 项目，添加了很多特性，形成了今天的GNU bison。

### 安装 bison

bison 的最新版本是 3.7.90，发布于 2021 年 8 月 13 日。

#### Ubuntu

Ubuntu 官方源中的 bison 的最新版本是 3.5.1，如果你需要安装最新版本的 bison，请自行探索.

- [ ] TODO: Ubuntu 16.04 和 18.04

```bash
$ sudo apt install bison
```

#### MacOS

- [ ] TODO

#### Windows

自行探索

### bison 的使用

bison 通常与 flex 配合使用，使用 flex 对输入文本进行词法分析，生成 token 流，bison 通过读取用户提供的语法规则，生成解析 token 流的代码。

bison 源文件的扩展名为 `.y`，分为声明、定义、规则、用户子程序四部分。我们以一个四则运算计算器为示例介绍 bison 和 flex 的配合使用（文法见本节开头）。

#### 生成 C 语言代码

首先编写 flex 源文件 `calc.l` 如下：

```c
/* calc.l */
%option noyywrap

%{
#include "calc.tab.h"
%}

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

- `LPAREN, RPAREN, ADDOP` 等 token 名称在 bison 源文件的定义部分被定义，由 bison 生成语法分析器代码后，定义在 `calc.tab.h` 文件中。
- `yylval` 是 flex 的全局变量，可以在 flex 和 bison 之间传值，默认类型是 `int`。这里的 `yylval` 的类型是 `union { char op; double num; }`，该类型在 bison 源文件的定义部分中定义。

然后编写 bison 的源文件 `calc.y`。

声明部分的代码被 `%{` 和 `%}` 包裹，会被 bison 原样复制到生成的代码中，你可以在这里书写声明。如下是一个声明部分的示例：

```c
/* calc.y */
%{
#include <stdio.h>
    int yylex(void);
    void yyerror(const char *s);
%}
```

这里声明了 `yylex()` 和 `yyerror(const char *)` 两个函数，其中 `yylex` 会在 flex 生成的代码中定义，`yyerror` 在最后的用户子程序部分定义。

定义部分用于定义一些 bison 中专有的变量、类型等。

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

这里包括对 token 的定义、部分 token 类型的定义、非终结符类型的定义以及 `yylval` 类型的定义。

规则部分的代码被 `%%` 和 `%%` 包裹。在这里编写一系列语法规则，使用 `:` 代表一个 `->` 或 `::=`，同一非终结符的不同规则使用 `|` 分隔，使用 `;` 表示一个非终结符的规则的结束。每条规则的后面可以插入一段 C 代码，在该规则被应用时，代码会执行。

bison 会将规则部分的第一个规则左部的非终结符作为语法的起始符号。

规则中的非终结符不需要预先定义，非终结符可以由 bison 根据所有规则的左部来确定。对于终结符，单个字符的终结符可以直接使用单引号包裹，多字符的终结符则需要在定义部分中定义，由 bison 分配一个编号。

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

在这里，我们对文法有了些许改动，使得生成的代码可以识别出换行符并输出运算结果，并能够进行多次运算。

- `$$` 表示规则左部非终结符的值；
- `$1, $2, ..., $n` 表示规则右部每一个符号的值，如果符号是终结符，它的值就是 `calc.l` 中对应的 `yylval` 的值。

用户子程序部分由自己实现，这里实现了解析错误时报错的 `yyerror` 函数。

```c
/* calc.y */
void yyerror(const char *s)
{
    fprintf(stderr, "%s\n", s);
}
```

代码全部编写完成后，分别使用 flex 和 bison 生成词法分析程序和语法分析程序。flex 生成的词法分析程序文件是 `lex.yy.c`，bison 生成的语法分析程序文件包括 `calc.tab.c, calc.tab.h`。

```bash
flex calc.l
bison -d calc.y
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

```bash
gcc lex.yy.c calc.tab.c driver.c -o calc
```

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
