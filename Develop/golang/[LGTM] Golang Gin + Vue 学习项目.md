## [LGTM] Golang Gin + Vue 学习项目

### 原文

https://qingwave.github.io/golang-vue-starter/

### 项目架构

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
```

#### 后端目录

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



#### 前端目录:试下amis？

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

