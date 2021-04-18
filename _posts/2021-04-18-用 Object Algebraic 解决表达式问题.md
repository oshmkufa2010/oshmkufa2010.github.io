---
layout: post
permalink: object-algebra
title: 用 Object Algebra 解决表达式问题
---

## 表达式问题

表达式问题（Expression Problem）通常用来衡量不同编程语言和编程范式的优缺点，由 Philip wadler 提出。

我们的目标是在不重新编译代码并保持类型安全的前提下扩展数据类型或者添加新的操作，举一个具体的例子，假设我们有如下的数据类型（本文代码全部用 typescript 实现）：

```typescript
interface Expr {
  eval(): number
}

class Lit implements Expr {
  private n: number
  constructor(n: number) {
    this.n = n
  }

  eval() {
    return this.n
  }
}

class Add implements Expr {
  private left: Expr
  private right: Expr
  constructor(left: Expr, right: Expr) {
    this.left = left
    this.right = right
  }

  eval() {
    return l.eval() + r.eval()
  }
}

```

假设上面的代码来自于第三方库，于是我们不能修改上面的代码，只能看到暴露出来的内容（比如我们只知道 `Lit` 实现了 `Expr`，因此我们可以在`Lit`的实例上调用 `eval`得到一个 `number`），现在，我们需要扩展这个库：

	- 添加新的数据类型，比如添加乘法表达式
	- 添加新的操作，比如 pretty print 语法树

得益于 OOP 的天然特性，添加新的数据类型可以轻松做到：

```typescript
class Mult implements Expr {
  private left: Expr
  private right: Expr
  constructor(left: Expr, right: Expr) {
    this.left = left
    this.right = right
  }

  eval() {
    return l.eval() * r.eval()
  }
}
```

但是添加新的操作就很困难了，我们必须修改`Expr`接口以及`Lit`和`Add`两个类，这可能需要重新编译或者根本不能做到。

一种可能的解决方案是扩展 `Expr` 接口然后定义 `Lit` 和 `Add` 的子类型，在子类型里添加新的操作，但是使用了 `Lit` 和 `Add` 的旧代码却无法使用新添加的操作，除非一一修改，这显然是一件麻烦的事情。

## 访问者模式

之所以无法添加新的操作是因为OOP将数据类型和方法（操作）放在了一起，于是我们尝试将操作与数据类型分离，访问者模式（visitor pattern）正是解决此类问题的方法，实现访问模式需要定义访问者（visitor），并且需要给每种数据类型实现 `accept`方法。

#### 1.定义访问者

在访问者模式中，访问者有两种形式：

- 内部访问者（internal visitor），类似于内部迭代器，控制流由数据类型本身掌管，访问者只需考虑如何组合子结构产生的结果。
- 外部访问者（external visitor），类似于外部迭代器，控制流由访问者自身掌管。

我们选用内部访问者，原因后面会说道。

访问者定义如下，类型参数 `A` 表示返回的类型

```typescript
interface Visitor<A> {
  lit(n: number): A
  add(left: A, right: A): A
}
```

于是我们可以轻松地实现多个访问者（操作），如果我们要对表达式求值，那么：

```typescript
class EvalVisitor implements Visitor<number> {
  lit(n: number): number {
    return n
  }
  add(left: number, right: number): number {
    return left + right
  }
}
```

如果要打印语法树：

```typescript
class PrintVisitor implements Visitor<string> {
  lit(n: number) {
    return n.toString()
  }
  add(left: string, right: string): string {
    return left + " + " + right
  }
}
```



#### 2.为数据类型实现 `accept` 方法

```typescript
interface Acceptable {
  accept<A>(vis: Visitor<A>): A
}

class Lit implements Expr implements Acceptable {
  private n: number
  constructor(n: number) {
    this.n = n
  }

  eval() {
    return this.n
  }

  accept<A>(vis: Visitor<A>): A {
    return vis.lit(this.n)
  }
}

class Add implements Expr implements Acceptable {
  private left: Expr
  private right: Expr
  constructor(left: Expr, right: Expr) {
    this.left = left
    this.right = right
  }

  eval() {
    return left.eval() + right.eval()
  }

  accept<A>(vis: Visitor<A>): A {
    // 我们的visitor如何被调用是在这里确定的，换句话说，是数据类型决定了控制流
    return vis.add(this.left.accept(vis), this.right.accept(vis))
  }
}
```

很明显，访问者模式有一些缺点：

