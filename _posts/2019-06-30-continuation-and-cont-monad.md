---
layout: post
permalink: continuation-and-cont-monad
title: Continuation与Cont Monad
---

## Continuation

`Continuation`是指当前计算完成之后所要发生的计算，在JavaScript中，很多函数都是异步的，就是说它们不直接返回值，而是通过传入一个回调来获取函数执行的结果：
```JavaScript
get("http://xxx.com/users", resp => console.log(resp.users))
```
这里的回调函数就是一种`Continuation`

CPS(Continuation Pass Style, 延续传递风格)就是指这样一种代码风格：函数不返回值，而是通过传递一个`Continuation`来获取函数计算的结果，例如：
```haskell
-- 普通风格，直接返回Int
add :: Int -> Int ->Int
add x y = x + y

square :: Int -> Int
square x = x * x

square $  add 1 2

-- CPS，返回(Int -> r) -> r, 通过传递一个Int -> r函数获取结果
add_cps :: Int -> Int -> ((Int -> r) -> r)
add_cps x y = \k -> k (x + y)

square_cps :: Int -> ((Int -> r) -> r)
square_cps x = \k -> k (x * x)

-- f(a, b, c) = a^2 + b^2 + c^2
f :: Int -> Int -> Int -> ((Int -> r) -> r)
f a b c = square_cps a $ \x -> square_cps b $ \y -> add_cps x y $ \z -> square_cps c $ \w -> add_cps w z
```

可以看出，CPS会使我们陷入回调地狱，为了解决这个问题，我们观察一下这几个CPS函数的返回类型

## Cont Monad
对于返回类型为`a`的函数，总是可以将它转换为返回类型为`(a -> r) -> r`的函数，这个转换过程叫做CPS变换
将返回类型`(a -> r) -> r`抽象成`Cont`类型：

```haskell
newtype Cont r a = Cont { runCont :: (a -> r) -> r }
```

`Cont r a`表示：这是一个未完成的计算，计算完成后的结果类型为`a`，但是不能直接取出这个结果，需要用一个类型为`a -> r`的函数(Continuation)来取出，取出的方式是调用`runCont`:
```haskell
cont = Cont $ add_cps 1 2
runCont cont show
```
然后我们会发现`Cont`有点像`Monad`，如果`contra`是一个类型为`Cont r a`的`Monad`，那么可以通过`contra >>= \a -> contrb`得到一个类型为`Cont r b`的`Monad`，多个CPS函数的复合就会变得简单了，因为Haskell为`Monad`类型类提供了`do notation`，可以让我们以命令式的方式组合多个`Monad`, 上面的平方和公式f可以改写为:
```haskell
f :: Int -> Int -> Int -> Cont r Int
f a b c = do
  x <- square_cont a
  y <- square_cont b
  z <- square_cont c
  add_cont x y z
```
比原始方案简单了很多

`Cont Monad`的实现如下：
```haskell
instance Monad (Cont r) where
  return a = Cont $ \k -> k a
  -- s :: Cont r a
  -- f :: a -> Cont r b
  s >>= f = Cont $ \br -> runCont s $ \a -> runCont (f a) br
```
可以证明，这个实现是满足Monad law的，这里不做赘述
