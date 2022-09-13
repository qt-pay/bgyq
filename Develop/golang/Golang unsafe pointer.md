## Golang unsafe pointer

借助unsafe.Pointer有时候可以挽回Go运行时（Go runtime）为了安全而牺牲的一些性能。

https://segmentfault.com/a/1190000039141774

### 反转移

Double Escape Character

`%%s`则变成字面量`%s`

`\\n`则变成字面量`\n`



### Why need Unsafe pointer

在 `Go` 中，所有的参数传递都是值传递，没有引用传递。

1. 如果参数占用内存过大，每次函数传递都需要变量拷贝，比较耗费内存；
2. 如果我们想要在函数内部修改变量的状态，并在调用完毕后看到这种修改，就需要使用指针

#### safe pointer limits

**两个任意指针类型不能随意转换**，只有两个类型的底层数据类型是一致的，才可以完成转换。

`*int`指针不能转成`*float`，同类型可以转换`type Myint int`,`*Myint == *int`

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/safe-pointer-limit-convert-type.jpg)

**不能对指针的地址进行算术运算**

定义一个变量 `a` ，初始化值为`1`然后取地址，对地址算数运算 `addr++` 会编译不通过；`*addr++` 编译通过，最后输出 `a=2`，其实 `*addr++` 被编译器解释为了`(*addr)++`，即解引用操作符 `*` 的优先级 高于 `自增符++`

### 内存对齐

#### why is memory-alignment

现代计算机中内存空间都是按照字节(byte)进行划分的，所以从理论上讲对于任何类型的变量访问都可以从任意地址开始，但是在实际情况中，在访问特定类型变量的时候经常在特定的内存地址访问，所以这就需要把各种类型数据按照一定的规则在空间上排列，而不是按照顺序一个接一个的排放，这种就称为内存对齐，**内存对齐是指首地址对齐，而不是说每个变量大小对齐。**

有些CPU可以访问任意地址上的任意数据，而有些CPU只能在特定地址访问数据，因此不同硬件平台具有差异性，这样的代码就不具有移植性，如果在编译时，将分配的内存进行对齐，这就具有平台可以移植性了

#### 字长 word size

CPU每次寻址都是要消费时间的，并且CPU 访问内存时，并不是逐个字节访问，而是以字长（word size）为单位访问，所以数据结构应该尽可能地在自然边界上对齐，如果访问未对齐的内存，处理器需要做两次内存访问，而对齐的内存访问仅需要一次访问，内存对齐后可以提升性能。举个例子：

假设当前CPU是32位的，并且没有内存对齐机制，数据可以任意存放，现在有一个int32变量占4byte，存放地址在0x00000002 – 0x00000005(纯假设地址)，这种情况下，每次取4字节的CPU第一次取到[0x00000000 – 0x00000003]，只得到变量1/2的数据，所以还需要取第二次，为了得到一个int32类型的变量，需要访问两次内存并做拼接处理，影响性能。如果有内存对齐了，int32类型数据就会按照对齐规则在内存中，上面这个int32变量数的字就会存在地址0x00000000处开始，那么处理器在取数据时一次性就能将数据读出来了，而且不需要做额外的操作，使用空间换时间，提高了效率。





#### 对齐系数

每个特定平台上的编译器都有自己的默认”对齐系数”，常用平台默认对齐系数如下：

- 32位系统对齐系数是4
- 64位系统对齐系数是8

这只是默认对齐系数，实际上对齐系数我们是可以修改的，之前写C语言的朋友知道，可以通过预编译指令`#pragma pack(n)`来修改对齐系数，因为C语言是预处理器的，但是在Go语言中没有预处理器，只能通过tags和命名约定来让Go的包可以管理不同平台的代码，但是怎么修改对齐系数，感觉Go并没有开放这个参数，找了好久没有找到，等后面再仔细看看，找到了再来更新！

#### 内存对齐好处：一次取值

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/memory_alignment.png)

变量 a、b 各占据 3 字节的空间，内存对齐后，a、b 占据 4 字节空间，CPU 读取 b 变量的值只需要进行一次内存访问。如果不进行内存对齐，CPU 读取 b 变量的值需要进行 2 次内存访问。第一次访问得到 b 变量的第 1 个字节，第二次访问得到 b 变量的后两个字节。

