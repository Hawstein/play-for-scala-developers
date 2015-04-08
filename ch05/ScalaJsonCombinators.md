# JSON Reads/Writes/Format Combinators

在 [JSON 基础](https://www.playframework.com/documentation/2.3.x/ScalaJson)一节中，我们介绍了 `Reads` 和 `Writes` 转换器。使用它们，我们可以在 `JsValue` 结构和其他数据类型之间做转换。这一节将更详细地介绍如何构建这些转换器，以及在转换的过程中如何进行验证。

这一节使用到的 `JsValue` 结构以及相应的模型：

```scala
import play.api.libs.json._

val json: JsValue = Json.parse("""
{
  "name" : "Watership Down",
  "location" : {
    "lat" : 51.235685,
    "long" : -1.309197
  },
  "residents" : [ {
    "name" : "Fiver",
    "age" : 4,
    "role" : null
  }, {
    "name" : "Bigwig",
    "age" : 6,
    "role" : "Owsla"
  } ]
}
""")
```

```scala
case class Location(lat: Double, long: Double)
case class Resident(name: String, age: Int, role: Option[String])
case class Place(name: String, location: Location, residents: Seq[Resident])
```

## JsPath

`JsPath` 是构建`Reads/Writes` 的核心。`JsPath` 指明了数据在 `JsValue` 中的位置。你可以使用 `JsPath` 对象（根路径）来定义一个 `JsPath` 子实例，语法与遍历 `JsValue` 相似：

```scala
import play.api.libs.json._

val json = { ... }

// Simple path
val latPath = JsPath \ "location" \ "lat"

// Recursive path
val namesPath = JsPath \\ "name"

// Indexed path
val firstResidentPath = (JsPath \ "residents")(0)
```

`play.api.libs.json` 包中为 `JsPath` 定义了一个别名：__（两个下划线），如果你喜欢的话，你也可以使用它：

```scala
val longPath = __ \ "location" \ "long"
```

## Reads

`Reads` 转换器用于将 `JsValue` 转换成其他类型。你可以通过组合与嵌套 `Reads` 来构造更复杂的 `Reads`。

你需要导入以下内容来创建 `Reads`：

```scala
import play.api.libs.json._ // JSON library
import play.api.libs.json.Reads._ // Custom validation helpers
import play.api.libs.functional.syntax._ // Combinator syntax
```

### Path Reads

`JsPath` 含有以下两个方法可用来创建特殊的 `Reads`，它将应用另一个 `Reads` 到指定路径的 `JsValue`：

* `JsPath.read[T](implicit r: Reads[T]): Reads[T]` - 创建一个 `Reads[T]`，它将应用隐式参数 `r` 到该路径的 `JsValue`。
* `JsPath.readNullable[T](implicit r: Reads[T]): Reads[Option[T]]` - 在路径可能缺失，或包含一个空值时使用。

> 注意：JSON 库为基本类型（如 String，Int，Double 等）提供了隐式 `Reads`。

定义具体某一路径的 `Reads`：

```scala
val nameReads: Reads[String] = (JsPath \ "name").read[String]
```

定义具体某一路径的 `Reads`，并且包含自定义样例类：

```scala
case class DisplayName(name:String)
val nameReads: Reads[DisplayName] = (JsPath \ "name").read[String].map(DisplayName(_))
```

### 复合 Reads

你可以将多个单路径 `Reads` 组合成一个复合 `Reads`，这样就可以转换复杂模型了。

为了便于理解，我们先将一个组合结果分解成两条语句。首先通过 `and` 组合子来组合 `Reads` 对象：

```scala
val locationReadsBuilder =
  (JsPath \ "lat").read[Double] and
  (JsPath \ "long").read[Double]
```

上面产生的结果类型为 'FunctionalBuilder[Reads]#CanBuild2[Double, Double]'，这只是一个中间结果，它将被用来创建一个复合 `Reads`。

第二步调用 `CanBuildX` 的 `apply` 方法来将单个的结果转换成你的模型，这将返回一个复合 `Reads`。如果你有一个带有构造器签名的样例类，你就可以直接使用它的 `apply` 方法：

```scala
implicit val locationReads = locationReadsBuilder.apply(Location.apply _)
```

将上述代码合成一条语句：

```scala
implicit val locationReads: Reads[Location] = (
  (JsPath \ "lat").read[Double] and
  (JsPath \ "long").read[Double]
)(Location.apply _)
```

### Reads 验证

`JsValue.validate` 方法已经在 [JSON 基础](https://www.playframework.com/documentation/2.3.x/ScalaJson) 一节中介绍过，我们推荐它进行验证以及用它将 `JsValue` 转换成其它类型。以下是基本用法：

```scala
val json = { ... }

val nameReads: Reads[String] = (JsPath \ "name").read[String]

val nameResult: JsResult[String] = json.validate[String](nameReads)

nameResult match {
  case s: JsSuccess[String] => println("Name: " + s.get)
  case e: JsError => println("Errors: " + JsError.toFlatJson(e).toString())
}
```

`Reads` 的默认验证比较简单，比如只检查类型转换错误。你可以通过使用 `Reads` 的验证 helper 来自定义验证规则。以下是一些常用的 helper：

* `Reads.email` - 验证字符串是邮箱地址格式。
* `Reads.minLength(nb)` - 验证一个字符串的最小长度。
* `Reads.min` - 验证最小数值。
* `Reads.max` - 验证最大数值。
* `Reads[A] keepAnd Reads[B] => Reads[A]` - 尝试 `Reads[A]` 及 `Reads[B]` 但最终只保留 `Reads[A]` 结果的运算符（如果你知道 Scala 解析组合子，即有 `keepAnd == <~`）。
* `Reads[A] andKeep Reads[B] => Reads[B]` - 尝试 `Reads[A]` 及 `Reads[B]` 但最终只保留 `Reads[B]` 结果的运算符（如果你知道 Scala 解析组合子，即有 `andKeep == ~>`）。
* `Reads[A] or Reads[B] => Reads` - 执行逻辑或并保留最后选中的 Reads 的运算符。

想添加验证，只需将相应 helper 作为 `JsPath.read` 的参数即可：

```scala
val improvedNameReads =
  (JsPath \ "name").read[String](minLength[String](2))
```

### 将它们合起来

通过使用复合 `Reads` 和自定义验证，我们可以为模型定义一组有效的 `Reads` 并应用它们：

```scala
import play.api.libs.json._
import play.api.libs.json.Reads._
import play.api.libs.functional.syntax._

implicit val locationReads: Reads[Location] = (
  (JsPath \ "lat").read[Double](min(-90.0) keepAnd max(90.0)) and
  (JsPath \ "long").read[Double](min(-180.0) keepAnd max(180.0))
)(Location.apply _)

implicit val residentReads: Reads[Resident] = (
  (JsPath \ "name").read[String](minLength[String](2)) and
  (JsPath \ "age").read[Int](min(0) keepAnd max(150)) and
  (JsPath \ "role").readNullable[String]
)(Resident.apply _)

implicit val placeReads: Reads[Place] = (
  (JsPath \ "name").read[String](minLength[String](2)) and
  (JsPath \ "location").read[Location] and
  (JsPath \ "residents").read[Seq[Resident]]
)(Place.apply _)


val json = { ... }

json.validate[Place] match {
  case s: JsSuccess[Place] => {
    val place: Place = s.get
    // do something with place
  }
  case e: JsError => {
    // error handling flow
  }
}
```

注意，复合 `Reads` 可以嵌套使用。在上面的例子中，`placeReads` 使用了前面定义的隐式 `locationReads` 和 `residentReads`。

## Writes

`Writes` 用于将其它类型转换成 `JsValue`。

为样例类构建简单的 Writes，只需在 `Writes` 的 apply 方法体中使用一个函数即可：

```scala
case class DisplayName(name:String)
implicit val displayNameWrite: Writes[DisplayName] = Writes {
  (displayName: DisplayName) => JsString(displayName.name)
}
```

你可以使用 `JsPath` 和组合子来构建复合 `Writes`，这一点与 `Reads` 非常相似。以下是我们模型的 `Writes`：

```scala
import play.api.libs.json._
import play.api.libs.functional.syntax._

implicit val locationWrites: Writes[Location] = (
  (JsPath \ "lat").write[Double] and
  (JsPath \ "long").write[Double]
)(unlift(Location.unapply))

implicit val residentWrites: Writes[Resident] = (
  (JsPath \ "name").write[String] and
  (JsPath \ "age").write[Int] and
  (JsPath \ "role").writeNullable[String]
)(unlift(Resident.unapply))

implicit val placeWrites: Writes[Place] = (
  (JsPath \ "name").write[String] and
  (JsPath \ "location").write[Location] and
  (JsPath \ "residents").write[Seq[Resident]]
)(unlift(Place.unapply))


val place = Place(
  "Watership Down",
  Location(51.235685, -1.309197),
  Seq(
    Resident("Fiver", 4, None),
    Resident("Bigwig", 6, Some("Owsla"))
  )
)

val json = Json.toJson(place)
```

复合 `Writes` 与复合 `Reads` 有以下不同点：

* 单个路径的 `Writes` 通过 `JsPath.write` 方法来创建。
* 将模型转换成 `JsValue` 无需验证，因此也不需要验证 helper。
* 中间结果 `FunctionalBuilder#CanBuildX`（由 `and` 组合子产生）接收一个函数，该函数将复合类型 `T` 转换成一个元组，该元组与单路径 `Writes` 匹配。尽管看起来和 `Reads` 非常对称，样例类的 `unapply` 方法返回的是属性元组的 `Option` 类型，因此需要使用 `unlift` 方法将元组提取出来。

## 递归类型

有一种特殊的情况是上面的例子没有讲到的，即如何处理递归类型的 `Reads` 和 `Writes`。`JsPath` 提供了 `lazyRead` 和 `lazyWrite` 方法来处理这种情况：

```scala
case class User(name: String, friends: Seq[User])

implicit lazy val userReads: Reads[User] = (
  (__ \ "name").read[String] and
  (__ \ "friends").lazyRead(Reads.seq[User](userReads))
)(User)

implicit lazy val userWrites: Writes[User] = (
  (__ \ "name").write[String] and
  (__ \ "friends").lazyWrite(Writes.seq[User](userWrites))
)(unlift(User.unapply))
```

## Format

`Format[T]` 是 `Reads` 和 `Writes` 组合的特性（trait），它可以代替 `Reads` 和 `Writes` 来做隐式转换。

### 通过 Reads 和 Writes 创建 Format

你可以通过 `Reads` 和 `Writes` 来构建针对同一类型的 `Format`：

```scala
val locationReads: Reads[Location] = (
  (JsPath \ "lat").read[Double](min(-90.0) keepAnd max(90.0)) and
  (JsPath \ "long").read[Double](min(-180.0) keepAnd max(180.0))
)(Location.apply _)

val locationWrites: Writes[Location] = (
  (JsPath \ "lat").write[Double] and
  (JsPath \ "long").write[Double]
)(unlift(Location.unapply))

implicit val locationFormat: Format[Location] =
  Format(locationReads, locationWrites)
```

### 使用组合子创建 Format

对于 `Reads` 和 `Writes` 对称的情况（实际应用中不一定能满足对称的条件），你可以直接通过组合子来创建 Format：

```scala
implicit val locationFormat: Format[Location] = (
  (JsPath \ "lat").format[Double](min(-90.0) keepAnd max(90.0)) and
  (JsPath \ "long").format[Double](min(-180.0) keepAnd max(180.0))
)(Location.apply, unlift(Location.unapply))
```
