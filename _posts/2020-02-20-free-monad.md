---
layout: post
permalink: free-monad
title: Free Monad
---

我们知道Monad是自函子范畴上的Monoid，而Free Monod就是自函子组成的List，它存储了这些自函子的原始结构，可以通过对比List的方式来理解一下Free Monad。

## List和foldMap
如果我们想在List上做一些操作，会使用foldMap：

```haskell
data List a = Nil | Cons a (List a)

foldMap :: Monoid m => (a -> m) -> （List a -> m）
foldMap f Nil = mempty
foldMap f (Cons x xs) = f x `mappend` foldMap f xs
```

通过实现各种Monoid，foldMap可以做各种List上的操作：

```haskell
-- Int求和
instance Monoid Int where
  mempty = 0
  mappend = (+)

sum' :: List Int -> Int
sum' = foldMap id

-- 取列表中最大值
instance Ord a => Monoid (Maybe a) where
  mempty = Nothing
  Nothing `mappend` y = y
  x `mappend` Nothing = x
  (Just x) `mappend` (Just y) = Just $ min x y

max' :: Ord a => List a -> Maybe a
max' = foldMap Just

-- 实现foldr
instance Monoid ((->) a a) where
  mempty = id
  f `mempty` g = f . g

foldr' :: (a -> b -> b) -> b -> List a -> b
foldr' abb b l = foldMap abb l b
```

可以看出foldMap决定了List是怎样进行遍历的(其他结构比如Tree也可以实现foldMap，foldMap反应了结构的遍历性质，详细请参考Foldable这个type class)，而各种不同的Monoid实现决定了遍历过程中元素是怎样结合在一起的。

## Free Monad和foldFree
Free Monad和List一样，也有类似于foldMap这样的函数：

```haskell
data Free f a = Pure a
              | Roll (f (Free f a))
              deriving (Functor)

instance Functor f => Monad (Free f) where
  return = Pure 
  (Pure a) >>= f = f a 
  (Roll ffa) >>= f = Roll (fmap (>>= f) ffa)
  -- 范畴论上的monad定义
  unit = return
  join = (>>= id)

-- 注意自函子范畴上的态射是自然变换
infixr 0 :~>
type f :~> g = forall x. f x -> g x
  
foldFree :: (Functor f, Monad m) => (f :~> m) -> (Free f :~> m)
foldFree _ (Pure a) = unit a
foldFree fm (Roll f) = join (fmap (foldFree fm) (fm f))
```

这里的foldFree和foldMap类似，只是将Hask范畴上的态射(->)换成了自函子范畴上的态射(:~>)，可以通过实现各种Monad，来决定函子之间的结合方式，从而对同样的Free Monad进行不同的“解释”。

## 案例：交互式DSL
我们来试着用Free Monad来实现一个交互式读取和输出的DSL:

```haskell
data InteractionF interaction = Say String (() -> interaction)
                              | Ask (String -> interaction)

instance Functor InteractionF where
  fmap f (Say str cont) = Say str (f . cont)
  fmap f (Ask cont) = Ask (f . cont)

type Interaction a = Free InteractionF a

say :: String -> Interaction ()
say str = Roll $ Say str $ \_ -> Pure ()

ask :: Interaction String
ask = Roll $ Ask $ \str -> Pure str

dsl :: Interaction ()
dsl = do
  say "you name?"
  name <- ask
  say "your name is" ++ name

-- 用IO Monad解释
interpreterByIO :: Interaction a :~> IO a
interpreterByIO (Say msg cont) = putStrLn msg >> return (cont ())
interpreterByIO (Ask cont) = fmap cont getLine

interpreter1 = foldFree interpreterByIO dsl

-- 用State Monad解释
interpreterByState :: InteractionF :~> State ([String], [String])
interpreterByState (Say msg cont) = state $ \(input, output) -> (cont (), (input, output ++ [msg]))
interpreterByState (Ask cont) = state $ \case 
  ((i : is), o) -> (cont i, (is, o))
  ([], o) -> (cont "", ([], o))

interpreter2 = foldFree interpreterByState dsl
```
