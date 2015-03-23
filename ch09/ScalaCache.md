# The Play cache API
## Play 缓存 API ##
使用[EHCache](http://ehcache.org/ "EHCache")为缓存API的默认实现.你同时可以通过一个插件来提供你自己的实现。

## 载入缓存API ##

Add cache into your dependencies list. For example, in build.sbt:
将cache加入到你的关联列表中。例如，在build.sbt中：

	libraryDependencies ++= Seq(
  		cache,
  		...
		)

## 访问缓存API ##

play.api.cache.Cache对象提供了缓存API.其需要一个注册的缓存插件。


    
	注意： 该API有意的最小化允许多种实现可被嵌入。如果你需要一个更具体的API，使用你的缓存插件提供的那一个。

使用这个简单的API你既可以存储数据于缓存中：

	Cache.set("item.key", connectedUser)

并在不久后获取它：

	val maybeUser: Option[User] = Cache.getAs[User]("item.key")

当其不存在时候这里还有一个简洁的帮助器去从缓存中获取或在缓存中设定值：

	val user: User = Cache.getOrElse[User]("item.key") {
  		User.findById(connectedUser)
	}

To remove an item from the cache use the remove method:
使用remove方法去从缓存中移除一个条目：

	Cache.remove("item.key")

## 缓存HTTP回应 ##

你可以使用标准的Action组合。

	注意： Play HTTP 结果实例对缓存是安全的并可以之后从用。

Play为默认的例子提供了一个默认的嵌入帮助器：

	def index = Cached("homePage") {
		  Action {
    		Ok("Hello world")
  			}
	}

甚至于：

	def userProfile = Authenticated {
  		user =>
    	Cached(req => "profile." + user) {
      		Action {
        		Ok(views.html.profile(User.find(user)))
      				}
    		}
	}

## 缓存控制 ##

你可以简单的控制什么是你想缓存的或者什么是你不想缓存的。

你可能仅需要结果为“缓存200 OK”

	def get(index: Int) = Cached.status(_ => "/resource/"+ index, 200) {
  		Action {
    	if (index > 0) {
      		Ok(Json.obj("id" -> index))
    		} else {
      			NotFound
    		}
  		}
		}


或者“缓存404未找到”的信息仅存在几分钟

	def get(index: Int) = {
  		val caching = Cached
    		.status(_ => "/resource/"+ index, 200)
    		.includeStatus(404, 600)

  		caching {
    		Action {
      		if (index % 2 == 1) {
        		Ok(Json.obj("id" -> index))
      		} else {
        		NotFound
      				}
    		}
 		}
	}