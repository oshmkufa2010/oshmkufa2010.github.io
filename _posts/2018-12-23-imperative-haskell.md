---
layout: post
title: 在Haskell里写“命令式”风格代码
permalink: imperative-haskell
date: 2018-12-23 22:23:21
---
前不久在Codewars上看见一道很有意思的[题目](https://www.codewars.com/kata/5453af58e6c920858d000823)，大致意思就是要你用Haskell这样纯粹的函数式编程语言来写如下看起来很“命令式”的代码：
```haskell
factorial :: Integer -> Integer
factorial n = def $ do
  result <- var 1
  i      <- var n
  while i (>0) $ do
    result *= i
    i      -= lit 1
  return result
```
乍一看，`def`、`while`等函数应该是可以实现的，唯独`*=`、`-=`等运算符与Haskell不可变变量的特性相去甚远，在Haskell中应该怎么模拟呢？

这个时候我想起了我多年前（误，其实就是一年前）写的一个辣鸡Lisp解释器，这个解释器的求值过程可以抽象为一个函数: `(value, env') = eval(expr, env)`
- 其中`env`是当前词法作用域所有变量的绑定，是(变量名, 值)这样的键值对
- `expr`是需要求值的表达式
- `value`是求值的结果
- `env'`是求值过程结束后新的变量绑定

 `*=`会改变`env`，具体来说就是**改变左边变量名所绑定的值**

在Haskell里，这种状态变化特别适合用`State Monad`来表达，表达式求值的过程，实际上就是`Env`这个`State`的变化过程

```haskell
type ImperativeExpr a = State Env a
```

由于无法通过变量名来标记变量，我们通过变量声明的顺序来标记变量：
```haskell
type Index = Int
newtype Variable = Variable Index

type Env = Map Index Integer

var :: Integer -> ImpreativeExpr Variable
var v = do
  env <- get
  let index = length env
  put $ insert index v env
  return $ Variable index
```

除了变量，这个题目中还有字面量这一概念，字面量很简单，它是不可变的，只需简单包住一下即可
```haskell
newtype Lit = Lit Integer

lit :: Integer -> Lit
lit = Lit
```
到目前为止，我们见到了两种表示“值”的类型，分别是可变的`Variable`和不可变的`Lit`：
- `Variable`可以出现在`*=`这种符号的左右两边，也可以当做`while`的条件变量
- `Lit`可以出现在`*=`这种符号的右边，也可当做`while`的条件变量，但是它不能出现在`*=`这种符号的左边，因为它是不可变的

综上，`Variable`和`Lit`是两种不同的类型，但是它们有一些共同点，就是它们都可以被当做“值”来使用，我们用一个`type class`来表现出这种共同点：
```haskell
class Value v where
  evalValue :: v -> Env -> Integer

instance Value Variable where
  evalValue (Variable index) env = env ! index

instance Value Lit where
  evalValue (Lit value) _ = value
```
试着实现一下`*=`：
```haskell
(*=) :: Value v => Variable -> v -> ImpreativeExpr ()
(Variable index) *= b = do
  env <- get
  let
    val1 = env ! index
    val2 = evalValue b env
    env' = insert index (op val1 val2) env
    in put env'
```
`while`的实现也就比较简单了，简单来说就是当不满足终止条件时就不断改变`Env`：
```haskell
while :: Value v => v -> (Integer -> Bool) -> ImpreativeExpr () -> ImpreativeExpr () 
while r f act = do
  env <- get
  let value = evalValue r env
  when (f value) $ do
    let (_, env') = runState act env
    put env'
    while r f act
```
`def`实际上就是对`ImperativeExpr`进行求值：
```haskell
def :: Value v => ImpreativeExpr v -> Integer
def m = let (v, env) = runState m empty in evalValue v env
```
完整代码见[这里](https://raw.githubusercontent.com/oshmkufa2010/codewars-haskell/master/src/Imperative.hs)
