### Golang QueryStart -- mihoyo

### T型人才

只有扩大知识范围，才能更好的在自己的知识领域深入。

Remote Job

* 英文读写能力
* github 项目
* 数据结构/算法基本功

我的目标，花大力气学习，出小力气工作。

### gopher

* return 

  为保证return的原子操作，尽量在函数声明时，函数声明return value ，function body中直接运行 return

* 

### Camel Case and Snake Case

Raw： `user login count`

Camel Case: `userLoginCount`

Snake Case: `user_login_count`

Snake Case(ALL Caps): `USER_LOGIN_COUNT`

There is no best method of combining words. The main thing is to be consistent with the convention used, and, if you’re in a team, to come to an agreement on the convention together.

thanksForReading!

ThanksForReading!

THANKS_FOR_READING_!

thanks-for-reading-!

Cheers!



### project-layout

官网yyds：https://golang.org/doc/gopath_code#ImportPaths

https://makeoptim.com/golang/standards/project-layout

Go code must be kept inside a workspace. A workspace is a directory hierarchy with three directories at its root:

* src contains Go source files organized into packages (one package per directory)
* pkg contains package objects
* bin contains executable commands

### package names：package unit

同一个package下的任何数据都可以被直接引用，包括小写的Struct和func。

The first statement in a Go source file must be

```go
package name
```

where `name` is the package's default name for imports. (All files in a package must use the same `name`.)

同一个目录下，默认就是同一个package。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/golang-package-dir.jpg)

注意，注意，注意：同一个package(dir)下不同文件中的function，可以直接相互引用.

```python
// 同级目录下的deploy.py
from .deploy import (
    create_deploy_job,
    DeployJobPostParams,
    update_deploy_job_by_id,
    DeployJobUpdateInfo,
    get_deploy_job_by_id,
)

// golang
// 可以直接引用同一个package(dir)下不同文件的方法，比如：runServer()
func startServer() {
	runServer(NewJaegerTracer(HttpServerName))
}

```

和python不一样，Py是基于文件的，引用不同文件中的方法必须要显示导入该文件。

> python在同一个模块内变量是可以直接使用的，但是无法使用别的模块的变量。此时可以建一个专门的全局变量管理模块来实现跨文件（.py）的全局变量。
>
> 模块(module)其实就是py文件

Golang一个package下面的不同文件的全局变量都可以直接引用.

```go
// http_run.go
package examples
var (
	serverPort = flag.String("port", "8000", "server port")
)

// http_server.go
package examples
func redirect(w http.ResponseWriter, r *http.Request) {
	http.Redirect(w, r,
		fmt.Sprintf("http://localhost:%s/gettime", *serverPort), 301)
}
```

#### Error：one directory, multi package ？

You cannot have two packages per directory.

A set of files sharing the same PackageName form the implementation of a package. An implementation may require that all source files for a package inhabit the same directory.

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/golang-package-dir-and-packagename.jpg)

一个文件夹下只能有一个package原则：

- import后面的其实是GOPATH开始的相对目录路径，包括最后一段。但由于一个目录下只能有一个package，所以import一个路径就等于是import了这个路径下的包。

- 注意，这里指的是“直接包含”的go文件。如果有子目录，那么子目录的父目录是完全两个包。

- - 比如你实现了一个计算器package，名叫calc，位于calc目录下；但又想给别人一个使用范例，于是在calc下可以建个example子目录（calc/example/），这个子目录里有个example.go（calc/example/example.go）。此时，example.go可以是main包，里面还可以有个main函数。

一个package的文件不能在多个文件夹下。

- 如果多个文件夹下有重名的package，它们其实是彼此无关的package。
- 如果一个go文件需要同时使用不同目录下的同名package，需要在import这些目录时为每个目录指定一个package的别名。

一个dir(package)下面如果有多个go files of different package就会报错--

```bash
$ cat ./* |grep ^package
package lib
package main
$ pwd
/go/src/pakagedemo/lib
$ ls
languages.go  main.go
# 提示信息
$ go install
found packages lib (languages.go) and main (main.go) in /go/src/pakagedemo/lib
# 执行go install 失败
$ echo $?
1

```

但是看etcd的源代码目录，如下一个mod目录下两个go file就是不同的package name

```bash
$ tree /go/src/etcd/tools/mod/
/go/src/etcd/tools/mod/
|-- go.mod
|-- go.sum
|-- install_all.sh
|-- libs.go
`-- tools.go

0 directories, 5 files

## 同一个package下面，两个go files的 package name is different.
$ cat /go/src/etcd/tools/mod/tools.go |grep -v "//"|head -n5
package tools
$ cat /go/src/etcd/tools/mod/libs.go |grep -v "//"|head -n5
package libs
root@974cf1a8c342:/go/src/pakagedemo/cmd#

```

### package main: executable

Executable commands must always use `package main`.

There is no requirement that package names be unique across all packages linked into a single binary, only that the import paths (their full file names) be unique.

`package main`只是一个标识，当运行`go install`时，该package下的文件生成一个executable file而不是一个以`.a`结尾的staic mode。

当`package main`出现时，才表示这些代码是executable program 而不是 shared library，这时才能在当前目录下执行`go build and go install`.

* go build : 当前目录上生成 executale program
* go install：在`GOPATH/bin`生成 executable program， 而不是生成`.a` file.

PS:  **executable file name is project name.**

When you build reusable pieces of code, you will develop a package as a shared library. But when you develop executable programs, you will use the package “main” for making the package as an executable program. The package “main” tells the Go compiler that the package should compile as an executable program instead of a shared library. The main function in the package “main” will be the entry point of our executable program. When you build shared libraries, you will not have any main package and main function in the package.

Here’s a sample executable program that uses package main in which the function main is the entry point.

```bash
root@974cf1a8c342:/go/src# pwd
/go/src
root@974cf1a8c342:/go/src# ls
aaa  etcd  first  test_install
root@974cf1a8c342:/go/src# tree first/
first/
|-- go.mod
`-- m1
    |-- f1.go
    `-- main.go

1 directory, 3 files

## all Go files are in package main.
root@974cf1a8c342:/go/src# cat first/m1/main.go
package main

import "fmt"

func main(){
    Test()
    fmt.Println("test")
}
root@974cf1a8c342:/go/src# cat first/m1/f1.go
package main

import "fmt"

func Test(){
   fmt.Println("test func")
}


root@974cf1a8c342:/go/src# cd first/
root@974cf1a8c342:/go/src/first# ls
go.mod 
root@974cf1a8c342:/go/src/first# go install
no Go files in /go/src/first
## 提示 package 必须在 GOROOT下面
root@974cf1a8c342:/go/src/first# go install m1
package m1 is not in GOROOT (/usr/local/go/src/m1)
root@974cf1a8c342:/go/src/first# go build m1
package m1 is not in GOROOT (/usr/local/go/src/m1)

## 在当前目录直接 go build 和 go install时，是没有问题的
root@974cf1a8c342:/go/src/first# cd m1/
root@974cf1a8c342:/go/src/first/m1# ls
f1.go  main.go
root@974cf1a8c342:/go/src/first/m1# go build
root@974cf1a8c342:/go/src/first/m1# ls
f1.go  m1  main.go
root@974cf1a8c342:/go/src/first/m1# ./m1
test func
test
root@974cf1a8c342:/go/src/first/m1# go install
root@974cf1a8c342:/go/src/first/m1# ls
f1.go  main.go
## executable file name is project name.
root@974cf1a8c342:/go/src/first/m1# ls /go/bin/
m1
root@974cf1a8c342:/go/src/first/m1# /go/bin/m1
test func
test

```

#### ELF

ELF (Executable and Linkable Format)是一种为可执行文件，目标文件，共享链接库和内核转储(core dumps)准备的标准文件格式。 Linux和很多类Unix操作系统都使用这个格式。

每个可执行文件中 编译进去了一个runtime库，这个runtime 包含了GC内存管理、Groutine调度、系统调用等等功能。并且由于runtime的存在，一个Go编译的可执行文件，其甚至不依赖 libc库。Go编译的可执行文件可以认为由这几个部分组成：

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/go-elf.png)

### Packages library: static mode

Let’s create a sample program for demonstrating a package. Create a source file `languages.go`  for the package `lib`  at the location `/go/src/packagedemo/` in the GOPATH. Since the `languages.go` belongs to the folder `lib`, the package name will be named `lib`.  All files created inside the lib folder belongs to lib package.

```bash
$ cat languages.go
package lib
var languages map[string]string
func init(){
	languages= make(map[string]string)
	languages["cs"] = "C #"
	languages["js"] = "JavaScript"
	languages["rb"] = "Ruby"
	languages["go"] = "Golang"
}
func Get(key string) (string){
	return languages[key]
}
func Add(key,value string){
	 languages[key]=value
}
func GetAll() (map[string]string){
	return languages
}

## code dir
$ pwd
/go/src/pakagedemo/lib
$ go install
## generate shared library
$ ls /go/pkg/linux_amd64/
pakagedemo
$ ls /go/pkg/linux_amd64/pakagedemo/
lib.a

```

The go install command will build the package `lib`  which will be available at the pkg subdirectory of GOPATH.

Let’s create a `main.go`  for making an executable program where we want to reuse the code specified in the package `lib`.

```bash
$ pwd
/go/src/pakagedemo/cmd

## pakagedemo project dir
$ tree ../
../
|-- cmd
|   `-- main.go   # command source
|-- go.mod
`-- lib
    `-- languages.go # package source

2 directories, 3 files


$ cat /go/src/pakagedemo/cmd/main.go
package main

import (
	"fmt"
	// 引用 lib.a
 	"packagedemo/lib"
)

func main() {
 lib.Add("dr","Dart")
 fmt.Println(lib.Get("dr"))
 languages:=lib.GetAll()
 for _, v := range languages {
  fmt.Println(v)
 }
}

$ pwd
/go/src/pakagedemo/cmd
$ go install
$ ls /go/bin/
cmd 
$ /go/bin/cmd
Dart
Ruby
Golang
Dart
C #
JavaScript

```

In the above program, we import the lib package and call the exported methods provided by the lib package.

#### non-main package

The `package main` tells the Go compiler that the package should compile as an executable program instead of a shared library. 

non-main package 它是一个 shared library，当然不能被执行了啊--

**go run: cannot run non-main package**

没有main 的package 不能被运行

```bash
$ go run worker.go
go run: cannot run non-main package
$ vim worker.go
$ cat worker.go
package main

func main()  {
  foo("dodo")
}
$ go run worker.go
# command-line-arguments
./worker.go:5:3: undefined: foo


```

end

### `()` 优先级

由于`()` 改变了优先级，所以下面的报错类型是不一样的。即原来是`float32 > * `，现在`(*float)`绑在一块了。

```go
// i type is *int
num := 5
i = &num

// Cannot convert expression of type '*int' to type '*float32'
f = (*float32)(i)

// Cannot convert expression of type '*int' to type 'float32'
f = *floate32(i)
```

end

### Golang源码编译

#### 静态编译

静态编译就是编译器在编译可执行文件的时候，将可执行文件需要调用的对应动态链接库(.so)中的部分提取出来，链接到可执行文件中去，使可执行文件在运行的时候不依赖于动态链接库。所以其优缺点与动态编译的可执行文件正好互补。

#### 动态编译

动态编译的可执行文件需要附带一个的动态链接库，在执行时，需要调用其对应动态链接库中的命令。所以其优点一方面是缩小了执行文件本身的体积，另一方面是加快了编译速度，节省了系统资源。缺点一是哪怕是很简单的程序，只用到了链接库中的一两条命令，也需要附带一个相对庞大的链接库；二是如果其他计算机上没有安装对应的运行库，则用动态编译的可执行文件就不能运行。

#### ldd and go build

如下这个 C程序编译出来后的二进制文件会需要这三个库文件，因此如果我们将它做移植时会因为缺失动态库文件而造成无法运行。

```bash
# ldd - print shared object dependencies
$ gcc -o hc hello.c
$ ldd ./hc
linux-vdso.so.1 (0x00007ffe49571000)
libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f805817f000)
/lib64/ld-linux-x86-64.so.2 (0x00007f8058772000)
$  du -sh hc
12K     hc

```

Go 编译出来的二进制文件是没有需要依赖的共享库的。

