---
title: "Flask Restful"
date: 2018-12-13T13:54:30+08:00
showDate: true
draft: false
tags: ["Python","flask"]
categories: ["python"]
---

# Flask-restful和Flask-sqlalchemy

## RESTful API

关于rest,我的理解就是一种设计风格,这种风格的特点就是资源就是一切,所有的操作都是面向资源的,服务端通过URI来知道你要获取的资源,通过HTTP方法来知道你要对资源进行什么操作,通过返回的状态码来告诉你操作后的状态(成功,失败等等...)

## flask-restful

### 一个最小的 API
关于flask-restful,就是为了快速构建restful api所开发的一个拓展.先看它官方文档的第一个例子(有改动)

```py
from flask import Flask
from flask_restful import Resource, Api

app = Flask(__name__)  # 实例化一个flask app对象
api = Api(app)  # 实例化restful api对象


class HelloWorld(Resource):   # 创建一个抽象资源类
    def get(self):  # 定义这个资源类的get方法(对应到http的GET)
        return {'hello': 'world'}  # 返回值


api.add_resource(HelloWorld, '/')  # 把资源类添加到api对象里,通过'127.0.0.1:5000/'来访问

if __name__ == '__main__':
    app.run(debug=True)
```
大概的流程就是通过flask app来创建api对象,然后api把资源类添加到api对象,然后加上访问资源的路径.就可以通过路由来访问的.

```bash
# lzj @ lzj in ~ [14:16:42] 
$ curl http://127.0.0.1:5000/
{
    "hello": "world"
}
```
###完整的例子

```python
from flask import Flask
from flask_restful import reqparse, abort, Api, Resource

app = Flask(__name__)
api = Api(app)

TODOS = {
    'todo1': {'task': 'build an API'},
    'todo2': {'task': '?????'},
    'todo3': {'task': 'profit!'},
}   # 资源,这里可以看成是数据库


def abort_if_todo_doesnt_exist(todo_id):  # 一个判断函数
    if todo_id not in TODOS:
        abort(404, message="Todo {} doesn't exist".format(todo_id))


parser = reqparse.RequestParser()   # 定义一个参数解析器
parser.add_argument('task', type=str)  # 添加参数到参数解析器


# Todo
#   show a single todo item and lets you delete them
class Todo(Resource):  # todo资源类
    def get(self, todo_id):  # GET方法
        abort_if_todo_doesnt_exist(todo_id)
        return TODOS[todo_id]

    def delete(self, todo_id):  # DELETE方法
        abort_if_todo_doesnt_exist(todo_id)
        del TODOS[todo_id]
        return {"message": "delete ok"}

    def put(self, todo_id):   # PUT方法
        args = parser.parse_args()  # 解析参数
        print(args)
        task = {'task': args['task']}
        TODOS[todo_id] = task
        return task, 201


# TodoList
#   shows a list of all todos, and lets you POST to add new tasks
class TodoList(Resource):  #todolist资源类
    def get(self):   # GET方法
        return TODOS

    def post(self):  # POST方法
        args = parser.parse_args()
        todo_id = int(max(TODOS.keys()).lstrip('todo')) + 1
        todo_id = 'todo%i' % todo_id
        TODOS[todo_id] = {'task': args['task']}
        return TODOS[todo_id], 201

##
## Actually setup the Api resource routing here
##
# 给资源分配路由
api.add_resource(TodoList, '/todos')
api.add_resource(Todo, '/todos/<todo_id>')


if __name__ == '__main__':
    app.run(debug=True)
```

在这个完整的例子里面,我们定义了多个资源类,每个资源有自己的http方法,如果通过资源类没有定义的方法请求资源路径,flask会自动帮我们返回`405 METHOD NOT ALLOWED`

### 关于参数解析

flask-restful内置支持了验证请求数据

```python
from flask_restful import reqparse

parser = reqparse.RequestParser()  # 参数解析器
parser.add_argument(name='rate', type=int, help='Rate to charge for this resource', location='json')  # 添加参数
args = parser.parse_args()  # 参数解析
```

#### 关于解析器的add_argument方法的一些参数:

|参数名|作用|
|---|---|
|name|表示的是要添加的参数的名字|
|type|参数的数据类型|
|help|就是当参数解析的时候缺少了这个参数flask返回的响应body的内容|
|location|参数出现的位置|

