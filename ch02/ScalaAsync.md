# Handling asynchronous results

## 构建异步controller

Play框架本身就是异步的。对每一个请求而言，Play都会使用异步，非阻塞的方式来处理。

默认配置下controller即是异步的了。换言之，应用应该避免在controller中做阻塞操作，例如，让controller等待某一个操作完成。此类阻塞操作的常见例子还有：调用JDBC，流处理（streaming）API，HTTP请求和长时间的计算任务。

尽管可以通过增加默认执行环境（execution context）中的线程数来提升阻塞controller的并发请求处理能力，但建议使用异步controller可以使应用更易于扩展，在负载较大的情况下依然能够保持响应。

## 创建非阻塞action

由于Play的异步工作方式，action应该做到尽可能的快，例如非阻塞。那么，在还没有得到返回结果的情况下，应该返回什么作为结果呢？答案是future结果！

一个`Future[Result]`最终会被替换成一个类型为`Result`的值。通过提供`Future[Result]`而非`Result`，我们能够在非阻塞的情况下快速生成结果。一旦从promise中得到了最终结果，Play就会返回该结果。

Web客户端在等待响应时是阻塞的，但服务器端没有发生阻塞，而且服务器资源依然可以服务其他客户端。

## 如何创建`Future[Result]`

在创建一个`Future[Result]`之前，我们需要先创建另一个future：这个future会返回一个真实的值，以便我们依此生成结果：

```scala
import play.api.libs.concurrent.Execution.Implicits.defaultContext

val futurePIValue: Future[Double] = computePIAsynchronously()
val futureResult: Future[Result] = futurePIValue.map { pi =>
  Ok("PI value computed: " + pi)
}
```

Play所有的异步API调用都会返回一个`Future`。例如，不管你是用`play.api.libs.WS`API在调用外部web服务，或是用Akka分配异步任务，亦或是使用`play.api.libs.Akka`与actor进行通讯。

以下是一个异步调用并获得一个`Future`的简单例子：

```scala
import play.api.libs.concurrent.Execution.Implicits.defaultContext

val futureInt: Future[Int] = scala.concurrent.Future {
  intensiveComputation()
}
```

注意：理解哪个线程运行了future非常重要，以上的两段代码都导入了Play的默认执行环境（execution context）。这是一个隐式（implicit）的参数，会被传入所有接受回调的future API方法中。执行环境（execution context）通常等价于线程池，但这并不是一定的。

简单的把同步IO封装入`Future`并不能将其转换为异步的。如果你不能通过改变应用架构来避免阻塞操作，那么该操作总会在某一时刻被执行的，而相应的线程则会被阻塞。所以，除了将操作封装于`Future`中，还需要让它运行在配置了足够线程来处理可预计并发的独立执行环境（execution context）中。更多信息请见[理解Play线程池](https://www.playframework.com/documentation/2.3.x/ThreadPools)

这对于使用Actor来阻塞操作也非常有用。Actor提供了一种非常简洁的模型来处理超时和失败，设置阻塞执行环境（execution context），并且管理了该服务的一切状态。Actor还提供了像`ScatterGatherFirstCompletedRouter`这样的模式来处理同时缓存和数据库请求，并且能够远程执行于后端服务器集群中。但使用actor可能有些多余，主要还是取决你想要的是什么。

## 返回future

迄今为止，我们一直都在调用`Action.apply`方法来构建action，为了发出一个异步的结果，我们需要调用`Action.async`：

```scala
import play.api.libs.concurrent.Execution.Implicits.defaultContext

def index = Action.async {
  val futureInt = scala.concurrent.Future { intensiveComputation() }
  futureInt.map(i => Ok("Got result: " + i))
}
```

## Action默认即是异步的

Play的[action](../ch1/ScalaActions.md)默认即是异步的。比如说，在下面这个controller中，`{ Ok(...) }`这部分代码并不是controller的方法体。这其实是一个传入`Action`对象`apply`方法的匿名函数，用来创建一个`Action`对象。运行时，你写的这个匿名函数会被调用并返回一个`Future`结果。

```scala
val echo = Action { request =>
  Ok("Got request [" + request + "]")
}
```

注意：`Action.apply`和`Action.async`创建的`Action`对象在Play内部会以同样的方式处理。他们都是异步的`Action`，而不是一个同步一个异步。`.async`构造器只是用来简化创建基于API并返回`Future`的action，让非阻塞的代码更加容易写。

## 处理超时

能够合理的处理超时通常很有用，当出现问题时可以避免浏览器无谓的阻塞和等待。简单的通过调用promise超时来构造另一个promise便能够处理这些情况：

```scala
import play.api.libs.concurrent.Execution.Implicits.defaultContext
import scala.concurrent.duration._

def index = Action.async {
  val futureInt = scala.concurrent.Future { intensiveComputation() }
  val timeoutFuture = play.api.libs.concurrent.Promise.timeout("Oops", 1.second)
  Future.firstCompletedOf(Seq(futureInt, timeoutFuture)).map {
    case i: Int => Ok("Got result: " + i)
    case t: String => InternalServerError(t)
  }
}
```