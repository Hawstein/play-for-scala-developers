# JSON with HTTP

通过 HTTP API 与 JSON 库的组合，Play 可以支持 Content-Type 为 JSON 的 HTTP 请求和响应。

> 关于控制器，Action 和路由，详情可见 [HTTP 编程](https://www.playframework.com/documentation/2.3.x/ScalaActions)

我们通过设计一个简单的、Restful 的 web 服务来说明一些必要的概念，通过 GET 来得到实体列表，POST 来创建新的实体。对于所有数据，该 web 服务使用的 Content-Type 均为 JSON。

以下是用于我们服务的模型：

```scala
case class Location(lat: Double, long: Double)

case class Place(name: String, location: Location)

object Place {

  var list: List[Place] = {
    List(
      Place(
        "Sandleford",
        Location(51.377797, -1.318965)
      ),
      Place(
        "Watership Down",
        Location(51.235685, -1.309197)
      )
    )
  }

  def save(place: Place) = {
    list = list ::: List(place)
  }
}
```

## 以 JSON 格式提供实体列表

首先，在控制器中导入必要的东西：

```scala
import play.api.mvc._
import play.api.libs.json._
import play.api.libs.functional.syntax._

object Application extends Controller {

}
```

在写 `Action` 之前，我们先要处理模型到 `JsValue` 转换的问题，通过定义一个隐式的 `Writes[Place]` 即可。

```scala
implicit val locationWrites: Writes[Location] = (
  (JsPath \ "lat").write[Double] and
  (JsPath \ "long").write[Double]
)(unlift(Location.unapply))

implicit val placeWrites: Writes[Place] = (
  (JsPath \ "name").write[String] and
  (JsPath \ "location").write[Location]
)(unlift(Place.unapply))
```

接着就可以写 `Action` 了：

```scala
def listPlaces = Action {
  val json = Json.toJson(Place.list)
  Ok(json)
}
```

`Action` 拿到一个包含 `Place` 对象的列表，使用 `Json.toJson` 将它们转换为 `JsValue`（用的是隐式 `Writes[Place]`），然后将这个作为结果的 body 返回。Play 识别出该结果是 JSON 格式，然后为响应设置适当的 `Content-Type` 和 body。

最后一步是为我们的 `Action` 添加路由，写在 `conf/routes` 中：

```scala
GET   /places               controllers.Application.listPlaces
```

我们可以通过浏览器或 HTTP 工具来发送请求进行测试，下面我们通过 curl 进行测试：

```shell
curl --include http://localhost:9000/places
```

响应是：

```shell
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
Content-Length: 141

[{"name":"Sandleford","location":{"lat":51.377797,"long":-1.318965}},{"name":"Watership Down","location":{"lat":51.235685,"long":-1.309197}}]
```

## 创建新实体

对于接下来的 `Action`，我们需要定义一个隐式的 `Reads[Place]` 来将 `JsValue` 转换成我们的模型。

```scala
implicit val locationReads: Reads[Location] = (
  (JsPath \ "lat").read[Double] and
  (JsPath \ "long").read[Double]
)(Location.apply _)

implicit val placeReads: Reads[Place] = (
  (JsPath \ "name").read[String] and
  (JsPath \ "location").read[Location]
)(Place.apply _)
```

然后，我们来定义这个 `Action`。

```scala
def savePlace = Action(BodyParsers.parse.json) { request =>
  val placeResult = request.body.validate[Place]
  placeResult.fold(
    errors => {
      BadRequest(Json.obj("status" ->"KO", "message" -> JsError.toFlatJson(errors)))
    },
    place => {
      Place.save(place)
      Ok(Json.obj("status" ->"OK", "message" -> ("Place '"+place.name+"' saved.") ))
    }
  )
}
```

这个 `Action` 比前面那个要复杂，需要注意以下几点：

* 该 `Action` 接收的请求的 `Content-Type` 需要是 `text/json` 或 `application/json`，body 包含的是要创建的实体的 JSON 表示。
* 它使用针对 JSON 的 `BodyParser` 来解析请求，并将 `request.body` 解析成 `JsValue`。
* 我们使用 `validate` 方法来做转换，它依赖于前面定义的隐式 `Reads[Place]`。
* 我们使用一个带有错误和成功处理的 `fold` 来处理 `validate` 的结果。这种模式也可以用于表单提交。
* 该 `Action` 发送的响应也是 JSON 格式的。

最后我们在 `conf/routes` 中加上路由绑定：

```scala
POST  /places               controllers.Application.savePlace
```

下面我们用有效及无效的请求来测试这个 action，以验证成功及错误处理的工作流。

使用有效数据测试：

```scala
curl --include
  --request POST
  --header "Content-type: application/json"
  --data '{"name":"Nuthanger Farm","location":{"lat" : 51.244031,"long" : -1.263224}}'
  http://localhost:9000/places
```

响应：

```scala
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
Content-Length: 57

{"status":"OK","message":"Place 'Nuthanger Farm' saved."}
```

使用无效数据测试（“name” 字段缺失）：

```scala
curl --include
  --request POST
  --header "Content-type: application/json"
  --data '{"location":{"lat" : 51.244031,"long" : -1.263224}}'
  http://localhost:9000/places
```

响应：

```scala
HTTP/1.1 400 Bad Request
Content-Type: application/json; charset=utf-8
Content-Length: 79

{"status":"KO","message":{"obj.name":[{"msg":"error.path.missing","args":[]}]}}
```

使用无效数据测试（“lat” 数据类型错误）：

```scala
curl --include
  --request POST
  --header "Content-type: application/json"
  --data '{"name":"Nuthanger Farm","location":{"lat" : "xxx","long" : -1.263224}}'
  http://localhost:9000/places
```

响应：

```scala
HTTP/1.1 400 Bad Request
Content-Type: application/json; charset=utf-8
Content-Length: 92

{"status":"KO","message":{"obj.location.lat":[{"msg":"error.expected.jsnumber","args":[]}]}}
```

## 总结

Play 天生支持 REST 和 JSON，因此开发此类服务应该是相当简单直观的。大部分的工作就是在为你的模型写 `Reads` 和 `Writes`，下一节我们将来详细介绍。
