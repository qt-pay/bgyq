## 菱形依赖-diamond dependency conflict



### dependency hell：依赖地狱

“依赖地狱”问题不是某种编程语言独有的问题，而是**一个在软件开发和发布领域广泛存在问题**。

当几个软件包对相同的共享包或库有依赖性，但它们依赖于不同的、不兼容的共享包版本时，就会出现依赖性问题。如果共享包或库只能安装一个版本，用户可能需要通过获得较新或较旧版本的依赖包来解决这个问题。反过来，这可能会破坏其他的依赖关系。

### why？

约定符号: A->B:v1 表示 项目A依赖于项目B的v1版本.

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/diamond-dependency.png)

依赖场景:  自己的项目P,  P->A, P->B,  A->C:v1, B->C:v2 , 见上图, 因形状得名 菱形依赖.

冲突原因: C的v1与v2有函数签名等的变化, 导致 实际运行时类找不到或语法校验错误.

能解决的情形: C的某个版本可以兼容, 想办法调整 pom 内容, 明确对该版本的依赖即可.

不能解决的情形: A用的方法只在 C:v1 中有, B用的方法只在 C:v2 中有, 则无解.
### java

Java也不能特别好的解决冲突问题，原因也很简单，jar依赖jar，那只要有版本要求不一样，就会有潜在的冲突风险。不过这几年加起来，我也只遇到过两三次就是了



### C++

统一的包管理系统，菱形依赖问题cmake、automake等工具链都“解决了”，但因为它们“都解决了”，所以还是没有解决。

没有包管理系统，所以还得手工组织源码结构，但又没有统一标准

https://www.zhihu.com/question/375368576

### golang

#### go mod why

```bash
 # why         explain why packages or modules are needed
 $  $ go mod why  -m pkg_name
```

#### go.sum

`go.sum` 的每一行都是一个条目，大致是这样的格式：

```
<module> <version>/go.mod <hash>
```

或者

```
<module> <version> <hash>
<module> <version>/go.mod <hash>
```

其中module是依赖的路径，version是依赖的版本号。hash是以`h1:`开头的字符串，表示生成checksum的算法是第一版的hash算法（sha256）。

```bash
github.com/google/uuid v1.1.1 h1:Gkbcsh/GbpXz7lPftLA3P6TYMwjCLYm83jiFQZF/3gY=  
github.com/google/uuid v1.1.1/go.mod h1:TIyPZe4MgqvfeYDBFedMoGGpEw/LqOeaOT+nhxU+yHo=
```

正常情况下，每个`依赖包版本`会包含两条记录，第一条记录为该`依赖包版本`整体（所有文件）的哈希值，第二条记录仅表示该`依赖包版本`中 `go.mod` 文件的哈希值，如果该`依赖包版本`没有 `go.mod` 文件，则只有第一条记录。如上面的例子中，`v1.1.1` 表示该`依赖包版本`整体，而 `v1.1.1/go.mod` 表示该`依赖包版本`中 `go.mod` 文件。

`依赖包版本`中任何一个文件（包括 `go.mod`）改动，都会改变其整体哈希值，此处再额外记录`依赖包版本`的 `go.mod` 文件主要用于计算依赖树时不必下载完整的`依赖包版本`，只根据 `go.mod` 即可计算依赖树。

每条记录中的哈希值前均有一个表示哈希算法的 `h1:`，表示后面的哈希值是由算法 `SHA-256` 计算出来的，自 Go module 从 v1.11 版本初次实验性引入，直至 v1.14 ，只有这一个算法。

`go.sum` 文件中记录的`依赖包版本`数量往往比 `go.mod` 文件中要多，这是因为二者记录的粒度不同导致的。`go.mod` 只需要记录直接依赖的`依赖包版本`，只在`依赖包版本`不包含 `go.mod` 文件时候才会记录间接`依赖包版本`，而 `go.sum` 则是要记录构建用到的所有`依赖包版本`。

##### go.sum合并冲突

公共库原来有版本甲。
开发者A的分支a依赖了公共库版本乙，开发者B的分支b依赖了公共库版本丙。他们分别给 go.sum 添加记录如下：

