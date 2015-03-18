# Templates syntax

## 基于 Scala 的类型安全的模板引擎

Play 自带了一个基于 Scala 的模板引擎，叫 [Twirl](https://github.com/playframework/twirl)，它的设计是受到了 ASP.NET Razor 的启发。它：

* **简洁、富于表现力且灵活**：它最小化了一个文件中的字符和按键数，使你的工作流快速、连续。与大部分模板语法不同，你无需在 HTML 内显式地标注服务端的代码块，解析器能从你的代码中自动推断出来。这使得模板语法简洁紧凑而富有表现力，写起来干净、快速而且好玩。
* **容易学习**：只需要学习少量概念，即可使你高效多产。你只需要使用简单的 Scala 概念及你已有的 HTML 技巧即可。
* **不是一门新语言**：我们有意识地选择不去创造一门新语言，而是想让 Scala 开发者能使用他们现有的 Scala 语言技巧，并提供一种使 HTML 构建过程更赞的模板标记语法。
* **可在任意文本编辑器中编辑**：它不需要特殊的开发工具，你可以在任意的纯文本编辑器中高效使用它。

模板会被编译，因此如果有错误，你可以在浏览器中直接看到：

![templatesError](https://www.playframework.com/documentation/2.3.x/resources/manual/scalaGuide/main/templates/images/templatesError.png)

## 概述

Play 的 Scala 模板是一个包含小段 Scala 代码块的文本文件。模板可以生成任意基于文本的格式，如 HTML，XML 或 CSV。

模板系统的设计使 HTML 使用者用起来会很舒服，前端开发者很容易就可以上手。

模板会被编译成标准 Scala 函数（使用一种简单的命名约定）。如果你创建一个 `views/Application/index.scala.html` 模板文件，它会生成一个 `views.html.Application.index` 类，这个类有一个 `apply()` 方法。

例如，下面是一个简单的模板：

```html
@(customer: Customer, orders: List[Order])

<h1>Welcome @customer.name!</h1>

<ul>
@for(order <- orders) {
  <li>@order.title</li>
}
</ul>
```

你可以在任意 Scala 代码中调用上述代码，就像你调用一个类的方法一样：

```scala
val content = views.html.Application.index(c, o)
```

## 神奇的 '@' 字符

Scala 模板只使用一个特殊字符：@。每次遇到这个字符，它就表明一条动态语句的开始。你无需显式地标注代码块的结束，动态语句在哪结束会被自动地推断出来：

```html
Hello @customer.name!
       ^^^^^^^^^^^^^
       Dynamic code
```

模板引擎是通过分析你的代码来推断代码块在哪结束，因此它只能支持简单的语句。如果你要插入一条复杂语句，用括号来显式地标注它：

```html
Hello @(customer.firstName + customer.lastName)!
       ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
                    Dynamic Code
```

如果你想插入一个包含多条语句的块，可以使用大括号：

```html
Hello @{val name = customer.firstName + customer.lastName; name}!
       ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
                             Dynamic Code
```

由于 `@` 是特殊字符，当你想使用 `@` 这个字符时，就需要转义它（即 `@@`）：

```html
My email is bob@@example.com
```

## 模板参数

模板就像函数，因此它也需要参数。参数必须被声明在模板文件的顶部：

```html
@(customer: Customer, orders: List[Order])
```

你也可以为参数提供默认值：

```html
@(title: String = "Home")
```

甚至可以提供几个参数组：

```html
@(title: String)(body: Html)
```

## 遍历

你可以使用关键词 `for` 进行遍历：

```html
<ul>
@for(p <- products) {
  <li>@p.name ($@p.price)</li>
}
</ul>
```

> 注意：确保 `{` 与 `for` 在同一行，用于指示表达式还没结束，会延续到下一行。

## if 块

if 块的使用并没有什么特别的，和 Scala 中的 `if` 用法相似：

```html
@if(items.isEmpty) {
  <h1>Nothing to display</h1>
} else {
  <h1>@items.size items!</h1>
}
```

## 声明可复用的代码块

你可以创建可复用的代码块：

```html
@display(product: Product) = {
  @product.name ($@product.price)
}

<ul>
@for(product <- products) {
  @display(product)
}
</ul>
```

注意，你也可以声明可复用的纯代码块：

```html
@title(text: String) = @{
  text.split(' ').map(_.capitalize).mkString(" ")
}

<h1>@title("hello world")</h1>
```

> 注意：在模板里声明代码块有时候是有用的，但请记住，模板里不宜放复杂的逻辑。这部分代码应该放在外部的 Scala 类中（如果你愿意，你也可以把它放在 `views/` 包下）。

按照惯例，一个可复用的代码块命名如果以 **implicit** 开头，它就会被标记为 `implicit`：

```html
@implicitFieldConstructor = @{ MyFieldConstructor() }
```

## 声明可复用的值

你可以使用 `defining` helper 方法来定义一个局部值：

```html
@defining(user.firstName + " " + user.lastName) { fullName =>
  <div>Hello @fullName</div>
}
```

## 导入声明

你可以在模板（或子模板）的开始处导入任何你想要的东西：

```html
@(customer: Customer, orders: List[Order])

@import utils._

...
```

为确保绝对的解析路径，可以在 import 语句前加上 **root** 前缀：

```html
@import _root_.company.product.core._
```

如果在所有的模板中，你都要导入一个相同的东西，那么你可以将它声明在 `build.sbt`：

```scala
TwirlKeys.templateImports += "org.abc.backend._"
```

## 注释

使用 `@* *@`，你就可以在模板中像服务端那样写块注释：

```html
@*********************
* This is a comment *
*********************@
```

你可以在模板最开始处写上注释，这样可以把你的模板文档化到 Scala API doc 中：

```html
@*************************************
 * Home page.                        *
 *                                   *
 * @param msg The message to display *
 *************************************@
@(msg: String)

<h1>@msg</h1>
```

## 转义

默认情况下，动态内容会根据模板的类型（如 HTML 或 XML）规则进行转义。如果你想输出原始内容，则需要将它包在模板的内容类型里。

例如，要输出原始的 HTML 内容：

```html
<p>
  @Html(article.content)
</p>
```
