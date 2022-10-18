## Golang gin-gonic web framework

### gin 101

https://medium.com/search?q=gin+101





### go gin-gonic Manager

https://github.com/gin-gonic/gin

#### go build  with json replacement

Gin uses `encoding/json` as default json package but you can change it by build from other tags.

> `encoding/json`是golang原生json库

```bash
# go-json
$ go build -tags=go_json .
## 可以直接运行监听
$ ./test
...

[GIN-debug] GET    /ping                     --> main.main.func1 (3 handlers)
[GIN-debug] [WARNING] You trusted all proxies, this is NOT safe. We recommend you to set a value.
Please check https://pkg.go.dev/github.com/gin-gonic/gin#readme-don-t-trust-all-proxies for details.
[GIN-debug] Environment variable PORT is undefined. Using port :8080 by default
[GIN-debug] Listening and serving HTTP on :8080
...
```

### rk-boot:?

https://github.com/rookie-ninja/rk-boot

项目管理？

### middleware

`Gin`中最终处理请求的逻辑是在`engine.handleHTTPRequest()` 这个函数

```go
func (engine *Engine) handleHTTPRequest(c *Context) {
	// ...

	// Find root of the tree for the given HTTP method
	t := engine.trees
	for i, tl := 0, len(t); i < tl; i++ {
		if t[i].method != httpMethod {
			continue
		}
		root := t[i].root
		// Find route in tree
		value := root.getValue(rPath, c.params, unescape)
		if value.params != nil {
			c.Params = *value.params
		}
		if value.handlers != nil {
			c.handlers = value.handlers
			c.fullPath = value.fullPath
			c.Next() //执行handlers
			c.writermem.WriteHeaderNow()
			return
		}
		// ...
		}
		break
	}

// ... 
}
```



其中`c.Next()` 是关键

```go
// Next should be used only inside middleware.
// It executes the pending handlers in the chain inside the calling handler.
// See example in GitHub.
func (c *Context) Next() {
	c.index++
	for c.index < int8(len(c.handlers)) {
		c.handlers[c.index](c) //执行handler
		c.index++
	}
}
```

从`Next()`方法我们可以看到它会遍历执行全部`handlers`（中间件也是`handler`），所以中间件中调不调用`Next()`方法并不会影响后续中间件的执行。

既然中间件中没有`Next()`不影响后续中间件的执行，那么在当前中间件中调用`c.Next()`的作用又是什么呢？

通过`Next()`函数的逻辑也能很清晰的得出结论：**在当前中间件中调用`c.Next()`时会中断当前中间件中后续的逻辑，转而执行后续的中间件和handlers，等他们全部执行完以后再回来执行当前中间件的后续代码。**

结论

1. **中间件代码最后即使没有调用`Next()`方法，后续中间件及`handlers`也会执行；**
2. **如果在中间件函数的非结尾调用`Next()`方法当前中间件剩余代码会被暂停执行，会先去执行后续中间件及`handlers`，等这些`handlers`全部执行完以后程序控制权会回到当前中间件继续执行剩余代码；**
3. **如果想提前中止当前中间件的执行应该使用`return`退出而不是`Next()`方法；**
4. **如果想中断剩余中间件及handlers应该使用`Abort`方法，但需要注意当前中间件的剩余代码会继续执行。**

### Restful API

https://go.dev/doc/tutorial/web-service-gin

