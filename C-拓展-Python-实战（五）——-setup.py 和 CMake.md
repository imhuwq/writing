---
title: C 拓展 Python 实战（五）—— setup.py 和 CMake
date: 2020-03-22 13:42:00
categories:
- 技术
tags:
- c++
- python
---

这是我的《C 拓展 Python 实战》系列的第五篇，也是这个系列最后一篇。  
在之前的文章中，我们的模块需要和 Python 一起编译，这次我们来看看如何使用 setup.py 来随时编译我们的模块。  
在此之上，我们再来看看如何结合 CMake 来把复杂的 C/C++ 项目打包成 Python Package。  
<!-- more -->

## 一、使用 setup.py 来打包 C/C++ 模块为 Python Package
新建一个项目目录，结构如下：  

```shell
py-cmodule-demo       # 项目目录
├── demo              # 常规的 python module
│   ├── __init__.py   # 常规的 python 文件
├── mymath            # c 模块目录
│   └── mymath.c      # c 模块代码
└── setup.py          # 安装脚本
```

`mymath.c` 文件的内容和我们之前的一样，是用 C 写的 Pyhon 模块：

``` c
/* mymath.c */
#include <Python.h>

PyObject *sum(PyObject *self, PyObject *args) {
    int num1, num2;
    PyArg_ParseTuple(args, "ii", &num1, &num2);
    return Py_BuildValue("i", num1 + num2);
}

static PyMethodDef MymathMethods[] = {
        {"sum", sum, METH_VARARGS, "add up two numbers and return their sum"},
        {NULL}
};

static struct PyModuleDef mymath_module = {
        PyModuleDef_HEAD_INIT,
        "mymath",
        PyDoc_STR("A hello world example to demonstrate writing a python module in c language"),
        -1,
        MymathMethods
};

PyMODINIT_FUNC PyInit_mymath(void) {
    PyObject *m = PyModule_Create(&mymath_module);
    return m;
}

```

最关键的部分在 `setup.py`，安装时所有的依赖和行为都在这里定义：

``` python
from setuptools import setup
from setuptools import Extension

mymath = Extension("mymath",
                   sources=["mymath/mymath.c"],
                   include_dirs=["/usr/include/python3.6m"], # 编译时需要的 include dirs
                   library_dirs=["/usr/lib/x86_64-linux-gnu/"] # 编译链接时寻找链接库的目录
                   )

setup(name="demo", # 我们的 package 的名字
      version="1.0", # 版本
      description="This is a demo package",
      packages=["demo"], # 打包时需要带上的本地 python 包
      ext_modules=[mymath] # 打包时需要编译的 c 模块
      )

```

最基本的框架就如上所示，比较简单。  
下面我们来试试：

``` shell
$ python setup.py install

$ python
Python 3.6.9 (default, Feb 26 2020, 21:43:10) 
[GCC 7.4.0] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> import mymath
>>> help(mymath)

>>> mymath.__doc__
'A hello world example to demonstrate writing a python module in c language'
>>> mymath.sum(99, 1)
100
>>> 

```


## 二、setup.py 如何使用 CMake 来编译复杂的项目
上面使用 `setup.py` 编译了一个简单的 C 模块。不过它的弊端也很明显，那就是需要在 `setup.py` 里面组织 C 项目的结构，包括指定 sources, include_dirs, libraries 和其它编译选项。当项目变得复杂的时候，`setup.py` 也就变得异常难以管理。  
而实际上，C 和 C++ 本身就有强大的 CMake 这个工具来进行项目管理了。在 `setup.py` 里面能不能直接用 CMake 呢？  
我都在写这篇文章了，答案当然是有。不过有一些细节需要注意到。  

加入 `CMakeLists.txt`，并修改 `setup.py`：  

``` shell
py-cmodule-demo
├── demo
│   ├── __init__.py
├── mymath
│   ├── CMakeLists.txt    # 加入 CMakeLists.txt
│   └── mymath.c
└── setup.py              # 修改 setup.py

```

先来看看 `CMakeLists.txt`：

``` cmake
project(mymath)
cmake_minimum_required(VERSION 3.10)

set(Python3_USE_STATIC_LIBS FALSE)
#set(CMAKE_POSITION_INDEPENDENT_CODE ON)  # 不知道为什么，setup.py 忽略了这个设置，所以要用上面那行命令

find_package(Python3 COMPONENTS Interpreter Development)

set(SOURCES "")
set(INCLUDES "")
set(LIBRARIES "")

list(APPEND SOURCES mymath.c)
list(APPEND INCLUDES ${Python3_INCLUDE_DIRS})
list(APPEND LIBRARIES ${Python3_LIBRARIES})

include_directories(${INCLUDES})
add_library(mymath SHARED ${SOURCES})
target_link_libraries(mymath PUBLIC ${LIBRARIES})

set_target_properties(mymath PROPERTIES OUTPUT_NAME "mymath" PREFIX "")  # 这一行非常重要，作用是取消编译出的库文件的 ‘lib’ 前缀，这样 python 编译器才能找到这个包

```

