# LLVM 相关工具链下载

**注意**：在我们的实验中要求 Clang 和 LLVM 的版本至少为 10.0。

## Ubuntu

### 20.04 或更新版本

对于 Ubuntu 20.04 或更新版本，官方源中的 LLVM 版本已经默认为 10+，因此执行以下命令即可安装：

```shell
$ sudo apt-get install llvm
$ sudo apt-get install clang
```

安装完成后可以通过以下命令进行测试：

```shell
$ clang -v # 查看版本，若出现版本信息则说明安装成功
$ lli --version # 查看版本，若出现版本信息则说明安装成功
```

### 18.04

对于 Ubuntu 18.04，官方源中的 LLVM 版本仍然停留在 6.0，因此你需要在安装时额外指定版本号：

```shell
$ sudo apt-get install llvm-10
$ sudo apt-get install clang-10
```

相应的，使用时也需要在末尾额外加上 `-10` 用来指定版本，如 `clang-10` 或 `lli-10`。（当然你也可以用 `alias` 设置别名）

完成安装后可以通过以下命令进行测试：

```shell
$ clang-10 -v # 查看版本，若出现版本信息则说明安装成功
$ lli-10 --version # 查看版本，若出现版本信息则说明安装成功
```

### 更老版本

快去更新。

### 如果你的 apt 因为某种原因不能用上述方式下载
把
```
# i386 not available
deb http://apt.llvm.org/focal/ llvm-toolchain-focal main
deb-src http://apt.llvm.org/focal/ llvm-toolchain-focal main
# 9
deb http://apt.llvm.org/focal/ llvm-toolchain-focal-9 main
deb-src http://apt.llvm.org/focal/ llvm-toolchain-focal-9 main
# 10
deb http://apt.llvm.org/focal/ llvm-toolchain-focal-10 main
deb-src http://apt.llvm.org/focal/ llvm-toolchain-focal-10 main
``` 
加到 `/etc/apt/sources.list`  
然后在终端执行  
`wget -O - https://apt.llvm.org/llvm-snapshot.gpg.key|sudo apt-key add -`  
然后在终端执行  
`apt-get install clang-10 lldb-10 lld-10`


## Redhat/Arch/...（Ubuntu/Debian 以外的）

因为问卷里面连上 2 名助教总共只有三个人用，所以不写了。通过包管理器下载 Clang 和 LLVM 即可（记得注意版本号）。

> ~~Fly B\*\*\*h~~

## macOS

### Clang

在 macOS 上，如果你已经安装过 XCode 或 XCode Command Line Tools，则其默认已经附带了 `clang`。

你可以在「终端」应用中输入以下命令进行测试：

```shell
$ clang -v # 查看 Clang 版本，若出现版本信息则说明安装成功
```

否则，你需要安装 XCode，或者运行以下命令安装 XCode Command Line Tools：

```shell
$ xcode-select --install
```

### LLVM

由于 XCode 自带的 LLVM 工具链并不完整，因此你需要手动安装 LLVM 相关的包。

```shell
$ brew install llvm
```

安装完成后，你需要在配置文件中将 LLVM 的路径添加到 `$PATH`。

```shell
echo 'export PATH="/usr/local/opt/llvm/bin:$PATH"' >> ~/.bash_profile
```

如果你使用的是 `zsh` 或者其他 shell，请自行在对应的配置文件中添加环境变量。

然后重启「终端」。

重启完成后，你可以在「终端」应用中输入以下命令进行测试：

```shell
$ lli --version # 查看 LLVM 版本，若出现版本信息则说明安装成功
```

## Windows

> “他们都大三了，该让他们使用 *nix 的东西了，不用写 Windows 的教程。”
>
> ——邵老师

当然在 Windows 上也可以安装相应的 Clang+LLVM 工具链，请自行摸索。

在本实验中，我们引入了 `libsysy` 库（在 [这里](https://github.com/BUAA-SE-Compiling/miniSysY-tutorial/blob/master/files/libsysy.zip) 可以看到）为我们的程序提供 IO 方面的操作。
