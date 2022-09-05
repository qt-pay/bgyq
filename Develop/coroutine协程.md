## coroutine协程

### coroutine概述

协程具备两个特点：

- 协程的本地数据在后续调用中始终保持
- 协程在控制离开时暂停执行，当控制再次进入时只能从离开的位置继续执行

协程： 用堆内存模拟栈内存，不断切换协程栈内存到线程的共享栈就行了。

线程是最小的执行单元~~ 协程将自己的stack 切换到线程的stack去执行。

它是在系统分配的Threads下实现N个协程，在线程栈的不断切换执行（纯用户态），从而实现并发。

即系统线程的切换（线程时间片到了）还是会影响coroutine的。

协程是个概念，由不同语言来自己实现。

system call 和 阻塞都会让出CPU试用权。

gevent是python的并发的网络库，它的协程基于greenlet，（greenlet切换协程要手动switch才可以）, 并基于libev实现快速事件循环（Linux上是epoll，FreeBSD上是kqueue，Mac OS X上是select）。

gevent封装了greenlet不用开发者显式的手动去切换协程，即**遇到IO事件主动让出线程，切换到其他协程执行**。相比于线程提高了CPU的利用率，协程之间上下文切换更快。gevent协程同一时间只有一个协程操作资源线程安全。

### stackful coroutine and stackless coroutine

实现一个协程的关键点在于如何保存、恢复和切换上下文。

每个协程切换的时候, 整个栈都会被切换, 看起来和线程没啥区别, 只是调度一个发生在用户态可以由用户控制, 一个发生在内核态由系统控制.

> gevent 是有栈协程，通过epoll完成用户态的协程切换。
>
> eg：导入gevent中的猴子补丁，来把原来python自带的socket变成基于epoll的socke

相比于有栈协程直接切换栈帧的思路，无栈协程在不改变函数调用栈的情况下，采用类似生成器（generator）的思路实现了上下文切换

无栈协程的实现, 要几个条件: 

1. 栈帧内保存的不是状态而是指向状态的指针. 
2. **所有帧的状态保存在堆上**

**协程是否为栈式（Stackful）构造，即是否可以在内部的嵌套调用中挂起**

栈式协程允许在内部的嵌套函数中挂起，恢复时从挂起点继续执行。非栈式协程只能在主体部分执行挂起操作，可以用来开发简单的迭代器或生成器，但遇到复杂些的控制结构时，会把问题搞得更加复杂。例如，如果生成器的生成项是通过递归或辅助函数生成的，必须创建出一系列相应层级结构的辅助生成器连续生成项直到到达原始调用点。非栈式协程也不足以实现用户级多任务。

greenlet是有栈协程，asyncio是无栈协程。区分起来最简单的就是如果同步跟异步调用有区别，比如要加个await，那就是无栈，八九不离十了。有栈协程上下文切换靠切换整个系统栈，而无栈协程共享同一个系统栈，所以切换时协程必须主动返回到开始调用协程的地方，下次需要自己恢复当时的现场。一般无栈协程会通过一个用户堆栈或者类似的数据结构来完成await调用的保存和恢复，asyncio中也是这样，它通过一个await链将coroutine串起来，实现了await级联调用。

有栈协程保存上下文跟进程/线程保存上下文的操作有什么区别？

不用切换到内核，不用保存全部寄存器等等

作者：灵剑
链接：https://www.zhihu.com/question/320424555/answer/653576288
来源：知乎
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。

如下： 阅读源码，可以很快找到答案

https://cloud.tencent.com/developer/article/1560282

#### 有栈协程伪代码

有栈的协程和操作系统多线程是很相似的。考虑以下伪代码：

```go
func routine() int
{
    var a = 5
    sleep(1000)
    a += 1
    return a
}
```

sleep调用时，会发生上下文的切换，当前的执行体被挂起，直到约定的时间再被唤醒。局部变量a 在切换时会被保存在栈中，切换回来后从栈中恢复，从而得以继续运行。所谓有栈就是指执行体本身的栈。每次创建一个协程，需要为它分配栈空间。究竟分配多大的栈的空间是一个技术活。分的多了，浪费，分的少了，可能会溢出。`Go`在这里实现了一个协程栈扩容的机制，相对比较优雅的解决了这个问提。

