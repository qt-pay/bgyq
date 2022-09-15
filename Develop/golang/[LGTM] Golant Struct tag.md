## [LGTM] Golant Struct tag

## 结构体标签

### 标签定义

Tag是结构体在编译阶段关联到成员的元信息字符串，在运行的时候通过反射的机制读取出来。

结构体标签由一个或多个键值对组成。键与值使用冒号分隔，值用双引号括起来。键值对之间使用一个空格分隔，具体的格式如下：

```golang
`key1:"value1" key2:"value2" key3:"value3"...`  // 键值对用空格分隔
```

key会指定反射的解析方式，如下： json(JSON标签) orm(Beego标签)、gorm(GORM标签)、bson(MongoDB标签)、form(表单标签)、binding(表单验证标签)

### 标签选项

```golang
type Student struct {
    ID   int     `json:"-"`            // 该字段不进行序列化
    Name string  `json:name,omitempy`  // 如果为类型零值或空值，序列化时忽略该字段
    Age  int     `json:age,string`     // 指定类型，支持string、number、boolen
}
```

注：encoding/json[官方文档](https://link.juejin.cn?target=https%3A%2F%2Fstudygolang.com%2Fstatic%2Fpkgdoc%2Fpkg%2Fencoding_json.htm)

## json标签

### JSON说明

JSON`数组`可以用于编码Go语言的`数组`和`slice`；由于JSON`对象`是一个字符串到值的映射，写成一系列的`name:value`对形式，因此JSON的`对象`类型可以用于编码Go语言的`map`和`结构体`。

将Go语言中结构体`slice`转为JSON的过程叫`编组`(marshaling)，编组通过`json.Marshal`函数完成。在编码时，默认使用Go语言结构体的成员名字作为JSON的对象（通过reflect反射技术）。只有导出的结构体成员才会被编码。

如果在结构体slice编码成JSON的时候使用自定义的成员名，可以使用`结构体成员Tag`来实现。

### JSON标签

```golang
type User struct {
    ID   int `json:"id"`  // 编码后的字段名为 id
    Name string           // 编码后的字段名为 自定义成员名 Name
    age  int              // 未导出字段不能编码
}
```

`json`为键名的标签对应的值用于控制`encoding/json`包的编码和解码的行为，并且`encoding/...`下面其它的包也遵循这个约定。

| 标签选项 | 使用说明                                                     |
| -------- | ------------------------------------------------------------ |
| -        | 字段不进行序列化 例：`json:"-"`                              |
| omitempy | 类型零值或空值，序列化时忽略该字段 例：`json:",omitempy"` 字段名省略的话用结构体字段名 |
| type     | 重新指定字段类型 例：`json:"age,string"`                     |

## gorm标签

### 模型定义

模型是标准的 struct,由基本数据类型以及实现了 [Scanner](https://link.juejin.cn?target=https%3A%2F%2Fpkg.go.dev%2Fdatabase%2Fsql%2F%3Ftab%3Ddoc%23Scanner) 和 [Valuer](https://link.juejin.cn?target=https%3A%2F%2Fpkg.go.dev%2Fdatabase%2Fsql%2Fdriver%23Valuer) 接口的自定义类型及其指针或别名组成。

GORM 定义一个 gorm.Model 结构体，如下所示：

```golang
type Model struct {
  ID        uint           `gorm:"primaryKey"`
  CreatedAt time.Time
  UpdatedAt time.Time
  DeletedAt gorm.DeletedAt `gorm:"index"`
}

```

gorm为键名的标签遵循GORM的解析规则，GORM支持如下tag，tag名大小写不敏感，建议使用`camelCase`风格，多个标签定义用分号(;)分隔

`[知识点]` Gorm建表时 AUTO_INCREMENT 不生效的问题

```golang
// AUTO_INCREMENT 不生效
Id  uint64 `gorm:"column:id;primaryKey;type:bigint(20);autoIncrement;comment:'主键'"`
// AUTO_INCREMENT 不生效
Id  uint64 `gorm:"column:id;type:bigint(20);autoIncrement;comment:'主键'"`
// AUTO_INCREMENT 生效 gorm会自动根据字段类型设置数据库字段类型并设置为主键
Id  uint64 `gorm:"column:id;autoIncrement;comment:'主键'"` //写成AUTO_INCREMENT也可以
```

| 标签选项               | 使用说明                                                     |
| ---------------------- | ------------------------------------------------------------ |
| column                 | 指定 db 列名                                                 |
| type                   | 列数据类型，推荐使用兼容性好的通用类型，例如：所有数据库都支持 bool、int、uint、float、string、time、bytes 并且可以和其他标签一起使用，例如：`not null`、`size`, `autoIncrement`… 像 `varbinary(8)` 这样指定数据库数据类型也是支持的。在使用指定数据库数据类型时，它需要是完整的数据库数据类型，如：`MEDIUMINT UNSIGNED not NULL AUTO_INCREMENT` |
| size                   | 指定列大小，例如：`size:256`                                 |
| primaryKey             | 指定列为主键                                                 |
| unique                 | 指定列为唯一                                                 |
| default                | 指定列的默认值，字符串默认值用单引号，例：`gorm:"default:'cn'"` |
| precision              | 指定列的精度                                                 |
| scale                  | 指定列大小                                                   |
| not null               | 指定列为 NOT NULL                                            |
| autoIncrement          | 指定列为自动增长，不可与`primaryKey`、`type`同时使用否则不生效，看上面`知识点`部分 |
| autoIncrementIncrement | 自动步长，控制连续记录之间的间隔                             |
| embedded               | 嵌套字段                                                     |
| embeddedPrefix         | 嵌入字段的列名前缀                                           |
| autoCreateTime         | 创建时追踪当前时间，对于 `int` 字段，它会追踪秒级时间戳，您可以使用 `nano`/`milli` 来追踪纳秒、毫秒时间戳，例如：`autoCreateTime:nano` |
| autoUpdateTime         | 创建/更新时追踪当前时间，对于 `int` 字段，它会追踪秒级时间戳，您可以使用 `nano`/`milli` 来追踪纳秒、毫秒时间戳，例如：`autoUpdateTime:milli` |
| index                  | 根据参数创建索引，多个字段使用相同的名称则创建复合索引，查看 [索引](https://link.juejin.cn?target=https%3A%2F%2Fgorm.io%2Fzh_CN%2Fdocs%2Findexes.html) 获取详情 |
| uniqueIndex            | 与 `index` 相同，但创建的是唯一索引                          |
| check                  | 创建检查约束，例如 `check:age > 13`，查看 [约束](https://link.juejin.cn?target=https%3A%2F%2Fgorm.io%2Fzh_CN%2Fdocs%2Fconstraints.html) 获取详情 |
| <-                     | 设置字段写入的权限， `<-:create` 只创建、`<-:update` 只更新、`<-:false` 无写入权限、`<-` 创建和更新权限 |
| ->                     | 设置字段读的权限，`->:false` 无读权限                        |
| -                      | 忽略该字段，`-` 无读写权限                                   |
| comment                | 迁移时为字段添加注释                                         |

示例：

```golang
// 内容模型
type Content struct {
    Model
    NewsId   uint64  `gorm:"column:news_id"`
    Content  string  `gorm:"column:content"`
}
```

`知识点` 自定义唯一索引

```golang
// Go 版本：go1.16.6   Gorm 版本：v1.9.16
// 尝试用 uniqueIndex 创建不生效,有解决方法的同学欢迎评论区留言 普通索引是生效的
Email string `gorm:"column:email;type:varchar(50);uniqueIndex:uidx_email"` // 不生效
Email string `gorm:"column:email;type:varchar(50);index:idx_email"`        // 生效

// 创建唯一索引 建表时会创建名称为 email 的唯一索引
Email string `gorm:"column:email;type:varchar(50);unique"`
// 创建自定义名称 uidx_email 的唯一索引
Email string `gorm:"column:email;type:varchar(50)" sql:"unique_index:uidx_email"`
```

`知识点` 自动更新时间

`GORM`约定使用`CreatedAt`、`UpdatedAt`追踪创建/更新时间。如果定义了这种字段，且默认值为零值，`GORM`在创建、更新时会自动填充当前时间。要使用不同名称的字段，您可以配置`autoCreateTime`、`autoUpdateTime`标签，如果想要保存 UNIX（毫/纳）秒时间戳，而不是 time，只需简单地将 time.Time 修改为 int 即可，毫/纳秒参数可以看上面表格示例。

```golang
// 时间自动创建和更新
type User struct { 
    // 自定义字段  使用时间戳填纳秒数充更新时间 
    Updated   int64 `gorm:"autoUpdateTime:nano"` 
    //自定义字段  使用时间戳毫秒数填充更新时间 
    Updated   int64 `gorm:"autoUpdateTime:milli"` 
    //自定义字段  使用时间戳秒数填充创建时间 
    Created   int64 `gorm:"autoCreateTime"` 
    // 默认创建时间字段  在创建时如果该字段值为零值，则使用当前时间填充 
    CreatedAt time.Time 
    // 默认更新时间字段  在创建时该字段值为零值或者在更新时，使用当前时间戳秒数填充 
    UpdatedAt int      
}
```

注：GORM模型 [官方文档](https://link.juejin.cn?target=https%3A%2F%2Fgorm.io%2Fzh_CN%2Fdocs%2Fmodels.html)

### 关联标签

GORM的关联类型有多重类型：`belongs to`、`has one`、`has many`、`many to many`具体结构体定义可参考问[文档](https://link.juejin.cn?target=https%3A%2F%2Fgorm.io%2Fzh_CN%2Fdocs%2Fbelongs_to.html)，关联模式使用的标签选项如下所示：

| 标签选项         | 使用说明                                                     |
| ---------------- | ------------------------------------------------------------ |
| foreignKey       | 指定当前模型的列作为连接表的外键 例：`gorm:"foreignKey:FieldId"` 其中FieldID是外键字段名 |
| references       | 指定引用表的列名，其将被映射为连接表外键                     |
| polymorphic      | 指定多态类型，比如模型名                                     |
| polymorphicValue | 指定多态值、默认表名                                         |
| many2many        | 指定连接表表名                                               |
| joinForeignKey   | 指定连接表的外键列名，其将被映射到当前表                     |
| joinReferences   | 指定连接表的外键列名，其将被映射到引用表                     |
| constraint       | 关系约束，例如：`OnUpdate`、`OnDelete`                       |

示例：

```golang
// 新闻模型
type News struct {
    Model
    Title   string   `gorm:"column:title;type:string;not null,default:''"`
    Content Content  `gorm:"foreignKey:NewsId" json:"content"` //指定外键
}
```

注：GORM关联模型 [官方文档](https://link.juejin.cn?target=https%3A%2F%2Fgorm.io%2Fzh_CN%2Fdocs%2Fassociations.html%23tags)

## form标签

Gin中提供了模型绑定，将表单数据和模型进行绑定，方便参数校验和使用。

### 模型绑定

```golang
// 表单数据
type LoginForm struct {
    Email     string    `form:"emial"`    
    Password  string    `form:"password"`
}
// model 或 service 层Model
type Email struct {
    Email       string
    Password    string
}
```

通过 form:"email" 对表单email数据进行绑定。然后通过Bind()、ShouldBind()等方法获取参数值。

```golang
func EmailLogin (c *gin.Context) {
    var email LoginForm
    if err := c.ShouldBind(&email); err != nil {
        ...
    }
    // 获取表单数据局
    args := Email {
        Email:     email.Email,
        Password:  email.Password,
    }
    // 对参数进行后续使用
    ...
}
```

## binding标签

Gin对于数据的校验使用的是 `validator.v10` [包](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2Fgo-playground%2Fvalidator)，该包提供多种数据校验方法，通过`binding:""`标签来进行数据校验。

我们对上面的表单模型添加数据校验标签如下：

```golang
type LoginForm struct {
    Email     string    `form:"emial" binding:"email"`    
    Password  string    `form:"password" binging:"required,min=6,max=10"`
}
```

特殊符号说明：

- 逗号（,）:分隔多个标签选项，逗号之间不能有空格，否则panic；
- 横线（-）:跳过该字段不做校验；
- 竖线（|）:使用多个选项，满足其中一个即可。

binding标签选项：

### 必需校验

| 标签选项  | 使用说明                             | 示例                        |
| --------- | ------------------------------------ | --------------------------- |
| required  | 表示该字段值必输设置，且不能为默认值 | `binding:required`          |
| omitempty | 如果字段未设置，则忽略它             | `binding:reqomitemptyuired` |

### 范围校验

范围验证: 切片、数组和map、字符串，验证其长度；数值，验证大小范围

| 标签选项 | 使用说明                                                     | 示例                        |
| -------- | ------------------------------------------------------------ | --------------------------- |
| len      | 参数值等于给定值                                             | `binding:"len=8"`等于8      |
| ne       | 不等于                                                       | `binding:"ne=8"`不等于8     |
| max      | 最大值，小于等于参数值                                       | `binding:"max=8"`小于等于8  |
| min      | 最小值，大于等于参数值                                       | `binding:"min=8"`大于等于8  |
| lte      | 参数值小于等于给定值                                         | `binding:"lte=8"`小于等于8  |
| gte      | 参数值大于等于给定值                                         | `binding:"gte=8"`大于等于8  |
| lt       | 参数值小于给定值                                             | `binding:"lt=8"`小于8       |
| gt       | 参数值大于给定值                                             | `binding:"gt=8"`大于8       |
| oneof    | 参数值只能是枚举值中的一个，值必须是数值或字符串，以空格分隔，如果字符串中有空格，将字符串用单引号包围 | `binding:"oneof=red green"` |

示例：

```golang
type User struct {
    Name string `form:"name" binding:"required,min=1,max=10"`
    Age  unit8  `form:"age" binding:"lte=150,gte=0"`
    sex  string `form:"sex" binding:"oneof=male female"`
}
```

注：[文档地址](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2Fgo-playground%2Fvalidator%2Fblob%2Fmaster%2FREADME.md%23comparisons)

### 字符串校验

| 标签选项   | 使用说明             | 示例                                        |
| ---------- | -------------------- | ------------------------------------------- |
| contains   | 参数值包含设置子串   | `binding:"contains=tom"`是否包含tom字符串   |
| excludes   | 参数值不包含设置子串 | `binding:"excludes=tom"`是否不包含tom字符串 |
| startswith | 字符串前缀           | `binding:"startswith=tom"`是否以tom开头     |
| endswith   | 字符串前缀           | `binding:"endswith=tom"`是否以tom结尾       |

示例：

```golang
type User struct {
    Name string `form:"name" binding:"required,contains=ac,endswith=ck"`
}
复制代码
```

注：[文档地址](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2Fgo-playground%2Fvalidator%2Fblob%2Fmaster%2FREADME.md%23strings)

### 字段校验

| 标签选项  | 使用说明                                                     |
| --------- | ------------------------------------------------------------ |
| eqcsfield | 跨不同结构体字段相等，比如`struct1 field1` 是否等于`struct2 field2` |
| necsfield | 跨不同结构体字段不相等                                       |
| eqfield   | 同一结构体字段相等验证，例如：输入两次密码                   |
| nefield   | 同一结构体字段不相等验证                                     |
| gtefield  | 大于等于同一结构体字段                                       |
| ltefield  | 小于等于同一结构体字段                                       |

示例：

```golang
// 跨不同结构体字段相等
type Struct1 struct { 
    Field1 string `validate:eqcsfield=Struct2.Field2` 
    Struct2 struct { 
        Field2 string 
    } 
}
// 同一结构体字段相等验证
type Email struct { 
    Email  string `validate:"lte=4"` 
    Pwd    string `validate:"min=10"` 
    Pwd2   string `validate:"eqfield=Pwd"`
}
// 同一结构体字段验证不相等
type User struct { 
    Name     string `validate:"lte=4"` 
    Age      int `validate:"min=20"` 
    Password string `validate:"min=10,nefield=Name"`
}
复制代码
```

### 其他校验

| 标签选项 | 使用说明                                     | 示例                            |
| -------- | -------------------------------------------- | ------------------------------- |
| ip       | 合法IP地址校验                               | `binding:"ip"`                  |
| email    | 合法邮箱校验                                 | `binding:"email"`               |
| url      | 合法的URL                                    | `binding:"url"`                 |
| uri      | 合法的URI                                    | `binding:"uri"`                 |
| uuid     | uuid验证                                     | `binding:"uuid"`                |
| datetime | 合法时间格式值校验                           | `binding:"datetime=2006-01-02"` |
| json     | JSON数据验证                                 | `validate:"json"`               |
| numeric  | 数值验证 正则：`^[-+]?[0-9]+(?:\\.[0-9]+)?$` | `validate:"numeric"`            |
| number   | 整数验证 正则：`^[0-9]+$`                    | `validate:"number"`             |
| alpha    | 字母字符串验证 正则：`^[a-zA-Z]+$`           | `validate:"alpha"`              |
| alphanum | 字母数字字符串验证 正则：`^[a-zA-Z0-9]+$`    | `validate:"alphanum"`           |
| ascii    | Ascii 字符验证                               | `validate:"ascii"`              |

示例：

```golang
type User struct { 
    Name     string  `validate:"required,min=1,max=10"` 
    Email    int     `validate:"required,email"`
    birthday string  `validate:"datetime=2006-01-02"`
    Pwd      string  `validate:"required,alphanum"`
    Score    srring  `validate:"numeric"`
}
复制代码
```

注：[文档地址](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2Fgo-playground%2Fvalidator%2Fblob%2Fmaster%2FREADME.md%23network)

## ini标签

在使用`go-ini`库操作.ini配置文件的时候，如果需要将配置文件字段映射到结构体变量，如果键名与字段名不相同，那么需要在结构标签中指定对应的键名。标准库`encoding/json`、`encoding/xml`解析时可以将键名`app_name`对应到字段名`AppName`，而go-ini库不可以，所以需要在结构体标签指定对应键名。

```ini
## 配置文件 cnf.ini
app_name  = awesome web
log_level = DEBUG
// 配置文件映射 结构体
type Config struct {
  AppName   string `ini:"app_name"`  // ini标签指定下键名
  LogLevel  string `ini:"log_level"`
}

```

### 原文

作者：紬绎青蛙
链接：https://juejin.cn/post/7005465902804123679
来源：稀土掘金
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。