```
common/lib 甲 h1:xxx  
common/lib 乙 h1:yyy  
common/lib 甲 h1:xxx  
common/lib 丙 h1:zzz  
```

之后公共库发布了版本丁，包含了版本乙和版本丙的功能。
然后合并分支a和分支b到主干，这时候就会有合并冲突。

现在有两个选择：

1. 把两个中间版本都纳入到 go.sum 进来
2. 既不选乙，也不选丙，直接采用版本丁

无论采用哪种方法，都需要手动介入。这无疑带来了不必要的工作量。

#### vx.y.z

#### indirect

A dependency of a module can be of two kinds

- **Direct** -A direct dependency is a dependency which the module directly imports.
- **Indirect** – It is the dependency that is imported by the module’s direct dependencies. Also, any dependency that is mentioned in the go.mod file but not imported in any of the source files of the module is also treated as an indirect dependency.

**go.mod** file only records the direct dependency. However, it may record an indirect dependency in the below cases

- Any indirect dependency which is not listed in the go.mod file of your direct dependency or if direct dependency doesn’t have a **go.mod** file, then that dependency will be added to the go.mod file with `//indirect` as the suffix
- Any dependency which is not imported in any of the source files of the module.

Also checksum of all direct and indirect dependencies will be recorded in the go.sum file.



用 go mod 的时候应该经常会看到 有的依赖后面会打了一个 // indirect 的标识位，这个标识位是表示 间接的依赖。

什么叫间接依赖呢？打个比方，项目 A 依赖了项目 B，项目 B 又依赖了项目 C，那么对项目 A 而言，项目 C 就是间接依赖，这里要注意，并不是所有的间接依赖都会出现在 go.mod 文件中。间接依赖出现在 go.mod 文件的情况，可能符合下面的场景的一种或多种：

* 直接依赖未启用 Go module
* 直接依赖 go.mod 文件中缺失部分依赖
* 没有任何源代码文件引用的依赖

##### 缺少go.mod文件

可能背景：之前使用`dep`进行package管理。	

如下图所示，Module A 依赖 B，但是 B 还未切换成 Module，即没有 go.mod 文件，此时，当使用 go mod tidy 命令更新 A 的 go.mod 文件时，B 的两个依赖 B1 和 B2 将会被添加到 A 的 go.mod 文件中（前提是 A 之前没有依赖 B1 和 B2），并且 B1 和 B2 还会被添加 // indirect 的注释。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/miss-go-mod.jpg)

Since colly version v1.2.0 doesn’t have a go.mod file , all dependencies required by colly will be added to the **go.mod** file with **//indirect** as suffix

```bash
$ cat go.mod
module test

go 1.14

require	github.com/gocolly/colly v1.2.0

$ go mod tidy
$ cat go.mod
module test

go 1.14

require (
	github.com/PuerkitoBio/goquery v1.6.0 // indirect
	github.com/antchfx/htmlquery v1.2.3 // indirect
	github.com/antchfx/xmlquery v1.3.3 // indirect
	github.com/gobwas/glob v0.2.3 // indirect
	github.com/gocolly/colly v1.2.0
	github.com/kennygrant/sanitize v1.2.4 // indirect
	github.com/saintfish/chardet v0.0.0-20120816061221-3af4cd4741ca // indirect
	github.com/temoto/robotstxt v1.1.1 // indirect
	golang.org/x/net v0.0.0-20201027133719-8eef5233e2a1 // indirect
	google.golang.org/appengine v1.6.7 // indirect
)
```



##### go.mod部分依赖缺失

如下图所示，Module B 虽然提供了 go.mod 文件中，但 go.mod 文件中只添加了依赖 B1，那么此时 A 在引用 B 时，则会在 A 的 go.mod 文件中添加 B2 作为间接依赖，B1 则不会出现在 A 的 go.mod 文件中。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/miss-partial-go-mod.jpg)

##### 已下载没有任何源代码引用的依赖

We also mentioned that any dependency which is not imported in any of the source file will be marked as `//indirect`.

