---
title: Python 的赋值和作用域
date: 2017-04-17 06:25:15
categories:
- 技术
tags:
- python
---

## 一、Python 中的赋值
**1.1 Python 中万物皆对象**  
```python
print(type(1))  # <class 'int'>
```

**1.2 Python 中赋值的意义**   
2.1 如果 = 号右边是*对象*， 则让 = 号左边的变量指向该对象， 成为对该对象的一个 **引用**  
2.2 如果 = 号右边是*变量*， 则让 = 号左边的变量指向右边变量所指的对象， 成为那个对象的 **引用**  
2.3 通过其它表达式而不是等号进行赋值的操作， 意义与上面相同
<!-- more -->
```python
# 定义一个变量 test_number， 该变量为对常量 1 的引用
test_number = 1
#　再定义一个变量 another_number, 该变量和 test_number　指向同一个对象(常量 1 )
another_number = test_number
assert another_number is test_number
# 改变 test_number, 令其指向常量 2, 此时 test_number 和 another_number 指向不同的变量
test_number = 2
assert another_number is not test_number
# 定义一个变量 test_dict, 该变量为对一个 dict 对象的引用
test_dict = {"name": "test_dict", "test_number": test_number}
# test_dict 的　test_number 属性和 test_number 指向同一个对象
assert test_dict["test_number"] is test_number
# 再定义一个变量　another_dict, 该变量和 test_dict 指向同一个 dict 对象
another_dict = test_dict
assert another_dict is test_dict
# 改变该 dict 对象的 "test_number" 属性, 令其和 another_number 指向同一个对象(常量１)
test_dict["test_number"] = another_number
# 由于改变的只是 dict 对象的属性, test_dict 和 another_dict 仍然指向的同一个对象
assert another_dict["test_number"] is another_number
assert another_dict is test_dict
```

### 二. Python 中变量的作用域
任何变量都有自己的作用域， 作用域有优先级。在 Python 中，作用域的优先级是这样的：  
1. 当前函数的作用域  
2. 外围的作用域  
3. 模块级别的作用域  
4. 内置函数的作用域  
对处于高优先级作用域中的变量进行更改， 并不会更改到低优先级作用域中的变量。
```python
test_number = 1
def test_function():
    print(test_number)
    test_number = 2

test_function()
print(test_number)
```

猜猜这段代码的输出？
其实这段代码会报错：

```python
---------------------------------------------------------------------------
UnboundLocalError                         Traceback (most recent call last)
<ipython-input-21-ee857f9d40b6> in <module>()
      5     test_number = 2
      6
----> 7 test_function()
      8 print(test_number)

<ipython-input-21-ee857f9d40b6> in test_function()
      2
      3 def test_function():
----> 4     print(test_number)
      5     test_number = 2
      6
UnboundLocalError: local variable 'test_number' referenced before assignment
```

为什么呢？
从直觉上来看， `print(test_number)` 时， `test_function` 作用域内还没有对 `test_number`
的定义， 所以应该输出模块级别作用域中的 `test_number`， 也就是 1. 后来 `test_number = 2`
只是在函数作用域内部修改了 `test_number`, 并不影响外部。 所以结果应该是：
```python
1
1
```

但其实上面的逻辑有错误。
>`test_function` 作用域内还没有对 `test_number` 的定义...后来 `test_number = 2` 只是
在函数作用域内部修改了 `test_number`  

`test_number = 2` 到底是对函数内部 `test_number` 的赋值还是定义? 如果是定义， 之前的
`print(test_number)` 中的 `test_number` 是什么意思？要知道当时它都还没有被定义。
而如果是赋值， 那它是什么时候定义的？
为了避免这种 delimma， Python 有这么一个规范， **在当前作用域内，每一个变量在第一次被赋值时
才被定义**。
那这样的话， 在上面的例子中， `test_number` 在第 5 行时被定义， 导致作用域内有了该变量，
第 4 行就不能再往外引用了。
而第 4 行时是无法得到 `test_number` 的引用的， 因为它到第 5 行才被定义。
删掉第 4 行或者第 5 行就会消除错误。

## 三、一些简单的应用
**3.1 首先是这个经典的函数参数问题**
```python
def test_function_locals(arg=[], arg2={}):
    arg.append(1)
    print(arg)

test_function_locals()
test_function_locals()
test_function_locals()

```
其结果是什么想必大家都知道了。但我以前只知道不应该使用 mutable 作为参数， 却不明白原因。
原因是什么呢？

```python
print(dir(test_function_locals))  # [..., __call__, ...]
print(test_function_locals.__defaults__)  # ([1, 1, 1], {}), 是一个 tuple
```

函数是个 `callable` 对象，这两个参数都是 `test_function_locals` 这个函数对象的属性， 属于
这个函数对象内部的作用域。
所以可以把函数调用想象成调用某个 object 的 `__call__` 函数， 该 `__call__` 函数引用并修改了
该 object 的 instance attribute。

**3.2 还是一个函数相关的问题**
```python
lambdas = []
for i in range(3):
    lambdas.append(lambda : print(i))

for lambda_func in lambdas:
    lambda_func()
```
结果是什么大家也知道。原因呢？
在三次迭代中， lambdas 分别加入了指一个不同的函数对象， 而不是重复加入了同一个函数对象三次。
这三个函数对象虽然不同，但所使用的 i 都是对外部作用域的 i 的引用。
当这三个函数被分别调用时， i 的值已经是 2 了。
并且， 如果在调用这些函数之前 `del i`， 则会报错:

```python
lambdas = []
for i in range(3):
    lambdas.append(lambda : print(i))

del i
for lambda_func in lambdas:
    lambda_func()
---------------------------------------------------------------------------
NameError                                 Traceback (most recent call last)
<ipython-input-61-66ac36a63814> in <module>()
      5 del i
      6 for lambda_func in lambdas:
----> 7     lambda_func()

<ipython-input-61-66ac36a63814> in <lambda>()
      1 lambdas = []
      2 for i in range(3):
----> 3     lambdas.append(lambda : print(i))
      4
      5 del i

NameError: name 'i' is not defined
```

