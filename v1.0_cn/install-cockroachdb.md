# 安装 CockroachDB

在 Mac、Linux 和 Windows 上安装 CockroachDB。

## Mac

### 使用二进制文件安装

1、下载 [CockroachDB v1.0.4 版本的 OSX 二进制包](https://binaries.cockroachdb.com/cockroach-v1.0.4.darwin-10.9-amd64.tgz)。

2、提取二进制文件：

```sh
$ tar xfz cockroach-v1.0.4.darwin-10.9-amd64.tgz
```

3、将得到的二进制文件移至 `PATH` 环境变量指定的目录，这样能更容易地在终端中运行：[cockroach 命令](https://www.cockroachlabs.com/docs/stable/cockroach-commands.html)

```sh
$ cp -i cockroach-v1.0.4.darwin-10.9-amd64/cockroach /usr/local/bin
```

4、确认 CockroachDB 是否安装成功：

```sh
$ cockroach version
```

### 下一步

[快速开始](start-a-local-cluster.md)搭建一个单节点或多节点的本地集群并通过内建的 SQL 客户端进行访问。

### 使用 Homebrew 安装

1、[安装 HomeBrew](https://brew.sh/)。

2、运行 Homebrew 命令安装 CockroachDB：

```sh
$ brew install cockroach
```

3、确认 CockroachDB 是否安装成功：

```sh
$ cockroach version
```

### 下一步

[快速开始](start-a-local-cluster.md)搭建一个单节点或多节点的本地集群并通过内建的 SQL 客户端进行访问。

### 从源码构建

1、安装必要的工具，如下：

|          工具        |             描述                         |
| ------------------- | ---------------------------------------- |
| C++ 编译器 | 必须支持 C++ 11。早于 6.0 版本的 GCC 由于[这个问题](https://gcc.gnu.org/bugzilla/show_bug.cgi?id=48891) 不能用。在 macOS 上，Xcode 足够用。 |
| Go | 需要 1.8.1 版本。 |
| Bash | 首选 4 以上版本，但从 3.x 以后的版本也听说能用。 |
| CMake | 3.81 以上版本听说能用。 |

**强烈推荐使用 64 位操作系统。并未在 32 位操作系统上进行 CockroachDB 安装和运行的测试。RAM 至少为 2GB。并且如果你想测试顺利，请使用 4GB RAM。**

2、下载 [CockroachDB v1.0.4 源代码](https://binaries.cockroachdb.com/cockroach-v1.0.4.src.tgz)。

3、提取源代码：

```sh
$ tar xfz cockroach-v1.0.4.src.tgz
```

4、在提取的目录中运行 `make build`：

```sh
$ cd cockroach-v1.0.4
$ make build
```
编译过程可能需要 10 分钟以上，请耐心等待!

> 缺省的二进制文件包含受 Apache License 2 (APL2) 许可保护的核心开源功能和受  CockroachDB Community License (CCL) 许可保护的企业功能。可以使用 `make buildoss` 构建一个排除企业功能的纯开源 (APL2) 版本。更多细节见[博客](https://www.cockroachlabs.com/blog/how-were-building-a-business-to-last/)。

5、安装 `cockroach` 二进制文件至 `/usr/local/bin` 目录中，这样就可以在任何目录下运行 [cockroach 命令](cockroach-commands.md)：

```sh
$ make install
```

如果在安装过程中遇到权限错误，请在命令前加上 `sudo`。

你也可以直接在编译的位置运行 `cockroach` 二进制文件，`./src/github.com/cockroachdb/cockroach/cockroach`，文档的剩余部分假设二进制文件在 PATH 中。

6、确认 CockroachDB 是否安装成功：

```sh
$ cockroach version
```

### 下一步

[快速开始](start-a-local-cluster.md)搭建一个单节点或多节点的本地集群并通过内建的 SQL 客户端进行访问。

### 使用 Docker 安装

> **警告**<br>在 Docker 中运行像 CockroachDB 这一类有状态的应用程序将会更加复杂且更易出错，除非你对 Docker 具有丰富的经验，因此我们推荐使用其它的安装和部署方式。

1、安装 [Mac 版的 Docker](https://docs.docker.com/docker-for-mac/install/) ，安装时请仔细检查所有的必要条件。

2、确认 Docker 的后台程序正在运行：

```sh
$ docker version
```

如果没有见到列出的服务程序，启动 Docker 应用。

3、从 [Docker 镜像库](https://hub.docker.com/r/cockroachdb/cockroach/)下载官方 CockroachDB 镜像：

```sh
$ docker pull cockroachdb/cockroach:v1.0.4
```

4、确认 CockroachDB 可运行：

```sh
$ docker run --rm cockroachdb/cockroach:v1.0.4 version
```

### 下一步

在一台主机上，跨跨多个 Docker 容器[快速启动](start-a-local-cluster-in-docker.md)一个多节点集群，使用 Docker 卷持久化节点数据，或者使用[编排](orchestration.md)工具在容器中探索运行一个物理上分布的集群。

## Linux

### 使用二进制文件安装

1、下载 [CockroachDB v1.0.4 版本的 Linux 二进制包](https://binaries.cockroachdb.com/cockroach-v1.0.4.linux-amd64.tgz)。

2、提取二进制文件：

```sh
$ tar xfz cockroach-v1.0.4.linux-amd64.tgz
```

3、将得到的二进制文件移至 `PATH` 环境变量指定的目录，这样能更容易地在终端中运行： [cockroach 命令](https://www.cockroachlabs.com/docs/stable/cockroach-commands.html)

```sh
$ cp -i cockroach-v1.0.4.linux-amd64/cockroach /usr/local/bin
```

如果在安装过程中遇到权限错误，请在命令前加上 `sudo`：

4、确认 CockroachDB 是否安装成功：

```sh
$ cockroach version
```

### 下一步

[快速开始](start-a-local-cluster.md)搭建一个单节点或多节点的本地集群并通过内建的 SQL 客户端进行访问。

### 从源码构建

1、安装必要的工具，如下

|         工具         |          描述                            |
| ------------------- | ---------------------------------------- |
| C++ 编译器 | 必须支持 C++ 11。早于 6.0 版本的 GCC 由于[这个问题](https://gcc.gnu.org/bugzilla/show_bug.cgi?id=48891) 不能用。在 macOS 上，Xcode 足够用。|
| Go | 需要 1.8.1 版本。|
| Bash | 首选 4 以上版本，但从 3.x 以后的版本也听说能用。|
| CMake | 3.81 以上版本听说能用。|
| [XZ Utils](https://tukaani.org/xz/) | 5.2.3+ 版本。|

**强烈推荐使用 64 位操作系统。并未在 32 位操作系统上进行 CockroachDB 安装和运行的测试。RAM 至少为 2GB。并且如果你想测试顺利，请使用 4GB RAM。**

2、下载 [CockroachDB v1.0.4 源代码](https://binaries.cockroachdb.com/cockroach-v1.0.4.src.tgz)。

3、提取源代码：

```sh
$ tar xfz cockroach-v1.0.4.src.tgz
```

4、在提取的目录中运行 `make build`：

```sh
$ cd cockroach-v1.0.4
$ make build
```
编译过程可能需要 10 分钟以上，请耐心等待!

> 缺省的二进制文件包含受 Apache License 2 (APL2) 许可保护的核心开源功能和受  CockroachDB Community License (CCL) 许可保护的企业功能。可以使用 `make buildoss` 构建一个排除企业功能的纯开源 (APL2) 版本。更多细节见[博客](https://www.cockroachlabs.com/blog/how-were-building-a-business-to-last/)。

5、安装 `cockroach` 二进制文件至 `/usr/local/bin` 目录中，这样就可以在任何目录中运行 [cockroach 命令](https://www.cockroachlabs.com/docs/stable/cockroach-commands.html)：

```sh
$ make install
```

如果在安装过程中遇到权限错误，请在命令前加上 `sudo`。

你也可以直接在编译的路径运行 `cockroach` 二进制文件，假设二进制文件在 PATH 路径中：`./src/github.com/cockroachdb/cockroach/cockroach`。

6、确认 CockroachDB 是否安装成功：

```sh
$ cockroach version
```

### 下一步

[快速开始](start-a-local-cluster.md)搭建一个单节点或多节点的本地集群并通过内建的 SQL 客户端进行访问。

### 使用 Docker 安装

> **警告**<br>在 Docker 中运行像 CockroachDB 这一类有状态的应用程序将会更加复杂且更易出错，除非你对 Docker 具有丰富的经验，因此我们推荐使用其它的安装和部署方式。

1、安装 [Docker Linux](https://docs.docker.com/engine/installation/linux/ubuntulinux/) 版，安装时请仔细检查所有的必要条件。

2、确认 Docker 的后台程序正在运行：

```sh
$ sudo docker version
```

如果没有见到列出的服务程序，启动 Docker 后台程序。

> 在 Linux 上，Docker 需要 sudo 权限）。

3、从 [Docker 镜像库](https://hub.docker.com/r/cockroachdb/cockroach/)下载官方 CockroachDB 镜像：

```sh
$ sudo docker pull cockroachdb/cockroach:v1.0.4
```

4、确认 CockroachDB 可运行：

```sh
$ sudo docker run --rm cockroachdb/cockroach:v1.0.4 version
```

### 下一步

在一台主机上，跨跨多个 Docker 容器[快速启动](start-a-local-cluster-in-docker.md)一个多节点集群，使用 Docker 卷持久化节点数据，或者使用[编排](orchestration.md)工具在容器中探索运行一个物理上分布的集群。

## Windows

### 使用二进制文件安装

> **警告**<br>原生的 Windows 版 CockroachDB 需要 Windows 8 或更高版本，并且尚未经过 Cockroach 实验室的大量测试。这个预先构建的二进制版本为方便本地开发和实验而提供，强烈不鼓励在 Windows 上部署 CockroachDB 的生产环境。

1、下载并提取 [CockroachDB v1.0.4 版本的 Windows 二进制包](https://binaries.cockroachdb.com/cockroach-v1.0.4.windows-6.2-amd64.zip)。

2、打开该二进制文件所在的目录，确认该 CockroachDB 包完整可运行：

```sh
PS C:\cockroach-v1.0.4.windows-6.2-amd64> .\cockroach.exe version
```
### 下一步

[快速开始](start-a-local-cluster.md)搭建一个单节点或多节点的本地集群并通过内建的 SQL 客户端进行访问。

### 使用 Docker 安装

> **警告**<br>在 Docker 中运行像 CockroachDB 这一类有状态的应用程序将会更加复杂且更易出错，除非你对 Docker 具有丰富的经验，因此我们推荐使用其它的安装和部署方式。

1、安装 [Windows 版 Docker](https://docs.docker.com/docker-for-windows/install/) 。
> Windows 版 Docker 要求 64 位 Windows 10 Pro 和 Microsoft Hyper-V。请阅读[官方文档](https://docs.docker.com/docker-for-windows/install/#what-to-know-before-you-install)查看更多细节。如果你的操作系统不满足以上要求，可以尝试使用 [Docker 工具箱](https://docs.docker.com/toolbox/overview/)。

2、打开命令提示符并确认 Docker 的后台程序正在运行：

```sh
PS C:\Users\username> docker version
```

如果没有见到列出的服务程序，请运行 Docker 后台程序。

3、[共享你的本地硬盘](https://docs.docker.com/docker-for-windows/#/shared-drives)。这样，当 Docker 容器被停止或被删除时，可以挂载本地目录作为保留节点数据的数据卷。

4、从 [Docker 镜像库](https://hub.docker.com/r/cockroachdb/cockroach/)下载官方 CockroachDB 镜像：

```sh
PS C:\Users\username> docker pull cockroachdb/cockroach:v1.0.4
```

5、确认 CockroachDB 可运行：

```sh
PS C:\Users\username> docker run --rm cockroachdb/cockroach:v1.0.4 version
```

### 下一步

在一台主机上，跨跨多个 Docker 容器[快速启动](start-a-local-cluster-in-docker.md)一个多节点集群，使用 Docker 卷持久化节点数据，或者使用[编排](orchestration.md)工具在容器中探索运行一个物理上分布的集群。

> **注意**<br>CockroachDB 集群中的每一个节点周期性地向 Cockroach 实验室匿名报告使用详情。使用详情的说明和怎样选择不参加报告使用详情请参看[诊断报告](https://www.cockroachlabs.com/docs/stable/diagnostics-reporting.html)。
