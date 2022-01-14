---
title: python装饰器
date: 2018-06-19 10:03:04
desc: 理解 python 装饰器
tags: python decorator
---

python 的装饰器 可以在不修改原有方法的情况下, 扩展方法, 非常有用, 在很多的框架框架中都在大量使用。 来一起学习它。

<!--more-->

一个最简单的方法

```python

def hello(name):
    return "hello " + name

hello("luowen") # output: hello luowen

```

再申明一个内嵌方法

```python
def wrap(name):
    def hello(name):
        return "hello"
    return hello() + name

wrap("luowen") # output: hello luowen

```
python的方法可以作为参数传递, 变形下

```python
def hello(name):
    return "hello " + name

def call_func(func):
    name = "luowen"
    return func(name)

call_func(hello) # 将hello方法作为参数传递 output: hello luowen
```

python 返回值也可以是方法, 上面栗子再变形
```python
def hello(name):
    return "hello " + name

def call_func(func):
    name = "luowen"
    return func

func = call_func(hello)
func() # 将hello方法作为参数传递 output: hello luowen, 返回的方法在一个闭包中
```

组合试试, 这就是一个装饰方法了

```python
def hello(name):
    return "hello " + name

def hello_decorator(func):
    def wrapper(name):
        return "*****{0}*****".format(func(name))
    return wrapper

wrap = hello_decorator(hello)
print(wrap("luowen")) # output  *****hello luowen******
```

上面栗子我们是直接调用编写的装饰方法, 下面直接使用python提供了语法糖调用

```python

@hello_decorator
def hello(name):
    return "hello " + name

def hello_decorator(func):
    def wrapper(name):
        return "*****{0}*****".format(func(name))
    return wrapper

hello("luowen") # output: *****hello luowen*****

```

装饰方法也可以传递参数

```python
def tag(ele):
    def tag_decorator(func):
        def wrapper(name):
            return "<{0}> {1} <{0}>".format(ele, func(name))
        return wrapper 
    return tag_decorator


@tag("p")
def hello(name):
    return "hello" + name

hello("luowen") # output: <p> hello luowen </p>
```

此处有个问题, `__name__, __doc__, __module__` .. 都被wapper覆盖了

```python

print(hello.__name__) # output: wrapper

```

解决

```python

from functools import wraps

def tag(ele):
    def tag_decorator(func):
        @wraps(func)
        def wrapper(name):
            return "<{0}> {1} <{0}>".format(ele, func(name))
        return wrapper 
    return tag_decorator


@tag("p")
def hello(name):
    return "hello" + name

hello("luowen") # output: <p> hello luowen </p>

```
