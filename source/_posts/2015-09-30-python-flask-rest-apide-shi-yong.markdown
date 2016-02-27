---
layout: post
title: "python Flask REST API的使用"
date: 2015-09-30 15:43:21 +0800
comments: true
categories: python
tags: [REST API, python]
styles: [data-table]
---


> “简单不先于复杂，而是在复杂之后” —— Alan Perlis



本文参考：Designing a RESTful API using Flask-RESTful
Posted by Miguel Grinberg under Python, Programming, REST, Flask.

<br>
#### **使用Flask的扩展构建REST API**
<br>

Flask-RESTful扩展简化了构建REST API服务器的过程，在开始之前，我们先来看一下本文中会使用到的几个api：

HTTP Method | URI | Action
-------- | ----
GET | http://[hostname]/todo/api/v1.0/tasks | Retrieve list of tasks
GET | http://[hostname]/todo/api/v1.0/tasks/[task_id] | Retrieve a task
POST | http://[hostname]/todo/api/v1.0/tasks | Create a new task
PUT | http://[hostname]/todo/api/v1.0/tasks/[task_id] | Update an existing task
DELETE | http://[hostname]/todo/api/v1.0/tasks/[task_id] | Delete a task

<br>
本文的web service提供的服务使用的资源是task，它包含如下字段：

* uri: unique URI for the task. String type.
* title: short task description. String type.
* description: long task description. Text type.
* done: task completion state. Boolean type.
<br>

<br>
#### **路由**
<br>

Flask-RESTful提供`Resource`基类用来定义一个给定的URL的不同的HTTP请求的响应方法，比如，在我们的例子中，可以继承`Resource`基类定义一个`TaskListAPI`类来实现`Tasks`资源的`GET`，`POST`等方法，另外可以继承`Resource`基类定义一个`TaskAPI`类来实现`Task`资源的`GET`，`PUT`，`DELETE`等方法：

```
from flask import Flask
from flask.ext.restful import Api, Resource

class TaskListAPI(Resource):

    def get(self):
		pass

class TaskAPI(Resource):

    def get(self, id):
		pass

    def put(self, id):
		pass

    def delete(self, id):
		pass


api.add_resource(TaskListAPI, '/todo/api/v1.0/tasks', endpoint = 'tasks')
api.add_resource(TaskAPI, '/todo/api/v1.0/tasks/<int:id>', endpoint = 'task')


```

`api.add_resource`方法使用指定的`endpoint`注册路由到相应的`资源API类`上，对应HTTP请求的动作会指定使用相应`资源API类`下面的相应方法。

<br>
#### **验证请求及约束**
<br>

当请求的数据不符合要求时，需要拒绝。Flask-RESTful 提供了一个方式来处理数据验证，它叫做`RequestParser`类。

```
from flask.ext.restful import reqparse

class TaskListAPI(Resource):
    def __init__(self):
        self.reqparse = reqparse.RequestParser()
        self.reqparse.add_argument('title', type = str, required = True,
            help = 'No task title provided', location = 'json')
        self.reqparse.add_argument('description', type = str, default = "", location = 'json')
        super(TaskListAPI, self).__init__()

    # ...

class TaskAPI(Resource):
    def __init__(self):
        self.reqparse = reqparse.RequestParser()
        self.reqparse.add_argument('title', type = str, location = 'json')
        self.reqparse.add_argument('description', type = str, location = 'json')
        self.reqparse.add_argument('done', type = bool, location = 'json')
        super(TaskAPI, self).__init__()

    # ...

```

先获得`reqparse.RequestParser`对象，对每一个参数进行约束验证。比如，对于创建一个`Task`来说，`title`字段是必须的，且必须是json格式的。

<br>
#### **生成响应**
<br>

Flask-RESTFul会自动将return的内容作为响应response发出:

```
return { 'task': make_public_task(task) }

```

另外，还可以加入自定义状态码：

```
return { 'task': make_public_task(task) }, 201

```

<br>
#### **生成外部可定位格式**
<br>

每一个请求的资源都有一个固定的URI，通常会将这个URI绑定在相应的资源中：

```
{
    "tasks": [
        {
            "description": "Need to find a good Python tutorial on the web",
            "done": false,
            "title": "Learn Python",
            "uri": "/todo/api/v1.0/tasks/2"
        },
        {
            "description": "Hello,world",
            "done": false,
            "title": "Hello,Urey",
            "uri": "/todo/api/v1.0/tasks/3"
        }
    ]
}

```

但是，我们创建资源的时候是使用的`id`标明，所以需要将内部的`id`转换成外部可定位的`URI`，Flask-RESTful提供了一个辅助函数能够很优雅地做到这样的转换，不仅仅能够把`id`转成`URI`并且能够转换其他的参数:

