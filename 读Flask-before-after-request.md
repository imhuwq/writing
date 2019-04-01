---
title: '读Flask:before(after)_request'
date: 2016-05-18 05:35:19
categories:
- 技术
tags:
- python
- web 开发
- flask
---

Flask中可以使用`@before_request`等装饰器来注册一些函数，这些函数会在view_func之前被执行，通常可以用来做数据库初始化等操作。  
```python  
def before_request(self, f):
    """Registers a function to run before each request."""
    self.before_request_funcs.append(f)
    return f

def preprocess_request(self):
    for func in self.before_request_funcs:
        rv = func()
        if rv is not None:
            return rv
```
<!-- more -->
从上面的源码可以看出来，被装饰的函数会被注册到Flask实例的`before_request_funcs`数列中，而在call `view_func`之前，会遍历这个数列并执行这些函数，并且如果有返回值的话，停止遍历并把返回值作为`view_func`的返回值进行返回。  
在Flask v0.1中还没有`before_first_request`，不过根据这个思路来也很容易实现。  
```python
class Flask(object):
    def __init__(self, ...):
        self.initiated = False
        self.before_first_request_funcs = []
        ...
    ...
    def before_first_request(self, f):
        self.before_first_request_funcs.append(f)
        return f

    def preprocess_request(self):
        if not self.initiated:
            for func in self.before_first_request_funcs:
                rv = func()
                if rv is not None:
                    return rv
            self.initiated = True

        for func in self.before_request_funcs:
            rv = func()
            if rv is not None:
                return rv
```

