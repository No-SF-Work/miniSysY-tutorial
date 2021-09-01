# LLVM 工具链简介

如果你还没有完成工具链的下载，请转到  todo。

如果你对 LLVM 工具链以及 LLVM IR 比较熟悉，可以跳过此节。

这部分会涉及到后面的知识，如果你看不懂一些地方，这是正常的。

llvm工具链里有很多很有意思的工具，下面我们会选择几个对实验来说较为重要的工具进行介绍。

在开始介绍前，让我们先编写一个最简单的 a+b 程序,下面的介绍里我们会以这个程序作为输入文件。

``` c
//main.c
int main(){
    int a =19260817;
   	int b =42;
    return a+b;
}
```

## clang

clang 是 llvm project 的一个子项目，是基于 llvm 架构的`c/c++/obj-c`编译器前端。

clang 的用法基本和 gcc 相同。

可以在终端输入 `clang -help` 查看所有指令

预计将在实验中用到的指令有

```shell
clang main.c -o main #生成可执行文件
clang -ccc-print-phases main.c #查看编译的过程
clang  -E -fsyntax-only -Xclang -dump-tokens main.c #生成tokens
clang  -fsyntax-only -Xclang -ast-dump main.c #生成语法树,-fsyntax-only的意思是防止编译器生成代码 
clang -S  -emit-llvm main.c -o main.ll -O0 #生成llvm ir,不开优化
clang -S  main.m -o main.s #生成汇编，本次实验里用处不大
clang -c main.m -o main.o #生成目标文件，本次实验里用处不大
```

试着在命令行里面输入这些指令，看看都输出了什么，我们会在别的地方详细介绍其中的一些内容。

## lli

lli 以 `.bc` 的格式解释（或者 JIT 编译）执行程序，他也能直接解释运行 `.ll`格式的文件。

在实验中，我们只需要用最简单的形式直接输入指令使用即可。

以 main.c 为例

```shell
# 1.我们首先生成 main.c 对应的 .ll 格式的文件
clang -S  -emit-llvm main.c -o main.ll -O0 
# 2.我们使用 lli 解释执行生成的 .ll 文件
lli main.ll
```

如果一切正常，在查看上一条指令的返回值时，你会看到。

``` shell
echo $?
187   #(19260817+42) mod 256
```

## llvm-link

lli 仅能解释单个 `.ll` 或者是 `.bc`格式的文件，当我们想要使用别的库的时候，就需要用到llvm-link

在本实验中，我们引入了 `libsysy`(libsysy在<u>这里</u>//todo)以后,会用到库里面的IO函数

比如，如果我们想要输出一个int的值，需要用到 `putint()`这个函数，把 main.c 改成了下面的样子

``` c
int main() {
    int a=19260817;
    int b=42;
    putint(a);
    return a+b;
}
```

如果我们按照上面的方法直接解释运行 main.ll 的话，会变成

```shell
lli main.ll
PLEASE submit a bug report to https://bugs.llvm.org/ and include the crash backtrace.
Stack dump:
0.      Program arguments: lli main.ll
zsh: segmentation fault (core dumped)  lli main.ll
```

这是因为 lli 只解释了 main.ll 这一个文件，找不到库函数 `putint`在哪，这时候就需要 llvm-link 了。

llvm-link 能够将多个 `.bc`或者是`.ll`格式的文件链接为一个文件

``` shell
# 1.我们先分别导出 libsysy 和 main.c 的 .ll 文件
clang -emit-llvm -S libsysy.c -o lib.ll 
clang -emit-llvm -S main.c -o main.ll
# 2.然后使用 llvm-link 将两个文件链接
llvm-link main.ll lib.ll  -o out.ll  
# 3.lli 解释运行
lli out.ll
19260817 
```

## 其他可能有用的指令

**llc** : 将`.ll`形式的文件编译到指定的体系结构的汇编语言

**opt**: LLVM模块化的优化器和分析器。它将LLVM源文件作为输入，对其运行指定的优化或分析，然后输出优化文件或分析结果。这个指令会在挑战任务的时候介绍，在此不再展开。

[拓展阅读](https://llvm.liuxfe.com/docs/man/lli)

