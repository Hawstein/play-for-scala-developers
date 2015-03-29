# The Logging API

在你的应用中使用日志有助于监控、调试、错误跟踪以及商业智能分析。Play 提供了日志 API，可以通过 `Logger` 对象来使用，Play 使用 [Logback](http://logback.qos.ch/) 作为日志引擎。

## 日志体系结构

日志 API 提供了一组构件来帮助你实现高效的日志记录策略。

### Logger

你的应用可以定义 `Logger` 实例来发送日志信息。每个 `Logger` 有一个名字，它会出现在日志信息中，该名字是用于配置层级关系的。

Logger 根据它们的名字会有一个层级继承结构。如果一个 logger 的名字后面加个点后是另外一个 logger 名字的前缀，那么我们就说第一个 logger 是第二个 logger 的祖先。例如，名为 “com.foo” 的 logger 是名为 “com.foo.bar.Baz” 的 logger 的祖先。所有的 logger 都继承自根 logger。Logger 继承机制的一个好处就是，如果你想配置一组 logger，那你只需要配置它们的共同祖先即可。

Play 应用有一个默认的 logger，叫 “application”。当然，你也可以创建你自己的 logger。Play 库使用的 logger 名叫 “play”，还有一些第三方库会自己命名 logger。

### 日志级别

日志级别是用来区分日志消息的严重性的。当你写日志请求时，你需要指定严重程度，它会出现在产生的日志消息里。

以下是一组可用的日志级别，以严重性的降序列出。

* `OFF` - 关闭日志，不作为消息分类。
* `ERROR` - 运行时错误，或不可预料的情况。
* `WARN` - 用在一些不建议使用的 API 提示上，或是其它一些不可预料但又算不上错误的运行时情况。
* `INFO` - 用在你感兴趣的运行时事件上，比如应用启动和关闭时。
* `DEBUG` - 系统工作流程上的一些细节信息。
* `TRACE` - 大部分细节信息。

日志级别是用来配置 logger 和 appender 打印日志的阈值的。例如，如果你把 logger 的日志级别设置为 `INFO`，则会打印 `INFO` 及严重性更高的日志（`INFO`，`WARN` 和 `ERROR`），而忽略严重性比它低的日志（`DEBUG`，`TRACE`）。使用 `OFF` 会忽略所有的日志请求。

## Appenders

日志 API 允许日志请求打印到一个或多个叫做 appender 的目标输出。appender 在配置中指定，可选的有：控制台，文件，数据库或其它输出。

appender 和 logger 组合可以帮助你路由及过滤日志信息。例如，你可以使用一个 appender 打印有用的信息用于分析，用另一个 appender 打印错误信息，用于运维团队的监控。

> 注意：想了解更多关于日志体系结构的信息，请移步：[Logback 文档](http://logback.qos.ch/manual/architecture.html)。

## 使用 Logger

首先导入 `Logger` 类及其伴生对象：

```scala
import play.api.Logger
```

### 默认 Logger

`Logger` 对象是默认的 logger，名为 “application”。你可以使用它来打印日志：

```scala
/ Log some debug info
Logger.debug("Attempting risky calculation.")

try {
  val result = riskyCalculation

  // Log result if successful
  Logger.debug(s"Result=$result")
} catch {
  case t: Throwable => {
    // Log error with message and Throwable.
    Logger.error("Exception with riskyCalculation", t)
  }
}
```

使用 Play 默认的日志配置，上面的语句会产生类似下面的控制台输出：

```scala
[debug] application - Attempting risky calculation.
[error] application - Exception with riskyCalculation
java.lang.ArithmeticException: / by zero
    at controllers.Application$.controllers$Application$$riskyCalculation(Application.scala:32) ~[classes/:na]
    at controllers.Application$$anonfun$test$1.apply(Application.scala:18) [classes/:na]
    at controllers.Application$$anonfun$test$1.apply(Application.scala:12) [classes/:na]
    at play.api.mvc.ActionBuilder$$anonfun$apply$17.apply(Action.scala:390) [play_2.10-2.3-M1.jar:2.3-M1]
    at play.api.mvc.ActionBuilder$$anonfun$apply$17.apply(Action.scala:390) [play_2.10-2.3-M1.jar:2.3-M1]
```

输出包含了日志级别（debug，error），logger 名称（application），消息，如果抛出异常的话，还会有堆栈跟踪信息。

### 创建你自己的 logger

也许你想在所有地方都用默认 logger，但这种使用方式并不好。用一个不同的名字创建你自己的 logger，这样配置起来会更灵活，可以过滤你的日志输出，并且精确知道日志的来源。

你可以使用 `Logger.apply` 工厂方法来创建一个新 logger，它需要一个名字作为参数：

```scala
val accessLogger: Logger = Logger("access")
```

一个常用的策略是，为每个类配置一个不同的 logger，并且用类名来命名它。你可以使用另一个工厂方法，它接受一个类作为参数：

```scala
val logger: Logger = Logger(this.getClass())
```

### 日志模式

高效地使用 logger 可以帮助你达到许多目标：

```scala
import scala.concurrent.Future
import play.api.Logger
import play.api.mvc._

trait AccessLogging {

  val accessLogger = Logger("access")

  object AccessLoggingAction extends ActionBuilder[Request] {

    def invokeBlock[A](request: Request[A], block: (Request[A]) => Future[Result]) = {
      accessLogger.info(s"method=${request.method} uri=${request.uri} remote-address=${request.remoteAddress}")
      block(request)
    }
  }
}

object Application extends Controller with AccessLogging {

  val logger = Logger(this.getClass())

  def index = AccessLoggingAction {
    try {
      val result = riskyCalculation
      Ok(s"Result=$result")
    } catch {
      case t: Throwable => {
        logger.error("Exception with riskyCalculation", t)
        InternalServerError("Error in calculation: " + t.getMessage())
      }
    }
  }
}
```

这个例子使用了 [action 组合](https://www.playframework.com/documentation/2.3.x/ScalaActionsComposition) 来定义 `AccessLoggingAction`，它会把日志打印到一个名为 “access” 的 logger。`Application` 控制器使用了这个 action，并且它使用了自己的 logger 来记录该应用里的事件。在配置中，你就可以将这些 logger 路由到不同的 appender，例如访问日志和应用日志。

如果你只想为指定 action 打印请求数据，那么上面的设计就足够了。想要打印所有的请求，则最好使用[过滤器](https://www.playframework.com/documentation/2.3.x/ScalaInterceptors)：

```scala
import scala.concurrent.ExecutionContext.Implicits.global
import scala.concurrent.Future
import play.api.Logger
import play.api.mvc._
import play.api._

object AccessLoggingFilter extends Filter {

  val accessLogger = Logger("access")

  def apply(next: (RequestHeader) => Future[Result])(request: RequestHeader): Future[Result] = {
    val resultFuture = next(request)

    resultFuture.foreach(result => {
      val msg = s"method=${request.method} uri=${request.uri} remote-address=${request.remoteAddress}" +
        s" status=${result.header.status}";
      accessLogger.info(msg)
    })

    resultFuture
  }
}

object Global extends WithFilters(AccessLoggingFilter) {

  override def onStart(app: Application) {
    Logger.info("Application has started")
  }

  override def onStop(app: Application) {
    Logger.info("Application has stopped")
  }
}
```

上面的代码中，我们在打印日志请求里加入了响应状态，当 `Future[Result]` 完成时会打印输出。注意，对于像应用开启和应用关闭这种事件，用全局对象来使用默认 logger 是一种明智的选择。

## 配置

详见[日志配置](https://www.playframework.com/documentation/2.3.x/SettingsLogger)。
