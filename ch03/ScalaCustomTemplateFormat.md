# Custom format addition

内建的模板引擎支持常见的模板格式（HTML，XML 等），但如果有需要，你可以非常容易地增加对自定义格式的支持。下面总结了添加自定义格式支持的步骤。

## 模板化概述

模板引擎通过将模板中的静态和动态内容组合到一起来构建最终结果。考虑如下模板：

```html
foo @bar baz
```

它由两个静态部分（`foo` 和 `baz`）和一个动态部分（`bar`）组成。模板引擎将这些拼接起来构建最终结果。事实上，为了避免跨站点脚本攻击，`bar` 的值在拼接到其它结果之前可以先进行转义。转义过程与具体的格式相关。例如，对于 HTML 格式，"<" 转义的结果是 "&lt;"。

模板引擎是如何知道一个模板的格式的呢？它会检查这个文件的扩展名，例如该文件的扩展名是 `.scala.html`，模板引擎就会把该文件与 HTML 格式关联起来。

最后，你的模板文件需要作为 HTTP 响应的主体，因此你还需要定义如何用模板渲染的结果来构建 Play 返回给客户端的结果。

综上所述，如果想让 Play 支持你自己定义的模板格式，需要做以下几件事：

* 为该格式实现文本整合过程
* 将你定义的格式与某一文件扩展名关联起来
* 最后，你需要告诉 Play 如何把模板渲染的结果作为 HTTP 响应体发送出去

## 实现一个模板格式

实现 `play.twirl.api.Format[A]` trait，该 trait 包含两个方法：`raw(text: String): A` 和 `escape(text: String): A`，分别用于整合模板的静态和动态部分。

类型参数 `A` 定义了模板渲染后的结果类型，例如，对于 HTML 模板，该类型是 `HTML`。该类型必须是 `play.twirl.api.Appendable[A]` trait 的子类，该 trait 定义了如何把各部分拼接在一起。

方便起见，Play 提供了 `play.twirl.api.BufferedContent[A]` 抽象类，它实现了 `play.twirl.api.Appendable[A]` trait，使用 `StringBuilder` 来构建它的结果。它还实现了 `play.twirl.api.Content` trait，因此 Play 知道如何将它作为 HTTP 响应体进行序列化（详情参见本节最后一部分）。

简而言之，你需要写两个类：一个定义结果（实现 `play.twirl.api.Appendable[A]`），另一个定义文本整合过程（实现 `play.twirl.api.Format[A]`）。例如，下面是对 HTML 格式的定义：

```scala
// The `Html` result type. We extend `BufferedContent[Html]` rather than just `Appendable[Html]` so
// Play knows how to make an HTTP result from a `Html` value
class Html(buffer: StringBuilder) extends BufferedContent[Html](buffer) {
  val contentType = MimeTypes.HTML
}

object HtmlFormat extends Format[Html] {
  def raw(text: String): Html = …
  def escape(text: String): Html = …
}
```

## 将该格式与某一文件扩展名关联起来

在编译整个项目源码之前，模板文件会在构建过程中被编译成 `.scala` 文件。`TwirlKeys.templateFormats` 是一个类型为 `Map[String, String]` 的 sbt 配置项，定义了文件扩展名与模板格式之间的映射。例如，如果 Play 不支持 HTML 格式，你就需要在构建文件中写入下面的配置来关联 `.scala.html` 文件和 `play.twirl.api.HtmlFormat` 格式：

```scala
TwirlKeys.templateFormats += ("html" -> "my.HtmlFormat.instance")
```

注意，箭头右边包含了一个类型为 `play.twirl.api.Format[_]` 的值的全限定名。

## 用模板渲染结果构建 HTTP 结果

类型 `A` 的值如果可以隐式转换为类型 `play.api.http.Writeable[A]` 的值，则 Play 可以将 `A` 的任意值写入 HTTP 响应体中。因此你所需要做的就是为你模板的结果类型 `A` 定义一个隐式转换的值。例如，以下展示了如何为 HTTP 定义一个这样的值：

```scala
implicit def writableHttp(implicit codec: Codec): Writeable[Http] =
  Writeable[Http](result => codec.encode(result.body), Some(ContentTypes.HTTP))
```

> 注意：如果你的模板结果类型扩展了 `play.twirl.api.BufferedContent`，你就只需要定义一个隐式的 `play.api.http.ContentTypeOf` 值：
> `implicit def contentTypeHttp(implicit codec: Codec): ContentTypeOf[Http] = ContentTypeOf[Http](Some(ContentTypes.HTTP))`
