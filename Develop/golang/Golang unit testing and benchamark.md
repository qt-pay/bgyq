## Golang unit testing and benchamark

### go test command

'Go test' recompiles each package along with any files with names matching the file pattern "*_test.go".

基准测试代码文件必须是_test.go结尾，和单元测试一样,因为都是通过`go test`命令执行的

在`*_test.go`文件中有三种类型的函数，单元测试函数、基准测试函数和示例函数。

|   类型   |         格式          |              作用              |
| :------: | :-------------------: | :----------------------------: |
| 测试函数 |   函数名前缀为Test    | 测试程序的一些逻辑行为是否正确 |
| 基准函数 | 函数名前缀为Benchmark |         测试函数的性能         |
| 示例函数 |  函数名前缀为Example  |       为文档提供示例文档       |

`go test`命令会遍历所有的`*_test.go`文件中符合上述命名规则的函数，然后生成一个临时的main包用于调用相应的测试函数，然后构建并运行、报告测试结果，最后清理测试中生成的临时文件。

```bash
$ ls
bench  go.mod  sync_pool.go
$ go test -bench .
?       bench/test      [no test files]
## go file end with _test.go
$ mv sync_pool.go sync_pool_test.go
$ go test -bench .
goos: linux
goarch: amd64
pkg: bench/test
cpu: Intel(R) Xeon(R) Platinum 8163 CPU @ 2.50GHz
BenchmarkUnmarshal-2                8922            131496 ns/op
BenchmarkUnmarshalWithPool-2        9127            131004 ns/op
PASS
ok      bench/test      2.414s

```



### 单元测试unit testing

#### 格式

每个测试函数必须导入`testing`包，测试函数的基本格式（签名）如下：

```go
func TestName(t *testing.T){
    // ...
}
```

测试函数的名字必须以`Test`开头，可选的后缀名必须以大写字母开头，举几个例子：

```go
func TestAdd(t *testing.T){ ... }
func TestSum(t *testing.T){ ... }
func TestLog(t *testing.T){ ... }
```

其中参数`t`用于报告测试失败和附加的日志信息。

#### go test -v

一个软件程序也是由很多单元组件构成的。单元组件可以是函数、结构体、方法和最终用户可能依赖的任意东西。总之我们需要确保这些组件是能够正常运行的。单元测试是一些利用各种方法测试单元组件的程序，它会将结果与预期输出进行比较。

接下来，我们在`base_demo`包中定义了一个`Split`函数，具体实现如下：

```go
// base_demo/split.go

package base_demo

import "strings"

// Split 把字符串s按照给定的分隔符sep进行分割返回字符串切片
func Split(s, sep string) (result []string) {
	i := strings.Index(s, sep)

	for i > -1 {
		result = append(result, s[:i])
		s = s[i+1:]
		i = strings.Index(s, sep)
	}
	result = append(result, s)
	return
}
```

在当前目录下，我们创建一个`split_test.go`的测试文件，并定义一个测试函数如下：

```go
// split/split_test.go

package split

import (
	"reflect"
	"testing"
)

func TestSplit(t *testing.T) { // 测试函数名必须以Test开头，必须接收一个*testing.T类型参数
	got := Split("a:b:c", ":")         // 程序输出的结果
	expectRes := []string{"a", "b", "c"}    // 期望的结果
	if !reflect.DeepEqual(expectRes, got) { // 因为slice不能比较直接，借助反射包中的方法比较
		t.Errorf("expected:%v, got:%v", expectRes, got) // 测试失败输出错误提示
	}
}
```

此时`split`这个包中的文件如下：

```bash
❯ ls -l
total 16
-rw-r--r--  1 root  staff  408  4 29 15:50 split.go
-rw-r--r--  1 root  staff  466  4 29 16:04 split_test.go
```

在当前路径下执行`go test`命令，可以看到输出结果如下：

```bash
❯ go test
PASS
ok      golang-unit-test-demo/base_demo       0.005s
```

一个测试用例有点单薄，我们再编写一个测试使用多个字符切割字符串的例子，在`split_test.go`中添加如下测试函数：

```go
func TestSplitWithComplexSep(t *testing.T) {
	got := Split("abcd", "bc")
	expectRes := []string{"a", "d"}
	if !reflect.DeepEqual(expectRes, got) {
		t.Errorf("expected:%v, got:%v", expectRes, got)
	}
}
```

现在我们有多个测试用例了，为了能更好的在输出结果中看到每个测试用例的执行情况，我们可以为`go test`命令添加`-v`参数，让它输出完整的测试结果。

