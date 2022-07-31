# 【转】SpringBoot集成Nacos2.0.3

## 一、搭建nacos-server2.0.3

### 下载编译好的包

```shell
COPYcd /opt

wget https://github.com/alibaba/nacos/releases/download/2.0.3/nacos-server-2.0.3.tar.gz

tar -xvf nacos-server-2.0.3.tar.gz

cd nacos/bin
```

### 启动命令

```shell
COPY# Linux/Unix/Mac
# 启动命令(standalone代表着单机模式运行，非集群模式):

sh startup.sh -m standalone

# 如果您使用的是ubuntu系统，或者运行脚本报错提示[[符号找不到，可尝试如下运行：

bash startup.sh -m standalone

# Windows
# 启动命令(standalone代表着单机模式运行，非集群模式):

startup.cmd -m standalone
```

### Centos7下启动提示

```shell
COPY[root@localhost bin]# sh startup.sh  -m standalone
/usr/java/jdk1.8/bin/java -Djava.ext.dirs=/usr/java/jdk1.8/jre/lib/ext:/usr/java/jdk1.8/lib/ext  -Xms512m -Xmx512m -Xmn256m -Dnacos.standalone=true -Dnacos.member.list= -Xloggc:/opt/nacos2.0.3/nacos/logs/nacos_gc.log -verbose:gc -XX:+PrintGCDetails -XX:+PrintGCDateStamps -XX:+PrintGCTimeStamps -XX:+UseGCLogFileRotation -XX:NumberOfGCLogFiles=10 -XX:GCLogFileSize=100M -Dloader.path=/opt/nacos2.0.3/nacos/plugins/health,/opt/nacos2.0.3/nacos/plugins/cmdb -Dnacos.home=/opt/nacos2.0.3/nacos -jar /opt/nacos2.0.3/nacos/target/nacos-server.jar  --spring.config.additional-location=file:/opt/nacos2.0.3/nacos/conf/ --logging.config=/opt/nacos2.0.3/nacos/conf/nacos-logback.xml --server.max-http-header-size=524288
nacos is starting with standalone
nacos is starting，you can check the /opt/nacos2.0.3/nacos/logs/start.out
```

### 关闭服务器

```shell
COPY# Linux/Unix/Mac
sh shutdown.sh

# Windows
shutdown.cmd

# 或者双击shutdown.cmd运行文件
# 或者kill -15 进程id
```

### 请求测试

```shell
COPY[root@localhost bin]# curl -v  http://127.0.0.1:8848/nacos
* About to connect() to 127.0.0.1 port 8848 (#0)
*   Trying 127.0.0.1...
* Connected to 127.0.0.1 (127.0.0.1) port 8848 (#0)
> GET /nacos HTTP/1.1
> User-Agent: curl/7.29.0
> Host: 127.0.0.1:8848
> Accept: */*
> 
< HTTP/1.1 302 
< Location: http://127.0.0.1:8848/nacos/
< Transfer-Encoding: chunked
< Date: Wed, 20 Oct 2021 01:27:10 GMT
< 
* Connection #0 to host 127.0.0.1 left intact
```

默认访问地址和密码

```bash
COPYhttp://127.0.0.1:8848/nacos

nacos/nacos
```

### VMware/Docker设置

#### **添加端口映射**

容器或者vm部署Nacos时，需要额外开放9848 9849端口。

> 当nacos客户端升级为2.x版本后，新增了gRPC的通信方式，新增了两个端口。这两个端口在nacos原先的端口上(默认8848)，进行一定偏移量自动生成.
>
> 端口 与主端口的偏移量 描述
> 9848 1000 客户端gRPC请求服务端端口，用于客户端向服务端发起连接和请求
> 9849 1001 服务端gRPC请求服务端端口，用于服务间同步等



## Nacos用处

#### 配置中心

将Nacos作为配置中心，应用启动时可以从Nacos获取相应的配置信息，类似Zest

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/nacos-config-center.png)

#### 注册中心

不同组件将自己提供的服务注册到Nacos的服务列表，用于消费者访问使用

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/nacos-discovery.png)





## 二、Nacos简单使用



### 2.1 新建命名空间

