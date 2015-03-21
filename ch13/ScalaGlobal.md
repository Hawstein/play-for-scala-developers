# 应用全局设置

## Global 对象

你可以在项目中定义一个 `Global` 对象来控制应用的全局设置。这个对象必须在默认（空）包中定义并扩展自 `GlobalSettings`。

``` scala
import play.api._

object Global extends GlobalSettings {

}
```

> 提示：你也可以在 `application.global` 设置字段中指定一个自定义 `GlobalSettings` 实现的类名。

## 关联应用启动和停止事件

你可以重写 `onStart` 和 `onStop` 方法来获取应用生命周期中的事件通知：

```scala
import play.api._

object Global extends GlobalSettings {

  override def onStart(app: Application) {
    Logger.info("Application has started")
  }

  override def onStop(app: Application) {
    Logger.info("Application shutdown...")
  }

}
```

## 提供应用错误讯息页

当你的应用出现一个异常，这将会引发 `onError` 操作。默认使用内部框架的错误讯息页：

```scala
import play.api._
import play.api.mvc._
import play.api.mvc.Results._
import scala.concurrent.Future

object Global extends GlobalSettings {

  override def onError(request: RequestHeader, ex: Throwable) = {
    Future.successful(InternalServerError(
      views.html.errorPage(ex)
    ))
  }

}
```

## 处理丢失的动作和绑定错误

如果框架无法为请求找到一个 `Action`，这将会引发 `onHandlerNotFound` 操作：

```scala
import play.api._
import play.api.mvc._
import play.api.mvc.Results._
import scala.concurrent.Future

object Global extends GlobalSettings {

  override def onHandlerNotFound(request: RequestHeader) = {
    Future.successful(NotFound(
      views.html.notFoundPage(request.path)
    ))
  }

}
```

如果获取的路由无法绑定请求的参数，这将会引发 `onBadRequest` 操作：

```scala
import play.api._
import play.api.mvc._
import play.api.mvc.Results._
import scala.concurrent.Future

object Global extends GlobalSettings {

  override def onBadRequest(request: RequestHeader, error: String) = {
    Future.successful(BadRequest("Bad Request: " + error))
  }

}
```