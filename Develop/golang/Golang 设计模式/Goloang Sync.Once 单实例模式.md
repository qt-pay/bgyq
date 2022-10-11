## Goloang Sync.Once 单实例模式

### 单实例模式

单例模式指的是在整个程序运行期间，我们只能初始化某个类一次，然后一直使用这个实例，尤其是在多线程的环境下，也要保证如此。



### 最简单单实例

对整个方法加锁最简单暴力，但是这样做性能最差。因为每个goroutine都要经历lock().

```go

type singleton struct {
	N int
}
var instance *singleton
var lock sync.Mutex
var res *singleton

func main()  {
	for i:=0;i<10;i++{
		j := i
		go func() {
			res = GetSingleInstance( j)
			fmt.Println(*res)
		}()

	}
	time.Sleep(8 * time.Second)
}

func GetSingleInstance(j int) *singleton {
	lock.Lock()
	fmt.Println("get lock", j)
	if instance == nil{
		instance = new(singleton)
		instance.N = j
	}
	lock.Unlock()
	return instance
}
// output
// 可以多次执行观察输出结果喔
get lock 0
{0}
get lock 1
{0}
get lock 3
{0}
get lock 4
{0}
get lock 2
{0}
get lock 6
{0}
get lock 5
{0}
get lock 7
{0}
get lock 8
{0}
get lock 9
{0}

```

N个goroutines并发执行时，哪个goroutine先lock就会将singleton instance初始化。

### 双检锁/双重检锁：安全？

双检锁，顾名思义，两次检查一次锁：

```go
type singleton struct {
	N int
}
var instance *singleton
var lock sync.Mutex
var res *singleton

func main()  {
	for i:=0;i<10;i++{
		j := i
		go func() {
			res = GetSingleInstance( j)
			fmt.Println(*res)
		}()

	}
	time.Sleep(8 * time.Second)
}

func GetSingleInstance(j int) *singleton {
	if instance == nil{
        // all goroutines pass if statement  pattern and  wait to lock...
		lock.Lock()
		fmt.Println("get lock", j)
		if instance == nil {
			instance = new(singleton)
			instance.N = j
		}
		lock.Unlock()
	}
	return instance
}
// output
get lock 0
{0}
get lock 1
{0}
get lock 5
{0}
{0}
get lock 3
{0}
get lock 9
{0}
get lock 6
{0}
get lock 7
{0}
get lock 4
{0}
get lock 8
{0}

```

这里加上第二个`instance == nil`的原因是：

**加锁之前可能有多个线程/协程都判断为空，这些线程/协程都会在这里等着加锁，它们最终也都会执行加锁操作，**不过加锁之后的代码在多个线程/协程之间是串行执行的，一个线程/协程判空之后创建了实例，其它线程/协程在判断c是否为空时必然得出false的结果，这样就能保证c仅创建一次(如果没有第二个if判断会导致实例创建多个？)。而且后续调用GetSingleInstance时都会仅执行第一次判空，得出false的结果，然后直接返回c。这样每个线程/协程最多只执行一次加锁操作，后续都只是简单的判断下就能返回结果，其性能必然不错。

去掉第二个if判断语句，执行结果如下：

```go
func GetSingleInstance(j int) *singleton {
	if instance == nil{
		lock.Lock()
		fmt.Println("get lock", j)
		instance = new(singleton)
		instance.N = j
		//if instance == nil {
		//	instance = new(singleton)
		//	instance.N = j
		//}

		lock.Unlock()
	}
	return instance
}
// output
get lock 0
{0}
get lock 1
{1}
get lock 5
{5}
get lock 3
{3}
get lock 9
{9}
get lock 6
{6}
get lock 7
{7}
get lock 4
{4}
{5}
get lock 8
{8}

```



#### Lock: happen-before

Mutex 排他锁的这一次Lock总是Happens After上一次的Unlock。

这就是说，纵容N个goroutines获取到了Mutex Lock，它们访问临界区资源也是串行执行的。

读写锁不同，因为读写锁可以允许同时Read.

锁只保证：它保护的某一个临界区内的代码在同一时刻只被某一个 goroutine 执行（也可以说接触到），而不会保证其中代码执行的原子性。所以，在其中代码执行期间也有可能被中断，中断的粒度通常是语句级别的。

#### Java 指令重排: 非原子操作

指令重排序：是编译器在不改变执行效果的前提下，对指令顺序进行调整，从而提高执行效率的过程。
一个最简单的重排序例子：

```java
int a = 1;
String b = "b";

```

对于这两行毫无关联的操作指令，编译器可能会将其顺序调整为：

```java
String b = "b";
int a = 1;
```

此时该操作并不会影响后续指令的执行和执行结果。
再回过头看我们的双检锁内部，对于"instance = new DoubleCheckLock();"这一行代码，它分为三个步骤执行：

1.分配一块内存空间
2.在这块内存上初始化一个DoubleCheckLock的实例
3.将声明的引用instance指向这块内存
第2和第3个步骤都依赖于第1个步骤，但是2和3之间没有依赖关系，那么如果编译器将2和3调换顺序，变成了：

