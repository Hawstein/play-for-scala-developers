# Content negotiation

内容协商（Content negotiation）这种机制使得将相同的资源（URI）提供为不同的表示这件事成为可能。这一点非常有用，比如说，在写 Web 服务的时候，支持几种不同的输出格式（XML，Json等）。服务器端驱动的协商是通过使用 `Accept*` 请求报头（header）来做的。你可以在[这里](http://www.w3.org/Protocols/rfc2616/rfc2616-sec12.html)找到更多关于内容协商的信息。

## 语言

你可以通过 `play.api.mvc.RequestHeader#acceptLanguages` 方法来获取针对一个请求的可接受语言列表，该方法从 `Accept-Language` 报头获取这些语言并根据它们的品质值来排序。Play 在 `play.api.mvc.Controller#lang` 方法中使用它，该方法为你的 action 提供了一个隐式的 `play.api.i18n.Lang` 值，因此它会自动选择最可能的语言（如果你的应用支持的话，否则会使用你应用的默认语言）。

## 内容

与上面相似，`play.api.mvc.RequestHeader#acceptedTypes` 方法给出针对一个请求的可接受结果的 MIME 类型列表。该方法从 `Accept` 请求报头获取这些 MIME 类型并根据它们的品质因子进行排序。

事实上，`Accept` 报头并不是真的包含 MIME 类型，而是媒体种类（比如一个请求如果接受的是所有文本结果，那媒体种类可设置为 `text/*`。而 `*/*` 表示所有类型的结果都是可接受的。）。控制器（Controller）提供了一个高级方法 `render` 来帮助你处理媒体种类。例如，考虑以下的 action 定义：

```scala
val list = Action { implicit request =>
  val items = Item.findAll
  render {
    case Accepts.Html() => Ok(views.html.list(items))
    case Accepts.Json() => Ok(Json.toJson(items))
  }
}
```

`Accepts.Html()` 和 `Accepts.Json()` 是两个提取器（extractor），用于测试提供的媒体种类是 `text/html` 还是 `application/json`。`render` 方法接受一个类型为 `play.api.http.MediaRange => play.api.mvc.Result` 的部分函数（partial function）作为参数，并按照优先顺序将它应用在 `Accept` 报头中的每个媒体种类，如果所有可接受的媒体种类都不被你的函数支持，那么会返回一个 `NotAcceptable` 结果。

例如，如果一个客户端发出的请求有如下的 `Accept` 报头：` */*;q=0.5,application/json`，意味着它接受任意类型的结果，但更倾向于要 JSON 类型的，上面那段代码就会给它返回 JSON 类型的结果。如果另一个客户端发出的请求的 `Accept` 报头是 `application/xml`，这意味着它只接受 XML 的结果，上述代码会返回 `NotAcceptable`。

## 请求提取器（Request extractors）

参见 API 文档中 `play.api.mvc.AcceptExtractors.Accepts` 对象，了解 Play 在 `render` 方法中所支持的 MIME 类型。使用 `play.api.mvc.Accepting` case 类，你可以很容易地为给定的 MIME 类型创建一个你自己的提取器。例如，下面的代码创建了一个提取器，用于检查媒体类型是否配置 `audio/mp3` MIME 类型。

```scala
val AcceptsMp3 = Accepting("audio/mp3")
render {
  case AcceptsMp3() => ???
}
```