```bash
❯ go test -v
=== RUN   TestSplit
--- PASS: TestSplit (0.00s)
=== RUN   TestSplitWithComplexSep
    split_test.go:20: expected:[a d], got:[a cd]
--- FAIL: TestSplitWithComplexSep (0.00s)
FAIL
exit status 1
FAIL    golang-unit-test-demo/base_demo 0.009s
```

从上面的输出结果我们能清楚的看到是`TestSplitWithComplexSep`这个测试用例没有测试通过。



#### go test -run：正则匹配

The first, called local directory mode, occurs when go test is invoked with no package arguments (for example, 'go test' or 'go test -v'). In this mode, go test compiles the package sources and tests found in the current directory and then runs the resulting test binary. 

在执行`go test`命令的时候可以添加`-run`参数，它对应一个正则表达式，只有函数名匹配上的测试函数才会被`go test`命令执行。

例如通过给`go test`添加`-run=Sep`参数来告诉它本次测试只运行`TestSplitWithComplexSep`这个测试用例：

```bash
❯ go test -run=Sep -v
=== RUN   TestSplitWithComplexSep
--- PASS: TestSplitWithComplexSep (0.00s)
PASS
ok      golang-unit-test-demo/base_demo 0.010s
```

#### 回归测试

我们修改了代码之后仅仅执行那些失败的测试用例或新引入的测试用例是错误且危险的，正确的做法应该是完整运行所有的测试用例，保证不会因为修改代码而引入新的问题。

```bash
❯ go test -v
=== RUN   TestSplit
--- PASS: TestSplit (0.00s)
=== RUN   TestSplitWithComplexSep
--- PASS: TestSplitWithComplexSep (0.00s)
PASS
ok      golang-unit-test-demo/base_demo 0.011s
```

测试结果表明我们的单元测试全部通过。

通过这个示例我们可以看到，有了单元测试就能够在代码改动后快速进行回归测试，极大地提高开发效率并保证代码的质量。



#### 并行测试

 想要在单元测试过程中使用并行测试，可以像下面的代码示例中那样通过添加`t.Parallel()`来实现。

```go
func TestSplitAll(t *testing.T) {
	t.Parallel()  // 将 TLog 标记为能够与其他测试并行运行
	// 定义测试表格
	// 这里使用匿名结构体定义了若干个测试用例
	// 并且为每个测试用例设置了一个名称
	tests := []struct {
		name  string
		input string
		sep   string
		expectRes  []string
	}{
		{"base case", "a:b:c", ":", []string{"a", "b", "c"}},
		{"wrong sep", "a:b:c", ",", []string{"a:b:c"}},
		{"more sep", "abcd", "bc", []string{"a", "d"}},
		{"leading sep", "沙河有沙又有河", "沙", []string{"", "河有", "又有河"}},
	}
	// 遍历测试用例
	for _, tt := range tests {
		tt := tt  // 注意这里重新声明tt变量（避免多个goroutine中使用了相同的变量）
		t.Run(tt.name, func(t *testing.T) { // 使用t.Run()执行子测试
			t.Parallel()  // 将每个测试用例标记为能够彼此并行运行
			got := Split(tt.input, tt.sep)
			if !reflect.DeepEqual(got, tt.expectRes) {
				t.Errorf("expected:%#v, got:%#v", tt.expectRes, got)
			}
		})
	}
}
```

执行`go test -v`的时候就会看到每个测试用例并不是按照我们定义的顺序执行，而是互相并行了。

#### 跳过部分测试用例

为了节省时间支持在单元测试时跳过某些耗时的测试用例。

```go
func TestTimeConsuming(t *testing.T) {
    if testing.Short() {
        t.Skip("short模式下会跳过该测试用例")
    }
    ...
}
```

当执行`go test -short`时就不会执行上面的`TestTimeConsuming`测试用例。

#### 网络测试(Network)

##### TCP/HTTP

假设需要测试某个 API 接口的 handler 能够正常工作，例如 helloHandler

```go
func helloHandler(w http.ResponseWriter, r *http.Request) {
	w.Write([]byte("hello world"))
}
```

那我们可以创建真实的网络连接进行测试：

```go
// test code
import (
	"io/ioutil"
	"net"
	"net/http"
	"testing"
)

func handleError(t *testing.T, err error) {
	t.Helper()
	if err != nil {
		t.Fatal("failed", err)
	}
}

func TestConn(t *testing.T) {
	ln, err := net.Listen("tcp", "127.0.0.1:0")
	handleError(t, err)
	defer ln.Close()

	http.HandleFunc("/hello", helloHandler)
	go http.Serve(ln, nil)

	resp, err := http.Get("http://" + ln.Addr().String() + "/hello")
	handleError(t, err)

	defer resp.Body.Close()
	body, err := ioutil.ReadAll(resp.Body)
	handleError(t, err)

	if string(body) != "hello world" {
		t.Fatal("expected hello world, but got", string(body))
	}
}
```

