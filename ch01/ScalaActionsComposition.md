# Actions composition

这一章引入了几种定义通用action的方法

## 自定义action构造器

[之前](ScalaActions.md)，我们已经介绍了几种声明一个action的方法 - 带有请求参数，无请求参数和带有body解析器（body parser）等。事实上还有其他一些方法，我们会在[异步编程](../ch02/ScalaAsync.md)中介绍。

这些构造action的方法实际上都是有由一个命名为`ActionBuilder`的特性（trait）所定义的，而我们用来声明所有action的`Action`对象只不过是这个特性（trait）的一个实例。通过实现自己的`ActionBuilder`，你可以声明一些可重用的action栈，并以此来构建action。

让我们先来看一个简单的日志装饰器例子。在这个例子中，我们会记录每一次对该action的调用。

第一种方式是在`invokeBlock`方法中实现该功能，每个由`ActionBuilder`构建的action都会调用该方法：

```scala
import play.api.mvc._

object LoggingAction extends ActionBuilder[Request] {
  def invokeBlock[A](request: Request[A], block: (Request[A]) => Future[Result]) = {
    Logger.info("Calling action")
    block(request)
  }
}
```

现在我们就可以像使用`Action`一样来使用它了：

```scala
def index = LoggingAction {
  Ok("Hello World")
}
```

`ActionBuilder`提供了其他几种构建action的方式，该方法同样适用于如声明一个自定义body解析器（body parser）等方法：

```scala
def submit = LoggingAction(parse.text) { request =>
  Ok("Got a bory " + request.body.length + " bytes long")
}
```

### 组合action

在大多数的应用中，我们会有多个action构造器，有些用来做各种类型的验证，有些则提供了多种通用功能等。这种情况下，我们不想为每个类型的action构造器都重写日志action，这时就需要定义一种可重用的方式。

可重用的action代码可以通过嵌套action来实现：

```scala
import play.api.mvc._

case class Logging[A](action: Action[A]) extends Action[A] {
  
  def apply(request: Request[A]): Future[Result] = {
    Logger.info("Calling action")
    action(request)
  }
  
  lazy val parser = action.parser
}
```

我们也可以使用`Action`的action构造器来构建，这样就不需要定义我们自己的action类了：

```scala
import play.api.mvc._

def logging[A](action: Action[A]) = Action.async(action.parser) { request =>
  Logger.info("Calling action")
  action(request)
}
```

Action同样可以使用`composeAction`方法混入（mix in）到action构造器中：

```scala
object LoggingAction extends ActionBuilder[Request] {
  def invokeBlock[A](request: Request[A], block: (Request[A]) => Future[Result]) = {
    block(request)
  }
  override def composeAction[A](action: Action[A]) = new Logging(action)
}
```

现在构造器就能像之前那样使用了：

```scala
def index = LoggingAction {
  Ok("Hello World")
}
```

我们也可以不用action构造器来混入（mix in）嵌套action：

```scala
def index = Logging {
  Action {
    Ok("Hello World")
  }
}
```

### 更多复杂的action

到现在为止，我们所演示的action都不会影响传入的请求。我们当然也可以读取并修改传入的请求对象：

```scala
import play.api.mvc._

def xForwardedFor[A](action: Action[A]) = Action.async(action.parser) { request =>
  val newRequest = request.headers.get("X-Forwarded-For").map { xff =>
    new WrappedRequest[A](request) {
      override def remoteAddress = xff
    }
  } getOrElse request
  action(newRequest)
}
```

```
注意： Play已经内置了对X-Forwarded-For头的支持
```

我们可以阻塞一个请求：

```scala
import play.api.mvc._

def onlyHttps[A](action: Action[A]) = Action.async(action.parser) { request =>
  request.headers.get("X-Forwarded-Proto").collect {
    case "https" => action(request)
  } getOrElse {
    Future.successful(Forbidden("Only HTTPS requests allowed"))
  }
}
```

最后，我们还可以修改返回的结果：

```scala
import play.api.mvc._
import play.api.libs.concurrent.Execution.Implicits._

def addUaHeader[A](action: Action[A]) = Action.async(action.parser) { request =>
  action(request).map(_.withHeaders("X-UA-Compatible" -> "Chrome=1"))
}
```

## 不同的请求类型