```bash
$ > test.go
$ > go.sum
## 或者使用go mod init重新初始化一个module
## init initializes and writes a new go.mod file in the current directory, in effect creating a new module rooted at the current directory. 
$ cat go.mod
module ab

go 1.17

## 在module rooted directory中下载 uuid package
## 对比 go.mod 和 go.sum
$ go get github.com/pborman/uuid
go get: added github.com/google/uuid v1.0.0
go get: added github.com/pborman/uuid v1.2.1
$ cat go.sum
github.com/google/uuid v1.0.0 h1:b4Gk+7WdP/d3HZH8EJsZpvV7EtDOgaZLtnaNGIu1adA=
github.com/google/uuid v1.0.0/go.mod h1:TIyPZe4MgqvfeYDBFedMoGGpEw/LqOeaOT+nhxU+yHo=
github.com/pborman/uuid v1.2.1 h1:+ZZIw58t/ozdjRaXh/3awHfmWRbzYxJoAdNJxe/3pvw=
github.com/pborman/uuid v1.2.1/go.mod h1:X/NO0urCmaxf9VXbdlT7C2Yzkj2IKimNn4k+gtPdI/k=
## 代码为空即没有引用uuid package
$ cat test.go |wc -l
0
## all dependency was  marked as //indirect.
$ cat go.mod
module ab

go 1.17

require (
        github.com/google/uuid v1.0.0 // indirect
        github.com/pborman/uuid v1.2.1 // indirect
)
root@e8f0cd70b274:/data/golang_code/indirectTest#

```



##### go get -u

利用go get升级或者降级某个依赖模块；

**go get -u升级所有依赖模块(直接以及间接)**



#### sub directory go.mod

0.0

#### 解决依赖地狱

包P1依赖包P3 V1.5版本，包P2依赖包P3 V2.0版本，App同时依赖包P1和包P2。于是问题出现了：构建工具在构建App这个应用时需要决策究竟要使用包P3的哪个版本：V1.5还是V2.0？当然这个问题存在的一个前提是：**App中只允许包含包P3的一个版本**。

如果P3的V1.5与V2.0版本是不兼容的版本，那么构建工具无论选择包P3的哪个版本，App的构建都会以失败告终。开发人员只能介入手工解决，解决方法无非是将P1对P3的依赖升级到V2.0，或将P2对P3的依赖降级为V1.5版本。但P1、P2多数情况下是第三方开源包，App的开发人员对P1、P2包的作者的影响力有限，因此这种手工解决的成功率也不高。

Go module构建模式是可以部分解决上述“依赖地狱”问题的。Go module解决这个问题的思路是：**语义导入版本(sematic import versioning)**，**即在包的导入路径上加上包的major version前缀**。

Go通过打破“App中只允许包含包P3的一个版本”这个前提实现了P1和P2各自使用自己依赖的版本。但这样做的前提是P1和P2依赖的P3版本的major版本号是不同的。在Go中，由于采用语义导入版本机制，major版本号不同的module被视为不同的module，即使它们源于同一个repository(比如上面的源于同一个P3的v1.5和v2.0就被视为两个不同的module)。