```bash
$ go build  -o hello hello.go
$ ldd hello
not a dynamic executable
$ du -sh hello
1.7M    hello

```

Go 编译出来的二进制文件比 C 语言编译出来的文件会大的多，这其实就是因为 Go 在编译时会将依赖库一起编译进二进制文件的缘故。

Go默认情况下是采取的静态编译。但是，**不是所有情况下，Go都不会依赖外部动态共享库**！

#### cgo

如下，使用golang的net库，会有外部依赖

```go
package main
import (
    "fmt"
    "net/http"
)
func main() {
    http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintf(w, "Hello, you've requested: %s\n", r.URL.Path)
    })
    http.ListenAndServe(":80", nil)
}


```

验证编译结果

```bash
$ go build  -o server server.go
$ ldd server
linux-vdso.so.1 (0x00007fff8e9ae000)
libpthread.so.0 => /lib/x86_64-linux-gnu/libpthread.so.0 (0x00007fcf6189e000)
libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007fcf616d9000)
/lib64/ld-linux-x86-64.so.2 (0x00007fcf618c6000)
$ du -sh server
5.9M    server

```

默认情况下 Go 语言是使用 Cgo 对 Go 程序进行构建的。当使用 Cgo 进行构建时，如果我们使用的包里用着使用 C 语言支持的代码，那么最终编译的可执行文件都是要有外部依赖的。

自C语言出现，已经累积了无数功能强大、性能卓越的C语言代码库，可以说难以替代；为了方便快捷的使用这些C语言库，Go语言提供 Cgo，可以用以在 Go 语言中调用 C语言库。

#### golang纯静态编译

默认情况下，`CGO_ENABLED=1`，代表着启用 Cgo 进行编译。即默认开始cgo，允许你在Go代码中调用C代码，Go的pre-compiled标准库的.a文件也是在这种情况下编译出来的。我们如果将 `CGO_ENABLED` 置为 0，那么就可以关闭 Cgo：

```bash
$ du -sh server
5.9M    server
$ CGO_ENABLED=0 go build -o server server.go
$ ldd server
not a dynamic executable
$ du -sh server
5.8M    server

```

#### externel linker

在CGO_ENABLED=1这个默认值的情况下，是否可以实现纯静态连接呢？答案是可以。即通过如下：

cmd/link 有两种工作模式：`internal linking`和`external linking`。

- `internal linking`：若用户代码中仅仅使用了 net、os/user 等几个标准库中的依赖 cgo 的包时，cmd/link 默认使用 `internal linking`，而无需启动外部external linker(如:gcc、clang等)，不过由于cmd/link功能有限，仅仅是将.o和pre-compiled的标准库的.a写到最终二进制文件中。
- `external linking`：将所有生成的.o都打到一个`.o`文件中，再将其交给外部的链接器，比如 gcc 或 clang 去做最终链接处理。

如果我们在写入参数 `-ldflags '-linkmode "external" -extldflags "-static"'`，那么 gcc/clang 将会去做静态链接，将`.o`中`undefined`的符号都替换为真正的代码，从而达到纯静态编译的目的：

```bash
$  go build -o server -ldflags '-linkmode "external" -extldflags "-static"' server.go
# command-line-arguments
/usr/bin/ld: /tmp/go-link-3984843045/000004.o: in function `_cgo_3c1cec0c9a4e_C2func_getaddrinfo':
/tmp/go-build/cgo-gcc-prolog:58: warning: Using 'getaddrinfo' in statically linked applications requires at runtime the shared libraries from the glibc version used for linking
$ ls server
server
$ du -sh server
6.9M    server

```



#### CGO_ENABLED=0 go install

 `go get XXX` binary being dynamically compiled.

When you download the binary with `go install`, by default it downloads with `CGO_ENABLED=1` (unless overriden), requiring most of the runtime libraries (including glibc) to be loaded at runtime. This might not work well in some container images, where the libraries are not present (e.g. images built from scratch/distro-less static images).

So to avoid the dependency with the container image's dependencies, always download a statically compiled one by setting the above flag to 0. Use the downloaded binary in your docker context

```bash
CGO_ENABLED=0 go install xxx
```



### go tool: 使用库文件编译

go build 和 go install 都需要使用源码来进行编译。但是有时候我们只有.a或者.so文件。并不能获取到第三方库的源码，这时就需要静态链接库编译的技巧

 一个现代编译器的主要工作流程如下： 源代码(source code) → 预处理器(preprocessor) → 编译器 (compiler) → 汇编程序(assembler) → 目标代码 (object code) → 链接器 (Linker) → 可执行文件 (executables)

代码编译过程：

首先要把源文件编译成中间代码文件，在Windows下也就是 .obj 文件，UNIX下是 .o 文件，即 Object File，这个动作叫做编译（compile）。然后再把大量的Object File合成执行文件，这个动作叫作链接（link）。

a阿里ilogtail，使用so编译

#### 静态函数库:`.a`

`.a` 是好多个.o合在一起,用于静态连接 ，即static  mode，多个.a可以链接生成可执行文件。

- 特点：实际上是简单的普通目标文件的集合，在程序执行前就加入到目标程序中。
- 优点：可以用以前某些程序兼容；描述简单；允许程序员把程序link起来而不用重新编译代码，也就数不需要外部函数的支持，节省了重新编译代码的时间（该优势目前已不明显）；开发者还可以对源代码保密。
- 这类库的名字一般是libxxx.a.利用静态函数库编译成的文件比较大,因为整个函数库的所有数据都会被整合进目标代码中。
- 缺点：如果静态函数库改变了,那么你的程序必须重新编译。

go install生成静态库时，只能以package为单位。

Arguments must be package paths or package patterns (with "..." wildcards). They must not be standard packages (like fmt), meta-patterns (std, cmd,all), or relative or absolute file paths.

```bash
$ pwd
/D/data_files/go_code/awesomeProject
$ head -n 2 go.mod
module test

## 必须将package directory移动到GOPATH/src
$ mv utils/ $GOPATH/src
$ cd $GOPATH/src/utils/
## 编译test.go生成静态库文件
## 执行go install test.go不会生成utils.a
$ go install
```

生成静态库文件的路径`${GOPATH}/pkg/${OS_CPU/package.a}`

```bash
$ ls /D/Program\ Files/Go/pkg/windows_amd64/utils.a
'/D/Program Files/Go/pkg/windows_amd64/utils.a'
```

将`$GOPATH/src/utils/`目录删除，并将`import "test/utils"`改为`import "utils"`，即模拟用户没有源码，只有静态库的情况。如下，使用静态库，代码成功执行。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/golang-tool-compile-link.jpg)

#### 共享态函数库:`.so`

`.o`编译的目标文件

`.so` 是shared object,用于动态连接的,使用时才载入。用于动态连接的,相当于windows下的dll，是Linux中的可执行文件。

- 共享函数库在可执行程序启动的时候加载，所有程序重新运行时都可自动加载共享函数库中的函数。相对于静态函数库,共享函数库在编译的时候 并没有被编译进目标代码中。
- 当程序执行到相关函数时才调用共享函数库里相应的函数,因此共享函数库所产生的可执行文件比较小.**由于共享函数库没有被整合进你的程序**,而是在程序运行时动态地申请并调用,所以程序的运行环境中必须提供相应的库.
- 共享函数库的改变并不影响你的程序,所以共享函数库的升级比较方便.

动态库出现的初衷是对于相同的库，多个进程可以共享同一个，以节省内存和磁盘资源。但是在磁盘和内存已经白菜价的今天，这两个作用已经显得微不足道了，那么除此之外动态库还有哪些存在的价值呢？从库开发角度来说，动态库可以隔离不同动态库之间的关系，减少链接时出现符号冲突的风险。而且对于 windows 等平台，动态库是跨越 VC 和 GCC 不同编译器平台的唯一的可行方式。

```bash
## 生成share object 文件
$ cat hello.c
#include <stdio.h>
void hi() {
    printf("Hello, Chandler!\n");
}
$ gcc -fPIC -shared -o libhello.so hello.c
$ ls
hello.c  libhello.so
$ pwd
/tmp/c

## 编写c header文件, 即告诉调用者可以调用哪些function
$ cat hi.h
void hi();


## 编写go file调用share object

## cgo CFLAGS: -I./  // 这里表示头文件所在的位置
## cgo LDFLAGS: -L/data/golang_code/cgolib/ -lhello  // 这里表示so库所在的位置
$ cat go_so.go
package main

/*
#cgo CFLAGS: -I./
#cgo LDFLAGS: -L/data/golang_code/cgolib/ -lhello
#include "hi.h"
 */
import "C"    // 注意这个地方与上面注释的地方不能有空行，并且不能使用括号如import ("C" "fmt")
import "fmt"

func main(){
    C.hi()
    fmt.Println("Hello c, welcome to go!")
}
roo
$ export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/data/golang_code/cgolib/
## 调用c function成功
$ go run go_so.go
Hello, Chandler!
Hello c, welcome to go!

```

##### library name

share library通常以lib开头，比如`libsum.so`。

第三方调用share object时，不用写`lib`直接写`sum`即可。

共享库文件的命名规则与静态库类似，都有固定的前缀后缀。中间的 xxx 即为库名称。.so 表示这个文件是一个动态库文件。.1 通常表示库的版本号，这样可以实现多个库版本共存的需求。

### init()

每个package中的`init()`在导入时，都会被执行-- 

```go
var WhatIsThe = AnswerToLife()

func AnswerToLife() int {
    return 42
}

func init() {
    WhatIsThe = 0
}

func main() {
    if WhatIsThe == 0 {
        fmt.Println("It's all a lie.")
    }
}
```

`AnswerToLife()` is guaranteed to run before `init()` is called, and `init()` is guaranteed to run before `main()` is called.

Keep in mind that `init()` is always called, regardless if there's main or not, so if you import a package that has an `init` function, it will be executed.

Additionally, you can have multiple `init()` functions per package; they will be executed in the order they show up in the file (after all variables are initialized of course). If they span multiple files, they will be executed in lexical file name order.

精彩的`init()`，再导入时执行，作用将定义的结构体注册到 scheme

```go
// 路径是： example/api/v1
//  guestbook_types.go中, 没有定义和导入SchemeBuilder
package v1

