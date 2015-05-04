# JSON Transformers

> 注意，这一节的内容最早由 Pascal Voitot 发表在 [mandubian.com](http://mandubian.com/2012/10/29/unveiling-play-2-dot-1-json-api-part3-json-transformers/) 上。（文章太旧，请带着批判的眼光去读。）

现在已经知道如何验证 JSON，以及如何将 JSON 转成任意结构或将任意结构转成 JSON。但当我开始用那些组合子来写 web 应用，我立即遇到了这样的情况：从网络中读取 JSON，验证它然后再将它转成 JSON。

## JSON coast-to-coast 设计介绍

### 我们注定要将 JSON 转成 OO 吗？

近几年来，几乎在所有的 web 框架中（除了最近出现的用 JS 写服务端的情况，JSON 就是默认的数据结构），我们都要做的一件事就是：从网络中读取 JSON，然后将 JSON 转成 OO 结构，比如转成类（或 Scala 中的样例类），为什么？

* 一个好的理由：OO 结构是「语言原生」的，对于你的业务逻辑，在保证与 web 层隔离的情况下，可以以一种无缝的方式操作数据。
* 一个更值得怀疑的理由：ORM 框架只能使用 OO 结构与数据库进行对接，我们基本上确信在现有 ORM 的特性下（不管好的坏的），这是无法通过其他方式来完成的。

### OO 转换真的是默认的使用案例吗？

在许多情况下，你并不需要真的对数据执行什么业务逻辑，更多的是在存储前或提取后，对数据进行验证和转换。

让我们来看一下「增/删/改/查」操作：

* 你通过网络请求获得数据，验证它们是没有问题的，然后将它们插入数据库，或用它们更新数据库。
* 另一种情况是，你从数据库中取得数据，然后将他们发送出去。

因此，一般情况下，对于「增/删/改/查」操作，你把 JSON 转换成 OO 结构仅仅是因为框架的限定。

> 我并不是因此就说你不应该把 JSON 转成 OO 结构，但大部分情况下你可以不必这么做。我们应该在只有真正的业务逻辑需要处理的时候，才将相关数据转成 OO 结构。

### 新技术玩家改变操作 JSON 的方式

除了上述事实，在数据库上我们多了一些新的选择，如 MongoDB 或 CouchDB，它们接收的是类似于 JSON 树的文档结构数据（BSON）。

对于这些数据库，我们还有一些好用的工具如 [ReactiveMongo](http://reactivemongo.org/)，它提供了一个响应式的环境以非常自然的方式流式地将数据写入与读出 MongoDB。

在写 [Play2-ReactiveMongo](https://github.com/zenexity/Play-ReactiveMongo) 模块的同时，我与 Stephane Godbillon 也在将 ReactiveMongo 集成到 Play2.1 中。除了为 Play2.1 提供 MongoDB 的便捷操作，该模块还支持 JSON 与 BSON 之间的转换。

> 这意味着你可以操作 JSON 流直接读写数据库，而无需将它们转成 OO 结构。

### JSON coast-to-coast 设计

考虑到这一点，我们可以简单地想象以下情形：

* 接收 JSON
* 验证 JSON
* 将 JSON 变换成对应数据库的文档结构
* 直接发送 JSON 给数据库

当从数据库中取数据对外服务时，也是相似的情况：

* 直接从数据库中取出 JSON 格式的数据
* 过滤/变换这个 JSON，选取那些允许展示给客户端的数据（例如，你应该不想把安全信息也发送出去）
* 直接发送 JSON 给客户端

在这种情况下，我们可以非常简单地想象在客户端与数据库之间操作 JSON 格式的数据流而无需做 JSON 以外的变换。自然地，当你将这种变换流融入 Play2.1 提供的响应式基础设施中，突然间就为你打开了新的视野。

> 这被我叫作 JSON coast-to-coast 设计：
> * 不要把 JSON 数据视为一块一块的，而是把它看成客户端与数据库之间的数据流
> * 把 JSON 流视为一个管道，你可以将它与其它管道相连，同时对它进行修改和变换
> * 以异步/非阻塞的方式看待数据流
>
> 这也是 Play2.1 成为响应式体系结构的一个原因。我相信把你的应用视为承载数据流的棱镜将极大改变你设计 web 应用的方式（如果看不懂，请略过，译者也看不懂）。它可能会开拓一个比传统架构更适应今天 web 应用需求的新领域。

因此，正如你自己推断出来的那样，想要直接基于验证和变换操作 JSON 流，我们需要一些新的工具。JSON 组合子是个不错的选择，但它们过于通用了。这就是为什么我们创造了一些更加专用的组合子和 API 来做这件事，我们把它们称为 JSON 变换器（JSON transformers）。

## JSON 变换器

JSON 变换器其实就是 `f:JSON => JSON`。因此一个 JSON 变换器可以是一个简单的 `Writes[A <: JsValue]`。但一个 JSON 变换器并不仅仅是一个函数，正如我们之前所说，我们想在变换 JSON 的同时验证它。最终结果是，一个 JSON 变换器其实是一个 `Reads[A <: JsValue]`。

> 注意：`Reads[A <: JsValue]` 并不仅仅能读取/验证，它还可以进行变换。

## 使用 `JsValue.transform` 而不是 `JsValue.validate`

我们提供了一个 helper 方法，以此帮助人们将 `Reads[T]` 视为一个变换器（transformer），而不仅仅是一个验证器（validator）。

```scala
JsValue.transform[A <: JsValue](reads: Reads[A]): JsResult[A]
```

该函数签名与 `JsValue.validate(reads)` 是类似的。

## 细节

在接下来的示例代码中，我们将使用以下的 JSON 数据：

```
{
  "key1" : "value1",
  "key2" : {
    "key21" : 123,
    "key22" : true,
    "key23" : [ "alpha", "beta", "gamma"],
    "key24" : {
      "key241" : 234.123,
      "key242" : "value242"
    }
  },
  "key3" : 234
}
```

## 案例 1：在 JsPath 中取 JSON 值

### 取值（作为 JsValue）

```scala
import play.api.libs.json._

val jsonTransformer = (__ \ 'key2 \ 'key23).json.pick

scala> json.transform(jsonTransformer)
res9: play.api.libs.json.JsResult[play.api.libs.json.JsValue] =
    JsSuccess(
      ["alpha","beta","gamma"],
      /key2/key23
    )
```

`(__ \ 'key2 \ 'key23).json...`
* 所有的 JSON 变换器都在 `JsPath.json` 中

`(__ \ 'key2 \ 'key23).json.pick`
* `pick` 是一个 `Reads[JsValue]`，根据给定的 JsPath 取值。这里值是：`["alpha","beta","gamma"]`

`JsSuccess(["alpha","beta","gamma"],/key2/key23)`
* 这就是一个取值成功后的 `JsResult`
* `/key2/key23` 表示读取这些值的 JsPath，不过不用管它，把它们放这里只是为了组成一个 `JsResult`
* 出来 `["alpha","beta","gamma"]` 是由于我们重写了 `toString`

> 注意：`jsPath.json.pick` 只会取 JsPath 中的值。

### 取值（作为类型）

```scala
import play.api.libs.json._

val jsonTransformer = (__ \ 'key2 \ 'key23).json.pick[JsArray]

scala> json.transform(jsonTransformer)
res10: play.api.libs.json.JsResult[play.api.libs.json.JsArray] =
    JsSuccess(
      ["alpha","beta","gamma"],
      /key2/key23
    )
```

```scala
(__ \ 'key2 \ 'key23).json.pick[JsArray]
```

`pick[T]` 是一个 `Reads[T <: JsValue]`，根据给定的 JsPath 取值（在我们的例子中，取出来是一个 `JsArray`）。

> 注意：`jsPath.json.pick[T <: JsValue]` 只提取 JsPath 中相应类型（T）的值。

## 案例 2：根据 JsPath 提取分支

### 提取分支（作为 JsValue）

```scala
import play.api.libs.json._

val jsonTransformer = (__ \ 'key2 \ 'key24 \ 'key241).json.pickBranch

scala> json.transform(jsonTransformer)
res11: play.api.libs.json.JsResult[play.api.libs.json.JsObject] =
  JsSuccess(
    {
      "key2": {
        "key24":{
          "key241":234.123
        }
      }
    },
    /key2/key24/key241
  )
```

`(__ \ 'key2 \ 'key24 \ 'key241).json.pickBranch`
* `pickBranch` 是一个 `Reads[JsValue]`，根据给定的 JsPath 提取对应的 JSON 分支。

`{"key2":{"key24":{"key241":234.123}}}`
* 分支提取结果。

> 注意：`jsPath.json.pickBranch` 根据 JsPath 提取单条分支以及其中的值。

## 案例 3：从输入 JsPath 拷贝值到新的 JsPath

```scala
import play.api.libs.json._

val jsonTransformer = (__ \ 'key25 \ 'key251).json.copyFrom( (__ \ 'key2 \ 'key21).json.pick )

scala> json.transform(jsonTransformer)
res12: play.api.libs.json.JsResult[play.api.libs.json.JsObject]
  JsSuccess(
    {
      "key25":{
        "key251":123
      }
    },
    /key2/key21
  )
```

`(__ \ 'key25 \ 'key251).json.copyFrom( reads: Reads[A <: JsValue] )`
* `copyFrom` 是一个 `Reads[JsValue]`
* `copyFrom` 使用提供的 `Reads[A]` 从给定的 JSON 中读取 JsValue
* `copyFrom` 将提取出来的 JsValue 拷贝到新分支的叶子节点

`{"key25":{"key251":123}}`
* `copyFrom` 读出值 `123`
* `copyFrom` 将这个值拷贝进新分支：`(__ \ 'key25 \ 'key251)`

> 注意：`jsPath.json.copyFrom(Reads[A <: JsValue])` 从输入 JSON 中读值，然后将它拷贝进新创建的分支中。

## 案例 4：拷贝整个输入 JSON & 更新一个分支

```scala
import play.api.libs.json._

val jsonTransformer = (__ \ 'key2 \ 'key24).json.update(
    __.read[JsObject].map{ o => o ++ Json.obj( "field243" -> "coucou" ) }
)

scala> json.transform(jsonTransformer)
res13: play.api.libs.json.JsResult[play.api.libs.json.JsObject] =
  JsSuccess(
    {
      "key1":"value1",
      "key2":{
        "key21":123,
        "key22":true,
        "key23":["alpha","beta","gamma"],
        "key24":{
          "key241":234.123,
          "key242":"value242",
          "field243":"coucou"
        }
      },
      "key3":234
    },
  )
```

`(__ \ 'key2).json.update(reads: Reads[A < JsValue])`
* 这是一个 `Reads[JsObject]`

`(__ \ 'key2 \ 'key24).json.update(reads)` 做了以下 3 件事：
* 从输入 JSON 中提取路径 `(__ \ 'key2 \ 'key24)` 上的值
* 应用 `reads` 在这个值上，并重新创建一个 `(__ \ 'key2 \ 'key24)` 分支，将 `reads` 的结果加到这个分支上
* 用新的分支替代输入 JSON 中的原有分支（因此它只能用于 JsObject 而非其它类型的 JsValue）

`JsSuccess({…},)`
* 我们可以看到在返回的结果中，并没有 JsPath 作为第二个参数，因为 JSON 操作已经从根 JsPath 处完成

> `jsPath.json.update(Reads[A <: JsValue])` 只能用于 JsObject，拷贝整个 JsObject 然后用提供的 `Reads[A <: JsValue]` 更新 jsPath

## 案例 5：在新分支中放置一个给定值

```scala
import play.api.libs.json._

val jsonTransformer = (__ \ 'key24 \ 'key241).json.put(JsNumber(456))

scala> json.transform(jsonTransformer)
res14: play.api.libs.json.JsResult[play.api.libs.json.JsObject] =
  JsSuccess(
    {
      "key24":{
        "key241":456
      }
    },
  )
```

`(__ \ 'key24 \ 'key241).json.put( a: => JsValue )`
* 这是一个 `Reads[JsObject]`
* 创建一个新的分支：`(__ \ 'key24 \ 'key241)`
* 将 `a` 值放入这个分支中

`jsPath.json.put( a: => JsValue )`
* 接收一个 JsValue 参数（按名称传参），甚至可以允许传递一个闭包

`jsPath.json.put`
* 并不关心输入 JSON 是什么
* 简单地用给定值替换输入 JSON

> 注意：`jsPath.json.put( a: => Jsvalue )` 用给定的值创建一个新分支，无需考虑输入 JSON 是什么

## 案例 6：从输入 JSON 中剪掉一个分支

```scala
import play.api.libs.json._

val jsonTransformer = (__ \ 'key2 \ 'key22).json.prune

scala> json.transform(jsonTransformer)
res15: play.api.libs.json.JsResult[play.api.libs.json.JsObject] =
  JsSuccess(
    {
      "key1":"value1",
      "key3":234,
      "key2":{
        "key21":123,
        "key23":["alpha","beta","gamma"],
        "key24":{
          "key241":234.123,
          "key242":"value242"
        }
      }
    },
    /key2/key22/key22
  )
```

`(__ \ 'key2 \ 'key22).json.prune`
* 这是一个 `Reads[JsObject]`，只用于 JsObject
* 从输入 JSON 中移除给定的 JsPath（`key2` 下的 `key22` 已经被剪掉了）

我们可以注意到输出的 JsObject 的键（key）的顺序与输入 JsObject 是不一样的。这是由 JsObject 的实现及合并机制所导致，但键的顺序不一致并不重要，因为我们重写了 `JsObject.equals` 方法，并且把这种情况考虑在内了。

> 注意：`jsPath.json.prune` 只能用于 JsObject，它的作用是从输入 JSON 中移除给定的 JsPath
> 还需注意以下两点：
> `prune` 暂时无法用于递归的 JsPath
> 如果 `prune` 无法找到可以删除的分支，它并不会产生错误，而是将 JSON 原样返回

## 案例 7：选择一个分支并在两处更新它的内容

```scala
import play.api.libs.json._
import play.api.libs.json.Reads._

val jsonTransformer = (__ \ 'key2).json.pickBranch(
  (__ \ 'key21).json.update(
    of[JsNumber].map{ case JsNumber(nb) => JsNumber(nb + 10) }
  ) andThen
  (__ \ 'key23).json.update(
    of[JsArray].map{ case JsArray(arr) => JsArray(arr :+ JsString("delta")) }
  )
)

scala> json.transform(jsonTransformer)
res16: play.api.libs.json.JsResult[play.api.libs.json.JsObject] =
  JsSuccess(
    {
      "key2":{
        "key21":133,
        "key22":true,
        "key23":["alpha","beta","gamma","delta"],
        "key24":{
          "key241":234.123,
          "key242":"value242"
        }
      }
    },
    /key2
  )
```

`(__ \ 'key2).json.pickBranch(reads: Reads[A <: JsValue])`
* 从输入 JSON 中提取分支 `__ \ 'key2` 并对该分支下的叶子结点应用 `reads`（只针对内容）

`(__ \ 'key21).json.update(reads: Reads[A <: JsValue])`
* 更新 `(__ \ 'key21)` 分支

`of[JsNumber]`
* 这就是一个 `Reads[JsNumber]`
* 从路径 `(__ \ 'key21)` 下提取一个 JsNumber

`of[JsNumber].map{ case JsNumber(nb) => JsNumber(nb + 10) }`
* 读取一个 JsNumber（即路径 `__ \ 'key21` 下的值 123）
* 使用 `Reads[A].map` 将值增加 10（创建一个比原来大 10 的值并替换原来的）

`andThen`
* 用于组合两个 `Reads[A]`
* 应用第一个 `reads` 后将结果以管道的方式送给第二个 `reads` 处理

`of[JsArray].map{ case JsArray(arr) => JsArray(arr :+ JsString("delta")`
* 读取一个 JsArray（即路径 `__ \ 'key23` 下的值 ["alpha","beta","gamma"]）
* 使用 `Reads[A].map` 在上述值后追加一个 `JsString("delta")`

> 注意：得到的结果仅是 `__ \ 'key2` 分支，因为我们只选取了它

## 案例 8：选取一条分支并将其子分支剪掉

```scala
import play.api.libs.json._

val jsonTransformer = (__ \ 'key2).json.pickBranch(
  (__ \ 'key23).json.prune
)

scala> json.transform(jsonTransformer)
res18: play.api.libs.json.JsResult[play.api.libs.json.JsObject] =
  JsSuccess(
    {
      "key2":{
        "key21":123,
        "key22":true,
        "key24":{
          "key241":234.123,
          "key242":"value242"
        }
      }
    },
    /key2/key23
  )
```

`(__ \ 'key2).json.pickBranch(reads: Reads[A <: JsValue])`
* 从输入 JSON 中提取分支 `__ \ 'key2` 并对该分支下的叶子结点应用 `reads`（只针对内容）

`(__ \ 'key23).json.prune`
* 从上述结果中移除 `__ \ 'key23` 分支

> 注意最终结果是一个没有 `key23` 的 `__ \ 'key2` 分支

## 那组合子（combinator）呢？

在话题变得枯燥无聊前，我及时打住了。

你只需要住记，你现在有一个非常强大的工具包来创建通用的 JSON 变换器。你可以组合，map，flatmap 这些变换器，因此几乎有无限种可能性。

最后，我们还要把这些新产生的 JSON 变换器与之前的 `Reads` 组合子组合起来用。

让我们通过下面的例子来展示（一个将 Gizmo 转为 Gremlin 的 JSON 变换器）。

下面是 Gizmo：

```scala
val gizmo = Json.obj(
  "name" -> "gizmo",
  "description" -> Json.obj(
    "features" -> Json.arr( "hairy", "cute", "gentle"),
    "size" -> 10,
    "sex" -> "undefined",
    "life_expectancy" -> "very old",
    "danger" -> Json.obj(
      "wet" -> "multiplies",
      "feed after midnight" -> "becomes gremlin"
    )
  ),
  "loves" -> "all"
)
```

以下是 Gremlin：

```scala
val gremlin = Json.obj(
  "name" -> "gremlin",
  "description" -> Json.obj(
    "features" -> Json.arr("skinny", "ugly", "evil"),
    "size" -> 30,
    "sex" -> "undefined",
    "life_expectancy" -> "very old",
    "danger" -> "always"
  ),
  "hates" -> "all"
)
```

让我们来写一个 JSON 变换器来完成 Gizmo 到 Gremlin 的变换：

```scala
import play.api.libs.json._
import play.api.libs.json.Reads._
import play.api.libs.functional.syntax._

val gizmo2gremlin = (
    (__ \ 'name).json.put(JsString("gremlin")) and
    (__ \ 'description).json.pickBranch(
        (__ \ 'size).json.update( of[JsNumber].map{ case JsNumber(size) => JsNumber(size * 3) } ) and
        (__ \ 'features).json.put( Json.arr("skinny", "ugly", "evil") ) and
        (__ \ 'danger).json.put(JsString("always"))
        reduce
    ) and
    (__ \ 'hates).json.copyFrom( (__ \ 'loves).json.pick )
) reduce

scala> gizmo.transform(gizmo2gremlin)
res22: play.api.libs.json.JsResult[play.api.libs.json.JsObject] =
  JsSuccess(
    {
      "name":"gremlin",
      "description":{
        "features":["skinny","ugly","evil"],
        "size":30,
        "sex":"undefined",
        "life_expectancy":
        "very old","danger":"always"
      },
      "hates":"all"
    },
  )
```

搞定！我不打算解释上面的变换了，因为看完上面的内容后你们应该能理解了。需要注意一点：

`(__ \ 'features).json.put(…)` 放在 `(__ \ 'size).json.update` 之后，因此它可以覆盖原有的 `(__ \ 'features)`。

`(Reads[JsObject] and Reads[JsObject]) reduce`
* 它将两个 `Reads[JsObject]` 合并（JsObject ++ JsObject）
* 它将同一个 JSON 应用到两个 `Reads[JsObject]`，不像 `andThen`，`andThen` 是将第一个 reads 的处理结果注入给第二个进行处理
