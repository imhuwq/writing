---
title: '读Flask: 一次完整的Request和Response周期'
date: 2016-05-23 05:12:33
categories:
- 技术
tags:
- python
- web 开发
- flask
---

Flask只是一个python web框架，框架和server之间的数据交流，都是基于PEP3333所规范的WSGI， server调用framwork的某个callable进行数据交流。  这个callable， 可以是定义了`__call__`方法的类，或者任何函数等。  
而Flask应用的数据入口和出口(callable)就是Flask类实例的`wsgi_app`函数。  
```python
def wsgi_app(self, environ, start_response):
    with self.request_context(environ):
        rv = self.preprocess_request()
        if rv is None:
            rv = self.dispatch_request()
        response = self.make_response(rv)
        response = self.process_response(response)
        return response(environ, start_response)
```
`wsgi_app`接受从server发过来的`environ`环境变量，并且根据这个变量创建一个request context，然后在这个context下进行数据处理，最后返回数据到server。  
<!-- more -->
## 一、_RequestContext  
`wsgi_app`函数中的`with self.request_context(environ)`调用了`_RequestContext`这个类，这是一个定义了`__enter__`和`__exit__`的类。当使用`with`语法的时候，初始化类后，就会执行`__enter__`函数，`with`语块执行完毕后，就会执行`__exit__`函数。  
下面是`_RequestContext`的源码：  
```python
class _RequestContext(object):
    def __init__(self, app, environ):
        self.app = app
        self.url_adapter = app.url_map.bind_to_environ(environ)
        self.request = app.request_class(environ)
        self.session = app.open_session(self.request)
        self.g = _RequestGlobals()
        self.flashes = None

    def __enter__(self):
        _request_ctx_stack.push(self)

    def __exit__(self, exc_type, exc_value, tb):
        if tb is None or not self.app.debug:
            _request_ctx_stack.pop()

```
初始化_RequestContext:   
1. 为每个request context 绑定当前app实例；  
2. 并将环境变量于app实例的map绑定，获得一个map\_adapter用于匹配rule和view\_func；  
3. 根据环境变量创建一个request实例；  
4. 调用app的open\_session函数，传入request实例，获取cookie，创建一个session；  
5. 创建变量g；  
6. 创建flash消息容器。  

初始化完毕后，根据`__enter__`函数，把当前request\_context放入一个FILO的stack中，request\_context处理完毕后，如果没有错误，再从stack中删除这个request\_context。
既然每次request都有一个request\_context，那又为什么要把它们放入stack中呢？  
官方文档的解释是方便内部转发。  

## 二、`preprocess_request`
```python
def wsgi_app(self, environ, start_response):
    with self.request_context(environ):
        rv = self.preprocess_request()
        if rv is None:
            rv = self.dispatch_request()
        response = self.make_response(rv)
        response = self.process_response(response)
        return response(environ, start_response)
```
由上可知`preprocess_request`总是最先被调用。  
`preprocess_request`涉及的源码片段如下：  
```python
class Flask(object):
...
  def before_request(self, f):
      self.before_request_funcs.append(f)
      return f

  def preprocess_request(self):
      for func in self.before_request_funcs:
          rv = func()
          if rv is not None:
              return rv
```  
首先，被before\_request装饰的view_func都会被添加到app的`before_request_funcs`容器中；  
然后，`preprocess_request`遍历这个容器，执行里面的函数；  
如果其中任何一个函数有返回值，则停止遍历，返回该值。  

