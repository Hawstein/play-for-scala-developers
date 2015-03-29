# Writing functional tests with ScalaTest
## 用ScalaTest写功能测试 ##
为你的应用编写测试可是一个复杂的过程。Play提供了帮助手册和应用存根。并且ScalaTest提供了一个整合库，[ScalaTest + Play](http://scalatest.org/plus/play "ScalaTest + Play"),使测试你的应用变得尽可能简单。


# 总览 #
测试文件位于“test”目录下。


你可以通过Play控制台运行测试。

	点击test运行所有测试。
	点击类名后跟随的test-only，比如，test-only my.namespace.MySpec 来运行一个测试类。
	点击test-quick运行只有已经失败的测试。
	输入一个前面带~的命令，比如 ~test-quick来持续性的运行测试。
	运行test:console来在控制台中查看测试帮助手册如 FakeApplication.
在Play中测试是基于SBT,[testing SBT](http://www.scala-sbt.org/0.13.0/docs/Detailed-Topics/Testing "testing SBT")章节中提供了更加详细的信息。
## 使用 ScalaTest + Play ##
   
为了能够使用ScalaTest + Play, 你将需要将它添加到你的构建中，通过改变projects/Build.scala 例如：

	val appDependencies = Seq(
  		// 在此处添加你的项目依赖,
  		"org.scalatestplus" %% "play" % "1.1.0" % "test"
	)
你不需要显性的添加ScalaTest到你的构建中。适当版本的ScalaTest会作为ScalaTest + Play的一个过渡的依赖关系。你将需要选择一个和你的Play版本匹配的一个版本的ScalaTest + Play，你可以通过ScalaTest + Play的[versions](http://www.scalatest.org/plus/play/versions "Versions")页面。

[ScalaTest + Play](http://scalatest.org/plus/play "ScalaTest + Play")中，你可以通过扩展PlaySpec的特性来定义测试类。

例如这个例子：

	import collection.mutable.Stack
	import org.scalatestplus.play._

	class StackSpec extends PlaySpec {

  		"A Stack" must {
    		"pop values in last-in-first-out order" in {
      		val stack = new Stack[Int]
      		stack.push(1)
      		stack.push(2)
      		stack.pop() mustBe 2
      		stack.pop() mustBe 1
    		}
    		"throw NoSuchElementException if an empty stack is popped" in {
    	  		val emptyStack = new Stack[Int]
    	  		a [NoSuchElementException] must be thrownBy {
    	    		emptyStack.pop()
    	  			}
    			}
  			}
		}
你或者可以通过[定义自己的基类](http://scalatest.org/user_guide/defining_base_classes "定义自己的基类")来替换使用PlaySpec.

你可以与Play一起或在IntelliJ IDEA(使用[Scala plugin](http://blog.jetbrains.com/scala/ "Scala Plugin"))或Eclipse(使用[Scala IDE](http://scala-ide.org/ "Scala IDE")和 [ScalaTest Eclipse plugin](http://scalatest.org/user_guide/using_scalatest_with_eclipse "ScalaTest Eclipse plugin"))中运行你的测试。请通过[IDE页面](https://www.playframework.com/documentation/2.3.x/IDE "IDE 页面")来获取更多详细信息。

## 适配器 ##

PlaySpec混合了ScalaTest的MustMatchers,所以你可以通过使用ScalaTest的适配器DSL来写声明变量：

	“Hello world” must endWith ("world")

请参考MustMatchers的文档来获取更多信息。

## Mockito ##

你可以通过使用mocks来隔离单元测试需要的外部依赖。例如，如果你的类依赖于一个外部类DataService,你可以为你的类采集适当的数据而不需要实例化一个DataService对象。

ScalaTest通过MockitoSuger特性提供与[Mockito](https://code.google.com/p/mockito/ "Mockito")集成。

为了使用Mockito，混合MockitoSUger到你的测试类中然后使用Mockito库来模拟依赖关系：



	case class Data(retrievalDate: java.util.Date)

	trait DataService {
  		def findData: Data
	}

	import org.scalatest._
	import org.scalatest.mock.MockitoSugar
	import org.scalatestplus.play._

	import org.mockito.Mockito._

	class ExampleMockitoSpec extends PlaySpec with MockitoSugar {

  		"MyService#isDailyData" should {
    		"return true if the data is from today" in {
      			val mockDataService = mock[DataService]
      			when(mockDataService.findData) thenReturn Data(new java.util.Date())

      			val myService = new MyService() {
        			override def dataService = mockDataService
      			}

      			val actual = myService.isDailyData
      			actual mustBe true
    			}
  			}
		}

模拟对于测试类的公共方法是非常有用的。模拟对象和私有方法是可能的，但是难以实现。

## 单元测试模型 ##


Play不需要通过模式去使用一个特定的数据库数据访问层。而且，如果应用使用Anorm或Slick,该模式往往将拥有一个数据库访问的内部引用。

	import anorm._
	import anorm.SqlParser._

	case class User(id: String, name: String, email: String) {
   		def roles = DB.withConnection { implicit connection =>
     	...
    	}
	}


对于单元测试，这种方法可以微妙的模拟roles方法。

一个通用的方法是保持模式尽可能的逻辑上从数据库分离，并抽象数据库的访问于一个仓库层的后面。



	case class Role(name:String)

	case class User(id: String, name: String, email:String)

	trait UserRepository {
  		def roles(user:User) : Set[Role]
	}

	class AnormUserRepository extends UserRepository {
  		import anorm._
  		import anorm.SqlParser._

  		def roles(user:User) : Set[Role] = {
    		...
  		}
	}

然后通过服务访问它们：

	class UserService(userRepository : UserRepository) {

  		def isAdmin(user:User) : Boolean = {
    		userRepository.roles(user).contains(Role("ADMIN"))
  		}
	}

通过这种方式， isAdmin方法可以通过模拟出UserRepository引用并传递其到该服务中来测试：

	class UserServiceSpec extends PlaySpec with MockitoSugar {

  		"UserService#isAdmin" should {
    		"be true when the role is admin" in {
      			val userRepository = mock[UserRepository]
      			when(userRepository.roles(any[User])) thenReturn Set(Role("ADMIN"))

      			val userService = new UserService(userRepository)

      			val actual = userService.isAdmin(User("11", "Steve", "user@example.org"))
      			actual mustBe true
    			}
  			}
		}


## 单元测试控制器 ##
在Play中控制器被定义为对象，所以更难来进行单元测试。在Play中可以通过[依赖注入](https://www.playframework.com/documentation/2.3.x/ScalaDependencyInjection "依赖注入")使用getControllerInstance来缓解。另一种方式去巧妙处理包含一个控制器的单元测试是针对这个控制器使用一个[隐式类型自引用](http://www.naildrivin5.com/scalatour/wiki_pages/ExplcitlyTypedSelfReferences "隐式类型自引用")的一种特质：


	trait ExampleController {
  		this: Controller =>

  		def index() = Action {
    		Ok("ok")
  		}
	}

	object ExampleController extends Controller with ExampleController

并接着测试这个特质：

	import scala.concurrent.Future

	import org.scalatest._
	import org.scalatestplus.play._
	
	import play.api.mvc._
	import play.api.test._
	import play.api.test.Helpers._

	class ExampleControllerSpec extends PlaySpec with Results {

  		class TestController() extends Controller with ExampleController

  		"Example Page#index" should {
    		"should be valid" in {
      		val controller = new TestController()
      		val result: Future[SimpleResult] = controller.index().apply(FakeRequest())
      		val bodyText: String = contentAsString(result)
      		bodyText mustBe "ok"
    		}
  		}
	}

## 单元测试基本动作 ##
测试Action或Filter需要测试一个EssentialAction([关于什么是EssentialAction的详细信息](https://www.playframework.com/documentation/2.3.x/HttpApi "关于什么是EssentialAction的详细信息"))

对此，测试Helpers.call可以像这样使用：

	class ExampleEssentialActionSpec extends PlaySpec {

  		"An essential action" should {
    		"can parse a JSON body" in {
      			val action: EssentialAction = Action { request =>
        		val value = (request.body.asJson.get \ "field").as[String]
        		Ok(value)
      		}

      			val request = FakeRequest(POST, "/").withJsonBody(Json.parse("""{ "field": "value" }"""))

     			 val result = call(action, request)

      			status(result) mustEqual OK
      			contentAsString(result) mustEqual "value"
    		}
  		}
	}