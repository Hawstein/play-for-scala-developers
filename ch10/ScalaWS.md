# The Play WS API

有的时候我们需要在 Play 应用中调用其它 HTTP 服务。Play 通过 WS 库来支持这一需求，它提供了一种方法来做异步 HTTP 调用。

使用 WS API 包含两个重要部分：构造请求以及处理响应。我们会先讨论如何构造 GET 和 POST 的 HTTP 请求，然后展示如何处理返回的响应。最后，我们会讨论一些常见用例。

## 构造请求

想要使用 WS，首先需要在你的 `build.sbt` 文件中加上 `ws` 库：

```scala
libraryDependencies ++= Seq(
  ws
)
```

然后导入以下内容：

```scala
import play.api.Play.current
import play.api.libs.ws._
import play.api.libs.ws.ning.NingAsyncHttpClientConfigBuilder
import scala.concurrent.Future
```

想要构建一个 HTTP 请求，你首先要用 `WS.url()` 来指定 URL：

```scala
val holder: WSRequestHolder = WS.url(url)
```

上面的语句返回一个类型为 `WSRequestHolder` 的值，你可以通过它指定各种 HTTP 选项，例如设置各种报头。你可以使用链式调用来构造复杂的请求。

```scala
val complexHolder: WSRequestHolder =
  holder.withHeaders("Accept" -> "application/json")
    .withRequestTimeout(10000)
    .withQueryString("search" -> "play")
```

最后你通过调用一个与 HTTP 方法对应的方法来发出请求，它会使用你之前设置的所有选项。

```scala
val futureResponse: Future[WSResponse] = complexHolder.get()
```

上述调用返回的类型是 `Future[WSResponse]`，返回的响应包含了服务器传来的数据。

### 带验证的请求

