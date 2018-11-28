---
title: '读 Flask:Blueprint 的大致实现'
date: 2016-09-18 05:56:53
categories:
- 技术
tags:
- python
- web 开发
- flask
- imhuwq
---

Flask 的 Blueprint 本质上是一个记录了一系列动作的类, 当 flask app 执行 `app.register_blueprint` 的时候把这些动作一股脑的全部倒入到 app 这个容器中去.  
其大致的执行顺序是:  
1. `app = Flask(__name__)`  
2. `bp = Blueprint('blueprint', __name__)`  
3. `bp.before_request' \ 'bp.route' \ 'bp.errorhandler`  
4. `app.register_blueprint`  
<!-- more -->
## 一、大致思路
第 3 步是 Blueprint 最常使用的地方, 被这些装饰器装饰的函数其实都是被 Blueprint 的 record 函数所调用.  
record 函数返回一个 lambda 函数并且把它保存在 Blueprint 的一个函数数列中(deferred\_funcs).  
所有这些 lambda 函数都只接受一个 BlueprintState 类的实例作为参数, 这个实例记录了 Blueprint 的状态(是否已经被注册, 所挂靠的 flask app 等).  
当 flask app 执行 register_blueprint 的时候, Flask 会迭代 Blueprint 的 deferred\_funcs 函数数列, 根据 BlueprintState 的状态来执行每个函数.  
最终的结果是被这些装饰器装饰的函数, 其实都被注册到了 flask app 中, 当然, 也加入了 Blueprint 的信息, 比如在 endpoint 之前加上 'blueprint\_name.'.  

## 二、源码分析
**2.1 `bp = Blueprint('bp', __name__)`**  
```python                                                                                                       
class Blueprint(_PackageBoundObject):                                                                           
    ...                                                                                                         
    def __init__(self, name, import_name, static_folder=None,                                                   
                 static_url_path=None, template_folder=None,                                                    
                 url_prefix=None, subdomain=None, url_defaults=None,                                            
                 root_path=None):                                                                               
        _PackageBoundObject.__init__(self, import_name, template_folder,                                        
                                     root_path=root_path)                                                       
        self.name = name                                                                                        
        self.url_prefix = url_prefix                                                                            
        self.subdomain = subdomain                                                                              
        self.static_folder = static_folder                                                                      
        self.static_url_path = static_url_path                                                                  
        self.deferred_functions = []                                                                            
        if url_defaults is None:                                                                                
            url_defaults = {}                                                                                   
        self.url_values_defaults = url_defaults                                                                 
    ...                                                                                                         
```

**2.2 `bp.route('/register', methods=['POST', 'GET']`**  
```python
def route(self, rule, **options):
    def decorator(f):
        endpoint = options.pop("endpoint", f.__name__)
        self.add_url_rule(rule, endpoint, f, **options)
        return f
    return decorator

def add_url_rule(self, rule, endpoint=None, view_func=None, **options):
    if endpoint:
        assert '.' not in endpoint, "Blueprint endpoints should not contain dots"
    self.record(lambda s:
        s.add_url_rule(rule, endpoint, view_func, **options))

def record(self, func):
    if self._got_registered_once and self.warn_on_modifications:
        from warnings import warn
        warn(Warning('The blueprint was already registered once '
                     'but is getting modified now.  These changes '
                     'will not show up.'))
    self.deferred_functions.append(func)
```

和 flask app 类似, `@route` 其实也是调用 `add_url_rule`, 不过在 blueprint 里面, `add_url_rule` 会去调用 `record` 函数.  
在调用 `record` 函数时, 传入的 lambda 函数是: `lambda s: s.add_url_rule(rule, endpoint, view_func, **options)`  
这个函数被添加到了蓝图的 `deferred_functions` 这个数列中.

**2.3 `bp.before_request`**  
```python
def before_request(self, f):
    self.record_once(lambda s: s.app.before_request_funcs
        .setdefault(self.name, []).append(f))
    return f

def before_app_request(self, f):
    self.record_once(lambda s: s.app.before_request_funcs
        .setdefault(None, []).append(f))
    return f

def before_app_first_request(self, f):
    self.record_once(lambda s: s.app.before_first_request_funcs.append(f))
    return f

def record_once(self, func):
    def wrapper(state):
        if state.first_registration:
            func(state)
    return self.record(update_wrapper(wrapper, func))
```

`before_request` 这一系列函数都是调用了 `record_once` 这个函数, 这个函数和 `record` 的区别在于, `record_once` 在把函数添加到 `deferred_function` 中之前, 会检查 blueprint 的状态.  
如果 blueprint 的状态不是第一次注册, 就会绕过这个函数.