#### 无栈协程伪代码

无栈协程顾名思义就是不使用栈和上下文切换来执行异步代码逻辑的机制。这里异步代码虽然是异步的，但执行起来看起来是一个同步的过程。从这一点上来看`Rust`协程与`Go`协程也没什么两样。举例说明：

```rust
async fn routine() 
{
    let mut a = 5;
    // 类似 Py需要await声明
    sleep(1000).await;
    a = a + 1;
    a
}
```

几乎是一样的流程。Sleep会导致睡眠，当时间已到，重新返回执行，局部变量a 内容应该还是5。`Go`协程是有栈的，所以这个局部变量保存在栈中，而`Rust`是怎么实现的呢？答案就是 Generator 生成的状态机。Generator 和闭包类似，能够捕获变量a，放入一个匿名的结构中，在代码中看起来是局部变量的数据 a，会被放入结构，保存在全局（线程）栈中。另外值得一提的是，Generator 生成了一个状态机以保证代码正确的流程。从sleep.await 返回之后会执行 a=a+1 这行代码。async routine() 会根据内部的 .await 调用生成这样的状态机，驱动代码按照既定的流程去执行。

按照一般的说法。无栈协程有很多好处。首先不用分配栈。因为究竟给协程分配多大的栈是个大问题。特别是在32位的系统下，地址空间是有限的。每个协程都需要专门的栈，很明显会影响到可以创建的协程总数。其次，没有上下文切换，貌似性能也许会好一些？当然，更大的好处是并不需要与CPU体系相关代码，也就有了更好的跨平台的能力。

### Asysmmetric and sysmetric Coroutine

对称与非对称最主要的区别在于是否存在传递程序控制权的行为。

`Coroutine`可以分为两种

- 非对称式(asymmetric)协程，又称半对称式(semi-asymmetric)协程，半协程(semi-coroutine)
- 对称式(symmetric)协程

非对称式(asymmetric)协程之所以被称为非对称的，是因为它提供了两种传递程序控制权的操作：

- coroutine.resume - (重)调用协程
- coroutine.yield - **挂起协程并将程序控制权返回给协程的调用者**



一个非对称协程可以看 做是从属于它的调用者的，二者的关系非常类似于例程(routine)与其调用者之间的关系。

对称式协程的特点是只有一种传递程序控制权的操作`coroutine.transfer`即将控制权直接传递给指定的协程。

曾经有这么一种说法，对称式和非对称式协程机制的能力并不等价,但事实上很容易根据前者来实现后者。在不少动态脚本语言(Python、Perl,Lua,Ruby)都提供了协程或与之相似的机制

#### Py的非对称无栈协程

这个真大佬：https://cloud.tencent.com/developer/article/1560282

python 中的上下文，被封装成了一个叫做 PyFrameObject 的结构，又称之为栈帧，看一下他的源码：https://github.com/xianth123/cpython/blob/master/Include/frameobject.h#L17

 cpython 中对 生成器的定义：*https://github.com/xianth123/cpython/blob/master/Include/genobject.h#L15*

这个结构体中，有三个最重要的成员

1. 指向生成器上下文的 指针
2. 一个指示生成器状态的字符串 未启动，停止，运行，结束
3. 生成器的字节码

也就是 上下文 + 指令序列 + 状态

在Python实际的执行中，会产生很多PyFrameObject对象，而这些对象会被链接起来，形成一条执行环境链表。这正是对x86机器上栈帧间关系的模拟。**在x86上，栈帧间通过esp指针和ebp指针建立了关系，使新的栈帧在结束之后能顺利回到旧的栈帧中，而Python正是利用f_back来完成这个动作。**

另外比较重要的两点就是各种环境变量表，以及程序运行必不可少的堆栈。f_lasti 记录了字节码运行的位置，这也就意味着在 PyFrameObject 中，我们可以随时恢复代码的运行。

