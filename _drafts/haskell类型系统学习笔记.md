---
layout: post
data: 2018-12-02 14:25:00
permalink: haskell-type-system-learning-note
title: Haskell类型系统学习笔记
---
1.类型 
类型即类型构造子,可以像函数一样将其他类型作为参数并返回一个类型,这样的类型叫复合类型;也可以像一个值一样,不接受任何参数而返回一个类型,这种情况可以看作是第一种情况的特殊情况
```Haskell
:k Either
Either :: * -> * -> *

:k Int
Int :: *
```

`->`也是一种类型构造子,即常见的函数类型`r -> a`:
```Haskell
:k (->)
(->) :: * -> * -> *
```
类型构造子也可以被柯里化:
```Haskell
:k Either Int
Either Int :: * -> *

:k (->) Int
(->)  Int :: * -> *
```
2.typeclass
typeclass类似于Java和golang中的interface,当一个类型实现了也给typeclass的时候,它就可以被用在任何一个接收该typeclass的场景中

