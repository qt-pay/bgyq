## Goloang Sync.Once 单实例模式

### 单实例模式

单例模式指的是在整个程序运行期间，我们只能初始化某个类一次，然后一直使用这个实例，尤其是在多线程的环境下，也要保证如此。



### 最简单单实例

对整个方法加锁最简单暴力，但是这样做性能最差。

```go
type singleton struct {

}
var instance *singleton
var lock sync.Mutex
var res *singleton

func main()  {
	for i:=0;i<10;i++{
		j := i
		go func() {
			res = GetSingleInstance( j)
			fmt.Println(&res)
		}()

	}
	time.Sleep(2 * time.Second)
}

func GetSingleInstance(j int) *singleton {
	lock.Lock()
	fmt.Println("get lock", j)
	if instance == nil{
		instance = new(singleton)
	}
	lock.Unlock()
	return instance
}
```



### 双检锁/双重检锁

双检锁，顾名思义，两次检查一次锁：

```go

type singleton struct {

}
var instance *singleton
var lock sync.Mutex
var res *singleton

func main()  {
	for i:=0;i<10;i++{
		j := i
		go func() {
			res = GetSingleInstance( j)
			fmt.Println(&res)
		}()

	}
	time.Sleep(2 * time.Second)
}

func GetSingleInstance(j int) *singleton {
	if instance == nil{
		lock.Lock()
		fmt.Println("get lock", j)
		if instance == nil {
			instance = new(singleton)
		}
		lock.Unlock()
	}
	return instance
}
// output
get lock 0
0xc01960
get lock 1
0xc01960
get lock 5
0xc01960
get lock 2
0xc01960
get lock 4
0xc01960
get lock 7
0xc01960
0xc01960
get lock 9
0xc01960
get lock 6
0xc01960
get lock 8
0xc01960

```

这里加上第二个`instance == nil`的原因是：

**加锁之前可能有多个线程/协程都判断为空，这些线程/协程都会在这里等着加锁，它们最终也都会执行加锁操作，**不过加锁之后的代码在多个线程/协程之间是串行执行的，一个线程/协程判空之后创建了实例，其它线程/协程在判断c是否为空时必然得出false的结果，这样就能保证c仅创建一次。而且后续调用GetSingleInstance时都会仅执行第一次判空，得出false的结果，然后直接返回c。这样每个线程/协程最多只执行一次加锁操作，后续都只是简单的判断下就能返回结果，其性能必然不错。

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

**Golang 赋值操作没有指令重排的问题，那这个双检锁怎么还是不安全的呢？**

因为**Golang中对变量的读和写都没有原子性的保证**，所以很可能出现这种情况：锁里边变量赋值只处理了一半，锁外边的另一个goroutine就读到了未完全赋值的变量。所以这个双检锁的实现是不安全的。

Golang中将这种问题称为data race，说的是对某个数据产生了并发读写，读到的数据不可预测，可能产生问题，甚至导致程序崩溃。



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

### CAS实现版本

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



#### ???

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