## MVC

### Python

#### WSGI服务

wsgiref模块其实就是将整个请求信息给封装了起来，就不需要你自己处理了，假如它将所有请求信息封装成了一个叫做request的对象，那么你直接request.path就能获取到用户这次请求的路径，request.method就能获取到本次用户请求的请求方式(get还是post)等，那这个模块用起来，我们再写web框架是不是就简单了好多啊。
对于真实开发中的python web程序来说，一般会分为两部分：**服务器程序和应用程序**。

* 服务器程序负责对socket服务器进行封装，并在请求到来时，对请求的各种数据进行整理。
* 应用程序则负责具体的逻辑处理。为了方便应用程序的开发，就出现了众多的Web框架，例如：Django、Flask、web.py 等。不同的框架有不同的开发方式，但是无论如何，开发出的应用程序都要和服务器程序配合，才能为用户提供服务。

这样，服务器程序就需要为不同的框架提供不同的支持。这样混乱的局面无论对于服务器还是框架，都是不好的。对服务器来说，需要支持各种不同框架，对框架来说，只有支持它的服务器才能被开发出的应用使用。最简单的Web应用就是先把HTML用文件保存好，用一个现成的HTTP服务器软件，接收用户请求，从文件中读取HTML，返回。如果要动态生成HTML，就需要把上述步骤自己来实现。不过，接受HTTP请求、解析HTTP请求、发送HTTP响应都是苦力活，如果我们自己来写这些底层代码，还没开始写动态HTML呢，就得花个把月去读HTTP规范。

正确的做法是底层代码由专门的服务器软件实现，我们用Python专注于生成HTML文档。因为我们不希望接触到TCP连接、HTTP原始请求和响应格式，所以，需要一个统一的接口协议来实现这样的服务器软件，让我们专心用Python编写Web业务。

这时候，标准化就变得尤为重要。我们可以设立一个标准，只要服务器程序支持这个标准，框架也支持这个标准，那么他们就可以配合使用。一旦标准确定，双方各自实现。这样，服务器可以支持更多支持标准的框架，框架也可以使用更多支持标准的服务器。

WSGI（Web Server Gateway Interface）就是一种规范，它定义了使用Python编写的web应用程序与web服务器程序之间的接口格式，实现web应用程序与web服务器程序间的解耦。

常用的WSGI服务器有uwsgi、Gunicorn。而Python标准库提供的独立WSGI服务器叫wsgiref，Django开发环境用的就是这个模块来做服务器。

#### MVC

Web服务器开发领域里著名的MVC模式，所谓MVC就是把Web应用分为模型(M)，控制器(C)和视图(V)三层，他们之间以一种插件式的、松耦合的方式连接在一起，模型负责业务对象与数据库的映射(ORM)，视图负责与用户的交互(页面)，控制器接受用户的输入调用模型和视图完成用户的请求。

model负责与数据库交互，view是核心，负责接收请求，获取数据，返回结果，template负责呈现内容到浏览器。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/mvc.png)

MVC has three layers:

Model:

It communicates with the database.

View:

It displays the data on the screen.

Controller:

It is an interconnection between the model and the view layer.

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/mvc-design.png)

#### Django：MTV

Django的MTV模式本质上和MVC是一样的，也是为了各组件间保持松耦合关系，只是定义上有些许不同，Django的MTV分别是值：

* M 代表模型（Model）： 负责业务对象和数据库的关系映射(ORM)。
* T 代表模板 (Template)：负责如何把页面展示给用户(html)。
* V 代表视图（View）： 负责业务逻辑，并在适当时候调用Model和Template。

除了以上三层之外，还需要一个URL分发器，它的作用是将一个个URL的页面请求分发给不同的View处理，View再调用相应的Model和Template，MTV的响应模式如下所示：

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/django-mtv.png)

#### flask：no ORM

It's not an MVC (for one, there's no model in flask -- however, you can combine it with something like **SQLAlchemy**), and I'm not sure what you mean by MTV. It's basically a wrapper around werkzeug which is a wrapper around pure WSGI. A wrapper with templating abilities.

Flask is actually not an MVC framework. It is a minimalistic framework which gives you a lot of freedom in how you structure your application, but MVC pattern is a very good fit for what Flask provides, at least in the way that MVC pattern is understood today in the context of web applications (which purists would probably object to).

##### model：sqlalchemy

Models represent the data and its related logic. They are the core of the application. It will decide the data that is being transferred between the controller and other business logic. For example, in our case, the controller is responsible for creating a table(Inserttable) in the MySQL database.

