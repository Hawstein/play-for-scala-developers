# Writing functional tests with specs2
# Writing functional tests with specs2
# 通过ScalaTest编写功能性测试 #


Play提供了一系列的类和简洁的方法来帮助实现功能性测试。大多数可以在play.api.test包或Helpers对象中找到。[ScalaTest + Play](http://doc.scalatest.org/plus-play/1.0.0/index.html#org.scalatestplus.play.package "ScalaTest + Play")集成库位ScalaTest构建了这种测试支持。

你可以通过以下的模块导入来访问所有的Play的内建测试支持和ScalaTest + Play:

	import org.scalatest._
	import play.api.test._
	import play.api.test.Helpers._
	import org.scalatestplus.play._

## FakeApplication ##

Play经常性需要一个运行的Application作为上下文：其通常通过play.api.Play.current提供。

为了为测试提供一个环境，Play提供了一个FakeApplication类其可以和一个不同的Global对象，附加配置或附加的插件一起被配置。


	val fakeApplicationWithGlobal = FakeApplication(withGlobal = Some(new GlobalSettings() {
  		override def onStart(app: Application) { println("Hello world!") }
	}))

如果所有或大多数你的测试类需要一个FakeApplication,并且它们可以共享相同的FakeApplication,与OneAppPerSuite特性混合使用。你可以通过定制FakeApplication来如果这个例子展示的那样重写PP:


	class ExampleSpec extends PlaySpec with OneAppPerSuite {
		//重写app如果你需要一个不仅有默认参数的FakeApplication。
  		implicit override lazy val app: FakeApplication =
    	FakeApplication(
      		additionalConfiguration = Map("ehcacheplugin" -> "disabled")
    	)

  		"The OneAppPerSuite trait" must {
    		"provide a FakeApplication" in {
      			app.configuration.getString("ehcacheplugin") mustBe Some("disabled")
    			}
    			"start the FakeApplication" in {
      				Play.maybeApplication mustBe Some(app)
    				}
  				}
		}

如果你需要每个测试去获取它自己的FakeApplication而不是共享相同的，请使用OneAppPerTest来替代：


	class ExampleSpec extends PlaySpec with OneAppPerTest {
		//重写app如果你需要一个不仅有默认参数的FakeApplication。
  		implicit override def newAppForTest(td: TestData): FakeApplication =
    	FakeApplication(
      		additionalConfiguration = Map("ehcacheplugin" -> "disabled")
    		)

  		"The OneAppPerTest trait" must {
    		"provide a new FakeApplication for each test" in {
      			app.configuration.getString("ehcacheplugin") mustBe Some("disabled")
    		}
    		"start the FakeApplication" in {
      			Play.maybeApplication mustBe Some(app)
    			}
  			}
		}，
ScalaTest + Play提供了OneAppPerSuite和OneAppPerTest的原因是来允许你去选择共享性策略来让你的测试最快的运行。如果你想让应用状态在成功的测试之间保持，你将需要使用OneAppPerSuite.如果每个测试需要一个清空的状态，你可以使用OneAppPerTest或使用OneAppPerSuite,但需要在每个测试结束来清理状态。甚至，如果你的测试集将尽可能快的运行多个测试类共享这个相同的应用，你可以定义一个主集合起与OneAppPerSuite混用和内嵌的集合其和ConfiguredApp混用，正如在[ConfiguredApp的文档](http://doc.scalatest.org/plus-play/1.0.0/index.html#org.scalatestplus.play.ConfiguredApp "ConfiguredApp的文档")中的例子展示的那样。你可以使用任意策略来实你的测试集最快的运行。

## 和一个服务器一起测试 ##
有时候你想使用真的HTTP栈来进行测试。如果你的测试类中的所有测试能够重用这个相同的服务器实例，你可以通过与OneServerPerSuite一起混用（其将为这个集合提供一个新的FakeApplication）：

	class ExampleSpec extends PlaySpec with OneServerPerSuite {
		// 重写app如果你需要一个不仅有默认参数的FakeApplication。
  		implicit override lazy val app: FakeApplication =
    	FakeApplication(
      		additionalConfiguration = Map("ehcacheplugin" -> "disabled"),
      		withRoutes = {
        		case ("GET", "/") => Action { Ok("ok") }
      		}
    	)

  	"test server logic" in {
    	val myPublicAddress =  s"localhost:$port"
    	val testPaymentGatewayURL = s"http://$myPublicAddress"
		//这个测试支付网关在它返回一个值之前需要针对这个服务器的一个回调...
    	val callbackURL = s"http://$myPublicAddress/callback"
    	// 从play.api.test.FutureAwaits中等待调用
    	val response = await(WS.url(testPaymentGatewayURL).withQueryString("callbackURL" -> callbackURL).get())

    	response.status mustBe (OK)
  		}
	}


如果你的测试类中的所有测试需要不同的服务器实例，请使用OneServerPerTest来代替（其将同时为这个测试集提供一个新的FakeApplication）:

	class ExampleSpec extends PlaySpec with OneServerPerTest {
		//重写newAppForTest如果你需要一个不仅有默认参数的FakeApplication.
  		override def newAppForTest(testData: TestData): FakeApplication =
   	 		new FakeApplication(
      			additionalConfiguration = Map("ehcacheplugin" -> "disabled"),
      			withRoutes = {
        			case ("GET", "/") => Action { Ok("ok") }
      			}
    		)

  		"The OneServerPerTest trait" must {
    			"test server logic" in {
      				val myPublicAddress =  s"localhost:$port"
      				val testPaymentGatewayURL = s"http://$myPublicAddress"
					//测试支付网关在返回一个结果之前需要针对这个服务器的一个回调...
      				val callbackURL = s"http://$myPublicAddress/callback"
      				// await is from play.api.test.FutureAwaits
      				val response = await(WS.url(testPaymentGatewayURL).withQueryString("callbackURL" -> callbackURL).get())

      				response.status mustBe (OK)
    				}
  				}
			}


OneServerPerSuite和OneServerPerTest特性为该服务器运行为port域提供了port号。其默认是19001，甚至你可以通过重写端口或设定系统属性testserver.port来更改。其将非常有用当与持续集成服务器集成在一起，因此端口可以被动态的为每个构建所保留。

你同时可以如之前例子所展示的那样通过重写app来自定义FakeApplication.

最后，如果允许多个测试类来共享这个相同的服务器将会给你比OneServerPerSuite或OneServerPerTest方式更好的性能，正如[ConfiguredServer文档](http://doc.scalatest.org/plus-play/1.0.0/index.html#org.scalatestplus.play.ConfiguredServer "ConfiguredServer文档")中的例子所展示的那样，你可以定义一个主测试集其与OneServePerSuite混用和嵌入式测试集其与ConfiguredServer混用。

## 和一个网页浏览器一起测试 ##

ScalaTest + Play库构基于[Selenium DSL](http://doc.scalatest.org/2.1.5/index.html#org.scalatest.selenium.WebBrowser "Selenium DSL")构建以使其简单的从网页浏览器中测试你的Play应用。
混合OneBrowserPerSuite到你的测试类中以便使用一个相同的浏览器来运行你的测试类中所有的测试。你将同时需要与一个BrowserFactory特性其将提供一个Selenium web驱动混用：ChromeFactory, FirefoxFactory, HtmlUnitFactory, InternetExplorerFactory, SafariFactory之一。

更进一步为了混合入一个BrowserFactory，你将需要混合进一个ServerProvider特性其提供了一个TestServer: ChromeFactory, FirefoxFactory, HtmlUnitFactory, InternetExplorerFactory, SafariFactory之一。

比如，以下的测试类混合进OneServerPerSuite和HtmUnitFactory:

	class ExampleSpec extends PlaySpec with OneServerPerSuite with OneBrowserPerSuite with HtmlUnitFactory {
		//重写app如果你需要一个不仅有默认参数的FakeApplication.
  	implicit override lazy val app: FakeApplication =
    	FakeApplication(
      		additionalConfiguration = Map("ehcacheplugin" -> "disabled"),
      		withRoutes = {
        		case ("GET", "/testing") =>
          		Action(
           		 Results.Ok(
              		"<html>" +
                	"<head><title>Test Page</title></head>" +
                	"<body>" +
                	"<input type='button' name='b' value='Click Me' onclick='document.title=\"scalatest\"' />" +
                	"</body>" +
                	"</html>"
            		).as("text/html")
          		)
      		}
    	)

  	"The OneBrowserPerTest trait" must {
    	"provide a web driver" in {
      		go to (s"http://localhost:$port/testing")
      		pageTitle mustBe "Test Page"
      		click on find(name("b")).value
      		eventually { pageTitle mustBe "scalatest" }
    		}
  		}
	}

如果你的每个测试需要一个新的浏览器实例，请使用OneBrowserPerTest来替代。比如OneBrowserPerSuite,你将需要同时与一个ServerProvider和BrowserFactory来混用:

	class ExampleSpec extends PlaySpec with OneServerPerTest with OneBrowserPerTest with HtmlUnitFactory {
		//重写newAppForTest如果你需要一个不仅有默认参数的FakeApplication.
  		override def newAppForTest(testData: TestData): FakeApplication =
    		new FakeApplication(
      			additionalConfiguration = Map("ehcacheplugin" -> "disabled"),
      			withRoutes = {
        			case ("GET", "/testing") =>
          				Action(
            				Results.Ok(
              					"<html>" +
                				"<head><title>Test Page</title></head>" +
                				"<body>" +
                				"<input type='button' name='b' value='Click Me' onclick='document.title=\"scalatest\"' />" +
                				"</body>" +
                				"</html>"
            				).as("text/html")
          				)
      				}
    			)

  	"The OneBrowserPerTest trait" must {
    	"provide a web driver" in {
      		go to (s"http://localhost:$port/testing")
      		pageTitle mustBe "Test Page"
      		click on find(name("b")).value
      		eventually { pageTitle mustBe "scalatest" }
    		}
  		}
	}



如果你需要多个测试类来共享这个相同的浏览器实例，混合OneBrowserPerSuite进一个主测试集和混合ConfiguredBrowser进多个内嵌的测试集。内嵌的测试集将一起共享相同的浏览器。请参考[ConfiguredBrowser特性的文档](http://doc.scalatest.org/plus-play/1.0.0/index.html#org.scalatestplus.play.ConfiguredBrowser "ConfiguredBrowser特性的文档")中的例子。


## 在多浏览器中运行相同的测试 ##

如果你想在多浏览器中运行测试，以保证你的应用可以在所有你支持的浏览器中正常的工作，你可以使用AllBrowserPerSuite或AllBrowsersPerTest特性。这些特性都声明了一个IndexedSeq[BrowserInfo]特性和一个抽象的sharedTests方法其拥有一个BrowerInfo.browsers域指定了那个浏览器你想让你的测试在其中运行。默认的是Chrome, Firefox, Internet Explorer, HtmlUnit和Safari. 你可以重写browsers如果默认的不符合你的需求.你在sharedTests方法中嵌入你想在多浏览器中运行的测试，将浏览器的名字置于每个测试名的后面（浏览器名字是可用的从BrowserInfo传入进sharedTests.）这里有一个使用AllBrowsersPerSuite的例子：


	class ExampleSpec extends PlaySpec with OneServerPerSuite with AllBrowsersPerSuite {
		//重写App如果你需要一个不仅有默认参数的FakeApplication.
  		implicit override lazy val app: FakeApplication =
    		FakeApplication(
      			additionalConfiguration = Map("ehcacheplugin" -> "disabled"),
      			withRoutes = {
        			case ("GET", "/testing") =>
          			Action(
            			Results.Ok(
              				"<html>" +
                			"<head><title>Test Page</title></head>" +
                			"<body>" +
                			"<input type='button' name='b' value='Click Me' onclick='document.title=\"scalatest\"' />" +
                			"</body>" +
                			"</html>"
            			).as("text/html")
          			)
      			}
    		)

  		def sharedTests(browser: BrowserInfo) = {
    		"The AllBrowsersPerSuite trait" must {
      			"provide a web driver " + browser.name in {
        			go to (s"http://localhost:$port/testing")
        			pageTitle mustBe "Test Page"
        			click on find(name("b")).value
        			eventually { pageTitle mustBe "scalatest" }
      			}
    		}
  		}
	}
所有被sharedTests所声明的测试将被在browsers域中提到的所有浏览器一起运行，只要它们在客户端系统中是可用的。为任何浏览器其不存在于客户端系统的测试将会自动被取消。注意你需要手动的为测试名称追加browser.name以保障这个集合内的每个测试都有一个独立的名字（ScalaTest需要它）.如果你保留其空白，当你运行你的测试的时候将得到一个重复测试名字的错误。

AllBrowsersPerSuite 将为每种类型的浏览器创建一个单独的实例并为在sharedTests中声明的所有测试中使用。如果你想要每个测试去拥有它自己的，全新品牌的浏览器实例，请使用AllBrowsersPerTest来替代：


	class ExampleSpec extends PlaySpec with OneServerPerSuite with AllBrowsersPerTest {
		//重写app如果你需要一个不仅有默认参数的FakeApplication.
  		implicit override lazy val app: FakeApplication =
    		FakeApplication(
      			additionalConfiguration = Map("ehcacheplugin" -> "disabled"),
      			withRoutes = {
        			case ("GET", "/testing") =>
          			Action(
            			Results.Ok(
              				"<html>" +
                			"<head><title>Test Page</title></head>" +
                			"<body>" +
                			"<input type='button' name='b' value='Click Me' onclick='document.title=\"scalatest\"' />" +
                			"</body>" +
               	 			"</html>"
            			).as("text/html")
          			)
      			}
    		)

  		def sharedTests(browser: BrowserInfo) = {
    			"The AllBrowsersPerTest trait" must {
      			"provide a web driver"  + browser.name in {
        			go to (s"http://localhost:$port/testing")
        			pageTitle mustBe "Test Page"
        			click on find(name("b")).value
        			eventually { pageTitle mustBe 	"scalatest" }
      			}
   	 		}
  		}
	}

尽管AllBrowserPerSuite和AllBrowsersPerTest都会针对不提供的浏览器类型而取消测试，这些测试
仍将在输出中显示为被取消。为了清理这些不必要的输出，你可以如下例子所示来通过重写browsers来排除那些永远不会被支持的网页浏览器：

	class ExampleOverrideBrowsersSpec extends PlaySpec with OneServerPerSuite with AllBrowsersPerSuite {

		  override lazy val browsers =
		    Vector(
		      FirefoxInfo(firefoxProfile),
		      ChromeInfo
		    )
		//重写app如果你需要一个不仅有默认参数的FakeApplication.
 		 implicit override lazy val app: FakeApplication =
    		FakeApplication(
      			additionalConfiguration = Map("ehcacheplugin" -> "disabled"),
      			withRoutes = {
        			case ("GET", "/testing") =>
         			 Action(
            			Results.Ok(
             			 	"<html>" +
               			 	"<head><title>Test Page</title></head>" +
                		 	"<body>" +
                			"<input type='button' name='b' value='Click Me' onclick='document.title=\"scalatest\"' />" +
                			"</body>" +
                			"</html>"
            				).as("text/html")
          				)
      				}
   			 )

  		def sharedTests(browser: BrowserInfo) = {
    		"The AllBrowsersPerSuite trait" must {
     			 "provide a web driver"  + browser.name in {
       				 	go to (s"http://localhost:$port/testing")
        				pageTitle mustBe "Test Page"
        				click on find(name("b")).value
        				eventually { pageTitle mustBe "scalatest" }
      				}
    			}
  			}
		}

之前的测试类仅将尝试和Firefox，chrome一起运行共享的测试（并自动化的取消测试如果一个浏览器是不可用的）。

## PlaySpec ##

PlaySpec为Play测试提供了一个便捷的“超集”ScalaTest基础类，你可以通过扩展
PlaySpec来自动得到WordSpec, MustMatchers, OptionValues, 和 WsScalaTestClient：

	class ExampleSpec extends PlaySpec with OneServerPerSuite with  ScalaFutures with IntegrationPatience {
		//重写app如果你需要一个不仅有默认参数的FakeApplication.
  		implicit override lazy val app: FakeApplication =
    		FakeApplication(
      			additionalConfiguration = Map("ehcacheplugin" -> "disabled"),
      			withRoutes = {
        			case ("GET", "/testing") =>
          			Action(
            			Results.Ok(
              				"<html>" +
                			"<head><title>Test Page</title></head>" +
                			"<body>" +
                			"<input type='button' name='b' value='Click Me' onclick='document.title=\"scalatest\"' />" +
                			"</body>" +
                			"</html>"
            				).as("text/html")
          				)
      				}
    			)	
  		"WsScalaTestClient's" must {

    		"wsUrl works correctly" in {
      			val futureResult = wsUrl("/testing").get
      			val body = futureResult.futureValue.body
      			val expectedBody =
        			"<html>" +
          			"<head><title>Test Page</title></head>" +
          			"<body>" +
          			"<input type='button' name='b' value='Click Me' onclick='document.title=\"scalatest\"' />" +
          			"</body>" +
          			"</html>"
      				assert(body == expectedBody)
    			}

    		"wsCall works correctly" in {
      		val futureResult = wsCall(Call("get", "/testing")).get
      		val body = futureResult.futureValue.body
      		val expectedBody =
        		"<html>" +
          		"<head><title>Test Page</title></head>" +
          		"<body>" +
          		"<input type='button' name='b' value='Click Me' onclick='document.title=\"scalatest\"' />" +
          		"</body>" +
          		"</html>"
      		assert(body == expectedBody)
    		}
  		}
	}

你可以将任何之前提到的特性混合进PlaySpec.
在之前例子中所展示的那些测试类中，测试类中的所有或大多数测试需要相同的样式。这个很常见但不总是这样。如果相同的测试类中不同的测试需要不同的样式，与特性MixedFixtures混用。然后使用这些无参功能: App, Server, Chrome, Firefox, HtmlUnit, InternetExplorer, 或 Safari之一来给予每个测试器需要的样式.
你不可以混合MixedFixtures进PlaySpec。因为MixedFixtures需要一个ScalaTest fixture.Suite但PlaySpec仅仅是一个不同的集合。如果你需要针对混合样式的一个简洁的基础类。通过扩展MixedPlaySpec来替代。这里是一个例子：

	// MixedPlaySpec 已经混合进MixedFixtures
	class ExampleSpec extends MixedPlaySpec {
	// 一些帮助器方法
  		def fakeApp[A](elems: (String, String)*) = FakeApplication(additionalConfiguration = Map(elems:_*),
    		withRoutes = {
      			case ("GET", "/testing") =>
        		Action(
          			Results.Ok(
            			"<html>" +
              			"<head><title>Test Page</title></head>" +
              			"<body>" +
              			"<input type='button' name='b' value='Click Me' onclick='document.title=\"scalatest\"' />" +
              			"</body>" +
              			"</html>"
          			).as("text/html")
        		)
    		})
  		def getConfig(key: String)(implicit app: Application) = app.configuration.getString(key)
	// 如果一个测试进需要一个FakeApplication，使用"new APP"：
  		"The App function" must {
    		"provide a FakeApplication" in new App(fakeApp("ehcacheplugin" -> "disabled")) {
      		app.configuration.getString("ehcacheplugin") mustBe Some("disabled")
    		}
    		"make the FakeApplication available implicitly" in new App(fakeApp("ehcacheplugin" -> "disabled")) {
     			 getConfig("ehcacheplugin") mustBe Some("disabled")
    		}
    		"start the FakeApplication" in new App(fakeApp("ehcacheplugin" -> "disabled")) {
      			Play.maybeApplication mustBe Some(app)
    		}
  		}
	// 如果一个测试需要一个FakeApplicaiton并运行TestServer,请使用"new Server":
  	"The Server function" must {
    	"provide a FakeApplication" in new Server(fakeApp("ehcacheplugin" -> "disabled")) 	{
      		app.configuration.getString("ehcacheplugin") mustBe Some("disabled")
    		}
    	"make the FakeApplication available implicitly" in new Server(fakeApp("ehcacheplugin" -> "disabled")) {
      		getConfig("ehcacheplugin") mustBe Some("disabled")
    	}
    	"start the FakeApplication" in new Server(fakeApp("ehcacheplugin" -> "disabled")) {
      		Play.maybeApplication mustBe Some(app)
    	}
    	import Helpers._
    	"send 404 on a bad request" in new Server {
      		import java.net._
      		val url = new URL("http://localhost:" + port + "/boom")
      		val con = url.openConnection().asInstanceOf[HttpURLConnection]
      		try con.getResponseCode mustBe 404
      		finally con.disconnect()
    	}
  	}

	// 如果一个测试需要一个FakeApplication,运行TestServer，和Selenium HtmlUnit驱动请使用"new HtmlUnit"：
  	"The HtmlUnit function" must {
    	"provide a FakeApplication" in new HtmlUnit(fakeApp("ehcacheplugin" -> "disabled")) {
      	app.configuration.getString("ehcacheplugin") mustBe Some("disabled")
    	}
    	"make the FakeApplication available implicitly" in new HtmlUnit(fakeApp("ehcacheplugin" -> "disabled")) {
      		getConfig("ehcacheplugin") mustBe Some("disabled")
    	}
    	"start the FakeApplication" in new HtmlUnit(fakeApp("ehcacheplugin" -> "disabled")) {
      		Play.maybeApplication mustBe Some(app)
    	}
    	import Helpers._
    	"send 404 on a bad request" in new HtmlUnit {
      		import java.net._
      		val url = new URL("http://localhost:" + port + "/boom")
      		val con = url.openConnection().asInstanceOf[HttpURLConnection]
      		try con.getResponseCode mustBe 404
      		finally con.disconnect()
    	}
    	"provide a web driver" in new HtmlUnit(fakeApp()) {
     		go to ("http://localhost:" + port + "/testing")
      		pageTitle mustBe "Test Page"
      		click on find(name("b")).value
      		eventually { pageTitle mustBe "scalatest" }
    	}
  	}

	// 如果一个测试需要一个FakeApplication,运行TestServer,和Selenium Firefox驱动请使用"new Firefox"：
  	"The Firefox function" must {
    	"provide a FakeApplication" in new Firefox(fakeApp("ehcacheplugin" -> "disabled")) {
      	app.configuration.getString("ehcacheplugin") mustBe Some("disabled")
    	}
    	"make the FakeApplication available implicitly" in new Firefox(fakeApp("ehcacheplugin" -> "disabled")) {
      		getConfig("ehcacheplugin") mustBe Some("disabled")
    	}
    	"start the FakeApplication" in new Firefox(fakeApp("ehcacheplugin" -> "disabled")) {
      		Play.maybeApplication mustBe Some(app)
    	}
    	import Helpers._
    	"send 404 on a bad request" in new Firefox {
      		import java.net._
      		val url = new URL("http://localhost:" + port + "/boom")
      		val con = url.openConnection().asInstanceOf[HttpURLConnection]
      		try con.getResponseCode mustBe 404
      		finally con.disconnect()
    	}
    	"provide a web driver" in new Firefox(fakeApp()) {
      		go to ("http://localhost:" + port + "/testing")
      		pageTitle mustBe "Test Page"
      		click on find(name("b")).value
      		eventually { pageTitle mustBe "scalatest" }
    	}
  	}
	
	// // 如果一个测试需要一个FakeApplication,运行TestServer,和Selenium Safari驱动请使用"new Safari"：
  	"The Safari function" must {
    	"provide a FakeApplication" in new Safari(fakeApp("ehcacheplugin" -> "disabled")) {
      		app.configuration.getString("ehcacheplugin") mustBe Some("disabled")
    	}
    	"make the FakeApplication available implicitly" in new Safari(fakeApp("ehcacheplugin" -> "disabled")) {
      		getConfig("ehcacheplugin") mustBe Some("disabled")
    	}
    	"start the FakeApplication" in new Safari(fakeApp("ehcacheplugin" -> "disabled")) {
      		Play.maybeApplication mustBe Some(app)
    	}
    	import Helpers._
    	"send 404 on a bad request" in new Safari {
      		import java.net._
      		val url = new URL("http://localhost:" + port + "/boom")
      		val con = url.openConnection().asInstanceOf[HttpURLConnection]
      		try con.getResponseCode mustBe 404
      		finally con.disconnect()
    	}
    	"provide a web driver" in new Safari(fakeApp()) {
      		go to ("http://localhost:" + port + "/testing")
      		pageTitle mustBe "Test Page"
      		click on find(name("b")).value
      		eventually { pageTitle mustBe "scalatest" }
    	}
  	}

	// 如果一个测试需要一个FakeApplication,运行TestServer,和Selenium Chrome驱动请使用"new Chrome"：
  	"The Chrome function" must {
    	"provide a FakeApplication" in new Chrome(fakeApp("ehcacheplugin" -> "disabled")) {
      		app.configuration.getString("ehcacheplugin") mustBe Some("disabled")
    	}
    	"make the FakeApplication available implicitly" in new Chrome(fakeApp("ehcacheplugin" -> "disabled")) {
      		getConfig("ehcacheplugin") mustBe Some("disabled")
    	}
    	"start the FakeApplication" in new Chrome(fakeApp("ehcacheplugin" -> "disabled")) {
      		Play.maybeApplication mustBe Some(app)
    	}
    	import Helpers._
    	"send 404 on a bad request" in new Chrome {
      		import java.net._
      		val url = new URL("http://localhost:" + port + "/boom")
      		val con = url.openConnection().asInstanceOf[HttpURLConnection]
      		try con.getResponseCode mustBe 404
      		finally con.disconnect()
    	}
    	"provide a web driver" in new Chrome(fakeApp()) {
      		go to ("http://localhost:" + port + "/testing")
      		pageTitle mustBe "Test Page"
      		click on find(name("b")).value
      		eventually { pageTitle mustBe "scalatest" }
    	}
  	}

	// 如果一个测试需要一个FakeApplication,运行TestServer,和Selenium InternetExplorer驱动请使用"new InternetExplorer"：
  	"The InternetExplorer function" must {
    	"provide a FakeApplication" in new InternetExplorer(fakeApp("ehcacheplugin" -> "disabled")) {
      		app.configuration.getString("ehcacheplugin") mustBe Some("disabled")
    	}
    	"make the FakeApplication available implicitly" in new InternetExplorer(fakeApp("ehcacheplugin" -> "disabled")) {
      		getConfig("ehcacheplugin") mustBe Some("disabled")
    	}
    	"start the FakeApplication" in new InternetExplorer(fakeApp("ehcacheplugin" -> "disabled")) {
     	 Play.maybeApplication mustBe Some(app)
    	}
    	import Helpers._
    	"send 404 on a bad request" in new InternetExplorer {
      		import java.net._
      		val url = new URL("http://localhost:" + port + "/boom")
      		val con = url.openConnection().asInstanceOf[HttpURLConnection]
      		try con.getResponseCode mustBe 404
      		finally con.disconnect()
    	}
    	"provide a web driver" in new InternetExplorer(fakeApp()) {
     		 go to ("http://localhost:" + port + "/testing")
      		pageTitle mustBe "Test Page"
      		click on find(name("b")).value
      		eventually { pageTitle mustBe "scalatest" }
    	}
  	}

	// 如果一个测试不需要任何的特殊样式，仅需要写成"in { () => ... "
  	"Any old thing" must {
    	"be doable without much boilerplate" in { () =>
       	1 + 1 mustEqual 2
     	}
  	}
	}

## 测试一个模板 ##
因为一个模板是一个标准的Scala功能，你可以从你的测试中执行它，并检测结果：


	"render index template" in new App {
  		val html = views.html.index("Coco")

  		contentAsString(html) must include ("Hello Coco")
	}

## 测试一个控制器 ##

You can call any Action code by providing a FakeRequest:
你可以通过提供一个FakRequest来调用任何Action代码：

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

技术上讲，在这里你不需要WithApption,尽管使用它并不会产生任何不利影响。

测试一个路由

你可以通过让Router来做而不是你自己来调用Action：

	"respond to the index Action" in new App(fakeApplication) {
  		val Some(result) = route(FakeRequest(GET, "/Bob"))

  		status(result) mustEqual OK
  		contentType(result) mustEqual Some("text/html")
  		charset(result) mustEqual Some("utf-8")
  		contentAsString(result) must include ("Hello Bob")
	}

## 测试一个模块 ##

如果你使用一个SQL数据库，你可以通过使用inMemoryDatabase来用一个H2数据库的内存实例替换数据库连接。

	val appWithMemoryDatabase = FakeApplication(additionalConfiguration = inMemoryDatabase("test"))
	"run an application" in new App(appWithMemoryDatabase) {

  		val Some(macintosh) = Computer.findById(21)

  		macintosh.name mustEqual "Macintosh"
  		macintosh.introduced.value mustEqual "1984-01-24"
	}


## 测试WS调用 ##

如果你正在调用一个网页服务，你可以使用WSTestClinet.这里两个调用可用，wsCall和wsUrl其将分别执行一个调用或字符串。注意他们期望在WithApplication的上下文里被调用。

	wsCall(controllers.routes.Application.index()).get()

	wsUrl("http://localhost:9000").get()