# Using Anorm to access your database
# **Anorm, 简单的SQL数据访问** #


Play框架包含了一个简单的数据访问层Anorm 其使用原生的SQL语言与数据库交互并且提供一个API去解析和传输结果数据集。

**Anorm 不是一个对象关联的映射器**

    
在以下的文档中，我们会使用[mysql world simple database](http://dev.mysql.com/doc/index-other.html "MySQL world simple database")
如果你想在你的应用中使用它，请依据MySQL网站的指导说明并且根据 [scala数据库文档页面](https://www.playframework.com/documentation/2.3.x/ScalaDatabase "scala 数据库文档页面")对其进行配置

## **总览** ##

现在如果使用原生的，古老的SQL语言访问一个SQL数据库会让人感到奇怪，尤其是对于java开发者定制化的使用一个高等级的关联器比如Hibernate去完全隐藏这一部分实现（SQL数据库访问）。

尽管我们同意这个看法，java开发中需要这类工具，但我们认为其并非必要当你拥有高级语言例如scala的能力的时候。反之，其会很快的得到相反的效果。

**使用JDBC是痛苦的，但我们可以提供一个更好的API**

我们同意直接使用JDBC API是令人生厌的，尤其在java语言中。你不得不在所有地方处理例外检测并且反复的围绕着结果集将所有的原始数据集传入进你自有的数据结构中去。

我们为JDBC提供了一个简单的API;使用scala你不需要被例外所烦扰，并且使用功能性语言传输数据异常简单。事实上，Play框架scala SQL访问层的实现目的就是为了提供几种APIs以使从JDBC数据传输到其他scala结构中去变得更为方便。

**你不需要其他DSL（定制化语言）去访问关系型数据库****


SQL已经是最好的DSL用来访问关系型数据库。我们不需要发明新的替代品。而且SQL语法和特性可以根据数据库厂商的不同而不同。

如果你尝试去从其他的私有的SQL比如DSL中剽窃点子你将不得不为每个厂商使用不同的‘方言’（比如Hibernate 框架），并且由于不使用某个数据库的特性功能而限制你自己的使用。

Play框架必要时会提供你预注射的SQL语句，但是这个主意并未掩盖其事实在底层我们仍然使用SQL语句。Play框架仅仅保证了在简单的查询中间少了打字，并且你将永远退回到原始的SQL。

**使用类型安全的DSL去生成SQL是一个错误**


某些辩解声称类型安全的DSL会得到更好的效果由于所有你的查询会被编译器检测。不幸的是编译器是基于原始模块定义所以你需要采用映射你的数据结构到数据库模式来经常自己编写。

这里并不保证这个原始模块是正确的，即使编译器声明你的代码和查询输入是正确的。其仍可能不知何故的运行失败因为一个在你实际的数据库定义中的错误匹配。

**控制你的SQL代码**

在小的示例中对象关联映射会工作的很好， 但是当你不得不处理复杂模式或者已经存在的数据库时，你将消耗大部分的时间与你的ORM做斗争以便保证你得到像样的SQL查询。

为简单的'Hello World'应用自己编写一个SQL查询会异常简单，但任何的真实应用，你针对你的SQL代码拥有完全的控制将最终会节省时间和简化你的代码。

## **添加Anorm到你的项目中** ##
你将需要添加 Anorm和JDBC到你的关联依赖中：


    libraryDependencies ++= Seq(
  		jdbc,
  		anorm
	)

## 执行SQL查询 ##

一开始你需要学习如何执行SQL查询，
首先， import anorm._,接着简单的使用SQL对象去建立查询。你需要一个数据库连接去运行一个查询，你可以从play.db.DB中获取得到。

	import anorm._ 
	import play.api.db.DB

	DB.withConnection { implicit c =>
 		 val result: Boolean = SQL("Select 1").execute()    
	} 


execute() 方法 会返回一个布尔值表明这个执行是否成功。
执行一个更新，你可以使用executeUpdate(),其将返回多少行被更新了。


	val result: Int = SQL("delete from City where id = 99").executeUpdate()

如果你输入数据其有一个自动生成的long主键，你可以调用executeInsert().

	val id: Option[Long] = 
  		SQL("insert into City(name, country) values ({name}, {country})")
  		.on('name -> "Cambridge", 'country -> "New Zealand").executeInsert()

当基于插入生成的键不是一个单一long, executeInsert 将被传入一个ResultSetParser返回正确的键。

	import anorm.SqlParser.str

	val id: List[String] = 
  	SQL("insert into City(name, country) values ({name}, {country})")
  	.on('name -> "Cambridge", 'country -> "New Zealand")
  	.executeInsert(str.+) // insertion returns a list of at least one string keys

由于scala支持多行字符串，可以针对复杂的SQL语句使用。

	val sqlQuery = SQL(
  		"""
    	select * from Country c 
    	join CountryLanguage l on l.CountryCode = c.Code 
    	where c.code = 'FRA';
  		"""
	)

如果你的SQL查询需要动态参数，你可以在此查询字段中声明大括号类似于{name},接着针对他们设置值：

	SQL（
  	"""
    select * from Country c 
    join CountryLanguage l on l.CountryCode = c.Code 
    where c.code = {countryCode};
  	"""
	).on("countryCode" -> "FRA")


你可以使用字符串插补来传递参数（请看以下的细节部分）

在某些实例中，查询结果中发现相同的名字存在于多个列，例如列名code同时存在于Country和CountryLanguage表中，其会变得含糊不清。

	import anorm.{ SQL, SqlParser }

	val code: String = SQL(
  		"""
    	select * from Country c 
    	join CountryLanguage l on l.CountryCode = c.Code 
    	where c.code = {countryCode}
  		""")
  		.on("countryCode" -> "FRA").as(SqlParser.str("code").single)

如果Country.Code是‘firest’和CountryLanguage是‘second’，在之前的code值中其会变为‘second’。使用适当的列名将会解决模糊不清的问题，表名如：

	import anorm.{ SQL, SqlParser }

	val code: String = SQL(
  		"""
    	select * from Country c 
    	join CountryLanguage l on l.CountryCode = c.Code 
    	where c.code = {countryCode}
  		""")
  		.on("countryCode" -> "FRA").as(SqlParser.str("Country.code").single)
		// code == "First"
列同时可以被位置而不是名字所定义：

	// Parsing column by name or position
	val parser = 
  	SqlParser(str("name") ~ float(3) /* third column as float */ map {
    case name ~ f => (name -> f)
  	}

	val product: (String, Float) = SQL("SELECT * FROM prod WHERE id = {id}").
  	on('id -> "p").as(parser.single)

java.util.UUID 可以被用作参数，在此例子中它的字符串值将转换为语法。

**# 使用字符串插入进行SQL查询 #**

从scala 2.10起支持定制字符串插入同时这里有一个单步替代SQL(queryString).on(params)如同之前所看到的。你可以缩短代码如下：

	val name = "Cambridge"
	val country = "New Zealand"

	SQL"insert into City(name, country) values ($name, $country)"

其同时支持多行字符串和内联表达式：

	val lang = "French"
	val population = 10000000
	val margin = 500000

	val code: String = SQL"""
  		select * from Country c 
    	join CountryLanguage l on l.CountryCode = c.Code 
    	where l.Language = $lang and c.Population >= ${population - margin}
    	order by c.Population desc limit 1"""
  	.as(SqlParser.str("Country.code").single)

该功能尝试更快，更正确和简单的通过Anorm获取数据。所以请自由的使用该功能任何时候你看到一个合并的SQL().on()功能（或者仅仅不带参数的SQL()).

## 使用Stream API获取数据 ##

第一种方式是使用Stream API 来访问一个选择查询的结果。
当你在任何SQL语句中调用apply()，你将收到一个Row实例的惰性的Stream，
当每行会被看作一个字典：

	// Create an SQL query
	val selectCountries = SQL("Select * from Country")
 
	// Transform the resulting Stream[Row] to a List[(String,String)]
	val countries = selectCountries().map(row => 
  		row[String]("code") -> row[String]("name")
	).toList


以下的示例我们将计算数据库中Country条目的数量，所以结果集会为一个单表和单行：

	// First retrieve the first row
	val firstRow = SQL("Select count(*) as c from Country").apply().head
 
	// Next get the content of the 'c' column as Long
	val countryCount = firstRow[Long]("c")


## 多值支持 ##

Anorm参数可以为多值，比如一系列字符串。
在此例中，值将被准备传输到JDBC。

	// With default formatting (", " as separator)
	SQL("SELECT * FROM Test WHERE cat IN ({categories})").
  	on('categories -> Seq("a", "b", "c")
	// -> SELECT * FROM Test WHERE cat IN ('a', 'b', 'c')

	// With custom formatting
	import anorm.SeqParameter
	SQL("SELECT * FROM Test t WHERE {categories}").
  		on('categories -> SeqParameter(
    	values = Seq("a", "b", "c"), separator = " OR ", 
    	pre = "EXISTS (SELECT NULL FROM j WHERE t.id=j.id AND name=",
    post = ")"))
	/* ->
	SELECT * FROM Test t WHERE 
	EXISTS (SELECT NULL FROM j WHERE t.id=j.id AND name='a') 
	OR EXISTS (SELECT NULL FROM j WHERE t.id=j.id AND name='b') 
	OR EXISTS (SELECT NULL FROM j WHERE t.id=j.id AND name='c')
	*/

一列同时可以为多值如果其类型为JDBC 数组(java.sql.Array),然后其可以被映射到数组或列表(Array[T]或 List[T]),提供的元素类型(T)同时在列映射中支持。

	import anorm.SQL
	import anorm.SqlParser.{ scalar, * }

	// array and element parser
	import anorm.Column.{ columnToArray, stringToArray }

	val res: List[Array[String]] =
  		SQL("SELECT str_arr FROM tbl").as(scalar[Array[String]].*)

## 批量更新 ##

当你需要多次执行不同参数的SQL语句，批量请求可以被使用（例如.执行批量的插入）.

	import anorm.BatchSql

	val batch = BatchSql(
  		"INSERT INTO books(title, author) VALUES({title}, {author}", 
  		Seq(Seq[NamedParameter](
    		"title" -> "Play 2 for Scala", "author" -> Peter Hilton"),
    		Seq[NamedParameter]("title" -> "Learning Play! Framework 2",
     		 "author" -> "Andy Petrella")))

	val res: Array[Int] = batch.execute() // array of update count

## 边缘例子 ##

传递任何差异从字符串或符号作为的参数名现在已经失效了。为了向后兼容，你可以激活 anorm.features.parameterWithUntypedName.

	import anorm.features.parameterWithUntypedName // activate

	val untyped: Any = "name" // deprecated
	SQL("SELECT * FROM Country WHERE {p}").on(untyped -> "val")

参数值的类型必须为可见的，并合理的在SQL语法中设定。
使用值比如Any,明确的或由于消除，导致编译错误"No implicit view available from Any => anorm.ParameterValue".


	// 错误的 #1
	val p: Any = "strAsAny"
	SQL("SELECT * FROM test WHERE id={id}").
  	on('id -> p) // Erroneous - No conversion Any => ParameterValue

	// 正确的 #1
	val p = "strAsString"
	SQL("SELECT * FROM test WHERE id={id}").on('id -> p)

	// 错误的 #2
	val ps = Seq("a", "b", 3) // inferred as Seq[Any]
	SQL("SELECT * FROM test WHERE (a={a} AND b={b}) OR c={c}").
  		on('a -> ps(0), // ps(0) - No conversion Any => ParameterValue
    	'b -> ps(1), 
    	'c -> ps(2))

	// 正确的 #2
	val ps = Seq[anorm.ParameterValue]("a", "b", 3) // Seq[ParameterValue]
	SQL("SELECT * FROM test WHERE (a={a} AND b={b}) OR c={c}").
 		 on('a -> ps(0), 'b -> ps(1), 'c -> ps(2))

	// 错误的 #3
	val ts = Seq( // Seq[(String -> Any)] due to _2
  		"a" -> "1", "b" -> "2", "c" -> 3)

	val nps: Seq[NamedParameter] = ts map { t => 
  	val p: NamedParameter = t; p
  	// Erroneous - no conversion (String,Any) => NamedParameter
	}

	SQL("SELECT * FROM test WHERE (a={a} AND b={b}) OR c={c}").on(nps :_*) 

	// 正确的 #3
	val nps = Seq[NamedParameter]( // Tuples as NamedParameter before Any
  		"a" -> "1", "b" -> "2", "c" -> 3)
	SQL("SELECT * FROM test WHERE (a={a} AND b={b}) OR c={c}").
  		on(nps: _*) // Fail - no conversion (String,Any) => NamedParameter

为了向后兼容，你可以激活如下的不安全的参数转换,接受非类型Any值和

	anorm.features.anyToStatement.
	import anorm.features.anyToStatement

	val d = new java.util.Date()
	val params: Seq[NamedParameter] = Seq("mod" -> d, "id" -> "idv")
	// Values as Any as heterogenous

	SQL("UPDATE item SET last_modified = {mod} WHERE id = {id}").on(params:_*)


不建议这样做因为会产生更多的隐性转译问题，比如非类型当使用setObject在语句中传值时会导致运行时转换错误。
在之前的例子中，java.util.Date接收为参数但是会伴随更多的数据库错误（由于其并不是验证过的JDBC类型）。


## 使用模式匹配 ##


你同时可以使用模式匹配来匹配和抽取行内容。在该例子中列名不重要。仅仅这种类型和序列的参数会被用于匹配。

如下的示例转换每个行为正确的scala类型：

	case class SmallCountry(name:String) 
	case class BigCountry(name:String) 
	case class France
 
	val countries = SQL("Select name,population from Country")().collect {
  		case Row("France", _) => France()
  		case Row(name:String, pop:Int) if(pop > 1000000) => BigCountry(name)
  		case Row(name:String, _) => SmallCountry(name)      
	}
注意当部分功能为定义时collect(...)会忽略了这些case，它允许你的代码安全的忽略你不需要的行。

## 使用for-comprehension ##

与SQL结果类型一起使用，行解析器可以被定义为for-comprehension。其可以非常有用当与大量的列一起使用，可能绕过case类的限制。

	import anorm.SqlParser.{ str, int }

	val parser = for {
  		a <- str("colA")
  		b <- int("colB")
	} yield (a -> b)

	val parsed: (String, Int) = SELECT("SELECT * FROM Test").as(parser.single)

## 获取数据连同执行上下文信息 ##


此外数据，查询执行包含了上下文信息比如SQL警告会增长（或许非常严重或者未必），尤其是当与SQL存储过程一起使用。

方式去获取查询数据连同上下文信息是使用executeQuery():


	import anorm.SqlQueryResult

	val res: SqlQueryResult = SQL("EXEC stored_proc {code}").
  		on('code -> code).executeQuery()

	// Check execution context (there warnings) before going on
	val str: Option[String] =
  		res.statementWarning match {
    		case Some(warning) =>
      			warning.printStackTrace()
      			None

    		case _ => res.as(scalar[String].singleOpt) // go on row parsing
  	}

## 特殊数据类型 ##

## Clobs ##

CLOBs/TEXTs 可以被抽取如:

	SQL("Select name,summary from Country")().map {
  		case Row(name: String, summary: java.sql.Clob) => name -> summary
	}

此时我们特别的选择使用map,当我们想要一个例外返回当行不是我们期望的格式。

## Binary ##

抽取二进制数据是可能非常简单的：

	SQL("Select name,image from Country")().map {
  		case Row(name: String, image: Array[Byte]) => name -> image
	}

## 数据库的互操作性 ##

注意不同的数据库会基于行返回不同的数据类型。比如，用一个SQL ‘smallint’会返回如一个Short当使用org.h2.Driver和一个整数当使用org.postgresql.Driver。一个方案是这个会简化写分离例子语句为每个数据库(比如.一个为开发环境和一个为产品环境)。

Anorm为scala类型提供了通用的映射从JDBC数据类型.
当需要时，它能够定制这种映射关系，例如当底层的数据库不支持布尔数据类型而返回整数代替。为了能够实现，你不得不为Column[T]提供一个新的隐性的转换，当T为目标scala类型：

	import anorm.Column

	// Custom conversion from JDBC column to Boolean
	implicit def columnToBoolean: Column[Boolean] = 
  		Column.nonNull { (value, meta) =>
    		val MetaDataItem(qualified, nullable, clazz) = meta
    		value match {
     		 case bool: Boolean => Right(bool) // Provided-default case
     		 case bit: Int      => Right(bit == 1) // Custom conversion
     		 case _             => Left(TypeDoesNotMatch(s"Cannot convert $value: ${value.asInstanceOf[AnyRef].getClass} to Boolean for column $qualified"))
    		}
  		}

为参数做的定制或指定DB转换可以被同时提供：

	import java.sql.PreparedStatement
	import anorm.ToStatement

	// Custom conversion to statement for type T
	implicit def customToStatement: ToStatement[T] = new ToStatement[T] {
  		def set(statement: PreparedStatement, i: Int, value: T): Unit =
   			 ??? // Sets |value| on |statement|
	}

如果引入类型接受null值，其必须在转换中合理的被操作。即使被类型所接受，当null必须被参数转换所拒绝，标志性状态NotNullGuard会被使用：new ToStatement[T] with NotNullGuard { /* ... */ }.

针对数据库特定参数，其可以作为模糊值被显性的传递。
如果你能承受风险，setObject将可以在语句中使用。

	val anyVal: Any = myVal
	SQL("UPDATE t SET v = {opaque}").on('opaque -> anorm.Object(anyVal))

## 处理Nullable列 ##
如果在数据库模式中一列可以包含Null值，你需要像操作Option类型一样操作它。

例如，Country表中的indepYear可以是空值，所以你需要像Option[Int]一样匹配它：

	SQL("Select name,indepYear from Country")().collect {
 	 case Row(name:String, Some(year:Int)) => name -> year
	}

If you try to match this column as Int it won’t be able to parse Null values. Suppose you try to retrieve the column content as Int directly from the dictionary:
如果你尝试匹配该列比如Int，其不能够解析Null值。假设你尝试从字典中直接抽取列内容比如Int:

	SQL("Select name,indepYear from Country")().map { row =>
  		row[String]("name") -> row[Int]("indepYear")
	}

这会产生一个UnexpectedNullableFound(COUNTRY.INDEPYEAR)例外如果其碰到一个null值，所以你需要将其映射到一个Option[Int],比如：

	SQL("Select name,indepYear from Country")().map { row =>
	  row[String]("name") -> row[Option[Int]]("indepYear")
	}


这同时也对解析器API适用，我们将在下节中看到。

## 使用解析器API ##

你可以是使用解析器API来生成一个通用的和可重用的解析器其可以解析任何选择查询的结果.

	注意： 这种方法非常有用，因为一个web程序中的大多数请求会返回相同的数据集。比如，如果你定义了一个解析器其能够从一个结果集中解析一个Country,和另一个Language解析器，你可以简单的封装它们从一个联合查询中解析Country和Language.

    首先你需要 import anorm.SqlParser._


## 获取一个单一结果 ##

首先你需要一个RowParser,比如，一个解析器能够解析一行为一个scala值。比如我们可以定义一个解析器去转换一个单列结果行为一个scala Long:

	val rowParser = scalar[Long]

接着我们需要将其转换到一个ResultSetParser.这里我们会创建一个解析器来解析一个单行：

	val rsParser = scalar[Long].single

所以该解析器能够解析一个结果集以返回一个Long.其非常有用当解析为一个简单的SQL搜索计算查询产生的结果：
	
	val count: Long = 
  		SQL("select count(*) from Country").as(scalar[Long].single)


## 获取一个简单的可选结果 ##


你需要从国家名字中获取到conutry_id，但是该查询可能会返回null.我们需要使用 singleOpt解析器来实现：

	val countryId: Option[Long] = 
  		SQL("select country_id from Country C where C.country='France'")
  		.as(scalar[Long].singleOpt)

## 获得一个更加复杂的结果 ##

让我们写一个更复杂的解析器：

str("name") ~ int("population"), 会创建一个 RowParser 以能够解析一行包含一个字符类型name列和一个整数类型population列。接着我们可以新建一个ResultSetParser 其将使用*尽可能的解析这种类型的行：

	val populations: List[String~Int] = {
  		SQL("select * from Country").as( str("name") ~ int("population") * ) 
	}

正如你所见的，该查询结果类型为List[String~Int] - 一列国名和人口项目

你同时可以重写相同的代码如下：

	val result: List[String~Int] = {
  		SQL("select * from Country")
  		.as(get[String] ("name") ~ get[Int] ("population") *)
	}

String~Int类型如何？这是一个Anorm类型在你的数据库访问代码之外并不能带来简洁的使用。你更需要简单的元组(String, Int)来代替。你可以在一个RowPaser上使用map功能去转换他的结果为一个更简洁的类型：

	val parser = str("name") ~ int("population") map { case n~p => (n,p) }

    Note: We created a tuple (String,Int) here, but there is nothing stopping you from transforming the RowParser result to any other type, such as a custom case class.

现在，因为转换 A ~ B ~ 类型为（A, B, C）是一个普通任务，我们可以提供一个平滑功能作相同的事情。所以你最终可以写为：

	val result: List[(String, Int)] = 
  		SQL("select * from Country").as(parser.*)

如果列表不为空，parser.+可以被用来替代 parser.*.

## 一个更加复杂的例子 ##

现在让我们尝试一个更加复杂的例子。如何解析以下查询的结果以便为一个国家的编码获取相关的国名和所有语言。

	select c.name, l.language from Country c 
    	join CountryLanguage l on l.CountryCode = c.Code 
    	where c.code = 'FRA'

让我们解析所有的行为一个 List[(String,String)] (一列name,Language元组)开始：

	var p: ResultSetParser[List[(String,String)]] = {
  		str("name") ~ str("language") map(flatten) *
	}

现在我们得到了此类的结果:

	List(
  		("France", "Arabic"), 
  		("France", "French"), 
  		("France", "Italian"), 
  		("France", "Portuguese"), 
  		("France", "Spanish"), 
  		("France", "Turkish")
	)

我们接着可以使用scala "聚集"API， 以便将其转换为期望的结果：

	case class SpokenLanguages(country:String, languages:Seq[String])

	languages.headOption.map { f =>
  		SpokenLanguages(f._1, languages.map(_._2))
	}

最终， 我们将得到适当的功能：

    case class SpokenLanguages(country:String, languages:Seq[String])

    def spokenLanguages(countryCode: String): Option[SpokenLanguages] = {
      val languages: List[(String, String)] = SQL(
      """
      select c.name, l.language from Country c 
      join CountryLanguage l on l.CountryCode = c.Code 
      where c.code = {code};
      """
    )
    .on("code" -> countryCode)
    .as(str("name") ~ str("language") map(flatten) *)

    languages.headOption.map { f =>
    SpokenLanguages(f._1, languages.map(_._2))
    }
    }


继续，让我们复杂化我们的示例以便将官方语言从其他语言中分离出来：

	case class SpokenLanguages(
  		country:String, 
  		officialLanguage: Option[String], 
  		otherLanguages:Seq[String]
		)


   		 def spokenLanguages(countryCode: String): Option[SpokenLanguages] = {
				val languages: List[(String, String, Boolean)] = SQL(
    			"""
      			select * from Country c 
      			join CountryLanguage l on l.CountryCode = c.Code 
      			where c.code = {code};
    			"""
  				)
  				.on("code" -> countryCode)
  				.as {
    			str("name") ~ str("language") ~ str("isOfficial") map {
     		 	case n~l~"T" => (n,l,true)
      			case n~l~"F" => (n,l,false)
    			} *
  	}

  		languages.headOption.map { f =>
    			SpokenLanguages(
      			f._1, 
      			languages.find(_._3).map(_._2),
      			languages.filterNot(_._3).map(_._2)
    			)
 		 }
	}


如果你在MySQL 示例数据库"world"中尝试，你将得到：

	$ spokenLanguages("FRA")

	> Some(
    	SpokenLanguages(France,Some(French), List(
    	Arabic, Italian, Portuguese, Spanish, Turkish
    		))
		)

## **JDBC 映射** ##

正如在该文档中所见到的，Anorm提供了基于JDBC和JVM类型之间的嵌入的转换器。

**列解析器**

以下表格描叙了哪种JDBC数据类型（getters 在 java.sql.ResultSet中, 第一列）可以被解析为哪种java/scala类型（例如， 整数列可被读为双整数值 ）。

    ↓jdbc / jvm➞ bigdecimal1 biginteger2 boolean byte double float 	int    long    short

    bigdecimal1 	yes 	   yes 	        no 	   no 	yes 	no 	yes 	yes 	no

    biginteger2 	yes 	   yes 	        no 	   no 	yes 	yes yes 	yes 	no

    boolean 	    no         no 	        yes    yes 	no 	    no 	yes 	yes 	yes

    byte 	        yes 	   no 	        no 	   yes 	yes 	yes no 	    no 	    yes

    double      	yes 	   no 	        no 	   no 	yes 	no 	no 	    no 	    no

    float 	        yes 	   no 	        no 	   no 	yes 	yes no  	no 	    no

    int 	        yes 	   yes 	        no 	   no 	yes 	yes yes 	yes 	no

    long 	        yes 	   yes 	        no 	   no 	no 	    no 	yes 	yes 	no

    short 	        yes 	   no 	        no 	   yes 	yes 	yes no 	    no 	    yes

        
        类型 java.math.BigDecimal 和 scala.math.BigDecimal.
        类型 java.math.BigInteger 和 scala.math.BigInt.


第二张表显示其他支持的类型（texts, dates,...）的映射。

    ↓JDBC / JVM➞ 	Char 	Date 	String 	UUID3

    Clob 			Yes 	No 	    Yes 	No

    Date 	        No 	    Yes 	No 	    No

    Long 	        No  	Yes 	No  	No

    String 	        Yes 	No 	    Yes 	Yes

    UUID 	        No 	    No 	    No 	    Yes

    
    类型 java.util.UUID.

一旦T被支持，可选的列将可以被解析为 Option[T]。