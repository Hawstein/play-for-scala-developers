# Direct upload and multipart/form-data

##�ڱ���ʹ��multipart/form-data�ϴ��ļ�

��WebӦ�����ϴ��ļ��ı�׼������ʹ����`multipart/form-data`����ı����������ø����������׼�����ݻ����һ����ע�⣺�ύ��ʱֻ����POST������������GET��

���ȹ���һ��HTML����

```
@helper.form(action = routes.Application.upload, 'enctype -> "multipart/form-data") {
    <input type="file" name="picture">    
    <p>
        <input type="submit">
    </p>    
}
```

Ȼ��������`multipartFormData`body parser������һ��`upload`������

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

`ref` �������ڶ�`TemporaryFile`����˵��������`mutipartFormData`��ȡ�������ļ��ϴ���Ĭ�Ϸ�ʽ��


**ע��:** ��Ҳ����ʹ��`anyContent`body parser��������Ϊ`request.body.asMultipartFormData`��������


������һ��POST·��

```scala
POST  /          controllers.Application.upload()
```

##ֱ���ļ��ϴ� 

��һ���ϴ��ļ��ķ������ڱ���ʹ��Ajax�첽�ϴ���������������������ݲ�����`multipart/form-data`�����ˣ������а����ļ����ݱ���

��ʱ���ǿ���ʹ��һ��body parser���������ݴ����ļ������磬 ����ʹ�� `temporaryFile` body parser��

```scala
def upload = Action(parse.temporaryFile) { request =>
  request.body.moveTo(new File("/tmp/picture/uploaded"))
  Ok("File uploaded")
}
```

##�Զ���body parser

����㲻�뾭����ʱ�ļ��������ֱ�Ӵ����ϴ����ļ���������Լ�дһ��`BodyParser`��������ô������Щ�������ݡ�

�������ʹ��`multipart/form-data`���룬���Կ���ʹ��Ĭ�ϵ�`mutipartFormData`��ȡ������Ҫ���ľ����Լ�дһ��`PartHandler[FilePart[A]]`�����ղ���headers��Ȼ���ṩһ��`Iteratee[Array[Byte], FilePart[A]]`��������ȷ�� `FilePart`��

**��һ�ڣ�** [Accessing an SQL database](../ch8/ScalaDatabase.md)
