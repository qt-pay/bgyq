## Golang Sync RWMutex and Mutex

### Lock本质：串行

所有的锁的作用都是保障goroutine的happen-before。

Mutex 排他锁的这一次Lock总是Happens After上一次的Unlock。

锁只保证：它保护的某一个临界区内的代码在同一时刻只被某一个 goroutine 执行（也可以说接触到），而不会保证其中代码执行的原子性。所以，在其中代码执行期间也有可能被中断，中断的粒度通常是语句级别的。

即，所有的goroutine都会进入临界区，但是会变成串行执行。

如下，所有的goroutine都会等待且拿到Lock

```go
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
```



### 原子操作：避免重排

Lock只是保障了并发的串行执行，但是不能保障操作的原子性。

**Golang中对变量的读和写都没有原子性的保证**，所以很可能出现这种情况：锁里边变量赋值只处理了一半，锁外边的另一个goroutine就读到了未完全赋值的变量。

以Java为例：

```javascript
instance=new DoubleCheckSingleton();
```

正常的底层执行顺序会转变成三步：

```javascript
(1) 给DoubleCheckSingleton类的实例instance分配内存

(2) 调用实例instance的构造函数来初始化成员变量

(3) 将instance指向分配的内存地址
```

无论在A线程当前执行到那一步骤，对B线程来说可能看到A的状态只能是两种1，2看到的都是null，3看到的非null，这是没问题的。

但是如果线程A在重排序的情况下，上面的执行顺序会变成1,3,2。现在假设A线程按1,3,2三个步骤顺序执行，当执行到第二步的时候。B线程开始调用这个方法，那么在第一个null的检查的时候，就有可能看到这个实例不是null，然后直接返回这个实例开始使用，但其实是有问题的，**因为对象还没有初始化，状态还处于不可用的状态，故而会导致异常发生。**即返回了一个未完全初始化的对象。

如下，Goroutine 1 给 instance写操作时，可能先分配了地址还没来得及写入实际的值，goroutine 2就判断`instance != nil`直接返回一个未写入完成的对象，直接GG。

```go
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
```



### sync.Mutex

`Sync.Mutext`一个标准的互斥锁，平时用的也比较多，用法也非常简单，lock用于加锁，unlock用于解锁，配合defer使用，完美

### sync.RWMutex

读写锁是互斥锁的升级版，它最大的优点就是支持多读，但是读和写、以及写与写之间还是互斥的，所以比较适合读多写少的场景。

它的实现里面有5个方式：

```go
func (rw *RWMutex) Lock()
func (rw *RWMutex) RLock()
func (rw *RWMutex) RLocker() Locker
func (rw *RWMutex) RUnlock()
func (rw *RWMutex) Unlock()
```

其中Lock()和Unlock()用于申请和释放写锁,RLock()和RUnlock()用于申请和释放读锁，RLocker()用于返回一个实现了Lock()和Unlock()方法的Locker接口。

sync.RWMutex 用于读锁和写锁分开的情况。使用时注意如下几点：
（1）RWMutex 是单写多读锁，该锁可以加多个读锁或者一个写锁；
（2）读锁占用的情况下会阻止写，不会阻止读，多个 goroutine 可以同时获取读锁；
（3）写锁会阻止其他 goroutine（无论读和写）进来，整个锁由该 goroutine 独占；
（4）适用于读多写少的场景。

```go
//  RWLock demo
func main() {
	var mutex *sync.RWMutex
	mutex = new(sync.RWMutex)


	channels := make([]chan int, 10)
	for i := 0; i < 10; i++ {
		channels[i] = make(chan int)
		go func(i int, c chan int) {
			mutex.Lock()
			fmt.Println("Read Locked: ", i)
			fmt.Println("Unlock the read lock: ", i)
			time.Sleep(time.Second * 2)
			mutex.Unlock()
			c <- i
		}(i, channels[i])
	}

	for _, c := range channels {
		fmt.Println("channel output:", <-c)
	}
}
// output
每隔两秒执行一个goroutine
...
```

RLock()可以运行多个goroutine同时加锁

```go
//  RWLock demo
func main() {
	var mutex *sync.RWMutex
	mutex = new(sync.RWMutex)


	channels := make([]chan int, 10)
	for i := 0; i < 10; i++ {
		channels[i] = make(chan int)
		go func(i int, c chan int) {
			mutex.RLock()
			fmt.Println("Read Locked: ", i)
			fmt.Println("Unlock the read lock: ", i)
			time.Sleep(time.Second * 2)
            mutex.RUnlock()
			c <- i
		}(i, channels[i])
	}

	for _, c := range channels {
		fmt.Println("channel output:", <-c)
	}
}
// output
总共耗时2s多一点，完成全部goroutine执行，并发执行。
```



### 引用

0. https://time.geekbang.org/discuss/detail/234812

1. https://blog.csdn.net/K346K346/article/details/90476721

