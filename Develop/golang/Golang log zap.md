## Golang log

Only use log.Fatal from main.main or int functions.

因为Fatal会导致defer 无法执行--

### time format

日志输出，必然会包含时间戳，但是golang很离谱，它的时间格式化是固定的 `2006-01-02 15:04:05 -0700` 。

| 数字 |           含义           |
| :--: | :----------------------: |
| 2006 |     Year(four-digit)     |
|  06  |     Year(two-digit)      |
|  01  |    Month(zero-padded)    |
|  02  | DD of month(zero-padded) |
|  15  |   24-hour(zero-padded)   |
|  04  |   Minute(zero-padded)    |
|  05  |   Second(zero-padded)    |

一般的困扰主要有：

- 不知道只能固定要这个时间，换其他的，出来的结果莫名其妙，然后一脸懵逼；
- 为什么没有像其他语言一样，yyyy-mm-dd 这样的形式？
- 这个时间有什么特殊意义吗？为什么挑这么个时间，完全记不住；

如下示例代码：

```go
func main() {
	fmt.Println("Now time:", time.Now().Format("2006-01-02 15:04:05"))
}
// output
Now time: 2022-06-17 12:29:53
```

为什么选择这个时间？不少人有这样的疑问。有人猜测是 Go 项目启动的时间等。但仔细研究，发现 Go Team 还是用心良苦，目的是解决大家记忆问题。

比如常规的 ymd 格式，以 PHP 为例，一般这样 `Y-m-d H:i:s`，输出类似：2021-08-03 09:30:00，但如果我想输出：`21-8-4 9:30:00`，你不查手册，能写出来吗？你看看 PHP 文档中关于 date 格式化的说明，头有点大，竟然那么多，虽然常用的形式，大部分人都记得，但遇到不怎么常用的，就得查手册了。

反观 Go 语言，它直接使用一个具体的时间来当做格式化字符串，需要什么格式，改这个时间格式即可。比如上面的例子，常规方式：2006-01-02 15:04:05，而 21-8-4 9:30:00 这种格式，只需要对应的改变值即可：06-1-2 3:04:05。而且，我查了下，PHP 没法表示没有前导零的分钟数和秒数，而 Go 很容易实现。很显然，Go 的方式是更合理、更易用的，对于各种变化，也能够更自如的应对。

#### 设计原理

只不过，很多人对这个具体的时间觉得记不住。这一点，Go 官方也考虑到了。毕竟采用特殊的时间，目的就是为了解决大家记忆问题，因此要确保这个特殊时间也好记。Go 是这么设计的：

```bash
1: month (January, Jan, 01, etc)
2: day
3: hour (15 is 3pm on a 24 hour clock)
4: minute
5: second
6: year (2006)
7: timezone (GMT-7 is MST)
```

刚好是 1 2 3 4 5 6 7，据此进行变化即可。

### log format demo

#### error log要全面

如下，一个服务启动报错，根据service找到它的启动命令，然后手动执行可以看到详细的报错

`Failed to parse configuration.Error msg 1 error(s) decoding:`错误提示很明显，该服务的配置文件启动上解析失败、

> Linux message文件也有记录

```bash
$ systemctl status test-agent.service 
● test-agent.service - UNI test Agent Daemon
   Loaded: loaded (/usr/lib/systemd/system/test-agent.service; enabled; vendor preset: disabled)
   Active: failed (Result: start-limit) since Wed 2022-09-07 15:52:31 CST; 7min ago
  Process: 16252 ExecStart=/bin/bash -c -l /usr/local/bin/test-agent (code=exited, status=1/FAILURE)
....


$ /bin/bash -c -l /usr/local/bin/test-agent
Failed to parse configuration.Error msg 1 error(s) decoding:

* 'ScsiIps[scsi2][0]' expected type 'string', got unconvertible type '[]interface {}'
test-agent/common/config.InitConfig
        /root/rpmbuild/BUILD/test-agent-release/common/config/config.go:114
main.main
        /root/rpmbuild/BUILD/test-agent-release/cmd/main.go:501
runtime.main
        /usr/local/go/src/runtime/proc.go:203
runtime.goexit
        /usr/local/go/src/runtime/asm_amd64.s:1357


$ cat /var/lib/message 
...

Sep  7 15:52:29 testnode systemd: Configuration file /usr/lib/systemd/system/network-audit-agent.service is marked executable. Please remove executable permission bits. Proceeding anyway.
Sep  7 15:52:29 testnode bash: Failed to parse configuration.Error msg 1 error(s) decoding:
Sep  7 15:52:29 testnode bash: * 'ScsiIps[scsi2][0]' expected type 'string', got unconvertible type '[]interface {}'
Sep  7 15:52:29 testnode bash: test-agent/common/config.InitConfig
Sep  7 15:52:29 testnode bash: /root/rpmbuild/BUILD/test-agent-release/common/config/config.go:114
Sep  7 15:52:29 testnode bash: main.main
Sep  7 15:52:29 testnode bash: /root/rpmbuild/BUILD/test-agent-release/cmd/main.go:501
Sep  7 15:52:29 testnode bash: runtime.main
Sep  7 15:52:29 testnode bash: /usr/local/go/src/runtime/proc.go:203
Sep  7 15:52:29 testnode bash: runtime.goexit
Sep  7 15:52:29 testnode bash: /usr/local/go/src/runtime/asm_amd64.s:1357
Sep  7 15:52:29 testnode systemd: test-agent.service: main process exited, code=exited, status=1/FAILURE
Sep  7 15:52:29 testnode systemd: Unit test-agent.service entered failed state.
Sep  7 15:52:29 testnode systemd: test-agent.service failed.
Sep  7 15:52:30 testnode systemd: test-agent.service holdoff time over, scheduling restart.
Sep  7 15:52:30 testnode systemd: Stopped UNI test Agent Daemon.
Sep  7 15:52:30 testnode systemd: Started UNI test Agent Daemon
```

end

### golang Package log

golang log库十分简单，package log提供了包级的接口，连logger对象都不需要创建，就可以直接使用。

这种无需创建logger变量而是直接使用包名+函数的方式写日志的方式减少了传递和管理logger变量的复杂性。

