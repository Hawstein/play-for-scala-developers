# WebSockets

[WebSocket](http://en.wikipedia.org/wiki/WebSocket) 是一种可以在浏览器内使用的 socket，它基于一种支持全双工通信的协议。只要服务器端和客户端之间存在一个活跃的 WebSocket 连接，它们之间就可以在任意时刻收发信息。

兼容 HTML5 的现代 web 浏览器通过 JavaScript WebSocket API 可以原生支持 WebSocket。然而，WebSocket 并不只局限于在浏览器中使用，有许多 WebSocket 客户端库可以用于服务器间通信，或者提供给原生的移动应用来使用。在这些情况下使用 WebSocket 有一个很大的优势，就是复用 Play 服务器已经使用的 TCP 端口。

## 处理 WebSocket

到目前为止，我们都是用 `Action` 实例来处理标准的 HTTP 请求，然后返回标准的 HTTP 响应。WebSocket 则完全不同，它无法通过标准的 `Action` 来处理。

Play 提供了两种不同的内建机制来处理 WebSocket。第一种使用 actor，第二种使用 iteratee。这两种机制都可以通过 Play 为 WebSocket 提供的构建器来使用。

## 用 actor 处理 WebSocket

想用 actor 来处理 WebSocket，我们需要一个 `akka.actor.Props` 对象来描述 actor，当 WebSocket 连接建立起来后，Play 应该创建这个 actor。Play 会提供一个 `akka.actor.ActorRef` 用于向它发送消息，因此我们可以用它（即下面的 out）来创建 `Props` 对象：

```scala
import play.api.mvc._
import play.api.Play.current

def socket = WebSocket.acceptWithActor[String, String] { request => out =>
  MyWebSocketActor.props(out)
}
```

在这个例子中，我们向它发送消息的 actor 定义如下：

```scala
import akka.actor._

object MyWebSocketActor {
  def props(out: ActorRef) = Props(new MyWebSocketActor(out))
}

class MyWebSocketActor(out: ActorRef) extends Actor {
  def receive = {
    case msg: String =>
      out ! ("I received your message: " + msg)
  }
}
```

任何从客户端收到的信息都会被发送到 actor（即 out），任何发送给 Play 提供的 actor 的信息都会被发送到客户端。上面的 actor 就简单地将它收到的信息在前面加上 `I reveived your message:`，然后返回给客户端。

### 检测 WebSocket 何时关闭

如果 WebSocket 已经关闭，Play 会自动停掉 actor。这意味着你可以通过实现 `postStop` 方法来清理 WebSocket 可能已经使用的资源。例如：

```scala
override def postStop() = {
  someResource.close()
}
```

### 关闭 WebSocket

当处理 WebSocket 的 actor 终止时，Play 会自动关闭 WebSocket。因此，你可以通过发送一个 `PoisonPill` 给你自己的 actor 来关闭 WebSocket：

```scala
import akka.actor.PoisonPill

self ! PoisonPill
```

### 拒绝 WebSocket 请求

有时候你可能想拒绝一个 WebSocket 请求，例如，必须是授权用户才能连接 WebSocket，或者 WebSocket 所关联到的资源不存在。Play 提供了 `tryAcceptWithActor` 来处理这种情况，允许你返回一个结果（如 forbidden 或 not found），或是返回一个处理 WebSocket 的 actor：

```scala
import scala.concurrent.Future
import play.api.mvc._
import play.api.Play.current

def socket = WebSocket.tryAcceptWithActor[String, String] { request =>
  Future.successful(request.session.get("user") match {
    case None => Left(Forbidden)
    case Some(_) => Right(MyWebSocketActor.props)
  })
}
```

### 处理不同类型的信息

到目前为止，我们看到的都是在处理 `String` 信息。其实，Play 也提供了内建的方式来处理 `Array[Byte]` 和 `JsValue` 信息。你可以把这些作为类型参数传递给 WebSocket 构建方法，例如：

```scala
import play.api.mvc._
import play.api.libs.json._
import play.api.Play.current

def socket = WebSocket.acceptWithActor[JsValue, JsValue] { request => out =>
  MyWebSocketActor.props(out)
}
```

你可以已经注意到了，上面有两个类型参数（都是 JsValue），这允许我们将接收进来的信息与发送出去的信息定义为不同的类型。这通常对于低级类型来说是没什么用的，但当你想把信息转换成高级类型时，它就非常有用了。

例如，我们想接收 JSON 类型的信息，然后把它解析成 `InEvent` 类型，再把返回的信息转成 `OutEvent` 类型。我们要做的第一件事是为 `InEvent` 和 `OutEvent` 创建 JSON 格式（用于隐式转换）：

```scala
import play.api.libs.json._

implicit val inEventFormat = Json.format[InEvent]
implicit val outEventFormat = Json.format[OutEvent]
```

接着，我们为这两个类型创建 WebSocket `FrameFormatter`：

```scala
import play.api.mvc.WebSocket.FrameFormatter

implicit val inEventFrameFormatter = FrameFormatter.jsonFrame[InEvent]
implicit val outEventFrameFormatter = FrameFormatter.jsonFrame[OutEvent]
```

最终，我们可以在 WebSocket 中使用这两个类型：

```scala
import play.api.mvc._
import play.api.Play.current

def socket = WebSocket.acceptWithActor[InEvent, OutEvent] { request => out =>
  MyWebSocketActor.props(out)
}
```

现在，在我们的 actor 中，我们会接收到类型为 `InEvent` 的信息，而发送出去的信息类型为 `OutEvent`。

## 用 iteratee 处理 WebSocket

actor 是一种更好的抽象来处理离散信息，而 iteratee 是一种更好的抽象来处理流。

处理 WebSocket 请求，使用 `WebSocket` 而不是 `Action`：

```scala
import play.api.mvc._
import play.api.libs.iteratee._
import play.api.libs.concurrent.Execution.Implicits.defaultContext

def socket = WebSocket.using[String] { request =>

  // Log events to the console
  val in = Iteratee.foreach[String](println).map { _ =>
    println("Disconnected")
  }

  // Send a single 'Hello!' message
  val out = Enumerator("Hello!")

  (in, out)
}
```

`WebSocket` 可以访问初始化 WebSocket 连接的 HTTP 请求的报头，允许你获取标准报头和会话数据。然而，它无权访问请求体或是 HTTP 响应。

当你通过这种方式来构建 WebSocket 时，你必须同时返回 `in` 和 `out` 两个通道。

* `in` 通道的类型是 `Iteratee[A,Unit]`（其中 `A` 是消息类型，这里我们用的是 `String`），对于每条传来的消息，它都会被通知到。当客户端的 socket 关闭后，它会收到 `EOF`。
* `out` 通道的类型是 `Enumerator[A]`，它会产生发送给 Web 客户端的消息。它也可以通过发送 `EOF` 在服务器端关闭连接。

在这个例子中，我们创建了一个简单的 iteratee，它会在控制台打印每条消息。同时，我们创建了一个简单的枚举器（enumerator）来发送一个 `Hello!` 消息。

> 小贴士：你可以在[这里](http://websocket.org/echo.html)测试 WebSocket，只需要把 location 设置为 `ws://localhost:9000` 即可。

下面的例子直接忽略输入数据，在发送完 **Hello!** 消息后就关闭 socket：

```scala
import play.api.mvc._
import play.api.libs.iteratee._

def socket = WebSocket.using[String] { request =>

  // Just ignore the input
  val in = Iteratee.ignore[String]

  // Send a single 'Hello!' message and close
  val out = Enumerator("Hello!").andThen(Enumerator.eof)

  (in, out)
}
```

下面是另外一个例子，其中输入数据被打印到标准输出，然后利用  `Concurrent.broadcast` 广播到客户端：

```scala
import play.api.mvc._
import play.api.libs.iteratee._
import play.api.libs.concurrent.Execution.Implicits.defaultContext

def socket =  WebSocket.using[String] { request =>

  // Concurrent.broadcast returns (Enumerator, Concurrent.Channel)
  val (out, channel) = Concurrent.broadcast[String]

  // log the message to stdout and send response back to client
  val in = Iteratee.foreach[String] {
    msg => println(msg)
      // the Enumerator returned by Concurrent.broadcast subscribes to the channel and will
      // receive the pushed messages
      channel push("I received your message: " + msg)
  }
  (in,out)
}
```
