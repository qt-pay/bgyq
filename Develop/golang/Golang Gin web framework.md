## Golang gin-gonic web framework



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



### Restful API

https://go.dev/doc/tutorial/web-service-gin

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



*Some other database products draw a distinction. For example, in the Oracle Database product, a schema represents only a part of a database: the tables and other objects owned by a single user.*

#### go-gorm demo

https://betterprogramming.pub/building-a-rest-api-with-go-gin-framework-and-gorm-38cb2d6353da







### 引用

1. https://www.zhihu.com/question/20355738/answer/111087933