**2.4 `app.register_blueprint(bp, url_prefix='bp')`**  
```python
def setupmethod(f):
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

@setupmethod
def register_blueprint(self, blueprint, **options):
    first_registration = False
    if blueprint.name in self.blueprints:
        assert self.blueprints[blueprint.name] is blueprint, \
            'A blueprint\'s name collision occurred between %r and ' \
            '%r.  Both share the same name "%s".  Blueprints that ' \
            'are created on the fly need unique names.' % \
            (blueprint, self.blueprints[blueprint.name], blueprint.name)
    else:
        self.blueprints[blueprint.name] = blueprint
        self._blueprint_order.append(blueprint)
        first_registration = True
    blueprint.register(self, options, first_registration)
```
`setupmethod` 是一个在 flask 类中使用的装饰器,因此第一个参数是 self.
这个装饰器的作用是确保函数在执行之前 处于 debug 模式的 app 没有收到过 request.  
`register_blueprint` 首先检查蓝图是否已经被注册, 如果没有, 把蓝图添加到 app 的 blueprints 字典以及 `_blueprint_order` 中.
最后, 调用蓝图的 `register` 函数, 传入 app 本身, options 字典和 first_registration = True. 

**2.5 `bp.register`**  
```python
def register(self, app, options, first_registration=False):
    self._got_registered_once = True
    state = self.make_setup_state(app, options, first_registration)
    if self.has_static_folder:
        state.add_url_rule(self.static_url_path + '/<path:filename>',
                           view_func=self.send_static_file,
                           endpoint='static')
    for deferred in self.deferred_functions:
        deferred(state)

def make_setup_state(self, app, options, first_registration=False):
    return BlueprintSetupState(self, app, options, first_registration)

class BlueprintSetupState(object):
    def __init__(self, blueprint, app, options, first_registration):
        self.app = app
        self.blueprint = blueprint
        self.options = options
        self.first_registration = first_registration
        subdomain = self.options.get('subdomain')
        if subdomain is None:
            subdomain = self.blueprint.subdomain
        self.subdomain = subdomain
        url_prefix = self.options.get('url_prefix')
        if url_prefix is None:
            url_prefix = self.blueprint.url_prefix
        self.url_prefix = url_prefix
        self.url_defaults = dict(self.blueprint.url_values_defaults)
        self.url_defaults.update(self.options.get('url_defaults', ()))

    def add_url_rule(self, rule, endpoint=None, view_func=None, **options):
        if self.url_prefix:
            rule = self.url_prefix + rule
        options.setdefault('subdomain', self.subdomain)
        if endpoint is None:
            endpoint = _endpoint_from_view_func(view_func)
        defaults = self.url_defaults
        if 'defaults' in options:
            defaults = dict(defaults, **options.pop('defaults'))
        self.app.add_url_rule(rule, '%s.%s' % (self.blueprint.name, endpoint),
                              view_func, defaults=defaults, **options)

```
`register` 函数主要干了四件事:  
 1. 把 blueprint 的 `_got_registered_once` 状态设置为 True  
 2. 创建一个 BlueprintSetupState 实例 state , 传入 app, blueprint 自身, 以及 options 和 first_registration = True 等信息  
 3. 如果 blueprint 自定义了 `static_folder`, 则调用 state 的 `add_url_rule` 方法,其作用是为 app 注册一个 bp.static 的 view\_func, url\_rule 为 `static_url_path + '/<path:filename>'`  
 4. 遍历 blueprint 的 `deferred_functions`, 执行其中的 lambda 函数, 传入 state 作为参数.    
 再来看一下那些 lambda 函数:  
 1. `lambda s: s.add_url_rule(rule, endpoint, view_func, **options)`  
 2. `lambda s: s.app.before_request_funcs.setdefault(self.name, []).append(f)`  
其实都是调用 state 的 app (也就是 blueprint 的 app) 对应的方法. 但是它能在执行调用函数之前,根据自身的状态来做一些绕行等操作.  

## 三、总结
我以前一直用一种父子关系来看待 app 和 blueprint, 即 blueprint 是 app 的一种特例, 它们有主从关系, 但是会同时运行. 
但是实际上, blueprint 注册的函数在执行 app.register 后就一股脑全部倒给 app 了, 也一直只有一个 app 实例在运行, 之后触发的函数都是添加了 blueprint 前缀的 app 的 view funcs.
在Flask 实现蓝图的方法中有两个亮点:
1. 把蓝图的 state 抽离出来成为一个类
2. 蓝图注册的方法全部放到回调函数数列中, 在注册的时候传入 state 参数

