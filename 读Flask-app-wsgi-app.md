---
title: '读Flask:app.wsgi_app'
date: 2016-05-18 05:30:37
categories:
- 技术
tags:
- python
- web 开发
- flask
---

`app.wsgi_app` 是Flask实例的入口，Flask从这里接受数据，返回处理后的数据。  
<!-- more -->
```python
class _RequestContext(object):
    def __init__(self, app, environ):
        # 绑定Flask实例
        self.app = app
        # 调用Flask实例的Map实例， 调用其bind_to_environ函数，传入环境变量，获得MapAdapter实例
        self.url_adapter = app.url_map.bind_to_environ(environ)
        self.request = app.request_class(environ)
        self.session = app.open_session(self.request)
        self.g = _RequestGlobals()
        self.flashes = None

    def __enter__(self):
        # _request_ctx_stack顶部增加一个_RequestContext实例
        _request_ctx_stack.push(self)

    def __exit__(self, exc_type, exc_value, tb):
        if tb is None or not self.app.debug:
            # _request_ctx_stack顶部移除_RequestContext实例
            _request_ctx_stack.pop()

class Flask(object):
    def wsgi_app(self, environ, start_response):
        # 获取环境变量，传入request_context函数
        # 得到一个_RequestContext类的实例
        # 先后执行_RequestContext类的__init__ 和__enter__
        with self.request_context(environ):
            # 执行preprocess_request, before_request等函数在这里面被执行
            rv = self.preprocess_request()
            # 如果没有返回值，则开始调用dispatch_request
            # 进行正常的path解析和view_func调用等
            if rv is None:
                rv = self.dispatch_request()
            # 而如果有返回值，则把该返回值进行返回
            #         而不再进行正常的dispatch_request
            response = self.make_response(rv)
            response = self.process_response(response)
            return response(environ, start_response)

    def request_context(self, environ):
        return _RequestContext(self, environ)

```

目前只看了`dispatch_request`后面的路径，session等都是在这里开始的。  
大致的路径是：  
1. 获取环境变量；  
2. 根据环境变量创建`_RequestContext`实例；  
3. 根据环境变量创建`MapAdapter`实例；  
4.` _request_ctx_stack`顶部增加一个`_RequestContext`实例；   
5. 执行`preprocess_request`, `before_request`等函数在这里面被执行；  
6.  如果`preprocess_request`没有返回值，则开始调用`dispatch_request`，进行正常的path解析和view\_func调用等；  
7.  而如果有返回值，则把该返回值进行返回， 而不再进行正常的`dispatch_request`；  
8.  `_request_ctx_stac`k顶部移除`_RequestContext`实例。

