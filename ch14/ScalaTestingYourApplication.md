## 使用specs2编写一个功能测试 ##
Play提供了一批类和简洁的方法可以用来帮助进行功能测试。其中的大多数可以在
Play.api.test包或在Helpers对象中找到。
你可以通过引入如下模块来添加这些方法和类：

	import play.api.test._
	import play.api.test.Helpers._

## FakeApplication ##

Play经常需要一个运行的Application作为上下文：其通常由Play.api.Play.current提供。

为了针对测试提供一个环境。Play提供了一个FakeApplication雷其可以和一个不同的全局对象，其他配置甚至其他插件一起被配置。


	val fakeApplicationWithGlobal = FakeApplication(withGlobal = Some(new GlobalSettings() {
  		override def onStart(app: Application) { println("Hello world!") }
	}))

## WithApplication ##

为了传递一个应用到一个例子，请使用WithApplication.一个显式的
FakeApplication可以被传递进去，但是默认的FakeApplication为了方便使用而被提供。
因为WithAppication是一个嵌入的Around块，你可以重写它来提供你自己的数据全域：

	abstract class WithDbData extends WithApplication {
  		override def around[T: AsResult](t: => T): Result = super.around {
    		setupData()
    		t
  		}

  		def setupData() {
    		// 配置数据
  		}
	}

	"Computer model" should {

  		"be retrieved by id" in new WithDbData {
    		// 你的测试代码
  		}
  		"be retrieved by email" in new WithDbData {
    		// 你的测试代码
  		}
	}

## WithServer ##

有时候你想从你的测试中测试真正的HTTP协议栈，在这种例子中你可以使用WithServer来启用一个测试服务器：

	"test server logic" in new WithServer(app = fakeApplicationWithBrowser, port = testPort) {
		// 这个测试支付网关在其返回一个结果前需要针对该服务器的一个回调...
  		val callbackURL = s"http://$myPublicAddress/callback"

  		// await is from play.api.test.FutureAwaits
  		val response = await(WS.url(testPaymentGatewayURL).withQueryString("callbackURL" -> callbackURL).get())

  		response.status must equalTo(OK)
		}
port值包含了服务器上运行的这个端口号。其默认的是19001，甚至你可以通过传递这个端口到WinthServer中或设置系统属性testserver.port来改变它。这个在与持续集成服务器结合的时候非常有用，所以这个端口可以为每个构建而动态保留。
一个FakeApplication也可以被传递到测试服务器，当设定定制的路由和测试WS调用时会非常有用：


	val appWithRoutes = FakeApplication(withRoutes = {
  		case ("GET", "/") =>
    	Action {
      	Ok("ok")
    	}
	})

	"test WS logic" in new WithServer(app = appWithRoutes, port = 3333) {
  		await(WS.url("http://localhost:3333").get()).status must equalTo(OK)
	}

## WithBrowser ##

如果你想使用一个浏览器来测试你的应用，你可以使用[Selenium WebDriver](http://code.google.com/p/selenium/?redir=1 "Selenium WebDriver").Play将为你启用该WebDriver,并使用FluentLenium提供的简洁的API来包裹它。如同WithServer一样使用WithBrowser，你可以更改端口，FakeApplication；而且你可以通过选择网页浏览器而使用：

	val fakeApplicationWithBrowser = FakeApplication(withRoutes = {
  		case ("GET", "/") =>
    	Action {
      	Ok(
        	"""
          	|<html>
          	|<body>
          	|  <div id="title">Hello Guest</div>
          	|  <a href="/login">click me</a>
          	|</body>
          	|</html>
        	""".stripMargin) as "text/html"
    	}
  		case ("GET", "/login") =>
    		Action {
      			Ok(
        			"""
          			|<html>
          			|<body>
          			|  <div id="title">Hello Coco</div>
          			|</body>
          			|</html>
        			""".stripMargin) as "text/html"
    			}
			})

		"run in a browser" in new WithBrowser(webDriver = WebDriverFactory(HTMLUNIT), app = fakeApplicationWithBrowser) {
  			browser.goTo("/")

  			// 检查页面
  			browser.$("#title").getTexts().get(0) must equalTo("Hello Guest")

  			browser.$("a").click()

  			browser.url must equalTo("/login")
  			browser.$("#title").getTexts().get(0) must equalTo("Hello Coco")
		}

## PlaySpecification ##

PlaySpecification是specification的一个扩展它不包含被默认的specs2规范所提供而与Play 帮助器方法相冲突的一些混合对象。为了便利它也混合进Play测试帮助器和类型。

	object ExamplePlaySpecificationSpec extends 	PlaySpecification {
  			"The specification" should {

    			"have access to HeaderNames" in {
      				USER_AGENT must be_===("User-Agent")
    				}

    				"have access to Status" in {
      					OK must be_===(200)
    					}
  					}
			}


## 测试一个view模板 ##

因为一个模板是一个标准的Scala功能，你可以从你的测试中执行它，并检测结果：

	"render index template" in new WithApplication {
  		val html = views.html.index("Coco")

  		contentAsString(html) must contain("Hello Coco")
	}


## 测试一个控制器 ##
你可以调用任何被一个FakeRequest所提供的任何Action:

	"respond to the index Action" in {
  		val result = controllers.Application.index()(FakeRequest())

  		status(result) must equalTo(OK)
  		contentType(result) must beSome("text/plain")
  		contentAsString(result) must contain("Hello Bob")
	}

技术上讲，你这里不需要WithApplication，尽管使用它并不会带来不利影响。

## 测试路由器 ##

取代自己调用Action，你可以让Router来做：


	"respond to the index Action" in new WithApplication(fakeApplication) {
  		val Some(result) = route(FakeRequest(GET, "/Bob"))

  		status(result) must equalTo(OK)
  		contentType(result) must beSome("text/html")
  		charset(result) must beSome("utf-8")
  		contentAsString(result) must contain("Hello Bob")
	}

## 测试一个模块 ##

如果你使用一个SQL数据库，你可以使用inMemoryDatabase来替换与一个数据库连接的H2数据库中的内存中实例。
	val appWithMemoryDatabase = FakeApplication(additionalConfiguration = inMemoryDatabase("test"))
	"run an application" in new WithApplication(appWithMemoryDatabase) {

  		val Some(macintosh) = Computer.findById(21)

  		macintosh.name must equalTo("Macintosh")
  		macintosh.introduced must beSome.which(_ must beEqualTo("1984-01-24"))
	}