## vue -- H3c

### 学习

Scrimba是一个有趣而快速的学习代码的方式！我们的交互式课程和教程将教你React、Vue、Angular、JavaScript、HTML、CSS等等。

https://xue.book118.com/sites/1194.html

https://scrimba.com/scrim/c6ggR3cG?pl=pnyzgAP

暂停了之后可以直接修改当前视频时间段的代码并调试...

### URL and 接口路径

一个URL就是一个接口：接口大致会分为一下几个部分：

http://127.0.0.1:8080/light?opt=open&use=yy&pwd=123456

**1.请求协议：**

   http — 普通的http请求

   https — 加密的http请求，传输数据更加安全

​    ftp — 文件传输协议，主要用来传输文件

**2.请求IP：**就是指提供接口的系统所部署的服务器地址

**3.请求端口：**如果不填端口，默认是80，否则需要填写端口号

**4.接口路径：**指系统提供的接口在什么位置

**5.接口参数：**参数在接口路径后，用“?”来表示路径地址完了，剩下的都是参数了，用“&”来区分参数个数，

#### URI

uri and url--

- URI：在RESTful架构中，每个URI代表一种资源

- URI规范：

  1. 不用大写
  2. 用中杠-，不用下划线_
  3. 路径中不能有动词，只能有名词
  4. 名词表示资源集合，要使用复数形式

- 通过标准HTTP方法对资源进行CRUD（将服务行为映射到标准HTTP动词）

  1. CRUD：增加(Create)、检索(Retrieve)、更新(Update)和删除(Delete)几个单词的首字母简写
  2. GET（SELECT）：从服务器取出/请求资源
  3. POST（CREATE）：在服务器新建资源
  4. PUT（UPDATE）：在服务器更新资源
  5. DELETE：从服务器删除资源

  0.0

### what is Vue？

从目前的前端社区环境来看，vue基本值得就是前端了。现在vue一般都是指vue.js,vue-router,vuex的统称。

#### Vue and JS

vue是js的框架

以前的前端，没有框架、没有工具链，就是写js，顶多用上jQuery。写完就刷新浏览器看看效果。 现在的前端开发越来越复杂，越来越”像开发服务器端“。 首先，有了框架，就比如Vue，遵循类似MVC的架构（vue是mvvm架构？？？），有了组件、依赖注入等服务器端的概念。代码不能随便写，不是简单调用jQuery就行的。 其次，需要脚手架工具(CLI)、Webpack、SCSS\LESS等等一些列工具，涵盖整个开发到测试的工具链。

#### VueJS and NodeJS

node是一个在浏览器外执行JavaScript语言的环境，就好比JRE是Java的运行环境。Vue是尤大写的MVVM前端框架，使用Vue-cli可以快速搭建一个在运行在node上的Vue项目。Element UI就是饿了么团队写的一个前端组建库，可以跟Vue配合使用，类似于jQuery使用jQWidgets。

1. VueJS app is not NodeJS app.

2. VueJS app is interpreted by the browser.

3. You just have to build your app on computer and host files as any static website, so any server can serve html and files.

