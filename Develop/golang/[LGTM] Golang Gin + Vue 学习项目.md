## [LGTM] Golang Gin + Vue 学习项目

### 原文

https://qingwave.github.io/golang-vue-starter/

### 项目架构

设计思想：

* 前端提供交互页面和操作逻辑
* 后端接口负责调用其他平台的接口实现自动化批量操作和项目自身的一些逻辑及算法
* gitops，提供标准化和自动化的各个运维模块部署、升级、和更新任务

#### 总体目录

```bash
├── Dockerfile
├── Makefile
├── README.md
├── bin
├── config # server配置
├── docs # swagger 生成文件
├── document # 文档
├── go.mod
├── go.sum
├── main.go # server入口
├── pkg # server业务代码
├── scripts # 脚本
├── static # 静态文件
└── web # 前端目录
|__ gitops # 运维自动化模块
```

#### 后端目录：listen 8080

```bash
├── pkg
│   ├── common # 通用包
│   ├── config # 配置相关
│   ├── container # 容器库
│   ├── controller # 控制器层，处理HTTP请求
│   ├── database # 数据库初始化，封装
│   ├── metrics # 监控相关
│   ├── middleware # http中间件
│   ├── model # 模型层
│   ├── repository # 存储层，数据持久化
│   ├── server # server入口，创建router
│   └── service # 逻辑层，处理业务
```



#### 前端目录: listen 8081

试下amis？

```bash
web
├── README.md
├── index.html
├── node_modules
├── package-lock.json
├── package.json
├── public
│   └── favicon.ico
├── src # 所有代码位于src
│   ├── App.vue # Vue项目入口
│   ├── assets # 静态文件
│   ├── axios # http请求封装
│   ├── components # Vue组件
│   ├── main.js
│   ├── router # 路由
│   ├── utils # 工具包
│   └── views # 所有页面
└── vite.config.js # vite配置
```



#### gitops： 模块化

将ansible  playbook或 kustomize 文件放到git，通过web调用，实现复杂的运维操作

项目中留个小工具，把这些复杂的运维操作直接pull下面，然后起个web接口调用就行。

这样可以将 运维自动化功能 和 项目整个功能架构分开，比较清晰，模块化管理。

部分 部署模块升级，也只需要从git上获取最新的ansible playbook即可

gayhub上丰富 ansible资源可以白嫖

### go mod tidy错误

`git clone`后，在本地部署项目，执行`go mod tidy`出现错误

```bash
$ go mod tidy
...
go: downloading git
hub.com/docker/distribution v2.8.0+incompatible
verifying github.com/docker/distribution@v2.8.0+incompatible: checksum mismatch
        downloaded: h1:u9vuu6qqG7nN9a735Noed0ahoUm30iipVRlhgh72N0M=
        go.sum:     h1:l9EaZDICImO1ngI+uTifW+ZYvvz7fKISBAKpg+MbWbY=

SECURITY ERROR
This download does NOT match an earlier download recorded in go.sum.
The bits may have been replaced on the origin server, or an attacker may
have intercepted the download attempt.
```

gayhub大佬说的不是太懂...

 Until then you can downgrade to `v2.8.0-beta.1` and exclude `v2.8.0` as there are next to no changes between that and `v2.8.0`.

操作如下：

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/go-mod-error.jpg)

还有一个解决方法：

The best fix, if the original commit cannot be restored for `v2.8.0+incompatible` would be to publish a new `v2.8.1+incompabtible` release with a `v2.8.1` tag.

#### excloud

`go.mod` 文件中的 `exclude` 指令用于排除某个包的特定版本，其与 `replace` 类似，也仅在当前 module 为 `main module` 时有效，其他项目引用当前项目时，`exclude` 指令会被忽略。

`exclude` 指令在实际的项目中很少被使用，因为很少会显式的排除某个包的某个版本，除非我们知道某个版本有严重 bug。 比如指令 `exclude github.com/google/uuid v1.1.0`，表示不使用 v1.1.0 版本。

 `exclude github.com/google/uuid v1.1.0` 指令后，编译时 `github.com/renhongcai/gomodule` 依赖的 uuid 版本会自动跳过 `v1.1.0`，即选择 `v1.1.1` 版本，相应的 `go.mod` 文件如下所示：

```go
module github.com/renhongcai/gomodule

go 1.13

require (
	github.com/google/uuid v1.1.1
	github.com/renhongcai/exclude v1.0.0
	golang.org/x/text v0.3.2
)

replace golang.org/x/text v0.3.2 => github.com/golang/text v0.3.2

exclude github.com/google/uuid v1.1.0
```