## 三、dispatch_request
```python
def wsgi_app(self, environ, start_response):
    with self.request_context(environ):
        rv = self.preprocess_request()
        if rv is None:
            rv = self.dispatch_request()
        response = self.make_response(rv)
        response = self.process_response(response)
        return response(environ, start_response)
```
由上可知，只有`request_context`没有返回值的时候，才会调用`dispatch_request`。说明`preprocess_request`的返回值可以越过正常的`dispatch_request`这一步。  
`dispatch_request`涉及的源码片段如下：  
```python
class Flask(object):
    ...
    def match_request(self):
        rv = _request_ctx_stack.top.url_adapter.match()
        request.endpoint, request.view_args = rv
        return rv

    def dispatch_request(self):
        try:
            endpoint, values = self.match_request()
            return self.view_functions[endpoint](**values)
        except HTTPException as e:
            handler = self.error_handlers.get(e.code)
            if handler is None:
                return e
            return handler(e)
        except Exception as e:
            handler = self.error_handlers.get(500)
            if self.debug or handler is None:
                raise
            return handler(e)
    ...

request = LocalProxy(lambda: _request_ctx_stack.top.request)
```
首先`dispatch_request`会调用app实例的`match_request`函数；  
 - 从\_request\_ctx\_stack中获取最新的request\_context  
 - 调用这个request\_context的url\_adapter，执行它的match函数，获得其返回值rv  
 - 解析rv，分别存储到request\_context的request实例的endpoint和view\_args属性中  
 - 返回rv  
然后从`match_request`的返回值中获得endpoint和values；  
根据endpoint在app实例的`view_functions`字典中索引到`view_func`函数对象，传入values作为参数。  
如果以上步骤报错，首先看看是不是HTTP类型的错误，如果是，根据HTTP错误的错误代码在app实例的`error_handlers`字典中索引`error_handler`函数，执行该函数；  
如果是其它类型的错误，调用500错误的`error_handler`函数， 如果是debug模式或者该`error_handler`函数没有找到，报错；否则，执行该函数。  
不管是调用的`view_func`还是`error_handler`，其结果都被返回。  

## 四、`make_response`
```python
def wsgi_app(self, environ, start_response):
    with self.request_context(environ):
        rv = self.preprocess_request()
        if rv is None:
            rv = self.dispatch_request()
        response = self.make_response(rv)
        response = self.process_response(response)
        return response(environ, start_response)

```
不管是从`preprocess_request`还是`dispatch_request`中获得了返回值，最后都调用app实例的`make_response`函数。  
这个函数的作用主要是对返回值通过Response类进行包装。  

## 五、process_response
```python
def wsgi_app(self, environ, start_response):
    with self.request_context(environ):
        rv = self.preprocess_request()
        if rv is None:
            rv = self.dispatch_request()
        response = self.make_response(rv)
        response = self.process_response(response)
        return response(environ, start_response)
```

获得一个Response实例后，再将这个实例传入`process_response`函数，进行最后的处理。  

```python
def process_response(self, response):
    session = _request_ctx_stack.top.session
    if session is not None:
        self.save_session(session, response)
    for handler in self.after_request_funcs:
        response = handler(response)
    return response
```

首先检查`request_context`的`session`是否为空，如果不是，调用app实例的`save_session`函数，把`session`保存到response实例中；  
然后遍历app的`after_request_funcs`，调用这些函数处理response；  
  - 这里也说明每个after\_request\_func至少接受一个response作为参数，返回一个response。  
最后返回response。

## 六、return response
```python
def wsgi_app(self, environ, start_response):

    with self.request_context(environ):

        rv = self.preprocess_request()

        if rv is None:

            rv = self.dispatch_request()

        response = self.make_response(rv)

        response = self.process_response(response)

        return response(environ, start_response)

```
最后一个语句`return response(environ, start_response)`是WSGI规范的语法，表明数据处理完毕，开始返回。
以上就是一次完整的request和response周期，简单概括起来就是如下：  
1. 传入环境变量创建request\_context；  
2. 遍历before\_request\_funcs，尝试获取返回值；  
3. dispatch\_request，dispatch\_request做的事情就多了，但都是在request\_context下进行的：  
  - `match_request`从request\_context中获得`rv`，解析`rv`获得endpoint和args，记录到request\_context的`request`实例中，返回`rv`    
  - 从`rv`中获得`endpoint`和`args`，匹配到view\_func，执行`view_func(**args)`，获得返回值  
  - 报错，则匹配到error\_handler，获得返回值  