为什么叫做无栈协程呢？因为generator没有自己的栈，只有它自己的帧。每次执行到这个coroutine的时候，是把其帧丢到调用者的 栈上的。

**无栈协程的本质就是一个状态机（state machine）**，它可以理解为在另一个角度去看问题，即**同一协程协程的切换本质不过是指令指针寄存器的改变**。

无栈协程的上下文就是

```python
## 
def gen_a():
    for i in range(3):
        # 让出控制权
        yield "A-%d" % i


def gen_b():
    for i in range(2):
        # 遇到 gen_a()的yeild，继续执行gen_b的for语句了
        gen_a()
        yield "B-%d" % i


print(list(gen_b()))
## output
['B-0', 'B-1']

## 使用yeild from 迭代生成器 gen_a()
def gen_a():
    for i in range(3):
        # 让出控制权
        yield "A-%d" % i


def gen_b():
    for i in range(2):
        # yield from = 
        yield from gen_a()
        yield "B-%d" % i


print(list(gen_b()))
## output
['A-0', 'A-1', 'A-2', 'B-0', 'A-0', 'A-1', 'A-2', 'B-1']

## 下面这个能理解吗？
## 因为仅让出控制权，但是没执行
def gen_a():
    for i in range(3):
        yield "A-%d" % i


def gen_b():
    for i in range(2):
        print(list(gen_a()))
        # 仅让出控制权
        yield "B-%d" % i


print(list(gen_b()))
## output
['A-0', 'A-1', 'A-2']
['A-0', 'A-1', 'A-2']
['B-0', 'B-1']
```

上面只是生成器，它是协程的基础：在 asyncio 的实现中，协程与生成器最大的区别，就是生成器的叶节点可以是 数字、函数、字符串等各种对象，但是 asyncio 的协程实现中，叶节点只能是 None 或者 future。

https://cloud.tencent.com/developer/article/1560282

这又用到了数据结构：树...

#### Python的有栈协程

gevent

### Python 的协程 ： 堆模拟栈！

协程： 用堆内存模拟栈内存，不断切换协程栈内存到线程的共享栈就行了。

线程是最小的执行单元~~ 协程将自己的stack 切换到线程的stack去执行。

在OS中，进程是资源分配单位，线程是最小的调度执行单位，这是大前提。所以，进程中至少有一个线程。

而Python的协程则是只在用户态利用事件机制实现切换的低成本并发手段。

https://zhuanlan.zhihu.com/p/94018082

CPU的内部组成：

* 时钟
* 运算器
* 控制器
* 寄存器

CPU是一系列寄存器的集合，而程序是将寄存器作为对象描述的。

使用高级语言编写的程序会在编译后转化成机器语言，然后通过CPU内部的寄存器来处理不同类型的CPU，其内部寄存器的数量，种类以及寄存器存储的数值范围都是不同的。根据功能的不同，我们可以将寄存器大致划分为八类。

**累加寄存器：**存储执行运算的数据和运算后的数据。

**标志寄存器：**存储运算处理后的CPU的状态。

> 条件和循环分支会使用到 `jump（跳转指令）`，会根据当前的指令来判断是否跳转，上面我们提到了`标志寄存器`，无论当前累加寄存器的运算结果是正数、负数还是零，标志寄存器都会将其保存（也负责溢出和奇偶校验）

**程序计数器：**存储下一条指令所在内存的地址。

**基址寄存器：**存储数据内存的起始地址。

**变址寄存器：**存储基址寄存器的相对地址。

**通用寄存器：**存储任意数据。

**指令寄存器：**存储指令。CPU内部使用，程序员无法通过程序对该寄存器进行读写操作。

**栈寄存器：**存储栈区域的起始地址。

其中，程序计数器，累加寄存器，标志寄存器，指令寄存器和栈寄存器都只有一个，其他的寄存器一般有多个。

其中，程序计数器执行下一个代码指令的地址，使得程序可以按顺序执行--

eg: 程序计数器地址： 100 --> 101 --> 102 --> 103 --> 104