当然这种解决方案是有代价的！第一个代价就是构建出来的app的二进制文件size变大了，因为二进制文件中包含了多个版本的P3的代码；第二个代价，可能也算不上代价，更多是要注意的是不同版本的module之间的类型、变量、标识符不能混用，我们以[go-redis/redis](https://github.com/go-redis/redis)这个开源项目举个例子。go-redis/redis最新大版本为v8.11.4，没有启用go.mod时的版本为v6.x.x，我们将这两个版本混用在一起：

```go
package main

import (
    "context"

    "github.com/go-redis/redis"
    redis8 "github.com/go-redis/redis/v8"
)

func main() {
    // 混合使用
    var rdb *redis8.Client
    rdb = redis.NewClient(&redis.Options{
        Addr:     "localhost:6379",
        Password: "", // no password set
        DB:       0,  // use default DB
    })
    _ = rdb
}
// Go编译器在编译这段代码时会报如下错误：

cannot use redis.NewClient(&redis.Options{…}) (value of type *"github.com/go-redis/redis".Client) as type *"github.com/go-redis/redis/v8".Client in assignment
//即redis v8下的Clie
```

即redis v8下的Client与redis Client并非一个类型，即使它们的内部字段相同，也不能混用在一起。

##### 采用go module机制的依赖包的冲突问题

这个例子也很简单，P1和P2都依赖module P3，分别依赖P3的v1.1.0版本和v1.2.0版本，和之前的例子一样，App既依赖P1，也依赖P2，这样就构成了图中右侧的“钻石结构”。不过，Go module的另外一个机制：最小版本选择(MVS)可以很好的解决这个依赖问题。

MVS机制同样是基于语义版本之上的，它会选择满足此App依赖的最小可用版本，这里P3的v1.1.0版本与v1.2.0版本的major版本号相同，因此按照语义版本规范，他们是兼容的版本，于是go命令选择了满足app依赖的最小可用版本为v1.2.0(此时P3的最高版本为v1.7.0，Go命令没有选择最高版本)。

不过**理论约定与实际常常脱节**，一旦P3的author在发布v1.2.0时没有识别出与v1.1.0的不兼容change，那么这种情况下，**不兼容v1.1.0的P3版本会导致对P1包的构建失败**！这就是采用go module机制的依赖包时最常见的“依赖地狱”问题。

可以看到，这个问题的根源还是在于go module的author。识别不兼容的change难吗？也不难，但的确繁琐，费脑子。那么作为module author, 如何尽量避免未按照语义版本规范发布版本号兼容实则不兼容的module版本呢？

Go社区的作法分为两类：

- 极端作法：
  - 不发布1.0版本，一直在0.x.y，不承诺新版本兼容老版本；
  - 每次发稍大一些的版本都升级major版本号，这样既避免了繁琐的检查是否有不兼容change的问题，也肯定不会给社区带去“依赖地狱”问题。
- 常规作法：
  - 检查是否有不兼容change，只有在存在不兼容change的情况下，才升级major版本。

Go官博曾经发表过一篇名为[《Keeping Your Modules Compatible》](https://go.dev/blog/module-compatibility)的文章，以告诉大家如何检查新的change中是否有不兼容的change。文中还提到Go团队已经开发了一个名为[gorelease的工具](https://github.com/golang/exp/tree/master/cmd/gorelease)来帮助Go开发者检测新版本与旧版本之间是否存在不兼容的变化(当然，工具也很可能不能完全扫描出来不兼容性change)，大家可以在[这里查看gorelease的详细用法](https://pkg.go.dev/golang.org/x/exp@v0.0.0-20220307200941-a1099baf94bf/cmd/gorelease)。

##### multi version:book:etcd

参考 gay hub etcd即可，client/v2 and client/v3，每个目录下各一个go.mod

看来是通过replace解决菱形依赖了

```bash
# 创建v2 dir
$ mkdir utils/v2
$ cp utils/test.go utils/v2/
$ vi utils/test.go
$ cp go.mod utils/v2/
# The -module flag changes the module's path (the go.mod file's module line).
$ go mod edit -module test.version/utils/v2 utils/v2/go.mod


$ vi go.mod
module test.version
## add replace
replace (
        test.version/utils => ./utils
        test.version/utils/v2 => ./utils/v2
)

go 1.17
## auto-add
$ go mod tidy
go: found test.version/utils/v2 in test.version/utils/v2 v2.0.0-00010101000000-000000000000
$ cat go.mod
module test.version

replace (
        test.version/utils => ./utils
        test.version/utils/v2 => ./utils/v2
)

go 1.17

require test.version/utils/v2 v2.0.0-00010101000000-000000000000

$ go run main.go
test func version 2

```



### 引用

1. https://jlbp.dev/what-is-a-diamond-dependency-conflict
2. https://blog.csdn.net/chuchus/article/details/41481335
3. https://www.zhihu.com/question/375368576
4. https://tonybai.com/2022/03/12/dependency-hell-in-go/
5. https://learnku.com/articles/47737