内存对齐对实现变量的原子性操作也是有好处的，每次内存访问是原子的，如果变量的大小不超过字长，那么内存对齐后，对该变量的访问就是原子的，这个特性在并发场景下至关重要。

简言之：合理的内存对齐可以提高内存读写的性能，并且便于实现变量操作的原子性。

#### 内存对齐原则:stuck_out_tongue_closed_eyes:Sizeof是Alignof的倍数

对于具体类型来说，对齐值=min(编译器默认对齐值，类型大小Sizeof长度)。也就是在默认设置的对齐值和类型的内存占用大小之间，取最小值为该类型的对齐值。eg： int8的对齐值就是1，而不是4或者8.

> 32位机器，最大是4
>
> 64位机器，最大是8，即`a int64`，Alignof返回值是8

struct在每个字段都内存对齐之后，其本身也要进行对齐，对齐值=min(默认对齐值，字段最大类型长度)。这条也很好理解，struct的所有字段中，最大的那个类型的长度以及默认对齐值之间，取最小的那个。

在unsafe包中有三个函数:

* func Sizeof(x ArbitraryType) uintptr
* func Offsetof(x ArbitraryType) uintptr
* func Alignof(x ArbitraryType) uintptr

unsafe.Sizeof 返回变量x的占用字节数，但不包含它所指向内容的大小，对于一个string类型的变量它的大小是16字节，一个指针类型的变量大小是8字节（基于64位机器，下文中的例子也是，不再说明）

unsafe.Offsetof 返回结构体成员地址相对于结构体首地址相差的字节数，称为偏移量，注意：x必须是结构体成员,unsafe.Offsetof(U.x) 等价于 reflect.TypeOf(U).Field(x).Offset

unsafe.Alignof 返迴变量x需要的对齐倍数，它可以被x地址整除

> unsafe.Sizeof是unsafe.Alignof的整倍数。

假设一个 struct 包含三个字段，`a int8`、`b int16`、`c int64`，顺序会对 struct 的大小产生影响吗？

```go

type demo1 struct {
	a int8
	b int16
}

type demo2 struct {
	a int8
	c int32
	b int16
}
 type demo3 struct {
 	a int64
 }

func main() {
	fmt.Println("OffSetof demo1{}.b: ", unsafe.Offsetof(demo1{}.b))
	fmt.Println("OffSetof demo2{}.b: ", unsafe.Offsetof(demo2{}.c)) 
	fmt.Println("Sizeof demo1 : ", unsafe.Sizeof(demo1{}))
	fmt.Println("Sizeof demo2 : ", unsafe.Sizeof(demo2{}))
	fmt.Println("Alignof demo1 : ", unsafe.Alignof(demo1{}))
	fmt.Println("Alignof demo2 : ", unsafe.Alignof(demo2{}))
	fmt.Println("Alignof demo3 : ", unsafe.Alignof(demo3{}))
}
// output
OffSetof demo1{}.b:  2
OffSetof demo2{}.b:  4
Sizeof demo1 :  4
Sizeof demo2 :  12
Alignof demo1 :  2
Alignof demo2 :  4
Alignof demo3 :  8
```

答案是会产生影响。每个字段按照自身的对齐倍数来确定在内存中的偏移量，字段排列顺序不同，上一个字段因偏移而浪费的大小也不同。

接下来逐个分析，首先是 demo1：

- a 是第一个字段，默认是已经对齐的，从第 0 个位置开始占据 1 字节。
- b 是第二个字段，对齐倍数为 2，因此，必须空出 1 个字节，偏移量才是 2 的倍数，从第 2 个位置开始占据 2 字节。
- c 是第三个字段，对齐倍数为 4，此时，内存已经是对齐的，从第 4 个位置开始占据 4 字节即可。

因此 demo1 的内存占用为 8 字节。

其实是 demo2：

- a 是第一个字段，默认是已经对齐的，从第 0 个位置开始占据 1 字节。
- c 是第二个字段，对齐倍数为 4，因此，必须空出 3 个字节，偏移量才是 4 的倍数，从第 4 个位置开始占据 4 字节。
- b 是第三个字段，对齐倍数为 2，从第 8 个位置开始占据 2 字节。

