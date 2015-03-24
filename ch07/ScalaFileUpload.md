# Direct upload and multipart/form-data

##在表单中使用multipart/form-data上传文件

在Web应用中上传文件的标准方法是使用以`multipart/form-data`编码的表单，这样能让附件数据与标准表单数据混合在一起。请注意：提交表单时只能用POST方法而不能用GET。

首先构建一个HTML表单：

```
@helper.form(action = routes.Application.upload, 'enctype -> "multipart/form-data") {
    <input type="file" name="picture">    
    <p>
        <input type="submit">
    </p>    
}
```

然后用利用`multipartFormData`body parser来定义一个`upload`动作：

```scala
def upload = Action(parse.multipartFormData) { request =>
  request.body.file("picture").map { picture =>
    import java.io.File
    val filename = picture.filename
    val contentType = picture.contentType
    picture.ref.moveTo(new File(s"/tmp/picture/$filename"))
    Ok("File uploaded")
  }.getOrElse {
    Redirect(routes.Application.index).flashing(
      "error" -> "Missing file")
  }
}
```

`ref` 属性用于对`TemporaryFile`进行说明。这是`mutipartFormData`提取器处理文件上传的默认方式。


**注意:** 你也可以使用`anyContent`body parser，将其作为`request.body.asMultipartFormData`来检索。


最后，添加一个POST路由

```scala
POST  /          controllers.Application.upload()
```

##直接文件上传 

另一种上传文件的方法是在表单中使用Ajax异步上传。在这种情况下请求内容不再是`multipart/form-data`编码了，而仅有包含文件内容本身。

这时我们可以使用一个body parser将请求内容存入文件。比如， 我们使用 `temporaryFile` body parser：

```scala
def upload = Action(parse.temporaryFile) { request =>
  request.body.moveTo(new File("/tmp/picture/uploaded"))
  Ok("File uploaded")
}
```

##自定义body parser

如果你不想经过临时文件缓存而是直接处理上传的文件，你可以自己写一个`BodyParser`来决定怎么处理这些大块的数据。

如果你想使用`multipart/form-data`编码，你仍可以使用默认的`mutipartFormData`提取器，你要做的就是自己写一个`PartHandler[FilePart[A]]`来接收部分headers，然后提供一个`Iteratee[Array[Byte], FilePart[A]]`来生成正确的 `FilePart`。

**下一节：** [Accessing an SQL database](../ch8/ScalaDatabase.md)