- 要求 `Lit` 和 `Add`实现 `accept`方法，第三方库不一定会按这种模式出牌。
- 添加新的数据类型需要修改所有的访问者，这恰好和之前的情况相反——添加新的操作变得容易，但是添加新的数据类型变得困难了，所以某些讲设计模式的文章会说访问者模式适用于数据类型比较稳定的场景。



## Object Algebra

访问者模式总结起来就是：`Expr`通过`accept`方法决定访问者如何被调用，访问者决定如何产生数据。我们之所以让`Expr`决定控制流是因为我们假设我们的程序先构造出了`Expr`然后对其进行各种操作，例如：

```typescript
const expr: Expr = new Add(new Lit(1), new Lit(2))
const s: string = expr.accept(printVisitor)
console.log(s)
```

但实际上不构造`Expr`也可以达到相同的目的：

```typescript
const vis: PrintVisitor = new PrintVisitor()
const s: string = vis.add(vis.lit(1), vis.lit(2))
console.log(s)
```

更进一步，我们可以提取出`vis`让程序更加抽象：

```typescript
function expr<A>(vis: Visitor<A>): A {
  return vis.add(vis.lit(1), vis.lit(2))
}
```

传入不同的`vis`，我们便可以对`expr`进行不同的操作，比如传入`PrintVisitor`就可以打印出`expr`的结构，如果传入`EvalVisitor`就可以对表达式进行求值。

此外，如果我们需要之前的`Expr`，我们可以实现这样一个访问者：

```typescript
class ExprFactory implements Visitor<Expr> {
  lit(n: number): Expr {
    return new Lit(n)
  }
	add(left: Expr, right: Expr): Expr {
    return new Add(left, right)
  }
}
```

给`expr`传入`ExprFactory`会得到一个`Expr`，与`new Add(new Lit(1), new Lit(2))`结果是一样的。

通过这种方式编写的程序不依赖特定的数据类型，只依赖`Visitor`，于是我们不用担心扩展数据类型的问题了。

更学术化一点，这样的`Visitor`叫做`Object Algebra Interface`，而实现了这种接口的类（例如`PrintVisitor`）叫做`Object Algebra`。为了与访问者模式区分，我们将`Visitor`改名为`IExprAlg`，同时也将其他访问者实现改名：

```typescript
interface IExprAlg<A> {
  lit(n: number): A
  add(left: A, right: A): A
}

class EvalAlg implements IExprAlg<number> {
  lit(n: number): number {
    return n
  }
  add(left: number, right: number): number {
    return left + right
  }
}
```

## Object Algebra 上的双向扩展

再来看看 Object Algebra 上的扩展问题：

### 添加新的操作

要添加新的操作，我们只需实现新的`Object Algebra`，例如打印语法树：

```typescript
class PrintAlg implements IExprAlg<string> {
  lit(n: number): {
    return n.toString()
  }
  add(left: string, right: string) {
    return left + " + " + right
  }
}
```

只需将新实现的`Object Algebra`传入之前泛化过的程序（如`expr`），而不需要其他额外工作。

### 添加新的数据类型

我们不用修改原本`Lit`和`Add`以及`Expr`的代码，要添加新的数据类型，只需遵从`Expr`原本的约束，如果你不需要`Expr`类型的数据，甚至可以不做这一步：

```typescript
class Mult implements Expr {
  private left: Expr
  private right: Expr
  constructor(left: Expr, right: Expr) {
    this.left = left
    this.right = right
  }

  eval() {
    return this.left.eval() * this.right.eval()
  }
}
```

与此同时，我们也要扩展 `ExprAlg`，它是一个接口，可以轻松扩展而不用修改原有的`ExprAlg`，因此也不会影响所有使用`ExprAlg`的代码：

```typescript
interface IExprMultAlg<A> extends IExprAlg<A> {
  mult(left: A, right: A): A
}
```

由于添加了新的数据类型，原有的操作也要扩展，比如，我们需要让之前打印语法树的操作能处理乘法：

```typescript
class PrintMultAlg extends PrintAlg implements IExprMultAlg<string> {
  mult(left: string, right: string): string {
    return left + " * " + right
  }
}
```

注意以上所有步骤中，我们只实现了必要的针对新数据类型（mult）的操作且添加的都是新的类和接口，没有冗余代码，也不会影响已有的代码，同时也是类型安全的。

因此，可以说 Object Algebra 完美地解决了表达式问题。

## 参考资料

[1]: https://homepages.inf.ed.ac.uk/wadler/papers/expression/expression.txt
[2]: https://www.cs.utexas.edu/~wcook/Drafts/2012/ecoop2012.pdf
[3]: https://en.wikipedia.org/wiki/Expression_problem
[4]: https://zhuanlan.zhihu.com/p/53810286