在本例中，在选择版本时，跳过 uuid `v1.1.0` 版本后还有 `v1.1.1` 版本可用，Go 命令行工具可以自动选择 `v1.1.1` 版本，但如果没有更新的版本时将会报错而无法编译

### Swaggo 

 swaggo，根据注释自动生成 API 文档，注释即文档，代替了手动编写Swagger  yaml 的部分。

主要分为以下几个步骤：

- 0）main 文件中添加注释-配置Server，服务信息
- 1）controller 中添加注释-配置接口，接口信息
- 2）swag init 生成 docs 目录
- 3）配置 handler 访问
- 4）访问测试

#### 配置server

```go
# main.go 添加server配置
// @title           Weave Server API
// @version         2.0
// @description     This is a weave server api doc.

// @license.name  Apache 2.0
// @license.url   http://www.apache.org/licenses/LICENSE-2.0.html

// @host      localhost:8080
// @BasePath  /

// @securityDefinitions.apikey JWT
// @in header
// @name Authorization

func main() {
    r := gin.Default()
    c := controller.NewController()
    v1 := r.Group("/api/v1")
    {
        accounts := v1.Group("/accounts")
        {
            accounts.GET(":id", c.ShowAccount)
        }
    //...
    }
    r.Run(":8080")
}
```



#### 每个接口添加注释

```go
// @Summary Get group
// @Description Get group
// @Produce json
// @Tags group
// @Security JWT
// @Param id path int true "group id"
// @Success 200 {object} common.Response{data=model.Group}
// @Router /api/v1/groups/{id} [get]
func (g *GroupController) Get(c *gin.Context) {
    ...
}
```

#####　注释语法：后续补充学习吧

Controller注释支持很多字段：

- **param**：接口请求参数，重要
  - Syntax：`param name`,`param type`,`data type`,`is mandatory?`,`comment` `attribute(optional)`
- **response、success、failure**：API 响应内容，如果成功失败返回值不一样也可以通过 success、failure 分别描述。
  - Syntax：`return code`,`{param type}`,`data type`,`comment`
- end

#### swag init: 根据注释生成docs package

需要先安装 swag，使用如下命令下载swag：

```bash
$ go get -u github.com/swaggo/swag/cmd/swag 
# 1.16 及以上版本
$ go install github.com/swaggo/swag/cmd/swag@latest 
```

在包含`main.go`文件的项目根目录运行`swag init`。这将会解析注释并生成需要的文件（`docs`文件夹和`docs/docs.go`）。

```bash
swag init 
```

注：`swag init` 默认会找当前目录下的 main.go 文件，如果不叫 main.go 也可以手动指定文件位置。

```bash
# -o 指定输出目录。
swag init -g cmd/api/api.go -o cmd/api/docs 
```

需要注意的是：swag init 的时候需要在项目根目录下执行，否则无法检测到所有文件中的注释。

> 比如在 /xxx 目录下执行 swag init 就只能检测到 xxx 目录下的，如果还有和 xxx 目录同级或者更上层的目录中的代码都检测不到。

init 之后会生成一个 docs 文件夹，这里面就是接口描述文件，生成后还需要将其导入到 main.go 中。

```go
// Package docs GENERATED BY SWAG; DO NOT EDIT
// This file was generated by swaggo/swag
package docs

import "github.com/swaggo/swag"
...
```



**在 设置 gin sever 中导入刚才生成的 docs 包**

上面例子中，`import _ "xxx/docs"`  是在main.go，这样场景比较简单。

下面例子gin server是在单独的server.go中初始化的，所以导入如下：

```go
// server.go
import (
 _ "github.com/qingwave/weave/docs"
...
)
func ...{
    gin.SetMode(conf.Server.ENV)

	e := gin.New()
	e.Use(
		gin.Recovery(),
		rateLimitMiddleware,
		...
	)

	e.LoadHTMLFiles("static/terminal.html")

	return &Server{
		engine:          e,
		config:          conf,
		logger:          logger,
		db:              db,
		rdb:             rdb,
		containerClient: conClient,
		kubeClient:      kubeClient,
		controllers:     controllers,
	}, nil
}

// main.go
import (
	"flag"
	"os"

	...
    // 间接导入了docs package
	"github.com/qingwave/weave/pkg/server"
     ...
)
```

end

#### 配置handler访问swagger ui

router 中增加 swagger 的 handler 了。

在 main.go 或其他地方增加一个 handler.

```go
engine := gin.New() 
engine.GET("swagger/*any", ginSwagger.WrapHandler(swaggerFiles.Handler)) 
```

访问测试：

项目运行起来后访问`ip:port/swagger/index.html` 即可看到 API 文档。

### 引用

1. https://www.lixueduan.com/posts/go/swagger/