4. To build your app use e.g. Webpack (https://github.com/vuejs-templates/webpack )

   > 在用 Vue 构建大型应用时推荐使用 NPM 安装[[1\]](https://cn.vuejs.org/v2/guide/installation.html#footnote-1)。NPM 能很好地和诸如 [webpack](https://webpack.js.org/) 或 [Browserify](http://browserify.org/) 模块打包器配合使用。同时 Vue 也提供配套工具来开发[单文件组件](https://cn.vuejs.org/v2/guide/single-file-components.html)。

Vue就一个90多KB的JS文件,你在页面引入 vue.min.js 就能使用Vue进行前端开发,有个IE9或更新版本的浏览器就能正常运行.

一个是基于浏览器端的 javascript （前端 JS）

一个是基于服务端的 javascript （后端 Node.js）

> nodejs是一个很强大的js 运行环境，类似于jvm之于java。

1. 语法一样（同一种语言操作对象不同）
2. 组成不一样

​       JavaScript：

- ECMAScript（语言基础，如：语法、数据类型结构以及一些内置对象）
- DOM（一些操作页面元素的方法）
- BOM（一些操作浏览器的方法）

​    Node.js：

- ECMAScript（语言基础，如：语法、数据类型结构以及一些内置对象）
- OS（操作系统）
- file（文件系统）
- net（网络系统）
- database（数据库）

浏览器通过内置一个Js引擎可以解释执行js代码，浏览器实现js标准规范（ECMAScript），提供API，可以使用JavaScript调用，支持我们操作页面DOM，调用某些浏览器操作等客户端功能，也可以说JavaScript = ECMAScript + DOM + 宿主环境（BOM），这就是所说的前端js；

而node.Js是一个基于Google V8引擎（js引擎的某一种）构建的JavaScript运行环境，提供API，可以使用JavaScript调用，支持我们读取本地文件，开启本地主机端口服务，发送，接受请求等服务端功能，node.JS = JavaScript + node环境(提供node API...)；

所以，如果要执行JavaScript代码，需要一个js引擎，你可以安装一个浏览器（内置js引擎），或nodejs环境（内置js引擎），如果你在js代码中调用了浏览器提供的API，则必须安装一个浏览器，若调用了nodejs API，则必须安装nodejs环境，然后以各自规定的方式加载、执行JavaScript代码。e。

end

### vue 构建

npm构建生成的默认目标目录就是`./dist`

#### npm传参构建

小野哥，通过docker build env 传入不同的env，使用不同的配置文件，构建出测试和生产不同的镜像。

主要是，数据库和调用其他组件的配置信息不同。

#### npm run 

npm 脚本的原理非常简单。每当执行`npm run`，就会自动新建一个 Shell，在这个 Shell 里面执行指定的脚本命令。因此，只要是 Shell（一般是 Bash）可以运行的命令，就可以写在 npm 脚本里面。

当前目录的`node_modules/.bin`子目录里面的所有脚本，都可以直接用脚本名调用，而不必加上路径。比如，当前项目的依赖里面有 Mocha，只要直接写`mocha test`就可以了。

> ```javascript
> "test": "mocha test"
> ```

而不用写成下面这样。

> ```javascript
> "test": "./node_modules/.bin/mocha test"
> ```

这个小设计挺好--

```bash
# npm run-script <command> [-- <args>...]
# aliases: run, rum, urn
$ cat package.json
{
  "name": "test",
  "version": "1.0.0",
  "description": "",
  "main": "hello.js",
  "scripts": {
    "hi": "echo \"ok?\"",
    "test": "echo \"Error: no test specified\" && exit 1",
    "dodo": "echo \"dodo\"",
    "docs:test": "echo 2333",
    ## 0.0 真正的build操作--
    "build": "vue-cli-service build"
  },
  "author": "",
  "license": "ISC",
  "devDependencies": {
    "autoprefixer": "^7.1.2"
  },
  "dependencies": {
    "build": "^0.1.4"
  }
}
## 这就是个 script_name
$ npm run docs:test

> test@1.0.0 docs:test /tmp/projects
> echo 2333

2333

```

end

#### NVM: multi-env

nvm is a version manager for [node.js](https://nodejs.org/en/), designed to be installed per-user, and invoked per-shell. `nvm` works on any POSIX-compliant shell (sh, dash, ksh, zsh, bash), in particular on these platforms: unix, macOS, and [windows WSL](https://github.com/nvm-sh/nvm#important-notes).

> Node version managers allow you to install and switch between multiple versions of Node.js and npm on your system so you can test your applications on multiple versions of npm to ensure they work for users on different versions.

`nvm` allows you to quickly install and use different versions of node via the command line.

**Example:**

```bash
$ nvm use 16
Now using node v16.9.1 (npm v7.21.1)
$ node -v
v16.9.1
$ nvm use 14
Now using node v14.18.0 (npm v6.14.15)
$ node -v
v14.18.0
$ nvm install 12
Now using node v12.22.6 (npm v6.14.5)
$ node -v
v12.22.6
```

Simple as that!

### vue 调用后端服务：vue仅处理DOM，

核心： vue定义了要访问的后端接口`url_path`，并通过Axios实例发起http调用。

至于怎么将`url_path`代理到后端服务，就需要借助代理服务器或者网关了。

常见：

* vue prxoy： 开发环境常用
* kong： k8s环境常用或者微服务架构
* nginx：虚拟机常用

**vue-cli3 脚手架搭建完成后，项目目录中没有 vue.config.js 文件，需要手动创建**

#### `.env` and `.env.XXX`

https://cli.vuejs.org/zh/guide/mode-and-env.html

**模式**是 Vue CLI 项目中一个重要的概念。默认情况下，一个 Vue CLI 项目有三个模式：

- `development` 模式用于 `vue-cli-service serve`
- `test` 模式用于 `vue-cli-service test:unit`
- `production` 模式用于 `vue-cli-service build` 和 `vue-cli-service test:e2e`

注意：.env文件无论是开发还是生成都会加载的公用文件

根据启动命令vue会自动加载对应的环境，vue是根据文件名进行加载的，比如执行npm run serve命令，会自动加载.env.development文件

属性名必须以`VUE_APP_`开头，比如`VUE_APP_URL` 、 `VUE_APP_XXX`

```bash
$ ls .env.*
.env .env.development .env.production
$ cat .env.production
#生产环境
NODE_ENV="production"

# api 数据是否加密
VUE_APP_API_ENCRYPT = true

# 因webseal没有开启websocket代理，这里采用企业微信地址代理
VUE_APP_MESSAGE_API = '/daip-message'

# qiankun应用入口
VUE_APP_QIANKUN_APIWEB_ENTRY = '//d.wiseda.cn/waf-apiweb/'

```

end

#### Vue调用其他前端接口

```vue
// window.open()用于打开一个新的浏览器窗口（浏览器被拦截）
    goHelpDoc() {
      window.open(`/daip-docs/`)
    }

===
<td class="bottom-list" @click="goHelpDoc">
    
```

然后另一个前端应用 daip-help-docs 只需要监听端口，并且在api网关上注册/daip-docs接口即可完成调用。

这里还是需要借助 服务网关或者Nginx等代理实现的。



#### vue + axios + vue proxy：重要

解决: 将vue要掉的后端接口通过proxy指向测试环境的后端地址。

https://www.cnblogs.com/code-duck/p/13414099.html

Axios 是一个基于 promise 的 HTTP 库，可以用在浏览器和 node.js 中。

axios前端通信框架，因为vue的边界很明确，就是为了处理DOM，所以并不具备通信功能，此时就需要额外使用一个通信框架与服务器交互；当然也可以使用jQuery提供的AJAX通信功能。

```bash
##  vue调用后端api使用import axios from 'axios';
#环境变量,可在VUE页面中通过process.env.VUE_APP_?调用, 必须以VUE_APP_开头
# 基础框架上下文地址+命名空间
VUE_APP_BASE_API = '/daip-server/waf/'

# 创新应用平台-主应用地址
VUE_APP_DAIP_BASE_API = '/daip-server/'

# 创新应用平台-全文检索地址
VUE_APP_DAIP_ES_BASE_API = '/daip-search/'

# 上下文路径
VUE_APP_PUBLICPATH = '/daip-web/'
```

Axios特性：

1、可以在浏览器中发送 XMLHttpRequests
2、可以在 node.js 发送 http 请求
3、支持 Promise API
4、拦截请求和响应
5、转换请求数据和响应数据
6、能够取消请求
7、自动转换 JSON 数据
8、客户端支持保护安全免受 XSRF 攻击

Vue-Axios实现跨域请求

`.env`配置文件

```js
VUE_APP_BASE_API=/server
```

`request.js`

```js
import axios from 'axios'
const test = axios.create({
  baseURL: process.env.VUE_APP_BASE_API, // api 的 base_url
  timeout: 50000, // request timeout
  headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
  withCredentials: true
})

export default test
```

创建api请求接口

```js
import request from './request.js'

/**
 * 查询所有用户信息
 */
export function list () {
  return request({
    url: '/all/users',
    method: 'get'
  })
}
```

**配置`vue.config.js`代理请求**

```js
module.exports = {
  devServer: {
    port: 8000, // 改端口号
    open: true,
    proxy: {
      // 以server开头的请求才会使用到该代理，即http://localhost:8000/server/query/users.
      '/server': {
        target: 'http://localhost:8081/', // 服务器地址
        // 如果要代理 websockets
        ws: true,
        changeOrigin: true, // 开启跨域
        pathRewrite: {
           // 当请求以/server开头时，将其置为空 则请求最终为http://localhost:8081/query/users
          '/server': '' 
        }
      }
    }
  }
}
```

end

**前端访问地址为：**http://localhost:8000/server/query/users

**会被代理解析为：**http://localhost:8081/query/users 访问到服务器端获取数据

即本地项目运行端口为8000，访问`IP:8000/server`会代理到`IP/8081/`.

0.0 Vue自带的反向代理。

#### vue + axios + k8s+ kong/nginx：总结

这其实也是微服务架构了，因为这里利用了k8s service机制做服务发现。kong可以通过service访问到k8s pod。

生产环境： vue定义后端接口的url，然后通过网关kong或者nginx实现后端的访问。

1. 定义后端接口url即`VUE_APP_XXX=xxx`

2. 创建axios访问后端接口即当前IP_Port+url

3. 根据后端url去访问接口即要保证vue监听的IP/域名加上vue定义的url可以访问到后端接口。

   到这里就简单了，这里可以是kong网关上注册的接口，可以是nginx配置的反向代理，甚至可以是vue proxy自己指定的代理。

k8s + kong环境是：先将vue要调用的后端服务在kong上配置，然后就可以访问了。因为vue代码里指明了要访问的url。

kong负责了将解析代理vue要访问的后端url。

1. vue在kong上注册了`/daip-server`，且代码定义了要访问`/daip-job`
2. daip-job组件在kong上注册了`/daip-job`服务，即可

nginx：

1. nginx 代理了vue 服务`location /daip-server`
2. nginx 代理了daip-job 服务`location /daip-job`

##### 小问题

vue proxy 和 kong都配置了，先生效哪个...

