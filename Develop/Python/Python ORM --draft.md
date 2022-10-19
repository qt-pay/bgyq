## Python ORM --draft

### pymysql: SQL

This package contains a pure-Python MySQL client library.

```py
#!/usr/bin/env python
# -*- coding:utf-8 -*-
import pymysql
  
# 创建连接
conn = pymysql.connect(host='127.0.0.1', port=3306, user='root', passwd='123', db='t1')
# 创建游标
cursor = conn.cursor()
  
# 执行SQL，并返回收影响行数
effect_row = cursor.execute("update hosts set host = '1.1.1.2'") 
  
# 提交，不然无法保存新建或者修改的数据
conn.commit()
  
# 关闭游标
cursor.close()
# 关闭连接
conn.close()
```





### SQLAlchemy: ORM

SQLAlchemy is the Python SQL toolkit and Object Relational Mapper that gives application developers the full power and flexibility of SQL.

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/SQLAlchemy-ORM.png)

```python
from sqlalchemy import Column, Integer, String
from sqlalchemy import create_engine
engine = create_engine('sqlite:///sales.db', echo = True)
from sqlalchemy.ext.declarative import declarative_base
Base = declarative_base()

class Customers(Base):
   __tablename__ = 'customers'
   
   id = Column(Integer, primary_key=True)
   name = Column(String)
   address = Column(String)
   email = Column(String)
   
from sqlalchemy.orm import sessionmaker
Session = sessionmaker(bind = engine)
session = Session()

c1 = Customers(name = 'Ravi Kumar', address = 'Station Road Nanded', email = 'ravi@gmail.com')

session.add(c1)
session.commit()
```

end

#### ORM or SQL?

SQLAlchemy can serve two purposes: making SQL interactions easier, and serving as a full-blown ORM. Even if you aren't interested in implementing an ORM (or don't know what an ORM is), SQLAlchemy is invaluable as a tool to simplify connections to SQL databases and performing raw queries. I thought that was an original statement as I wrote it, but it turns out SQLAlchemy describes itself this way:

> SQLAlchemy consists of two distinct components, known as the **Core** and the **ORM**.  The Core is itself a fully featured SQL abstraction toolkit, providing a smooth layer of abstraction over a wide variety of DBAPI implementations and behaviors, as well as a SQL Expression Language which allows expression of the SQL language via generative Python expressions.

ORM把数据库映射成对象

- 数据库的表（table） --> 类（class）
- 记录（record，行数据）--> 对象（object）
- 字段（field）--> 对象的属性（attribute）



### DAO

data access object，数据访问对象。

#### 定义与应用

The purpose of the DAO is to hide the implementation details of the data access mechanism. 

DAO即Data-Access-Object，直译过来就是数据访问对象。很多人对DAO的理解是负责链接数据库，实际上并不是。它是一个接口，一个DAL的实现，可能连接数据库，也可能连接到Redis。包括Wikipedia在内，我没找到更多讨论DAO职责的内容，很多人使用它都是基于一些约定俗成的规则，包括以下:

1. 一个Model 对应一个DAO。比如Account –> AccountDAO; Book –> BookDAO
2. DAO不限制identifier方式的参数。比如你可以在get方法上定义String参数。与之相对，Repository侧重于”领域(Domain)”

一个标准的DAO如下:

```java
public interface AccountDAO {
    Account get(String userName);
    void create(Account account);
    void update(Account account);
    void delete(String userName);
}
```

有一点开发经验的朋友可能知道，随着开发不断推进，可能AccountDAO会变成下面这样

```java
public interface AccountDAO {
 
    Account get(String userName);
    void create(Account account);
    void update(Account account);
    void delete(String userName);
 
    List getAccountByLastName(String lastName);
    List getAccountByAgeRange(int minAge, int maxAge);
    void updateEmailAddress(String userName, String newEmailAddress);
    void updateFullName(String userName, String firstName, String lastName);
 
}
```

再过一段时间，可能这个类会变的更复杂。我们DAO开始变得臃肿，对于调用者而言变得复杂，当一个DAO出现几十个query时，调用时可能会一头雾水。



#### DAO and ORM

DAO 屏蔽了数据获取的逻辑，只提供一个 getUser(id) 的方法，不管里面怎么实现，需要数据用这个方法取就行。

然后看 DAO 内部，可以用很多方法去实现。

最简单的方法，直接查库，query('select * from users where id = ' + id)，然后返回数据，需要什么数据就直接写 SQL。
ORM 就是把常用的 SQL 封装好，这样用里面提供的类似 getUserById(id) 的方法就能省掉上面的字符串拼接、参数处理、校验、返回数据格式化等问题，直接返回映射好的 object 。