```go
package main

import (
    "log"

    "github.com/carlosm27/apiwithgorm/grocery"
    "github.com/carlosm27/apiwithgorm/model"
    "github.com/gin-gonic/gin"
)

func main() {

    db, err := model.Database()
    if err != nil {
        log.Println(err)
    }
    db.DB()

    router := gin.Default()
	// sample example
    router.GET("/groceries", grocery.GetGroceries)
    // curl IP:port/grocery/8 ==> gin.Context.Param("id")获取参数
    // Param returns the value of the URL param.
	// It is a shortcut for c.Params.ByName(key)
	//     router.GET("/user/:id", func(c *gin.Context) {
	//         // a GET request to /user/john
	//         id := c.Param("id") // id == "john"
	//     })
    router.GET("/grocery/:id", grocery.GetGrocery)
    router.POST("/grocery", grocery.PostGrocery)
    router.PUT("/grocery/:id", grocery.UpdateGrocery)
    router.DELETE("/grocery/:id", grocery.DeleteGrocery)

    log.Fatal(router.Run(":10000"))
}

// modle
package model 
import ( 
	"gorm.io/gorm" 
) 
type Grocery struct { 
	gorm.Model 
	Name string `json:"name"` 
	Quantity int `json:"quantity"` 
}
// db
package model

import (
    "gorm.io/driver/sqlite"
    "gorm.io/gorm"
)


func Database() (*gorm.DB, error) {

    db, err := gorm.Open(sqlite.Open("./database.db"), &gorm.Config{})

    if err != nil {
        log.Fatal(err.Error())
    }

    if err = db.AutoMigrate(&Grocery{}); err != nil {
        log.Println(err)
    }


    return db, err

}

// struct
package grocery

import (
    "net/http"

    "github.com/carlosm27/apiwithgorm/model"
    "github.com/gin-gonic/gin"
)

type NewGrocery struct {
    Name     string `json:"name" binding:"required"`
    Quantity int    `json:"quantity" binding:"required"`
}

type GroceryUpdate struct {
    Name     string `json:"name"`
    Quantity int    `json:"quantity"`
}

// handler func

func PostGrocery(c *gin.Context) {

    var grocery NewGrocery

    if err := c.ShouldBindJSON(&grocery); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }

    newGrocery := model.Grocery{Name: grocery.Name, Quantity: grocery.Quantity}

    db, err := model.Database()
    if err != nil {
        log.Println(err)
    }

    if err := db.Create(&newGrocery).Error; err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
        return
    }

    c.JSON(http.StatusOK, newGrocery)
}

func DeleteGrocery(c *gin.Context) {

    var grocery model.Grocery

    db, err := model.Database()
    if err != nil {
        log.Println(err)
    }

    if err := db.Where("id = ?", c.Param("id")).First(&grocery).Error; err != nil {
        c.JSON(http.StatusNotFound, gin.H{"error": "Grocery not found!"})
        return
    }

    if err := db.Delete(&grocery).Error; err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
        return
    }
    c.JSON(http.StatusOK, gin.H{"message": "Grocery deleted"})

}
// getAlbumByID locates the album whose ID value matches the id
// parameter sent by the client, then returns that album as a response.
func getAlbumByID(c *gin.Context) {
    id := c.Param("id")

    // Loop through the list of albums, looking for
    // an album whose ID value matches the parameter.
    for _, a := range albums {
        if a.ID == id {
            c.IndentedJSON(http.StatusOK, a)
            return
        }
    }
    c.IndentedJSON(http.StatusNotFound, gin.H{"message": "album not found"})
}
```

通常：

* model 目录下放 strut（db table）
* handler目录下放每个model对应的增删改查
* service目录下放multi-model的关联操作，比如根据app id 获取group，owner，
* api目录就是接口

#### 路由组

```go
package main

import (
	"github.com/gin-gonic/gin"
)

//路由
func main() {
	r := gin.Default()
	//路由组1， 处理GET请求
	v1 := r.Group("/v1")
	{
		v1.GET("/", Index)
		v1.GET("/test", Test)
	}
	//路由组2，处理Post请求
	v2 := r.Group("/v2")
	{
		v2.POST("/login", Login)
		v2.POST("/submit", Submit)
	}
    r.Run(":8080")
}

```



### ORM

https://gorm.io/docs/

#### Mysql scheme

对于mysql，schema和database可以理解为等价的.

As defined in the MySQL Glossary:

*In MySQL, physically, a schema is synonymous with a database. You can substitute the keyword SCHEMA instead of DATABASE in MySQL SQL syntax, for example using CREATE SCHEMA instead of CREATE DATABASE.*



*Some other database products draw a distinction. For example, in the Oracle Database product, a schema represents only a part of a database: the tables and other objects owned by a single user.*

对于mysql，schema和database可以理解为等价的.

As defined in the MySQL Glossary:

*In MySQL, physically, a schema is synonymous with a database. You can substitute the keyword SCHEMA instead of DATABASE in MySQL SQL syntax, for example using CREATE SCHEMA instead of CREATE DATABASE.*



Some other database products draw a distinction. For example, in the Oracle Database product, a schema represents only a part of a database: the tables and other objects owned by a single user.

#### go-gorm demo

