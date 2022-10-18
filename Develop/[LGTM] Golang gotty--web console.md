## [LGTM] Golang gotty--web console

### 目的

给k8s 集群的pod 提供一个 web console可以便捷查看容器信息

### gotty方案

gotty 核心 ：GoTTY 是一个简单的命令行工具，可将CLI工具转换为web应用程序。

最省心，在每个k8s集群上部署一个gotty pod即可

如下：

```bash
## 启动k8s terminal
gotty --permit-arguments kubectl --kubeconfig ./test.conf
## 传入 pod name 和 namespace即可
http://127.0.0.1:8080/?arg=exec&arg=-it&arg=nginx-demo-5b79dc546f-cs6fz&arg=-n&arg=web&arg=bash

## 类似
kubectl --kubeconfig ./test exec -it nginx-demo-5b79dc546f-cs6fz -n web bash
```

这样有个很大的问题，就是安全行，上面的安装上面的 规则任意一个人都可以通过他看到的pod name 和 namespace

### 优化

#### jwt

gotty引入jwt，将传入的参数转成jwt实现加密性

```bash
gotty_server_ip + "/" + "?jwt={Token}"
```

改造工作：

* gotty handerfunc 添加对自定义的jwt的解析
* gotty 启动时可以指定jwt secret 然后用于解析jwt，验证jwt信息，创建ws web console

```go
// gotty 增加启动 option


// http handler 处理自定义的jwt token
func (server *Server) processWSConn(ctx context.Context, conn *websocket.Conn) error {
}
```



#### agent: 集中式和分布式

一个k8s集群使用demonset每个节点一个agent还是一个集群一个agent？

#### 审计功能

Web Console给用户进入容器提供了便利，用户可以执行任何操作，同时为了安全，记录下用户的操作也非常有必要。

这里最简单的方法就是从命令进程stdout中读取到内容，通过websocket返回的同时，也输出到一个日志文件中，

```go
// 查看gotty源码在 localcommand 输出到ws connect时，捎带写入到本地磁盘一份
```

日志文件可以根据自身业务规则定义文件名，方便检索。

#### 残留进程清理

在调试过程中，曾出现一个问题，在进入到容器，进行一系列操作后，如果使用exit退出，则一切正常，**但是如果直接关闭掉浏览器网页，最后会发现连接到容器中的shell进程没有退出（默认使用exec -- bash，多次关闭页面后会导致积累很多bash进程）**，会一直存在，慢慢积累，残留进程就会越来越多。

这里采用的解决办法是，在连接到容器后，增加一步初始操作，将当前shell的进程id保存到一个文件中，在监测到连接关闭后(不管是正常的关闭还是任何异常关闭)，执行清理工作。

### 实现

#### go cli

https://github.com/urfave/cli

cli is a simple, fast, and fun package for building command line apps in Go. The goal is to enable developers to write fast and distributable command line applications in an expressive way.

#### struct tag: 有用？

```go
type Options struct {
	Address             string           `hcl:"address" flagName:"address" flagSName:"a" flagDescribe:"IP address to listen" default:"0.0.0.0"`
	Port                string           `hcl:"port" flagName:"port" flagSName:"p" flagDescribe:"Port number to liten" default:"8080"`
	PermitWrite         bool             `hcl:"permit_write" flagName:"permit-write" flagSName:"w" flagDescribe:"Permit clients to write to the TTY (BE CAREFUL)" default:"false"`
	EnableBasicAuth     bool             `hcl:"enable_basic_auth" default:"false"`
    ...
}


// hcl is a package for decoding HCL into usable Go structures.
//
// hcl input can come in either pure HCL format or JSON format.
// It can be parsed into an AST, and then decoded into a structure,
// or it can be decoded directly from a string into a structure.
//
// If you choose to parse HCL into a raw AST, the benefit is that you
// can write custom visitor implementations to implement custom
// semantic checks. By default, HCL does not perform any semantic
// checks.
package hcl

```

#### localcommand

应该是这里解析转换？

#### go 1.17 编译 vendor项目



### 引用

1. https://cloud.tencent.com/developer/article/1416063