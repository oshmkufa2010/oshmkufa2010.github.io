---
layout: post
title: 如何用Idris写一个Type Safe的sprintf
permalink: idris-type-safe-sprintf
---
类型安全的`sprintf`是入门依赖类型(dependent type)的最好例子，如果为`sprintf`函数标记类型，会发现根据参数format的内容不同，函数接受的类型不同，比如
```idris
sprintf "%c"  -- Char -> String
sprintf "%d, %c" -- Integer -> Char -> String
```
可以看出，`sprintf format`的类型是随着`format`的值变化而变化的，换句话说，`sprintf format`的类型依赖于`format`的值，这种类型依赖于值的类型就叫做*依赖类型*

在Idris中，`sprintf`函数的类型表示为：
```idirs
sprintf : (format: String) -> FormatType format
```
这里的`FormatType`是一个返回值为类型的函数：
```idris
FormatType : String -> Type
```
函数可以返回一个类型，即first class type，是Idris对依赖类型提供支持的手段，也是Idris相比Haskell最大的优势

下面为类型安全的`sprintf`的全部实现
```idris
%default total

-- 第一步是解析format串，并将其表示为一个下面的代数数据类型
data Format =
    IntegerF Format
  | DoubleF Format
  | CharF Format
  | Lit String Format
  | End

parseFormat : String -> Format
parseFormat = parse . unpack
  where
    parse : List Char -> Format
    parse [] = End
    parse ('%' :: '%' :: xs) = Lit "%" (parse xs)
    parse ('%' :: 'd' :: xs) = IntegerF (parse xs)
    parse ('%' :: 'f' :: xs) = DoubleF (parse xs)
    parse ('%' :: 'c' :: xs) = CharF (parse xs)
    parse (x :: xs) = Lit (pack [x]) (parse xs)

-- 第二步根据上面解析出的format计算类型
FormatType' : Format -> Type
FormatType' (IntegerF fmt) = Integer -> FormatType' fmt
FormatType' (DoubleF fmt) = Double -> FormatType' fmt
FormatType' (CharF fmt) = Char -> FormatType' fmt
FormatType' (Lit _ fmt) = FormatType' fmt
FormatType' End = String

FormatType : String -> Type
FormatType = FormatType' . parseFormat

-- 第三步，实现sprintf，sprintf的返回类型为FormatType fmt
sprintf : (fmt: String) -> FormatType fmt
sprintf fmt = print (parseFormat fmt) ""
  where
    print : (fmt : Format) -> String -> FormatType' fmt
    print (IntegerF fmt) acc = \i => print fmt $ acc ++ (show i) 
    print (DoubleF fmt) acc = \d => print fmt $ acc ++ (show d)
    print (CharF fmt) acc = \c => print fmt $ acc ++ (pack [c])
    print (Lit str fmt) acc = print fmt $ acc ++ str
    print End acc = acc
```
简单测试：
```shell
:t sprintf "plus: %d + %d = %c"
sprintf "plus: %d + %d = %c" : Integer -> Integer -> Char -> String

sprintf "plus: %d + %d = %c" 1 2 '3'
"plus: 1 + 2 = 3" : String
```
