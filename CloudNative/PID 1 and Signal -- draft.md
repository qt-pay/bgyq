## PID 1 and Signal -- H3C

### Linux PID 1

#### 进程re-parent

写的真棒

https://manjusaka.itscoder.com/posts/2021/02/13/a-simple-introduction-about-the-init-process-in-container/

父进程退出后，所属的子进程，将按照如下顺序进行 re-parent

1. 线程组里其余可用线程（这里的线程有所不一样，可以暂时忽略）
2. 在当前所属的进程树上不断寻找设置了 **PR_SET_CHILD_SUBREAPER** 进程
3. 在前面两者都无效的情况下，re-parent 到当前 PID Namespace 中的 1 号进程上

#### Signal

```bash
Signal     Value     Action   Comment
----------------------------------------------------------------------
SIGHUP        1       Term    Hangup detected on controlling terminal
                              or death of controlling process
SIGINT        2       Term    Interrupt from keyboard
SIGQUIT       3       Core    Quit from keyboard
SIGILL        4       Core    Illegal Instruction
SIGABRT       6       Core    Abort signal from abort(3)
SIGFPE        8       Core    Floating point exception
SIGKILL       9       Term    Kill signal
SIGSEGV      11       Core    Invalid memory reference
SIGPIPE      13       Term    Broken pipe: write to pipe with no
                              readers
SIGALRM      14       Term    Timer signal from alarm(2)
SIGTERM      15       Term    Termination signal
SIGUSR1   30,10,16    Term    User-defined signal 1
SIGUSR2   31,12,17    Term    User-defined signal 2
SIGCHLD   20,17,18    Ign     Child stopped or terminated
SIGCONT   19,18,25    Cont    Continue if stopped
SIGSTOP   17,19,23    Stop    Stop process
SIGTSTP   18,20,24    Stop    Stop typed at tty
SIGTTIN   21,21,26    Stop    tty input for background process
SIGTTOU   22,22,27    Stop    tty output for background process
```

end

#### 键盘输入和信号量

When you do a `ctrl+c`, it’s the same as sending a `SIGINT` signal and when you type `ctrl+z`, it’s the same as sending `SIGTSTP`, and when we type `fg` or `bg` that is the same as sending a `SIGCONT` signal.



###  容器PID 1

#### PID 1 核心功能要求

容器内一号进程所需要承担的职责

1. 信号的转发
2. Z 进程的回收

