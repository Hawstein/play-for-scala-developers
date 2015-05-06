# Session 和 Flash 域

## 它们在 Play 中有何不同

如果你必须跨多个 HTTP 请求来保存数据，你可以把数据存在 Session 或是 Flash 域中。存储在 Session 中的数据在整个会话期间可用，而存储在 Flash 域中数据只对下一次请求有效。

Session 和 Flash 数据不是由服务器来存储，而是以 Cookie 机制添加到每一次后续的 HTTP 请求。这也就意味着数据的大小是很受限的（最多 4KB），而且你只能存储字符串类型的值。在 Play 中默认的 Cookie 名称是 `PLAY_SESSION`。默认的名称可以在 application.conf 中通过配置 `session.cookieName` 的值来修改。

> 如果 Cookie 的名字改变了，可以使用上一节中「设置和丢弃 Cookie」提到的同样的方法来使之前的 Cookie 失效。

当然了，Cookie 值是由密钥签名了的，这使得客户端不能修改 Cookie 数据（否则它会失效）。

Play 的 Session 不能当成缓存来用。假如你需要缓存与某一会话相关的数据，你可以使用 Play 内置的缓存机制并在用户会话中存储一个唯一的 ID 与之关联。

> 技术上来说，Session 并没有超时控制，它在用户关闭 Web 浏览器后就会过期。如果你的应用需要一种功能性的超时控制，那就在用户 Session 中存储一个时间戳，并在应用需要的时候用它（例如：最大的会话持续时间，最大的非活动时间等）。

## 存储数据到 Session 中

由于 Session 是个 Cookie 也是个 HTTP 头，所以你可用操作其他 Result 属性那样的方式操作 Session 数据：

```scala
Ok("Welcome!").withSession(
  "connected" -> "user@gmail.com")
```

这会替换掉整个 Session。假如你需要添加元素到已有的 Session 中，方法是先添加元素到传入的 Session 中，接着把它作为新的 Session：

```scala
Ok("Hello World!").withSession(
  request.session + ("saidHello" -> "yes"))
```

你可以用同样的方式从传入的 Session 中移除任何值：

```scala
Ok("Theme reset!").withSession(
  request.session - "theme")
```

## 从 Session 中读取值

你可以从 HTTP 请求中取回传入的 Session：

```scala
def index = Action { request =>
  request.session.get("connected").map { user =>
    Ok("Hello " + user)
  }.getOrElse {
    Unauthorized("Oops, you are not connected")
  }
}
```

## 丢弃整个 Session

有一个特殊的操作用来丢弃整个 Session：

```scala
Ok("Bye").withNewSession
```

## Flash 域

Flash 域工作方式非常像 Session，但有两点不同：

* 数据仅为一次请求而保留。
* Flash 的 Cookie 未被签名，这留给了用户修改它的可能。

> Flash 域只应该用于简单的非 Ajax 应用中传送 success/error 消息。由于数据只保存给下一次请求，而且在复杂的 Web 应用中无法保证请求的顺序，所以在竞争条件（race conditions）下 Flash 可能不那么好用。

下面是一些使用 Flash 域的例子：

```scala
def index = Action { implicit request =>
  Ok {
    request.flash.get("success").getOrElse("Welcome!")
  }
}

def save = Action {
  Redirect("/home").flashing(
    "success" -> "The item has been created")
}
```

为了在你的视图中使用 Flash 域，需要加上 Flash 域的隐式转换：

`@()(implicit flash: Flash) ...`
`@flash.get("success").getOrElse("Welcome!") ...`

如果出现 `could not find implicit value for parameter flash: play.api.mvc.Flash` 的错误，像下面这样加上 `implicit request=>` 就解决了：

```scala
def index() = Action {
  implicit request =>
    Ok(views.html.Application.index())
}
```
