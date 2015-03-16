# Comet sockets

## 使用分块响应来创建 comet sockets

**分块响应**（chunked responses）的一个好处是可以用来创建 comet sockets。comet socket 是一块只包含 `<script>` 元素的 `text/html` 响应。在每一个块中，我们都写入一个 `<script>` 标签，这样一来，它会被浏览器立即执行。通过这种方式，我们可以实时地从服务器发送各种事件到浏览器：将每条信息包在一个 `<script>` 标签中（它会调用一个 JavaScript 回调函数），然后将它写到响应块里。

让我们通过以下例子来验证上面的概念：下面的枚举器（enumerator）产生了 3 个 `<script>` 标签，每个都会调用浏览器的 `console.log` JavaScript 函数：

```scala
def comet = Action {
  val events = Enumerator(
    """<script>console.log('kiki')</script>""",
    """<script>console.log('foo')</script>""",
    """<script>console.log('bar')</script>"""
  )
  Ok.chunked(events).as(HTML)
}
```

如果你在浏览器中运行这个 action，你会看到 3 个事件在浏览器控制台打印日志。

对于上述例子，我们可以用一种更好的方法来做，即使用 `play.api.libs.iteratee.Enumeratee` 这个适配器，它能将 `Enumerator[A]` 转成 `Enumerator[B]`。下面我们使用它将标准信息包到 `<script>` 标签里：

```scala
import play.twirl.api.Html

// Transform a String message into an Html script tag
val toCometMessage = Enumeratee.map[String] { data =>
  Html("""<script>console.log('""" + data + """')</script>""")
}

def comet = Action {
  val events = Enumerator("kiki", "foo", "bar")
  Ok.chunked(events &> toCometMessage)
}
```

> 注意：`events &> toCometMessage` 只是 `events.through(toCometMessage)` 的另一种表述方式。

## 使用 helper：`play.api.libs.Comet`

Play 提供了相应的 helper 来处理 comet 分块流，效果与上面所写的方法基本一样。

> 注意：事实上，它做得更多。比如为了浏览器的兼容性，它提供了一个初始的空的缓冲数据。此外，它支持字符串和 JSON 格式的信息。你还可以通过类型类来扩展它，使它支持更多信息类型。

前面的例子可以重写如下：

```scala
def comet = Action {
  val events = Enumerator("kiki", "foo", "bar")
  Ok.chunked(events &> Comet(callback = "console.log"))
}
```

## iframe 流技术

写 comet socket 的标准方法是在 HTML `iframe` 中加载无限的分块 comet 响应，并指定一个回调函数来调用父级 frame：

```scala
def comet = Action {
  val events = Enumerator("kiki", "foo", "bar")
  Ok.chunked(events &> Comet(callback = "parent.cometMessage"))
}
```

HTML 页面如下：

```html
<script type="text/javascript">
  var cometMessage = function(event) {
    console.log('Received event: ' + event)
  }
</script>

<iframe src="/comet"></iframe>
```

> 注意：本章概念可参考：`http://en.wikipedia.org/wiki/Comet_(programming)`
