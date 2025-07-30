---
title: Tornado 实现 decorator 路由
date: 2016-12-24 06:07:40
categories:
- 技术
tags:
- python
- web 开发
- tornado
---

我也实现了一个 Flask decorator 风格的 Tornado 路由，实现的方式极大地参考了 Flask 的过程(add_url_rule 和 Blueprint)。   
Tornado 在新建 Application 的时候， 需要传入一个 handlers 参数， 原本这个 handlers 需要自己手动构建:  收集各个 RequestHandler，给他们分配路径，组成一个 handlers tuple。 这样维护起来非常麻烦。  
我实现的功能是：  
1. 通过 decorator 给每个 RequestHander 即时分配 url pattern；  
2. 支持根据 API 版本和 Resource 类型自动给 url pattern 添加前缀；  
3. 可以通过 RequestHander 的类名反向查出 url。  
功能一点都不复杂， 实现起来也简单， 不到 100 行代码。下面是我的实现过程。
<!-- more -->
## 一、理念
后端提供的 API 应该组织起来， 比如 api\_v1, api\_v2, api\_test, api\_internal。这里可以抽象出一个 API Class。  
每一套 API 其实是一系列资源的集合， 比如 user, video, post。 这里又可以抽象出一个 Resource Class， 通过 API Class 进行管理。  
每一个资源其实就对应到 Tornado 的 RequestHandler 了。  
所以伪代码应该是：
```python
apiv1 = API(name='api_v1', prefix='/api/v1')  # 实例化一个 API Class
user = apiv1.create_resource('user', prefix='/user')  # 通过 API 对象管理 resource
@user.route('/profile/')  # 通过 resource 对象管理 handler， 完整路径是 /api/v1/user/profile
class UserProfile(BaseHandler):
    @authenticated
    def get(self):
        return self.write('user profile')
apiv1.register_resources()  # 注册所有 resource 到 API 对象下
app = Application(
    handlers=apiv1.handlers  # 所有 handler 可以通过 api.handlers 获取
)
```
## 二、API Class
```python
class API:
    def __init__(self, name, prefix):
        self.name = name
        if not prefix.endswith('/'):
            prefix += '/'
        self.prefix = prefix
        self.handlers = []
        self.resources = []

    def route(self, path, kwargs=None, name=None):
        if path.startswith('/'):
            path = path[1:]
        path = '{0}{1}'.format(self.prefix, path)
        def handler_wrapper(handler):
            self.handlers.append((path, handler, kwargs, name))
            def handler_initializer(*args, **kwargs):
                return handler(*args, **kwargs)
            return handler_initializer
        return handler_wrapper

    def route_handler(self, path, handler, kwargs=None, name=None):
        if self.prefix:
            if path.startswith('/'):
                path = path[1:]
            path = '{0}{1}'.format(self.prefix, path)

        self.handlers.append((path, handler, kwargs, name))

    def create_resource(self, name, prefix):
        name = '{0}.{1}'.format(self.name, name)
        resource = _Resource(name, prefix)
        self.resources.append(resource)
        return resource

    def register_source(self, source):
        records = source.route_records
        for record in records:
            record(self)

    def register_resources(self):
        for resource in self.resources:
            self.register_source(resource)
```

## 三、Resource Class
```python
class _Resource:
    def __init__(self, name, prefix):
        self.name = name
        if not prefix.endswith('/'):
            prefix += '/'
        self.prefix = prefix
        self.route_records = []

    def route(self, path, kwargs=None, name=None):
        prefix = self.prefix
        if path.startswith('/'):
            path = path[1:]
        path = '{0}{1}'.format(prefix, path)

        def handler_wrapper(handler):
            if name is None:
                handler_name = '{0}.{1}'.format(self.name, handler.__name__)
            else:
                handler_name = name
            self.route_records.append(
                lambda x: x.route_handler(path, handler, kwargs=kwargs, name=handler_name))
            return handler
        return handler_wrapper
```