而在目前，在容器场景下，大家主要使用两个方案来作为自己的容器内一号进程，[dumb-init](https://github.com/Yelp/dumb-init) 和 [tini](https://github.com/krallin/tini)。这两个方案对于容器内孤儿与 Z 进程的处理都算是 OK。但是信号转发的实现上一言难尽。

pid namespace 本身的特性，我们来看看，[pid_namespaces](https://man7.org/linux/man-pages/man7/pid_namespaces.7.html) 中的相关介绍

> If the “init” process of a PID namespace terminates, the kernel terminates all of the processes in the namespace via a SIGKILL signal.

当前 pid ns 内的一号进程退出的时候，内核直接 SIGKILL 处理该 pid ns 内的剩余进程。

#### 容器终止流程

https://tasdikrahman.me/2019/04/24/handling-singals-for-applications-in-kubernetes-docker/



#### 容器re-parent

一个容器的的流程如下

1. Docker Daemon 向 containerd 发送指令
2. containerd 创建一个 containterd-shim 进程
3. containerd-shim 创建一个 runc 进程
4. runc 进程将根据 OCI 标准，设置相关环境（创建 cgroup，创建 ns 等），然后执行 `entrypoint` 中的设定的命令
5. runc 在执行完相关设置后，将自我退出，此时其子进程（即容器命名空间内的1号进程）将被 re-parent 给 containerd-shim 进程。

OK，上面 step 5 操作，就需要依赖我们上节中讲到的 **prctl** 和 **PR_SET_CHILD_SUBREAPER** 。

自此，containerd-shim 将承担容器内进程相关的操作，即便其父进程退出，子进程也会根据 re-parent 的流程托管到 containerd-shim 进程上。

#### 优雅下线容器

假设我一个服务需要实现一个需求叫做优雅下线。通常而言，我们会在暴力杀死进程之前，利用 SIGTERM 信号实现这个功能。但是在容器时期有个问题，我们一号进程，可能不是程序本身（比如大家习惯性的会考虑在 entrypoint 中用 bash 去裹一层），或者经过一些特殊场景，容器中的进程，全部已经托管在 containerd-shim 上了。而 contaninerd-shim 是不具备信号转发的能力的。

所以在这样一些场景下，我们就需要考虑额外引入一些组件来完成我们的需求。这里以一个非常轻量级的专门针对容器的设计的一号进程项目：这就是tini 和 dumb-init 工具诞生的原因了



#### pid 1设置不合理的坑

我们一个测试服务，Spring Cloud 的，在下线后，节点无法从注册中心摘除，然后百思不得其解，最后查到问题，，
本质上是这样，POD 被摘除的时候，K8S Scheduler 会给 POD 的 ENTRYPOINT 发一个 SIGTERM 信号，然后等待三十秒（默认的 graceful shutdown 超时实践)，还没响应就会 SIGKILL 直接杀
问题在于，我们 Eureka 版的服务是通过 start.sh 来启动的，ENTRYPOINT [“/home/admin/start.sh”]，容器里默认是 /bin/sh 是 fork/exec 模式，导致我服务进程没法正确的收到 SIGTERM 信号，然后一直没结束就被 SIGKILL 了。

刺激不刺激。除了信号转发无法正常处理以外，我们应用程序常见的一个常见处理的问题就是 Z 进程的出现，即子进程结束之后，无法正确的回收。比如早期 puppeteer 臭名昭著的 Z 进程问题。 在这种情况下，除了应用程序本身的问题以外，另外可能的原因是在守护进程这样的场景下，孤儿进程 re-parent 之后的进程，不具备回收子进程的功能

#### Docker 删除容器

When you issue a `docker stop`, docker send `SIGTERM` to the process running as PID 1 inside the container and waits for 10 seconds before it sends a `SIGKILL` to the kernel which will then straight terminate the process, if the process hasn’t terminated within that time frame.

A `docker kill` will not give the container process, an opportunity to stop gracefully, but will straight ahead kill it.

#### Kubernetes 删除Pod

When you do a

```bash
$ kubectl delete pods mypod
```

it will send a `SIGTERM` and then wait for a set number of seconds to send a `SIGKILL` to the process, this period is known as the grace termination period of the pod and can be configured in the [podSpec](https://kubernetes.io/docs/api-reference/v1.9/#podspec-v1-core)

If your process doesn’t handle `SIGTERM`, then it will get `SIGKILL`ed. Processes which are killed are immediately removed from the etcd and the API, without waiting for the process to actually terminate on the node.

If you have anything which needs a graceful shutdown, you need to implement a handler for `SIGTERM`.

If there are more than one containers in a pod, they both will be sent a `SIGTERM` and one would want to implement the right strategy on how they should be terminated.

#### 容器PID 1 最佳实践

 exec 裸起大法好（但需要应用自己处理信号量），不用dumb-init/tini 作一号进程平安。

应用的信号，进程回收这些基础行为应该应用自决。任何管杀不管埋而寄托于一号进程的行为，都是对于生产的不负责任。

### 总结

1. Linux 通过re-parent，完成进程间的管理，也是僵尸进程诞生的原因
2. Linux 下 PID 1两个核心：转发信号量和处理僵尸进程
3. Docker 创建容器过程通过：Pid namespace + re-parent实现容器PID 1
4. k8s/docker 干掉 容器是靠两个信号量: `SIGTERM` and `SIGKILL` 
5. 容器PID 1进程应用有转发和处理信号量的能力，且最好使用`exec`启动。
6. tini 和 dumn-init 听凑合

### tini

https://github.com/krallin/tini/issues/180