demo2 的对齐倍数由 c 的对齐倍数决定，也是 4，因此，demo2 的内存占用为 12 字节。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/memory_alignment_order.png)



#### 空struct{}对齐

空 `struct{}` 大小为 0，作为其他 struct 的字段时，一般不需要内存对齐。但是有一种情况除外：即当 `struct{}` 作为结构体最后一个字段时，需要内存对齐。因为如果有指针指向该字段, 返回的地址将在结构体之外，如果此指针一直存活不释放对应的内存，就会有内存泄露的问题（该内存不因结构体释放而释放）。

```go
type Empty struct {
}

type demo1 struct {
	a int8
	e Empty
}

type demo2 struct {
	e Empty
	a int8
}


func main() {
	fmt.Println("OffSetof demo1{}.b: ", unsafe.Offsetof(demo1{}.a))
	fmt.Println("OffSetof demo2{}.b: ", unsafe.Offsetof(demo2{}.a))
	fmt.Println("Sizeof demo1 : ", unsafe.Sizeof(demo1{}))
	fmt.Println("Sizeof demo2 : ", unsafe.Sizeof(demo2{}))
	fmt.Println("Alignof demo1 : ", unsafe.Alignof(demo1{}))
	fmt.Println("Alignof demo2 : ", unsafe.Alignof(demo2{}))
}
// output
OffSetof demo1{}.b:  0
OffSetof demo2{}.b:  0
// Empty struct 在结尾时，size加1
Sizeof demo1 :  2
Sizeof demo2 :  1
Alignof demo1 :  1
Alignof demo2 :  1

```



#### 4 or 8

没有64bits类型的数据（int8，float64）时，Alignof都是4。





### 指针类型转换： void

变量值所在内存地址的值不等于该内存地址存储的变量值” and “变量值不等于该变量的内存地址”

`unsafe.Pointer` 可以在不同的指针类型之间做转化，从而可以表示任意可寻址的指针类型：

1. 任何类型的指针都可以被转化为 `unsafe.Pointer`；
2. `unsafe.Pointer` 可以被转化为任何类型的指针；
3. `uintptr` 可以被转化为 `unsafe.Pointer`；
4. `unsafe.Pointer` 可以被转化为 `uintptr`

```go
func main()  {
	//i := 10
	var i int
	i = 10
	var p *int = &i
	var fp *float32= (*float32)(unsafe.Pointer(p))
	*fp = *fp * 10
	fmt.Println("i value:", i)  // 100
    fmt.Println("p value:", *p)
	fmt.Println("fp value:", *fp)
	var tP interface{}
	tP = *p
	res,err := tP.(int)
	if err{
		fmt.Println("Int assertion ", res)
	}
	var tFP interface{}
	tFP = *fp
	res2,e2 := tFP.(float32)
	if e2{
		fmt.Println("Float32 assertion:", res2)
		fmt.Printf("Float32 assertion %%f: %f\n", res2)
	}
	var f float32
	f = 100
	var tt interface{}
	tt = f
	res3,e3 := tt.(float32)
	if e3{
		fmt.Println("native float32 assertion:", res3)
	}
}
// output
i value: 100
p value: 100
fp value: 1.4e-43
Int assertion  100
Float32 assertion: 1.4e-43
Float32 assertion %f: 0.000000
native float32 assertion: 100
```

将指向 `int` 类型的指针转化为了 `unsafe.Pointer` 类型，再转化为 `*float32` 类型（参考前面的 `unsafe.Pointer` 转化规则 1、2）并进行运算，最后发现 `i` 的值发生了改变。

这个示例说明了 `unsafe.Pointer` 是一个万能指针，可以在任何指针类型之间做转化，这就绕过了 Go 的类型安全机制，所以是不安全的操作。`*fp`指针指向的值仍是一个`int`类型的数值的二进制，而不是float类型的二进制值。因为浮点数的bits和int类型的bits代表含义不一样，所以值会发送变化。





### uintptr：运算指针

`uintptr` 是 Go 内置的可用于存储指针的整型，而整型是可以进行数学运算的！因此，将 `unsafe.Pointer` 转化为 `uintptr` 类型后，就可以让本不具备运算能力的指针具备了指针运算能力：

