---
layout: post_with_mathjax
title: Yoneda Lemma
permalink: yoneda-lemma
author: cailin
---

## Hask 范畴

Haskell 里的所有范畴都是 Hask 范畴，这是一个 **集合范畴**(Category Of Set)，集合范畴是什么意思呢， 就是说这个范畴的 Obj 都是集合，而 Obj 之间的态射 (箭头) 就是函数，在 Haskell 里，每种类型都可以看做是值的集合，例如 `Bool` 类型是 True 和 False 组成的集合, `Char` 类型是所有字符的集合, 而 `Integer` 是所有整数的集合

我们在 Haskell 里看到的 fmap 都是这种签名:
```Haskell
fmap :: (a -> b) -> (F a -> F b)
```
注意后面的 `(F a -> F b)`，这实际上是经过函子 F 映射后的范畴里的态射，依然是函数

## Hom 范畴与 Hom 函子

定义如下的类型

```Haskell
newtype Hom a r = Hom {getHom :: a -> r }
```

其实它就是 `(->) a r`

所有 `Hom a r` 组成的范畴是 Hask 的子范畴，称为 Hom 范畴
从Hask到Hom范畴的函子就是Hom函子，通过固定a或者r都可以从Hom集合产生Hom函子，而这种选择使得Hom函子分为协变Hom函子和逆变Hom函子

协变Hom函子, 固定a, 记作Hom(a, -)：

```Haskell
instance Functor (Hom a) where
  -- f :: x -> y
  -- g :: a -> x
  -- fmap :: (x -> y) -> (Hom a x -> Hom a y)
  fmap f (Hom g) = Hom (g . f)
```
逆变Hom函子，固定r，记作Hom(-, r):
```Haskell
newtype Op r x = Op { getOp :: x -> r }

instance Functor (Op r) where
  fmap :: (r -> s) -> Op r -> Op s
  fmap f (Op xr) = Op (f . xr)
```


## Fct(Hask, Hom(Hask, -)) 范畴
**Hask 范畴** 到 **Hom(Hask, Hask)组成的范畴** 之间的 **所有函子** 组成的范畴，是一个函子范畴，这个范畴的 Obj 是函子，而态射是函子间的自然变换

这不是一个 Hask 范畴，因此 Functor 不能用 Haskell 提供的 Functor Typeclass 来描述
对于 C(Hask 范畴) 中的态射 `a -> b`, 在函子范畴里应该被映射成一个自然变换, 因为自然变换正是函子范畴里 Obj 之间的态射
```Haskell
from :: (a -> b) -> (forall x. (b -> x) -> (a -> x))
from ab = \bx -> bx . ab
```
同时，对于函子范畴里的态射，也就是函子之间的自然变换，我们其实也可以将它们映射到 Hask 范畴：
```Haskell
to :: (forall x. (b -> x) -> (a -> x)) -> (a -> b)
to nat = nat id
```
因此，Hask 范畴和 Fct(Hask, Hom(Hask, -)) 范畴是同构的