新建一个命名空间，待会儿，我们整合时就使用这个名为`springboot`的命名空间

### 2.2 新增配置

#### Data ID

在 Nacos Spring Cloud 中，`dataId` 的完整格式如下：

```yaml
${prefix}-${spring.profiles.active}.${file-extension}
```

- `prefix` 默认为 `spring.application.name` 的值，也可以通过配置项 `spring.cloud.nacos.config.prefix`来配置。
- `spring.profiles.active` 即为当前环境对应的 profile，详情可以参考 [Spring Boot文档](https://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-profiles.html#boot-features-profiles)。 **注意：当 `spring.profiles.active` 为空时，对应的连接符 `-` 也将不存在，dataId 的拼接格式变成 `${prefix}.${file-extension}`**
- `file-exetension` 为配置内容的数据格式，可以通过配置项 `spring.cloud.nacos.config.file-extension` 来配置。目前只支持 `properties` 和 `yaml` 类型。

#### Group

默认为DEFAULT_GROUP,可以对不同类型的微服务配置文件进行分组管理。配置文件通过，可以用作多环境、多模块、多版本之间区分配置。
SpringCloud`spring.cloud.nacos.config.group=GROUP`

SpringBoot`nacos.config.group=GROUP`

#### Namespace

推荐使用命名空间来区分不同环境的配置，因为使用`profiles`或`group`会是不同环境的配置展示到一个页面，而Nacos控制台对不同的`Namespace`做了Tab栏分组展示

yaml文件如下：

```yaml
server:
  port: 8082
  servlet:
    context-path: /demo
    session:
      timeout: 7200s
  # 开启gzip
  compression:
    enabled: true
    mime-types: application/json,application/xml,text/html,text/xml,text/plain,application/javascript,text/css,text/javascript
    min-response-size: 2KB
```

添加完成，为后面整合做准备

## 三、SpringBoot整合Nacos2.0.3

> [官方文档]([Nacos Spring Boot 快速开始](https://nacos.io/zh-cn/docs/quick-start-spring-boot.html))
>
> [Nacos 2.0.3版本发布，继续提升集群稳定性及升级稳定性](https://nacos.io/zh-cn/blog/2.0.3-release.html)
>
> [官方demo](https://github.com/nacos-group/nacos-examples/tree/master/nacos-spring-boot-example)（是个辣鸡吧，官方demo）

注意官方博客中的【客户端支持2.0】，所以这次我们整合的的包应该是0.2.10，因为版本 [0.2.x.RELEASE](https://mvnrepository.com/artifact/com.alibaba.boot/nacos-config-spring-boot-starter) 对应的是 Spring Boot 2.x 版本，版本 [0.1.x.RELEASE](https://mvnrepository.com/artifact/com.alibaba.boot/nacos-config-spring-boot-starter) 对应的是 Spring Boot 1.x 版本。

### 3.1 注意事项

#### 版本问题

整合过程中发现nacos客户端在初始化`NacosBootConfigurationPropertiesBinder`时，如果spring-boot版本太高，可能报错，本人从2.5.x降级为2.3.12.RELEASE才正常

```bash
nested exception is java.lang.NoClassDefFoundError: org/springframework/boot/context/properties/ConfigurationBeanFactoryMetadata
```

#### 服务器读取nacos配置出错

如果nacos的配置中有中文 直接运行jar可能会报错 编码不同导致的

```java
org.yaml.snakeyaml.error.YAMLException: java.nio.charset.MalformedInputExcept
```

解决方案是 java 命令增加 编码选项

```shell
-Dfile.encoding=utf-8 
```

#### 整合其他cloud的客户端

如果你并不是使用`nacos-spring-boot v-x.x.x`而是如下配置

```xml
         <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
            <version>${nacos.version}</version>
        </dependency>
        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
            <version>${nacos.version}</version>
        </dependency>
```

注意bootstrap.yml不生效的问题，可通过以下方式解决

在启动时增加环境变量的配置

```ini
spring.cloud.bootstrap.enabled = true
```

或者

```ini
spring.config.use-legacy-processing = true
```

### 3.2 配置管理

demo项目地址：https://github.com/xucux/nacos-config-discovery-demo/tree/main/nacos-config-discovery-springboot

**注意：我这边没有配置启动类中的`@NacosPropertySource(dataId = "example", autoRefreshed = true)`这个注解**

**application.yml**

```yaml
COPYspring:
  application:
    name: nacos-config-discovery
  profiles:
    active: dev
  config:
    use-legacy-processing: true
```

**application-dev.yml**

`namespace: 24373c8c-3894-419b-83eb-b6b9433a9721`就是上文中创建的名称为`springboot`的命名空间

```yaml
COPYnacos:
  discovery:
    server-addr: 192.168.122.22:8848
    username: nacos
    password: nacos
  config:
    server-addr: 192.168.122.22:8848
    username: nacos
    password: nacos
    namespace: 24373c8c-3894-419b-83eb-b6b9433a9721
    data-id: nacos-config-discovery-dev.yaml
    auto-refresh: true
    group: TEST_GROUP
    type: yaml
    bootstrap:
      enable: true
      log-enable: true
```

启动项目，控制台输出，服务列表，存在新注册的服务名`nacos-config-discovery`

### 3.3 服务注册

使用注解@PostConstruct，在服务启动后自动向Nacos服务注册

```java
@Configuration
public class NacosRegisterConfiguration {

    @Value("${server.port}")
    private int serverPort;

    @Value("${spring.application.name}")
    private String applicationName;

    @NacosInjected
    private NamingService namingService;

    @PostConstruct
    public void registerInstance() throws NacosException {
        namingService.registerInstance(applicationName, "127.0.0.1", serverPort, "DEFAULT");
    }
}
```

启动应用后，在Nacos管理界面可以看到新注册的服务实例

### 3.4 服务调用

模拟调用获取学生信息

创建一个项目`nacos-config-discovery-producer`

详细工程地址：https://github.com/xucux/nacos-config-discovery-demo

主要使用`nacos-config-discovery-springboot`调用`nacos-config-discovery-producer`

启动两个工程，可以发现nacos上已经出现了两个服务：

在`nacos-config-discovery-springboot`中创建一个Controller，作为对外api和接口消费者

```java
@Slf4j
@RestController
@RequestMapping("/student")
public class StudentController {

    @NacosInjected
    private NamingService namingService;


    private RestTemplate restTemplate = new RestTemplate();


    /**
     * 获取学生信息
     * @param id
     * @return
     */
    @GetMapping("/info/{id}")
    public Object getStudentInfo(@PathVariable("id")Integer id){
        Map<String,Object> info = new HashMap<>();
        info.put("id",id);
        info.put("info",queryStudentInfo(id));

        return info;
    }

    private JSONObject queryStudentInfo(Integer id) {
        try {
            if (namingService != null) {
                // 选择user_service服务的一个健康的实例（可配置负载均衡策略）
                Instance instance = namingService.selectOneHealthyInstance("nacos-producer-service");
                // 拼接请求接口url并请求选取的实例
                String url = "http://" + instance.getIp() + ":" + instance.getPort() + "/demo/student/detail/"+id;
                ResponseEntity<JSONObject> entity = restTemplate.getForEntity(url, JSONObject.class);
                return entity.getBody();
            }
        } catch (Exception e) {
            log.error("查询学生数据失败", e);
        }
        return null;
    }

}
```

在`nacos-config-discovery-producer`中创建一个Controller，作为服务生产者

```java
@Slf4j
@RestController
@RequestMapping("/student")
public class StudentController {

    /**
     * 获取学生信息
     * @param id
     * @return
     */
    @GetMapping("/detail/{id}")
    public Object getStudentInfo(@PathVariable("id")Integer id){
        Map<String,Object> info = new HashMap<>();
        info.put("id",id);
        info.put("userName","test");
        info.put("nickName","小明");
        return info;
    }


}
```

调用接口

```shell
COPY[root@localhost bin]# curl -X GET http://192.168.1.212:8082/demo/student/info/1
{"id":1,"info":{"nickName":"小明","id":1,"userName":"test"}}
```

> 博客内容遵循 署名-非商业性使用-相同方式共享 4.0 国际 (CC BY-NC-SA 4.0) 协议

## 原文

https://xu.vercel.app/post/2021/10/20/ebe0fd672cfa55b6/