```python
## 假设对应指令为
## 100： 将n赋值为0
## 101: 如果n大于0，则跳转到104
## 102： 将n+1
## 103： 返回 n
## 104：  返回n
```

end

#### 线程（概念）

在OS中，进程是资源分配单位，线程是最小的调度执行单位，这是大前提。所以，进程中至少有一个线程。

而协程则是只在用户态利用事件机制实现切换的低成本并发手段。

一个标准的线程由线程id、程序计数器(PC)、寄存器集合和栈组成。

每个线程类似一个独立进程，不同的是线程之间共享地址空间，能够访问到相同的数据。线程之间共享进程的内存空间(包括代码段、数据段、堆等)以及以下进程级的资源（如打开文件和信号）。

一个经典的进程和线程的关系如下图所示：

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/线程和进程的对比.jpg)

#### 线程上下文切换

Linux中线程和进程是同一个 task struct，但是同一进程的所有线程共享memory--

假设有2个线程运行在一个处理器上，从运行一个线程(T1)切换到另一个线程(T2)时，一定会发生上下文切换。对于进程，我们需要将状态保存到进程控制块(PCB)中，现在我们需要一个或多个线程控制块(TCB)来保存每个线程的状态，但是和进程上下文切换相比，线程在进行上下文切换的时候**地址空间保持不变**(即不需要切换当前使用的页表，因为同一进程下的线程内存页是共享的)。一个拥有多线程的进程的地址空间，如下图所示，我们可以看到每个线程拥有有自己的栈。

所以，线程的上下文切换是是将当前状态保存到自己的线程栈即可。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/线程-1.jpg)

#### 实现协程的理论依据

**有栈协程就是实现了一个用户态的线程**，用户可以在**堆**上模拟出协程的栈空间，当需要进行协程上下文切换的时候，主线程只需要交换栈空间和恢复协程的一些相关的寄存器的状态就可以实现一个用户态的线程上下文切换，没有了从用户态转换到内核态的切换成本，协程的执行也就更加高效。

斯，怪不得怪不得，线程的切换完全在用户态，代码可以控制，不需要从用户态到内核态拷贝的开销--

#### 协程上下文切换： 手动

利用ucontext提供的四个函数`getcontext(),setcontext(),makecontext(),swapcontext()`可以在一个进程中实现用户级的线程切换。

 在类System V环境中,在头文件< ucontext.h > 中定义了两个结构类型，mcontext_t和ucontext_t和四个函数getcontext(),setcontext(),makecontext(),swapcontext().利用它们可以在一个进程中实现用户级的线程切换。

 mcontext_t类型与机器相关，并且不透明.ucontext_t结构体则至少拥有以下几个域:

