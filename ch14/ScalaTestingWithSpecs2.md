# Testing with specs2
# 使用specs2来测试你的应用 #

为你的应用编写测试是一个参与的过程。Play为你提供了一个默认的测试框架，并提供了帮助器和应用存根使测试你的应用尽可能的简单。


## 概述 ##

测试文件的位置在"test"文件夹中。这里有两个简单的测试文件其可以被用作为模板。

你可以从Play控制台运行测试。

	点击test来按钮运行所有测试。
	点击标注测试类名字比如：test-only my.namespace.MySpec的test-only按钮来运行一个测试类。
	点击test-quick按钮来运行会失败的测试类。
	运行一个前面带波浪线的的命令比如~test-quick来持续的运行测试。
	点击test:console按钮来在控制台中访问测试帮助器如FakeApplication.


Testing in Play is based on SBT, and a full description is available in the testing SBT chapter.
在Play中测试是基于SBT，完整的描述请参考[testing SBT](http://www.scala-sbt.org/0.13.0/docs/Detailed-Topics/Testing "testing SBT")章节。

## 使用specs2 ##

在specs2中，测试组织成规格，其包含了运行基于不同代码路径运行的测试的系统。

规格扩展了Specification特性big使用should/in格式：


	import org.specs2.mutable._

	class HelloWorldSpec extends Specification {

  		"The 'Hello world' string" should {
    		"contain 11 characters" in {
      		"Hello world" must have size(11)
    	}
    	"start with 'Hello'" in {
      		"Hello world" must startWith("Hello")
    	}
    	"end with 'world'" in {
      		"Hello world" must endWith("world")
    		}
  		}
	}

规格可以在IntelliJ IDEA(使用Scala插件)或Eclipse（使用Scala IDE）中运行，更多细节请参考[IDE网页](https://www.playframework.com/documentation/2.3.x/IDE "IDE页面")。

注意： 基于[展示编译器](https://scala-ide-portfolio.assembla.com/spaces/scala-ide/support/tickets/1001843-specs2-tests-with-junit-runner-are-not-recognized-if-there-is-package-directory-mismatch#/activity/ticket: "展示编译器")的一个漏洞，在Eclipse中测试必须被定义为一个特定的格式：


	包名字必须与目录路径名完全一致。
	规格必须与@RunWith(classOf[JUnitRunner])一起声明。

这里一个Eclipse中可用的规格：
	package models // 这个文件必须存在于一个名字为“models”的目录中

	import org.specs2.mutable._
	import org.specs2.runner._
	import org.junit.runner._

	@RunWith(classOf[JUnitRunner])
	class ApplicationSpec extends Specification {
  		...
	}

## 适配器 ##

当你使用一个示例，你必须返回一个示例结果，通常，你将看到一个包含mast字段的声明：
	"Hello world" must endWith("world")
这个跟随must关键词的表达式被称为matchers.适配器返回一个示例结果，通常成功或失败。这个示例不会被编译如果其不会返回一个结果。
最有用的适配器是[匹配结果](http://etorreborre.github.io/specs2/guide/org.specs2.guide.Matchers.html#Match+results "匹配结果")。用来检测相等性，判断部分和两者其一的结果，甚至检测是否抛出异常。
这里同时有部分性适配器其允许在测试中使用XML和JSON匹配。

## Mockito ##

Mocks 用来隔离单元测试和外部依赖。例如，如果你的类以来一个外部的DataService类，你可以针对你的类输入适当的数据而不需要实例化一个DataService对象。

Mockito继承与specs2中作为默认的mocking库。
为了使用Mockito,添加以下的引用到你的程序中：


	import org.specs2.mock._

	你可以模拟出引用类如：
	trait DataService {
  		def findData: Data
	}

	case class Data(retrievalDate: java.util.Date)

	import org.specs2.mock._
	import org.specs2.mutable._

	import java.util._

	class ExampleMockitoSpec extends Specification with Mockito {

  		"MyService#isDailyData" should {
    		"return true if the data is from today" in {
      			val mockDataService = mock[DataService]
      			mockDataService.findData returns Data(retrievalDate = new java.util.Date())

      			val myService = new MyService() {
        		override def dataService = mockDataService
      		}

      		val actual = myService.isDailyData
      		actual must equalTo(true)
    		}
  		}
  
	}

Mocking 在测试类的公共方法时尤其有效。Mocking对象和私有方法也可以但是异常困难。

## 使用测试模块 ##
Play不需要模块来使用特定的数据库数据访问层。设置如果应用使用Anorm或Slick,这个模块内部将拥有一个针对数据库访问的引用。

	import anorm._
	import anorm.SqlParser._

	case class User(id: String, name: String, email: String) {
   		def roles = DB.withConnection { implicit connection =>
      		...
    	}
	}

针对单元测试，这种方式可以技巧的模拟出roles方法。
一个通用的方式是保持模块从数据库中分离出来并且尽可能的逻辑化，并抽象出基于一个库层的数据库访问。

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

以这种方式，isAdmin方法可以通过模拟出UserRepository引用并传递其到服务中来被测试：

	object UserServiceSpec extends Specification with Mockito {

  		"UserService#isAdmin" should {
    		"be true when the role is admin" in {
      			val userRepository = mock[UserRepository]
      			userRepository.roles(any[User]) returns Set(Role("ADMIN"))

      			val userService = new UserService(userRepository)
      			val actual = userService.isAdmin(User("11", "Steve", "user@example.org"))
      			actual must beTrue
    			}
  			}
		}

## 单元测试控制器 ##

Controllers are defined as objects in Play, and so can be trickier to unit test. In Play this can be alleviated by dependency injection using getControllerInstance. Another way to finesse unit testing with a controller is to use a trait with an explicitly typed self reference to the controller:
在Play中控制器被定义为对象，因此更难被单元测试使用。在Play中这可以通过依赖注入使用getControllerInstance来缓解。另一种方式去处理有一个控制器的单元测试是针对这个控制器使用一个有[显示的类型自我引用](http://www.naildrivin5.com/scalatour/wiki_pages/ExplcitlyTypedSelfReferences)的特性。

	trait ExampleController {
  		this: Controller =>

  		def index() = Action {
    		Ok("ok")
  		}
	}

	object ExampleController extends Controller with ExampleController

	然后测试该特性：
	import play.api.mvc._
	import play.api.test._
	import scala.concurrent.Future

	object ExampleControllerSpec extends PlaySpecification with Results {

  		class TestController() extends Controller with ExampleController

  		"Example Page#index" should {
    		"should be valid" in {
      			val controller = new TestController()
      			val result: Future[Result] = controller.index().apply(FakeRequest())
      			val bodyText: String = contentAsString(result)
      			bodyText must be equalTo "ok"
    		}
  		}
	}


## 单元测试EssentialAction ##

测试Action或Filter需要测试一个EssentialAction([更详细的信息请参见什么是EssentialAction](https://www.playframework.com/documentation/2.3.x/HttpApi "更详细的信息请参见什么是EssentialAction"))

对此，这个测试Helpers.call可是这样使用：

	object ExampleEssentialActionSpec extends PlaySpecification {

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