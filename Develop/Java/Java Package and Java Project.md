## Java Package and Java Project

### java class

java class name 必须和 java file name保持一致。



### java package:fire:

`package` 的一个很重要的作用，为类提供一个类似命名空间的管理，避免同名的类产生冲突。

`package` 的命名一般分为几个部分，`身份标识.开发者名/团队名/公司名.项目名.模块名`，对于身份标识，主要是用来标识是个人开发还是团队开发，个人开发主要使用的标识有：*indi*（个体项目）、*onem*（单人项目）、*pers*（个人项目）、*priv*（私有项目），团队开发主要使用的标识有：*team*（团队项目）、*com*（公司项目）。

但其实没有特殊要求的话，使用域名倒写是最常见的，因为域名是不会重复的

**在java中，如果在同一个包的类互相引用，是不是就不用import?**

答：如果所谓的"包"是指package,那么确实不需要import,

​    如果所谓的包是指一个jar包，那么不管是否在同一个jar中，那么都是需要import关键字的

若类在不同的package中，那么在一个类中要调用另一个package中的类（必须是public类，非public类不支持不同包间访问），需要在类名前明确加上package名称；不过，java中存在一个让java程序员偷懒的特性，叫做import关键字。使用import就可以在一个package中导入另一个package中的类，不过import和C语言和C++中的#include是不同的，import并不会在当前java文件中嵌入另一个package中的类的代码，只是告诉java文件，不属于该包的类能够到哪里去寻找而已：
**classpath是java在编译程序时查找类文件的路径，java编译器会在classpath中包含有的路径中查找java的类文件。**

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/java-package-name.jpg)

创建package时，命名觉得了目录层级

在使用package的时候，**如果java文件中使用了package，那么该java文件必须放在命名与package名称相同的目录下**
java解释器会将package中的.解释为目录分隔符/，也就是说该文件的目录结构为：`...com/example/springbootnacosp/controller/TestController.java`

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/java-package-name-and-dir.png)

### SpringBoot多模块项目

**使用阿里云镜像地址`https://start.aliyun.com`替换`https://start.spring.io`**

避免拉取springboot脚手架依赖异常

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/replace-start-spring-io.png)

#### 项目结构

Spring Boot典型项目结构为例，创建出来的项目应该总体分为三大层：

- `项目根目录/src/main/java`：放置项目Java源代码
- `项目根目录/src/main/resources`：放置项目静态资源和配置文件
- `项目根目录/src/test/java`：放置项目测试用例代码

而位于`/src/main/java`目录下的Java源代码的组织结构大家比较关心，这地方也只能给出一个通常典型的结构，毕竟不同项目和团队实践不一样，稍许有区别，但整体安排应该差不多。而且如果是**多模块**的项目的话，下面的结构应该只对应其中一个模块，其他模块的代码组织也大致差不多。

```bash
|_annotation：放置项目自定义注解
|_aspect：放置切面代码
|_config：放置配置类
|_constant：放置常量、枚举等定义
   |__consist：存放常量定义
   |__enums：存放枚举定义
|_controller：放置控制器代码
|_filter：放置一些过滤、拦截相关的代码
|_mapper：放置数据访问层代码接口
|_model：放置数据模型代码
   |__entity：放置数据库实体对象定义
   |__dto：存放数据传输对象定义
   |__vo：存放显示层对象定义
|_service：放置具体的业务逻辑代码（接口和实现分离）
   |__intf：存放业务逻辑接口定义
   |__impl：存放业务逻辑实际实现
|_utils：放置工具类和辅助代码
```

如果从一个用户访问一个网站的情况来看，对应着上面的项目代码结构来分析，可以贯穿整个代码分层：

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/springboot-arch.jpg)

**每当拿到一个新的项目到手时，只要按照这个思路去看别人项目的代码，应该基本都是能理得顺的**。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/springboot-arch-2.jpg)