```c
       typedef struct ucontext {
           struct ucontext *uc_link;
           sigset_t         uc_sigmask;
           stack_t          uc_stack;
           mcontext_t       uc_mcontext;
           ...
       } ucontext_t;
```

 当当前上下文(如使用makecontext创建的上下文）运行终止时系统会恢复uc_link指向的上下文；uc_sigmask为该上下文中的阻塞信号集合；uc_stack为该上下文中使用的栈；uc_mcontext保存的上下文的特定机器表示，包括调用线程的特定寄存器等。

 简单说来， ` getcontext`获取当前上下文，`setcontext`设置当前上下文，`swapcontext`切换上下文，`makecontext`创建一个新的上下文， 并隐式调用了`setcontext`。

下面讲解四个函数的作用，详细的函数使用方法参考man手册:

\1. 初始化ucp结构体，将当前的上下文保存到ucp中

https://man7.org/linux/man-pages/man3/getcontext.3.html

```c
//  The function getcontext() initializes the structure pointed to by ucp to the currently active context.
//The ucp parameter passed to the makecontext shall be initialized by a call to getcontext.   
int getcontext(ucontext_t *ucp); 
```

\2. 修改用户线程的上下文指向参数ucp，在调用makecontext之前必须调用getcontext初始化一个ucp，并且需要分配一个栈空间给初始化后的ucp，当上下文通过setcontext或者swapcontext激活后，就会紧接着调用第二个参数指向的函数func，参数argc代表 func所需的参数，在调用makecontext之前你需要初始化参数ucp->uc_link，这个参数表示func()执行之后，用户线程将要切换到ucp->uc_link所代表的上下文，其实是隐式的调用了setcontext函数。

```c
  void makecontext(ucontext_t *ucp, void (*func)(), int argc, ...);
```

\3. 设置当前的上下文为ucp，setcontext的上下文ucp应该通过getcontext或者makecontext取得，如果调用成功则不返回。

```c
   int setcontext(const ucontext_t *ucp);
```

\4. 保存当前上下文到oucp结构体中，然后激活upc上下文。

```c
// The swapcontext() function saves the current context in the structure pointed to by oucp, and then activates the context pointed to by ucp.
int swapcontext(ucontext_t *oucp, ucontext_t *ucp);   
```

写个简单的例子来看一下如何使用：

这只是个简单的切换，当`func`中用阻塞操作时，比如sleep，一直卡在这个协程的上下文不好。耽误CPU，所有这个例子只是展示下切换。

而且，这个例子有点大问题，这里的两个协程没有共享数据，当实际代码中，如果进程的协程有共享数据了怎么处理呢？？？这么简单的上下文切换肯定满足不了的--

```c
#include <ucontext.h>
#include <stdio.h>

int func(void *arg) {
  puts("this is func");
}

void coroutine_test() {
  char stack[1024 * 128];
  ucontext_t child, main;
  // 获取当前上下文 and initializes context with  currently context
  getcontext(&child);

  // 分配栈空间 uc_stack.ss_sp 指向栈顶
  child.uc_stack.ss_sp = stack;
  child.uc_stack.ss_size = sizeof(stack);
  child.uc_stack.ss_flags = 0;
  // 指定后继上下文
  child.uc_link = &main;
  // child.uc_link = NULL;
  // create context with functin
  makecontext(&child, (void (*)(void))func, 0);

  //切换到child上下文，保存当前上下文到main
  swapcontext(&main, &child);
  // 如果设置了后继上下文，func函数指向完后会返回此处 如果设置为NULL，就不会执行这一步
  puts("this is coroutine_test");
}

int main() {
  coroutine_test();
  return 0;
}

```

end

#### 协程上下文切换：自动/调度器

上面的例子，展示了协程切换的底层，但是有个问题，这需要手动控制切换，当协程有N个时，你无法根据当前线程是否阻塞然后去切换到其他线程的上下文去执行--。而且对于协程共享的数据也要处理好。

**调度器中包含协程运行时的共享栈stack，共享栈可以认为所有的协程在运行时用的都是同一块栈，当调用协程的时候，将自己的栈拷贝到共享栈即可。当协程切出时，将栈空间再复制出来。**

这才是真正的coroutine例子--

```c
#include "coroutine.h"
#include <stdio.h>

struct args {
    int n;
};

static void
foo(struct schedule * S, void *ud) {
    struct args * arg = ud;
    int start = arg->n;
    int i;
    for (i=0;i<5;i++) {
        printf("coroutine %d : %d\n",coroutine_running(S) , start + i);
        // 切出当前协程
        coroutine_yield(S);
    }
}

static void
test(struct schedule *S) {
    struct args arg1 = { 0 };
    struct args arg2 = { 100 };

    // 创建两个协程
    int co1 = coroutine_new(S, foo, &arg1);
    int co2 = coroutine_new(S, foo, &arg2);
    printf("main start\n");
    while (coroutine_status(S,co1) && coroutine_status(S,co2)) {
        // 使用协程co1
        coroutine_resume(S,co1);
        // 使用协程co2
        coroutine_resume(S,co2);
    } 
    printf("main end\n");
}

int 
main() {
    // 创建一个协程调度器
    struct schedule * S = coroutine_open();
    test(S);
    // 关闭协程调度器
    coroutine_close(S);
    return 0;
}
```

end

核心对象

\1. `struct schedule* S` 协程调度器

```c
   struct schedule {
       char stack[STACK_SIZE]; // 运行时栈，此栈即是共享栈，一个时刻只能被一个协程占用。
       ucontext_t main; // 主协程的上下文
       int nco;        // 当前存活的协程个数
       int cap;        // 协程管理器的当前最大容量，即可以同时支持多少个协程。如果不够了，则进行2倍扩容
       int running;    // 正在运行的协程ID
       struct coroutine **co; // 一个一维数组，用于存放所有协程。其长度等于cap
   };
```

**调度器中包含协程运行时的共享栈stack，共享栈可以认为所有的协程在运行时用的都是同一块栈，当调用协程的时候，将自己的栈拷贝到共享栈即可。当协程切出时，将栈空间再复制出来。**

**还包括主协程上下文，可以认为是协程执行完毕之后回到的上下文。**

以及一个一维数组，用来存放调度器包括的所有协程。

\2. `coroutine`协程

`coroutine`结构体包括协程需要执行的函数，协程自己的上下文，用于上下文切换，以及stack用来保存运行时栈。

协程状态

协程有4个状态，分别是READY、RUNNING、SUSPEND、DEAD这四个状态。状态转换如图所示：

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/协程状态.jpg)

