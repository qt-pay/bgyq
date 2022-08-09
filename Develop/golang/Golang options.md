## Golang options

### 初始化对象参数

`Go`创建对象时，如何优雅的传递初始化参数？这里所说的优雅，指的是：

1. 支持传递多个参数
2. 参数个数、类型发生变化时，尽量保持接口的兼容性
3. 参数支持默认值
4. 具体的参数可根据调用方需关心的程度，决定是否提供默认值

`Go`并不像`c++`和`python`那样，支持函数默认参数。所以使用`Go`时，我们需要一种方便、通用的手法来完成这件事。

`Go`的很多开源项目都使用`Option`模式，但各自的实现可能有些许细微差别。

### 可变参数：`...`

#### 语法糖

可变参数实际上是 slice 类型的参数的语法糖。

* 给可变参数函数不传入参数时，可变参数slice为nil
* 当传入多个同类型变量时，可变参数为有值的slice
* 当传入slice作为参数给可变参数时，需要借助`...`运算符

```go
func test(str ...string) {
	for _,s := range str{
		fmt.Println(s)
	}
}
func main() {
	
	sliceTest := []string{"a", "b", "c"}
	test("a", "b", "3")
    // Error: test(sliceTest)
	test(sliceTest...)

}
```

通过将可变参数运算符`...` 加在现有切片后，可以将其传递给可变参数运算符。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/golang-可变参数.jpg)

#### 混合参数：可变and固定

```go
func Printf(format string, a ...interface{}) (n int, err error) {
    /* this is a pass-through with a... */  
    return Fprintf(os.Stdout, format, a...)
}

fmt.Printf("%d %s %f", 1, "string", 3.14)    // output: "1 string 3.14"
```

查看 Printf 的签名时，会发现它接受一个名为 format 的字符串和一个可变参数。

这是因为format是必需的参数。Printf 强制您提供它，否则代码将无法编译。

### 无Option模式，传统方式

当一个结构体增加一个属性时...，变动很多

```go
type Customer struct {
	id 			int
	name    string
    // 添加性别字段
	gender 	string
}

```

如下，会涉及到

* 修改初始化函数

  ```go
  func NewCustomer(id int, name, gender string) Customer {
  	return Customer{
  		id: id,
  		name: name,
      	// 添加性别字段
  		gender: gender,
  	}
  }
  ```

* 修改初始化代码

  ```go
  // 修改多处初始化代码... 很烦...
  customer := NewCustomer(1, "lazyr", "man")
  ```

每次修改属性值都会导致大量代码修改。

### Option模式：base 

传进来的参数进行自动装配。

```go
// 定义一个给Customer初始化的函数的类型
type Option func(customer *Customer)

// 参数为初始化参数的值,返回值为初始化函数
func WithId(id int) Option {
	return func(customer *Customer) {
		customer.id = id
	}
}

func WithName(name string) Option {
	return func(customer *Customer) {
		customer.name = name
	}
}


// 将用户自定义的各种初始化函数传递进来，进行遍历赋值
func NewCustomer(options ...Option) Customer {
	customer := Customer{}
	for _, option := range options {
		// 用户自定义的初始化函数
		option(&customer)
	}
	return customer
}
```

这样就可以动态初始化参数，并且也可以完成自定义参数值了，这样以后再增加只用于新业务的字段，也不需要改旧业务代码了

```go
func main() {
	// 连同参数和初始化函数一并传进去
	customer := NewCustomer(WithId(1), WithName("lazyr"))
	fmt.Println(customer)
}
```

这样就完成了普通初始化向动态初始化的转变，利用闭包+可变参数的特性实现了类似于重载机制的性质，也就是Golang中常见的模式—Option模式。

### Option模式：带默认值

```go
package main

type Foo struct {
  key string
  option Option

  // ...
}

type Option struct {
  num  int
  str  string
}

type ModOption func(option *Option)

func New(key string, modOption ModOption) *Foo {
  option := Option{
    num: 100,
    str: "hello",
  }
    //调用modOption func即main{}中New()的第二个参数
  modOption(&option)

  return &Foo{
    key: key,
    option: option,
  }
}

// ...

func main() {
  New("iamkey", func(option *Option) {
    // 调用方只设置 num
    option.num = 200
  })
}
```

换成高级的写法就是把`option.num=200`换成function（这个真的有必要吗,有必要，这样写不好看），在加上可变参数

```go
func WithNum(num int) ModOption {
  return func(option *Option) {
    option.num = num
  }
}

func WithStr(str string) ModOption {
  return func(option *Option) {
    option.str = str
  }
}

// 可变参数
func New(key string, modOptions ...ModOption) *Foo {
  option := Option{
    num: 100,
    str: "hello",
  }

  for _, fn := range modOptions {
    fn(&option)
  }

  return &Foo{
    key: key,
    option: option,
  }
}
```

最终调用代码

```go
New("iamkey", WithNum(200), WithStr("world"))
```



### Option模式总结

#### 核心：解耦

type + 可变参数+函数类型变量+指针参数+闭包？

> 好像确实用闭包...

通过函数操作结构体成员变量，实现解耦，当增加参数时，代码修改量很少而且很工整。

事实上，`Option`模式除了在创建对象时可以使用，里面的一些`API`设计思想，`Go`的小技巧，在编写普通函数时也可以使用。

模式说白了就是一种套路。在实现功能的基础之上，大家都熟悉了某种固有套路的写法，都按着这个套路走，那么代码的可读性、可维护性就更高些。

- 优点
  - 支持传递多个参数，并且在参数个数、类型发生变化时保持兼容性；
  - 任意顺序传递参数；
  - 支持默认值；
  - 方便拓展；
- 缺点
  - 增加许多function，成本增大；
  - 参数不太复杂时，尽量少用；
- 前面一直说初始化初始化，但Option不止可以在初始化过程使用，只要是函数传参是可变的情况下都可以尝试使用Option模式，但若是只有几个简单参数且后续不会发生变化，还是写死吧，别过度设计了。

### 练习demo

```go
type Person struct {
	name string
	age int
}

// 这里需要把fucntion变成type才能声明函数类型的slice
type Option func(person * Person)

func withName(name string) Option  {
	return func(person *Person) {
		person.name = name
	}
}

func withAge(age int) Option{
	return func(person *Person) {
		person.age = age
	}
}

func newPerson(options ...Option) Person{
    // 初始化值
	person := &Person{
		name: "dodo",
		age: 18,
	}
	for _,option := range options{
        // 可能传入的参数是包含nil的slice
		if option != nil{
			option(person)
		}
	}
	return *person
}


func main()  {
	personChandler := newPerson()
	fmt.Println(personChandler)
	personChandler30 := newPerson(withAge(30))
	fmt.Println(personChandler30)
	personEveryOne := newPerson(withName("Sunny"), withAge(18))
	fmt.Println(personEveryOne)

}

// output
{dodo 18}
{dodo 30}
{Sunny 18}
```

### 开源项目demo

`go.uber.org/ratelimit`库

```go
func main() {
	rl := ratelimit.New(100,ratelimit.Per(20 * time.Second)) // per second

	prev := time.Now()
	for i := 0; i < 10; i++ {
		now := rl.Take()
		fmt.Println(i, now.Sub(prev))
		prev = now
	}

// source code

```

### 引用

1. http://lazyr.top/archives/golang%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F-option%E6%A8%A1%E5%BC%8F
2. https://cloud.tencent.com/developer/article/1763856