https://betterprogramming.pub/building-a-rest-api-with-go-gin-framework-and-gorm-38cb2d6353da

```go
package main

import (
"gorm.io/gorm"
"gorm.io/driver/mysql"
)
type Product struct {
        Code  string
        Price uint
}

func main() {
        dsn := "root:123456@tcp(172.17.0.4:3306)/test?charset=utf8mb4&parseTime=True&loc=Local"
        db, err := gorm.Open(mysql.Open(dsn), &gorm.Config{})
        if err != nil {
                panic("failed to connect database")
        }

        // Migrate the schema
        // 根据struct创建table
        db.AutoMigrate(&Product{})

        // Create
        db.Create(&Product{Code: "D42", Price: 100})

        // Read
        var product Product
        db.First(&product, 1)                 // find product with integer primary key
        db.First(&product, "code = ?", "D42") // find product with code D42

        // Update - update product's price to 200
        db.Model(&product).Update("Price", 200)
        // Update - update multiple fields
        db.Model(&product).Updates(Product{Price: 200, Code: "F42"}) // non-zero fields
        db.Model(&product).Updates(map[string]interface{}{"Price": 200, "Code": "F42"})

        // Delete - delete product
        db.Delete(&product, 1)
}

```

#### go-gorm demo 2

```go
package Database

import (
   "fmt"
   "gorm.io/driver/mysql"
   "gorm.io/gorm"
   "gorm.io/gorm/schema"
   "time"
   "ruiyi/plum/gin/Common"
)

var DB *gorm.DB

func MysqlInit() (err error) {
   mysqlConfig := Common.MysqlInfo
   DB, err = gorm.Open(mysql.New(mysql.Config{
      DSN: fmt.Sprintf("%s:%s@tcp(%s:%d)/%s?charset=utf8&parseTime=True&loc=Local", mysqlConfig.User,
         mysqlConfig.Password, mysqlConfig.Host, mysqlConfig.Port, mysqlConfig.Database),
      DefaultStringSize:         256,   // string 类型字段的默认长度
      DisableDatetimePrecision:  true,  // 禁用 datetime 精度，MySQL 5.6 之前的数据库不支持
      DontSupportRenameIndex:    true,  // 重命名索引时采用删除并新建的方式，MySQL 5.7 之前的数据库和 MariaDB 不支持重命名索引
      DontSupportRenameColumn:   true,  // 用 `change` 重命名列，MySQL 8 之前的数据库和 MariaDB 不支持重命名列
      SkipInitializeWithVersion: false, // 根据版本自动配置
   }), &gorm.Config{NamingStrategy: schema.NamingStrategy{
      SingularTable: mysqlConfig.IsPlural, //允许表名复数  `User` 的表名应该是 `t_users` 如果该值为true则表名为`t_user`
      //TablePrefix:   mysqlConfig.TablePrefix, // 表名前缀
   }})
   sqlDB, err := DB.DB()
   //设置空闲连接池中连接的最大数量
   sqlDB.SetMaxIdleConns(mysqlConfig.MaxIdLeConn)
   //设置打开数据库连接的最大数量
   sqlDB.SetMaxOpenConns(mysqlConfig.MaxOpenConn)
   //设置了连接可复用的最大时间
   sqlDB.SetConnMaxLifetime(time.Hour)
   //defer sqlDB.Close()
   return err
}
```

end

### json渲染？

#### `gin.H{}`

`gin.H`主要是为了开发者很方便的构建出一个`map`对象，不止用于`c.JSON`方法，也可以用于其他场景。

```go
func main()  {

	r := gin.Default()
	r.GET("/name/:name", func(context *gin.Context) {
        // H is a shortcut for map[string]interface{}
		// type H map[string]interface{}
		context.JSON(http.StatusOK, gin.H{"data": context.Param("name") })
	})
	r.Run()
}
// output
{
    "name": "Robin"
}
```

end

#### `context.Json()`

`Json()` serializes the given struct as JSON into the response body.
It also sets the Content-Type as "application/json".

