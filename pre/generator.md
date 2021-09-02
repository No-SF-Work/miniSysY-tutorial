# 解析器的生成工具简介

在后续完成一个完整的编译器的实验中，我们不反对你使用解析器生成工具来自动生成编译器前端词法分析和语法分析部分的代码。

如果你打算手动实现你的编译器的词法分析和语法分析，可以跳过此节。

这部分会涉及到后面的知识，如果你看不懂一些地方，这是正常的。

本节所使用的示例文法如下：

```c
integer -> dec | oct | hex
dec -> nonzero-digit | dec digit
oct -> '0' | oct oct-digit
hex -> hex-prefix hex-digit | hex hex-digit
hex-prefix -> '0x' | '0X'
nonzero-digit -> '1' | '2' | '3' | '4' | '5' | '6' | '7' | '8' | '9'
oct-digit -> '0' | '1' | '2' | '3' | '4' | '5' | '6' | '7'
hex-digit -> '0' | '1' | '2' | '3' | '4' | '5' | '6' | '7' | '8' | '9' | 'a' | 'b' | 'c' | 'd' | 'e' | 'f' | 'A' | 'B' | 'C' | 'D' | 'E' | 'F'
```

## Flex

Flex（the Fast Lexical Analyzer Generator）是 Lawrence Berkeley 实验室研发的一款**词法分析程序的生成工具**，它的前身是我们在课程中学到的 Lex。

### 安装 Flex

### Flex 文件结构




## Bison

Bison 是一款**语法分析程序的生成工具**。Bison 的前身是 yacc。yacc 由贝尔实验室的 S.C.Johnson 基于 LR 分析技术，于 1975～1978 年写成。大约1985年，UC Berkeley 的研究生 Bob Corbett 使用改进的内部算法实现了伯克利 yacc，来自 FSF 的 Richard Stallman 改写了伯克利 yacc 并将其用于 GNU 项目，添加了很多特性，形成了今天的GNU Bison。

### 安装 Bison

## ANTLR

ANTLR（ANother Tool for Language Recognition）是一款强大的语法分析器生成工具，基于 LL(*) 分析技术。ANTLR 通过解析用户自定义的上下文无关文法，自动生成词法分析器、语法分析器。

ANTLR 支持多种代码生成目标，包括 Java、C++、C#、Python、Go、JavaScript、Swift 等。

### 安装 ANTLR

### ANTLR 的文法文件：`.g4`

### 使用 ANTLR 生成词法分析器和语法分析器

### 词法分析器和语法分析器的使用（以 Java、C++ 为例）

