---
layout: post
title: Ruby的block
date: '2018-12-02 14:12:00'
permalink: rubys-block
---
### 初识block
```Ruby
# 内部迭代器
[1, 2, 3].each {|x| puts x }
# 用f(x) = x * 2将原集合映射到一个新集合
[1, 2, 3].map {|x| x * 2 }
# 挑选出一个集合里的所有奇数
[1, 2, 3].select {|x| x%2 != 0 }
# 动态定义一个问候方法
define_method :greet do |name|
  puts "hello, #{name}"
end
...
```
### 带block方法的定义
假设我们自己要定义一个可以接受block的方法,我们应该怎样来定义,又怎样使用传进来的block呢?
这里假设我们要给`Array`类实现一个方法`my_each`使之与`each`有相同的效果,大概有下面两种途径

1.可以直接传递给方法, 在方法内部可以通过yield来调用传入的代码块:
```Ruby
class Array
  def my_each
    i = 0
    if block_given?
      while i < self.size
        yield self[i]
        i += 1
      end
    end
    self
  end
end
[1, 2, 3].my_each {|x| puts x }
```
这里通过`block_given?`方法判断调用者是否给方法传递了block, 通过`yield`调用传入的block并将参数传递给block,整个过程都不直接与block打交道而是用`block_given?`和`yield`这种专门处理block的方式

2.block可以和Proc对象相互转换, 变成对象之后,可以像其他对象一样被传递,被返回,被复制等
```Ruby
# 代码块被转换成了Proc的对象p
p = Proc.new {|x| puts x }
# p叫做可调用对象,可以通过call方法被调用
p.call('hello')
# 语法糖, 和上面的call等效
p.('hello')
p['hello']
```
那么可以这样来实现my_each方法:
```Ruby
class Array
  def my_each(p=nil)
    i = 0
    unless p.nil?
      while i < self.size
        p.call(self[i])
        i += 1
      end
    end
    self
  end
end
p = Proc.new{|x| puts x}
[1, 2, 3].my_each(p)
```
这样做是把p完全当成了方法的普通参数, 因此必须要先把block转换成Proc对象才能传入, 这与`Array#each`相比太麻烦了,幸好Ruby提供了语法糖:
```Ruby
class Array
  def my_each(&p)
    i = 0
    unless p.nil?
      while i < self.size
        p.call(self[i])
        i += 1
      end
    end
    self
  end
end
[1, 2, 3].my_each {|x| puts x }
```
如果将参数写成`&p`这样的形式并且放在参数表的最末,那么方法接受到block之后就会自动把block转换成Proc对象并且存进p中,然后在方法内部我们就知道p是一个Proc对象,就可以按照之前的思路处理了

`&`是一个运算符, 这个运算符总是会把Proc对象转换成一个block.由于block对于Ruby程序员总是不可见的,所以不能单独使用它,但却可以把block当作参数传递给方法:
```Ruby
p = Proc.new{|x| puts x }
[1, 2, 3].my_each(&p)
```
### block的语法糖
```Ruby
# 将字符串列表里的每个元素都转换成整数
['1', '2', '3'].map{|x| x.to_i } # => [1, 2, 3]
# 求列表里所有数的和
[1, 2, 3].reduce(0) {|x, y| x + y } # => 6
```
实际上还有更简单的写法:
```Ruby
['1', '2', '3'].map(&:to_i) # => [1, 2, 3]
[1, 2, 3].reduce(0, &:+)
```
之所以能这么做,是因为有`Symbol#to_proc`这个方法的存在
`to_proc`会返回一个Proc对象:
```Ruby
to_i = :to_i.to_proc
to_i.call('1') # => 1

add = :+.to_proc
add.call(1, 2) # => 3
```
一个实现了`to_proc`方法的对象如果和`&`运算符结合,会导致对象的`to_proc`方法被调用,然后被`&`转换成block传递进方法
```Ruby
['1', '2', '3'].map(&:to_i)
# 相当于以下两步
to_i = :to_i.to_proc
['1', '2', '3'].map(&to_i)
```
想象一下`Symbol#to_proc`的实现

```Ruby
class Symbol
  def to_proc
    Proc.new {|x| x.send(self)}
  end
end
```
思考题: 如果`to_proc`要支持多个参数,比如上面的`[1, 2, 3].reduce(0, &:+)`,要怎么实现?


### 作用域
#### 1.Ruby的"作用域门"
Ruby会在三个地方关闭前一个作用域,同时打开一个新的作用域,它们分别是
- 类定义(class)
- 模块定义(module)
- 方法定义(def)
```Ruby
v1 = 1
class MyClass # 作用域门: 进入class
  v2 = 2
  local_variables # => ['v2']
  def my_method # 作用域门: 进入def
    v3 = 3
    local_variables # => ['v3']
  end
  
  local_variables # => ['v2']
end # 作用域门: 离开class

local_variables # => ['v1']
```