1.分配一块内存空间
2.将声明的引用instance指向这块内存
3.在这块内存上初始化一个DoubleCheckLock的实例
当线程A执行到第2步时，instance已经不为null了，因为它指向了这块内存，此时如果线程B走到了"if (instance == null)"，那么线程B其实拿到的还是一个null，因为这块内存还没有初始化，这就出现了问题。

指令重排序是导致出现线程不安全的直接原因，而根本原因则是对象实例化不是一个原子操作。

Java volatile关键字 主要用来避免重排序问题导致其他的线程看到了一个已经分配内存和地址但没有初始化的对象，也就是说这个对象还不是处于可用状态，就被其他线程引用了。

下面的代码在多线程环境下不是原子执行的。

```javascript
instance=new DoubleCheckSingleton();
```

正常的底层执行顺序会转变成三步：

```javascript
(1) 给DoubleCheckSingleton类的实例instance分配内存

(2) 调用实例instance的构造函数来初始化成员变量

(3) 将instance指向分配的内存地址
```

上面的三步，无论在A线程当前执行到那一步骤，对B线程来说可能看到A的状态只能是两种1，2看到的都是null，3看到的非null，这是没问题的。

但是如果线程A在重排序的情况下，上面的执行顺序会变成1,3,2。现在假设A线程按1,3,2三个步骤顺序执行，当执行到第二步的时候。B线程开始调用这个方法，那么在第一个null的检查的时候，就有可能看到这个实例不是null，然后直接返回这个实例开始使用，但其实是有问题的，**因为对象还没有初始化，状态还处于不可用的状态，故而会导致异常发生。**即返回了一个未完全初始化的对象。

#### golang 指令重排：data race

在Go语言规范中，赋值操作分为两个阶段：第一阶段对赋值操作左右两侧的表达式进行求值，第二阶段赋值按照从左至右的顺序执行。

根据汇编分析，**Golang 单条赋值操作没有指令重排的问题，那这个双检锁怎么还是不安全的呢？**

因为**Golang中对变量的读和写都没有原子性的保证**，所以很可能出现这种情况：锁里边变量赋值只处理了一半，锁外边的另一个goroutine就读到了未完全赋值的变量。所以这个双检锁的实现是不安全的。

> 即其他goroutine读取到了只分配内存地址但是没有写入值的instance，就会报错。

Golang中将这种问题称为data race，说的是对某个数据产生了并发读写，读到的数据不可预测，可能产生问题，甚至导致程序崩溃。

单条赋值操作没有重排序的问题，但是**重排序问题在Golang中还是存在的**，稍不注意就可能写出BUG来。比如下边这段代码：

```go
a=1
b=1
c=a+b
```

在执行这段程序的goroutine中并不会出现问题，但是另一个goroutine读取到`b==1`时并不代表此时`a==1`，因为a=1和b=1的执行顺序可能会被改变。针对重排序问题，Golang并没有暴露类似volatile的关键字，因为理解和正确使用这类能力进行并发编程的门槛比较高

就是无法保障happen-before

### CAS优化双检锁：保障flag读写原子操作

```go
type Conn struct {
    Addr  string
    State int
}

var c *Conn
var mu sync.Mutex
var done uint32

func getInstance() *Conn {
    if atomic.LoadUint32(&done) == 0 {
        mu.Lock()
        defer mu.Unlock()
        if done == 0 {
            // 只要把标志位设置为1即可
            defer atomic.StoreUint32(&done, 1)
            c = &Conn{"127.0.0.1:8080", 1}
        }
    }
    return c
}
```

改变的地方就是sync.Once做的两个增强；原子读写和defer中更改完成标识。

当然如果要做的工作仅限于此，还不如直接使用sync.Once。

有时候我们需要的单例不是一成不变的，比如在ylog中需要每小时创建一个日志文件的实例，再比如需要为每一个用户创建不同的单例；再比如创建实例的过程中发生了错误，可能我们还会期望再执行实例的创建过程，直到成功。这两个需求是sync.Once无法做到的。

```go
i:=0
func newConn() *Conn {
    fmt.Println("newConn")
    div := i
    i++
    k := 10 / div
    return &Conn{"127.0.0.1:8080", k}
}

func getInstance() *Conn {
    if atomic.LoadUint32(&done) == 0 {
        mu.Lock()
        defer mu.Unlock()

        if done == 0 {
            defer func() {
                if r := recover(); r == nil {
                    defer atomic.StoreUint32(&done, 1)
                }
            }()
			// panic
            c = newConn()
        }
    }
    return c
}
```





### Sync.Once 原子操作

sync.Once源码很短很清楚，介绍了怎么解决这个并发问题的：