`coroutine_new`函数用来创建一个协程，协程进行状态转移的核心实现位于`coroutine_resume`函数和`coroutine_yield`函数中。

* READY --> RUNNING

  从READY状态转移到RUNNING状态进行的操作，和我们上面讲ucontext举的例子一样。首先，我们初始化协程的上下文，将协程的栈空间指向调度器中的共享栈，uc_link参数设定为S->main，当执行完makecontext指定的函数之后就会返回到调用coroutine_resume函数的地方。

  ```c
  // coroutine_resume
  switch(status) {
      case COROUTINE_READY:
          //初始化ucontext_t结构体,将当前的上下文放到C->ctx里面
          getcontext(&C->ctx);
          // 将当前协程的运行时栈的栈顶设置为S->stack，每个协程都这么设置，这就是所谓的共享栈。（注意，这里是栈顶）
          C->ctx.uc_stack.ss_sp = S->stack; 
          C->ctx.uc_stack.ss_size = STACK_SIZE;
          C->ctx.uc_link = &S->main; // 如果协程执行完，将切换到主协程中执行
          S->running = id;
          C->status = COROUTINE_RUNNING;
  
          // 设置执行C->ctx函数, 并将S作为参数传进去
          uintptr_t ptr = (uintptr_t)S;
          makecontext(&C->ctx, (void (*)(void)) mainfunc, 2, (uint32_t)ptr, (uint32_t)(ptr>>32));
  
          // 将当前的上下文放入S->main中，并将C->ctx的上下文替换到当前上下文
          swapcontext(&S->main, &C->ctx);
          break;
  ```

  end

* RUNNING --> SUSPEND

  当执行coroutine_yield函数之后，我们看到程序会首先保存当前协程的运行时栈，然后把协程的状态修改为挂起状态，并切换到主协程中。

  ```c
  void
  coroutine_yield(struct schedule * S) {
      // 取出当前正在运行的协程
      int id = S->running;
      assert(id >= 0);
  
      struct coroutine * C = S->co[id];
      assert((char *)&C > S->stack);
  
      // 将当前运行的协程的栈内容保存起来
      _save_stack(C,S->stack + STACK_SIZE);
  
      // 将当前栈的状态改为 挂起
      C->status = COROUTINE_SUSPEND;
      S->running = -1;
  
      // 所以这里可以看到，只能从协程切换到主协程中
      swapcontext(&C->ctx , &S->main);
  }
  ```

  end

* SUSPEND --> RUNNING

  当从挂起状态恢复为执行状态时，会将协程的运行时栈拷贝到共享栈，并再次切换上下文回到调用coroutine_yield函数的地方。

  ```c
  case COROUTINE_SUSPEND:
          // 将协程所保存的栈的内容，拷贝到当前运行时栈中
          // 其中C->size在yield时有保存
          memcpy(S->stack + STACK_SIZE - C->size, C->stack, C->size);
          S->running = id;
          C->status = COROUTINE_RUNNING;
          swapcontext(&S->main, &C->ctx);
          break;
  ```

  end

