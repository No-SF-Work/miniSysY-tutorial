# Part 8 作用域与块

在 Part 8 中，你的编译器需要完善对语句块和作用域的语义支持。

语法规则没有变化。

## 语义约束

### `Block`

- `Block` 表示语句块。语句块会创建作用域，语句块内声明的变量的生命周期在该语句块内。
- 同一作用域中不能有同名的变量或常量。
- 语句块内可以再次定义与语句块外同名的变量或常量（通过 `Decl` 语句)，其作用域从定义处开始到该语句块尾结束，它隐藏**语句块外**的同名变量或常量。

## 示例

### 样例 1

样例程序 1：

```c
int main() {
    int a = getint();
    {
        int b = 2;
        putint(a + b);
        int a = getint();
        putint(a + b);
    }
    int b = a + 2;
    putint(a + b);
    return 0;
}
```

示例 IR 1：

```llvm
declare i32 @getint()
declare void @putint(i32)
define dso_local i32 @main() {
    %1 = alloca i32
    %2 = alloca i32
    %3 = alloca i32
    %4 = alloca i32
    %5 = call i32 @getint()
    store i32 %5, i32* %4
    store i32 2, i32* %3
    %6 = load i32, i32* %4
    %7 = load i32, i32* %3
    %8 = add i32 %6, %7
    call void @putint(i32 %8)
    %9 = call i32 @getint()
    store i32 %9, i32* %2
    %10 = load i32, i32* %2
    %11 = load i32, i32* %3
    %12 = add i32 %10, %11
    call void @putint(i32 %12)
    %13 = load i32, i32* %4
    %14 = add i32 %13, 2
    store i32 %14, i32* %1
    %15 = load i32, i32* %4
    %16 = load i32, i32* %1
    %17 = add i32 %15, %16
    call void @putint(i32 %17)
    ret i32 0
}
```

输入样例 1：

```c
1 5
```

输出样例 1：

```c
374
```

### 样例 2

样例程序 2：

```c
int main() {
    const int c1 = 10 * 5 / 2;
    const int c2 = c1 / 2, c3 = c1 * 2;
    if (c1 > 24) {
        int c1 = 24;
        putint(c2 - c1 * c3);
        putch(10);
    }
    {
        int c2 = c1 / 4;
        putint(c3 / c2);
        {
            int c3 = c1 * 4;
            putint(c3 / c2);
        }
    }
    putch(10);
    putint(c3 / c2);
    return 0;
}
```

示例 IR 2：

```llvm
declare void @putint(i32)
declare void @putch(i32)
define dso_local i32 @main() {
    %1 = alloca i32
    %2 = alloca i32
    %3 = alloca i32
    %4 = icmp sgt i32 25, 24
    br i1 %4, label %5, label %9

5:
    store i32 24, i32* %3
    %6 = load i32, i32* %3
    %7 = mul i32 %6, 50
    %8 = sub i32 12, %7
    call void @putint(i32 %8)
    call void @putch(i32 10)
    br label %9

9:
    %10 = sdiv i32 25, 4
    store i32 %10, i32* %2
    %11 = load i32, i32* %2
    %12 = sdiv i32 50, %11
    call void @putint(i32 %12)
    %13 = mul i32 25, 4
    store i32 %13, i32* %1
    %14 = load i32, i32* %1
    %15 = load i32, i32* %2
    %16 = sdiv i32 %14, %15
    call void @putint(i32 %16)
    call void @putch(i32 10)
    %17 = sdiv i32 50, 12
    call void @putint(i32 %17)
    ret i32 0
}
```

输出样例 2:

```c
-1188
816
4
```

### 样例 3

样例程序 3：

```c
int main() {
    int a = 1;
    int a = 2;
    return 0;
}
```

输出样例 3：

编译器直接以**非 0 的返回值**退出。