#### 创建父项目

1、创建父工程：File -> New -> Project...，选择 Spring Initializr -> Next -> 填写项目信息 -> Next -> 依赖先不用选择 -> Next -> Finish。

2、删除父项目中无用的 .mvn 目录、src 目录、mvnw 及 mvnw.cmd 文件。

**mvnw**
全名是maven wrapper，保证使用Maven版本一致的工具。
**.mvn**
用于存放maven-wrapper.properties和相关jar包。
**mvn.cmd**
执行mvnw命令的cmd入口。

3、修改父项目 pom.xml 文件，多模块项目架构时，父项目打包类型 packaging 必须指定为 pom。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/parent-project-pom.png)

#### 创建子项目

1、选择父项目右键 -> New -> Module，创建三个子工程 模块：servA、servB、servC

2、子工程全部使用 Spring Initializr  的方式进行创建，下面以 servA为例进行演示，其余的也是同理。创建的同时可以指定依赖 比如可以指定数据库依赖，web 模块可以指定 web 依赖，也可以创建后再添加依赖。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/initialize-springboot-with-web-module.png)

3、修改子模块中的父工程，删除默认的 spring-boot-starter-parent，改为自己的父项目，子模块中删除默认的 relativePath 标签，此时默认是 ../pom.xml，会从本地路径中获取 parent 的 pom。

```xml
	<parent>
        <groupId>com.example</groupId>
        <artifactId>multi-project</artifactId>
        <version>0.0.1-SNAPSHOT</version>
        <!--子模块中删除默认的 relativePath 标签-->
    </parent>
```

4、修改父项目 pom.xml 文件，添加 modules 指定子模块：

```xml
    <!--多模块架构时，父项目必须是 pom 类型-->
    <packaging>pom</packaging>
    <!--指定子模块-->
    <modules>
        <module>servA</module>
        <module>servB</module>
        <module>servC</module>
    </modules>
```

5、增加子模块间依赖，类似下面servA模块依赖servB

```xml
<dependencies>
     <dependency>
         <groupId>com.example</groupId>
         <artifactId>ServB</artifactId>
         <version>0.0.1-SNAPSHOT</version>
     </dependency>
</dependencies>

```

#### 修改启动配置

1、删除 web 模块以外的启动类和 application.properties 配置文件，整个项目只需要一个启动类和一个配置文件。

2、删除子模块中默认的 spring-boot-starter、spring-boot-starter-test 依赖，因为它们在父项目中已经存在了。

3、添加子模块的依赖，basic-controller 依赖 basic-server，basic-server 依赖 basic-dao。

4、多模块项目中并不是每一个子模块都需要打成可执行的 Jar 包，通常只是 web 模块才需要，所以需要将 web 模块外的启动类删了，同时修改 web 模块外的项目(包括父项目)，删除其中的打包插件 spring-boot-maven-plugin。

#### demo

如下，最简单是spring boot demo

```java
<!-- main/java/com.example.serva/ServAApplication.java -->
// sprint boot 默认web server 启动入口
package com.example.serva;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
// java class name and java file name is same.
public class ServAApplication {

    public static void main(String[] args) {
        SpringApplication.run(ServAApplication.class, args);
    }

}

// 手动添加 http path and function的映射
<-- main/java/com.example.serva/controller/TestController.java --!>
package com.example.serva.controller;

import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class TestController
{
    @RequestMapping("/")
    public String hello()
    {
        return "Hello web";
    }
}    
```

end

### 引用

1. https://juejin.cn/post/7035991244312444965
2. https://blog.liboliu.com/a/140
3. https://blog.csdn.net/wangmx1993328/article/details/121189232
4. https://zhuanlan.zhihu.com/p/115403195
5. https://blog.csdn.net/FengGLA/article/details/54869858
6. https://mfrank2016.github.io/breeze-blog/2020/05/04/java/basic/java-package/