* end

#### 协程运行栈如何保存: 6

https://zhuanlan.zhihu.com/p/84935949

**调度器中包含协程运行时的共享栈stack，共享栈可以认为所有的协程在运行时用的都是同一块栈，当调用协程的时候，将自己的栈拷贝到共享栈即可。当协程切出时，将栈空间再复制出来。**

但是，每个协程的栈可以能大小不一样啊，这么办、？？

协程： 在共享的Stack中，划分一个个独立的coroutine stack.

共享栈的意思是共享同一块空间，但这块空间的内容是每个协程独占的.即一个时刻只有一个协程占用共享栈。

当执行coroutine_yield函数的时候，会发现该函数会调用`_save_stack`将当前协程的栈保存起来，因为coroutine是基于共享栈的，所以协程的栈内容需要单独保存起来。这里有一个很trick的点，那就是当前协程的运行栈怎么保存起来，也就是如何获取协程的栈空间。

一开始我们会在初始化调度器的时候设置共享栈的大小，stack指向栈顶，为了降低内存的占用，我们保存协程栈的时候不会直接保存一份和共享栈一样大小的栈空间，这时候我们需要找到该协程的栈顶位置。

下面代码的实现非常巧妙，他声明了一个局部变量dummy，而dummy的地址就是栈顶位置，大家可以参考上面讲的foo函数栈增长部分，相信你一定也能想到。

这里我们需要了解下栈内存空间的布局，即栈的生长方向是从高地址往低地址。我们只要找到栈的栈顶和栈底的地址，就可以找到整个栈内存空间了。

在coroutine中，因为协程的运行时栈的内存空间是自己分配的。在coroutine_resume阶段设置了`C->ctx.uc_stack.ss_sp = S.S->stack`。根据以上理论，栈的生长方向是高地址到低地址，因此栈底的就是内存地址最大的位置，即`S->stack + STACK_SIZE`就是栈底位置。

这里特意用到了一个dummy变量，这个dummy的作用非常关键也非常巧妙，大家可以细细体会下。因为dummy变量是刚刚分配到栈上的，此时就位于**栈的最顶部位置**。整个内存布局如下图所示：



![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/协程内存地址.jpg)

因此整个栈的大小就是从栈底到栈顶，`S->stack + STACK_SIZE - &dummy`。

这个dummy是在要切出协程栈的时候才分配的变量，肯定指向栈底。

```c
struct coroutine {
    coroutine_func func; // 协程所用的函数
    void *ud;  // 协程参数
    ucontext_t ctx; // 协程上下文
    struct schedule * sch; // 该协程所属的调度器
    ptrdiff_t cap;   // 已经分配的内存大小
    ptrdiff_t size; // 当前协程运行时栈，保存起来后的大小
    int status; // 协程当前的状态
    char *stack; // 当前协程的保存起来的运行时栈
};

static void
_save_stack(struct coroutine *C, char *top) {
    // 这个dummy很关键，是求取整个栈的关键.以为dummy刚入栈，所以肯定在栈底。
    // 这个非常经典，涉及到linux的内存分布，栈是从高地址向低地址扩展，因此
    // S->stack + STACK_SIZE就是运行时栈的栈底
    // dummy，此时在栈中，肯定是位于最底的位置的，即栈顶
    // top - &dummy 即整个栈的容量
    char dummy = 0;
    assert(top - &dummy <= STACK_SIZE);
    // 如果已分配内存小于当前栈的大小，则释放内存重新分配
    if (C->cap < top - &dummy) {
        free(C->stack);
        C->cap = top-&dummy;
        C->stack = malloc(C->cap);
    }
    C->size = top - &dummy;
    // 从dummy拷贝size内存到C->stack
    memcpy(C->stack, &dummy, C->size);
}
```

end

#### 协程与异步

哈哈哈，搞明白了。

协程，只是在用户态实现了context(上下文)的切换。切换到其他上下文，该阻塞还是要阻塞啊。

异步，通过事件机制(select/poll/epoll)，再不同的协程/线程/进程之间切换执行，不会阻塞。