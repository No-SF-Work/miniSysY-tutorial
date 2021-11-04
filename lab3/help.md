# Lab 3 实验指导

## 为什么示例 IR 中使用 `alloca`、`store` 和 `load`

[LLVM IR SSA 介绍](../pre/llvm_ir_ssa.md)

## 在本地调试带有库函数的 LLVM IR

`lli` 仅能运行单个 `.ll` 文件，当我们想要使用别的库的时候，就需要用到 `llvm-link`。

在本实验中，我们引入了 `libsysy` 库（在 [这里](https://github.com/BUAA-SE-Compiling/miniSysY-tutorial/blob/master/files/libsysy.zip) 可以看到）为我们的程序提供 IO 方面的操作。

```c
/* libsysy.c */
#include "libsysy.h"
#include <stdio.h>
/* Input & output functions */
int getint() {
    int t;
    scanf("%d", &t);
    return t;
}
int getch() {
    char c;
    scanf("%c", &c);
    return (int)c;
}
int getarray(int a[]) {
    int n;
    scanf("%d", &n);
    for (int i = 0; i < n; i++)
        scanf("%d", &a[i]);
    return n;
}
void putint(int a) { printf("%d", a); }
void putch(int a) { printf("%c", a); }
void putarray(int n, int a[]) {
    printf("%d:", n);
    for (int i = 0; i < n; i++)
        printf(" %d", a[i]);
    printf("\n");
}
```

```c
/* libsysy.h */
#ifndef __SYLIB_H_
#define __SYLIB_H_

#include <stdarg.h>
#include <stdio.h>
#include <sys/time.h>
/* Input & output functions */
int  getint(), getch(), getarray(int a[]);
void putint(int a), putch(int a), putarray(int n, int a[]);
#endif
```

你需要使用 `clang` 将 `libsysy.c` 编译成 `.ll` 文件，然后使用 `llvm-link` 与你想要运行的调用 miniSysY 运行时库函数的 `.ll` 文件链接，生成新的 LLVM IR，再使用 `lli` 解释执行。

例如：

```llvm
; main.ll
declare i32 @getint()
declare void @putint(i32)
define dso_local i32 @main(){
    %1 = alloca i32
    %2 = call i32 @getint()
    store i32 %2, i32* %1
    %3 = load i32, i32* %1
    %4 = add i32 %3, 4
    call void @putint(i32 %4)
    ret i32 0
}
```

```shell
$ clang -emit-llvm -S libsysy.c -o lib.ll
$ ./你的编译器 main.sy -o main.ll
$ llvm-link main.ll lib.ll -S -o out.ll
$ lli out.ll
```
