# JSON Macro Inception

> 注意，这一节的内容最早由 Pascal Voitot 发表在 [mandubian.com](http://mandubian.com/2012/11/11/JSON-inception/) 上。（文章太旧，请带着批判的眼光去读。）
>
> 这个特性还处于实验中，因为 Scala 宏在 Scala 2.10.0 中仍是实验性的。如果你不想使用 Scala 中的实验性特性，请手写 Reads/Writes/Format，同样可以达到一样的效果。

## 写样例类（case class）的默认 Reads/Writes/Format 是非常无聊的

还记得你是如何为一个样例类写 `Reads[T]` 的吗：

```scala
import play.api.libs.json._
import play.api.libs.functional.syntax._

case class Person(name: String, age: Int, lovesChocolate: Boolean)

implicit val personReads = (
  (__ \ 'name).read[String] and
  (__ \ 'age).read[Int] and
  (__ \ 'lovesChocolate).read[Boolean]
)(Person)
```

你为这个样例类写了 5 行代码，你知道吗，许多人认为为他们的类写一个 `Reads[TheirClass]` 是非常不 cool 的，因为像 Java 的 JSON 框架，如 Jackson 或 Gson，会为你做这些事情，而你根本不需要写这些多余的代码。

我们会这么说 Play2.1 的 JSON 序列化与反序列化：

* 完全类型安全的
* 完全编译的
* 在运行时使用内省/反射机制，无需任务操作

但对于一些人来说，以上的好处并无法抹平额外代码带来的麻烦。

我们相信这是一个非常好的方法，因此坚持并提出：

* JSON 简化语法
* JSON 组合子
* JSON 变换器

虽然被加强了，但仍然没改变要写额外代码的事实。

## 做一个极简主义者

鉴于我们是完美主义者，我们提出了一种新的方法来达到同样的效果：

```scala
import play.api.libs.json._
import play.api.libs.functional.syntax._

case class Person(name: String, age: Int, lovesChocolate: Boolean)

implicit val personReads = Json.reads[Person]
```

只需要一行！你马上可能会问：

> 它有使用运行时字节码增强吗？ -> 没有
> 它有使用运行时自省机制吗？ -> 没有
> 它会打破类型安全吗？ -> 不会

所以呢？

> 在创造了 JSON coast-to-coast 设计一词后，让我们把它叫做：JSON Inception。

# JSON Inception

## 代码等价性

正如之前所解释的：

```scala
import play.api.libs.json._
// please note we don't import functional.syntax._ as it is managed by the macro itself

implicit val personReads = Json.reads[Person]

// IS STRICTLY EQUIVALENT TO writing

implicit val personReads = (
  (__ \ 'name).read[String] and
  (__ \ 'age).read[Int] and
  (__ \ 'lovesChocolate).read[Boolean]
)(Person)
```

## Inception 等式

下面是描述 inception 概念的等式：

```scala
(Case Class INSPECTION) + (Code INJECTION) + (COMPILE Time) = INCEPTION
```

### 样例类检查

也许你自己就可以推断出来，为了达到前面说的代码等价性，我们需要：

* 检查 `Person` 样例类
* 提取 `name`，`age` 和 `lovesChocolate` 3 个字段以及它们的类型
* 隐式解析类型类
* 找到 `Person.apply`

### 注入？

我要立马阻止你，并不是你想的那样

> 代码注入并不是依赖注入。不是 Spring 那套东西，没有 IOC，也没有 DI。

我是故意使用这个词的，因为我知道说到「注入」，大家马上会联想到 IOC 和 Spring。但我还是想用这个词的真实涵义还重新建立大家对它的认识。这里，代码注入指的就是在编译期，我们把代码注入到已编译的 Scala AST 中（Abstract Syntax Tree，抽象语法树）。

因此，`Json.reads[Person]` 会被编译并用下面内容替换到编译后的 AST 中：

```scala
(
  (__ \ 'name).read[String] and
  (__ \ 'age).read[Int] and
  (__ \ 'lovesChocolate).read[Boolean]
)(Person)
```

不多也不少。

### 编译期

没错，一切都是在编译期执行的。并没有运行时字节码增强，也没有运行时自省。

> 由于一切都是在编译期解析的，因此如果没有导入各个字段类型所需的隐式转换，就会报编译错误。

## Json inception 是 Scala 2.10 中的宏

我们需要启用 Scala 的一个特性来支持 Json inception：

* 编译期代码增强
* 编译期类/隐式检查
* 编译期代码注入

以上这些可以由 Scala 2.10 中的一个新的实验性特性来提供：[Scala 宏](http://scalamacros.org/)。

Scala 宏是一个拥有具大潜力的新特性（仍是实验性的）。有了它，可以：

* 在编译期使用 Scala 提供的反射 API 进行代码自省
* 在当前的编译上下文中，访问所有的导入和隐式内容
* 创造新的代码表达式，产生编译错误（如果有的话）并将它们注入编译链

请注意：

* 我们使用 Scala 宏，因为它正好满足我们的需求
* 我们使用 Scala 宏作为推动者，而不是目的本身
* 宏是一个帮助你生成代码的 helper，这样你就不用自己写这部分代码
* 它并没有增加或隐藏代码
* 我们遵循的是 no-surprise 原则

你可能发现了，写宏并不是一个简单的过程，因为你写的宏是在编译期执行的。

```scala
So you write macro code
  that is compiled and executed
  in a runtime that manipulates your code
     to be compiled and executed
     in a future runtime…
```

这也是为什么我把它叫做 Inception。

因此想完全按照你想做的来，来需要一些练习。提供的 API 也相当复杂，文档也不齐全。因此，当你开始使用宏时，你需要有坚持不懈的精神。

## Writes[T] & Format[T]

> 请注意，JSON inception 只能用于含有 `unapply/apply` 方法的结构。

自然地，你可将它用于 `Writes[T]` 和 `Format[T]`。

## Writes[T]

```scala
import play.api.libs.json._

implicit val personWrites = Json.writes[Person]
```

## Format[T]

```scala
import play.api.libs.json._

implicit val personWrites = Json.format[Person]
```

## 特殊模式

* 你可以在伴生对象（companion object）中定义你的 Reads/Writes

这样当你操作你的类的一个实例，隐式的 Reads/Writes 就会被自动推断出来。

```scala
import play.api.libs.json._

case class Person(name: String, age: Int)

object Person{
  implicit val personFmt = Json.format[Person]
}
```

* 你现在可以为单字段样例类定义 Reads/Writes

```scala
import play.api.libs.json._

case class Person(names: List[String])

object Person{
  implicit val personFmt = Json.format[Person]
}
```

## 已知限制

* 不要覆盖伴生对象中的 apply 函数，因为这样一来宏就有几个 apply 函数，而不知道选择哪个。
* 只有当 apply 和 unapply 方法有相应的输入输出类型时，才能用 Json 宏。这对样例类来说是很自然的，但如果你想要特性（trait）也能达到一样的效果，你就需要实现与样例类中相同的 apply/unapply 方法。
* Json 宏可用于以下类型：Option/Seq/List/Set/Map[String,_]，如果想用于其它类型，你需要进行测试，如果不行，请使用传统方式手写 Reads/Writes。
