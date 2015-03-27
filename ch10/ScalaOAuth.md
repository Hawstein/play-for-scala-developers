# Accessing resources protected by OAuth

[OAuth](http://oauth.net/) 是发布受保护数据并与之交互的一种简单方式。它是授与你访问权的更安全可靠的方式。例如，你可以通过它访问你在 Twitter 上的用户数据。

OAuth 有 2 个非常不同的版本：[OAuth 1.0](http://tools.ietf.org/html/rfc5849) 和 [OAuth 2.0](http://oauth.net/2/)。版本 2 非常简单，无需库和 helper 的支持。因此 Play 只提供 OAuth 1.0 的支持。

## 用法

想用 OAuth，首先需要把 `ws` 库添加到你的 `build.sbt` 文件中：

```scala
libraryDependencies ++= Seq(
  ws
)
```

## 需要的信息

OAuth 需要你将你的应用注册到服务提供方。确保检查了你提供的回调 URL，因为如果回调 URL 不匹配的话，服务提供方可能会拒绝你的调用。本地使用时，你可以在 `/etc/hosts` 中为本机伪造一个域名。

服务提供方会给你：

* 应用 ID
* 密钥
* 请求令牌 URL（Request Token URL）
* 访问令牌 URL（Access Token URL）
* 授权 URL

## 验证流程

大部分事情都由 Play 的库完成。

* 从服务器获取请求令牌（服务器之间的调用）。
* 将用户重定向到服务提供方，在那里他会授权你的应用可以使用他的数据。
* 服务提供方会将用户重定向回去，并给你一个检验器。
* 有了检验器，你就可以将请求令牌换成访问令牌（服务器之间的调用）。

现在你可以用访问令牌去访问受保护的用户数据了。

## 例子

```scala
object Twitter extends Controller {

  val KEY = ConsumerKey("xxxxx", "xxxxx")

  val TWITTER = OAuth(ServiceInfo(
    "https://api.twitter.com/oauth/request_token",
    "https://api.twitter.com/oauth/access_token",
    "https://api.twitter.com/oauth/authorize", KEY),
    true)

  def authenticate = Action { request =>
    request.getQueryString("oauth_verifier").map { verifier =>
      val tokenPair = sessionTokenPair(request).get
      // We got the verifier; now get the access token, store it and back to index
      TWITTER.retrieveAccessToken(tokenPair, verifier) match {
        case Right(t) => {
          // We received the authorized tokens in the OAuth object - store it before we proceed
          Redirect(routes.Application.index).withSession("token" -> t.token, "secret" -> t.secret)
        }
        case Left(e) => throw e
      }
    }.getOrElse(
      TWITTER.retrieveRequestToken("http://localhost:9000/auth") match {
        case Right(t) => {
          // We received the unauthorized tokens in the OAuth object - store it before we proceed
          Redirect(TWITTER.redirectUrl(t.token)).withSession("token" -> t.token, "secret" -> t.secret)
        }
        case Left(e) => throw e
      })
  }

  def sessionTokenPair(implicit request: RequestHeader): Option[RequestToken] = {
    for {
      token <- request.session.get("token")
      secret <- request.session.get("secret")
    } yield {
      RequestToken(token, secret)
    }
  }
}
```

```scala
object Application extends Controller {

  def timeline = Action.async { implicit request =>
    Twitter.sessionTokenPair match {
      case Some(credentials) => {
        WS.url("https://api.twitter.com/1.1/statuses/home_timeline.json")
          .sign(OAuthCalculator(Twitter.KEY, credentials))
          .get
          .map(result => Ok(result.json))
      }
      case _ => Future.successful(Redirect(routes.Twitter.authenticate))
    }
  }
}
```