这个 `CMakeLists.txt` 文件虽小，但是五脏俱全。尤其需要注意最后一行，当时我在这里卡了很久。  

接下来对 `setup.py` 进行修改，让它能够根据 CMake 进行编译：

``` python
import os
import pathlib
from setuptools import setup
from setuptools import Extension
from setuptools.command.build_ext import build_ext


class CMakeExtension(Extension):
    """
    自定义了 Extension 类，忽略原来的 sources、libraries 等参数，交给 CMake 来处理这些事情
    """

    def __init__(self, name):
        super().__init__(name, sources=[])


class BuildExt(build_ext):
    """
    自定义了 build_ext 类，对 CMakeExtension 的实例，调用 CMake 和 Make 命令来编译它们
    """
    def run(self):
        for ext in self.extensions:
            if isinstance(ext, CMakeExtension):
                self.build_cmake(ext)
        super().run()

    def build_cmake(self, ext):
        cwd = pathlib.Path().absolute()

        build_temp = f"{pathlib.Path(self.build_temp)}/{ext.name}"
        os.makedirs(build_temp, exist_ok=True)
        extdir = pathlib.Path(self.get_ext_fullpath(ext.name))
        extdir.mkdir(parents=True, exist_ok=True)

        config = "Debug" if self.debug else "Release"
        cmake_args = [
            "-DCMAKE_LIBRARY_OUTPUT_DIRECTORY=" + str(extdir.parent.absolute()),
            "-DCMAKE_BUILD_TYPE=" + config
        ]

        build_args = [
            "--config", config,
            "--", "-j8"
        ]

        os.chdir(build_temp)
        self.spawn(["cmake", f"{str(cwd)}/{ext.name}"] + cmake_args)
        if not self.dry_run:
            self.spawn(["cmake", "--build", "."] + build_args)
        os.chdir(str(cwd))


mymath = CMakeExtension("mymath")

setup(name="demo",
      version="1.1",
      description="This is a demo package",
      packages=["demo"],
      ext_modules=[mymath],  # mymath 现在是 CMakeExtension 类的实例了
      cmdclass={"build_ext": BuildExt}  # 使用自定义的 build_ext 类
      )

```

对 `setup.py` 主要的改动为：自定义 Extension 类，并通过自定义的 build_ext 类对这些实例通过 CMake 和 make 进行编译。  

执行 `python setup.py build` 后，观察 `build` 文件夹下面的编译输出，会发现这样的文件结构：  

```shell
build
├── bdist.linux-x86_64
├── lib.linux-x86_64-3.6
│   ├── demo  # 这其实就是对 demo 文件夹的一个复制，常规的 python module 打包时都是如此操作
│   │   └── __init__.py
│   ├── mymath.cpython-36m-x86_64-linux-gnu.so  # 这是一个空文件，不用管它
│   └── mymath.so  # 这是 cmake 编译出来的文件，我们在 CMakeLists.txt 里面取消了 ‘lib’ 前缀。这样 python 就能直接 “import mymath” 了
└── temp.linux-x86_64-3.6
    └── ...

```

执行 `python setup.py install` 就能安装了。  

## 三、总结
C 项目除了在 CMake 中删除输出文件的 ‘lib’ 前缀，几乎不用做任何改动。`setup.py` 文件通过自定义的 Extension 类和 build_ext 类调用 CMake 和 Make 来编译 C 模块。  
写代码时继续用 CMake 来组织程序，发布时再使用 `setup.py` 轻轻松松地打包成 Python Package，美滋滋～  

虽然文章中用的实例非常简单，但是这种解决方案完全适用大型的 C 程序。更多的细节可以参考我的项目 [font2png](https://github.com/imhuwq/font2png)，它使用强大而复杂的 fontforge 开源库把字体文件转换为 png 图片。在这个项目中，fontforge 的模块就全部是通过 `setup.py` 来编译的。  


### 引用
1. [Building C and C++ Extensions with distutils](https://docs.python.org/3.6/extending/building.html#building-c-and-c-extensions-with-distutils)
2. [Extending setuptools extension to use CMake in setup.py?](https://stackoverflow.com/questions/42585210/extending-setuptools-extension-to-use-cmake-in-setup-py)