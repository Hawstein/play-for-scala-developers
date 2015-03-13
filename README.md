# Play 中文文档

[![Join the chat at https://gitter.im/Hawstein/play-for-scala-developers](https://badges.gitter.im/Join%20Chat.svg)](https://gitter.im/Hawstein/play-for-scala-developers?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)

Play 的 Scala API 在 play.api 包中。

> play 包中的 API 是为 Java 开发者准备的，比如 play.mvc。对于 Scala 开发者，请使用 play.api.mvc。

## 参与翻译

首先，你得有个 Github 账号，然后：

1. 打开[这个项目](https://github.com/Hawstein/play-for-scala-developers)，猛戳「Fork」按钮
1. 把 Fork 到你账号下的项目 Clone 到本地：git clone git@github.com:你的账号名/play-for-scala-developers.git
1. 在项目目录下，创建一个新分支来工作，比如新分支名叫 dev，则：git branch dev
1. 切换到新分支：git checkout dev
1. 运行 git remote add upstream https://github.com/Hawstein/play-for-scala-developers.git 把原始项目库添加为上游库，用于同步最新内容
1. 在 dev 分支下进行翻译工作，比如你在 ScalaRock.md 上做修改（Play Doc：https://www.playframework.com/documentation/2.3.x/ScalaHome）
1. 提交你的工作：git add ScalaRock.md，然后 git commit -m '翻译 ScalaRock'。在这些过程中，你都可以用 git status 查看状态
1. 运行 git remote update 更新
1. 运行 git fetch upstream master 拉取上游库的更新到本地
1. 运行 git rebase upstream/master 将上游库的更新合并到你的 dev 分支
1. 运行 git push origin master 将你的提交 push 到你的库中
1. 登录 Github，在你 Fork 的项目页有个 「Pull Request」按钮，点击它，填写一些说明，然后提交
1. 重复步骤 6～12

为了避免多人重复翻译同一章节，在翻译前先进行章节的认领。可以在帖子 http://scalachina.org/topic/5501c59784ddfe6644e8c8d4 回复里认领相应章节，也可以加 QQ 群，在群里说。群号：312213800。我会把认领的章节更新到这个帖子和项目的 README 里。

## 文档目录

文档地址：https://www.playframework.com/documentation/2.3.x/ScalaHome

* [HTTP programming](ch01/ScalaActions.md)
  * [Actions, Controllers and Results](ch01/ScalaActions.md)    认领人：[Hawstein](https://github.com/Hawstein)
  * [HTTP routing](ch01/ScalaRouting.md)    认领人：[boulevard23](https://github.com/boulevard23)
  * [Manipulating results](ch01/ScalaResults.md)
  * [Session and Flash scopes](ch01/ScalaSessionFlash.md)
  * [Body parsers](ch01/ScalaBodyParsers.md)
  * [Actions composition](ch01/ScalaActionsComposition.md)
  * [Content negotiation](ch01/ScalaContentNegotiation.md)
* [Asynchronous HTTP programming](ch02/ScalaAsync.md)
  * [Handling asynchronous results](ch02/ScalaAsync.md)
  * [Streaming HTTP responses](ch02/ScalaStream.md)
  * [Comet sockets](ch02/ScalaComet.md)
  * [WebSockets](ch02/ScalaWebSockets.md)
* [The template engine](ch03/ScalaTemplates.md)
  * [Templates syntax](ch03/ScalaTemplates.md)
  * [Common use cases](ch03/ScalaTemplateUseCases.md)
  * [Custom format addition](ch03/ScalaCustomTemplateFormat.md)
* [Form submission and validation](ch04/ScalaForms.md)
  * [Handling form submission](ch04/ScalaForms.md)
  * [Protecting against CSRF](ch04/ScalaCsrf.md)
  * [Custom Validations](ch04/ScalaCustomValidations.md)
  * [Custom Field Constructors](ch04/ScalaCustomFieldConstructors.md)
* [Working with Json](ch05/ScalaJson.md)
  * [JSON basics](ch05/ScalaJson.md)
  * [JSON with HTTP](ch05/ScalaJsonHttp.md)
  * [JSON Reads/Writes/Format Combinators](ch05/ScalaJsonCombinators.md)
  * [JSON Transformers](ch05/ScalaJsonTransformers.md)
  * [JSON Macro Inception](ch05/ScalaJsonInception.md)
* [Working with XML](ch06/ScalaXmlRequests.md)
  * [Handling and serving XML requests](ch06/ScalaXmlRequests.md)
* [Handling file upload](ch07/ScalaFileUpload.md)
  * [Direct upload and multipart/form-data](ch07/ScalaFileUpload.md)
* [Accessing an SQL database](ch08/ScalaDatabase.md)
  * [Configuring and using JDBC](ch08/ScalaDatabase.md)
  * [Using Anorm to access your database](ch08/ScalaAnorm.md)
  * [Integrating with other database access libraries](ch08/ScalaDatabaseOthers.md)
* [Using the Cache](ch09/ScalaCache.md)
  * [The Play cache API](ch09/ScalaCache.md)
* [Calling WebServices](ch10/ScalaWS.md)
  * [The Play WS API](ch10/ScalaWS.md)
  * [Connecting to OpenID services](ch10/ScalaOpenID.md)
  * [Accessing resources protected by OAuth](ch10/ScalaOAuth.md)
* [Integrating with Akka](ch11/ScalaAkka.md)
  * [Setting up Actors and scheduling asynchronous tasks](ch11/ScalaAkka.md)
* [Internationalization](ch12/ScalaI18N.md)
  * [Messages externalisation and i18n](ch12/ScalaI18N.md)
* [The application Global object](ch13/ScalaGlobal.md)
  * [Application global settings](ch13/ScalaGlobal.md)
  * [Intercepting requests](ch13/ScalaInterceptors.md)
* [Testing your application](ch14/ScalaTestingYourApplication.md)
  * [Testing with ScalaTest](ch14/ScalaTestingYourApplication.md)
  * [Writing functional tests with ScalaTest](ch14/ScalaFunctionalTestingWithScalaTest.md)
  * [Testing with specs2](ch14/ScalaTestingWithSpecs2.md)
  * [Writing functional tests with specs2](ch14/ScalaFunctionalTestingWithSpecs2.md)
* [Logging](ch15/ScalaLogging.md)
  * [The Logging API](ch15/ScalaLogging.md)
