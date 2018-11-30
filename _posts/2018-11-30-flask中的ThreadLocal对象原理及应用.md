---
layout: post
title: flask中的ThreadLocal对象原理及应用
tags:
  - flask
  - python
permalink: the-principle-and-aplication-of-threadlocal-object-in-flask
date: 2017-04-09 23:28:33
---

```Python
class Local(object):
    __slots__ = ('__storage__', '__ident_func__')

    def __init__(self):
        object.__setattr__(self, '__storage__', {})
        object.__setattr__(self, '__ident_func__', get_ident)

    def __iter__(self):
        return iter(self.__storage__.items())

    def __call__(self, proxy):
        """Create a proxy for a name."""
        return LocalProxy(self, proxy)

    def __release_local__(self):
        self.__storage__.pop(self.__ident_func__(), None)

    def __getattr__(self, name):
        try:
            return self.__storage__[self.__ident_func__()][name]
        except KeyError:
            raise AttributeError(name)

    def __setattr__(self, name, value):
        ident = self.__ident_func__()
        storage = self.__storage__
        try:
            storage[ident][name] = value
        except KeyError:
            storage[ident] = {name: value}

    def __delattr__(self, name):
        try:
            del self.__storage__[self.__ident_func__()][name]
        except KeyError:
            raise AttributeError(name)
```
这是flask里使用的`ThreadLocal`的定义，实际上是Werkzeug实现的。可以看出，Local对象在初始化时被绑定了两个属性：`__storage__`和`__ident_func__`，其中`__storage__`用于存储各种属性值，而`__ident_func__`被设置为函数`get_ident`，该函数可以获取当前线程或协程的id，这个id是对每个线程或者协程来说是唯一的，因此用这个id作为隔板将`__storage__`隔开（参考`__getattr__`和`__setattr__`两个方法的实现），可以保证ThreadLocal对象在被多个线程存取的情况下每个隔间的内容都不会被其他线程或协程修改，从而实现了线程安全。这样做的效果就是我们看到一个全局变量被多个线程存取，但是每个线程都察觉不到其他线程的存在。

在`ThreadLocal`的基础上，Werkzeug实现了两个类：`LocalStack`和`LocalProxy`，前者其实就是线程安全的栈，即对于每个线程来说，`LocalStack`总是只能被自己push，pop。

至于后者，每个`LocalProxy`对象在初始化时都接受一个callable的对象，这个callable对象在被调用后返回一个`ThreadLocal`对象，此后对这个`LocalProxy`对象的几乎所有操作都会被转发到`ThreadLocal`对象。
这是LocalProxy的代码片段：
```Python
class LocalProxy(object):


    def __init__(self, local, name=None):
        object.__setattr__(self, '_LocalProxy__local', local)
        object.__setattr__(self, '__name__', name)
        if callable(local) and not hasattr(local, '__release_local__'):
            # "local" is a callable that is not an instance of Local or
            # LocalManager: mark it as a wrapped function.
            object.__setattr__(self, '__wrapped__', local)

    def _get_current_object(self):

        if not hasattr(self.__local, '__release_local__'):
            return self.__local()
        try:
            return getattr(self.__local, self.__name__)
        except AttributeError:
            raise RuntimeError('no object bound to %s' % self.__name__)

    @property
    def __dict__(self):
        try:
            return self._get_current_object().__dict__
        except RuntimeError:
            raise AttributeError('__dict__')
```
可以看出对LocalProxy对象的属性读取都被转发到`_get_current_object`的调用返回上，而这个`_get_current_object`的调用实际是`self.__local`的调用，在初始化方法中，`self.__local`的被指定为传进来的callable对象(注意Python的对成员变量命名的trick)，因此外界对LocalProxy的每次操作都会引发callable对象的调用，虽然我们看到的LocalProxy对象好像是一个静态的全局对象，但对它的每次读取属性都是在实时地调用传进来的callable对象。
举个例子：
```Python
request = LocalProxy(partial(_lookup_req_object, 'request'))
```
这就是我们熟悉的flask的全局对象request，其中函数`_lookup_req_object`的作用是取当前线程/协程中`LocalStack`对象栈顶对象的某个属性，这里的`partial(_lookup_req_object, 'request')`实际返回了一个callable对象（参考偏函数的用法），这个callable对象每次被调用都会返回`LocalStack`栈顶的名为request的属性，实际就是当前要处理的request对象。与`LocalProxy`配合使用，就使得这个全局request对象总是指向当前线程/协程栈顶的request对象，也就是当前要处理的request对象。
同理，flask对于session，g，current_app等全局对象都是同样的实现方式，只不过他们存在的栈不同，指向栈帧中的对象不同。
