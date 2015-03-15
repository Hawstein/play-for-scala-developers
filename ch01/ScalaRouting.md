# HTTP routing

## 内建 HTTP 路由

路由是一个负责将传入的 HTTP 请求转换为 Action 的组件。

一个 HTTP请求通常被 MVC 框架视为一个事件（event）。这个事件包含了两个主要信息：

* 请求路径（例如：`/clients/1542`，`/photos/list`），其中包括了查询串（如 ?page=1&max=3）
* HTTP 方法（例如：GET，POST 等）

路由定义在了 `conf/routes` 文件里，该文件会被编译。也就是说你可以直接在你的浏览器中看到这样的路由错误：

![Route Error in Browser](https://www.playframework.com/documentation/2.3.x/resources/manual/scalaGuide/main/http/images/routesError.png)

## 路由文件的语法

Play 的路由使用 `conf/routes` 作为配置文件。这个文件列出了该应用需要的所有路由。每个路由都包含了一个 HTTP 方法和一个 URI 模式，两者被关联到一个 Action 生成器。

让我们来看下路由的定义：

```scala
GET   /clients/:id           controllers.Clients.show(id: Long)
```

每个路由都以 HTTP 方法开头，后面跟着一个 URI 模式。最后是一个方法的调用。

你也可以使用 `#` 字符添加注释：

```scala
# 显示一个 client
GET   /clients/:id           controllers.Clients.show(id: Long)
```

## HTTP 方法

HTTP 方法可以是任何 HTTP 支持的有效方法（`GET`，`POST`，`PUT`，`DELETE`，`HEAD`）。

## URI 模式

URI 模式定义了路由的请求路径。其中部分的请求路径可以是动态的。

### 静态路径

例如，为了能够准确地匹配 `GET /clients/all` 请求，你可以这样定义路由：

```scala
GET   /clients/all           controllers.Clients.list()
```

### 动态匹配

如果你想要在定义的路由中根据 ID 获取一个 client，则需要添加一个动态部分：

```scala
GET   /clients/:id           controllers.Clients.show(id: Long)
```

> 注意：一个 URI 模式可以有多个动态的部分。

动态部分的默认匹配策略是通过正则表达式 `[^/]+` 定义的，这意味着任何被定义为 `:id` 的动态部分只会匹配一个 URI。

### 跨 / 字符的动态匹配

如果你想要一个动态部分能够匹配多个由 `/` 分隔开的 URI 路径，你可以用 `*id` 来定义，此时匹配的正则表达式则为 `.+`：

```scala
GET   /file/*name            controllers.Application.download(name)
```

对于像 `GET /files/images/logo.png` 这样的请求，动态部分 `name` 匹配到的是 `images/logo.png`。

### 使用自定义正则表达式做动态匹配

你也可以为动态部分自定义正则表达式，语法为 `$id<regex>`：

```scala
GET   /items/$id<[0-9]+>     controllers.Items.show(id: Long)
```

## 调用 Action 生成器方法

路由定义的最后一部分是个方法的调用。这里必须定义一个返回 `play.api.mvc.Action` 的合法方法，一般是一个控制器 action 方法。

如果该方法不接收任何参数，则只需给出完整的方法名：

```scala
GET   /                      controllers.Application.homePage()
```

如果这个 action 方法定义了参数，Play 则会在请求 URI 中查找所有参数。从 URI 路径自身查找，或是在查询串里查找：

```scala
# 从路径中提取参数 page
GET   /:page                 controllers.Application.show(page)
```

```scala
# 从查询串中提取参数 page
GET   /                      controllers.Application.show(page)
```

这是定义在 `controllers.Application` 里的 `show` 方法：

```scala
def show(page: String) = Action {
  loadContentFromDatabase(page).map { htmlContent =>
    Ok(htmlContent).as("text/html")
  }.getOrElse(NotFound)
}
```

### 参数类型

对于类型为 `String` 的参数来说，可以不写参数类型。如果你想要 Play 帮你把传入的参数转换为特定的 Scala 类型，则必须显式声明参数类型：

```scala
GET   /clients/:id           controllers.Clients.show(id: Long)
```

同样的，`controllers.Clients` 里的 `show` 方法也需要做相应更改：

```scala
def show(id: Long) = Action {
  Client.findById(id).map { client =>
    Ok(views.html.Clients.display(client))
  }.getOrElse(NotFound)
}
```

### 设定参数固定值

有时候，你会想给参数设置一个固定值：

```scala
# 从路径中提取参数 page，或是为 / 设置固定值
GET   /                     controllers.Application.show(page = "home")
GET   /:page                controllers.Application.show(page)
```

### 设置参数默认值

你也可以给参数提供一个默认值，当传入的请求中找不到任何相关值时，就使用默认参数：

```scala
# 分页链接，如 /clients?page=3
GET   /clients              controllers.Clients.list(page: Int ?= 1)
```

### 可选参数

同样的，你可以指定一个可选参数，可选参数可以不出现在请求中：

```scala
# 参数 version 是可选的。如 /api/list-all?version=3.0
GET   /api/list-all         controllers.Api.list(version: Option[String])
```

## 路由优先级

多个路由可能会匹配到同一个请求。如果出现了类似的冲突情况，第一个定义的路由（以定义顺序为准）会被启用。

## 反向路由

路由同样可以通过 Scala 的方法调用来生成 URL。这样做能够将所有的 URI 模式定义在一个配置文件中，让你在重构应用时更有把握。

Play 的路由会为路由配置文件里定义的每个控制器，在 `routes` 包中生成一个「反向控制器」，其中包含了同样的 action 方法和签名，不过返回值类型为 `play.api.mvc.Call` 而非 `play.api.mvc.Action`。

`play.api.mvc.Call` 定义了一个 HTTP 调用，并且提供了 HTTP 请求方法和 URI。

例如，如果你创建了这样一个控制器：

```scala
package controllers

import play.api._
import play.api.mvc._

object Application extends Controller {

  def hello(name: String) = Action {
    Ok("Hello " + name + "!")
  }
}
```

并且在 `conf/routes` 中设置了该方法：

```scala
# Hello action
GET   /hello/:name          controllers.Application.hello(name)
```

接着你就可以调用 `controllers.routes.Application` 的 `hello` action 方法，反向得到相应的 URL：

```scala
// 重定向到 /hello/Bob
def helloBob = Action {
  Redirect(routes.Application.hello("Bob"))
}
```