更多的例子可以参考[链接](http://www.pythondoc.com/Flask-RESTful/)

## ORM

根据wiki百科的介绍orm就是
`对象关系映射（英语：Object Relational Mapping，简称ORM，或O/RM，或O/R mapping），是一种程序设计技术，用于实现面向对象编程语言里不同类型系统的数据之间的转换。从效果上说，它其实是创建了一个可在编程语言里使用的“虚拟对象数据库”。是一种为了解决面向对象与关系数据库存在的互不匹配的现象的技术。简单的说，ORM是通过使用描述对象和数据库之间映射的元数据，将程序中的对象自动持久化到关系数据库中。`

## sqlalchemy

### 定义数据库模型

sqlalchemy是python主流的orm框架之一,它通过定义一些模型类,来映射到数据库中的表:

```python
# 导入:
from sqlalchemy import Column, String, create_engine
from sqlalchemy.orm import sessionmaker
from sqlalchemy.ext.declarative import declarative_base

# 创建对象的基类:
Base = declarative_base()

# 定义User对象:
class User(Base):
    # 表的名字:
    __tablename__ = 'user'

    # 表的结构:
    id = Column(String(20), primary_key=True)  # String对应于数据库的varchar
    name = Column(String(20))
```

这个User类可以直接映射到数据库中的user表(通过tablename设置)

### 数据库CRUD

由于有了ORM，我们向数据库表中添加一行记录，可以视为添加一个User对象：

```python
# 初始化数据库连接:
engine = create_engine('mysql+mysqlconnector://root:password@localhost:3306/test')
# 创建DBSession类型:
DBSession = sessionmaker(bind=engine)

# 创建session对象:
session = DBSession()

# 创建新User对象:
new_user = User(id='5', name='Bob')

# 添加到session:
session.add(new_user)

# 查找User
u = session.query(User).filter_by(id='5')

print(u.name) # >>> Bob

# 更新

u.name = "Jhon"

print(u.name) # >>> Jhon

# 删除

session.delete(u)

# 提交即保存到数据库:
session.commit()
# 关闭session:
session.close()
```

## flask-sqlalchemy

flask-sqlalchemy是sqlalchemy在flask的拓展,通过flask-sqlalchemy我们可以很方便的在flask中使用sqlalchemy,

### 一个简单的应用

```python
from flask import Flask
from flask.ext.sqlalchemy import SQLAlchemy

app = Flask(__name__)  # 实例化一个app对象
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:////tmp/test.db'  # 配置sqlalchemy的uri
db = SQLAlchemy(app)  # 通过以app为参数实例化一个db


class User(db.Model):  # 定义User模型
    id = db.Column(db.Integer, primary_key=True)  # id字段
    username = db.Column(db.String(80), unique=True)  # username字段
    email = db.Column(db.String(120), unique=True)  # email字段

    def __repr__(self):
        return '<User %r>' % self.username

if __name__ == '__main__':
    db.create_all()  # 创建通过模型所有表
```

关于flask-sqlalchemy的使用可以参考[链接](http://www.pythondoc.com/flask-sqlalchemy/quickstart.html)

## flask架构设计

### 一个小的flask项目

一个小的完整flask app可以是下面这种目录.

```zsh
├── app.py
├── config.py
├── manage.py
├── model
│   ├── __init__.py
│   ├── user.py
├── myauth.py
├── readme.md
├── req.txt
├── resource
│   ├── __init__.py
│   ├── user_api.py
├── test
│   ├── __init__.py
│   ├── test.py
└── util
    ├── __init__.py
    └── util.py
```

`config.py`是配置文件,`model`文件夹里面定义数据库模型,`resource`定义资源对象,`test`里面是关于代码的测试,`util.py`定义一些工具方法(像获取七牛资源这种),`manage.py`就是项目入口,里面定义资源的路由.

### 说说云家园的架构

云家园的项目里有个app文件夹,里面的每一个子文件夹都是一个模块,每个模块的结构都类似于上面那个完整的项目.

然后每个模块创建flask的blueprint然后在全局app里面注册

```python
def create_app(config_name):
    app = Flask(__name__)  # 

    from .main import main as main_blueprint
    app.register_blueprint(main_blueprint, url_prefix='/api/main')

    from .user import user as user_blueprint
    app.register_blueprint(user_blueprint, url_prefix='/api/user')

    from .admin import admin as admin_blueprint
    app.register_blueprint(admin_blueprint, url_prefix='/api/admin')
```
create_app方法的部分代码