4. 对第2步或者第3步的返回值通过`make_response`包装成`Response`类的实例`response`；  
5. `process_response`检查cookie和保存cookie到`response`，再遍历after\_request\_funcs，对`response`进行最后的处理;
6. 返回`response`

## 七、一些常用的变量或者方法
`render_template`， `url_for`, `flash`和`current_app`， `request`是常用的方法以及变量，下面是对它们源码的分析。  

**7.1 `url_for`**  
```python
 def url_for(endpoint, **values):
     return _request_ctx_stack.top.url_adapter.build(endpoint, values)
```
调用request\_context的url_adapter，执行它的`build`函数。  
`build`函数和`match`函数不同，它接受endpoint和values参数，返回构建的url，而`match`返回endpoint和values。  

**7.2 flash**  
```python
def flash(message):
    session['_flashes'] = (session.get('_flashes', [])) + [message]

def get_flashed_messages():
    flashes = _request_ctx_stack.top.flashes
    if flashes is None:
        _request_ctx_stack.top.flashes = flashes = \
            session.pop('_flashes', [])
    return flashes
session = LocalProxy(lambda: _request_ctx_stack.top.session)
```

当`flash(msg)`的时候，`msg`其实是被存储到session中的；  
当获取`msg`的时候，首先尝试从request\_context中获取flashes， 如果为空，则从session中pop一个(获取并删除)，再返回。  

**7.3 current_app和request**  
`current_app`其实就是request\_context所绑定的app实例:  
```python
current_app = LocalProxy(lambda: _request_ctx_stack.top.app)
```
`request`则是在初始化request\_context的时候根据环境变量而实例化的一个`Request`类：  
```python
class _RequestContext(object):
    def __init__(self, app, environ):
        self.app = app
        ...
        self.request = app.request_class(environ)
        ...
request = LocalProxy(lambda: _request_ctx_stack.top.request)
```

**7.4 render_template**  
`render_template`调用了jinja2，但是有一些`request`这样的变量在template中也适用，是因为flask把这些变量设置成了默认的template\_context。
```python
def render_template(template_name, **context):
    current_app.update_template_context(context)
    return current_app.jinja_env.get_template(template_name).render(context)

def _default_template_ctx_processor():
    """Default template context processor.  Injects `request`,
    `session` and `g`.
    """
    reqctx = _request_ctx_stack.top
    return dict(
        request=reqctx.request,
        session=reqctx.session,
        g=reqctx.g
    )

class Flask(object):
    def __init__(self, package_name):
        ...
        self.template_context_processors = [_default_template_ctx_processor]
        ...
    ...
    def update_template_context(self, context):
    reqctx = _request_ctx_stack.top
    for func in self.template_context_processors:
        context.update(func())
    ...
```

一般情况下，`render_template`的语法是：  
```python
render_template('index.html', title=title, name=name, about=about)
```

其中`'index.html'`是`template_name`， `title=title, name=name, about=about`合并为`context`字典；  
在调用jinja2渲染模板之前，Flask首先调用request\_context下的`current_app`，执行其`update_template_context`函数，传入`context`字典；  
app实例的`update_template_context`会遍历app实例的`template_context_processors`，将其中每一个函数的返回值与`context`字典合并；  
而app实例默认的template\_context\_processor就是`_default_template_ctx_processor`，它会返回一个包含了request\_context下的`request`, `session`和`g`的字典。  
所以不管如何，jinja2都会得到一个包含了`request`, `session`和`g`的template\_context。  
最后渲染的模板：  
```python
return current_app.jinja_env.get_template(template_name).render(context)
```

## 八、局限以及后续方向
局限：  
1. 版本是Flask 0.1  
2. Werkzeug提供的`Map`, `Rule`, `Request`, `Response`, `SecureCookie`类是非常重要的类，但目前还是个黑匣子

后续方向：  
继续往Flask更新的版本读，Werkzeug暂时不读源码，但需要理解调用模块的大致作用。  

