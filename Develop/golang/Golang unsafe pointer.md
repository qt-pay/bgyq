## Golang unsafe pointer

借助unsafe.Pointer有时候可以挽回Go运行时（Go runtime）为了安全而牺牲的一些性能。

https://segmentfault.com/a/1190000039141774

### 反转移

Double Escape Character

`%%s`则变成字面量`%s`

`\\n`则变成字面量`\n`





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

这个示例说明了 `unsafe.Pointer` 是一个万能指针，可以在任何指针类型之间做转化，这就绕过了 Go 的类型安全机制，所以是不安全的操作。

上述例子，`*fp`的值为什么是`1.4e-43`即`0`。

https://www.ruanyifeng.com/blog/2010/06/ieee_floating-point_representation.html

#### float精度和有效位

精度主要取决于尾数部分的位数。

对于 float32（单精度）来说，表示尾数的为23位，除去全部为0的情况以外，最小为2^-23，约等于1.19*10^-7，所以float小数部分只能精确到后面6位，加上小数点前的一位，即有效数字为7位。

同理 float64（单精度）的尾数部分为 52位，最小为2^-52，约为2.22*10^-16，所以精确到小数点后15位，加上小数点前的一位，有效位数为16位。

**float32**，也即我们常说的单精度，存储占用4个字节，也即4*8=32位，其中1位用来符号，8位用来指数，剩下的23位表示尾数

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/float32.webp)



**float64**，也即我们熟悉的双精度，存储占用8个字节，也即8*8=64位，其中1位用来符号，11位用来指数，剩下的52位表示尾数

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/float64.webp)

#### 科学计数法

浮点数类型的取值范围可以从很微小到很巨大。浮点数取值范围的极限值可以在 math 包中找到：

- 常量 math.MaxFloat32 表示 float32 能取到的最大数值，大约是 3.4e38；
- 常量 math.MaxFloat64 表示 float64 能取到的最大数值，大约是 1.8e308；
- float32 和 float64 能表示的最小值分别为 1.4e-45 和 4.9e-324。


所以，`1.4e-43`就是`1.4e-45 * 10 * 10`,  因为`i`初始化值为10



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

### 引用

1. https://geekr.dev/posts/go-pointer-usage
2. https://juejin.cn/post/6844904104423063560