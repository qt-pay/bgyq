## 【转】 Sentinel集成Nacos、SpringBoot实现熔断与限流-demo



### Nacos部署

#### 声明

Nacos的官网是我见过最垃圾的了

**当nacos客户端升级为2.x版本后，新增了gRPC的通信方式，新增了两个端口。这两个端口在nacos原先的端口上(默认8848)，进行一定偏移量自动生成。**

| 端口 | 与主端口的偏移量 | 描述                                                       |
| :--- | :--------------- | :--------------------------------------------------------- |
| 9848 | 1000             | 客户端gRPC请求服务端端口，用于客户端向服务端发起连接和请求 |
| 9849 | 1001             | 服务端gRPC请求服务端端口，用于服务间同步                   |

容器部署时，需要多声明两个端口，艹

#### 容器部署

```bash
docker run --name nacos-quick -e MODE=standalone -p 8848:8848 -p 9849:9848 -d nacos/nacos-server:2.0.2
```

注意，不要使用`v2.1.0`垃圾阿里，测这个版本没，shoot



### Sentinel 熔断配置示例:fire:

#### 创建三个springboot应用

创建一个项目，包含ServA、ServB和ServC三个模块

创建三个微服务ServA、ServB、ServC分别运行于8000、9000、10000端口，通过Nacos完成服务注册与链路通信，其中ServA依赖ServB，ServB依赖ServC，ServC需要3秒处理请求

依赖模拟：在执行SerA之前先调用ServB。

```java
//ServA

@RestController
public class TestApi
{
    @Autowired
    RestTemplate restTemplate;

    @RequestMapping("/testMethod")
    public String testMethod()
    {
        System.out.println("微服务a请求微服务b");
        String forObject = restTemplate.getForObject("http://ServB/testMethod", String.class);
        System.out.println("收到微服务b返回："+forObject);
        System.out.println("微服务a执行完毕");
        return "微服务a执行完毕";
    }
}

//ServB

public class TestApi
{
    @Autowired
    RestTemplate restTemplate;

    @RequestMapping("/testMethod")
    public String testMethod()
    {
        System.out.println("微服务b请求微服务c");
        String forObject = restTemplate.getForObject("http://ServC/testMethod", String.class);
        System.out.println("收到微服务c返回："+forObject);
        System.out.println("微服务b执行完毕");
        return "微服务b执行完毕";
    }
}

// ServC

@RestController
public class TestApi
{
    @Autowired
    RestTemplate restTemplate;

    @RequestMapping("/testMethod")
    public String testMethod() throws InterruptedException {
        System.out.println("微服务c开始执行");
        Thread.sleep(3000);
        System.out.println("微服务c执行完毕");
        return "微服务c执行完毕";
    }
}
```

启动服务

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/nacos-server-list.png)

#### 安装Sentinel

容器启动

#### 配置Sentinel

如果微服务所属节点配置了多张网卡或者采用本地虚拟机，Sentinel对微服务的IP识别结果可能不是你所期望的，我们可以通过添加client-ip配置项，告之Sentinel微服务的IP地址；同理Nacos可以添加ip配置项，指定在Nacos注册的IP地址

```yaml
server:
  port: 10000
spring:
  cloud:
    nacos:
      discovery:
        service: ServC
      server-addr: 192.168.6.242:8848
    sentinel:
      transport:
        dashboard: 192.168.6.242:808
        client-ip: 192.168.51.114
```

sentinel的配置非常简洁，我们可以在簇点链路查看微服务注册的路由，根据路由设置响应的规则

##### 调用链数据

测试 微服务是否创建成功，并与Sentinel建立连接

访问微服务A http://ip:8000/testMethod，访问Sentinel控制台,可见3个微服务已被Sentinel捕获

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/sentinel-trace.png)

##### 限流

我们设置微服务C的`/testMethod`接口只允许一个线程访问，

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/sentinel-限流-1.png)

因为服务C的`/testMethod`接口会延时3s后再访问，所以同时发起两个请求时，可以看到限流的效果。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/sentinel限流-2.png)

同时像微服务A发起两个请求，第一个请求正常完成，但第二个请求返回500状态码。

查看微服务B控制台可知，调用微服务C时，请求被Sentinel拒绝，抛出了tooManyRequests异常

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/sentinel-限流-3.png)

##### 熔断

已知微服务C执行时间为3秒，添加一个降级规则，当微服务c执行时间超过1秒时，对微服务C进行熔断，熔断窗口时间10秒。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/sentienl-熔断-1.png)

第一次请求微服务A，可以成功获得响应，同时触发了Sentinel的熔断机制

10秒内，再次访问微服务A时，Sentinel抛出异常

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/sentinel-熔断-2.png)

### Spring Cloud微服务调用客户端

在spring cloud 中有两种服务调用方式，一种是ribbon+restTemplate ，另一种是feign。相对来说，feign因为注解使用起来更简便。而restTemplate需要我们自定义一个RestTemplate，手动注入，并设置成LoadBalance。



#### restTemplate  

