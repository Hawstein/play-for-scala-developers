# Messages externalisation and i18n

##指定应用支持语言
你可以按**ISO 639-2** 标准语言代码或者**ISO 3166-1 alpha-2**标准国家代码来指定应用支持语言，譬如`fr`或者`en-US`。

语言指定信息存放在应用的 `conf/application.conf`文件中：

```scala
application.langs="en,en-US,fr"
```

##外部化信息（Externalizing messages）

你可以在`conf/messages.xxx`文件中外部化（externalize）信息。

`conf/messages`文件默认适配所有语言。你可以添加你所指定语言所对应的文件，譬如`conf/messages.fr`或`conf/messages.en-US`。

你可以利用`play.api.i18n.Messages`对象检出(retrieve)信息：

```scala
val title = Messages("home.title")
```

所有国际化相关的API都隐式调用`play.api.i18n.Lang`参数，其中参数的值根据当前环境给出。你可以自己显式地指定：

```scala
val title = Messages("home.title")(Lang("fr"))
```

**注意:**如果你在当前范围发出一个隐式请求，程序会根据头部的 `Accept-Language`从应用支持语言匹配一个，然后隐式地返回给你一个`Lang`参数。你应当像这样在模板文件中添加`Lang`变量： `@()(implicit lang: Lang)`。


##消息格式

消息使用 `java.text.MessageFormat`库来定义格式，譬如你的消息这样定义：

```
files.summary=The disk {1} contains {0} file(s).
```
那么你可以这样指定参数：

```
Messages("files.summary", d.files.length, d.name)
```

##关于单引号


由于我们使用`java.text.MessageFormat`来定义消息格式，所以要注意的是单引号会被作为转意字符。

譬如你定义了如下一段消息：

比如，你可能会定义这样一段信息：

```scala
info.error=You aren''t logged in!
```

```scala
example.formatting=When using MessageFormat, '''{0}''' is replaced with the first parameter.
```

你应期望如下结果：

```scala
Messages("info.error") == "You aren't logged in!"
```

```scala
Messages("example.formatting") == "When using MessageFormat, '{0}' is replaced with the first parameter."
```

##从HTTP请求中提取支持语言

你可以（像这样）从给定的HTTP请求提取出支持语言：

```
def index = Action { request =>
  Ok("Languages: " + request.acceptLanguages.map(_.code).mkString(", "))
}
```

**下一节：** [The application Global object](../ch13/ScalaGlobal.md)