```go
type Book struct {
	ID     string `json:"id"`
	Title  string `json:"title"`
	Author string `json:"author"`
}

var books = []Book{
	{ID: "1", Title: "Harry Potter", Author: "J. K. Rowling"},
	{ID: "2", Title: "The Lord of the Rings", Author: "J. R. R. Tolkien"},
	{ID: "3", Title: "The Wizard of Oz", Author: "L. Frank Baum"},
}

func main() {
	r := gin.New()

    // JSON serializes the given struct as JSON into the response body.
	// It also sets the Content-Type as "application/json".
	//func (c *Context) JSON(code int, obj any) {
		//c.Render(code, render.JSON{Data: obj})
	//}
	r.GET("/books", func(c *gin.Context) {
		c.JSON(http.StatusOK, books)
	})
	r.Run()
}
// output
// 格式没有缩进
[{"id":"1","title":"Harry Potter","author":"J. K. Rowling"},{"id":"2","title":"The Lord of the Rings","author":"J. R. R. Tolkien"},{"id":"3","title":"The Wizard of Oz","author":"L. Frank Baum"}]
```

end

#### IndentedJSON 美化

`Json()`输出的JSON字符串都是扁平的，没有缩进，不美观。对于这种情况,Gin提供了便捷的`c.IndentedJSON()`方法，让输出的JSON更好看。

```go
r.GET("/books", func(c *gin.Context) {
		c.IndentedJSON(http.StatusOK, books)
})
// output
[
    {
        "id": "1",
        "title": "Harry Potter",
        "author": "J. K. Rowling"
    },
    {
        "id": "2",
        "title": "The Lord of the Rings",
        "author": "J. R. R. Tolkien"
    },
    {
        "id": "3",
        "title": "The Wizard of Oz",
        "author": "L. Frank Baum"
    }
]
```

#### PureJSON

对于JSON字符串中特殊的字符串，比如`<`，Gin默认是转义的，比如变成`\ u003c`，但是有时候我们为了可读性，需要保持原来的字符，不进行转义，这时候我们就可以使用`PureJSON`

```go
	r.GET("/json", func(c *gin.Context) {
		c.JSON(200, gin.H{
			"message": "<b>Hello, world!</b>",
		})
	})

	r.GET("/purejson", func(c *gin.Context) {
		c.PureJSON(200, gin.H{
			"message": "<b>Hello, world!</b>",
		})
	})
```

用这两个API进行对比，我们运行访问`http://localhost:8080/json`，显示信息如下：

```javascript
{"message":"\u003cb\u003eHello, world!\u003c/b\u003e"}
```

特殊字符已经被转义了，现在访问`http://localhost:8080/pureJson`，显示的就是原始信息：

```javascript
{"message":"<b>Hello, world!</b>"}
```

可读性更强。

#### AsciiJSON

如果要把非`Ascii`字符串转为unicode编码，Gin同样提供了非常方便的方法。

```go
r.GET("/asciiJSON", func(c *gin.Context) {
		c.AsciiJSON(200, gin.H{"message": "hello 王"})
	})
```

通过`c.AsciiJSON`方法，Gin可以把所有的非`Ascii`字符全部转义为`unicode`编码，现在我们运行看看结果。

```javascript
{"message":"hello \u98de\u96ea\u65e0\u60c5"}
```

#### json加速

在Gin中，提供了两种JSON解析器，用于生成JSON字符串。默认的是Golang(Go语言)内置的JSON，当然你也可以使用jsoniter，据说速度很快。如果要使用jsoniter，我们在`go build`编译的时候只需要这么做即可：

```javascript
go build -tags=jsoniter .
```



#### wrap?

如何wrap下json返回内容

```go
func main() {
	r := gin.Default()

	// gin.H 是map[string]interface{}的缩写
	r.GET("/someJSON", func(c *gin.Context) {
		// 方式一：自己拼接JSON
		c.JSON(http.StatusOK, gin.H{"message": "Hello world!"})
	})
	r.GET("/moreJSON", func(c *gin.Context) {
		// 方法二：使用结构体
		var msg struct {
			Name    string `json:"user"`
			Message string
			Age     int
		}
		msg.Name = "小王子"
		msg.Message = "Hello world!"
		msg.Age = 18
		c.JSON(http.StatusOK, msg)
	})
	r.Run(":8080")
}
```



### log

#### write log to file