In the figure above you can see the illustration that only, the model has access to the database. Models access the database directly and that data is being used by the controller and ultimately for the view to display.

```python
from flask_sqlalchemy import SQLAlchemy

db = SQLAlchemy()

# Creating the Inserttable for inserting data into the database


class Inserttable(db.Model):
    '''Data for ON/OFF should be dumped in this table.'''

    __tablename__ = 'inserttable'
    id = db.Column(db.Integer, primary_key=True, autoincrement=True)
    machineid = db.Column(db.Integer, primary_key=False)
    stateid = db.Column(db.Integer, primary_key=False)
    speed = db.Column(db.Integer, nullable=False)
    statechange = db.Column(db.Integer, nullable=False)
    unixtime = db.Column(db.Integer, nullable=False)
    extras = db.Column(db.String(80), nullable=False)
    state = db.Column(db.String(80), nullable=False)

    # method used to represent a class's objects as a string
    def __repr__(self):
        return '<machineid %r>' % self.machineid
        
```

##### controller

The controller is like a middle-man between view and models. They take input and talk directly to the model that will communicate with the database. Here you are only calling the business logic from the services layer.

```python
import json
from models.machine import Inserttable, db
from services.user_service import insert_logic, create_logic

def index():
    return {'status': 'OK',
            'localhost:5000/machines/create': 'Create table in mysql database',
            'localhost:5000/machines/insert': 'Insert data in mysql database table(Inserttable)'}


def create():
    
    create_logic()


# insert data into table.
def insert():
    
    insert_logic()    
```

Note: controller doesn’t communicate directly with the database there is a model between the database and controller.

In the above code index, create and insert method talk to the model. The controller tells the model what to do. The model then communicates with the database and fetches data then comes the view part.

##### services layer (business logic):

You can call it an additional layer on top of the controller. It is helpful when your application grows and more features are added to it, it is a better practice to separate the business logic from the application. A service layer adds an additional layer of abstraction between the application and the business logic. It has the core business logic.

> Business logic part
>
> Here you can see that I’m reading data from the data.json and using the StateId int value’s binary I decide what to set the state column to in the MYSQL database table(inserttable). You can simply add some data in fields in the table instead of this business logic.

You can think of services as a worker. While the controller is the manager that just handles what to do, services are the workers that do the actual work and return what is required by the API user. In summary:

- controller(manager): manages the work.
- services(worker): do the work.

```python
import json
from models.machine import Inserttable, db

def create_logic():
    try:
        # create tables if not exists.
        db.create_all()
        db.session.commit()
        return '==================TABLES CREATED=================='

    except Exception as e:
        print(e)
        return '==================TABLES NOT CREATED!!!=================='


def insert_logic():
    data = json.load(open("data.json", 'r'))  # reading file data.json
    for i, b in enumerate('{0:016b}'.format(data['StateId'])):
        if int(b) == 1:
            example = Inserttable(machineid=data["MachineId"], stateid=data["StateId"],
                                  speed=data["Speed"], statechange=data["StateChange"],
                                  unixtime=data["UnixTime"], extras=data["Extras"],
                                  state="ON")

            db.session.add(example)
            # db.session.commit()

        else:
            example = Inserttable(machineid=data["MachineId"], stateid=data["StateId"],
                                  speed=data["Speed"], statechange=data["StateChange"],
                                  unixtime=data["UnixTime"], extras=data["Extras"],
                                  state="OFF")

            db.session.add(example)
            # db.session.commit()
    return '==================DATA INSERTED=================='
    db.session.commit()
    # db.session.close()
```



##### view

Now the view is the part of the application that is responsible for displaying the data. It can be a simple return string or a fully-fledged HTML page with a beautiful design. I’m not implementing the view part in this tutorial because my major focus is the backend functioning of the app.



##### 目录结构规划

```bash
# 每个目录下都有一个application.py
$ tree Project_Name
|_ api (dir)
  _ application.py
|_ model(dir)
  _ application.py
|_ handler(dir)
  _ application.py
|_ service(dir)
  _ application.py

```

作用分析：

* api   application.py： http 最外层接口GET OR POST
* mode application.py：定义数据结构
* handler application.py： application自身的业务逻辑
* service application.py： 和application相关的业务逻辑，比如：查询一个用户下的全部application等

### 引用

1. https://www.cnblogs.com/DengSchoo/p/12506925.html
2. https://medium.com/@nxtbyt/minimal-flask-application-using-mvc-design-pattern-842845cef703