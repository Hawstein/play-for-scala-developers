# Protecting against CSRF

跨站请求伪造（CSRF）是一种安全漏洞。攻击者欺骗受害者浏览器，并使用受害者的会话发送请求。由于每个请求中都含有会话令牌（session token），如果攻击者能够强制使用受害者的浏览器发送请求，也就相当于受害者在以用户的名义发送请求。

最好能先熟悉一下 CSRF ，哪些攻击手段属于 CSRF ，哪些不是。建议参考 [OWASP 的相关内容](https://www.owasp.org/index.php/Cross-Site_Request_Forgery_%28CSRF%29)。

简单的说，攻击者能够强迫受害者的浏览器发送以下请求：

* 所有的 `GET` 请求
* 内容类型为 `application/x-www-form-urlencoded`，`multipart/form-data` 和 `text/plain` 的 `POST` 请求

攻击者不能：

* 强迫浏览器使用 `PUT` 和 `DELETE` 请求
* 强迫浏览器发送其他内容类型的请求，如 `application/json`
* 强迫浏览器发送新的 cookie ，而不是服务器已经设置了的 cookie
* 强迫浏览器设置任意报头，而不是浏览器通常会在请求中添加的那些报头

由于 `GET` 请求是无法更改的，应用中使用该请求不会有任何危险。所以唯一需要防御 CSRF 攻击的是带有如上提到的内容类型的 `POST` 请求。

## Play 的 CSRF 防御

Play 支持了多种方法来验证是否为 CSRF 请求。最主要的机制是 CSRF 令牌（token）。该令牌需设置在查询字符串或是每个提交的表单正文中，并且还要设置于用户会话中。Play 之后会验证这两个令牌是否存在并匹配。

为了能够简单的防御那些非浏览器请求，例如通过 AJAX 发送的请求，Play 同样支持以下几种：

* 如果包头中有 `X-Requested-With`，则 Play 会认为该请求安全。很多流行的 Javascript 库都会在请求中加入 `X-Requested-With`，比如 jQuery。
* 如果 `Csrf-Token` 报头的值为 `nocheck`，或是一个有效的 CSRF 令牌，则 Play 会认为该请求安全。

## 应用全局 CSRF 过滤

Play 提供了全局 CSRF 过滤，可以将其应用于所有请求。这是给应用添加 CSRF 防御最简单的方法。在你项目中的 `build.sbt` 内添加  Play 过滤 helper 依赖，就能启用全局过滤了：

```scala
libraryDependencies += filters
```

现在需要将过滤器添加到 `Global` 对象中：

```scala
import play.api._
import play.api.mvc._
import play.filters.csrf._

object Global extends WithFilters(CSRFFilter()) with GlobalSettings {
  // ... onStart, onStop etc
}
```

### 获得当前令牌

当前 CSRF 令牌可通过调用 `getToken` 方法获取。该方法接收一个隐式的 `RequestHeader`，所以要确保在作用域中设置该参数。

```scala
import play.filters.csrf.CSRF

val token = CSRF.getToken(request)
```

Play 提供了一些模板 helper 来辅助添加 CSRF 令牌到表单内。第一个是添加到 action URL 的查询字符串中：

```scala
@import helper._

@form(CSRF(routes.ItemsController.save())) {
    ...
}
```

渲染后的表单如下：

```scala
<form method="POST" action="/items?csrfToken=1234567890abcdef">
   ...
</form>
```

如果你不想在查询字符串中设置令牌，Play 还提供了 helper，将 CSRF 令牌作为隐藏域添加到表单中：

```scala
@form(routes.ItemsController.save()) {
    @CSRF.formField
    ...
}
```

渲染后的表单如下：

```scala
<form method="POST" action="/items">
   <input type="hidden" name="csrfToken" value="1234567890abcdef"/>
   ...
</form>
```

所有表单 helper 方法都要求在作用域中设置隐式的令牌或是请求。通常，如果没有的话，需要在你的模板中设置隐式的 `RequestHeader` 参数。

### 在会话中添加 CSRF 令牌

为了保证能在表单中找到 CSRF 令牌并发回客户端，如果传入的请求中没有令牌，全局过滤器会为所有接收 HTML 的 `GET` 请求生成一个新的令牌。

## 基于单个 action 的 CSRF 过滤

有时候应用全局 CSRF 过滤并不合适，比如应用可能需要允许部分跨站表单的提交。有一些并非基于会话的标准，如 OpenID 2.0，需要使用跨站表单提交，或是在服务器到服务器的 RPC 通讯中使用表单提交。

在这样的情况下，Play 提供了两个 action，可供组合到应用的 action 中。

第一个是 `CSRFCheck` action，提供了验证操作。需添加到所有接收已认证会话 POST 表单提交的 action 中：

```scala
import play.api.mvc._
import play.filters.csrf._

def save = CSRFCheck {
  Action { req =>
    // handle body
    Ok
  }
}
```

第二是个 `CSRFAddToken` action，在传入请求没有令牌的情况下会生成一个 CSRF 令牌。需添加到所有渲染表单的 action 中：

```scala
import play.api.mvc._
import play.filters.csrf._

def form = CSRFAddToken {
  Action { implicit req =>
    Ok(views.html.itemsForm())
  }
}
```

更简便的方法是将这些 action 和 Play 的 `ActionBuilder` 一起组合使用：

```scala
import play.api.mvc._
import play.filters.csrf._

object PostAction extends ActionBuilder[Request] {
  def invokeBlock[A](request: Request[A], block: (Request[A]) => Future[Result]) = {
    // authentication code here
    block(request)
  }
  override def composeAction[A](action: Action[A]) = CSRFCheck(action)
}

object GetAction extends ActionBuilder[Request] {
  def invokeBlock[A](request: Request[A], block: (Request[A]) => Future[Result]) = {
    // authentication code here
    block(request)
  }
  override def composeAction[A](action: Action[A]) = CSRFAddToken(action)
}
```

这样可以最大限度的减少在编写 action 时所需的样板代码（boiler plate code）：

```scala
def save = PostAction {
  // handle body
  Ok
}

def form = GetAction { implicit req =>
  Ok(views.html.itemsForm())
}
```

## CSRF 配置选项

以下选项可在 `application.conf` 中配置：

* `csrf.token.name` - 应用于会话和请求正文/查询字符串中的令牌名称。默认为 `csrfToken`。
* `csrf.cookie.name` - 如果配置了该选项，Play 会根据该名称将 CSRF 令牌存储到 cookie，而非会话中。
* `csrf.cookie.secure` - 如果设置了 `csrf.cookie.name`，则 CSRF cookie 是否需要设置安全标志位。默认该值与 `session.secure` 相同。
* `csrf.body.bufferSize` - 为了能在正文中读取令牌，Play 必须缓存正文，并在可能的情况下进行解析。该选项设置了缓存正文时最大缓存大小。默认为 100k。
* `csrf.sign.tokens` - Play 是否使用签名 CSRF 令牌。签名 CSRF 令牌保证了每个请求的令牌值是随机的，这样可以防御 BREACH 攻击。