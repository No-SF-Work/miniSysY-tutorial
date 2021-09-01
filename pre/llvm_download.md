# llvm工具链下载

### Ubuntu 20.04

ubuntu官方源里面的llvm和clang是 10 版本的，已经满足我们的需求（ llvm 以及 clang 版本大于10即可）

我们用新的 ubuntu20.04 iso 试过，直接 apt-get 就行

```shell
sudo apt-get install llvm
sudo apt-get install clang
```

```shell
clang -v #查看版本，若出现版本信息则说明安装成功
lli --version #查看版本，若出现版本信息则说明安装成功 
```

### Ubuntu 18.04

Ubuntu 18.04 版本里默认的llvm是 6.0 版本的，并不满足我们的需求。 你需要

```shell
sudo apt-get install clang-10
sudo apt-get install llvm-10
```

然后在需要使用`clang`和`lli,llc,llvm-link`的地方的末尾加上`-10 ` (或者 alias )

```shell
clang-10 -v #查看版本，若出现版本信息则说明安装成功
lli-10 --version #查看版本，若出现版本信息则说明安装成功 
```

如果你使用的是小于 18.04 版本的 Ubuntu， 快更新吧。

### redhat 系 arch系 以及除了 Ubuntu 的 debian 系

因为问卷里面连上2名助教总共只有三个人用，所以不写了

剩下的那位同学请自己下载~~Fly B***h~~ 

### macOS

如果你没有下载 Xcode ，快去下载，如果你有 Xcode ，那应该是带有llvm以及clang的。

```shell
clang -v #查看版本，若出现版本信息则说明安装成功
lli --version #查看版本，若出现版本信息则说明安装成功 
```

### Windows

“他们都大三了，该让他们使用 *nix 的东西了，不用写 Windows 的教程。”——邵老师

用 windows 也是可以的，请自行摸索。