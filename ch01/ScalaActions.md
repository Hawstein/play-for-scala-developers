# Actions, Controllers and Results

## 什么是 Action

Play 应用收到的大部分请求都是由 `Action` 来处理的。

`play.api.mvc.Action` 就是一个处理收到的请求然后产生结果发给客户端的函数（`play.api.mvc.Request => play.api.mvc.Result`）。

```scala
val echo = Action { request =>
  Ok("Got request [" + request + "]")
}
```

Action 返回一个类型为 `play.api.mvc.Result` 的值，代表了发送到 web 客户端的 HTTP 响应。在这个例子中，`OK` 构造了一个 **200 OK** 的响应，并包含一个 **text/plain** 类型的响应体。

## 构建一个 Action

`play.api.mvc.Action` 的伴生对象（companion object）提供了一些 helper 方法来构造一个 Action 值。

最简单的一个函数是 `OK`，它接受一个表达式块作为参数并返回一个 `Result`：

```scala
Action {
  Ok("Hello world")
}
```

这是构造 Action 的最简单方法，但在这种方法里，我们并没有使用传进来的请求。实际应用中，我们常常要使用调用这个 Action 的 HTTP 请求。

因此，还有另一种 Action 构造器，它接受一个函数 `Request => Result` 作为输入：

```scala
Action { request =>
  Ok("Got request [" + request + "]")
}
```

实践中常常把 `request` 标记为 `implicit`，这样一来，其它需要它的 API 能够隐式使用它：

```scala
Action { implicit request =>
  Ok("Got request [" + request + "]")
}
```

最后一种创建 Action 的方法是指定一个额外的 `BodyParser` 参数：

```scala
Action(parse.json) { implicit request =>
  Ok("Got request [" + request + "]")
}
```

这份手册后面会讲到 Body 解析器（Body Parser）。现在你只需要知道，上面讲到的其它构造 Action 的方法使用的是一个默认的解析器：**任意内容 body 解析器**（Any content body parser）。

## 控制器（Controller）是 action 生成器

一个 `Controller` 就是一个产生 `Action` 值的单例对象。

定义一个 action 生成器的最简单方式就是定义一个无参方法，让它返回一个 `Action` 值：

```scala
package controllers

import play.api.mvc._

object Application extends Controller {

  def index = Action {
    Ok("It works!")
  }

}
```

当然，生成 action 的方法也可以带参数，并且这些参数可以在 `Action` 闭包中访问到：

```scala
def hello(name: String) = Action {
  Ok("Hello " + name)
}
```

## 简单结果

到目前为止，我们就只对简单结果感兴趣：HTTP 结果。它包含了一个状态码，一组 HTTP 报头和发送给 web 客户端的 body。

这些结果由 `play.api.mvc.Result` 定义：

```scala
def index = Action {
  Result(
    header = ResponseHeader(200, Map(CONTENT_TYPE -> "text/plain")),
    body = Enumerator("Hello world!".getBytes())
  )
}
```

当然，Play 提供了一些 helper 方法来构造常见结果，比如说 `OK`。下面的代码和上面的代码是等效的：

```scala
def index = Action {
  Ok("Hello world!")
}
```

上面两段代码产生的结果是一样的。

下面是生成不同结果的一些例子：

```scala
val ok = Ok("Hello world!")
val notFound = NotFound
val pageNotFound = NotFound(<h1>Page not found</h1>)
val badRequest = BadRequest(views.html.form(formWithErrors))
val oops = InternalServerError("Oops")
val anyStatus = Status(488)("Strange response type")
```

上面的 helper 方法都可以在 `play.api.mvc.Results` 特性（trait）和伴生对象（companion object）中找到。

## 重定向也是简单结果

重定向到一个新的 URL 是另一种简单结果。然而这些结果类型并不包含一个响应体。

同样地，有一些 helper 方法可以来创建重定向结果：

```scala
def index = Action {
  Redirect("/user/home")
}
```

默认使用的响应类型是：`303 SEE_OTHER`，当然，如果你有需要，可以自己设定状态码：

```scala
def index = Action {
  Redirect("/user/home", MOVED_PERMANENTLY)
}
```

## 「TODO」 dummy 页面

你可以使用一个定义为 `TODO` 的空的 `Action` 实现，它的结果是一个标准的 `Not implemented yet` 页面：

```scala
def index(name:String) = TODO
```