- `net.Listen("tcp", "127.0.0.1:0")`：监听一个未被占用的端口，并返回 Listener。
- 调用 `http.Serve(ln, nil)` 启动 http 服务。
- 使用 `http.Get` 发起一个 Get 请求，检查返回值是否正确。
- 尽量不对 `http` 和 `net` 库使用 mock，这样可以覆盖较为真实的场景。

##### httptest

针对 http 开发的场景，使用标准库 `net/http/httptest` 进行测试更为高效。

上述的测试用例改写如下：

```go
// test code
import (
	"io/ioutil"
	"net/http"
	"net/http/httptest"
	"testing"
)

func TestConn(t *testing.T) {
	req := httptest.NewRequest("GET", "http://example.com/foo", nil)
	w := httptest.NewRecorder()
	helloHandler(w, req)
	bytes, _ := ioutil.ReadAll(w.Result().Body)

	if string(bytes) != "hello world" {
		t.Fatal("expected hello world, but got", string(bytes))
	}
}
```

使用 httptest 模拟请求对象(req)和响应对象(w)，达到了相同的目的。

#### setup and teardown

如果在同一个测试文件中，每一个测试用例运行前后的逻辑是相同的，一般会写在 setup 和 teardown 函数中。例如执行前需要实例化待测试的对象，如果这个对象比较复杂，很适合将这一部分逻辑提取出来；执行后，可能会做一些资源回收类的工作，例如关闭网络连接，释放文件等。标准库 `testing` 提供了这样的机制：

```go
func setup() {
	fmt.Println("Before all tests")
}

func teardown() {
	fmt.Println("After all tests")
}

func Test1(t *testing.T) {
	fmt.Println("I'm test1")
}

func Test2(t *testing.T) {
	fmt.Println("I'm test2")
}

func TestMain(m *testing.M) {
	setup()
	code := m.Run()
	teardown()
	os.Exit(code)
}
```

- 在这个测试文件中，包含有2个测试用例，`Test1` 和 `Test2`。
- 如果测试文件中包含函数 `TestMain`，那么生成的测试将调用 TestMain(m)，而不是直接运行测试。
- 调用 `m.Run()` 触发所有测试用例的执行，并使用 `os.Exit()` 处理返回的状态码，如果不为0，说明有用例失败。
- 因此可以在调用 `m.Run()` 前后做一些额外的准备(setup)和回收(teardown)工作。

执行 `go test`，将会输出

```bash
$ go test
Before all tests
I'm test1
I'm test2
PASS
After all tests
ok      example 0.006s
```



### 表格驱动测试

#### 介绍

表格驱动测试不是工具、包或其他任何东西，它只是编写更清晰测试的一种方式和视角。

编写好的测试并非易事，但在许多情况下，表格驱动测试可以涵盖很多方面：表格里的每一个条目都是一个完整的测试用例，包含输入和预期结果，有时还包含测试名称等附加信息，以使测试输出易于阅读。

使用表格驱动测试能够很方便的维护多个测试用例，避免在编写单元测试时频繁的复制粘贴。

表格驱动测试的步骤通常是定义一个测试用例表格，然后遍历表格，并使用`t.Run`对每个条目执行必要的测试。

#### 示例

官方标准库中有很多表格驱动测试的示例，例如fmt包中便有如下测试代码：

```go
var flagtests = []struct {
	in  string
	out string
}{
	{"%a", "[%a]"},
	{"%-a", "[%-a]"},
	{"%+a", "[%+a]"},
	{"%#a", "[%#a]"},
	{"% a", "[% a]"},
	{"%0a", "[%0a]"},
	{"%1.2a", "[%1.2a]"},
	{"%-1.2a", "[%-1.2a]"},
	{"%+1.2a", "[%+1.2a]"},
	{"%-+1.2a", "[%+-1.2a]"},
	{"%-+1.2abc", "[%+-1.2a]bc"},
	{"%-1.2abc", "[%-1.2a]bc"},
}
func TestFlagParser(t *testing.T) {
	var flagprinter flagPrinter
	for _, tt := range flagtests {
		t.Run(tt.in, func(t *testing.T) {
			s := Sprintf(tt.in, &flagprinter)
			if s != tt.out {
				t.Errorf("got %q, expectRes %q", s, tt.out)
			}
		})
	}
}
```