import (
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

// EDIT THIS FILE!  THIS IS SCAFFOLDING FOR YOU TO OWN!
// NOTE: json tags are required.  Any new fields you add must have json tags for the fields to be serialized.

// GuestbookSpec defines the desired state of Guestbook
type GuestbookSpec struct {
	// INSERT ADDITIONAL SPEC FIELDS - desired state of cluster
	// Important: Run "make" to regenerate code after modifying this file

	// Foo is an example field of Guestbook. Edit Guestbook_types.go to remove/update
	Foo string `json:"foo,omitempty"`
	// add
	Name string `json:"name_json,omitempty"`
}

// GuestbookStatus defines the observed state of Guestbook
type GuestbookStatus struct {
	// INSERT ADDITIONAL STATUS FIELD - define observed state of cluster
	// Important: Run "make" to regenerate code after modifying this file
	// add
	Status string `json:"Status"`
}

// +kubebuilder:object:root=true

// Guestbook is the Schema for the guestbooks API
type Guestbook struct {
	metav1.TypeMeta   `json:",inline"`
	metav1.ObjectMeta `json:"metadata,omitempty"`

	Spec   GuestbookSpec   `json:"spec,omitempty"`
	Status GuestbookStatus `json:"status,omitempty"`
}

// +kubebuilder:object:root=true

// GuestbookList contains a list of Guestbook
type GuestbookList struct {
	metav1.TypeMeta `json:",inline"`
	metav1.ListMeta `json:"metadata,omitempty"`
	Items           []Guestbook `json:"items"`
}

func init() {
	SchemeBuilder.Register(&Guestbook{}, &GuestbookList{})
}

// groupversion.go中，定义了SchemeBuilder。
package v1

import (
	"k8s.io/apimachinery/pkg/runtime/schema"
	"sigs.k8s.io/controller-runtime/pkg/scheme"
)

var (
	// GroupVersion is group version used to register these objects
	GroupVersion = schema.GroupVersion{Group: "webapp.dodo", Version: "v1"}

	// SchemeBuilder is used to add go types to the GroupVersionKind scheme
	SchemeBuilder = &scheme.Builder{GroupVersion: GroupVersion}

	// AddToScheme adds the types in this group-version to the given scheme.
	AddToScheme = SchemeBuilder.AddToScheme
)

// 全局的视角
// 即在其他package 引用v1 package 时，调用SchemeBuilder完成注册--
import (
	"context"
	...
	webappv1 "example/api/v1"
)
```

初看时，很迷，但是静下来，发现它真不错--

### 值拷贝

In a function call, the function value and arguments are evaluated in the usual order. After they are evaluated, the parameters of the call are passed by value to the function and the called function begins execution.

官方文档已经明确说明：Go里边函数传参只有值传递一种方式: 值传递。

值拷贝的语义：无论参数做什么修改，都不能作用到源变量身上，保证源变量参数不能被修改。

如下demo：

* slice a变量执行的底层数字arr
* slice a作为argument，传入changSlice funct
* 发送值拷贝形参arrArg被赋值，slice a 和 arrArg是两个不同的内存地址，但是它们的value相同，值复制
* arrArg也指向底层数字arr，所以修改arrArg时slice a也被修改

```go
func main() {
	arr := [5]int{0, 1, 2, 3, 4}
	s := arr[1:]
	changeSlice(s)
	fmt.Println(s)
	fmt.Println(arr)
}

func changeSlice(arrArg []int) {
	for i := range arrArg {
		arrArg[i] = 10
	}
}
```

#### 编译器优化值拷贝

在reflect包中，会涉及到编译器优化值拷贝的工作。

* reflect.TypeOf()语义是func TypeOf(i interface{}) Type {}，参数是个空接口类型，它需要接收一个地址。

  因为iface结构体就两个指针成员变量

* 也不能直接使用&a作为参数，因为这样违背了值传递的语义（不能直接修改原变量）

* 编译器就会创建临时变量copy_of_a，然后在&copy_of_a

* 这样既满足了传参值拷贝的语义，又满足了空接口参数对值的要求

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/golang-value-copy.jpg)

#### 反射修改变量值

使用场景：做orm和一些将json自动解析为结构体的操作。

### 语法糖

#### `...`

```go
// 合并两个slice
s_less := []int{1, 3, 5}
s_more := []int{2, 4, 6}
var s_new []int
s_new = s_new.append(s_new, s_less...)
s_new = s_new.append(s_new, s_more...)
```



### 类型元数据

无论内置类型还行自定义类型，都有全局唯一的元数据信息

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/golang-type-metadata.jpg)



### 内置数据类型

golang 所有的类型数据都有一个pair(type, value) ，这就是反射的实现。

静态语言的反射，Python很简单直接`dir()`

#### int

- 有符号（负号）：int8 int16 int32 int64
- 无符号（负号）：uint8 uint16 uint32 uint64

#### byte and rune

- byte就是unit8的别名
- rune就是int32的别名

#### float

浮点类型的值有float32和float64(没有 float 类型)

#### string：只读

字符串是由字符组成的数组，C 语言中的字符串使用字符数组 `char[]` 表示。数组会占用一片连续的内存空间，而内存空间存储的字节共同组成了字符串，**Go 语言中的字符串只是一个只读的字节数组。**

**Golang中string只是一个只读的字节数组**

string的底层数据结构就是byte数组。

```go
// string is the set of all strings of 8-bit bytes, conventionally but not
// necessarily representing UTF-8-encoded text. A string may be empty, but
// not nil. Values of string type are immutable.
type string string

type stringStruct struct {
    // string 整体替换时即换了一个指针指向地址
    str unsafe.Pointer
    len int
}

// Cannot use '"a"' (type string) as type uint8
package main

import "fmt"

func main() {
	//var b [3]int
	b := [3]byte{1,3,5}
	s := "test"
    // error
	// s[1] = "a"
    // 但是s 可以整体替换
    s = "a"
	b[1] = 33
	fmt.Println(s,b)
}
```

string与[]byte在底层结构上是非常的相近（后者的底层表达仅多了一个cap属性，因此它们在内存布局上是可对齐的），这也就是为何builtin中内置函数copy会有一种特殊情况`copy(dst []byte, src string) int`的原因了。

Java、Python 以及很多编程语言的字符串也都是不可变的，这种不可变的特性可以保证不会引用到意外发生改变的值，Go 语言的字符串可以作为哈希的键，所以如果哈希的键是可变的，不仅会增加哈希实现的复杂度，还可能会影响哈希的比较。

只读只意味着字符串会分配到只读的内存空间，但是 Go 语言只是不支持直接修改 `string` 类型变量的内存空间，但仍然可以通过在 `string` 和 `[]byte` 类型之间反复转换实现修改这一目的：

1. 先将这段内存拷贝到堆或者栈上；
2. 将变量的类型转换成 `[]byte` 后并修改字节数据；
3. 将修改后的字节数组转换回 `string`；

应用场景：你不知道string的数据内容，但是只知道要修改第N个字符。

```go
package main
import (
	"fmt"
)

func main(){
	str := "test"
	var b []byte
	fmt.Printf("string: %s, address: %p \n", str, &str )
	b = []byte(str)
	fmt.Printf("turn string to []byte: %v []byte addrss: %p\n", b, b)
	fmt.Println("edit byte array: b[0] = 65")
	b[0] = 65
	fmt.Println("turn new []byte to string:", string(b))
	fmt.Printf("str address: %p\n", &str)
}

// output
string: test, address: 0xc0000481e0 
turn string to []byte: [116 101 115 116] []byte addrss: 0xc000060098
edit byte array: b[0] = 65
turn new []byte to string: Aest
str address: 0xc0000481e0
```

##### string-to-unit8:unsafe

通过unsafe和reflect包，可以实现另外一种转换方式，称为强转换，可以直接将string 底层结构题中执行数组的指针。

```go
package main

import (
	"reflect"
	"unsafe"
)


// string 底层 结构体
// src/runtime/string.go
type stringStruct struct {
    // string 整体替换时即换了一个指针指向地址
    str unsafe.Pointer
    len int
}

//
func String2Bytes(s string) []byte {
    // 反射
    sh := (*reflect.StringHeader)(unsafe.Pointer(&s))
    bh := reflect.SliceHeader{
        Data: sh.Data,
        Len:  sh.Len,
        Cap:  sh.Len,
    }
    return *(*[]byte)(unsafe.Pointer(&bh))
}

func Bytes2String(b []byte) string {
    return *(*string)(unsafe.Pointer(&b))
}
```

将string的底层数组地址赋值给slice结构体后，仍然不能修改这个`slice.Data`。因为string的内存地址空间是在只读内存空间的。

**使用unsafe包将slice指向原来的string内存地址，这样即便转换了类型仍然无法修改string只读内存的内容。**

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/golang-string-addr.jpg)

如下所示，类型转换成功了但是还是不能修改string执行的只读内存

```go
func main()  {
	s := "test"
	b := String2Bytes(s)
	var test interface{}
	test = b

	s = Bytes2String(b)
	fmt.Println(s)
	_, ok := test.([]byte)
	if ok{
		fmt.Println("turn string to []byte, success")
	}
	b[1] = 65
}
// output
test
turn string to []byte, success
unexpected fault address 0x565342
fatal error: fault
[signal 0xc0000005 code=0x1 addr=0x565342 pc=0x54c661]
```



#### array:大小固定

因为array空间申请后大小固定不变，所以array初始化时必须指定size，纵容是使用`...`语法糖也要在编译阶段计算出array size的。

初始化:

```go
var b [8]int
b := [3]int{1,3,5}
// 使用…即表示数组元素的个数是根据元素个数自动推导。
func main()  {
	arrayTest := [6]int{}
	arrayTestTwo := [...]int{1, 3,4}
	arrayTestThree := [...]int{0:2, 8:9}
	fmt.Println(arrayTest)
	fmt.Println(arrayTestTwo)
	fmt.Println(arrayTestThree)
}
// output
[0 0 0 0 0 0]
[1 3 4]
[2 0 0 0 0 0 0 0 9]
```

Array初始化不能使用make。

数组设计之初是在形式上依赖内存分配而成的，所以必须在使用前预先请求空间。这使得数组有以下特性：

1. 请求空间以后**大小固定**，不能再改变（数据溢出问题）；
2. 在内存中有**空间连续性**的表现，中间不会存在其他程序需要调用的数据，为此数组的专用内存空间；
3. 在旧式编程语言中（如有中阶语言之称的C），程序不会对数组的操作做下界判断，也就有潜在的越界操作的风险（比如会把数据写在运行中程序需要调用的核心部分的内存上）。

过程式编程语言最常见的特征之一就是数组的概念。数组看起来很简单，但是在编程语言中要实现数组需要解决很多问题，例如：

- 数组长度是固定还是可变？
- 数组长度是否属于数组类型的一部分，比如作为一个字段存在。还是通过计算得到？
- 多维数组又是怎么样的？
- 空数组是否有意义？

这些问题的答案会决定数组是否仅仅是语言的一个特征或者是语言设计中的一个核心部分。



#### slice:指向底层数组

Construction

```go
var s []int                   // a nil slice
s1 := []string{"foo", "bar"}
s2 := make([]int, 2)          // same as []int{0, 0}
s3 := make([]int, 2, 4)       // same as new([4]int)[:2]
fmt.Println(len(s3), cap(s3)) // 2 4
```

- The default **zero value** of a slice is `nil`. The functions `len`, `cap` and `append` all regard `nil` as an empty slice with 0 capacity.
- You create a slice either by a **slice literal** or a call to the `make`function, which takes the **length** and an optional **capacity** as arguments.
- The built-in `len`and `cap` functions retrieve the length and capacity.

make是Go的内置函数，专门用于分配和初始化指定大小的slice、map、chan类型，它返回的是一个type。而new则不同，new返回的是一个*type,也就是一个指针类型，指向type的零值。

在使用make初始化slice的时候，其第二个参数是slice的长度length（必填，可为0），第三个参数是容量capacity（选填），new的话只有一个参数。

比如下面这些写法，前面2种大部分情况下是等价的，使用起来并没有多大区别，但是第三种new的写法就稍微有点区别，因为它的返回值是指针。

不错，https://www.bilibili.com/video/BV1CV411d7W8?from=search&seid=5779708666083454939

```go
// 不可直接进行下标操作
// func new(Type) *Type
// new 的作用是根据传入的类型分配一片内存空间并返回指向这片内存空间的指针
li := new([]int)
li[0] = 5
invalid operation: li[0] (type *[]int does not support indexing) 	

// 可以直接进行下标操作
// func make(t Type, size ...IntegerType) Type
// make()
package main

import "fmt"
func main()  {
	li := make([]string, 5, 5)
	li[0] = "Robin"
	li[1] = "hi"
    // 传统C写法： for key=0;key<len(li);key++{}
	for key, value := range li{
		fmt.Println(key, value)
	}
}
// output
0 Robin
1 hi
2 
3 
4 
```

区别slice 和 array：slice 复制是指针的拷贝，array是值，因为array申请好后长度不可变。

Golang都是值拷贝。

```go
// slice结构体data字段存了指向的内存地址
// 直接创建slice变量时，系统默认给你创建了。
// slice指向的data内存地址，不够时会字段扩容
// so,slice 有bug
// slice_row  
// slice_s1 = slice_row
// extend slice_row to slice data memory address change
// slice_s2 = slice_row
// Now, slice_s1 不等于 slice_s2
type SliceHeader struct {
	Data uintptr
	Len  int
	Cap  int
}
```

##### slice值传递

Go的slice类型中包含了一个array指针以及len和cap两个int类型的成员。

Go中的参数传递实际都是值传递，将slice作为参数传递时，函数中会创建一个slice参数的副本，这个副本同样也包含array,len,cap这三个成员。

副本中的array指针与原slice指向同一个地址，所以当修改副本slice的元素时，原slice的元素值也会被修改。但是如果修改的是副本slice的len和cap时，原slice的len和cap仍保持不变。

```go
func main() {
	s := []int{1,2,3}
	a := [3]int{1,2,3}

	sCopy := s
	aCopy := a

	a[2] = 4
	s[2] = 4
	fmt.Println("source a:", a)
	fmt.Println("aCopy:", aCopy)
	fmt.Println("source s:", s)
	fmt.Println("sCopy:", sCopy)
}
// output
source a: [1 2 4]
aCopy: [1 2 3]
source s: [1 2 4]
sCopy: [1 2 4]
```

如果在操作副本时由于扩容操作导致重新分配了副本slice的array内存地址，那么之后对副本slice的操作则完全无法影响到原slice，包括slice中的元素。

```go
func main() {
	s := []int{1,2,3,4,5}
	sCopy := s
	sCopy = append(sCopy, 9)
	fmt.Println("source s", s, "capability: ", cap(s))
	fmt.Println("sCopy", sCopy, "capability: ", cap(sCopy))
}
// output
source s [1 2 3 4 5] capability:  5
sCopy [1 2 3 4 5 9] capability:  10

```

end

#### `[N]T` and `[]T`

 array ([size]T) to a slice ([]T) by slicing it:

主要哟，这就是array 和 slice的区别--

#### []unit8 and []byte

这是一个东西

```go
// byte is an alias for uint8 and is equivalent to uint8 in all ways. It is
// used, by convention, to distinguish byte values from 8-bit unsigned
// integer values.
type byte = uint8
```

end

#### map： 引用类型

创建map

```go
ageMp := make(map[string]int)
// 指定 map 长度
ageMp := make(map[string]int, 8)

// ageMp 为 nil，不能向其添加元素，会直接panic
var ageMp map[string]int

// 这样可以,类似于先声明在初始化。
var m map[string]interface{}
m = make(map[string]interface{})
```

One of the most useful data structures in computer science is the hash table. Many hash table implementations exist with varying properties, but in general they offer fast lookups, adds, and deletes. Go provides a built-in map type that implements a hash table.

hash table 是计算机数据结构中一个最重要的设计。大部分 hash table 都实现了快速查找、添加、删除的功能。Go 语言内置的 map 实现了上述所有功能。

map 的设计也被称为 “The dictionary problem”，它的任务是设计一种数据结构用来维护一个集合的数据，并且可以同时对集合进行增删查改的操作。最主要的数据结构有两种：`哈希查找表（Hash table）`、`搜索树（Search tree）`。

哈希查找表一般会存在“碰撞”的问题，就是说不同的 key 被哈希到了同一个 bucket。一般有两种应对方法：`链表法`和`开放地址法`。

```go
package main
import (
	"fmt"
)

func main(){
	var test map[string]int
	test["a"] = 1

	fmt.Println(test)
}

// output
panic: assignment to entry in nil map

// 输出test 指针地址
package main
import (
	"fmt"
)

func main(){
	var test map[string]int

	fmt.Printf("test address: %p\n", test)

    if test == nil{
		fmt.Println("test is nil")
	}
    
	test["a"] = 1

	fmt.Println(test)

}
// 真的是 0x0
test address: 0x0
test is nil
panic: assignment to entry in nil map

```

end

##### makemap

```go
// make(map[string]int) 通过汇编查看调用的是 makemap()
func makemap(t *maptype, hint int64, h *hmap, bucket unsafe.Pointer) *hmap {
    // 省略各种条件检查...

    // 找到一个 B，使得 map 的装载因子在正常范围内
    B := uint8(0)
    for ; overLoadFactor(hint, B); B++ {
    }

    // 初始化 hash table
    // 如果 B 等于 0，那么 buckets 就会在赋值的时候再分配
    // 如果长度比较大，分配内存会花费长一点
    buckets := bucket
    var extra *mapextra
    if B != 0 {
        var nextOverflow *bmap
        buckets, nextOverflow = makeBucketArray(t, B)
        if nextOverflow != nil {
            extra = new(mapextra)
            extra.nextOverflow = nextOverflow
        }
    }

    // 初始化 hamp
    if h == nil {
        h = (*hmap)(newobject(t.hmap))
    }
    h.count = 0
    h.B = B
    h.extra = extra
    h.flags = 0
    h.hash0 = fastrand()
    h.buckets = buckets
    h.oldbuckets = nil
    h.nevacuate = 0
    h.noverflow = 0

    return h
}

// 引用传递
package main
import (
	"fmt"
)

func test_map(m_arg map[string]int){
	fmt.Println("func edit m: m[\"Robin\"] = 20 ")
	m_arg["Robin"] = 20

}

func main(){
	m_obj := make(map[string]int)
	m_obj["Robin"] = 18
	fmt.Println(m_obj)
	test_map(m_obj)
	fmt.Println(m_obj)

}

// output
map[Robin:18]
func edit m: m["Robin"] = 20 
map[Robin:20]


```

makemap 和 makeslice 的区别，带来一个不同点：当 map 和 slice 作为函数参数时，在函数参数内部对 map 的操作会影响 map 自身；而对 slice 却不会。因为map是一个`(*hmap)类型数据。

```go
func main() {
	s := []int{1,2,3,4,5}
	testM := make(map[int]int)
	for i:=0;i<5;i++{
		testM[i]=i
	}
	testSlice(s)
	fmt.Println("source slice: ", s)
	testMap(testM)
	fmt.Println("source map: ", testM)
}

func testSlice(s1 []int){
    // 必须使用append()使slice执行的底层数组扩容。
	s1 = append(s1, 99)
	s1 = append(s1, 99)
	fmt.Println("slice in func:", s1)
}

func testMap(m1 map[int]int)  {
	m1[len(m1)]=len(m1)
	m1[len(m1)]=len(m1)
	fmt.Println(m1)
}
//output
slice in func: [1 2 3 4 5 99 99]
source slice:  [1 2 3 4 5]
map[0:0 1:1 2:2 3:3 4:4 5:5 6:6]
source map:  map[0:0 1:1 2:2 3:3 4:4 5:5 6:6]

```

主要原因：一个是指针（`*hmap`），一个是结构体（`slice`）。Go 语言中的函数传参都是值传递，在函数内部，参数会被 copy 到本地。`*hmap`指针 copy 完之后，仍然指向同一个 map，因此函数内部对 map 的操作会影响实参。而 slice 被 copy 后，会成为一个新的 slice，对它进行的操作不会影响到实参

https://segmentfault.com/a/1190000023879178



`map[string]interface{} ` 特殊，map真难用啊 666

全局变量： var Job = make(map[string]interface{})

To create an empty map, use the builtin `make`:`make(map[key-type]val-type)`.

https://www.linkinstar.wiki/2019/06/03/golang/source-code/graphic-golang-map/

end

##### map get

Go 语言中读取 map 有两种语法：带 comma(逗号--) 和 不带 comma。当要查询的 key 不在 map 里，带 comma 的用法会返回一个 bool 型变量提示 key 是否在 map 中；而不带 comma 的语句则会返回一个 value 类型的零值。如果 value 是 int 型就会返回 0，如果 value 是 string 类型，就会返回空字符串。

```go
value := m["name"]
fmt.Printf("value:%s", value)

value, ok := m["name"]
  if ok {
    fmt.Printf("value:%s", value)
  }
```

两种语法对应到底层两个不同的函数，那么在底层是如何定位到key的呢？稍后我们对函数进行源码分析。

```go
func mapaccess1(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer
func mapaccess2(t *maptype, h *hmap, key unsafe.Pointer) (unsafe.Pointer, bool)

// demo
func main() {

    m := make(map[string]int)

    m["k1"] = 7
    m["k2"] = 13

    fmt.Println("map:", m)

    v1 := m["k1"]
    fmt.Println("v1: ", v1)

    fmt.Println("len:", len(m))

    delete(m, "k2")
    fmt.Println("map:", m)
	// 这个retrun pairs 有点意思
    _, prs := m["k2"]
    fmt.Println("prs:", prs)

    n := map[string]int{"foo": 1, "bar": 2}
    fmt.Println("map:", n)
}
// output 
map: map[k1:7 k2:13]
v1:  7
len: 2
map: map[k1:7]
prs: false
map: map[bar:2 foo:1]
```

##### key的定位

key 经过哈希计算后得到哈希值，共 64 个 bit 位（64位机，32位机就不讨论了，现在主流都是64位机），计算它到底要落在哪个桶时，只会用到最后 B 个 bit 位。还记得前面提到过的 B 吗？如果 B = 5，那么桶的数量，也就是 buckets 数组的长度是 2^5 = 32。例如，现在有一个 key 经过哈希函数计算后，得到的哈希结果是：

用最后的 5 个 bit 位，也就是 `01010`，值为 10，也就是 10 号桶。这个操作实际上就是取余操作，但是取余开销太大，所以代码实现上用的位操作代替。

如何定位key:

https://segmentfault.com/a/1190000022118894

##### map[string]interface{}

map的key肯定是固定不可变的，但是有时map的value是不确定的。

```go
func main() {


	m := make(map[string]interface{})
	//data := make([]map[string]string,1)
	var data []map[string]string
	item := make(map[string]string)
	item["age"] = "20"
	item["gender"] = "female"
	data = append(data, item)
	m["name"] = "Robin"
	m["data"] = data
	fmt.Printf("%v\n", m)

	res, _ := json.Marshal(m)
	fmt.Printf("%s\n", res)
}

// output
map[data:[map[age:20 gender:female]] name:Robin]
{"data":[{"age":"20","gender":"female"}],"name":"Robin"}

// 小疑惑，data := make([]map[string]string,1)声明时
2 // len(data)输出为2 --
map[data:[map[] map[age:20 gender:female]] name:Robin]
{"data":[null,{"age":"20","gender":"female"}],"name":"Robin"}
```

end

##### map并发不安全

map不是并发安全的 , 当有多个并发的groutine读写同一个map时 

会出现panic错误:

fatal error: concurrent map writes

```go
package main

import "fmt"

func main(){
	for  {
		test()
	}
}

func test() {
	c := make(chan bool)
	m := make(map[string]string)
	go func() {
		m["1"] = "a" // First conflicting access.
		c <- true
	}()
	m["1"] = "b" // Second conflicting access.
	<-c
	for k, v := range m {
		fmt.Println(k, v)
	}
}
```

解决方法：



##### map应用demo

要求：将N个字符串平均分成三份或者引申将100个主机分散到三个云平台可用区。

```go
func main(){
	s2 := []string{"test", "foo", "dodo", "sunny", "sum", "Robin"}
	res := make(map[string]int)

	for i:=0;i<len(s2);i++{
		res[s2[i]]=i%3
	}

	fmt.Println(res)
	for k,v:=range res{
		if v == 0{
			fmt.Println(k)
		}
	}
}
// output
map[Robin:2 dodo:2 foo:1 sum:1 sunny:0 test:0]
sunny
test
```



#### incomparable type

Currently (Go 1.19), Go doesn't support comparisons (with the `==` and `!=` operators) for values of the following types:

- slice types
- map types
- function types
- any struct type with a field whose type is incomparable and any array type which element type is incomparable.

Above listed types are called incomparable types. All other types are called comparable types. Compilers forbid comparing two values of incomparable types.

相同长度的数组可以比较大小

```go
func main{
    va := [3]int{1,2,3}
	vb := [3]int{1,2,3}
	if va == vb{
		fmt.Println("va == vb")
	}

}
// output
va == vb
```

长度不同的数组无法`==`比较

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/array-length-comparable.png)



