# 处理XML请求和提供XML响应

## 处理XML请求
XML请求是指HTTP请求将有效的XML格式的内容作为请求体，该HTTP请求必须在`Content-Type`中指定它的MIME类型为`application/xml`或者`text/xml`。

一般来说，`Action`使用**任意内容**解析器，这使得程序可以接受XML格式的请求体（实际上是作为`NodeSeq`类处理）：

```scala
def sayHello = Action { request =>
  request.body.asXml.map { xml =>
    (xml \\ "name" headOption).map(_.text).map { name =>
      Ok("Hello " + name)
    }.getOrElse {
      BadRequest("Missing parameter [name]")
    }
  }.getOrElse {
    BadRequest("Expecting Xml data")
  }
}
```

更好（也更简单）的方法是明确给定我们要用的`BodyParser`，让Play框架直接将请求体内容当作XML格式解析：

```scala
def sayHello = Action(parse.xml) { request =>
  (request.body \\ "name" headOption).map(_.text).map { name =>
    Ok("Hello " + name)
  }.getOrElse {
    BadRequest("Missing parameter [name]")
  }
}
```

```
注意：在使用XML内容解析器时，request.body本身就是一个有效的NodeSeq实例
```

在命令行中可以使用**cURL**测试XML请求处理接口：

```
curl 
  --header "Content-type: application/xml" 
  --request POST 
  --data '<name>Guillaume</name>' 
  http://localhost:9000/sayHello
```

返回的内容是：

```
HTTP/1.1 200 OK
Content-Type: text/plain; charset=utf-8
Content-Length: 15

Hello Guillaume
```

## 提供XML响应
在之前的例子中，我们展示了如何使用Play框架处理XML请求，但是响应的格式是`text/plain`。下面的代码展示了如何返回一个XML格式的HTTP响应。

```scala
def sayHello = Action(parse.xml) { request =>
  (request.body \\ "name" headOption).map(_.text).map { name =>
    Ok(<message status="OK">Hello {name}</message>)
  }.getOrElse {
    BadRequest(<message status="KO">Missing parameter [name]</message>)
  }
}
```

然后还是使用**cURL**命令测试，返回的内容如下：

```
HTTP/1.1 200 OK
Content-Type: application/xml; charset=utf-8
Content-Length: 46

<message status="OK">Hello Guillaume</message>
```