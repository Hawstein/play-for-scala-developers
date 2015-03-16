# JSON 基础

现代Web应用程序经常需要解析和生成JSON（JavaScrpit Object Notation）格式的数据。Play框架通过JSON库的支持可以完成上述任务。

JSON是一种轻量级的数据交换格式，请看下面一个例子：

```
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
```

如果想了解跟多关于JSON的知识，请访问[json.org](http://json.org/)

## Play框架的JSON库
`play.api.libs.json`包中包含表示JSON数据的数据结构和用于将这些数据结构与其他数据格式互相转换的实用工具。

### JsValue
这是一个特质（trait），可以表示任何JSON值。JSON库中通过一系列对JsVlaue进行扩展的case类来表示各种有效的JSON类型。

- JsString
- JsNumber
- JsBoolean
- JsObject
- JsArray
- JsNull

你可以利用这些多种多样的JsValue类型来构造任何JSON结构。

### Json

Json对象提供一些工具，这些工具用于将数据格式转换为JsValue结构或者逆向转换。

### JsPath

JsPath用于表示JsValue内部结构的路径，类似于XPath对XML的意义。可以利用JsPath析取JsValue结构，或者对隐式转换进行模式匹配。

## 构造JsValue实例

### 字符串解析

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
### 类构造器

```scala
import play.api.libs.json._

val json: JsValue = JsObject(Seq(
  "name" -> JsString("Watership Down"),
  "location" -> JsObject(Seq("lat" -> JsNumber(51.235685), "long" -> JsNumber(-1.309197))),
  "residents" -> JsArray(Seq(
    JsObject(Seq(
      "name" -> JsString("Fiver"),
      "age" -> JsNumber(4),
      "role" -> JsNull
    )),
    JsObject(Seq(
      "name" -> JsString("Bigwig"),
      "age" -> JsNumber(6),
      "role" -> JsString("Owsla")
    ))
  ))
))  
```

通过```Json.obj```和```Json.arr```构造可能更简单些。注意大部分值不需要显式得用JsValue类封装，工厂方法会执行隐式转换（接下来是一个例子）。

```scala
import play.api.libs.json.{JsNull,Json,JsString,JsValue}

val json: JsValue = Json.obj(
  "name" -> "Watership Down",
  "location" -> Json.obj("lat" -> 51.235685, "long" -> -1.309197),
  "residents" -> Json.arr(
    Json.obj(
      "name" -> "Fiver",
      "age" -> 4,
      "role" -> JsNull
    ),
    Json.obj(
      "name" -> "Bigwig",
      "age" -> 6,
      "role" -> "Owsla"
    )
  )
)
```
### Writers转换器
Scala中通过工具方法```Json.toJson[T](T)(implicit writes:Writes[T])```。这个功能通过类型转换器```Writes[T]```将T类型的数据转换为JsValue。

Play框架的JSON库API接口提供了大部分基础类型的隐式```Writes```，例如```Int```，```Double```，```String```和```Boolean```。当然，该JSON库也有针对包含上述基本类型元素的集合的```Writes```转换器。

```scala
import play.api.libs.json._

// basic types
val jsonString = Json.toJson("Fiver")
val jsonNumber = Json.toJson(4)
val jsonBoolean = Json.toJson(false)

// collections of basic types
val jsonArrayOfInts = Json.toJson(Seq(1, 2, 3, 4))
val jsonArrayOfStrings = Json.toJson(List("Fiver", "Bigwig"))
```

如果想把自己定义的模型转换成JsValues，你需要定义隐式的```Writes```转换器，并将它们引入执行环境。

```scala
import play.api.libs.json._

// basic types
val jsonString = Json.toJson("Fiver")
val jsonNumber = Json.toJson(4)
val jsonBoolean = Json.toJson(false)

// collections of basic types
val jsonArrayOfInts = Json.toJson(Seq(1, 2, 3, 4))
val jsonArrayOfStrings = Json.toJson(List("Fiver", "Bigwig"))
To convert your own models to JsValues, you must define implicit Writes converters and provide them in scope.

case class Location(lat: Double, long: Double)
case class Resident(name: String, age: Int, role: Option[String])
case class Place(name: String, location: Location, residents: Seq[Resident])
import play.api.libs.json._

implicit val locationWrites = new Writes[Location] {
  def writes(location: Location) = Json.obj(
    "lat" -> location.lat,
    "long" -> location.long
  )
}

implicit val residentWrites = new Writes[Resident] {
  def writes(resident: Resident) = Json.obj(
    "name" -> resident.name,
    "age" -> resident.age,
    "role" -> resident.role
  )
}

implicit val placeWrites = new Writes[Place] {
  def writes(place: Place) = Json.obj(
    "name" -> place.name,
    "location" -> place.location,
    "residents" -> place.residents)
}

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

作为备选，你可以通过配合模式（combinator pattern）来定义自己的```Writes```转换器。

```
注意：关于配合模式（combinator pattern）在[JSON Reads/Writes/Formats Combinators](https://www.playframework.com/documentation/2.3.x/ScalaJsonCombinators)一节中有详细介绍。
```

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
```
## 析取JsValue结构
你可以析取JsValue结构并读取特定的值。语法和功能类似于Scala处理XML的方法。

```
注意：下面的例子应用在前面创建的JsValue结构上
```

### Simple path```\```
将```\```操作符应用于一个```JsValue```可以返回跟相关域对应的属性。下面假设有一个JsObject：

```scala
val lat = json \ "location" \ "lat"
// 返回JsNumber(51.235685)
```
### Recursive path```\\```
使用```\\```操作符将会递归查找当前对象中以及所有依赖对象中的所有对应域。

```scala
val names = json \\ "name" 
// returns Seq(JsString("Watership Down"), JsString("Fiver"), JsString("Bigwig"))
```
### Index lookup(for JsArrays)
可以通过索引值从```JsArray```中获取值。

```scala
val bigwig = (json \ "residents")(1)
// returns {"name":"Bigwig","age":6,"role":"Owsla"}
```

## 解析JsValue结构
### 字符串工具
- 微型

```scala
val minifiedString: String = Json.stringify(json)
{"name":"Watership Down","location":{"lat":51.235685,"long":-1.309197},"residents":[{"name":"Fiver","age":4,"role":nul
```

- 可读的

```scala
val readableString: String = Json.prettyPrint(json)
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
```
### JsValue.as或者Jsvalue.asOpt
将JsValue对象转换成其他类型的最简单的方法是使用```JsValue.as[T](implicit fjs：Reads[T]):T``` 需要自定义一个类型转换器```Reads[T]```来讲```JsValue```转换成```T```类型的数据(```和Writes[T]相反```)。 跟```Writes```一样，JSON库提供了```Reads```转换器需要的基本类型。

```scala
val name = (json \ "name").as[String]
// "Watership Down"

val names = (json \\ "name").map(_.as[String])
// Seq("Watership Down", "Fiver", "Bigwig")
```

如果路径（path）不存在或者转换失败，```as```方法会抛出```JsResultException```异常。更安全的方法是使用```JsValue.asOpt[T](implicit fjs:Reads[T]):Option[T]```

```scala
val nameOption = (json \ "name").asOpt[String]
// Some("Watership Down")

val bogusOption = (json \ "bogus").asOpt[String]
// None
```
尽管asOpt方法更安全，但是如果有错误也不能捕捉到。

### 使用有效性（validation）
推荐使用```validate```方法将```JsValue```转换为其他类型(这个方法含有一个```Read```类型的参数)。这个方法同时执行有效性验证和类型转换操作，返回的结果类型是```JsResult```。```JsResult```通过两个类实现：

- JsSuccess——表示验证/转换成功并封装结果。
- JsError——表示验证/转换不成功，并包含有错误列表。

可以使用多种模式来处理验证结果:

```scala
val json = { ... }

val nameResult: JsResult[String] = (json \ "name").validate[String]

// Pattern matching
nameResult match {
  case s: JsSuccess[String] => println("Name: " + s.get)
  case e: JsError => println("Errors: " + JsError.toFlatJson(e).toString()) 
}

// Fallback value
val nameOrFallback = nameResult.getOrElse("Undefined")

// map
val nameUpperResult: JsResult[String] = nameResult.map(_.toUpperCase())

// fold
val nameOption: Option[String] = nameResult.fold(
  invalid = {
    fieldErrors => fieldErrors.foreach(x => {
      println("field: " + x._1 + ", errors: " + x._2)
    })
    None
  },
  valid = { 
    name => Some(name)
  }
)
```

### 由JsValue转换为模型（model）
如果要将JsValue转换为模型，你要定义隐式```Reads[T]```转换器，其中T是模型的类型。

```
注意：此处实现Reads所用的模式和自定义有效性的技术细节在[JSON Reads/Writes/Formats Combinators](https://www.playframework.com/documentation/2.3.x/ScalaJsonCombinators)一节中有详细介绍。
```

```scala
case class Location(lat: Double, long: Double)
case class Resident(name: String, age: Int, role: Option[String])
case class Place(name: String, location: Location, residents: Seq[Resident])
import play.api.libs.json._
import play.api.libs.functional.syntax._

implicit val locationReads: Reads[Location] = (
  (JsPath \ "lat").read[Double] and
  (JsPath \ "long").read[Double]
)(Location.apply _)

implicit val residentReads: Reads[Resident] = (
  (JsPath \ "name").read[String] and
  (JsPath \ "age").read[Int] and
  (JsPath \ "role").readNullable[String]
)(Resident.apply _)

implicit val placeReads: Reads[Place] = (
  (JsPath \ "name").read[String] and
  (JsPath \ "location").read[Location] and
  (JsPath \ "residents").read[Seq[Resident]]
)(Place.apply _)


val json = { ... }

val placeResult: JsResult[Place] = json.validate[Place]
// JsSuccess(Place(...),)

val residentResult: JsResult[Resident] = (json \ "residents")(1).validate[Resident]
// JsSuccess(Resident(Bigwig,6,Some(Owsla)),)
```

**下一节：** [JSON with HTTP](https://www.playframework.com/documentation/2.3.x/ScalaJsonHttp)
