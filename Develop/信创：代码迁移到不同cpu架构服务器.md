## 信创：代码迁移到不同cpu架构服务器

### k8s pod

k8s 通过 pause容器实现了同一个pod中共享存储和网络命名空间，而且还屏蔽了底层运行时容器共享方案的差异行。

docker和Kata容器的共享肯定不一样。

### 问题

同一份代码，运行在不通架构CPU服务器上需要编译适配么？

#### 跨平台软件迁移

软件迁移是指将某个可运行的程序，由原来的环境迁移到另一个环境，并重新运行。改变的环境可能是处理器架构、操作系统、软件运行环境等。总的来说，软件移植是个“脏活”，需要开发者修改源码、编译、再修改、再编译，费时费力。

#### x86_64 and arm64 and aarch64

64 代表支持 64bit 指令集, V8 之后开始支持的, 目前 arm64 只有 V8, 但是之后出了 V9, 那也是 arm64.

AArch64是ARMv8 架构的一种执行状态。

ARM64是全新指令集（arm设计）；x64是x86_64（Intel设计）和AMD64（AMD设计）的简称。

### 答案

分开发语言

按照翻译方式的不同，高级语言通常可以分为两类：一类是编译翻译，一类是解释翻译，分别对应着编译型语言和解释型语言。

#### 编译型语言

典型的如C、C++语言，都属于编译型语言，源代码到执行的过程概括如图1-1所示。C/C++编译好的程序是机器指令，由操作系统加载到存储器（一般为内存）后由CPU直接执行。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/编译语言执行过程.png)

基于编译型语言开发的应用程序，例如C/C++语言应用程序，其编译后得到可执行程序，可执行程序执行时依赖的指令是CPU架构相关的。因此，基于x86架构编译的C/C++语言应用程序，无法直接在TaiShan服务器（华为aarch64）运行，需要进行移植编译。

#### 解释型语言

典型的如Java、Python语言，都属于解释型语言，源代码到执行的过程概括如图所示。Java/Python编译好的程序是平台无关的字节码，由虚拟机解释执行，虚拟机完成平台差异的屏蔽。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/解释型语言执行过程.png)

基于解释型语言开发的应用程序，是CPU架构不相关的，例如Java、Python，将这类应用程序移植到TaiShan服务器，无需修改和重新编译，按照与x86一致的方式部署和运行应用程序即可。

Java应用程序jar包内，可能包含基于C/C++语言开发的so库文件（**SO 库文件或可执行文件**），这类so库需要移植编译，移植编译so库遇到的问题，使用编译得到的so库重新打包jar包。

#### 汇编语言 on diff cpu

这类应用本身占比较少，重新编译不行，需要重新写一遍，如果不能重写，在指令集翻译工具研发推出后也可以解决这个问题。

CPU指令集都变了，所以需要重新写。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/x86-and-aarch.png)

相比x86 架构，华为鲲鹏处理器的优势较为明显：

- 多核，性能提升 20%，云应用支持度更好，更灵活；
- 支持 8 个 DDR 通道，传统 CPU 仅 6 个，吞吐率提升 25%；
- SOC 芯片，一颗芯片四合一，包含 CPU、南桥、网卡和 SAS 控制器，效能提升 30%；
- 集成压缩、加密、重删等硬件加速引擎的处理器，大大提升应用的性能，释放更多 CPU 算力。

#### demo

如下，该python镜像进行某些骚操作，导致arm 64服务器上镜像无法运行。根本原因还是镜像制作时，是在x86或arm服务器上，镜像编译时没加参数，导致镜像无法跨cpu架构平台使用。

Docker in fact detects the Apple M1 Pro platform as `linux/arm64/v8`

```bash
$ docker run --rm -it --entrypoint=python k8s_test:v1
standard_init_linux.go:228: exec user process caused: exec format error
$ docker run --rm -it --entrypoint=sh k8s_test:v1
standard_init_linux.go:228: exec user process caused: exec format error

## Docker in fact detects the Apple M1 Pro platform as linux/arm64/v8
# Build for ARM64 (default)
docker build -t <image-name>:<version>-arm64 .

# Build for ARM64 
docker build --platform=linux/arm64 -t <image-name>:<version>-arm64 .

# Build for AMD64
docker build --platform=linux/amd64 -t <image-name>:<version>-amd64 .
```



### 建议

建议阅读下《程序员的自我修养》

### 引用

1. https://picture.iczhiku.com/weixin/message1579689833847.html
2. https://bbs.huaweicloud.com/blogs/176363
3. 