---
title: '读 Python: functools.wraps 做了什么'
date: 2016-09-17 06:25:15
categories:
- 技术
tags:
- python
- imhuwq
---

Python 标准库 functools 里面的 `wraps` 装饰器经常用于写装饰器，之前只知道它可以用来保留被装饰函数的元数据，但具体实现的方式和究竟保留了哪些数据都不清楚。最近看 Flask 源码时读到一行代码 `return self.record(update_wrapper(wrapper, func))`。 稍微看了一下原来 `@wraps(func)` 其实就是调用了 `update_wrapper` 这个函数，于是索性看个明白。
<!-- more -->
## 一、相关源码
```python
WRAPPER_ASSIGNMENTS = ('__module__', '__name__', '__qualname__', '__doc__',
                       '__annotations__')
WRAPPER_UPDATES = ('__dict__',)
def update_wrapper(wrapper,
                   wrapped,
                   assigned = WRAPPER_ASSIGNMENTS,
                   updated = WRAPPER_UPDATES):
    for attr in assigned:
        try:
            value = getattr(wrapped, attr)
        except AttributeError:
            pass
        else:
            setattr(wrapper, attr, value)
    for attr in updated:
        getattr(wrapper, attr).update(getattr(wrapped, attr, {}))
    wrapper.__wrapped__ = wrapped
    return wrapper

def wraps(wrapped,
          assigned = WRAPPER_ASSIGNMENTS,
          updated = WRAPPER_UPDATES):
    return partial(update_wrapper, wrapped=wrapped,
                   assigned=assigned, updated=updated)
```

## 二、源码分析
这段代码其实很简单:  
1. `@wraps` 默认保留的原来函数的 `__module__`, `__name__`, `__qualname__`, `__doc__` 和 `__annotations__` 数据；  
2. `@wraps` 默认 update 新函数的 dict 属性；  
3. `@wraps(funcs)` 返回的是一个 `update_wrapper` 的 partial， 默认 `wrapped` 参数为传入的 `func`， 被它装饰的函数作为 `update_wrapper` 的 `wrapper` 参数；  
4. `update_wrapper` 只是 update `wrapper` 的元数据，没做其它任何事情。

## 三、其它收获
`try...except...` 中 `try` 的部分应该只包含可能会出错部分的代码，其它的放到 `else` 中去：  
```python
# 正确的方法
try:
    f = open('some-file.txt')
except IOError:
    print("File doesn't exist :-(")
else:
    print(f.read())

# 不合适的方法
try:
    f = open('some-file.txt')
    print(f.read())
except IOError:
    print("File doesn't exist :-(")
```