```go
func main() {
    // Disable Console Color, you don't need console color when writing the logs to file.
    gin.DisableConsoleColor()

    // Logging to a file.
    f, _ := os.Create("gin.log")
    gin.DefaultWriter = io.MultiWriter(f)

    // Use the following code if you need to write the logs to file and console at the same time.
    // gin.DefaultWriter = io.MultiWriter(f, os.Stdout)

    r := gin.Default()
    r.GET("/ping", func(c *gin.Context) {
        c.String(200, "pong")
    })

    r.Run(":8080")
}
```



#### zap log rotate

zap作为log middleware，使用gin engine加载使用即可。

```go
// Use attaches a global middleware to the router. i.e. the middleware attached through Use() will be
// included in the handlers chain for every single request. Even 404, 405, static files...
// For example, this is the right place for a logger or error management middleware.
func (engine *Engine) Use(middleware ...HandlerFunc) IRoutes {
	engine.RouterGroup.Use(middleware...)
	engine.rebuild404Handlers()
	engine.rebuild405Handlers()
	return engine
}

// demo
r := gin.Default()
// 注册zap相关中间件
r.Use(logger.GinLogger(), logger.GinRecovery(true))

```

完整代码实例

```go
package logger

import (
	"gin_zap_demo/config"
	"net"
	"net/http"
	"net/http/httputil"
	"os"
	"runtime/debug"
	"strings"
	"time"

	"github.com/gin-gonic/gin"
	"github.com/natefinch/lumberjack"
	"go.uber.org/zap"
	"go.uber.org/zap/zapcore"
)

var lg *zap.Logger

// InitLogger 初始化Logger
func InitLogger(cfg *config.LogConfig) (err error) {
	writeSyncer := getLogWriter(cfg.Filename, cfg.MaxSize, cfg.MaxBackups, cfg.MaxAge)
	encoder := getEncoder()
	var l = new(zapcore.Level)
	err = l.UnmarshalText([]byte(cfg.Level))
	if err != nil {
		return
	}
	core := zapcore.NewCore(encoder, writeSyncer, l)

	lg = zap.New(core, zap.AddCaller())
	zap.ReplaceGlobals(lg) // 替换zap包中全局的logger实例，后续在其他包中只需使用zap.L()调用即可
	return
}

func getEncoder() zapcore.Encoder {
	encoderConfig := zap.NewProductionEncoderConfig()
	encoderConfig.EncodeTime = zapcore.ISO8601TimeEncoder
	encoderConfig.TimeKey = "time"
	encoderConfig.EncodeLevel = zapcore.CapitalLevelEncoder
	encoderConfig.EncodeDuration = zapcore.SecondsDurationEncoder
	encoderConfig.EncodeCaller = zapcore.ShortCallerEncoder
	return zapcore.NewJSONEncoder(encoderConfig)
}

func getLogWriter(filename string, maxSize, maxBackup, maxAge int) zapcore.WriteSyncer {
	lumberJackLogger := &lumberjack.Logger{
		Filename:   filename,
		MaxSize:    maxSize,
		MaxBackups: maxBackup,
		MaxAge:     maxAge,
	}
	return zapcore.AddSync(lumberJackLogger)
}

// GinLogger 接收gin框架默认的日志
func GinLogger() gin.HandlerFunc {
	return func(c *gin.Context) {
		start := time.Now()
		path := c.Request.URL.Path
		query := c.Request.URL.RawQuery
		c.Next()

		cost := time.Since(start)
		lg.Info(path,
			zap.Int("status", c.Writer.Status()),
			zap.String("method", c.Request.Method),
			zap.String("path", path),
			zap.String("query", query),
			zap.String("ip", c.ClientIP()),
			zap.String("user-agent", c.Request.UserAgent()),
			zap.String("errors", c.Errors.ByType(gin.ErrorTypePrivate).String()),
			zap.Duration("cost", cost),
		)
	}
}

// GinRecovery recover掉项目可能出现的panic，并使用zap记录相关日志
func GinRecovery(stack bool) gin.HandlerFunc {
	return func(c *gin.Context) {
		defer func() {
			if err := recover(); err != nil {
				// Check for a broken connection, as it is not really a
				// condition that warrants a panic stack trace.
				var brokenPipe bool
				if ne, ok := err.(*net.OpError); ok {
					if se, ok := ne.Err.(*os.SyscallError); ok {
						if strings.Contains(strings.ToLower(se.Error()), "broken pipe") || strings.Contains(strings.ToLower(se.Error()), "connection reset by peer") {
							brokenPipe = true
						}
					}
				}

				httpRequest, _ := httputil.DumpRequest(c.Request, false)
				if brokenPipe {
					lg.Error(c.Request.URL.Path,
						zap.Any("error", err),
						zap.String("request", string(httpRequest)),
					)
					// If the connection is dead, we can't write a status to it.
					c.Error(err.(error)) // nolint: errcheck
					c.Abort()
					return
				}

				if stack {
					lg.Error("[Recovery from panic]",
						zap.Any("error", err),
						zap.String("request", string(httpRequest)),
						zap.String("stack", string(debug.Stack())),
					)
				} else {
					lg.Error("[Recovery from panic]",
						zap.Any("error", err),
						zap.String("request", string(httpRequest)),
					)
				}
				c.AbortWithStatus(http.StatusInternalServerError)
			}
		}()
		c.Next()
	}
}

```