通常表格是匿名结构体切片，可以定义结构体或使用已经存在的结构进行结构体数组声明。name属性用来描述特定的测试用例。

接下来让我们试着自己编写表格驱动测试：

```go
func TestSplitAll(t *testing.T) {
	// 定义测试表格
	// 这里使用匿名结构体定义了若干个测试用例
	// 并且为每个测试用例设置了一个名称
	tests := []struct {
		name  string
		input string
		sep   string
		expectRes  []string
	}{
		{"base case", "a:b:c", ":", []string{"a", "b", "c"}},
		{"wrong sep", "a:b:c", ",", []string{"a:b:c"}},
		{"more sep", "abcd", "bc", []string{"a", "d"}},
		{"leading sep", "沙河有沙又有河", "沙", []string{"", "河有", "又有河"}},
	}
	// 遍历测试用例
	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) { // 使用t.Run()执行子测试
			got := Split(tt.input, tt.sep)
			if !reflect.DeepEqual(got, tt.expectRes) {
				t.Errorf("expected:%#v, got:%#v", tt.expectRes, got)
			}
		})
	}
}
```

在终端执行`go test -v`，会得到如下测试输出结果：

```bash
❯ go test -v
=== RUN   TestSplit
--- PASS: TestSplit (0.00s)
=== RUN   TestSplitWithComplexSep
--- PASS: TestSplitWithComplexSep (0.00s)
=== RUN   TestSplitAll
=== RUN   TestSplitAll/base_case
=== RUN   TestSplitAll/wrong_sep
=== RUN   TestSplitAll/more_sep
=== RUN   TestSplitAll/leading_sep
--- PASS: TestSplitAll (0.00s)
    --- PASS: TestSplitAll/base_case (0.00s)
    --- PASS: TestSplitAll/wrong_sep (0.00s)
    --- PASS: TestSplitAll/more_sep (0.00s)
    --- PASS: TestSplitAll/leading_sep (0.00s)
PASS
ok      golang-unit-test-demo/base_demo 0.010s
```



### 基准测试

在日常开发中，基准测试是必不可少的，基准测试主要是通过测试CPU和内存的效率问题，来评估被测试代码的性能，进而找到更好的解决方案。

而Go语言中自带的benchmark则是一件非常神奇的测试利器。有了它，开发者可以方便快捷地在测试一个函数方法在串行或并行环境下的基准表现。指定一个时间（默认是1秒），看测试对象在达到或超过时间上限时，最多能被执行多少次和在此期间测试对象内存分配情况。

```go
// B is a type passed to Benchmark functions to manage benchmark
// timing and to specify the number of iterations to run.
type B struct {

}
import (
    "fmt"
    "testing"
)
func BenchmarkSprint(b *testing.B) {
    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        fmt.Sprint(i)
    }
}

```

代码说明:

1. 基准测试代码文件必须是_test.go结尾，和单元测试一样；
2. 基准测试的函数以Benchmark开头；
3. 参数须为 *testing.B；
4. 基准测试函数不能有返回值；
5. b.ResetTimer是重置计时器，这样可以避免for循环之前的初始化代码的干扰；
6. b.N是基准测试框架提供的，Go会根据系统情况生成，不用用户设定，表示循环的次数，因为需要反复调用测试的代码，才可以评估性能；

```bash
$ go test -bench=. -run=none 
goos: linux
goarch: amd64
pkg: bench/test/bench
cpu: Intel(R) Xeon(R) Platinum 8163 CPU @ 2.50GHz
BenchmarkSprint-2        9041901               124.2 ns/op
PASS
ok      bench/test/bench        1.267s
```

`-bench=args`，接受一个表达式作为参数，匹配基准测试的函数，`.`一个点表示运行所有的基准测试。

因为默认情况下 go test 会运行单元测试，为了防止单元测试的输出影响我们查看基准测试的结果，可以使用-run=匹配一个从来没有的单元测试方法，过滤掉单元测试的输出，我们这里使用none，因为我们基本上不会创建这个名字的单元测试方法

接下来再解释下输出的结果：

- 函数名后面的-2，表示运行时对应的 GOMAXPROCS 的值；
- 接着的 9041901表示运行 for 循环的次数，也就是调用被测试代码的次数，也就是在b.N的范围内执行的次数；
- 最后的 124.2 ns/op表示每次需要花费 124.2纳秒；

#### 并行benchmark

