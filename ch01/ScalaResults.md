# Manipulating results

## 改变默认的 Content-Type

Content-Type 能够从你所指定的响应体的 Scala 值自动推断出来。

例如：

```scala
val textResult = Ok("Hello World!")
```

将会自动设置 `Content-Type` 报头为 `text/plain`， 而：

```scala
val xmlResult = Ok(<message>Hello World!</message>)
```

会设置 Content-Type 报头为 `application/xml`。

> 提示：这是由 `play.api.http.ContentTypeOf` 类型类（type class）完成的。

这相当有用，但是有时候你想去改变它。只需要调用 Result 的 `as(newContentType)` 方法来创建一个新的、类似的、具有不同 `Content-Type` 报头的 Result：

```scala
val htmlResult = Ok(<h1>Hello World!</h1>).as("text/html")
```

或者用下面这种更好的方式：

```scala
val htmlResult2 = Ok(<h1>Hello World!</h1>).as(HTML)
```

> 注意：使用 `HTML` 代替 `"text/html"` 的好处是会为你自动处理字符集，这时实际的 Content-Type 报头会被设置为 `text/html; charset=utf-8`。稍后我们就能看到。

## 处理 HTTP 报头

你还能添加（或更新）结果的任意 HTTP 报头：

```scala
val result = Ok("Hello World!").withHeaders(
  CACHE_CONTROL -> "max-age=3600",
  ETAG -> "xx")
```

需要注意的是，如果一个 Result 已经有一个 HTTP 报头了，那么新设置的会覆盖前面的。

## 设置和丢弃 Cookie

Cookie 是一种特殊的 HTTP 报头，但是我们提供了一系列 helper 方法来简化操作。

你可以像下面那样很容易地添加 Cookie 到 HTTP 响应中：

```scala
val result = Ok("Hello world").withCookies(
  Cookie("theme", "blue"))
```

或者，要丢弃先前存储在 Web 浏览器中的 Cookie：

```scala
val result2 = result.discardingCookies(DiscardingCookie("theme"))
```

你也可以在同一个响应中同时添加和丢弃 Cookie：

```scala
val result3 = result.withCookies(Cookie("theme", "blue")).discardingCookies(DiscardingCookie("skin"))
```

## 改变基于文本的 HTTP 响应的字符集

对于基于文本的 HTTP 响应，正确处理好字符集是很重要的。Play 处理它的方式是采用 `utf-8` 作为默认字符集。

字符集一方面将文本响应转换成相应的字节来通过网络 Socket 进行传输，另一方面用正确的 `;charset=xxx` 扩展来更新 `Content-Type` 报头。

字符集由 `play.api.mvc.Codec` 类型类自动处理。仅需要引入一个隐式的 `play.api.mvc.Codec` 实例到当前作用域中，从而改变各种操作所用到的字符集：

```scala
object Application extends Controller {

  implicit val myCustomCharset = Codec.javaSupported("iso-8859-1")

  def index = Action {
    Ok(<h1>Hello World!</h1>).as(HTML)
  }

}
```

这里，因为作用域中存在一个隐式字符集的值，它会被应用到 `Ok(...)` 方法来将 XML 消息转化成 `ISO-8859-1` 编码的字节，同时也用于生成 `text/html;charset=iso-8859-1` Content-Type 报头。

现在，如果你想知道 `HTML` 方法是怎么工作的，以下是它的定义：

```scala
def HTML(implicit codec: Codec) = {
  "text/html; charset=" + codec.charset
}
```

如果你的 API 中需要以通用的方式来处理字符集，你可以按照上述方法进行操作。
