## k8s 灰度发布

灰度发布两个核心点：流量管理、发布流程管理。先来分析流量管理，不算Istio等组件新引入的流量管理策略，仅以Kubernetes本身的组件来看。一个常见的后端服务具有如下网络拓扑结构：ingress作为流量入口、转发到Service、Service转发到Deployment下辖的RS管理的Pod（严格来说，Service背后就直接是Pod了，但此处为方便说明控制关系，将Deployment和RS也加入了其中）。

在于蓝绿发布强调蓝绿环境（即新版和旧版服务）的平等性，流量拆分粒度较粗；金丝雀发布拥有更细粒度的流量拆分。



![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/k8s-canary.png)



### 按Service拆流量

1. 发布新应用的Deployment、Service资源
2. 利用Nginx Ingress Controller的`nginx.ingress.kubernetes.io/canary`系列注解或`nginx.ingress.kubernetes.io/service-match`系列注解控制流量在新旧Service之间的分配
3. 测试新服务
4. 修改流量分配，将流量全都导入新服务，关闭旧服务

> 这种方式能够在七层进行流量控制，粒度可以任意细

#### Nginx ingress实现：依赖域名

**依赖：依赖外部DNS或者hosts文件完成域名到Ingress node IP的地址解析。**

类似一个个nginx server配置

核心：使用同一个域名创建不同的Nginx ingress实例，然后对应不同的service，并通过annotations

指定流量权重。

```yaml
## 原yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: nginx
  annotations:
    kubernetes.io/ingress.class: nginx
spec:
  rules:
  - host: canary.example.com
    http:
      paths:
      - backend:
          serviceName: nginx-v1
          servicePort: 80
        path: /

## 基于权重的canary
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "10"
  name: nginx-canary
spec:
  rules:
  - host: canary.example.com
    http:
      paths:
      - backend:
          serviceName: nginx-v2
          servicePort: 80
        path: /
        
 ## 基于cookies canary
 apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-by-header: "Region"
    nginx.ingress.kubernetes.io/canary-by-header-pattern: "cd|sz"
  name: nginx-canary
spec:
  rules:
  - host: canary.example.com
    http:
      paths:
      - backend:
          serviceName: nginx-v2
          servicePort: 80
        path: /
```



### 按Deployment拆流量

1. 发布新应用的Deployment，将标签修改得符合原有Service的匹配规则，这样原Service就有两个应用
2. 由于无法在七层精确控制流量打到新应用，因此只能观察新应用的监控表现是否正常
3. 删除旧应用

> 这种方式依赖的特性是Service的负载均衡，新应用分得的流量是pod的占比数，不适合精细控制的场合

#### 依赖网关控制流量

一个service通过app label选择对应的Deployment，然后通过app和app-canary deployment name的不同来区别Deployment Object。

Canary Deployment创建时通过注入Env variable来控制流量比重，当然这个变量需要能被自己的Gateway识别到，才能做到流量的按比例分流。

Canary Deployment删除对象时，通过唯一的`pod-template-hash`来区别。因为Canary Deployment和Deployment 都属于一个app ，app metadata一致即label selector一致。



### 按RS拆

这种方式无法在用户层使用，需要为Deployment编写新的控制器，控制新版本发布时RS切换流程，由原来的自动滚动发布变更为可手动控制的手动滚动发布，得到的效果和按Deployment拆分一致，但不需要用户再去创建和删除Deployment资源，一切滚动都自动化。argo rollout的默认做法就是这样

了解了灰度原理，但在实现上还是需要我们手动增删资源、发布后测试，argo rollout将这一切实现了自动化。主要功能如下

- 灰度流程自动化，命令行控制灰度的进度
- 支持多种流量管理方式，包括服务网格等
- 提供自动化分析能力。可根据prometheus、普通http接口、Job执行结果情况控制灰度进度，实现完全的自动化

> argo rollout也不是没有缺点：学习成本高、流量管理方式固定、和gitops兼容不好等。尤其是流量管理方式，如果网络拓扑结构不在其支持方式内，就无法使用，这也是下文没有使用它的原因。

### 引用

1. https://zou8944.com/2022/06/01/K8S%E5%81%9A%E7%81%B0%E5%BA%A6%E5%8F%91%E5%B8%83/
2. https://www.cnblogs.com/tencent-cloud-native/p/13865502.html