然后定义日志相关配置：

```go
package config

import (
	"encoding/json"
	"io/ioutil"
)

// Config 整个项目的配置
type Config struct {
	Mode       string `json:"mode"`
	Port       int    `json:"port"`
	*LogConfig `json:"log"`
}

// LogConfig 日志配置
type LogConfig struct {
	Level      string `json:"level"`
	Filename   string `json:"filename"`
	MaxSize    int    `json:"maxsize"`
	MaxAge     int    `json:"max_age"`
	MaxBackups int    `json:"max_backups"`
}

// Conf 全局配置变量
var Conf = new(Config)

// Init 初始化配置；从指定文件加载配置文件
func Init(filePath string) error {
	b, err := ioutil.ReadFile(filePath)
	if err != nil {
		return err
	}
	return json.Unmarshal(b, Conf)
}
```

在项目中先从配置文件加载配置信息，再调用`logger.InitLogger(config.Conf.LogConfig)`即可完成logger实例的初识化。其中，通过`r.Use(logger.GinLogger(), logger.GinRecovery(true))`注册我们的中间件来使用zap接收gin框架自身的日志，在项目中需要的地方通过使用`zap.L().Xxx()`方法来记录自定义日志信息。

```go
package main

import (
	"fmt"
	"gin_zap_demo/config"
	"gin_zap_demo/logger"
	"net/http"
	"os"

	"go.uber.org/zap"

	"github.com/gin-gonic/gin"
)

func main() {
	// load config from config.json
	if len(os.Args) < 1 {
		return
	}

	if err := config.Init(os.Args[1]); err != nil {
		panic(err)
	}
	// init logger
	if err := logger.InitLogger(config.Conf.LogConfig); err != nil {
		fmt.Printf("init logger failed, err:%v\n", err)
		return
	}

	gin.SetMode(config.Conf.Mode)

	r := gin.Default()
	// 注册zap相关中间件
	r.Use(logger.GinLogger(), logger.GinRecovery(true))

	r.GET("/hello", func(c *gin.Context) {
		// 假设你有一些数据需要记录到日志中
		var (
			name = "q1mi"
			age  = 18
		)
		// 记录日志并使用zap.Xxx(key, val)记录相关字段
		zap.L().Debug("this is hello func", zap.String("user", name), zap.Int("age", age))

		c.String(http.StatusOK, "hello liwenzhou.com!")
	})

	addr := fmt.Sprintf(":%v", config.Conf.Port)
	r.Run(addr)
}

```

end

### tracing

### JWT

JWT就是一种基于Token的轻量级认证模式，服务端认证通过后，会生成一个JSON对象，经过签名后得到一个Token（令牌）再发回给用户，用户后续请求只需要带上这个Token，服务端解密之后就能获取该用户的相关信息了。

##### why not cookie-session？

通常使用的是Cookie-Session的认证方式。大致的流程如下：

1. 用户使用用户名个密码登录，发送信息到服务器。
2. 服务器拿到用户的信息，后会生成一份保存用户信息的session数据和与之对应的cookie id，然后保存session在在服务器，把对应的session id返回给用户并保存在浏览器。
3. 后续来自该用户的每次请求都会携带相应的cookie。
4. 服务端通过带来的cookie找到之前保存的与之对应的用户信息。

那么这种方式有什么弊端呢？

