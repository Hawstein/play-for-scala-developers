# Accessing an SQL database

## 配置 JdbC连接池 ##

Play框架提供了一个用于管理JDBC连接池的插件.你可以像配置任何数据库一样针对你的需求对它进行配置。

为了激活数据库插件，请将jdbc加入到你的构建关联关系中：

libraryDependencies += jdbc

接着你必须在 conf/application.conf 文件中配置一个连接池。为了使用便利，默认的JDBC数据源必须调用并且相关联的配置属性应该为db.default.driver和db.default.url.


如果配置不正确或有所遗漏在你的浏览器中你将会直接得到如下的通知：

![dberror]( https://www.playframework.com/documentation/2.3.x/resources/manual/scalaGuide/main/sql/images/dbError.png )

    注意: 你最好用双引号包住JDBC URL的配置值，因为':'在配置语法中为反转字符。

## H2数据库引擎配置属性 ##
#使用H2数据库引擎”基于内存“模式作为默认的数据库配置 
db.default.driver=org.h2.Driver
db.default.url="jdbc:h2:mem:play"

# 使用H2数据库引擎”持久化“模式作为默认的数据库配置
db.default.driver=org.h2.Driver
db.default.url="jdbc:h2:/path/to/db-file"

关于H2数据库更详细的信息请参考 [http://www.h2database.com/html/cheatSheet.html](http://www.h2database.com/html/cheatSheet.html "H2数据库引擎图表") 

SQLite 数据库引擎连接属性

# 使用SQLite数据库引擎最为默认的数据库配置
db.default.driver=org.sqlite.JDBC
db.default.url="jdbc:sqlite:/path/to/db-file"

PostgreSQL数据库引擎连接属性

# 使用PostgreSQL数据库引擎作为默认的数据库配置
db.default.driver=org.postgresql.Driver
db.default.url="jdbc:postgresql://database.example.com/playdb"


MySQL 数据库引擎连接属性

#使用MySQL数据库引擎作为默认的数据库配置 
# 以playdbuser为用户连接playdb
db.default.driver=com.mysql.jdbc.Driver
db.default.url="jdbc:mysql://localhost/playdb"
db.default.user=playdbuser
db.default.password="a strong password"

如何在命令窗口中查看SQL表达式？

db.default.logStatements=true

logger.com.jolbox=DEBUG // for EBean

如何配置多个数据源？

#订购者的数据库配置 
db.orders.driver=org.h2.Driver
db.orders.url="jdbc:h2:mem:orders"

# 消费者的数据库配置
db.customers.driver=org.h2.Driver
db.customers.url="jdbc:h2:mem:customers"

配置JDBC驱动

Play框架仅可以和h2数据库驱动进行捆绑。因此，为了在应用环境中使用你必须将你的数据库驱动添加为依赖包。
例如，如果你使用MySQL5数据库，你需要为该连接器添加如下的相关依赖：
libraryDependencies += "mysql" % "mysql-connector-java" % "5.1.27"

或者如果该驱动无法从软件仓库中找到，你可以将其放到你项目中的非管理的关联库目录下。

访问JDBC数据源

play.api.db软件包提供了访问配置数据源的方法：

import play.api.db._

val ds = DB.getDataSource()

获取一个JDBC连接
这里有几种方式可以获取一个JDBC连接。最简单的方式为：

val connection = DB.getConnection()

以下的代码向你展示了一个简单的JDBC如何和MySQL5.*一起使用的例子：

package controllers
import play.api.Play.current
import play.api.mvc._
import play.api.db._

object Application extends Controller {

  def index = Action {
    var outString = "Number is "
    val conn = DB.getConnection()
    try {
      val stmt = conn.createStatement
      val rs = stmt.executeQuery("SELECT 9 as testkey ")
      while (rs.next()) {
        outString += rs.getString("testkey")
      }
    } finally {
      conn.close()
    }
    Ok(outString)
  }

}


但是你必须调用close()在某一点上的打开的连接返回到连接池。另一种方式是利用Play框架来关闭该连接：

// 访问“默认”数据库

DB.withConnection { conn =>

  // 根据你的需求配置连接
}

非默认数据库配置：

// 访问“订阅者”数据库而不是“默认”数据库

DB.withConnection("orders") { conn =>

  // 根据你的需求配置连接
}

在最后的代码段中连接会自动关闭。

    建议: 每个与此连接相关联的表达式和结果集也会同时关闭

变量是用来设定连接的自动提交为非法的并且对该代码段进行事务管理

DB.withTransaction { conn =>

  // 根据你的需求配置连接
}