```go
// console output
log.Println(time.Now().Format("2006-01-02 15:04:05"),"test")
// output
2022/09/29 14:58:20 2022-09-29 14:58:20 test

// file output
func main() {
    file, err := os.OpenFile("./test.log", os.O_CREATE|os.O_APPEND|os.O_WRONLY, 0644)
    if err != nil {
        panic(err)
    }
    log.SetOutput(file)
    log.Println("this is go standard log package")
}
// file record
2022/09/29 14:58:20 this is go standard log package

```





### logrus

在Go生态中，[logrus](https://tonybai.com/2018/01/13/the-problems-i-encountered-when-writing-go-code-issue-1st)可能是使用最多的Go日志库，它不仅提供结构化的日志，更重要的是与标准库log包在api层面兼容。在性能不敏感的领域，logrus确实是不二之选。

简洁易用

### zap

Uber开源的zap日志库有更高的性能，经历过大厂性能敏感场景考验。

zap的主要优化点包括：

- 避免使用interface{}带来的开销（拆装箱、[对象逃逸到堆上](https://mp.weixin.qq.com/s/ALzzT1kQ3VfYQly1LT7mQw)）
- 坚决不用反射，每个要输出的字段（field）在传入时都携带类型信息
- 使用sync.Pool减少堆内存分配（针对代表一条完整日志消息的zapcore.Entry），降低对GC压力、

#### init logger:NewProduction and zapcore.NewCore

`zap.NewProduction()`是一个比较高级的API接口，包含了一系列初始化操作，可以自己拆分配置

```go
{
    ...
	zap.NewProduction()
	cfg := zap.NewProductionConfig()
	core := zapcore.NewCore(
		zapcore.NewJSONEncoder(cfg.EncoderConfig),
		zapcore.AddSync(writer),
		zapcore.Level(level),
	)
	logger := &Logger{
        // 
		l:     zap.New(core),
		level: level,
	}
	return logger
}
```

下面是源码：

```go
// It's a shortcut for NewProductionConfig().Build(...Option).
func NewProduction(options ...Option) (*Logger, error) {
	return NewProductionConfig().Build(options...)
}
// NewProductionConfig is a reasonable production logging configuration.
// Logging is enabled at InfoLevel and above.
//
// It uses a JSON encoder, writes to standard error, and enables sampling.
// Stacktraces are automatically included on logs of ErrorLevel and above.
func NewProductionConfig() Config {
	return Config{
		Level:       NewAtomicLevelAt(InfoLevel),
		Development: false,
		Sampling: &SamplingConfig{
			Initial:    100,
			Thereafter: 100,
		},
		Encoding:         "json",
		EncoderConfig:    NewProductionEncoderConfig(),
		OutputPaths:      []string{"stderr"},
		ErrorOutputPaths: []string{"stderr"},
	}
}

// Build constructs a logger from the Config and Options.
func (cfg Config) Build(opts ...Option) (*Logger, error) {
	enc, err := cfg.buildEncoder()
	if err != nil {
		return nil, err
	}

	sink, errSink, err := cfg.openSinks()
	if err != nil {
		return nil, err
	}

	if cfg.Level == (AtomicLevel{}) {
		return nil, errors.New("missing Level")
	}

	log := New(
		zapcore.NewCore(enc, sink, cfg.Level),
		cfg.buildOptions(errSink)...,
	)
    // 加上zap log options
	if len(opts) > 0 {
		log = log.WithOptions(opts...)
	}
	return log, nil
}


// NewCore creates a Core that writes logs to a WriteSyncer.
func NewCore(enc Encoder, ws WriteSyncer, enab LevelEnabler) Core {
	return &ioCore{
		LevelEnabler: enab,
		enc:          enc,
		out:          ws,
	}
}

// For sample code, see the package-level AdvancedConfiguration example.
func New(core zapcore.Core, options ...Option) *Logger {
	if core == nil {
		return NewNop()
	}
	log := &Logger{
		core:        core,
		errorOutput: zapcore.Lock(os.Stderr),
		addStack:    zapcore.FatalLevel + 1,
		clock:       zapcore.DefaultClock,
	}
	return log.WithOptions(options...)
}

// WithOptions clones the current Logger, applies the supplied Options, and
// returns the resulting Logger. It's safe to use concurrently.
func (log *Logger) WithOptions(opts ...Option) *Logger {
	c := log.clone()
	for _, opt := range opts {
		opt.apply(c)
	}
	return c
}

```



#### quick start

 NewProduction builds a sensible production Logger that writes InfoLevel and above logs to standard error as JSON.

zap初始化logger实例时，可以选择`Sugar()`wraps functon。可以使用任意的key-value pairs不用指明value type，但是性能会有损耗。



In contexts where performance is nice, but not critical, use the `SugaredLogger`. It's 4-10x faster than other structured logging packages and includes both structured and `printf`-style APIs.

```go
//  Sugar wraps the Logger to provide a more ergonomic, but slightly slower API

logger, _ := zap.NewProduction()
defer logger.Sync() // flushes buffer, if any
sugar := logger.Sugar()
sugar.Infow("failed to fetch URL",
  // Structured context as loosely typed key-value pairs.
  "url", url,
  "attempt", 3,
  "backoff", time.Second,
)
sugar.Infof("Failed to fetch URL: %s", url)

// output
{"level":"info","ts":1664442232.17884,"caller":"awesomeProject/test.go:66","msg":"failed to fetch URL","url":"www.test.com","attempt":3,"backoff":1}
```

When performance and type safety are critical, use the `Logger`. It's even faster than the `SugaredLogger` and allocates far less, **but it only supports structured logging**.

性能好，但是写起来麻烦点要指明字段类型。

```go
logger, _ := zap.NewProduction()
defer logger.Sync()
logger.Info("failed to fetch URL",
  // Structured context as strongly typed Field values.
  zap.String("url", url),
  zap.Int("attempt", 3),
  zap.Duration("backoff", time.Second),
)

// output
{"level":"info","ts":1664442232.17884,"caller":"awesomeProject/test.go:72","msg":"Failed to fetch URL: www.test.com"}
```



#### 自定义zap输出格式

自定义zap log的输出格式。

原来的格式：

```js
{"level":"info","ts":1617979575129,"caller":"cmd/main.go:62","msg":"prom server start success， port 10081"}
```

希望输出的样子

```js
[2021-04-10 17:27:55.419] [INFO] [212fb8d8-4e30-44fd-a6f2-d0a9b9799b9d] [cmd/main.go:62]  prom server start success, port 10081
```

解决方案

查阅了一下官方zap的输出格式时通过`zapcore.EncoderConfig`对象进行配置的， 所以我们也只需要修改它的一个初始化过程便可， 其中212fb8d8-4e30-44fd-a6f2-d0a9b9799b9d是一个全局的请求id（trace_id）我们可以先忽略这个内容部分，先看怎么改。

NewCore接收三个参数，也是Core的主要组成部分，它们如下图：

```text
                                 ┌───────────────┐
                                 │               │
                                 │               │
                      ┌─────────►│     Encoder   │
                      │          │               │
                      │          │               │
                      │          └───────────────┘
┌────────────────┐    │
│                ├────┘
│                │               ┌───────────────┐
│                │               │               │
│      Core      ├──────────────►│  WriteSyncer  │
│                │               │               │
│                ├─────┐         │               │
└────────────────┘     │         └───────────────┘
                       │
                       │
                       │         ┌───────────────┐
                       │         │               │
                       └────────►│  LevelEnabler │
                                 │               │
                                 │               │
                                 └───────────────┘
```

* Encoder是日志消息的编码器
* WriteSyncer是支持Sync方法的io.Writer，含义是日志输出的地方，我们可以很方便的通过zap.AddSync将一个io.Writer转换为支持Sync方法的WriteSyncer；
* LevelEnabler则是日志级别相关的参数。

定制日志的输出格式，重点是修改Encoder。

zap内置了两类编码器，一个是ConsoleEncoder，另一个是JSONEncoder。ConsoleEncoder更适合人类阅读，而JSONEncoder更适合机器处理。zap提供的两个最常用创建Logger的函数：NewProduction和NewDevelopment则分别使用了JSONEncoder和ConsoleEncoder。两个编码器默认输出的内容对比如下：

```json
// ConsoleEncoder（NewDevelopment创建)
2021-07-11T09:39:04.418+0800    INFO    zap/testzap2.go:12  failed to fetch URL {"url": "localhost:8080", "attempt": 3, "backoff": "1s"}

// JSONEncoder (NewProduction创建)
{"level":"info","ts":1625968332.269727,"caller":"zap/testzap1.go:12","msg":"failed to fetch URL","url":"localhost:8080","attempt":3,"backoff":1}
```

ConsoleEncoder输出的内容跟适合我们阅读而JSONEncoder输出的结构化日志更适合机器/程序处理。

```go
const (
	logTmFmtWithMS = "2006-01-02 15:04:05.000"
)

func initCore(l *Log) zapcore.Core {
	opts := []zapcore.WriteSyncer{
		zapcore.AddSync(&lumberjack.Logger{
			Filename:  filepath.Join(l.logDir, l.logFileName), // ⽇志⽂件路径
			MaxSize:   l.logMaxSize,                           // 单位为MB,默认为512MB
			MaxAge:    l.logMaxAge,                            // 文件最多保存多少天
			LocalTime: l.localTime,                            // 采用本地时间
			Compress:  l.logCompress,                          // 是否压缩日志
		}),
	}

	if l.stdout {
		opts = append(opts, zapcore.AddSync(os.Stdout))
	}

	syncWriter := zapcore.NewMultiWriteSyncer(opts...)

	// 自定义时间输出格式
	customTimeEncoder := func(t time.Time, enc zapcore.PrimitiveArrayEncoder) {
		enc.AppendString("[" + t.Format(logTmFmtWithMS) + "]")
	}
	// 自定义日志级别显示
	customLevelEncoder := func(level zapcore.Level, enc zapcore.PrimitiveArrayEncoder) {
		enc.AppendString("[" + level.CapitalString() + "]")
	}

	// 自定义文件：行号输出项
	customCallerEncoder := func(caller zapcore.EntryCaller, enc zapcore.PrimitiveArrayEncoder) {
		enc.AppendString("[" + l.traceId + "]")
		enc.AppendString("[" + caller.TrimmedPath() + "]")
	}

	encoderConf := zapcore.EncoderConfig{
		CallerKey:     "caller_line", // 打印文件名和行数
		LevelKey:      "level_name",
		MessageKey:    "msg",
		TimeKey:       "ts",
		StacktraceKey: "stacktrace",
		LineEnding:    zapcore.DefaultLineEnding,
		EncodeTime:     customTimeEncoder,   // 自定义时间格式
		EncodeLevel:    customLevelEncoder,  // 小写编码器
		EncodeCaller:   customCallerEncoder, // 全路径编码器
		EncodeDuration: zapcore.SecondsDurationEncoder,
		EncodeName:     zapcore.FullNameEncoder,
	}

	// level大写染色编码器
	if l.enableColor {
		encoderConf.EncodeLevel = zapcore.CapitalColorLevelEncoder
	}

	// json 格式化处理
	if l.jsonFormat {
		return zapcore.NewCore(zapcore.NewJSONEncoder(encoderConf),
			syncWriter, zap.NewAtomicLevelAt(l.logMinLevel))
	}

	return zapcore.NewCore(zapcore.NewConsoleEncoder(encoderConf),
		syncWriter, zap.NewAtomicLevelAt(l.logMinLevel))
}
```

或者是只修改JsonEncoder

```go
var std = New(os.Stderr, InfoLevel, WithCaller(true))

type Option = zap.Option

var (
    WithCaller    = zap.WithCaller
    AddStacktrace = zap.AddStacktrace
)

// New create a new logger (not support log rotating).
func New(writer io.Writer, level Level, opts ...Option) *Logger {
    if writer == nil {
        panic("the writer is nil")
    }
    cfg := zap.NewProductionConfig()
    cfg.EncoderConfig.EncodeTime = func(t time.Time, enc zapcore.PrimitiveArrayEncoder) {
        enc.AppendString(t.Format("2006-01-02T15:04:05.000Z0700"))
    }

    core := zapcore.NewCore(
        zapcore.NewJSONEncoder(cfg.EncoderConfig),
        zapcore.AddSync(writer),
        zapcore.Level(level),
    )
    logger := &Logger{
        l:     zap.New(core, opts...),
        level: level,
    }
    return logger
}
```

定制后，我们的log包输出的内容就变成了如下这样了：

```go
{"level":"info","ts":"2021-07-11T10:45:38.858+0800","caller":"log/log.go:33","msg":"demo:","app":"start ok"}
```

#### 写入多个文件：access.log and error.log

利用`zap.NewTree()`实现

```go
// NewTee creates a Core that duplicates log entries into two or more
// underlying Cores.
//
// Calling it with a single Core returns the input unchanged, and calling
// it with no input returns a no-op Core.
func NewTee(cores ...Core) Core {
	switch len(cores) {
	case 0:
		return NewNopCore()
	case 1:
		return cores[0]
	default:
		return multiCore(cores)
	}
}
```

下面是一种简单的实现思路：

```go
infoWriter := getWriter("./logs/demo_info.log")
errorWriter := getWriter("./logs/demo_error.log")


func getWriter(filename string) io.Writer {
    // 生成rotatelogs的Logger 实际生成的文件名 demo.log.YYmmddHH
    // demo.log是指向最新日志的链接
    // 保存7天内的日志，每1小时(整点)分割一次日志
    hook, err := rotatelogs.New(
        strings.Replace(filename, ".log", "", -1)+"-%Y%m%d%H.log", // 没有使用go风格反人类的format格式
        //rotatelogs.WithLinkName(filename),
        //rotatelogs.WithMaxAge(time.Hour*24*7),
        //rotatelogs.WithRotationTime(time.Hour),
    )

    if err != nil {
        panic(err)
    }
    return hook
}

core := zapcore.NewTee(
        zapcore.NewCore(encoder, zapcore.AddSync(infoWriter), infoLevel),
        zapcore.NewCore(encoder, zapcore.AddSync(errorWriter), errorLevel),
)

//
log := zap.New(core, zap.AddCaller()) 
errorLogger = log.Sugar()
```



#### tracing and zap

实现方法主要是`context`的`WithValue()`配合`zap`的自定义追加字段特性来处理的，大家同样可以使用自己喜欢的`log`库来实现。

```go
package zlog

import (
	"github.com/gin-gonic/gin"
	"go.uber.org/zap"
	"go.uber.org/zap/zapcore"
	"strconv"
)

// ???
// 每次 const 出现时，都会让 iota 初始化为0.
const loggerKey = iota

var Logger *zap.Logger

// 初始化日志配置

func init() {
	level := zap.DebugLevel
	core := zapcore.NewCore(
		zapcore.NewJSONEncoder(zap.NewProductionEncoderConfig()), // json格式日志（ELK渲染收集）
		zapcore.NewMultiWriteSyncer(zapcore.AddSync(os.Stdout)),  // 打印到控制台和文件
		level,                                                    // 日志级别
	)

	// 开启文件及行号
	development := zap.Development()
	Logger = zap.New(core,
		zap.AddCaller(),
		zap.AddStacktrace(zap.ErrorLevel),	// error级别日志，打印堆栈
		development)
}

// 给指定的context添加字段（关键方法）
// strconv.Itoa函数的参数是一个整型数字，它可以将数字转换成对应的字符串类型的数字。
// demo: NewContext(ctx, zap.String("traceId", traceId))
func NewContext(ctx *gin.Context, fields ...zapcore.Field) {
	ctx.Set(strconv.Itoa(loggerKey), WithContext(ctx).With(fields...))
}

// 从指定的context返回一个zap实例（关键方法）
func WithContext(ctx *gin.Context) *zap.Logger {
	if ctx == nil {
		return Logger
	}
    
	l, _ := ctx.Get(strconv.Itoa(loggerKey))
    //fmt.Printf("%+v\n", ctx)
	//fmt.Printf("%T\n", l)
	//fmt.Printf("%+v\n", l)
    // var a interface{}
    // value, ok := a.(string)
    // type assertion
	ctxLogger, ok := l.(*zap.Logger)
	if ok {
		return ctxLogger
	}
	return Logger
}


```

0.0 没看懂

```go
// gin.Context
// Set is used to store a new key/value pair exclusively for this context.
// It also lazy initializes  c.Keys if it was not used previously.
func (c *Context) Set(key string, value any) {
	c.mu.Lock()
	if c.Keys == nil {
		c.Keys = make(map[string]any)
	}

	c.Keys[key] = value
	c.mu.Unlock()
}

// Get returns the value for the given key, ie: (value, true).
// If the value does not exist it returns (nil, false)
// l, _ := ctx.Get(strconv.Itoa(loggerKey))
func (c *Context) Get(key string) (value any, exists bool) {
	c.mu.RLock()
	value, exists = c.Keys[key]
	c.mu.RUnlock()
	return
}

// With creates a child logger and adds structured context to it. Fields added
// to the child don't affect the parent, and vice versa.
func (log *Logger) With(fields ...Field) *Logger {
	if len(fields) == 0 {
		return log
	}
	l := log.clone()
	l.core = l.core.With(fields)
	return l
}

//zap.Field
type Field struct {
	Key       string
	Type      FieldType
	Integer   int64
	String    string
	Interface interface{}
}
```

通过中间件，我们可以方便的为每个request添加日志上下文关键字段，这儿只添加了traceId和一些请求信息字段，我们还可以根据应用场景添加其它自定义字段。

```go
func traceLoggerMiddleware() gin.HandlerFunc {
	return func(ctx *gin.Context) {
	    // 每个请求生成的请求traceId具有全局唯一性
		u1, _ := uuid.NewV4()
		traceId := u1.String()
		zlog.NewContext(ctx, zap.String("traceId", traceId))
		
		// 为日志添加请求的地址以及请求参数等信息
		zlog.NewContext(ctx, zap.String("request.method", ctx.Request.Method))
		headers, _ := json.Marshal(ctx.Request.Header)
		zlog.NewContext(ctx, zap.String("request.headers", string(headers)))
		zlog.NewContext(ctx, zap.String("request.url", ctx.Request.URL.String()))
		
		// 将请求参数json序列化后添加进日志上下文
		if ctx.Request.Form == nil {
			ctx.Request.ParseMultipartForm(32 << 20)
		}
		form, _ := json.Marshal(ctx.Request.Form)
		zlog.NewContext(ctx, zap.String("request.params", string(form)))
		
		ctx.Next()
	}
}

```

测试demo

```go
package main

import (
	"encoding/json"
	"github.com/gin-gonic/gin"
	"github.com/gofrs/uuid"
	"go.uber.org/zap"
	"test/zlog"
)

type Test struct {
}

func (t Test) Test1(ctx *gin.Context) {
	name := ctx.Query("name")
	// 注意打印日志都需要通过WithContext(ctx)来获得zapLogger

	zlog.WithContext(ctx).Debug("测试日志", zap.String("name", name))
	ctx.String(200, "test")
}
func main()  {
	r := gin.Default()
    // 使用中间件
	r.Use(traceLoggerMiddleware())
	r.GET("/ping", func(c *gin.Context) {
		zlog.WithContext(c).Debug("测试日志", zap.String("name", "Robin"))
		c.String(200, "pong")
	})
	r.GET("/test", Test{}.Test1)

	r.Run(":8080")
}
```



#### Wrap zap as log package

封装下zap让它和golang package log一样好用、

实现说明：

- 参考标准库log包，我们提供包级函数接口，底层是创建的默认Logger: std；

- 你可以使用New函数创建了自己的Logger变量，但此时只能使用该实例的方法实现log输出，如果期望使用包级函数接口输出log，需要调用ResetDefault替换更新std实例的值，这样后续调用包级函数(Info、Debug）等就会输出到新实例的目标io.Writer中了。不过最好在输出任何日志前调用ResetDefault换掉std；

- 由于zap在输出log时要告知具体类型，zap封装出了Field以及一些sugar函数(Int、String等)，这里为了不暴露zap给用户，我们使用[type alias语法](https://tip.golang.org/ref/spec#Type_declarations)定义了我们自己的等价于zap.Field的类型log.Field：

  `type Field = zap.Field`

- 基于[“函数是一等公民”](https://www.imooc.com/read/87/article/2420)的特性，将zap的一些配合log输出的sugar函数（Int、String等）暴露给用户（这也是[Go单元测试在export_test.go经常用到的方法](https://www.imooc.com/read/87/article/2436)）：

  ```go
  var (
      Skip        = zap.Skip
      Binary      = zap.Binary
      Bool        = zap.Bool
      Boolp       = zap.Boolp
      ByteString  = zap.ByteString
      ... ...
  )
  ```

- 使用[method value语法](https://tip.golang.org/ref/spec#Method_values)将std实例的各个方法以包级函数的形式暴露给用户，简化用户对logger实例的获取

  ```go
  var (
      Info   = std.Info
      Warn   = std.Warn
      Error  = std.Error
      DPanic = std.DPanic
      Panic  = std.Panic
      Fatal  = std.Fatal
      Debug  = std.Debug
  )
  ```

- end



```go
package lo

import (
"io"
"os"

"go.uber.org/zap"
"go.uber.org/zap/zapcore"
)

type Level = zapcore.Level

type Logger struct {
	l     *zap.Logger // zap ensure that zap.Logger is safe for concurrent use
	level Level
}

type Field = zap.Field

const (
	InfoLevel   Level = zap.InfoLevel   // 0, default level
	WarnLevel   Level = zap.WarnLevel   // 1
	ErrorLevel  Level = zap.ErrorLevel  // 2
	DPanicLevel Level = zap.DPanicLevel // 3, used in development log
	// PanicLevel logs a message, then panics
	PanicLevel Level = zap.PanicLevel // 4
	// FatalLevel logs a message, then calls os.Exit(1).
	FatalLevel Level = zap.FatalLevel // 5
	DebugLevel Level = zap.DebugLevel // -1
)
// function variables for all field types
// in github.com/uber-go/zap/field.go

var (
	Skip        = zap.Skip
	Binary      = zap.Binary
	Bool        = zap.Bool
	Boolp       = zap.Boolp
	ByteString  = zap.ByteString
	Complex128  = zap.Complex128
	Complex128p = zap.Complex128p
	Complex64   = zap.Complex64
	Complex64p  = zap.Complex64p
	Float64     = zap.Float64
	Float64p    = zap.Float64p
	Float32     = zap.Float32
	Float32p    = zap.Float32p
	Int         = zap.Int
	Intp        = zap.Intp
	Int64       = zap.Int64
	Int64p      = zap.Int64p
	Int32       = zap.Int32
	Int32p      = zap.Int32p
	Int16       = zap.Int16
	Int16p      = zap.Int16p
	Int8        = zap.Int8
	Int8p       = zap.Int8p
	String      = zap.String
	Stringp     = zap.Stringp
	Uint        = zap.Uint
	Uintp       = zap.Uintp
	Uint64      = zap.Uint64
	Uint64p     = zap.Uint64p
	Uint32      = zap.Uint32
	Uint32p     = zap.Uint32p
	Uint16      = zap.Uint16
	Uint16p     = zap.Uint16p
	Uint8       = zap.Uint8
	Uint8p      = zap.Uint8p
	Uintptr     = zap.Uintptr
	Uintptrp    = zap.Uintptrp
	Reflect     = zap.Reflect
	Namespace   = zap.Namespace
	Stringer    = zap.Stringer
	Time        = zap.Time
	Timep       = zap.Timep
	Stack       = zap.Stack
	StackSkip   = zap.StackSkip
	Duration    = zap.Duration
	Durationp   = zap.Durationp
	Any         = zap.Any

	Info   = std.Info
	Warn   = std.Warn
	Error  = std.Error
	DPanic = std.DPanic
	Panic  = std.Panic
	Fatal  = std.Fatal
	Debug  = std.Debug
)


var std = New(os.Stderr, InfoLevel)

// New create a new logger (not support log rotating).
func New(writer io.Writer, level Level) *Logger {
	if writer == nil {
		panic("the writer is nil")
	}
	zap.NewProduction()
	cfg := zap.NewProductionConfig()
	core := zapcore.NewCore(
		zapcore.NewJSONEncoder(cfg.EncoderConfig),
		zapcore.AddSync(writer),
		zapcore.Level(level),
	)
	logger := &Logger{
		l:     zap.New(core),
		level: level,
	}
	return logger
}


func (l *Logger) Debug(msg string, fields ...Field) {
	l.l.Debug(msg, fields...)
}

func (l *Logger) Info(msg string, fields ...Field) {
	l.l.Info(msg, fields...)
}

func (l *Logger) Warn(msg string, fields ...Field) {
	l.l.Warn(msg, fields...)
}

func (l *Logger) Error(msg string, fields ...Field) {
	l.l.Error(msg, fields...)
}
func (l *Logger) DPanic(msg string, fields ...Field) {
	l.l.DPanic(msg, fields...)
}
func (l *Logger) Panic(msg string, fields ...Field) {
	l.l.Panic(msg, fields...)
}
func (l *Logger) Fatal(msg string, fields ...Field) {
	l.l.Fatal(msg, fields...)
}

// not safe for concurrent use
func ResetDefault(l *Logger) {
	std = l
	Info = std.Info
	Warn = std.Warn
	Error = std.Error
	DPanic = std.DPanic
	Panic = std.Panic
	Fatal = std.Fatal
	Debug = std.Debug
}



func Default() *Logger {
	return std
}



func (l *Logger) Sync() error {
	return l.l.Sync()
}

func Sync() error {
	if std != nil {
		return std.Sync()
	}
	return nil
}

```



#### log rotate：lumberjack

log rotate方案通常有两种，一种是基于logrotate工具的外部方案，一种是log库自身支持轮转。[zap库与logrotate工具的兼容性似乎有些问题](https://github.com/uber-go/zap/issues/797)，zap[官方FAQ也推荐第二种方案](https://github.com/uber-go/zap/blob/master/FAQ.md#does-zap-support-log-rotation)。

Zap doesn't natively support rotating log files, since we prefer to leave this to an external program like `logrotate`.

However, it's easy to integrate a log rotation package like [`gopkg.in/natefinch/lumberjack.v2`](https://godoc.org/gopkg.in/natefinch/lumberjack.v2) as a `zapcore.WriteSyncer`.

```go
// AddSync converts an io.Writer to a WriteSyncer. It attempts to be
// intelligent: if the concrete type of the io.Writer implements WriteSyncer,
// we'll use the existing Sync method. If it doesn't, we'll add a no-op Sync.
func AddSync(w io.Writer) WriteSyncer {
	switch w := w.(type) {
	case WriteSyncer:
		return w
	default:
		return writerWrapper{w}
	}
}
```

zap并不是原生支持rotate，而是通过外部包来支持，zap提供了WriteSyncer接口可以方便我们为zap加入rotate功能。目前在支持logrotate方面，natefinch的lumberjack是应用最为官方的包

```go
package main

import (
	"io"
	"os"
	"time"

	"go.uber.org/zap"
	"go.uber.org/zap/zapcore"
	"gopkg.in/natefinch/lumberjack.v2"
)

type Level = zapcore.Level

const (
	InfoLevel   Level = zap.InfoLevel   // 0, default level
	WarnLevel   Level = zap.WarnLevel   // 1
	ErrorLevel  Level = zap.ErrorLevel  // 2
	DPanicLevel Level = zap.DPanicLevel // 3, used in development log
	// PanicLevel logs a message, then panics
	PanicLevel Level = zap.PanicLevel // 4
	// FatalLevel logs a message, then calls os.Exit(1).
	FatalLevel Level = zap.FatalLevel // 5
	DebugLevel Level = zap.DebugLevel // -1
)

type Field = zap.Field

func (l *Logger) Debug(msg string, fields ...Field) {
	l.l.Debug(msg, fields...)
}

func (l *Logger) Info(msg string, fields ...Field) {
	l.l.Info(msg, fields...)
}

func (l *Logger) Warn(msg string, fields ...Field) {
	l.l.Warn(msg, fields...)
}

func (l *Logger) Error(msg string, fields ...Field) {
	l.l.Error(msg, fields...)
}
func (l *Logger) DPanic(msg string, fields ...Field) {
	l.l.DPanic(msg, fields...)
}
func (l *Logger) Panic(msg string, fields ...Field) {
	l.l.Panic(msg, fields...)
}
func (l *Logger) Fatal(msg string, fields ...Field) {
	l.l.Fatal(msg, fields...)
}

// function variables for all field types
// in github.com/uber-go/zap/field.go

var (
	Skip        = zap.Skip
	Binary      = zap.Binary
	Bool        = zap.Bool
	Boolp       = zap.Boolp
	ByteString  = zap.ByteString
	Complex128  = zap.Complex128
	Complex128p = zap.Complex128p
	Complex64   = zap.Complex64
	Complex64p  = zap.Complex64p
	Float64     = zap.Float64
	Float64p    = zap.Float64p
	Float32     = zap.Float32
	Float32p    = zap.Float32p
	Int         = zap.Int
	Intp        = zap.Intp
	Int64       = zap.Int64
	Int64p      = zap.Int64p
	Int32       = zap.Int32
	Int32p      = zap.Int32p
	Int16       = zap.Int16
	Int16p      = zap.Int16p
	Int8        = zap.Int8
	Int8p       = zap.Int8p
	String      = zap.String
	Stringp     = zap.Stringp
	Uint        = zap.Uint
	Uintp       = zap.Uintp
	Uint64      = zap.Uint64
	Uint64p     = zap.Uint64p
	Uint32      = zap.Uint32
	Uint32p     = zap.Uint32p
	Uint16      = zap.Uint16
	Uint16p     = zap.Uint16p
	Uint8       = zap.Uint8
	Uint8p      = zap.Uint8p
	Uintptr     = zap.Uintptr
	Uintptrp    = zap.Uintptrp
	Reflect     = zap.Reflect
	Namespace   = zap.Namespace
	Stringer    = zap.Stringer
	Time        = zap.Time
	Timep       = zap.Timep
	Stack       = zap.Stack
	StackSkip   = zap.StackSkip
	Duration    = zap.Duration
	Durationp   = zap.Durationp
	Any         = zap.Any

	Info   = std.Info
	Warn   = std.Warn
	Error  = std.Error
	DPanic = std.DPanic
	Panic  = std.Panic
	Fatal  = std.Fatal
	Debug  = std.Debug
)

// not safe for concurrent use
func ResetDefault(l *Logger) {
	std = l
	Info = std.Info
	Warn = std.Warn
	Error = std.Error
	DPanic = std.DPanic
	Panic = std.Panic
	Fatal = std.Fatal
	Debug = std.Debug
}

type Logger struct {
	l     *zap.Logger // zap ensure that zap.Logger is safe for concurrent use
	level Level
}

var std = New(os.Stderr, InfoLevel, WithCaller(true))

func Default() *Logger {
	return std
}

type Option = zap.Option

var (
	WithCaller    = zap.WithCaller
	AddStacktrace = zap.AddStacktrace
)

type RotateOptions struct {
	MaxSize    int
	MaxAge     int
	MaxBackups int
	Compress   bool
}

type LevelEnablerFunc func(lvl Level) bool

type TeeOption struct {
	Filename string
	Ropt     RotateOptions
	Lef      LevelEnablerFunc
}

func NewTeeWithRotate(tops []TeeOption, opts ...Option) *Logger {
	var cores []zapcore.Core
	cfg := zap.NewProductionConfig()
	cfg.EncoderConfig.EncodeTime = func(t time.Time, enc zapcore.PrimitiveArrayEncoder) {
		enc.AppendString(t.Format("2006-01-02T15:04:05.000Z0700"))
	}

	for _, top := range tops {
		top := top

		lv := zap.LevelEnablerFunc(func(lvl zapcore.Level) bool {
			return top.Lef(Level(lvl))
		})

		w := zapcore.AddSync(&lumberjack.Logger{
			Filename:   top.Filename,
			MaxSize:    top.Ropt.MaxSize,
			MaxBackups: top.Ropt.MaxBackups,
			MaxAge:     top.Ropt.MaxAge,
			Compress:   top.Ropt.Compress,
		})

		core := zapcore.NewCore(
			zapcore.NewJSONEncoder(cfg.EncoderConfig),
			zapcore.AddSync(w),
			lv,
		)
		cores = append(cores, core)
	}

	logger := &Logger{
		l: zap.New(zapcore.NewTee(cores...), opts...),
	}
	return logger
}

// New create a new logger (not support log rotating).
func New(writer io.Writer, level Level, opts ...Option) *Logger {
	if writer == nil {
		panic("the writer is nil")
	}
	cfg := zap.NewProductionConfig()
	cfg.EncoderConfig.EncodeTime = func(t time.Time, enc zapcore.PrimitiveArrayEncoder) {
		enc.AppendString(t.Format("2006-01-02T15:04:05.000Z0700"))
	}

	core := zapcore.NewCore(
		zapcore.NewJSONEncoder(cfg.EncoderConfig),
		zapcore.AddSync(writer),
		zapcore.Level(level),
	)
	logger := &Logger{
		l:     zap.New(core, opts...),
		level: level,
	}
	return logger
}

func (l *Logger) Sync() error {
	return l.l.Sync()
}

func Sync() error {
	if std != nil {
		return std.Sync()
	}
	return nil
}

func main() {
	var tops = []TeeOption{
		{
			Filename: "access.log",
			Ropt: RotateOptions{
				MaxSize:    1,
				MaxAge:     1,
				MaxBackups: 3,
				Compress:   true,
			},
			Lef: func(lvl Level) bool {
				return lvl <= InfoLevel
			},
		},
		{
			Filename: "error.log",
			Ropt: RotateOptions{
				MaxSize:    1,
				MaxAge:     1,
				MaxBackups: 3,
				Compress:   true,
			},
			Lef: func(lvl Level) bool {
				return lvl > InfoLevel
			},
		},
	}

	logger := NewTeeWithRotate(tops)
	ResetDefault(logger)

	for i := 0; i < 20000; i++ {
		Info("demo3:", String("app", "start ok"),
			Int("major version", 3))
		Error("demo3:", String("app", "crash"),
			Int("reason", -1))
	}

}
```

在TeeOption中加入了RotateOptions（当然这种绑定并非必须)，并使用lumberjack.Logger作为io.Writer传给zapcore.AddSync，这样创建出来的logger既有写多日志文件的能力，又让每种日志文件具备了自动rotate的功能。

##### gin with zap and lumberjack

只需要给zap添加一个lumberjack的WriteSyncer即可。

```go
package zlog

import (
	"fmt"
	"github.com/gin-gonic/gin"
	"go.uber.org/zap"
	"go.uber.org/zap/zapcore"
	"gopkg.in/natefinch/lumberjack.v2"
	"os"
	"strconv"
)

const loggerKey = iota

var Logger *zap.Logger

// 初始化日志配置

func init() {
	level := zap.DebugLevel
    
    // 
	w := zapcore.AddSync(&lumberjack.Logger{
		Filename:   "testAccess.log",
		MaxSize:    1,
		MaxBackups: 1,
		MaxAge:     1,
		Compress:   true,
	})

	core := zapcore.NewCore(
        // json格式日志
		zapcore.NewJSONEncoder(zap.NewProductionEncoderConfig()), 
        // 打印到控制台和文件
		zapcore.NewMultiWriteSyncer(zapcore.AddSync(os.Stdout), zapcore.AddSync(w)), 
        // 日志级别
		level,                                                   
	)

	// 开启文件及行号
	development := zap.Development()
	Logger = zap.New(core,
		zap.AddCaller(),
		zap.AddStacktrace(zap.ErrorLevel),	// error级别日志，打印堆栈
		development)
}
```

##### 大佬的demo

```go
package log

import (
	"io"
	"os"
	"time"

	"go.uber.org/zap"
	"go.uber.org/zap/zapcore"
	"gopkg.in/natefinch/lumberjack.v2"
)

type Level = zapcore.Level

const (
	InfoLevel   Level = zap.InfoLevel   // 0, default level
	WarnLevel   Level = zap.WarnLevel   // 1
	ErrorLevel  Level = zap.ErrorLevel  // 2
	DPanicLevel Level = zap.DPanicLevel // 3, used in development log
	// PanicLevel logs a message, then panics
	PanicLevel Level = zap.PanicLevel // 4
	// FatalLevel logs a message, then calls os.Exit(1).
	FatalLevel Level = zap.FatalLevel // 5
	DebugLevel Level = zap.DebugLevel // -1
)

type Field = zap.Field

func (l *Logger) Debug(msg string, fields ...Field) {
	l.l.Debug(msg, fields...)
}

func (l *Logger) Info(msg string, fields ...Field) {
	l.l.Info(msg, fields...)
}

func (l *Logger) Warn(msg string, fields ...Field) {
	l.l.Warn(msg, fields...)
}

func (l *Logger) Error(msg string, fields ...Field) {
	l.l.Error(msg, fields...)
}
func (l *Logger) DPanic(msg string, fields ...Field) {
	l.l.DPanic(msg, fields...)
}
func (l *Logger) Panic(msg string, fields ...Field) {
	l.l.Panic(msg, fields...)
}
func (l *Logger) Fatal(msg string, fields ...Field) {
	l.l.Fatal(msg, fields...)
}

// function variables for all field types
// in github.com/uber-go/zap/field.go

var (
	Skip        = zap.Skip
	Binary      = zap.Binary
	Bool        = zap.Bool
	Boolp       = zap.Boolp
	ByteString  = zap.ByteString
	Complex128  = zap.Complex128
	Complex128p = zap.Complex128p
	Complex64   = zap.Complex64
	Complex64p  = zap.Complex64p
	Float64     = zap.Float64
	Float64p    = zap.Float64p
	Float32     = zap.Float32
	Float32p    = zap.Float32p
	Int         = zap.Int
	Intp        = zap.Intp
	Int64       = zap.Int64
	Int64p      = zap.Int64p
	Int32       = zap.Int32
	Int32p      = zap.Int32p
	Int16       = zap.Int16
	Int16p      = zap.Int16p
	Int8        = zap.Int8
	Int8p       = zap.Int8p
	String      = zap.String
	Stringp     = zap.Stringp
	Uint        = zap.Uint
	Uintp       = zap.Uintp
	Uint64      = zap.Uint64
	Uint64p     = zap.Uint64p
	Uint32      = zap.Uint32
	Uint32p     = zap.Uint32p
	Uint16      = zap.Uint16
	Uint16p     = zap.Uint16p
	Uint8       = zap.Uint8
	Uint8p      = zap.Uint8p
	Uintptr     = zap.Uintptr
	Uintptrp    = zap.Uintptrp
	Reflect     = zap.Reflect
	Namespace   = zap.Namespace
	Stringer    = zap.Stringer
	Time        = zap.Time
	Timep       = zap.Timep
	Stack       = zap.Stack
	StackSkip   = zap.StackSkip
	Duration    = zap.Duration
	Durationp   = zap.Durationp
	Any         = zap.Any

	Info   = std.Info
	Warn   = std.Warn
	Error  = std.Error
	DPanic = std.DPanic
	Panic  = std.Panic
	Fatal  = std.Fatal
	Debug  = std.Debug
)

// not safe for concurrent use
func ResetDefault(l *Logger) {
	std = l
	Info = std.Info
	Warn = std.Warn
	Error = std.Error
	DPanic = std.DPanic
	Panic = std.Panic
	Fatal = std.Fatal
	Debug = std.Debug
}

type Logger struct {
	l     *zap.Logger // zap ensure that zap.Logger is safe for concurrent use
	level Level
}

var std = New(os.Stderr, InfoLevel, WithCaller(true))

func Default() *Logger {
	return std
}

type Option = zap.Option

var (
	WithCaller    = zap.WithCaller
	AddStacktrace = zap.AddStacktrace
)

type RotateOptions struct {
	MaxSize    int
	MaxAge     int
	MaxBackups int
	Compress   bool
}

type LevelEnablerFunc func(lvl Level) bool

type TeeOption struct {
	Filename string
	Ropt     RotateOptions
	Lef      LevelEnablerFunc
}

func NewTeeWithRotate(tops []TeeOption, opts ...Option) *Logger {
	var cores []zapcore.Core
	cfg := zap.NewProductionConfig()
	cfg.EncoderConfig.EncodeTime = func(t time.Time, enc zapcore.PrimitiveArrayEncoder) {
		enc.AppendString(t.Format("2006-01-02T15:04:05.000Z0700"))
	}

	for _, top := range tops {
		top := top

		lv := zap.LevelEnablerFunc(func(lvl zapcore.Level) bool {
			return top.Lef(Level(lvl))
		})

		w := zapcore.AddSync(&lumberjack.Logger{
			Filename:   top.Filename,
			MaxSize:    top.Ropt.MaxSize,
			MaxBackups: top.Ropt.MaxBackups,
			MaxAge:     top.Ropt.MaxAge,
			Compress:   top.Ropt.Compress,
		})

		core := zapcore.NewCore(
			zapcore.NewJSONEncoder(cfg.EncoderConfig),
			zapcore.AddSync(w),
			lv,
		)
		cores = append(cores, core)
	}

	logger := &Logger{
		l: zap.New(zapcore.NewTee(cores...), opts...),
	}
	return logger
}

// New create a new logger (not support log rotating).
func New(writer io.Writer, level Level, opts ...Option) *Logger {
	if writer == nil {
		panic("the writer is nil")
	}
	cfg := zap.NewProductionConfig()
	cfg.EncoderConfig.EncodeTime = func(t time.Time, enc zapcore.PrimitiveArrayEncoder) {
		enc.AppendString(t.Format("2006-01-02T15:04:05.000Z0700"))
	}

	core := zapcore.NewCore(
		zapcore.NewJSONEncoder(cfg.EncoderConfig),
		zapcore.AddSync(writer),
		zapcore.Level(level),
	)
	logger := &Logger{
		l:     zap.New(core, opts...),
		level: level,
	}
	return logger
}

func (l *Logger) Sync() error {
	return l.l.Sync()
}

func Sync() error {
	if std != nil {
		return std.Sync()
	}
	return nil
}


// main 
package main

import (
	"github.com/bigwhite/zap-usage/pkg/log"
)

func main() {
	var tops = []log.TeeOption{
		{
			Filename: "access.log",
			Ropt: log.RotateOptions{
				MaxSize:    1,
				MaxAge:     1,
				MaxBackups: 3,
				Compress:   true,
			},
			Lef: func(lvl log.Level) bool {
				return lvl <= log.InfoLevel
			},
		},
		{
			Filename: "error.log",
			Ropt: log.RotateOptions{
				MaxSize:    1,
				MaxAge:     1,
				MaxBackups: 3,
				Compress:   true,
			},
			Lef: func(lvl log.Level) bool {
				return lvl > log.InfoLevel
			},
		},
	}

	logger := log.NewTeeWithRotate(tops)
	log.ResetDefault(logger)

	for i := 0; i < 20000; i++ {
		log.Info("demo3:", log.String("app", "start ok"),
			log.Int("major version", 3))
		log.Error("demo3:", log.String("app", "crash"),
			log.Int("reason", -1))
	}

}
```

end



### 概述

https://www.liwenzhou.com/posts/Go/use_zap_in_gin/#autoid-0-0-1

### 引用

1. https://tonybai.com/2021/07/14/uber-zap-advanced-usage/
2. https://cloud.tencent.com/developer/article/1811437
3. https://tehub.com/a/3RsRiVgWHc
4. https://www.liwenzhou.com/posts/Go/use_zap_in_gin/#autoid-0-0-1