如果你需要使用 HTTP 验证，你可以要构造请求的过程中指定它，使用用户名、密码以及 [AuthScheme](https://www.playframework.com/documentation/2.3.x/api/scala/index.html#play.api.libs.ws.WSAuthScheme)。AuthScheme 的有效样例对象（case objects）有：`BASIC`，`DIGEST`，`KERBEROS`，`NONE`，`NTLM` 和 `SPNEGO`。

```scala
WS.url(url).withAuth(user, password, WSAuthScheme.BASIC).get()
```

### 带重定向的请求

如果一个 HTTP 请求的结果是 302 或 301 重定向，你可以自动执行重定向而无需另一次的调用。

```scala
WS.url(url).withFollowRedirects(true).get()
```

### 带查询参数的请求

参数可以被指定为一系列的 K/V 元组。

```scala
WS.url(url).withQueryString("paramKey" -> "paramValue").get()
```

### 带额外报头的请求

报头也可以被指定为一系列的 K/V 元组。

```scala
WS.url(url).withHeaders("headerKey" -> "headerValue").get()
```

如果你想以特殊格式发送纯文本，你需要显式地定义内容类型：

```scala
WS.url(url).withHeaders("Content-Type" -> "application/xml").post(xmlString)
```

### 带虚拟主机的请求

你可以用一个字符串来指定一个虚拟主机：

```scala
WS.url(url).withVirtualHost("192.168.1.1").get()
```

### 带超时时间的请求

如果你想指定请求的超时时间，你可以使用 `withRequestTimeout` 来设置一个单位为毫秒的值：

```scala
WS.url(url).withRequestTimeout(5000).get()
```

### 提交表单数据

想要 post 一个 url-form-encoded 的数据，你需要给 `post` 方法传一个类型为 `Map[String, Seq[String]]` 的值。

```scala
WS.url(url).post(Map("key" -> Seq("value")))
```

### 提交 JSON 数据

post Json 数据最简单的方法就是使用 JSON 库。

```scala
import play.api.libs.json._
val data = Json.obj(
  "key1" -> "value1",
  "key2" -> "value2"
)
val futureResponse: Future[WSResponse] = WS.url(url).post(data)
```

### 提交 XML 数据

post XML 数据最简单的方法是使用 XML 字面量。XML 字面量用起来很方便，但速度不快。追求效率的话，请考虑使用 XML 视图模板或 JAXB 库。

```scala
val data = <person>
  <name>Steve</name>
  <age>23</age>
</person>
val futureResponse: Future[WSResponse] = WS.url(url).post(data)
```

## 处理响应

处理 [Response](https://www.playframework.com/documentation/2.3.x/api/scala/index.html#play.api.libs.ws.Response) 可以通过在 [Future](http://www.scala-lang.org/api/current/index.html#scala.concurrent.Future) 内做映射来简单完成。

下面给出的例子都有一些共同的依赖，在这里先简单地说明一下。

任何时候 `Future` 上的一个操作，都是需要一个隐式的执行上下文的，这一点声明了回调应该运行在哪个线程池。通常，Play 默认的执行上下文就足够了：

```scala
implicit val context = play.api.libs.concurrent.Execution.Implicits.defaultContext
```

下面的例子还使用了以下样例类来进行序列化与反序列化：

```scala
case class Person(name: String, age: Int)
```

### 处理 JSON 响应

你可以通过调用 `response.json` 将响应当作 JSON 对象来处理。

```scala
val futureResult: Future[String] = WS.url(url).get().map {
  response =>
    (response.json \ "person" \ "name").as[String]
}
```

Play 的 JSON 库还有一个有用的特性，可以将一个隐式的 `Reads[T]` 直接映射成一个类：

```scala
import play.api.libs.json._

implicit val personReads = Json.reads[Person]

val futureResult: Future[JsResult[Person]] = WS.url(url).get().map {
  response => (response.json \ "person").validate[Person]
}
```

### 处理 XML 响应

你可以通过调用 `response.xml` 将响应当作 XML 字面量来处理。

```scala
val futureResult: Future[scala.xml.NodeSeq] = WS.url(url).get().map {
  response =>
    response.xml \ "message"
}
```

### 处理大块响应

调用 `get()` 或 `post()` 方法会导致一个问题，就是只有当响应体完全载入到内存中，响应才可用。当你在下载几个 G 的大文件时，这可能会导致另人不爽的 GC，甚至出现内存不足的错误。

`WS` 可以让你通过 [iteratee](https://www.playframework.com/documentation/2.3.x/Iteratees) 来增量地使用响应。`WSRequestHolder` 的 `stream()` 和 `getStream()` 方法返回一个 `Future[(WSResponseHeaders, Enumerator[Array[Byte]])]`。其中，枚举器包含了响应体。

下面是一个常见的例子，使用 iteratee 计算响应返回的字节数：

```scala
import play.api.libs.iteratee._

// Make the request
val futureResponse: Future[(WSResponseHeaders, Enumerator[Array[Byte]])] =
  WS.url(url).getStream()

val bytesReturned: Future[Long] = futureResponse.flatMap {
  case (headers, body) =>
    // Count the number of bytes returned
    body |>>> Iteratee.fold(0l) { (total, bytes) =>
      total + bytes.length
    }
}
```

当然，通常情况下你不会像上面那样只是计算数据的字节数，更常见的情况是把响应返回的数据写到另一个地方去，比如写入文件：

```scala
import play.api.libs.iteratee._

// Make the request
val futureResponse: Future[(WSResponseHeaders, Enumerator[Array[Byte]])] =
  WS.url(url).getStream()

val downloadedFile: Future[File] = futureResponse.flatMap {
  case (headers, body) =>
    val outputStream = new FileOutputStream(file)

    // The iteratee that writes to the output stream
    val iteratee = Iteratee.foreach[Array[Byte]] { bytes =>
      outputStream.write(bytes)
    }

    // Feed the body into the iteratee
    (body |>>> iteratee).andThen {
      case result =>
        // Close the output stream whether there was an error or not
        outputStream.close()
        // Get the result or rethrow the error
        result.get
    }.map(_ => file)
}
```

另一种情况是，当前服务器把拿到的响应体流式写入另一个响应中，返回给它所服务的对象：

```scala
def downloadFile = Action.async {

  // Make the request
  WS.url(url).getStream().map {
    case (response, body) =>

      // Check that the response was successful
      if (response.status == 200) {

        // Get the content type
        val contentType = response.headers.get("Content-Type").flatMap(_.headOption)
          .getOrElse("application/octet-stream")

        // If there's a content length, send that, otherwise return the body chunked
        response.headers.get("Content-Length") match {
          case Some(Seq(length)) =>
            Ok.feed(body).as(contentType).withHeaders("Content-Length" -> length)
          case _ =>
            Ok.chunked(body).as(contentType)
        }
      } else {
        BadGateway
      }
  }
}
```

`POST` 和 `PUT` 的调用需要手动调用 `withMethod` 方法：

```scala
val futureResponse: Future[(WSResponseHeaders, Enumerator[Array[Byte]])] =
  WS.url(url).withMethod("PUT").withBody("some body").stream()
```

## 常见模式和用例

### 链式 WS 调用

使用 for 解析是一种链接 WS 调用的好方式。for 解析应该和 [Future.recover](http://www.scala-lang.org/api/current/index.html#scala.concurrent.Future) 配合使用，用于处理可能的失败。

```scala
val futureResponse: Future[WSResponse] = for {
  responseOne <- WS.url(urlOne).get()
  responseTwo <- WS.url(responseOne.body).get()
  responseThree <- WS.url(responseTwo.body).get()
} yield responseThree

futureResponse.recover {
  case e: Exception =>
    val exceptionData = Map("error" -> Seq(e.getMessage))
    WS.url(exceptionUrl).post(exceptionData)
}
```

### 在控制器中使用

当在控制器中构建请求时，你可以将响应映射为 `Future[Result]`。这可以与 Play 的 `Action.async` 结合使用，详见：[Handling Asynchronous Results](https://www.playframework.com/documentation/2.3.x/ScalaAsync)。

```scala
def wsAction = Action.async {
  WS.url(url).get().map { response =>
    Ok(response.body)
  }
}
status(wsAction(FakeRequest())) must_== OK
```

## 使用 WSClient

WSClient 是底层 AsyncHttpClient 的 wrapper。有时候你需要定义多个客户端，建议使用不同的配置文件，或使用模拟的方式。

默认的客户端可以由 WS 单例调用：

```scala
val client: WSClient = WS.client
```

你可以直接从代码中定义一个 WS 客户端，通过 `WS.clientUrl()` 隐式地使用。注意，当你配置你的客户端时，你应该使用 `NingAsyncHttpClientConfigBuilder` 来做 TLS 配置：

```scala
val clientConfig = new DefaultWSClientConfig()
val secureDefaults:com.ning.http.client.AsyncHttpClientConfig = new NingAsyncHttpClientConfigBuilder(clientConfig).build()
// You can directly use the builder for specific options once you have secure TLS defaults...
val builder = new com.ning.http.client.AsyncHttpClientConfig.Builder(secureDefaults)
builder.setCompressionEnabled(true)
val secureDefaultsWithSpecificOptions:com.ning.http.client.AsyncHttpClientConfig = builder.build()
implicit val implicitClient = new play.api.libs.ws.ning.NingWSClient(secureDefaultsWithSpecificOptions)
val response = WS.clientUrl(url).get()
```

> 注意：如果你实例化一个 NingWSClient 对象，它并不会使用 WS 插件系统，因此它不会在 `Application.onStop` 里自动关闭。当处理结束时，你需要使用 `client.close()` 来手动关闭客户端。这会释放 AsyncHttpClient 使用的底层 ThreadPoolExecutor。客户端关闭失败可能会导致内存不足的异常（尤其在开发模式下，需要经常性重新加载应用的时候）。

也可以像下面直接使用：

```scala
val response = client.url(url).get()
```

或使用磁铁模式（magnet pattern）来自动匹配合适的客户端：

```scala
object PairMagnet {
  implicit def fromPair(pair: (WSClient, java.net.URL)) =
    new WSRequestHolderMagnet {
      def apply(): WSRequestHolder = {
        val (client, netUrl) = pair
        client.url(netUrl.toString)
      }
    }
}

import scala.language.implicitConversions
import PairMagnet._

val client = WS.client
val exampleURL = new java.net.URL(url)
val response = WS.url(client -> exampleURL).get()
```

默认情况下，配置一般写在 `application.conf` 里，但你也可以直接在代码中设置配置：

```scala
import com.typesafe.config.ConfigFactory
import play.api.libs.ws._
import play.api.libs.ws.ning._

val configuration = play.api.Configuration(ConfigFactory.parseString(
  """
    |ws.followRedirects = true
  """.stripMargin))

val classLoader = app.classloader // Play.current.classloader or other
val parser = new DefaultWSConfigParser(configuration, classLoader)
val builder = new NingAsyncHttpClientConfigBuilder(parser.parse())
```

你也可以直接访问底层的 [async client](http://sonatype.github.io/async-http-client/apidocs/reference/com/ning/http/client/AsyncHttpClient.html)。

```scala
import com.ning.http.client.AsyncHttpClient

val client: AsyncHttpClient = WS.client.underlying
```

可以直接访问底层类是非常重要的，因为 WS 在有的时候会有一些限制：

* `WS` 并不直接支持 multi-part-form 数据的上传。你可以使用底层客户端的 [RequestBuilder.addBodyPart](http://asynchttpclient.github.io/async-http-client/apidocs/com/ning/http/client/RequestBuilder.html) 来做。
* `WS` 不支持流式数据上传。在这种情况下，你应该使用 AsyncHttpClient 提供的 `FeedableBodyGenerator`。

## 配置 WS

在 `application.conf` 文件中使用如下属性来配置 WS 客户端：

* `ws.followRedirects`：配置客户端做 301、302 重定向（默认为 true）。
* `ws.useProxyProperties`：使用系统的 http 代理设置（http.proxyHost，http.proxyPort）（默认为 true）。
* `ws.useragent`：配置 User-Agent 报头。
* `ws.compressionEnabled`：如果使用 gzip/deflater 编码，则将它设置为 true（默认为 false）。

### 用 SSL 配置 WS

想要配置 WS 在 SSL/TLS（HTTPS）之上使用 HTTP，请移步：[配置 WS SSL](https://www.playframework.com/documentation/2.3.x/WsSSL)

### 配置超时时间

WS 中有 3 种不同的超时。超时会导致 WS 请求中断。

* `ws.timeout.connection`：连接远程主机的最大等待时间（默认是 120 秒）。
* `ws.timeout.idle`：请求保持空闲状态的最大时间（此时连接已经建立，在等待更多的数据）（默认是 120 秒）。
* `ws.timeout.request`：你能允许的请求使用的总时间（当达到这个时间时，请求就会中断，哪怕远程主机仍然在发送数据）（默认是 none，为的是可以处理流式数据）。

在一个具体的连接中，你可以用 `withRequestTimeout()` 来覆盖配置文件中的请求超时设置（详见「构造请求」一节）。
