---
title: '读Flask:dispatch_request'
date: 2016-05-18 05:24:13
categories:
- 技术
tags:
- python
- web 开发
- flask
---

之前读到 `app.route` 实际上是调用 `add_url_rule` 函数，主要做了两件事:  
1. 注册 `view_func` 到Flask实例  
2. 建立一个Rule实例，绑定Rule实例到Map实例  
那收到一个匹配Map中某个Rule之后，怎么再call到原来的 `view_func` 呢？  
<!-- more -->
```python
def match_request(self):
    # _request_ctx_stack是一个stack类型的数列
    # 里面存储的是_RequestContext实例
    # 这个实例的url_adapter是Flask实例的Map实例所返回的MapAdapter实例
    # MapAdapter实例的match函数遍历Map实例中的所有Rule实例，执行Rule实例的match函数
    # Rule实例的match函数返回rule所绑定的endpoint和其它参数
    rv = _request_ctx_stack.top.url_adapter.match()
    # 解析出endpoint和其它参数
    request.endpoint, request.view_args = rv
    return rv

def dispatch_request(self):
    try:
        # 获得endpoint和参数
        endpoint, values = self.match_request()
        # 根据endpoint在view_funcs字典中找出函数，传入参数
        return self.view_functions[endpoint](**values)

    # 如果HTTP类型报错
    except HTTPException, e:
        # 检查是否有自定义error_handler
        handler = self.error_handlers.get(e.code)
        # 没有，返回错误
        if handler is None:
            return e
        # 有，执行error handler， 传入错误代码作为参数
        return handler(e)

    # 其它类型报错
    except Exception, e:
        # 类似上面的过程，不过只搜寻500 error handler
        handler = self.error_handlers.get(500)
        if self.debug or handler is None:
            raise
        return handler(e)
    # 由错误处理过程也可以知道，@error_handle所注册的函数，是以错误代码为id的
```
从这里可以知道，Flask每收到一次request时，都会遍历一次map， 执行map中每个rule的match函数，根据request路径进行匹配，如果匹配成功则获得endpoint和参数，再根据字典获得view_func，并传入参数。

