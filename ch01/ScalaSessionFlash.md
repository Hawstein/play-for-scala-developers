# Session and Flash scopes

##Session域和Flash域在Play中有何不同
如果你必须跨多个HTTP请求来保存数据，你可以把数据存在Session域或是Flash域中。存储在Session域中的数据在整个会话期间可用，而存储在Flash域中数据只对下一次请求有效。

理解到Session和Flash数据不是由服务器来存储，而是以Cookie机制添加到每一次后续的HTTP请求是很重要的。这也就意味着数据的大小是很受限的（最多4KB），而且你只能存储字符串型值。在Play中默认的Cookie名称是`PLAY_SESSION`。默认的名称可以在application.conf中使用名称为`session.cookieName`的键来修改。

`如果Cookie的名字改变了，可以使用上一节中“设置和废弃Cookie”提到的同样的方法来使之前的Cookie失效。`

当然了，Cookie值是由密钥签名了的，这使得客户端不能修改Cookie数据（或者说这样做会让它失效）。

Play的Session不能当成缓存来用。假如你需要缓存与某一会话相关的数据，你可以使用Play内置的缓存机制并在用户会话中存储一个唯一的ID与之关联。

Session没有技术上的超时控制，它在用户关闭Web浏览器后就会过期。如果你的应用需要一种功能性的超时控制，那就在用户Session中存储一个时间戳，并在应用需要的时候用它（例如：最大的会话持续时间，最大的非活动时间等）。

##存储数据到Session中
由于Session是个 Cookie也是个HTTP头，所以你可用操作其他Result属性那样的方式操作Session数据：
```scala
Ok("Welcome!").withSession(
  "connected" -> "user@gmail.com")
```
这会替换掉整个Session。假如你需要添加元素到已有的Session中，方法是先添加元素到传入的Session中，接着把它作为新的Session：
```scala
Ok("Hello World!").withSession(
  request.session + ("saidHello" -> "yes"))
```
你可以用同样的方式从传入的 Session 中移除任何值：
```scala
Ok("Theme reset!").withSession(
  request.session - "theme")
```

##从Session中读取值
你可以从 HTTP 请求中取回传入的Session：
```scala
def index = Action { request =>
  request.session.get("connected").map { user =>
    Ok("Hello " + user)
  }.getOrElse {
    Unauthorized("Oops, you are not connected")
  }
}
```

##废弃整个Session
有一个特殊的操作用来废弃整个Session：
```scala
Ok("Bye").withNewSession
```

##Flash域
Flash域工作方式非常像Session，但有两点不同：
- 数据仅为一次请求而保留。
- Flash的Cookie未被签名，这留给了用户修改它的可能。

`Flash域仅能用于简单的非Ajax应用中在传送success／error的消息。由于数据只保存给下一次请求，而且在复杂的Web应用中无法保证请求的顺序，所以在竞争条件下Flash可能不那么好使。`

下面是一些使用Flash域的例子：
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
为了在你的视图中使用Flash域，需要加上Flash域的隐式转换：
`@()(implicit flash: Flash) ...` `@flash.get("success").getOrElse("Welcome!") ...`

如果出现“could not find implicit value for parameter flash: play.api.mvc.Flash”的错误，像下面这样加上“implicit request=>”就解决了：
```scala
def index() = Action {
  implicit request =>
    Ok(views.html.Application.index())
}
```