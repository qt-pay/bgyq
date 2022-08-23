## k8s init container

Init containers are exactly like regular containers, except:

* Init containers always run to completion.
* Each init container must complete successfully before the next one starts.

If a Pod's init container fails, the kubelet repeatedly restarts that init container until it succeeds. However, if the Pod has a restartPolicy of Never, and an init container fails during startup of that Pod, Kubernetes treats the overall Pod as failed.



![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/pod-init-to-ready.png)

### 引用

1. https://loft.sh/blog/kubernetes-init-containers/