# Streaming HTTP responses

## 标准响应和 Content-Length 报头

从 HTTP 1.1 开始，服务器为了保持一个连接的连通并服务多个 HTTP 请求和响应，必须在响应中写入合适的 `Content-Length` HTTP 报头。

一般情况下，当你发送一个简单结果时并不会指定 `Content-Length` 报头，比如：

```scala
def index = Action {
  Ok("Hello World")
}
```

当然，由于你所发送的内容显而易见，Play能够计算出内容长度并生成适当的报头。

需要注意的是，基于文本的内容长度计算并没有如你所见的这般简单，因为 `Content-Length` 报头的计算取决于将字符转换为字节码的字符编码。

事实上，我们之前看到的响应体都是由 `play.api.libs.iteratee.Enumerator` 所指定的：

```scala
def index = Action {
  Result(
    header = ResponseHeader(200),
    body = Enumerator("Hello World")
  )
}
```

也就是说，为了能够正确的计算出 `Content-Length` 报头，Play必须读取整个枚举器（enumerator），并将其加载入内存中。

## 发送大量数据

如果对于简单的枚举器（enumerator）来说，将所有数据加载入内存中并不是个问题，但大数据集呢？假设我们想要返回给 web 客户端一个很大的文件。

让我们先来看看怎么创建一个 `Enumerator[Array[Byte]]` 来枚举整个文件的内容：

```scala
val file = new java.io.File("/tmp/fileToServe.pdf")
val fileContent: Enumerator[Array[Byte]] = Enumerator.fromFile(file)
```

这看起来很简单对吧？让我们接着用这个枚举器（enumerator）来指定响应体：

```scala
def index = Action {

  val file = new java.io.File("/tmp/fileToServe.pdf")
  val fileContent: Enumerator[Array[Byte]] = Enumerator.fromFile(file)    
    
  Result(
    header = ResponseHeader(200),
    body = fileContent
  )
}
```

事实上，这里有一个问题。因为我们并没有指定 `Content-Length` 报头，Play 需要自己来计算，唯一的方法就是读取整个枚举器（enumerator）并加载入内存，然后再计算响应的长度。

问题在于我们并不想将整个大文本加载入内存中。为了避免这种情况，我们必须自己来指定 `Content-Length` 报头。

```scala
def index = Action {

  val file = new java.io.File("/tmp/fileToServe.pdf")
  val fileContent: Enumerator[Array[Byte]] = Enumerator.fromFile(file)    
    
  Result(
    header = ResponseHeader(200, Map(CONTENT_LENGTH -> file.length.toString)),
    body = fileContent
  )
}
```

Play 会以一种惰性方式来读取这个枚举器（enumerator），在数据块可用时才将其复制到 HTTP 响应中。

## 处理文件

当然，Play 提供了简单易用的 helper 来处理本地文件：

```scala
def index = Action {
  Ok.sendFile(new java.io.File("/tmp/fileToServe.pdf"))
}
```

这个 helper 能够根据文件名计算出 `Content-Type` 报头，并且添加 `Content-Disposition` 报头告诉 web 浏览器该如何处理这个响应。默认会让 web 浏览器下载该文件并在响应中添加 `Content-Disposition: attachment; filename=fileToServe.pdf` 报头。

你也可以指定文件名：

```scala
def index = Action {
  Ok.sendFile(
    content = new java.io.File("/tmp/fileToServe.pdf"),
    fileName = _ => "termsOfService.pdf"
  )
}
```

如果你想以 `inline` 的方式处理该文件：

```scala
def index = Action {
  Ok.sendFile(
    content = new java.io.File("/tmp/fileToServe.pdf"),
    inline = true
  )
}
```

这样就不用指定文件名了，因为 web 浏览器根本不会尝试下载，而是将所有文本显示在 web 浏览器窗口中。这对于那些浏览器本身已经支持了的文本类型非常有用，如文本，html 和图片。

## 分块响应

到现在为止，流处理文件内容工作的非常好，主要是因为能够在流处理之前算出文本长度。但是如果是动态文本，无法确定文本大小呢？

对于这样的响应，我们需要使用 **分块传输编码** （Chunked transfer encoding）。

**分块传输编码**（Chunked transfer encoding）是一种定义在 Hypertext Transfer Protocol（HTTP）1.1 版本中的数据传输机制，其中 web 服务器会将文本连续分块处理。它使用了 `Transfer-Encoding` HTTP 响应报头而非 `Content-Length` 报头，如果你没有使用 `Content-Length` 的话，则必须使用这个报头。由于没有 `Content-Length`，服务器无需在开始传输响应到客户端（通常是 web 客户端）之前就知道内容的长度。 Web 服务器在知道内容总长前就能够传输动态生成的内容响应。

在发送每个数据块前，都会先发送块的大小，客户端能够依此判断是否已完整接收该数据块。数据传输会在收到一个长度为零的数据块后终结。

[http://en.wikipedia.org/wiki/Chunked_transfer_encoding](http://en.wikipedia.org/wiki/Chunked_transfer_encoding)

这样做的好处是能够实时处理数据，一旦数据可用我们便会发送。缺点是由于浏览器不知道内容长度，无法显示正确的下载进度。

假设我们有个服务，提供了一个动态 `InputStream` 来计算一些数据。首先我们需要为这个流创建一个 `Enumerator`：

```scala
val data = getDataStream
val dataContent: Enumerator[Array[Byte]] = Enumerator.fromStream(data)
```

这样就能使用 `ChunkedResult` 来流处理这些数据了：

```scala
def index = Action {

  val data = getDataStream
  val dataContent: Enumerator[Array[Byte]] = Enumerator.fromStream(data)
  
  ChunkedResult(
    header = ResponseHeader(200),
    chunks = dataContent
  )
}
```

Play 同样提供了一些 helper ：

```scala
def index = Action {

  val data = getDataStream
  val dataContent: Enumerator[Array[Byte]] = Enumerator.fromStream(data)
  
  Ok.chunked(dataContent)
}
```

当然，我们可以使用任一 `Enumerator` 来指定数据块：

```scala
def index = Action {
  Ok.chunked(
    Enumerator("kiki", "foo", "bar").andThen(Enumerator.eof)
  )
}
```

我们可以检查服务器发回的 HTTP 响应：

```scala
HTTP/1.1 200 OK
Content-Type: text/plain; charset=utf-8
Transfer-Encoding: chunked

4
kiki
3
foo
3
bar
0
```

我们得到了三个数据块，最后还跟了一个空数据块用来结束响应。