- 不支持跨域
- 通过数据库查询响应的用户信息耗时
- CSRF攻击(跨站伪造请求)

#### 生成token

```go
type MyClaims struct {
   Username string `json:"username"`
   jwt.StandardClaims
}

// 定义过期时间
const TokenExpireDuration = time.Hour * 2

//定义secret
var MySecret = []byte("这是一段用于生成token的密钥")

//生成jwt
func GenToken(username string) (string, error) {
   c := MyClaims{
      username,
      jwt.StandardClaims{
         ExpiresAt: time.Now().Add(TokenExpireDuration).Unix(),
         Issuer:    "my-project",
      },
   }
   //使用指定的签名方法创建签名对象
   token := jwt.NewWithClaims(jwt.SigningMethodHS256, c)

   //使用指定的secret签名并获得完成的编码后的字符串token
   return token.SignedString(MySecret)
}

```



#### 解析and认证token

函数签名

```go
func ParseWithClaims(tokenString string, claims Claims, keyFunc Keyfunc) (*Token, error) {
	return new(Parser).ParseWithClaims(tokenString, claims, keyFunc)
}

// Parse methods use this callback function to supply
// the key for verification.  The function receives the parsed,
// but unverified Token.  This allows you to use properties in the
// Header of the token (such as `kid`) to identify which key to use.
type Keyfunc func(*Token) (interface{}, error)

```

demo

```go

//解析JWT
func ParseToken(tokenString string) (*MyClaims, error) {
   //解析token
   token, err := jwt.ParseWithClaims(tokenString, &MyClaims{}, func(token *jwt.Token) (i interface{}, err error) {
      return MySecret, nil
   })
   if err != nil {
      return nil, err
   }
   // token, err := jwt.ParseWithClaims(tokenString, claims, func(token *jwt.Token) (interface{}, error) {
 //   return []byte("<YOUR VERIFICATION KEY>"), nil
//})
   if claims, ok := token.Claims.(*MyClaims); ok && token.Valid {
      return claims, nil
   }
   return nil, errors.New("invalid token")
}
```

end

#### gin and jwt

##### 生成jwt

两个接口一个生成jwt并创建cookies保存，一个验证jwt

```go
func authHandler(c *gin.Context) {
	// 用户发送用户名和密码过来
	var user UserInfo
	err := c.ShouldBind(&user)
	if err != nil {
		c.JSON(http.StatusOK, gin.H{
			"code": 2001,
			"msg":  "无效的参数",
		})
		return
	}
	// 校验用户名和密码是否正确
	if user.Username == "test" && user.Password == "foo" {
		// 生成Token
		tokenString, _ := GenToken(user.Username)
		c.JSON(http.StatusOK, gin.H{
			"code": 2000,
			"msg":  "success",
			"data": gin.H{"token": tokenString},
		})
		return
	}
	c.JSON(http.StatusOK, gin.H{
		"code": 2002,
		"msg":  "鉴权失败",
	})
	return
}
```

end

##### 注入jwt中间件：前端配合？？？:heavy_exclamation_mark:

是不是少了一个中间件，每次请求的时候将token注入到client请求？？？

B/S 架构client怎么加入jwt

**客户端收到服务器返回的 JWT，可以储存在 Cookie 里面，也可以储存在 localStorage。**

> 前端检测有这个token自动带上去？？？
>
> 前端存的超时时间应该和token超时时间一样

此后，客户端每次与服务器通信，都要带上这个 JWT。你可以把它放在 Cookie 里面自动发送，但是这样不能跨域，所以更好的做法是放在 HTTP 请求的头信息`Authorization`字段里面。

> ```javascript
> Authorization: Bearer <token>
> ```

另一种做法是，跨域的时候，JWT 就放在 POST 请求的数据体里面。