#### Sync.map

官方的[faq](https://golang.org/doc/faq#atomic_maps)已经提到内建的`map`不是线程(goroutine)安全的。\

因为map为引用类型，所以即使函数传值调用，参数副本依然指向映射m, 所以多个goroutine并发写同一个映射m， 写过多线程程序的同学都知道，对于共享变量，资源，并发读写会产生竞争的， 故共享资源遭到破坏



--

#### `*unsafe.Pointer()`

Go 的指针是不支持指针运算和转换。

```go
func main(){
	var i *int
	var f *float32
	num := 5
	i = &num
	f = (*float32)(i)
	fmt.Println(f)
}
// output
cannot convert numPointer (type *int) to type *float32
```



在go中，任何类型的指针`*T`都可以转换为unsafe.Pointer类型的指针，它可以存储任何变量的地址。同时，unsafe.Pointer类型的指针也可以转换回普通指针，而且可以不必和之前的类型\*T相同。另外，unsafe.Pointer类型还可以转换为uintptr类型，该类型保存了指针所指向地址的数值，从而可以使我们对地址进行数值计算。以上就是强转换方式的实现依据。

Golang中的指针，与C/C++的指针却又不同，笔者觉得主要表现在下面的两个方面：

- 弱化了指针的操作，在Golang中，指针的作用仅是操作其指向的对象，不能进行类似于C/C++的指针运算，例如指针相减、指针移动等。
- 指针类型不能进行转换，如`*int`不能转换为`*int32`。

上述的两个限定主要是为了简化指针的使用，减少指针使用过程中出错的机率，提高代码的Robust（耐操性）。

 `unsafe.Pointer`它表示任意类型且可寻址的指针值，可以在不同的指针类型之间进行转换（类似 C 语言的 `void *` 的用途）

其包含四种核心操作：

- 任何类型的指针值都可以转换为 Pointer
- Pointer 可以转换为任何类型的指针值
- uintptr 可以转换为 Pointer
- Pointer 可以转换为 uintptr

典型栗子：

`*int32` <---> `unsafe.Pointer` <--->`*float32`

```go
package main

import (
	"fmt"
	"unsafe"
)

func main(){
	var i *int
	var f *float32
	num := 5
	i = &num
	tempP := unsafe.Pointer(i)
	f = (*float32)(tempP)
	fmt.Printf("i pointer address value: %p\n",i)
	fmt.Printf("f pointer address value: %p\n",f)
	fmt.Println("pointer f value is :",*f)
}

// output
i pointer address value: 0xc000064090
f pointer address value: 0xc000064090
pointer f value is : 7e-45

```

 `unsafe.Pointer` 可以让你的变量在不同的指针类型转来转去，也就是表示为任意可寻址的指针类型。第二是 `uintptr` 常用于与 `unsafe.Pointer` 打配合，用于做指针运算，巧妙地很,没有特殊必要的话。是不建议使用 `unsafe` 标准库，它并不安全。

##### string turn []byte with unsafe.Pointer

就是解决`unsafe.Pointer` 完成指针的转换，再借助`reflect`完成结构体的构建。比如：将`StringHeader.Len`取出来，用来构建`SliceHeader.Len and SliceHeader.Cap`

确实棒，这样自己造轮子的感觉！

```go
package main

import (
	"fmt"
	"reflect"
	"unsafe"
)

func byteToString(b []byte) string{
	return *(*string)(unsafe.Pointer(&b))
}

func stringToBte(s string) []byte{
	sHeader := (* reflect.StringHeader)(unsafe.Pointer(&s))
	bHeader := reflect.SliceHeader{
		Data: sHeader.Data,
		Len: sHeader.Len,
		Cap: sHeader.Len,
	}
	return *(*[]byte)(unsafe.Pointer(&bHeader))
}

func main(){
	// byte slice
	b := []byte{97,98,99,100}
	s := "test"
	fmt.Printf("turn []byte %v to string %v \n",b, byteToString(b))
	fmt.Printf("turn string %v to []byte %v \n",s, stringToBte(s))
}
// output
turn []byte [97 98 99 100] to string abcd 
turn string test to []byte [116 101 115 116] 


```

end

#### Channel

原子性，自带的。channel 中的元素值都具有原子性，是不可被分割的。channel 中的每个元素都只能被某一个 Goroutine 接收--

Go 语言中最常见的、也是经常被人提及的设计模式就是：不要通过共享内存的方式进行通信，而是应该通过通信的方式共享内存。

> Do not communicate by sharing memory; instead, share memory by communicating. A send on a channel happens before the corresponding receive from that channel completes. (Golang Spec)

Go 语言提供了一种不同的并发模型，即通信顺序进程（Communicating sequential processes，CSP）[1](https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-channel/#fn:1)。Goroutine 和 Channel 分别对应 CSP 中的实体和传递信息的媒介，Goroutine 之间会通过 Channel 传递数据。

 Channel 分成了以下三种类型：

- 同步 Channel — 不需要缓冲区，发送方会直接将数据交给（Handoff）接收方；
- 异步 Channel — 基于环形缓存的传统生产者消费者模型；
- `chan struct{}` 类型的异步 Channel — `struct{}` 类型不占用内存空间，不需要实现缓冲区和直接发送（Handoff）的语义；

channel存在`3种状态`：

1. nil，未初始化的状态，只进行了声明，或者手动赋值为`nil`

2. active，正常的channel，可读或者可写

3. closed，已关闭，**千万不要误认为关闭channel后，channel的值是nil**

   已经关闭的channel，读操作读到的是零值`zero-value`，即channel int是`0`，channel string是`""`。

channel可进行`3种操作`：

1. 读

   ```go
   ch <- v    // Send v to channel ch.
   v := <-ch  // Receive from ch, and
              // assign value to v.
   ```

   end

2. 写

3. 关闭

场景：读channel，但不确定channel是否关闭时

原理：读已关闭的channel会得到零值(zero-value)，如果不确定channel，可使用`ok`进行检测。ok的结果和含义：

- `true`：读到数据，并且通道没有关闭。
- `false`：通道关闭，无数据读到。

用法：

```go
if v, ok := <- ch; ok {
    fmt.Println(v)
}
// 零值
package main

import (
	"fmt"
)

func main() {
	news := make(chan int)
	close(news)
	fmt.Println(<- news)
}
```

end

##### 我理解的channel

channel 一个简单可靠的并发模型。

在`1:1`和`1:N`的并发模型下，简单可靠易实现。

一个生产者使用一个Unbufferd Channel or buffered is 1 channel时，它丢就进去的数据一定是在确保被消费之后，才投递新的数据的。

> 因为消费者不消费channel数据时，生产者会被阻塞。但是为了确保消费者正确消耗掉数据，可以在消费者增加ACK机制，即消费成功后给生产者一个应答。

应用场景：

一个总控制信号channel，来终止所有的goroutine group~~ 很实用

`N:M`这样并发模型，channel合适吗？

##### Unbufferd and bufferd

无缓冲：发送和接收动作是同时发生的。如果没有 goroutine 读取 channel （<- channel），则发送者 (channel <-) 会一直阻塞。如果仅有一个goroutine且一直阻塞，就会被判定成死锁。

缓冲：缓冲 channel 类似一个有容量的队列。当队列满的时候发送者会阻塞；当队列空的时候接收者会阻塞。

交付保证是基于一个问题：“我是否需要保证由特定的 goroutine 发送的信号已经被收到了？

 channel 选项是无缓冲，缓冲>1 或缓冲=1。

- 有保证

  * 因为信号的接收在信号发送完成之前就发生了。 
  * 一个没有缓冲的通道可以保证发送的信号已经收到。

- 无保证

  * 因为信号的发送是在信号接收完成之前发生的。
  * 一个大小>1 的缓冲通道不能保证发送的信号已经收到。

- 延迟保证

  * 因为第一个信号的接收，在第二个信号发送完成之前就发生了。

    即第一个丢进channel的数据如果不被接受，第二个是发不了的。

  * 一个大小=1 的缓冲通道为您提供了一个延迟的保证。它可以保证发送的前一个信号已经收到。



缓冲区的大小绝不是一个随机数，它必须是为一些定义好的约束而计算出来的。在计算中没有无穷远，所有的东西都必须有一个明确定义的约束，无论是时间还是空间。

##### `<-chan` and `chan<-`

chan : read-write

<-chan ：read only

chan<- : write only

Both will work indeed. But one will be more constraining. The form with the arrow pointing away from the `chan` keyword means that the returned channel will only be able to be pulled from by client code. No pushing allowed : the pushing will be done by the random number generator function(`<-chan`). Conversely, there's a third form with the arrow pointing towards `chan`, that makes said channel write-only to clients(`chan <-`).

```go

package main

import "fmt"
// 只有generator进行对outCh进行写操作，返回声明
func generator(n int) <-chan int {
	outCh := make(chan int)
	go func(){
		for i:=0;i<n;i++{
			outCh<-i
			fmt.Println("send: ", i)
		}
        // 发完才会close
		close(outCh)
		fmt.Println("close")
	}()
	// 为什么不能直接 return 不加参数呢
	return outCh
}

// consumer只读inCh的数据，声明为<-chan int
// 可以防止它向inCh写数据
func consumer(inCh <-chan int) {
	fmt.Println("read channel")
	for x := range inCh {
		fmt.Println("receive: ", x)
	}
}
func main()  {
	consumer(generator(3))
}

// output
read channel
receive:  0
send:  0
send:  1
receive:  1
receive:  2
send:  2
close

```

end

##### channel deadlock（重要）

Golang 自带死锁检测机制，当发现当前协程再也没有机会被唤醒时，则会 panic。

###### nil channel deadlock

当channel设置为nil，且是single goroutine进行读写都会导致死锁。

关闭 nil channel会引起 Panic 。

* read: `Receive may block because of 'nil' channel `
* write: `Send may block because of 'nil' channel `

但是有个特例，当`nil`的通道在`select`的某个`case`中时（有其他case可以解除case），这个case会阻塞，但不会造成死锁。

核心：golang 有死锁检测，如果一直阻塞就会死锁。有可以打破一直阻塞的线程时，就会避免死锁、

```go

package main

import (
	"fmt"
	"time"
)

func main() {
	news := make(chan string)
	news = nil
	printAllNews(news)
}

func printAllNews(news chan string) {
	for {
		select {
		case <- news :
			fmt.Println("news is nil")
         // 有解除阻塞避免死锁的条件
		case <-time.After(time.Second * 10):
			fmt.Println("Timeout: News feed finished")
			//return
		}
	}
}
// output: 去掉return 则一直输出
Timeout: News feed finished
Timeout: News feed finished
...
Timeout: News feed finished



// 去掉其他啊case时，还是会死锁
package main

import (
	"fmt"
)

func main() {
	news := make(chan string)
	news = nil
	printAllNews(news)
}

func printAllNews(news chan string) {
	for {
		select {
         // 没有其他case解决死锁--依旧会死锁
		case <- news :
			fmt.Println("news is nil")
		}
	}
}

// output
fatal error: all goroutines are asleep - deadlock!

```

end



###### single  coroutine deadlock(重要)

核心：单协程一直阻塞就会是死锁--

Channel和goroutine的调度器机制是紧密相连的，**一个发送操作——或许是整个程序——可能会永远阻塞。**

没有buffer的channel只能通过另一个goroutine去读,否则就阻塞了。

单个goroutine操作无缓冲的channel时，会直接死锁--

```go
package main

import (
"fmt"
)

func main() {
	ch := make(chan int)
	ch <- 1
	fmt.Println(<-ch) // 1

}

// output
fatal error: all goroutines are asleep - deadlock!

goroutine 1 [chan send]:
main.main()
	D:/mihoyo-last/study code/awesomeProject/t2.go:109 +0x60


// cache buffer channel
package main

import (
"fmt"
)

func main() {
	ch := make(chan int, 3)
	ch <- 1
	fmt.Println(<-ch) // 1

}

// output
1

package main

import (
	"fmt"
)

func main() {
	ch := make(chan int, 3)
	ch <- 1
	fmt.Println(<-ch) // 1
	fmt.Println(<-ch) // 1

}
// output
1
fatal error: all goroutines are asleep - deadlock!

goroutine 1 [chan receive]:
main.main()
	D:/mihoyo-last/study code/awesomeProject/t2.go:73 +0x110

Process finished with exit code 2
```

只有一个goroutine即主goroutine的，且使用的是无缓冲信道的情况下。

无缓冲信道不存储值，无论是传值还是取值都会阻塞。这里只有一个主协程的情况下，第一段代码是阻塞在传值，第二段代码是阻塞在取值。因为一直卡住主协程，系统一直在等待，所以系统判断为死锁，最终报deadlock错误并结束程序。

有缓冲的channel，在缓存满之前完全是解耦的操作，即一个goroutine可以在buffer未满的情况下，对一个有缓存的channel同时进行取存操作不会阻塞死锁



##### channel的行为

https://www.infoq.cn/article/wZ1kKQLlsY1N7gigvpHo



##### for + channel

最简单的，使用for遍历channel, 一个写一个读--

```go
package main

import "fmt"

func main() {
	count := 5
	ch := generate(count)
	for i := 0; i < count; i++ {
		fmt.Println(<-ch)
	}
}
func generate(count int) <-chan int {
	ch := make(chan int)
	go func() {
		for i := 0; i < count; i++ {
		ch <- i
		}
	}()
	return ch
}
```

end

##### range + channel

```go
package main

import "fmt"

func main() {

    queue := make(chan string, 2)
    queue <- "one"
    queue <- "two"
    // 很重要
    close(queue)

    for elem := range queue {
        fmt.Println(elem)
    }
}

// output
one
two
```

进阶可看

https://betterprogramming.pub/manging-go-channels-with-range-and-close-98f93f6e8c0c

如果，range 读一个没有closed channel 就会死锁。

For channels, the iteration values produced are the successive values sent on the channel **until the channel is closed.** If the channel is nil, the range expression blocks forever.

```go
package main

import "fmt"

func main() {
	count := 5
	ch := generate(count)
	for v := range ch {
		fmt.Println(v)
	}
}

func generate(count int) <-chan int {
	ch := make(chan int)
	go func() {
		for i := 0; i < count; i++ {
			ch <- i
		}

	}()
	return ch
}
// output
0
1
2
3
4
fatal error: all goroutines are asleep - deadlock!
```



end

##### select + channel

Go 语言的 `select` 与操作系统中的 `select` 比较相似。	　

`select`线性扫描，即轮询,仅知道有I/O事件发生，却不知是哪几个流，只会无差异轮询所有流，找出能读数据或写数据的流进行操作。时间复杂的O(n).

使用`select`和`time.After`，看操作和定时器哪个先返回，处理先完成的，就达到了超时控制的效果

`time.After`本质还是一个channel

```go
func NewTimer(d Duration) *Timer {
	c := make(chan Time, 1)
	t := &Timer{
		C: c,
		r: runtimeTimer{
			when: when(d),
			f:    sendTime,
			arg:  c,
		},
	}
	startTimer(&t.r)
	return t
}

```

https://golangbyexample.com/select-with-nil-channel-golang/

Send or receive operation on nil channel blocks forever. Hence a use case of having a nil channel in the select statement is to disable that case statement after the the send or receive operation is completed on that case statement. The channel then can simply be set to nil. That case statement will be ignored when the select statement is executed again and receive or send operation will be e waited on another case statement. So it is meant to ignore that case statement and execute the other case statement

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	news := make(chan string)
	news = nil
	//go newsFeed(news)
	printAllNews(news)
}

func printAllNews(news chan string) {
	for {
		select {
		case <- news :
			fmt.Println("news is nil")
		case <-time.After(time.Second * 10):
			fmt.Println("Timeout: News feed finished")
			return
		}
	}
}

// output
Timeout: News feed finished

Process finished with exit code 0

```

end

Channel 的事件触发机制与 select 的 case 机制结合，差不多成为 golang 编程范式了。从不同的并发执行的 goroutine 中获取值可以通过关键字 select 来完成，select 是 golang 特别有趣的用法:)。

```go
package main
import (
	"fmt"
	"math/rand"
	"time"
)

const letterBytes = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ"

func RandStringBytes(n int) string {
	b := make([]byte, n)
	for i := range b {
		b[i] = letterBytes[rand.Intn(len(letterBytes))]
	}
	return string(b)
}

func main(){
	ch1 := make(chan string)
	ch2 := make(chan string)
	var num int
	num = 0

	go func() {
		for {
			time.Sleep(time.Second * 2 )
			ch1 <-  RandStringBytes(3)
		}
	}()

	go func(){
		time.Sleep(time.Second * 4)
		fmt.Println("sleep 4 seconds and close(ch2)")
		close(ch2)
	}()

	for {
		select {
		case v, _ := <-ch1:
			fmt.Println("case is ch1", v)
         // channel close 后，还会触发ch2， 因为 <- ch2 会返还零值-- 
		case <-ch2:
			time.Sleep(time.Second * 4)
			num = num + 1
			fmt.Println("case is ch2", num)
         // 仅在ch1 和 ch2 为产生数据时触发了
		default:
			time.Sleep(time.Second * 1)
			fmt.Println("case default sleep 1 sec... ")
		}

	}
}
	
/// output
case default sleep 1 sec... 
case default sleep 1 sec... 
case is ch1 XVl
case default sleep 1 sec... 
sleep 4 seconds and close(ch2)
case default sleep 1 sec... 
case is ch1 Bzg
case is ch2 1
case is ch1 bai
case is ch2 2
case is ch1 CMR
...
case is ch1 KQF
// 一直触发 case ch2
case is ch2 10
case is ch2 11
case is ch2 12
case is ch2 13
...
```

end

下面是个比较流弊的栗子

在实现微服务网关时，一般会有一个主控协程（manager goroutine）负责处理各个子 channel 的调度，是非常优雅的实现 [代码](https://github.com/pandaychen/gobetween/blob/master/src/server/scheduler/scheduler.go#L117)。

```go
	/**
	 * Goroutine updates and manages backends
	 * 启动一个独立的goroutine
	 */
	go func() {
		for {
			select {

			/* ----- discovery ----- */

			// handle newly discovered backends
			case backends := <-this.Discovery.Discover():
				//接收服务发现获取的后端列表
				this.HandleBackendsUpdate(backends)
				this.Healthcheck.In <- this.Targets()		//将新的backends放入healthy探测
				this.StatsHandler.BackendsCounter.In <- this.Targets()	//

			/* ------ healthcheck ----- */

			// handle backend healthcheck result
			case checkResult := <-this.Healthcheck.Out:
				// 从healthy check中获取结果
				this.HandleBackendLiveChange(checkResult.Target, checkResult.Live)

			/* ----- stats ----- */

			// push current backends to stats handler
			case <-backendsPushTicker.C:
				this.StatsHandler.Backends <- this.Backends()

			// handle new bandwidth stats of a backend
			case bs := <-this.StatsHandler.BackendsCounter.Out:
				this.HandleBackendStatsChange(bs.Target, &bs)

			/* ----- operations ----- */

			// handle backend operation
			case op := <-this.ops:
				this.HandleOp(op)

			// elect backend（在TakeBackend函数中触发channel读取）
			case electReq := <-this.elect:
				this.HandleBackendElect(electReq)

			/* ----- stop ----- */

			// handle scheduler stop
			//scheduler退出时，回收资源和优雅退出
			case <-this.stop:
				log.Info("Stopping scheduler ", this.StatsHandler.Name)
				backendsPushTicker.Stop()
				this.Discovery.Stop()
				this.Healthcheck.Stop()
				metrics.RemoveServer(fmt.Sprintf("%s", this.StatsHandler.Name), this.backends)
				return
			}
		}
	}()
```

end

##### 生产者-消费者模型(有点问题)

生产者产生一些数据将其放入 channel；然后消费者按照顺序，一个一个的从 channel 中取出这些数据进行处理。这是最常见的 channel 的使用方式。当 channel 的缓冲用尽（无数据）时，生产者必须等待（阻塞）。

WaitGroup 的描述是：`一个 WaitGroup 对象可以等待一组协程结束`。使用方法是：

1. main协程通过调用 `wg.Add(delta int)` 设置worker协程的个数，然后创建worker协程；

   给计数器增加delta，比如启动1个协程就增加1。

   下面栗子中，`wg.Add(1)` ，它怎么知道这个1是哪个goroutine或者函数

2. worker协程执行结束以后，都要调用 `wg.Done()`；

   协程退出前执行，把计数器减1。

3. main协程调用 `wg.Wait()` 且被block，直到所有worker协程全部执行结束后返回。

   在等待计数器减为0之前，Wait()会一直阻塞当前的goroutine

也就是说，Add()用来增加要等待的goroutine的数量，Done()用来表示goroutine已经完成了，减少一次计数器，Wait()用来等待所有需要等待的goroutine完成。

**Add函数一定要在Wait函数执行前执行**，这在Add函数的[文档](https://link.segmentfault.com/?url=https%3A%2F%2Fgolang.org%2Fsrc%2Fsync%2Fwaitgroup.go%3Fs%3D2022%3A2057%23L43)中就提示了: *Note that calls with a positive delta that occur when the counter is zero must happen before a Wait.*。

如何确保Add函数一定在Wait函数前执行呢？在协程情况下，我们不能预知协程中代码执行的时间是否早于Wait函数的执行时间，但是，我们可以在分配协程前就执行Add函数，然后再执行Wait函数，以此确保。

```go
package main

import (
        "fmt"
        "sync"
        "time"
)

func producer(wg *sync.WaitGroup, c chan int64, max int) {
        defer wg.Done()
        for {
                for i := 0; i < max; i++ {
                        c <- time.Now().Unix()
                }

                time.Sleep(1 * time.Second)
        }
}

func consumer(wg *sync.WaitGroup, c chan int64) {
        defer wg.Done()
        var v int64
        ok := true

        for ok {
                if v, ok = <-c; ok {
                        fmt.Println(v)
                }
        }
}

func main() {
        var wg sync.WaitGroup
        var chan_size int = 10
        chan1 := make(chan int64, chan_size)
        go producer(&wg, chan1, chan_size)
        wg.Add(1)
        go consumer(&wg, chan1)
        wg.Add(1)

        wg.Wait()
        close(chan1)
}
```

end

##### 生成器：Generator

```go
package main

import "fmt"

func uuidgenerator() <-chan string {
        id := make(chan string)
        go func() {
                var counter int64 = 0
                for {
                        id <- fmt.Sprintf("%x", counter)
                        counter += 1
                }
        }()
        return id
}
func main() {
        id := uuidgenerator()
        for i := 0; i < 10; i++ {
                fmt.Println(<-id)
        }
}
```

end

#### 关闭 channel 的优雅做法（记住）

有点意思，因为主动关闭的channel还是可以读到zero-value。

在主动（或被动）关闭 channel 的时候，最好设置 channel 为 nil（nil channel 在 select 中很有用），这是一个好习惯，通过将已经关闭的 channel 置为 nil，下次 select 将会阻塞在该 channel 上，使得 select 继续下面的分支 evaluation。

```go
package main

import "fmt"
import "time"

func main() {
        var c1, c2 chan int = make(chan int), make(chan int)
        go func() {
                time.Sleep(time.Second * 5)
                c1 <- 5
                close(c1)
        }()

        go func() {
                time.Sleep(time.Second * 7)
                c2 <- 7
                close(c2)
        }()

        for {
                select {
                case x, ok := <-c1:
                        if !ok {
                                c1 = nil
                        } else {
                                fmt.Println(x)
                        }
                case x, ok := <-c2:
                        if !ok {
                                c2 = nil
                        } else {
                                fmt.Println(x)
                        }
                }
                if c1 == nil && c2 == nil {
                        break
                }
        }
        fmt.Println("over")
}
```

end

#### 深拷贝

针对，引用类型/地址类型数据的 拷贝，避免影响源数据--

#### channel 并发栗子

##### select default case 性能陷阱: 看

大佬的测试栗子

https://github.com/sinomoe/sino.moe/issues/11

Similar to switch select can also have a default case. This default case will be executed if no send it or receive operation is ready on any of the case statement. So in a way default statement prevent the select from blocking forever.  So a very important point to note is that the default statement makes the select non-blocking. If the select statement doesn’t contain a default case then it can block forever until one send or receive operation is ready on any case statement. Let’s see a example to fully understand it

```go
package main

import "fmt"

func main() {
    ch1 := make(chan string)
    select {
    case msg := <-ch1:
        fmt.Println(msg)
    default:
        fmt.Println("Default statement executed")
    }
}
// output
Default statement executed
```

如果，去掉`default-case:` 直接导致死锁

```go
func main()  {
	ch := make(chan int)

	select {
	case <-ch:
		fmt.Println("ok")
	//default:
        // 这个语句不影响，主要是 default case statement
		//fmt.Println("false")
	}
}

fatal error: all goroutines are asleep - deadlock!

goroutine 1 [chan receive]:
main.main()
	D:/data_files/Go/src/etcd/dodo_test/test.go:96 +0x5d
```

但是，在高并发的场景下使用`default-case`会导致性能极大损耗

由于`defalut`语句为空或者其逻辑很短，相当于引入了一个死循环占用了绝大多数CPU时间，导致性能骤降，所以在使用`select`时，能避免使用`defalut`就应尽量避免，如果必须要使用`defalut`，其内部的逻辑也不应该太少，以免导不必要的致性能损失。

```go
func test(num chan int){
	timeout := time.After(30 * time.Second)
	goID := GetGoid()
	someMapMutex.Lock()
	countGoID[goID]=0
	someMapMutex.Unlock()
	fmt.Printf("Goroutine %d begin to call.\n",goID)
	for {
		select {
		case  <-num:
			someMapMutex.Lock()
			countGoID[goID]++
			someMapMutex.Unlock()
		case <- timeout:
			return
		//default:
		}
	}
}

func main() {
	c := make(chan  int)
	cNum := 500000
	go test(c)
	go test(c)
	go test(c)
	go test(c)
	go test(c)
	go test(c)
	go test(c)
	for i:=0;i<cNum;i++{
		c <- i
	}
	fmt.Printf("total send %d times data to channel\n", cNum)
	for k,v := range countGoID{
		fmt.Printf("Goroutine %d was called %d times.\n", k,v)
	}
}
```

由于使用了`default` 且 有timeout ，多个goroutine + default_case时，导致CPU性能损坏，执行goroutine效率变低，且channel input数量多，在timeout到期时，goroutine <- chan，全部结束了，就留下main goroutine 执行chan <- 所以死锁了--

大佬的测试栗子

https://github.com/sinomoe/sino.moe/issues/11

### 自定义类型

#### type

通过Type 关键字在原有类型基础上构造出一个新类型。但是**新类型不会拥有原基础类型所附带的方法**。

函数也是一个确定的类型，就是以函数签名作为类型。这种类型的定义例如

```go
type Option func(person * Person)
```

根据函数签名声明一个新的类型，就可以使用这个新类型创建function slice了

```go
// sliceTest := make([]Option, 2, 3)
func newPerson(options ...Option) Person{}
...
sliceTest := make([]Option, 2, 2)
personTest := newPerson(sliceTest...)
fmt.Println("personTest:", personTest)
```

如果不使用type重新定义变量...也可以就是不太好看--

```go
func main()  {
	s := make([]func(name string), 2, 3)
	s[0] = func(n string) {
		fmt.Println(n)
	}
	test(s[0])
}

func test(fs ...func(name string))  {
	for _, f := range fs {
		f("aa")
	

```

end

##### 别名and新类型

type myType = int32 相当于别名，没有新的type metadata产生。

`byte`是`uint8`的一个内置别名，`rune`是`int32`的一个内置别名.

```go
// byte is an alias for uint8 and is equivalent to uint8 in all ways. It is
// used, by convention, to distinguish byte values from 8-bit unsigned
// integer values.
type byte = uint8

// rune is an alias for int32 and is equivalent to int32 in all ways. It is
// used, by convention, to distinguish character values from integer values.
type rune = int32
```

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/golang-type-alias-and-newtype.jpg)

#### struct：值类型

Struct Name自身小写也不能其他package被导入使用。

一个自定义的数据类型，就是一个结构体（等同于自定义int、string、float等数据类型）；创建、引用对象和值类型的基本数据类型一致。

```go
// 值类型
type Test struct{
    Name string
}
test := Test{"name":"Robin",}

//引用类型，底层是(*hmap)
mapTest := make(map[string]string)
mapTest["Name"]="Robin"
```

声明定义结构体，要注意：

* 首字母大写是公有的，首字母小写是私有的
* 可以结合json tag实现不同序列化需求

```go
package utils

import "fmt"

type Eggo struct {
    // name string 则不能被其他文件引用
	Name string
	Age int
}

func (receiver Eggo) A()  {
	fmt.Println("a")
}

func (receiver Eggo) B()  {
	fmt.Println("b")
}

// 引用结构体
package main

import (
	"fmt"
	"test/utils"
)

func main()  {
	a := utils.Eggo{
		Name: "aa",
		Age: 17,
	}
	fmt.Println(a.Name)
	a.A()

}

```

struct的属性是否被导出，也遵循大小写的原则：首字母大写的被导出，首字母小写的不被导出。

1. **如果struct名称首字母是小写的，这个struct不会被导出。连同它里面的字段也不会导出，即使有首字母大写的字段名**。
2. **如果struct名称首字母大写，则struct会被导出，但只会导出它内部首字母大写的字段，那些小写首字母的字段不会被导出**。

也就是说，struct的导出情况是混合的。

但并非绝对如此，**如果struct嵌套了，那么即使被嵌套在内部的struct名称首字母小写，也能访问到它里面首字母大写的字段**。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/golang同package可以直接引用.jpg)

##### struct tag 101:imp:

反序列话时，key必须和struct字段或者字段对应tag相等

```go
type User struct {
	Id        int       `json:"id"`
	Name      string    `json:"name"`
	Email      string    `json:"emailAddress"`
}

func main() {
	u := &User{}
	str := []byte(`{"name":"lee","id":5266, "mail":"test@163.com"}`)    //这个不能导出mail，因为mail不是`Email`, 也不是`emailAddress`,所以不能导出
	//str := []byte(`{"name":"lee","id":5266, "emailAddress":"test@163.com"}`)
	if err := json.Unmarshal(str, u); err != nil {
		return
	}
	fmt.Println(u)
}
// output
&{5266 lee }
```

##### struct tag 102: map

`valid`是自定义的校验字段

```go
type Person struct {
    Name string    `json:"name"`
    Age  int       `json:"age" valid:"1-100"`
}
 
func (p * Person) validation() bool {
    v := reflect.ValueOf(*p)
    tag := v.Type().Field(1).Tag.Get("valid")
    val := v.Field(1).Interface().(int)
    fmt.Printf("tag=%v, val=%v\n", tag, val)
    
    result := strings.Split(tag, "-")
    var min, max int
    min, _ = strconv.Atoi(result[0])
    max, _ = strconv.Atoi(result[1])
 
    if val >= min && val <= max {
        return true
    } else {
        return false
    }
}
 
func main() {
    person1 := Person { "tom", 12 }
    if person1.validation() {
        fmt.Printf("person 1: valid\n")
    } else {
        fmt.Printf("person 1: invalid\n")
    }
    person2 := Person { "tom", 250 }
    if person2.validation() {
        fmt.Printf("person 2 valid\n")
    } else {
        fmt.Printf("person 2 invalid\n")
    }
}
```

##### struct tag 103：advanced map:six:

```go
package main

import (
  "fmt"
  "reflect"
  "regexp"
  "strings"
)

// Name of the struct tag used in examples.
const tagName = "validate"

// Regular expression to validate email address.
var mailRe = regexp.MustCompile(`\A[\w+\-.]+@[a-z\d\-]+(\.[a-z]+)*\.[a-z]+\z`)

// Generic data validator.
type Validator interface {
  // Validate method performs validation and returns result and optional error.
  Validate(interface{}) (bool, error)
}

// DefaultValidator does not perform any validations.
type DefaultValidator struct {
}

func (v DefaultValidator) Validate(val interface{}) (bool, error) {
  return true, nil
}

// StringValidator validates string presence and/or its length.
type StringValidator struct {
  Min int
  Max int
}

func (v StringValidator) Validate(val interface{}) (bool, error) {
  l := len(val.(string))

  if l == 0 {
    return false, fmt.Errorf("cannot be blank")
  }

  if l < v.Min {
    return false, fmt.Errorf("should be at least %v chars long", v.Min)
  }

  if v.Max >= v.Min && l > v.Max {
    return false, fmt.Errorf("should be less than %v chars long", v.Max)
  }

  return true, nil
}

// NumberValidator performs numerical value validation.
// Its limited to int type for simplicity.
type NumberValidator struct {
  Min int
  Max int
}

func (v NumberValidator) Validate(val interface{}) (bool, error) {
  num := val.(int)

  if num < v.Min {
    return false, fmt.Errorf("should be greater than %v", v.Min)
  }

  if v.Max >= v.Min && num > v.Max {
    return false, fmt.Errorf("should be less than %v", v.Max)
  }

  return true, nil
}

// EmailValidator checks if string is a valid email address.
type EmailValidator struct {
}

func (v EmailValidator) Validate(val interface{}) (bool, error) {
  if !mailRe.MatchString(val.(string)) {
    return false, fmt.Errorf("is not a valid email address")
  }
  return true, nil
}

// Returns validator struct corresponding to validation type
func getValidatorFromTag(tag string) Validator {
  args := strings.Split(tag, ",")

  switch args[0] {
  case "number":
    validator := NumberValidator{}
    fmt.Sscanf(strings.Join(args[1:], ","), "min=%d,max=%d", &validator.Min, &validator.Max)
    return validator
  case "string":
    validator := StringValidator{}
    fmt.Sscanf(strings.Join(args[1:], ","), "min=%d,max=%d", &validator.Min, &validator.Max)
    return validator
  case "email":
    return EmailValidator{}
  }

  return DefaultValidator{}
}

// Performs actual data validation using validator definitions on the struct
func validateStruct(s interface{}) []error {
  errs := []error{}

  // ValueOf returns a Value representing the run-time data
  v := reflect.ValueOf(s)

  for i := 0; i < v.NumField(); i++ {
    // Get the field tag value
    tag := v.Type().Field(i).Tag.Get(tagName)

    // Skip if tag is not defined or ignored
    if tag == "" || tag == "-" {
      continue
    }

    // Get a validator that corresponds to a tag
    validator := getValidatorFromTag(tag)

    // Perform validation
    valid, err := validator.Validate(v.Field(i).Interface())

    // Append error to results
    if !valid && err != nil {
      errs = append(errs, fmt.Errorf("%s %s", v.Type().Field(i).Name, err.Error()))
    }
  }

  return errs
}

type User struct {
  Id    int    `validate:"number,min=1,max=1000"`
  Name  string `validate:"string,min=2,max=10"`
  Bio   string `validate:"string"`
  Email string `validate:"email"`
}

func main() {
  user := User{
    Id:    0,
    Name:  "superlongstring",
    Bio:   "",
    Email: "foobar",
  }

  fmt.Println("Errors:")
  for i, err := range validateStruct(user) {
    fmt.Printf("\t%d. %s\n", i+1, err.Error())
  }
}
```





#### interface

##### `Type Assertion`（断言）

Golang断言是基于`interface{}`实现的。

`Type Assertion`（断言）是用于`interface value`的一种操作，语法是`x.(T)`，`x`是`interface type`的表达式，而`T`是`asserted type`，被断言的类型。

```go
// 断言表达式1: 如果断言成功，就会返回值给res，如果断言失败，就会触发panic。
func main()  {
	b := [2]byte{11, 12}
	var test interface{}
	test = b
	res := test.(int)
	fmt.Println(res)
}
// output
panic: interface conversion: interface {} is [2]uint8, not int

// 断言表达式 2: 断言失败将 ok 的值设为 false ，表示断言失败，此时t 为 T 的零值。
func main()  {

	b := [2]byte{11, 12}
	var test interface{}
	test = b

	res, ok := test.(int)
	if ok{
		fmt.Println("turn string to []byte, success")
	}
	fmt.Println(res)

}
// output
0
```

end

### select随机性

Execution of a "select" statement proceeds in several steps:

1. ...
2. If one or more of the communications can proceed, a single one that can proceed is chosen via a uniform pseudo-random selection. Otherwise, if there is a default case, that case is chosen. If there is no default case, the "select" statement blocks until at least one of the communications can proceed.
3. ...

确实是随机的，但是随机逻辑还没研究。

```go
package main

import "fmt"

func main()  {
	for  {
		testSelect()
	}
}

func testSelect()  {
	chOne := make(chan string, 1)
	chTwo := make(chan string, 1)
	chThree := make(chan string, 1)

	chOne <- "1"
	chTwo <- "2"
	chThree <- "3"
	
	select {
	case <-chOne:
		fmt.Println("One")
	case <-chTwo:
		fmt.Println("Two")
	case <-chThree:
		fmt.Println("Three")
	default:
		fmt.Println("default")
	}
}
// output
Two
One
Two
Three
Two
Two
One
One
```

#### 实现原理

case也是一个channel？

```go
// Select case descriptor.
// Known to compiler.
// Changes here must also be made in src/cmd/compile/internal/walk/select.go's scasetype.
type scase struct {
	c    *hchan         // chan
	elem unsafe.Pointer // data element
}
```

源码包`src/runtime/select.go:selectgo()`定义了select选择case的函数：

也是依赖sudog...

```go
// 伪代码
func selectgo(cas0 *scase, order0 *uint16, ncases int) (int, bool) {
    //1. 锁定scase语句中所有的channel
    //2. 按照随机顺序检测scase中的channel是否ready
    //   2.1 如果case可读，则读取channel中数据，解锁所有的channel，然后返回(case index, true)
    //   2.2 如果case可写，则将数据写入channel，解锁所有的channel，然后返回(case index, false)
    //   2.3 所有case都未ready，则解锁所有的channel，然后返回（default index, false）
    //3. 所有case都未ready，且没有default语句
    //   3.1 将当前协程加入到所有channel的等待队列
    //   3.2 当将协程转入阻塞，等待被唤醒
    //4. 唤醒后返回channel对应的case index
    //   4.1 如果是读操作，解锁所有的channel，然后返回(case index, true)
    //   4.2 如果是写操作，解锁所有的channel，然后返回(case index, false)
}
```

对于读channel的case来说，如`case elem, ok := <-chan1:`, 如果channel有可能被其他协程关闭的情况下，一定要检测读取是否成功，因为close的channel也有可能返回，此时ok == false。

### Json序列化

#### Struct序列化

结构体的字段除了名字和类型外，还可以有一个可选的标签（tag）：它是一个附属于字段的字符串，可以是文档或其他的重要标记。比如在我们解析 json 或生成 json 文件时，常用到 encoding/json 包，它提供一些默认标签，例如：omitempty 标签可以在序列化的时候忽略 0 值或者空值；而 “-” 标签的作用是不进行序列化，其效果和直接将结构体中的字段写成小写的效果一样。

```go
type Info struct {
    Name string
    Age  int   `json:"age,string"`     //这样生成的json对象中，age就为字符串
    Gender  string  `json:"gender,omitempty"`  // 可以忽略Gender的空值
    Test string `json:"-"`  // 序列化时忽略这个字段
    test string `json:test`
}

// demo
type Info struct {
	Name  string `json:"name"`
	Age string `json:"age,int"`
	Gender string `json:"gender,omitempty"`
	Test string `json:"-"`
	test string `json:"test"`
}

func main() {

	infoTest := Info{
		Name: "Robin",
		Age: "18",
		Gender: "Man",
		Test: "TT",
		test: "tt",
	}
	res, err := json.Marshal(infoTest)
	if err != nil{
		panic("marshal error")
	}
	fmt.Printf("Marshar string: %s\n", res)
}
// output
Marshar string: {"name":"Robin","age":"18","gender":"Man"}
```

end

#### map序列化

```go
func main() {

	infoTest := Info{
		Name: "Robin",
		Age: "18",
		Gender: "Man",
		Test: "TT",
		test: "tt",
	}

	m := make(map[string]string)
	m["Game"] = "lol"
	resM, err := json.Marshal(m)
	if err != nil{
		panic("marshal error")
	}
	fmt.Printf("Marshar string: %s\n", resM)

	mapUnmarshal := make(map[string]string)

	json.Unmarshal(res, &mapUnmarshal)
	fmt.Printf("UnMarshar %v\n", mapUnmarshal)
	for k,v := range mapUnmarshal{
		fmt.Println(k, v)
	}
}
// output
Marshar string: {"Game":"lol"}
UnMarshar map[age:18 gender:Man name:Robin]
name Robin
age 18
gender Man
```

end

#### map[string]interface{}

对于value类型不确定的数据使用`interface{}`作为value

```go
// 元数据
{
    "device": "1",
    "data": [
        {
            "humidity": "27",
            "time": "2017-07-03 15:23:12"
        }
    ]
}


func main() {
	// marshal string
	json_str := "{\"device\": \"1\",\"data\": [{\"humidity\": \"27\",\"time\": \"2017-07-03 15:23:12\"}]}"

	m := make(map[string]interface{})
	mTest := make(map[string]map[string]string)

	json.Unmarshal([]byte(json_str), &m)
	fmt.Printf("%v\n", m)

    // Error: one value is string, another is map type
	json.Unmarshal([]byte(json_str), &mTest)
	fmt.Printf("%v\n", mTest)
}

// output
map[data:[map[humidity:27 time:2017-07-03 15:23:12]] device:1]
map[data:map[] device:map[]]

```

end

#### slice 序列化

```go
func testSlice() {
	//定义一个slice，元素是map
	var m map[string]interface{}
	var s []map[string]interface{}
	m = make(map[string]interface{})
	m["username"] = "user1"
	m["age"] = 18
	m["sex"] = "man"
	s = append(s, m)
	m = make(map[string]interface{})
	m["username"]="user2"
	m["age"]=188
	m["sex"]="male"
	s=append(s,m)
	data, err := json.Marshal(s)
	if err != nil {
		fmt.Printf("json.marshal failed,err:", err)
		return
	}
	fmt.Printf("%s\n", string(data))

}
// output
[{"age":18,"sex":"man","username":"user1"},{"age":188,"sex":"male","username":"user2"}]
```



### T and *T

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/golang-method.jpg)

指针类似数据在编译阶段根据操作系统和cpu类型是可以确定的，所以为了支持接口:

声明`func(t T)A(){}`时，会自动生成`func(pt *T)A(){}`。

因此，Golang不允许为`T`和`*T`定义同名method。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/golang-interface-and-method.jpg)

分析可以执行文件可以看到`*T`method会包含`T`同名method。

但是链接器会去掉最终没有引用的method，so有些看不到

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/golang-method-t-and-pt.jpg)

### nil and zero-value

`nil`

> Cannot use 'nil' as type string

Golang 中定义不同类型的变量，不是通过声明就是通过 make 或 new 。 未显式初始化时，内存将被赋予一个默认的初始化，该默认值便为该类型的零值。不同的类型有不同的零值。

| 类型     | 零值           |
| :------- | :------------- |
| 数值类型 | 0              |
| 布尔类型 | false          |
| 字符串   | ""（空字符串） |
| slice    | nil            |
| map      | nil            |
| 指针     | nil            |
| 函数     | nil            |
| 接口     | nil            |
| channel  | nil            |

示例代码：

```go

package main

import (
	"fmt"
)

func main() {
	var testIntZeroValue int
	var testStringZeroValue string
	var testArrayZeroValue [8]int
	var testSliceZeroValue []int
	var testMapZeroValue map[string]int
	var testChannelZeroValue chan int
	//testMapZeroValue["a"]=1li


	//testChannelZeroValue <- 2

	fmt.Println("testIntZeroValue:", testIntZeroValue)
	fmt.Println("testStringZeroValue:", testStringZeroValue)
	fmt.Println("testArrayZeroValue:", testArrayZeroValue)
	fmt.Println("testSliceZeroValue:", testSliceZeroValue)
	fmt.Println("testMapZeroValue:", testMapZeroValue)
	fmt.Println("testChannelZeroValue:", testChannelZeroValue)

	testMapZeroValue["aa"] = 1
}

// output
testIntZeroValue: 0
testStringZeroValue: 
testArrayZeroValue: [0 0 0 0 0 0 0 0]
testSliceZeroValue: []
testMapZeroValue: map[]
testChannelZeroValue: <nil>
panic: assignment to entry in nil map

goroutine 1 [running]:
main.main()
	D:/mihoyo-last/study code/awesomeProject/t2.go:88 +0x40a
```

end

### fmt

格式

The verbs:

`%+v`挺有用的，可以额外 输出 struct的fields

General:

```
%v	the value in a default format
	when printing structs, the plus flag (%+v) adds field names
%#v	a Go-syntax representation of the value
%T	a Go-syntax representation of the type of the value
%%	a literal percent sign; consumes no value
```

Boolean:

```
%t	the word true or false
```

Integer:

```
%b	base 2
%c	the character represented by the corresponding Unicode code point
%d	base 10
%o	base 8
%O	base 8 with 0o prefix
%q	a single-quoted character literal safely escaped with Go syntax.
%x	base 16, with lower-case letters for a-f
%X	base 16, with upper-case letters for A-F
%U	Unicode format: U+1234; same as "U+%04X"
```

end

#### `*Print*`

* Print: 输出到控制台(不接受任何格式化，它等价于对每一个操作数都应用 %v)
* Println: 输出到控制台并换行
* Printf:  只可以打印出格式化的字符串。只可以直接输出字符串类型的变量
* Sprintf: 格式化并返回一个字符串而不带任何输出。
* Fprintf: 格式化并输出到 io.Writers 而不是 os.Stdout

```go
opts := metav1.ListOptions{
	LabelSelector: "app=...",
	FieldSelector: fmt.Sprintf("spec.ports[0].nodePort=%s", port),
}
_, err := c.CoreV1().Services(namespace).List(opts)
```

end

#### %+v

真的有用, 结构体嵌套时，仅使用`%v`看不到key，只能看到value很难受。

```go
deploymentInfo, err := client.AppsV1().Deployments(podNamespace).Get(
			context.TODO(),
			deploymentName,
			metav1.GetOptions{},
		)
fmt.Printf("%v\n",deploymentInfo.Status)
//output：懵逼
{1 3 3 3 3 0 [{Progressing...

//output: 清晰
fmt.Printf("%+v\n",deploymentInfo.Status)
{ObservedGeneration:1 Replicas:3 UpdatedReplicas:3 ReadyReplicas:3 AvailableReplicas:3 UnavailableReplicas:0 Conditions:...
```

end

### main

Golang 的 main是入口函数： main function must have no arguments and no return values.

### return xxx(×) and return(√)

这个太强了，golang可以在函数声明时，直接声明好 return value variable，并且直接使用`return` 返回。

```go
package main

import (
	"fmt"
)

func foo() (r string, err error) {
	return "foo", err
}
func bar() (r string, err error) {
	r = "bar"
	return
}
// 啧啧 不赋值就是nil
func test() (r string) {
	return
}
func main() {
	fooRet, _ := foo()
	barRet, _ := bar()
	fmt.Printf("foo return values :%s\n", fooRet)
	fmt.Printf("bar return values :%s\n", barRet)
	fmt.Printf("test return values :%s\n", test())

}
// output
foo return values :foo
bar return values :bar
test return values :
```

`return xxx` + defer

```go
package main

import "fmt"

func f() (result int) {
	defer func() {
		result++
	}()
	return 0
}
func main() {
	fmt.Println(f())
}

// output
1
```

打印的结果是`1`而不是`0`.

因为`return 0` 等于`result = 0; return` 而defer是执行在`return` 之前的。

所以，正确的执行顺序就是：

* result = 0
* defer：result ++
* return

所以，f()返回值为1.

结论：

* `return` 是原子操作

* `return xxx` 不是原子操作

  `result = 0 ` 语句不是一条原子调用，return xxx其实是赋值＋ret指令



### make

make是Go的内置函数，专门用于分配和初始化指定大小的slice、map、chan类型，它返回的是一个type。而new则不同，new返回的是一个*type,也就是一个指针类型，指向type的零值。

`cap` tells you the capacity of the underlying array. `len` tells you how many items are in the array.

make 只能用于 slice，map，channel 三种类型，make(T, args) 返回的是初始化之后的 T 类型的值，这个新值并不是 T 类型的零值，也不是指针 *T，是经过初始化之后的 T 的引用。

当make初始化len为0时，是不能直接使用下标的。

```go
import "fmt"
const maxPods = 110
func main()  {
	li := make([]string, 0, maxPods)
	fmt.Printf("li equal nil? : %v\n", li == nil)
}
// output
// 因为 slice的cap 和 len 字段都有值了 只有data是nil
// 随意会输出 false
li equal nil? : false


// len is 0
func main()  {
	li := make([]int, 0)
	li[0] = 5
	fmt.Println(li)
}
// output
panic: runtime error: index out of range [0] with length 0
```

end

### new

`new` 的作用是为类型申请一片内存空间，并返回指向这片内存的指针。

new(T)会为T类型的新项目，分配被置零的存储，并且返回它的地址，一个类型为*T的值。

```go
i := new(int)
// func new(Type) *Type
// 上面和下面的操作是等价的
var v int
i := &v
```

end

## 引用

1. https://segmentfault.com/a/1190000020086645
2. https://space.bilibili.com/567195437/video
3. https://www.zhihu.com/question/60426831/answer/176204563
4. https://cloud.tencent.com/developer/article/1766867
5. https://juejin.cn/post/7053450610386468894
6. https://oldpan.me/archives/linux-a-so-o-tell
7. https://segmentfault.com/a/1190000023879178
8. https://juejin.cn/post/6844904177022271501
9. https://www.imooc.com/article/261217
10. https://learnku.com/docs/bettercoding/1.0/structure-label/6968#52a4a5
11. https://cloud.tencent.com/developer/article/1200349
12. https://www.jianshu.com/p/eb539203a982
13. https://www.cnblogs.com/wuyepeng/p/13910678.html
14. https://blog.csdn.net/whatday/article/details/109771182
