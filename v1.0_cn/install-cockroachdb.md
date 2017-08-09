# 安装 CockroachDB

在 Mac、Linux 或 Windows 上安装 CockroachDB。

## Mac

### 通过二进制文件进行安装

1、下载 [CockroachDB v1.0.4 版本的 OSX 二进制包](https://binaries.cockroachdb.com/cockroach-v1.0.4.darwin-10.9-amd64.tgz)

2、提取二进制文件

```sh
$ tar xfz cockroach-v1.0.4.darwin-10.9-amd64.tgz
```

3、将得到的二进制文件移至 `PATH` 环境变量指定的目录，这样能更容易地在终端中运行 [cockroach 命令](https://www.cockroachlabs.com/docs/stable/cockroach-commands.html)

```sh
$ cp -i cockroach-v1.0.4.darwin-10.9-amd64/cockroach /usr/local/bin
```

4、确认 CockroachDB 是否安装成功

```sh
$ cockroach version
```

### 使用 Homebrew 进行安装

1、[安装 HomeBrew](https://brew.sh/)

2、运行 Homebrew 命令安装 CockroachDB

```sh
$ brew install cockroach
```

3、确认 CockroachDB 是否安装成功

```sh
$ cockroach version
```

### 从源码进行编译

1、安装必要的工具，如下

|                |                                      |
| ------------------- | ---------------------------------------- |
|C++ 编译器|必须支持 C++ 标准，GCC 6.0 优先，如不能运行，查看如下 [issues](https://gcc.gnu.org/bugzilla/show_bug.cgi?id=48891)，在 macOS 下 Xcode 可做支持|
|Go|1.8.1 版本 Go 语言环境|
|Bash|版本为 4 以上，但已知的 3.x 版本的 Bash 也可以正常运行|
|CMake|版本为 3.81 以上|
-----------

**强烈推荐使用 64 位操作系统。并未在 32 位操作系统上进行 CockroachDB 安装和运行的测试。RAM 至少为 2GB。并且如果你想测试顺利，请使用 4GB RAM。**

2、下载 [CockroachDB v1.0.4 源代码](https://binaries.cockroachdb.com/cockroach-v1.0.4.src.tgz)

3、提取源代码

```sh
$ tar xfz cockroach-v1.0.4.src.tgz
```

4、在生成的目录中运行 `make build`

```sh
$ cd cockroach-v1.0.4
$ make build
```
编译过程可能需要 10 分钟以上，请耐心等待!

> 二进制的源文件包含受 Apache License 2 (APL2) 协议保护的核心开源代码和受  CockroachDB Community License (CCL) 协议保护的企业内部代码。可以使用 `make buildoss` 构建一个排除企业内部代码的纯开源 (APL2) 版本。[查看更多细节](https://www.cockroachlabs.com/blog/how-were-building-a-business-to-last/)

5、安装 `cockroach` 二进制文件至 `/usr/local/bin` 目录中，这样就可以在任何目录中运行 [cockroach 命令](https://www.cockroachlabs.com/docs/stable/cockroach-commands.html)

```sh
$ make install
```

如果在安装过程中遇到权限错误，请在命令前加上 `sudo`

你也可以直接在编译的路径运行 `cockroach` 二进制文件，假设二进制文件在 PATH 路径中：`./src/github.com/cockroachdb/cockroach/cockroach`。

6、确认 CockroachDB 是否安装成功

```sh
$ cockroach version
```

### 使用 Docker 进行安装

#### 警告

**在 Docker 中运行像 CockroachDB 这一类有状态的应用程序将会更加复杂且更易出错，除非你对 Docker 具有丰富的经验，因此我们推荐使用其它的安装和部署方式。**