当组合action允许在HTTP请求和响应的层面进行一些额外的操作时，你自然而然的就会想到构建数据转换的管道（pipeline），为请求本身增加上下文（context）或是执行一些验证。你可以把`ActionFunction`当做是一个应用在请求上的方法，该方法参数化了传入的请求类型和输出类型，并将其传至下一层。每个action方法可以是一个模块化的处理，如验证，数据库查询，权限检查，或是其他你想要在action中组合并重用的操作。

Play还有一些预定义的特性（trait），它们实现了`ActionFunction`，并且对不同类型的操作都非常有用：

* `ActionTransformer`可以更改请求，比如添加一些额外的信息。
* `ActionFilter`可选择性的拦截请求，比如在不改变请求的情况下处理错误。
* `ActionRefiner`是以上两种的通用情况
* `ActionBuilder`是一种特殊情况，它接受`Request`作为参数，所以可以用来构建action。

你可以通过实现`invokeBlock`方法来定义你自己的`ActionFunction`。通常为了方便，会定义输入和输出类型为`Request`（使用`WrappedRequest`），但这并不是必须的。

### 验证

Action方法最常见的用例之一就是验证。我们可以简单的实现自己的验证action转换器（transformer），从原始请求中获取用户信息并添加到`UserRequest`中。需要注意的是这同样也是一个`ActionBuilder`，因为其输入是一个`Request`：

```scala
import play.api.mvc._

class UserRequest[A](val username: Option[String], request: Request[A]) extends WrappedRequest[A](request)

object UserAction extends
    ActionBuilder[UserRequest] with ActionTransformer[Request, UserRequest] {
  def transform[A](request: Request[A]) = Future.successful {
    new UserRequest(request.session.get("username"), request)
  }
}
```

Play提供了内置的验证action构造器。更多信息请参考[这里](https://www.playframework.com/documentation/2.3.x/api/scala/index.html#play.api.mvc.Security$$AuthenticatedBuilder$)

```
注意：内置的验证action构造器只是一个简便的helper，目的是为了用尽可能少的代码为一些简单的用例添加验证功能，其实现和上面的例子非常相似。

如果你有更复杂的需求，推荐实现你自己的验证action
```

### 为请求添加信息

现在让我们设想一个REST API，处理类型为`Item`的对象。在`/item/:itemId`的路径下可能有多个路由，并且每个都需要查询该`item`。这种情况下，将逻辑写在action方法中非常有用。

首先，我们需要创建一个请求对象，将`Item`添加到`UserRequest`中：

```scala
import play.api.mvc._

class ItemRequest[A](val item: Item, request: UserRequest[A]) extends WrappedRequest[A](request) {
  def username = request.username
}
```

现在，创建一个action修改器（refiner）查找该item并返回`Either`一个错误（`Left`）或是一个新的`ItemRequest`（`Right`）。注意这里的action修改器（refiner）定义在了一个方法中，用来获取该item的id：

```scala
def ItemAction(itemId: String) = new ActionRefiner[UserRequest, ItemRequest] {
  def refine[A](input: UserRequest[A]) = Future.successful {
    ItemDao.findById(itemId)
      .map(new ItemRequest(_, input))
      .toRight(NotFound)
  }
}
```

### 验证请求

最后，我们希望有个action方法能够验证是否继续处理该请求。例如，我们可能需要检查`UserAction`中获取的user是否有权限使用`ItemAction`中得到的item，如果不允许则返回一个错误：

```scala
object PermissionCheckAction extends ActionFilter[ItemRequest] {
  def filter[A](input: ItemRequest[A]) = Future.successful {
    if (!input.item.accessibleByUser(input.username))
      Some(Forbidden)
    else
      None
  }
}
```

### 合并起来

现在我们可以将所有这些action方法链起来（从`ActionBuilder`开始）， 使用`andThen`来创建一个action：

```scala
def tagItem(itemId: String, tag: String) =
  (UserAction andThen ItemAction(itemId) andThen PermissionCheckAction) { request =>
    request.item.addTag(tag)
    Ok("User " + request.username + " tagged " + request.item.id)
  }
```

Play同样支持[全局过滤API](https://www.playframework.com/documentation/2.3.x/ScalaHttpFilters)，对于全局的过滤非常有用。