DAO 层可以通过 ORM 实现增删改查，也可以不通过。

至于说需要封装什么方法，这就要看你的程序业务需求而定了。
最基础的当然是增删改查。

关键在于多一层封装，就把数据控制逻辑和业务逻辑解耦了。
以后更换数据控制方法，比如从 ORM 变成 SQL，或者从 SQL 变成 ORM，甚至是更换数据库类型等，都不会对上层业务造成影响。

orm 的定义基本就是 Model 里边对数据库的一个映射。

dao 就是一个事务里边的的组合体。

上边再来一个 service，基本就成型了。web 的话，还有一个 handler 的层。

#### demo：go-java- DAO

 https://github.com/dushixiang/next-terminal
开源项目，分了 service 层和 repository 层，repository 主要是用来操作某一个表的，例如根据 ID 查询可能会很多地方用到，repository 可以写一个 FindById 函数，参数是 ID，返回是数据库表的结构体，这样就不必每次都去拿全局 DB.Where("id = ?", id).First(&User{}) 查询了，也更方便检索有哪些地方用了这个函数。

service 层主要也是写一个会复用到的函数，不复用的直接就用 repository 操作了。

### repository 

How is the Repository pattern different? As far as I can tell, it isn’t. Saying a Repository is *different* to a DAO because you’re dealing with/return a collection of objects can’t be right; DAOs can also return collections of objects.

Everything I’ve read about the repository pattern seems rely on this distinction: bad DAO design vs good DAO design (aka repository design pattern).

Ref: https://stackoverflow.com/a/11384997

Repository这个概念出自”DDD”(Domain-Driven-Design)，Repository Pattern提议所有操作最好是基于领域(Domain)进行。同时，他的行为像Java里面的Collection。当然，这是一种抽象的概念。如Collection中存在add、update、remove，contains等函数。同样的，你应该把这些定义到Repository中。

```java
import java.util.List;
import com.thinkinginobjects.domainobject.Account;
 
public interface AccountRepository {
 
    void addAccount(Account account);
    void removeAccount(Account account);
    void updateAccount(Account account); // Think it as replace for set
 
    List query(AccountSpecification specification);

}
```

我们可以看出AccountRepository中removeAccount并不像DAO那样传进一个userName，而是传入一个Account对象。如果理解不了这点，想一下你在java里面是怎么操作删除List的即可。

上面DAO中提到可能随着业务复杂度增加，DAO会出现大量的查询接口。我们可以注意到AccountRepository多了个List query(AccountSpecification specification);

```java
import com.thinkinginobjects.domainobject.Account;
 
public interface AccountSpecification {
    boolean specified(Account account);
}
public class AccountSpecificationByUserName implements AccountSpecification, HibernateSpecification {
 
    private String desiredUserName;
 
    public AccountSpecificationByUserName(String desiredUserName) {
        super();
        this.desiredUserName = desiredUserName;
    }
 
    @Override
    public boolean specified(Account account) {
        return account.hasUseName(desiredUserName);
    }
 
    @Override
    public Criterion toCriteria() {
        return Restrictions.eq("userName", desiredUserName);
    }
 
}
```

这是DDD里面的解决方案，使用Specification Pattern来解决上述问题。需要注意的是，这不是Respository特有的东西，你完全可以用到DAO上。而且个人对是否应该使用这个模式持保留态度。

#### Repository and Dao

从上面的讨论你可以看出，Repository Pattern更加面向对象，提出了更多的规约对以往的DAO Pattern做了一些补充，不过实现起来更加复杂。事实上，没有人阻止你把Repository那些东西应用在DAO上。所以不必要过于纠结这两者，你可以把二者结合一起用，也可以分开用，具体怎么用，就是看自己的业务场景

### 自定义ORM

ORM怎么处理复杂sql语句

有的人工作内容就只是 CRUD，用 ORM 就足够了，根本接触不到复杂的系统，甚至认为系统复杂是设计的问题，而不是业务本身就复杂。

低并发场景，例如订单，用户属性，用 ORM，高并发场景，如优惠券，活动用 SQL，但凡需要复杂的多表联合查询的，都用业务逻辑解决，数据库就是存数据的，不处理业务逻辑。

#### 将多表联合查转换成业务逻辑



### 引用

1. https://www.cnblogs.com/aylin/p/5770888.html
2. https://www.tutorialspoint.com/sqlalchemy/sqlalchemy_orm_adding_objects.htm
3. https://www.v2ex.com/t/775162
4. https://wiyi.org/dao-vs-repository.html