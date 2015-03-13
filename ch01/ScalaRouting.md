# HTTP routing

## 内建HTTP路由
路由是一个负责将传入的HTTP请求转换成Action的组件。

一个HTTP请求通常被MVC框架视为一个event。这个event包含了两个主要信息：

* 请求路径（例如： `/clients/1542`，`/photos/list`），其中包括了查询语句
* HTTP方法（例如：GET，POST等）

路由定义在了`conf/routes`文件中，该文件会被编译。也就是说你可以直接在你的浏览器中看到这样的路由错误：

![Route Error in Browser](https://www.playframework.com/documentation/2.3.x/resources/manual/scalaGuide/main/http/images/routesError.png)
