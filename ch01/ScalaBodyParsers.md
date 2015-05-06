# 请求体解析器（Body parsers）

## 什么是请求体解析器？

一个 HTTP 的 PUT 或 POST 请求包含一个请求体。请求体可以是请求头中 `Content-Type` 指定的任何格式。在 Play 中，一个请求体解析器将请求体转换为对应的 Scala 值。

然而，HTTP 请求体可能非常的大，**请求体解析器**不能等到所有的数据都加载进内存再去解析它们。`BodyParser[A]` 实际上是一个 `Iteratee[Array[Byte]，A]`，也就是说它一块一块的接收数据（只要 Web 浏览器在上传数据）并计算出类型为 `A` 的值作为结果。

让我们来看几个例子。

* 一个 **text** 类型的请求体解析器能够把逐块的字节数据连接成一个字符串，并把计算得到的字符串作为结果（`Iteratee[Array[Byte]，String]`）。
* 一个 **file** 类型的请求体解析器能够把逐块的字节数据存到一个本地文件，并以 `java.io.File` 引用作为结果（`Iteratee[Array[Byte]，File]`）。
* 一个 **s3** 类型的请求体解析器能够把逐块的字节数据推送给 Amazon S3 并以 S3 对象 ID 作为结果 (`Iteratee[Array[Byte]，S3ObjectId]`)。

此外，**请求体解析器**在开始解析数据之前已经访问了 HTTP 请求头，因此有机会可以做一些预先检查。例如，请求体解析器能检查一些 HTTP 头是否正确设置了，或者检查用户是否有权限上传一个大文件。

> 注意：这就是为什么请求体解析器不是一个真正的`Iteratee[Array[Byte]，A]`，确切的说是一个`Iteratee[Array[Byte]，Either[Result，A]]`，也就是说它在无法为请求体计算出正确的值时，可以直接发送 HTTP 结果（典型的像 `400 BAD_REQUEST`、`412 PRECONDITION_FAILED` 或者 `413 REQUEST_ENTITY_TOO_LARGE`)。

一旦请求体解析器完成了它的工作且返回了类型为 `A` 的值时，相应的 `Action` 函数就会被执行，此时计算出来的请求体的值也已被传入到请求中。

## 更多关于 Action 的内容

前面我们说 `Action` 是一个 `Request => Result` 函数，这不完全对。让我们更深入地看下 `Action` 这个特性（trait）：

```scala
trait Action[A] extends (Request[A] => Result) {
  def parser: BodyParser[A]
}
```

首先我们看到有一个泛型类 A，然后一个 Action 必须定义一个 `BodyParser[A]`。`Request[A]` 的定义如下：

```scala
trait Request[+A] extends RequestHeader {
  def body: A
}
```

`A` 是请求体的类型，我们可以使用任何 Scala 类型作为请求体，例如 `String`，`NodeSeq`，`Array[Byte]`，`JsonValue`，或是 `java.io.File`，只要我们有一个请求体解析器能够处理它。

总的来说，`Action[A]` 用返回类型为 `BodyParser[A]` 的方法去从 HTTP 请求中获取类型为 `A` 的值，并构建出 `Request[A]` 类型的对象传递给 Action 代码。

## 默认的请求体解析器：AnyContent

在我们前面的例子中还从未指定过请求体解析器，那它是怎么工作的呢？如果你不指定自己的请求体解析器，Play 就会使用默认的，它把请求体处理成一个 `play.api.mvc.AnyContent` 实例。

默认的请求体解析通过查看 `Content-Type` 报头来决定要处理的请求体类型：

- **text/plain**：`String`
- **application/json**：`JsValue`
- **application/xml**，**text/xml** 或者 **application/XXX+xml**：`NodeSeq`
- **application/form-url-encoded**：`Map[String，Seq[String]]`
- **multipart/form-data**：`MultipartFormData[TemporaryFile]`
- 任何其他的类型：`RawBuffer`

例如：

```scala
def save = Action { request =>
  val body: AnyContent = request.body
  val textBody: Option[String] = body.asText

  // Expecting text body
  textBody.map { text =>
    Ok("Got: " + text)
  }.getOrElse {
    BadRequest("Expecting text/plain request body")
  }
}
```

## 指定一个请求体解析器

Play 中可用的请求体解析器定义在 `play.api.mvc.BodyParsers.parse` 中。

例如，定义了一个处理 text 类型请求体的 Action（像前面示例中那样）：

```scala
def save = Action(parse.text) { request =>
  Ok("Got: " + request.body)
}
```

你知道代码是如何变简单的吗？这是因为如果发生了错误，`parse.text` 这个请求体解析器会发送一个 `400 BAD_REQUEST` 的响应。我们在 Action 代码中没有必要再去做检查。我们可以放心地认为 `request.body` 中是合法的 `String`。

或者，我们也可以这么用：

```scala
def save = Action(parse.tolerantText) { request =>
  Ok("Got: " + request.body)
}
```
这个方法不会检查 `Content-Type` 报头，而总是把请求体加载为字符串。

> 小贴士: 在 Play 中所有的请求体解析都提供有 `tolerant` 样式的方法。

这是另一个例子，它将把请求体存为一个文件：

```scala
def save = Action(parse.file(to = new File("/tmp/upload"))) { request =>
  Ok("Saved the request content to " + request.body)
}
```

## 组合请求体解析器

在前面的例子中，所有的请求体都会存到同一个文件中。这会产生难以预料的问题，不是吗？我们来写一个定制的请求体解析器，它会从请求会话中得到用户名，并为每个用户生成不同的文件：

```scala
val storeInUserFile = parse.using { request =>
  request.session.get("username").map { user =>
    file(to = new File("/tmp/" + user + ".upload"))
  }.getOrElse {
    sys.error("You don't have the right to upload here")
  }
}

def save = Action(storeInUserFile) { request =>
  Ok("Saved the request content to " + request.body)
}
```
> 注意：这里我们并没有真正的写自己的请求体解析器，只不过是组合了现有的。这样做通常足够了，能应付多数情况。从头写一个`请求体解析器`会在高级主题部分涉及到。

## 最大内容长度

基于文本的请求体解析器（如`text`，`json`，`xml` 或 `formUrlEncoded`）会有一个最大内容长度，因为它们要加载所有内容到内存中。

默认的最大内容长度是 100KB，但是你也可以在代码中指定它：

```scala
// Accept only 10KB of data.
def save = Action(parse.text(maxLength = 1024 * 10)) { request =>
  Ok("Got: " + text)
}
```

> 小贴士：默认的内容大小可在 `application.conf` 中定义：
> `parsers.text.maxLength=128K`

你可以在任何请求体解析器中使用 `maxLength`：

```scala
// Accept only 10KB of data.
def save = Action(parse.maxLength(1024 * 10, storeInUserFile)) { request =>
  Ok("Saved the request content to " + request.body)
}
```