##### `ip:port/path`的HTTP请求方式

通过HTTP方式调用REST API接口方式直接请求调用服务的ip地址和接口路径，**此方式类似前端调用后端模式一致**，并没有真实用到微服务的架构，对于所有的第三方服务都可以使用该种方式请求。

`IP:Port`是网关的信息还行，如果是服务提供方的IP和port肯定不行。

```java
//ip:port/path 的请求方式
@RequestMapping("/getConsumerInfo/{id}")
public String getConsumerInfo(@PathVariable("id") String id){
    RestTemplate restTemplate = new RestTemplate();
    MultiValueMap<String,Object> params = new LinkedMultiValueMap<>();
    params.add("id", id);
    String url = "http://127.0.0.1:8090";
    String path = "/getProducerinfo";
    String result = restTemplate.postForObject(producer_prefix + path, params, String.class);
    
    return "hello cloud，we get message：" + result;
}
```

严格的讲这种方式并没有真正使用到微服务方式，且不能支持服务的负载均衡。

因为这种方式需要知道服务提供方的IP，如果服务的ip 或者端口变更了的话，我们就需要修改配置了。虽然可以在yml配置管理，但是，ip 或端口一改，项目中也得跟着修改！ 

##### LocaBalancerClient: 根据服务名获取IP

此方式需要指定提供方注册的服务名进行请求，而负载均衡客户端`LocaBalancerClient`用来获取提供方在注册中心的信息和真实url地址，并使用RestTemplate发起请求。

```java
//LocaBalancerClient + RestTemplate 的请求方式
@Autowired
private LoadBalancerClient loadBalancerClient;

@RequestMapping("/getConsumerInfo/{id}")
public String getConsumerInfo(@PathVariable("id") String id){
    //根据服务名获取服务真实地址
    ServiceInstance service = loadBalancerClient.choose("PRODUCT");
    String url = String.format("http://%s:%s/getProducerinfo",service.getHost(),service.getPort());
    
    RestTemplate restTemplate = new RestTemplate();
    MultiValueMap<String,Object> params = new LinkedMultiValueMap<>();
    params.add("id", id);
    String result = restTemplate.postForObject(url, params, String.class);
    
    return "hello cloud，we get message：" + result;
}
```

##### `@LoadBalanced`： 根据服务名获取IP

@LoadBalanced注解配合RestTemplate注入`这样写更简介。

使用Spring Cloud提供的@LoadBalanced注解配置服务地址自动实现，直接使用服务的注册名称进行请求调用。

此种方式使用方式最简单方便，只需要对RestTemplate进行自定义配置并使用@LoadBalanced注解，并在需要的地方注入RestTemplate对象使用即可，推荐使用该调用方式。

RestTemplate配置信息：

```java
@Configuration
public class RestConfig {

    @Bean
    @LoadBalanced
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
}
```

在调用服务时注入并使用RestTemplate对象

```java
//注入RestTemplate对象进行调用
@Autowired
private RestTemplate restTemplate;
private String producer_prefix = "http://producer";

@RequestMapping("/getConsumerInfo/{id}")
public String getConsumerInfo(@PathVariable("id") String id){
    MultiValueMap<String,Object> params = new LinkedMultiValueMap<>();
    params.add("id", id);
    String result = restTemplate.postForObject(producer_prefix + "/getProducerInfo", params, String.class);
    
    return "hello cloud，we get message：" + result;
}
```

#### feign client

Feign是一种比较规范的微服务调用方式，这也是Spring Cloud Netflix项目提供的服务调用框架。使用Feign进行服务调用时需要额外引入其依赖信息：

```xml
<!-- feign依赖 -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```

Feign使用时，需要在消费模块种定义调用服务的接口类信息，以此作为调用服务的映射服务。

```java
//定义服务提供模块对应服务的映射接口类
@FeignClient(value = "PRODUCER")
public interface ProducerFeign {

    @PostMapping("/getProducerInfo")
    String getProductById(@RequestParam("id") String id);
}
```

然后在控制器种定义Feign调用的逻辑，和调用自身业务逻辑层一致，只需要注入映射接口并执行其中的方法就可以返回结果。

```java
//注入并使用ProducerFeign
@Autowired
private ProducerFeign producerFeign;

@RequestMapping("/getConsumerInfoByFeign/{id}")
public String getConsumerInfoByFeign(@PathVariable("id") String id){
    String result = producerFeign.getProductById(id);
    return "hello cloud，we get message：" + result;
}
```

最后需要在消费者模块启动类上使用`@EnableFeignClients`注解来开启Feign功能。



#### Httpclient

#### Okhttp

#### Httpurlconnection



## 转载

1. https://juejin.cn/post/7035991244312444965
2. https://blog.liboliu.com/a/140
3. https://blog.csdn.net/wangmx1993328/article/details/121189232
4. https://zhuanlan.zhihu.com/p/115403195
5. https://blog.csdn.net/FengGLA/article/details/54869858
6. https://mfrank2016.github.io/breeze-blog/2020/05/04/java/basic/java-package/



