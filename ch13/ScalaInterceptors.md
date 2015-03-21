# 侦听请求

## 使用过滤器

你可以使用过滤组件侦听来自应用的请求，转换请求和响应。过滤器能很好地实现你的应用中的横切关注点（[Cross-cutting concern](http://en.wikipedia.org/wiki/Cross-cutting_concern)）。你可以通过扩展 `Filter` 特质来创建一个过滤器，然后将其加入 `Global` 对象。
以下示例创建了一个记录所有动作结果的访问记录过滤器：

```scala
import play.api.Logger
import play.api.mvc._

object AccessLoggingFilter extends Filter {
  def apply(next: (RequestHeader) => Future[Result])(request: RequestHeader): Future[Result] = {
    val result = next(request)
    Logger.info(request + "\n\t => " + result)
    result
  }
}

object Global extends WithFilters(AccessLoggingFilter)
```

扩展了 `WithFilters` 类的 `Global` 对象让你可以通过传递多个过滤器来组成过滤链。

> 注意：`WithFilters` 现在扩展了 `GlobalSettings` 特质。

另一个体现过滤器用处的示例，在调用某个动作前检查授权：

```scala
object AuthorizedFilter {
  def apply(actionNames: String*) = new AuthorizedFilter(actionNames)
}

class AuthorizedFilter(actionNames: Seq[String]) extends Filter {

  def apply(next: (RequestHeader) => Future[Result])(request: RequestHeader): Future[Result] = {
    if(authorizationRequired(request)) {
      /* do the auth stuff here */
      println("auth required")
      next(request)
    }
    else next(request)
  }

  private def authorizationRequired(request: RequestHeader) = {
    val actionInvoked: String = request.tags.getOrElse(play.api.Routes.ROUTE_ACTION_METHOD, "")
    actionNames.contains(actionInvoked)
  }


}


object Global extends WithFilters(AuthorizedFilter("editProfile", "buy", "sell")) with GlobalSettings {}
```

> 提示：`RequestHeader.tags` 提供了大量关于调用动作的路由的有用信息。

## 重写 onRouteRequest

`Global` 对象的另一个重要方面是它提供了一个监听请求和在请求被分派给一个动作前执行业务逻辑的方法。

> 提示：这种关联也可以被用来劫持请求，让开发者插入他们自己的请求路由机制。

让我们来看看这在实际中是如何运作的：

```scala
import play.api._
import play.api.mvc._

// Note: this is in the default package.
object Global extends GlobalSettings {

  override def onRouteRequest(request: RequestHeader): Option[Handler] = {
    println("executed before every request:" + request.toString)
    super.onRouteRequest(request)
  }

}
```

也可以使用 `Action 组合`来监听某个特定 Action 方法。