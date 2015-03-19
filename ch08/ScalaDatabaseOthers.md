# Integrating with other database access libraries

你可以通过Play框架使用任何你喜欢的SQL数据库访问模块，可以简单的从play.api.db.DB 帮助器获取一个连接或数据源.

## 与ScalaQuery集成 ##

至此你可以集成任何JDBC访问层其需要一个JDBC数据源。比如，与ScalaQuery集成：

	import play.api.db._
	import play.api.Play.current

	import org.scalaquery.ql._
	import org.scalaquery.ql.TypeMapper._
	import org.scalaquery.ql.extended.{ExtendedTable => Table}

	import org.scalaquery.ql.extended.H2Driver.Implicit._ 

	import org.scalaquery.session._

	object Task extends Table[(Long, String, Date, Boolean)]("tasks") {
    
  		lazy val database = Database.forDataSource(DB.getDataSource())
  
  		def id = column[Long]("id", O PrimaryKey, O AutoInc)
  		def name = column[String]("name", O NotNull)
  		def dueDate = column[Date]("due_date")
  		def done = column[Boolean]("done")
  		def * = id ~ name ~ dueDate ~ done
  
  		def findAll = database.withSession { implicit db:Session =>
      		(for(t <- this) yield t.id ~ t.name).list
  		}
  
	}


## 通过JNDI展示数据源 ##

有些库期望通过JNDI获取数据源引用。你可以在conf/application.conf中添加以下配置来通过JNDI展示任何Play框架管理的数据源：


	db.default.driver=org.h2.Driver
	db.default.url="jdbc:h2:mem:play"
	db.default.jndiName=DefaultDS