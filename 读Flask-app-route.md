---
title: '读Flask:app.route'
date: 2016-05-14 05:44:32
categories:
- 技术
tags:
- python
- web 开发
- flask
---

追本朔源弄清楚`@app.route`后面到底发生了什么。  
理解以备注的形式写在源码上，截取的源码只有相关的片段。
<!-- more -->
## 一、`app.route` 的定义
找到`app.route`：  
1. `app.route`调用了app的`route`函数来装饰一个view函数  
2. `app`是`Flask`类的实例，创建于`app = Flask(__name__)`

## 二、Flask类的定义
**2.1 找到Flask类**  
先直接找到`class Flask`：  
```python
# Flask类是_PackageBoundObject的子类
class Flask(_PackageBoundObject):
  ...
  def __init__(self, import_name, static_path=None, static_url_path=None,
             static_folder='static', template_folder='templates',
             instance_path=None, instance_relative_config=False):
     # 调用父类并初始化
    _PackageBoundObject.__init__(self, import_name,
                                 template_folder=template_folder)
    # Flask实例的view_funcs集合
    self.view_functions = {}
    ...
  ...      
```

找到`_PackageBoundObject`类：      
```python
class _PackageBoundObject(object):
    def __init__(self, import_name, template_folder=None):
        # 获得import_name属性，值为声明类时的第一个key arg
        self.import_name = import_name
        self.template_folder = template_folder
        ...
```

**2.2 Flask类的`route`函数**  
再看看`route`函数的作用：  
```python
class Flask(_PackageBoundObject):
  ...
  def route(self, rule, **options):
      # 说明route是一个装饰器
      def decorator(f):
          # 如果参数中有endpoint，就把它赋给endpoint变量并且从参数中删除掉
          # 如果没有，就赋予endpoint变量为None的值
          endpoint = options.pop('endpoint', None)
          # 再调用add_url_rule函数
          #     传入 rule, endpoint, 被装饰的函数, 和其它所有参数
          self.add_url_rule(rule, endpoint, f, **options)
          return f
      return decorator
  ...
```

**2.3 Flask类的`add_url_rule`函数**
通过上面分析可以知道`route`函数主要是对`add_url_rule`函数的调用：  
```python
# add_url_rule也被装饰，源代码稍后分析
@setupmethod
def add_url_rule(self, rule, endpoint=None, view_func=None, **options):
    # 如果没有指定endpoint，则设置endpoint为函数名称
    if endpoint is None:
        # _endpoint_from_view_func的作用是返回函数名称，代码就不贴了
        endpoint = _endpoint_from_view_func(view_func)
    # 把endpoint加入options字典
    options['endpoint'] = endpoint
    从options字典中获取methods
    methods = options.pop('methods', None)
    # 如果没有指定methods，首先检查view_func有没有methods属性
    # 如果也没有，就设为’GET‘
    if methods is None:
        methods = getattr(view_func, 'methods', None) or ('GET',)
    # methods去重
    methods = set(methods)
    # 从view_func函数中获得这些，作用暂时未知
    required_methods = set(getattr(view_func, 'required_methods', ()))
    provide_automatic_options = getattr(view_func,
        'provide_automatic_options', None)
    if provide_automatic_options is None:
        if 'OPTIONS' not in methods:
            provide_automatic_options = True
            required_methods.add('OPTIONS')
        else:
            provide_automatic_options = False

    # 合并required_methods 和methods
    methods |= required_methods
    ...
    # 这里调用了Flask类的url_rule_class
    rule = self.url_rule_class(rule, methods=methods, **options)
    rule.provide_automatic_options = provide_automatic_options
    # 这里调用了Flask类的url_map
    # url_map的add方法接受一个Rule实例
    self.url_map.add(rule)
    # view_func排重检查
    if view_func is not None:
        # 根据endpoint在Flask实例的view_functions中检索已经存在的view_func
        old_func = self.view_functions.get(endpoint)
        # 如果view_func已经在Flask实例已有的view_functions中,
        #    并且二者不是同一个函数，则抛出错误
        if old_func is not None and old_func != view_func:
            raise AssertionError('View function mapping is overwriting an '
                                 'existing endpoint function: %s' % endpoint)
        # 把view_func添加到Flask实例的view_functions字典中
        # key为endpoint, value为view_func函数
        self.view_functions[endpoint] = view_func
        # 从对view_func的处理可知：
        # 调用一次@app.route，就会把被装饰的函数注册到Flask实例的view_functions字典中
        #        这个字典中的key=value是endpoint=view_func的形式
        #        endpoint可以在route函数的参数中指定
        #        所以理论上可以使用同名的view_func函数，但是需要为函数设置不同的endpoint
```
看看`setupmethod`函数做了什么：  

```python
def setupmethod(f):
    # 涉及到验证，已经说得很明白
    # 不太清楚具体作用，但目前似乎不需要
    def wrapper_func(self, *args, **kwargs):
        if self.debug and self._got_first_request:
            raise AssertionError('A setup function was called after the '
                'first request was handled.  This usually indicates a bug '
                'in the application where a module was not imported '
                'and decorators or other functionality was called too late.\n'
                'To fix this make sure to import all your view modules, '
                'database models and everything related at a central place '
                'before the application starts serving requests.')
        return f(self, *args, **kwargs)
    return update_wrapper(wrapper_func, f)
```

**2.4 Flask实例的`url_rule_class` 和`url_map`**   
```python
class Flask(_PackageBoundObject):
  ...
  # url_rule_class是Flask类的属性，调用了Rule类
  url_rule_class = Rule
  ...
  def __init__(...):
    ...
    # url_map是Flask实例的属性，调用了Map类
    self.url_map = Map()
    ...
    # Rule类和Map类都是werkzeug里routing模块下的类
```

## 三、本文总结
总结和`@app.route`相关的引用以及其作用。  

**3.1 Flask类**  
Flask类基于`_PackageBoundObject`，其实`_PackageBoundObject`类也只是初始化`import_name`和`template_folder`这个属性，方便文件路径处理。  
Flask实例有一个`view_functions`字典，里面的key=value记录了该实例的`view_func`，key为`view_func`的`endpoint`(默认为函数名称，也可以在route函数的参数中指定), value为函数本身。  

**3.2 `route`函数**  
`route`函数是一个装饰器，会从参数中获取`endpoin`t(如果没有则为None)，并把`rule`, `endpoint`, 和被装饰的函数等作为参数传给`add_url_rule`函数。  
`route`函数的核心还是`add_url_rule`函数。  

**3.3 `add_url_rule`函数**  
首先会试图从参数或者函数名称中获取endpoint。  
然后从参数或者`view_func`属性中获取methods。  
然后调用`Flask`实例的`url_rule_class`， 这是一个Rule类。传入`rule`,` methods`和`view_func`剩下的其它参数，实例化一个`Rule`类。  
再调用Flask实例的`url_map`，这是一个`Map`类的实例。 调用这个类的`add`函数，传入刚才实例化的`Rule`类。

**3.4 `Rule`类和`Map`类**  
`Rule`和`Map`都是werkzeug库routing模块下的类。  
一个Flask app 通常都有一个`Map`实例，里面包含若干在`add_url_rule`函数中被创建的`Rule`实例。  
在收到一个`Request`之后，通常会遍历`Map`实例，获得请求路径与其对应的view\_func，从而触发对应的函数。  