```
from flask.ext.restful import fields, marshal

task_fields = {
    'title': fields.String,
    'description': fields.String,
    'done': fields.Boolean,
    'uri': fields.Url('task')
}

class TaskAPI(Resource):
    # ...

    def put(self, id):
        # ...
        return { 'task': marshal(task, task_fields) }
        
class TaskListAPI(Resource):
    # ...

    def post(self, id):
        # ...
        return {'task': marshal(task, task_fields)}, 201

```

其中，`task_fields`结构用于作为`marshal`函数的模板。fields.Uri 是一个用于生成一个`URL`的特定的参数。它需要的参数是`endpoint`。这样，当`PUT`更新一个资源或者`POST`上传一个资源时，会自动进行相应的转换（在本例中，将内部资源转换为外部资源）。

<br>
#### **安全认证**
<br>

Flask-RESTful提供了一个扩展`Flask-HTTPAuth`进行认证。

```
from flask.ext.httpauth import HTTPBasicAuth
# ...
auth = HTTPBasicAuth()
# ...

class TaskAPI(Resource):
    decorators = [auth.login_required]
    # ...

class TaskAPI(Resource):
    decorators = [auth.login_required]
    # ...

```

使用`auth`获得一个`HTTPBasicAuth`类型的对象，之后，在需要认证的`资源API`中加入一个装饰器，这样，每一个方法都必须在认证的前提下进行访问。

最简单的认证方式如下，提供相应的账户和密码。

```

@auth.get_password
def get_password(username):
    if username == 'miguel':
        return 'python'
    return None

```

如果认证失败的处理方法：

```
@auth.error_handler
def unauthorized():
    # return 403 instead of 401 to prevent browsers from displaying the default
    # auth dialog
    return make_response(jsonify({'message': 'Unauthorized access'}), 401)
    
```

<br>
**全部代码：**
<br>

```
#!flask/bin/python

"""Alternative version of the ToDo RESTful server implemented using the
Flask-RESTful extension."""

from flask import Flask, jsonify, abort, make_response
from flask.ext.restful import Api, Resource, reqparse, fields, marshal
from flask.ext.httpauth import HTTPBasicAuth

app = Flask(__name__, static_url_path="")
api = Api(app)
auth = HTTPBasicAuth()


@auth.get_password
def get_password(username):
    if username == 'miguel':
        return 'python'
    return None


@auth.error_handler
def unauthorized():
    # return 403 instead of 401 to prevent browsers from displaying the default
    # auth dialog
    return make_response(jsonify({'message': 'Unauthorized access'}), 401)

tasks = [
    {
        'id': 1,
        'title': u'Buy groceries',
        'description': u'Milk, Cheese, Pizza, Fruit, Tylenol',
        'done': False
    },
    {
        'id': 2,
        'title': u'Learn Python',
        'description': u'Need to find a good Python tutorial on the web',
        'done': False
    }
]

task_fields = {
    'title': fields.String,
    'description': fields.String,
    'done': fields.Boolean,
    'uri': fields.Url('task')
}


class TaskListAPI(Resource):
    decorators = [auth.login_required]

    def __init__(self):
        self.reqparse = reqparse.RequestParser()
        self.reqparse.add_argument('title', type=str, required=True,
                                   help='No task title provided',
                                   location='json')
        self.reqparse.add_argument('description', type=str, default="",
                                   location='json')
        super(TaskListAPI, self).__init__()

    def get(self):
        return {'tasks': [marshal(task, task_fields) for task in tasks]}

    def post(self):
        args = self.reqparse.parse_args()
        task = {
            'id': tasks[-1]['id'] + 1,
            'title': args['title'],
            'description': args['description'],
            'done': False
        }
        tasks.append(task)
        return {'task': marshal(task, task_fields)}, 201


class TaskAPI(Resource):
    decorators = [auth.login_required]

    def __init__(self):
        self.reqparse = reqparse.RequestParser()
        self.reqparse.add_argument('title', type=str, location='json')
        self.reqparse.add_argument('description', type=str, location='json')
        self.reqparse.add_argument('done', type=bool, location='json')
        super(TaskAPI, self).__init__()

    def get(self, id):
        task = [task for task in tasks if task['id'] == id]
        if len(task) == 0:
            abort(404)
        return {'task': marshal(task[0], task_fields)}

    def put(self, id):
        task = [task for task in tasks if task['id'] == id]
        if len(task) == 0:
            abort(404)
        task = task[0]
        args = self.reqparse.parse_args()
        for k, v in args.items():
            if v is not None:
                task[k] = v
        return {'task': marshal(task, task_fields)}

    def delete(self, id):
        task = [task for task in tasks if task['id'] == id]
        if len(task) == 0:
            abort(404)
        tasks.remove(task[0])
        return {'result': True}


api.add_resource(TaskListAPI, '/todo/api/v1.0/tasks', endpoint='tasks')
api.add_resource(TaskAPI, '/todo/api/v1.0/tasks/<int:id>', endpoint='task')


if __name__ == '__main__':
    app.run(debug=True)


```