通常客户端传输 JWT 是通过 header 中的 **Authorization** 字段，好处是避免了 [CORS](https://developer.mozilla.org/en-US/docs/Glossary/CORS)攻击。并且使用 **Bearer** 模式。



##### 验证中间件

用户通过上面的接口获取Token之后，后续就会携带着Token再来请求我们的其他接口，这个时候就需要对这些请求的Token进行校验操作了，很显然我们应该实现一个检验Token的中间件，具体实现如下：

```go
// JWTAuthMiddleware 基于JWT的认证中间件
func JWTAuthMiddleware() func(c *gin.Context) {
	return func(c *gin.Context) {
		// 客户端携带Token有三种方式 1.放在请求头 2.放在请求体 3.放在URI
		// 这里假设Token放在Header的Authorization中，并使用Bearer开头
		// 这里的具体实现方式要依据你的实际业务情况决定
		authHeader := c.Request.Header.Get("Authorization")
		if authHeader == "" {
			c.JSON(http.StatusOK, gin.H{
				"code": 2003,
				"msg":  "请求头中auth为空",
			})
			c.Abort()
			return
		}
		// 按空格分割
		parts := strings.SplitN(authHeader, " ", 2)
		if !(len(parts) == 2 && parts[0] == "Bearer") {
			c.JSON(http.StatusOK, gin.H{
				"code": 2004,
				"msg":  "请求头中auth格式有误",
			})
			c.Abort()
			return
		}
		// parts[1]是获取到的tokenString，我们使用之前定义好的解析JWT的函数来解析它
		mc, err := ParseToken(parts[1])
		if err != nil {
			c.JSON(http.StatusOK, gin.H{
				"code": 2005,
				"msg":  "无效的Token",
			})
			c.Abort()
			return
		}
		// 将当前请求的username信息保存到请求的上下文c上
		c.Set("username", mc.Username)
		c.Next() // 后续的处理函数可以用过c.Get("username")来获取当前请求的用户信息
	}
}
```

##### 验证接口

```go
r.GET("/home", JWTAuthMiddleware(), homeHandler)

func homeHandler(c *gin.Context) {
	username := c.MustGet("username").(string)
	c.JSON(http.StatusOK, gin.H{
		"code": 2000,
		"msg":  "success",
		"data": gin.H{"username": username},
	})
}
```





### goroutine pool

 C++/Java 实现线程池时，通常可能为了解决创建取消线程开销过大的问题，同时也为不同优先级的请求提供不同的调度模式。

但是 Golang 已经实现了 M:N 的用户态线程 Goroutine，还要必要在 Golang 里面使用线程池吗？

`fasthttp`是有一个[workerpool](https://link.zhihu.com/?target=https%3A//github.com/valyala/fasthttp/blob/master/workerpool.go)的，其实就是`goroutine`池。每建立一个`tcp`连接，就会用一个`goroutine`处理它，`tcp`连接断开后，`goroutine`不会立即释放。有新的`tcp`连接时，塞给空闲的`goroutine`进行处理。每隔10秒，清理掉空闲的`goroutine`，这时候，这些`goroutine`才会被移入空闲g队列中。

所以，当服务一直处于活跃状态时，使用`goroutine`池可以大大**延迟空闲g队列调度**，而不是每个连接结束后立马释放`goroutine`。

目前`workerpool`的上限是256*1024，也就是同时能维持这么多`tcp`连接。

fasthttp 作者的观点

> *Initially fasthttp server was creating a goroutine per each incoming connection as net/http.Server do. Spawning a goroutine per connection is fast - multi-million goroutines per second may be created on an average hardware. I just optimized it further with workerpool.go.*
> *- valyala(fasthttp作者)*


fasthttp 当前维护者的观点

> *I have done a lot of testing and having a goroutine pool is still slightly faster than just starting a new goroutine every time you need one.*
> *Seeing as our code is currently stable with the pool I don't think we should remove it at this point. But maybe we should in the future to simplify our code.*
> \- Erik

**用池是可以提升运行效率，降低内存使用的。所以内存吃紧的可以用池优化。**

### 引用

1. https://www.zhihu.com/question/20355738/answer/111087933
2. https://www.zhihu.com/question/406955904/answer/1359178859
3. https://blog.dianduidian.com/post/gin-%E4%B8%AD%E9%97%B4%E4%BB%B6next%E6%96%B9%E6%B3%95%E5%8E%9F%E7%90%86%E8%A7%A3%E6%9E%90/
4. https://juejin.cn/post/7120039561538830367
5. https://medium.com/@wattanai.tha/go-tutorial-series-ep-1-building-rest-api-with-gin-7c17c7ab1d5b
6. https://cloud.tencent.com/developer/article/1579400
7. https://juejin.cn/post/7093035836689612836
8. https://www.liwenzhou.com/posts/Go/jwt_in_gin/