```go
func BenchmarkSprints(b *testing.B) {
    b.RunParallel(func(pb *testing.PB) {
        for pb.Next() {
            // do something
            fmt.Sprint("parallel benchmark")
        }
    })
}
```

- RunParallel并发的执行benchmark。RunParallel创建p个goroutine然后把b.N个迭代测试分布到这些goroutine上。
- goroutine的数目默认是GOMAXPROCS。如果要增加non-CPU-bound的benchmark的并个数，在执行RunParallel之前那就使用`b.SetParallelism(p int)`来设置，最终goroutine个数就等于p * runtime.GOMAXPROCS(0)，。

```go
numProcs := b.parallelism * runtime.GOMAXPROCS(0)
```

- 所以并行的用法比较适合IO密集型的测试对象。



#### 使用参数

* ` -benchtime t` : 在测试某个函数性能的时候，并不是每次执行都会得到一模一样的效果，我连续执行10次，可能会有10次不一样的结果，这时候我们可能会选择添加一个指定的采样时间，来得出一个平均值。

  eg：`-benchtime=5s`

  该参数还可支持特殊形式**Nx**，用来指定被测函数的迭代次数，如：`-benchtime=100Nx`

  指定了迭代100次，则每个函数都会只迭代100次。

* `-count n`: 进行几轮benchmark

* `-cup n `：指定使用些cpu core

* `-benchemem`: 显示压测内存

```bash

$ go test -bench . -s 3  -benchmem
flag provided but not defined: -s
Usage of /tmp/go-build1116057703/b001/test.test:
  -test.bench regexp
        run only benchmarks matching regexp
  # 常用显示内存
  -test.benchmem
        print memory allocations for benchmarks
  -test.benchtime d
        run each benchmark for duration d (default 1s)
  -test.blockprofile file
        write a goroutine blocking profile to file
  -test.blockprofilerate rate
        set blocking profile rate (see runtime.SetBlockProfileRate) (default 1)
  -test.count n
        run tests and benchmarks n times (default 1)
  -test.coverprofile file
        write a coverage profile to file
  -test.cpu list
        comma-separated list of cpu counts to run each test with
  -test.cpuprofile file
        write a cpu profile to file
  -test.failfast
        do not start new tests after the first test failure
  -test.list regexp
        list tests, examples, and benchmarks matching regexp then exit
  -test.memprofile file
        write an allocation profile to file
  -test.memprofilerate rate
        set memory allocation profiling rate (see runtime.MemProfileRate)
  -test.mutexprofile string
        write a mutex contention profile to the named file after execution
  -test.mutexprofilefraction int
        if >= 0, calls runtime.SetMutexProfileFraction() (default 1)
  -test.outputdir dir
        write profiles to dir
  -test.paniconexit0
        panic on call to os.Exit(0)
  -test.parallel n
        run at most n tests in parallel (default 2)
  ## -run=none，表示不运行任何测试用例
  -test.run regexp
        run only tests and examples matching regexp
  -test.short
        run smaller test suite to save time
  -test.shuffle string
        randomize the execution order of tests and benchmarks (default "off")
  -test.testlogfile file
        write test action log to file (for use only by cmd/go)
  -test.timeout d
        panic test binary after duration d (default 0, timeout disabled)
  -test.trace file
        write an execution trace to file
  -test.v
        verbose: print additional output
exit status 2
FAIL    bench/test      0.004s
$ go test -bench . -count 3  -benchmem
goos: linux
goarch: amd64
pkg: bench/test
cpu: Intel(R) Xeon(R) Platinum 8163 CPU @ 2.50GHz
BenchmarkUnmarshal-2                8596            133154 ns/op            1392 B/op          8 allocs/op
BenchmarkUnmarshal-2                9133            131493 ns/op            1392 B/op          8 allocs/op
BenchmarkUnmarshal-2                9040            132028 ns/op            1392 B/op          8 allocs/op
BenchmarkUnmarshalWithPool-2        8678            130315 ns/op             240 B/op          7 allocs/op
BenchmarkUnmarshalWithPool-2        9144            130903 ns/op             240 B/op          7 allocs/op
BenchmarkUnmarshalWithPool-2        8773            130570 ns/op             240 B/op          7 allocs/op
PASS
ok      bench/test      7.118s

```



### 引用原文

1. https://segmentfault.com/a/1190000040868502
1. https://www.liwenzhou.com/posts/Go/golang-unit-test-0/
1. https://geektutu.com/post/quick-go-test.html#6-%E7%BD%91%E7%BB%9C%E6%B5%8B%E8%AF%95-Network