```go
type Once struct {
	done uint32
	m    Mutex
}

func (o *Once) Do(f func()) {
    if atomic.LoadUint32(&o.done) == 0 {
      o.doSlow(f)
   }
}

func (o *Once) doSlow(f func()) {
   o.m.Lock()
   defer o.m.Unlock()
   if o.done == 0 {
       // defer 灵性
      defer atomic.StoreUint32(&o.done, 1)
      f()
   }
}
```

- `atomic.LoadUint32` 用于原子加载地址（也就是 &o.done），返回加载到的值；

- o.done 为 0 是代表尚未执行。若同时有两个 goroutine 进来，发现 o.done 为 0（此时 `f` 尚未执行），就会进入 `o.doSlow(f)` 的慢路径中（slow path）；

- `doSlow` 使用 `sync.Mutex` 来加锁，一个协程进去，其他的被阻塞在锁的地方（注意，此时是阻塞，不是直接返回，这是和 CAS 方案最大的差别）；

- 经过 `o.m.Lock()` 获取到锁以后，如果此时 o.done 还是 0，意味着依然没有被执行，此时就可以放心的调用 `f`来执行了。否则，说明当前协程在被阻塞的过程中，已经失去了调用`f` 的机会，直接返回。

- `defer atomic.StoreUint32(&o.done, 1)` 是这里的精华，必须等到`f()` 返回，在 defer 里才能够去更新 o.done 的值为 1。

  `defer atomic.StoreUint32(&o.done, 1) `，使用原子写改变done的值为1，代表目标函数已经执行过。它会在目标函数f执行完毕，doSlow方法返回之前执行。这个设计很精妙，精确控制了改写done值的时机。

可以看出，这里用的也是双检锁的模式，只不过做了两个增强：一是使用原子读写，避免了并发读写的内存数据不一致问题；二是在defer中更改完成标识，保证了代码执行顺序，不会出现完成标识更改逻辑被编译器或者CPU优化提前执行

第二个`o.done == 0 ` 不能省略

如果在当前 goroutine 的 atomic.LoadUint32(&o.done) == 0 和 o.doSlow(f) 的执行间隙有其他 goroutine 执行到了 atomic.StoreUint32(&o.done, 1) ，那么再等到当前 goroutine 继续执行的时候 done 可就不是 0 了。

#### CAS实现版本实现Once.Do ：有问题

采用`CAS`原子操作来代替`if o.done == 0 `这个判断，如下：

```go
// demo
type MyOnce struct {
    flag uint32
    lock sync.Mutex
}

func (m *MyOnce)Do(f func())  {
    if atomic.LoadUint32(&m.flag) == 0{
        m.lock.Lock()
        defer m.lock.Unlock()
        // 0 和 1 原子互换，即如果m值为0，就原子性的将它设置为1.
        if atomic.CompareAndSwapUint32(&m.flag,0,1){
            f()
        }
    }
}

func testDo()  {
    mOnce := MyOnce{}
    for i := 0;i<10;i++{
        go func() {
            mOnce.Do(func() {
                fmt.Println("test my once only run once")
            })
        }()
    }
}
```

不能使用`atomic.CompareAndSwapUint32`,因为它是先完成了标志位值的设置才去真正执行func。如果func执行失败呢？？？

正确的逻辑应该是，func执行成功后才将标志位设置为1。

o.done 是那个“作为参数值的函数”被执行完毕的标志，你在执行这个函数前就让 o.done 变成 1，那如果该函数没有正常执行完毕又该怎么算呢？ 注意，在这个函数执行完毕之前，所以想通过同一个 once 去执行参数函数的goroutine都会被阻塞。这也是加锁的作用之一。

Golang 源码中有注释，这么设计是出于 `Do()` 返回时必须保证 `f()` 已经执行完毕了，如果使用  `atomic.CompareAndSwap` 的话如果条件不满足直接返回了，无法保证 `f()` 已经执行完了



语法：

```go
func CompareAndSwapUint32(addr *uint32, old, new uint32) (swapped bool)
```

这里，addr 表示地址，old 表示 uint32 值，即旧值，new 是 uint32 新值，new将与旧值交换自身。

注意：(*uint32) 是指向 uint32 值的指针。而 uint32 是位长为 32 的整数类型。而且，int32 包含从 0 到 4294967295 的所有无符号 32 位整数的集合。

返回值：如果交换完成则返回true，否则返回false。

其实使用 `atomic.CompareAndSwapUint32` 是一个非常直观的方案，这样的话 `Do` 的实现就变成了

```go
func (o *OnceA) Do(f func()) {
  if !atomic.CompareAndSwapUint32(&o.done, 0, 1) {
    return
  }
  f()
}

```

这样的问题在于，一旦出现 CAS 失败的情况，成功协程会继续执行 `f`，但其他失败协程不会等待 `f` 执行结束。而`Do` 的API定位对此有着强要求，当一次 `once.Do` 返回时，执行的 `f` 一定是完成的状态。



### 引用

1. https://blog.csdn.net/zy13608089849/article/details/82703192
2. https://segmentfault.com/a/1190000041903401
3. https://juejin.cn/post/7088305487753510925