# Connecting to OpenID services

OpenID 是一种使用一个账号访问多个服务的协议。作为一个 web 开发者，你可以使用 OpenID 让用户用他们已有的账号（例如 Google 账号）来进行登录。在企业中，你可以使用 OpenID 来连接公司的 SSO 服务器。

## OpenID 工作流一览

* 用户给你他的 OpenID（一个 URL）。
* 你的服务器检查 URL 后面的内容，然后产生一个 URL，并将用户重定向到那。
* 用户在他的 OpenID 提供商那确认授权，然后重定向回你的服务器。
* 你的服务器收到重定向返回的信息，并与 OpenID 提供商检查信息是否正确。

如果你的用户使用的都是同一个 OpenID 提供商，那么可以忽略第一步（例如你决定全部都使用 Google 账号来登录）。

## 用法

使用 OpenID，首先需要把 `ws` 库添加到你的 `build.sbt` 文件中：

```scala
libraryDependencies ++= Seq(
  ws
)
```


## 在 Play 中使用 OpenID

OpenID API 有两个重要的函数：

* `OpenID.redirectURL` 计算用户需要重定向的 URL，这个过程包括异步地获取用户的 OpenID 页面，这就是为什么它返回的是 `Future[String]`。如果 OpenID 是无效的，返回的 `Future` 就会失败。
* `OpenID.verifiedId` 需要一个隐式的 `Request`，并且检查它来建立用户的信息，包含他验证过的 OpenID。它会异步地请求一下 OpenID 服务器以确认信息的真实性，这也是为什么它返回的是一个 `Future[UserInfo]`。如果信息不正确或者服务器的检查结果不正确（例如重定向 URL 是伪造的），返回的 `Future` 就会失败。

如果 `Future` 失败，你可以定义一个回退将用户重定向回登录页面或者返回 `BadRequest`。

下面请看一个用例：

```scala
def login = Action {
  Ok(views.html.login())
}

def loginPost = Action.async { implicit request =>
  Form(single(
    "openid" -> nonEmptyText
  )).bindFromRequest.fold(
    { error =>
      Logger.info("bad request " + error.toString)
      Future.successful(BadRequest(error.toString))
    },
    { openId =>
      OpenID.redirectURL(openId, routes.Application.openIDCallback.absoluteURL())
        .map(url => Redirect(url))
        .recover { case t: Throwable => Redirect(routes.Application.login) }
    }
  )
}

def openIDCallback = Action.async { implicit request =>
  OpenID.verifiedId.map(info => Ok(info.id + "\n" + info.attributes))
    .recover {
      case t: Throwable =>
      // Here you should look at the error, and give feedback to the user
      Redirect(routes.Application.login)
    }
}
```

## 扩展属性

用户的 OpenID 给你的是他的身份标识，该协议也支持获取一些[扩展属性](http://openid.net/specs/openid-attribute-exchange-1_0.html)，如 Email 地址，名或者姓。

你可以从 OpenID 服务器请求一些可选属性或必需属性。要求提供某些必需属性意味着如果用户没有提供的话，他将无法登录你的服务。

你在重定向 URL 中请求扩展属性：

```scala
OpenID.redirectURL(
    openid,
    routes.Application.openIDCallback.absoluteURL(),
    Seq("email" -> "http://schema.openid.net/contact/email")
)
```

这样一来，OpenID 服务器提供的 `UserInfo` 中就会有这个属性了。