```go
func main()  {
	arr := [3]int{1, 2, 3}
	ap := &arr
	sp := (*int)(unsafe.Pointer(uintptr(unsafe.Pointer(ap)) + unsafe.Sizeof(arr[0])))
	*sp += 3
	fmt.Println(arr)
}
//output
[1 5 3]
```

将数组 `arr` 的内存地址赋值给指针 `ap`，然后通过 `unsafe.Pointer` 这个桥梁转化为 `uintptr` 类型，再加上数组元素偏移量（通过 `unsafe.Sizeof` 函数获取），就可以得到该数组第二个元素的内存地址，最后通过 `unsafe.Pointer` 将其转化为 `int` 类型指针赋值给 `sp` 指针，并进行修改，最终打印的结果是：`[1 5 3]`

这样一来，就可以绕过 Go 指针的安全限制，实现对指针的动态偏移和计算了，这会导致即使发生数组越界了，也不会报错，而是返回下一个内存地址存储的值，这就破坏了内存安全限制，所以这也是不安全的操作，我们在实际编码时要尽量避免使用，必须使用的话也要非常谨慎。



### nil pointer dereference

nil 最好不要作为参数传给函数--

```go
func main(){
	var n *int
	*n = 4
}

// panic: runtime error: invalid memory address or nil pointer dereference
n 刚初始化的值是<nil>，无法被访问
// 校正
func main(){
	var n *int
	fmt.Println(n)
	num := 3
	n = &num
	fmt.Println(*n)

}
// output
<nil>
3
```

#### 指针和指向指针的指针

通常指针做形参，是不是不应该直接修改形参的指向，而是应该只修改指针指向的内容。

```go
func main(){
	var test *int	
	go testNum(test)
	fmt.Println("main goroutine waiting goroutine flush")
	time.Sleep(8  * time.Second)
	fmt.Println("main goroutine test addr",test)
	fmt.Println("main goroutine test value",*test)

	time.Sleep(3 * time.Second)

}

func testNum(n *int){
	fmt.Println("goroutine origin n", n)
	num := 5
    // 这里n替换了原来执行的指针，即此时n已经不指向main中的test了
	n = &num
	fmt.Println("goroutine init n", n)
	fmt.Println("goroutine change n value to ", *n)
	time.Sleep(4 * time.Second)
}

// output 
main goroutine waiting goroutine flush
goroutine origin n <nil>
goroutine init n 0xc00008c000
goroutine change n value to  5
// test 仅初始化声明了，没有赋值所以为<nil>
main goroutine test addr <nil>
panic: runtime error: invalid memory address or nil pointer dereference
```

修正版

```go
var test *int
func main(){

	go testNum(test)
	fmt.Println("main goroutine waiting goroutine flush")
	time.Sleep(8  * time.Second)
	fmt.Println("main goroutine test addr",test)
	fmt.Println("main goroutine test value",*test)

	time.Sleep(3 * time.Second)

}

func testNum(n *int){
	fmt.Println("goroutine origin n", n)
	num := 5
	n = &num
    // 或者不要形参n，直接给全局变量test赋值
	test = n
	fmt.Println("goroutine init n", n)
	fmt.Println("goroutine change n value to ", *n)
	time.Sleep(4 * time.Second)
}
```

end

### Unsafe Pointer应用场景

https://studygolang.com/articles/35592?fr=sidebar

#### 类型转换：内存size要大于等于

利用 Pointer 作为中介，完成 T1 类型 到 T2 类型的转换，`T1` 和 `T2` 是任意类型，如果 T1 的内存占用大于等于 T2，并且 T1 和 T2 的内存布局一致，可以利用 Pointer 作为中介，完成 T1类型 到 T2类型的转换。（如果T1 的内存占用小于 T2，那么 T2 剩余部分没法赋值，就会有问题）

`math` 包中的 `Float64bits` 函数将一个 `float64` 值转换为一个 `uint64`值，`Float64frombits` 为此转换的逆转换，即 Float64bits(Float64frombits(x)) == x。



### 引用

1. https://geekr.dev/posts/go-pointer-usage
2. https://juejin.cn/post/6844904104423063560
3. https://studygolang.com/articles/35592?fr=sidebar
4. https://www.coolcou.com/color-model/memory-alignment.html