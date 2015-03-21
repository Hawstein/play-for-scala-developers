# Handling form submission

## 概述

表单的处理和提交是每个 web 应用的重要部分。Play 自带了一些功能，简化了简易表单的处理，并使处理复杂表单成为可能。

Play 的表单处理方式基于数据绑定的概念。当数据来自于一个 POST 请求时，Play 会查找格式化的值并将其绑定到 `Form` 对象上。Play 可以根据绑定的表单及数据构建一个样本类（case class），调用自定义的验证器，等等。

通常，表单会在 `Controller` 实例中直接使用。不过 `Form` 的定义并不需要和样本类（case class）或是模型一致匹配：他们纯粹都是为了处理输入，不同的 POST 使用不同的 `Form` 非常合理。

## 导入

为了使用表单，必须在类中先导入以下几个包：

```scala
import play.api.data._
import play.api.data.Forms._
```

## 表单基础

让我们先来看一下基本的表单处理：

* 定义一个表单，
* 定义表单中的约束，
* 在一个 action 中验证该表单，
* 在视图模板中显示表单，
* 最后，在视图模板中处理表单的结果（或是错误）

最终的结果看起来是这样的：

![Form Result](https://www.playframework.com/documentation/2.3.x/resources/manual/scalaGuide/main/forms/images/lifecycle.png)

### 定义一个表单

首先，定义一个包含了表单中所有元素的样本类（case class）。这里我们想要获取一个用户的名字和年龄，因此我们创建了一个 `UserData` 对象：

```scala
case class UserData(name: String, age: Int)
```

现在我们有了一个样本类（case class），接着需要定义一个 `Form` 结构。定义表单的方法是将表单的数据绑定到样本类（case class）的实例中，定义如下：

```scala
val userForm = Form(
  mapping(
    "name" -> text,
    "age" -> number
  )(UserData.apply)(UserData.unapply)
)
```

[表单](https://www.playframework.com/documentation/2.3.x/api/scala/index.html#play.api.data.Forms$)对象定义了 `mapping` 方法。这个方法接收表单的名字和约束作为参数，并接收另外两个方法：一个 `apply` 方法和一个 `unapply` 方法。因为 `UserData` 是一个样本类（case class），我们可以将 `apply` 和 `unapply` 直接插入到 `mapping` 方法中。

**注意：**由于表单处理的实现问题，一个单一元组（tuple）或是 mapping 最多只能有18个元素。如果你的表单有超过18个元素，你需要将他们通过列表或是嵌套值分开来。

当你使用了 Map ，表单会创建一个带有绑定值的 UserData 实例：

```scala
val anyData = Map("name" -> "bob", "age" -> "21")
val userData = userForm.bind(anyData).get
```

但大多数的时候会在 Action 中使用表单，其中的数据由请求提供。`Form` 包含了 `bindFormRequest`，接收一个请求作为隐式（implicit）参数。如果你定义了一个隐式（implicit）请求，那么 `bindFormRequest` 就能够找到他。

```scala
val userData = userForm.bindFromRequest.get
```

**注意：**这里使用 `get` 其实是有问题的。如果表单没能绑定数据，那么 `get` 会抛出异常。在接下来的几章中我们会介绍一种更安全的方法来处理输入。

Play 并没有限制说只能用样本类（case class） 来映射你的表单。只要正确设置了 `apply` 和 `unapply` 方法，就能传入你想要的任何东西，比如元组可以使用 `Forms.tuple` 来映射，或是模型样本类（model case class）。但是，为表单指定一个样本类（case class）有以下这些好处：

* **表单指定的样本类（case class）简单易用。**样本类（case class）原本便被设计为存储数据的简易容器，即写即用，和表单功能天然匹配。
* **表单指定的样本类（case class）功能强大。**元组易于使用，但元组并不允许你自定义 `apply` 或是 `unapply` 方法，且只能通过元数（arity，如 _1, _2 等）来引用其中的数据。
* **表单指定的样本类（case class）专为表单设计。**重用模型样本类（model case class）确实很方便，但模型通常都含有一些额外的领域逻辑，甚至于一些数据持久化的细节，这些都会导致耦合过于紧密。另外，如果表单和模型并不是1：1严格映射的话，敏感数据必须被显示的忽略，以此来抵御[篡改参数](https://www.owasp.org/index.php/Web_Parameter_Tampering)攻击。

### 定义表单的约束

`text` 约束认定空字符串依然有效。也就是说 `name` 可以为空且不会报错，这显然不是我们想要的。一个保证 `name` 有值的方法是使用 `nonEmptyText` 约束。

```scala
val userFormConstraints2 = Form(
  mapping(
    "name" -> nonEmptyText,
    "age" -> number(min = 0, max = 100)
  )(UserData.apply)(UserData.unapply)
)
```

如果表单的输入没有满足该表单的约束条件，则会报错：

```scala
val boundForm = userFormConstraints2.bind(Map("bob" -> "", "age" -> "25"))
boundForm.hasErrors must beTrue
```

预置的约束定义在了[表单对象](https://www.playframework.com/documentation/2.3.x/api/scala/index.html#play.api.data.Forms$)中：

* `text`：对应于 `scala.String`，可选参数为 `minLength` 和 `maxLength`。
* `nonEmptyText`：对应于 `scala.String`，可选参数为 `minLength` 和 `maxLength`。
* `number`：对应于 `scala.Int`，可选参数为 `min`, `max` 和 `strict`。
* `longNumber`：对应于 `scala.Long`，可选参数为 `min`， `max`，和 `strict`。
* `bigDecimal`：参数为 `precision` 和 `scale`。
* `date`，`sqlDate`，`jodaDate`：对应于 `java.util.Date`， `java.sql.Date` 和 `org.joda.time.DateTime`，可选参数为 `pattern` 和 `timeZone`。
* `jodaLocalDate`：对应于 `org.joda.time.LocalDate`，可选参数为 `pattern`。
* `email`：对应于 `scala.String`，使用邮件正则表达式.
* `boolean`：对应于 `scala.Boolean`。
* `checked`：对应于 `scala.Boolean`。
* `optional`：对应于 `scala.Option`。

### 定义特殊约束

你可以使用[验证包](https://www.playframework.com/documentation/2.3.x/api/scala/index.html#play.api.data.validation.package)在样本类（case class）中定义你自己的特殊约束。

```scala
val userFormConstraints = Form(
  mapping(
    "name" -> text.verifying(nonEmpty),
    "age" -> number.verifying(min(0), max(100))
  )(UserData.apply)(UserData.unapply)
)
```

你也可以直接在样本类（case class）中定义特殊约束：

```scala
def validate(name: String, age: Int) = {
  name match {
    case "bob" if age >= 18 =>
      Some(UserData(name, age))
    case "admin" =>
      Some(UserData(name, age))
    case _ =>
      None
  }
}

val userFormConstraintsAdHoc = Form(
  mapping(
    "name" -> text,
    "age" -> number
  )(UserData.apply)(UserData.unapply) verifying("Failed form constraints!", fields => fields match {
    case userData => validate(userData.name, userData.age).isDefined
  })
)
```

当然，你还能定义你自己的验证器。详情请见[自定义验证器](https://www.playframework.com/documentation/2.3.x/ScalaCustomValidations)。

### 在 Action 中验证表单

现在已经有了约束，我们需要在 action 中验证表单，并处理表单错误。

我们使用 `fold` 方法来实现，它接收了两个函数：绑定失败时第一个会被调用，绑定成功则会调用第二个。

```scala
userForm.bindFromRequest.fold(
  formWithErrors => {
    // binding failure, you retrieve the form containing errors:
    BadRequest(views.html.user(formWithErrors))
  },
  userData => {
    /* binding success, you get the actual value. */
    val newUser = models.User(userData.name, userData.age)
    val id = models.User.create(newUser)
    Redirect(routes.Application.home(id))
  }
)
```

绑定失败时，我们用了 BadRequest 来渲染页面，并将错误作为参数传入该页。如果使用了视图 helper （在下面讨论），那么任何绑定于一个元素的错误都会被渲染在该元素边上。

绑定成功时，我们发出了一个 `Redirect` ，路由到 `routes.Application.home`，而不是渲染一个视图模板。这种模式称为 [POST 后重定向](http://en.wikipedia.org/wiki/Post/Redirect/Get)，这是一种很好的防止重复提交的方式。

**注意**：当使用了 `flashing` 或是在其他方法中用到了 [flash 域](../ch01/ScalaSessionFlash.md)，使用 “POST 后重定向” 是**必需的**，因为新的 cookie 只能在重定向 HTTP 请求后获取。

### 在视图模板中显示表单

有了表单之后，我们就能在[模板引擎](../ch03/ScalaTemplates.md)中调用它。做法是在视图模板中将表单作为参数引入。对于 `user.scala.html`来说，该页的开头是这样的：

```scala
@(userForm: Form[UserData])
```

由于 `user.scala.html` 需要接收一个表单，在渲染 `user.scala.html` 时应先传入一个空的 `userForm`。

```scala
def index = Action {
  Ok(views.html.user(userForm))
}
```

首先是要创建[表单标签](http://www.w3.org/TR/html5/forms.html#the-form-element)。这是一个简单的视图 `helper`， 创建了一个[表单标签](http://www.w3.org/TR/html5/forms.html#the-form-element)，并根据你所传入的逆向路由设置了 `action` 和 `method` 的标签参数。

```scala
@helper.form(action = routes.Application.userPost()) {
  @helper.inputText(userForm("name"))
  @helper.inputText(userForm("age"))
}
```

你可以在 `views.html.helper` 包中找到多个输入 helper。传入一个表单域，他们就能显示出相应的 HTML 输入，设置值和约束，并在表单绑定失败时报错。

**注意：**你可以在模板中使用 `@import heler._` 来避免在调用 helper 时前置 `@helper`。

Play 有多个输入 helper，其中最有用的是：

* `form`: 渲染[表单](http://www.w3.org/TR/html-markup/form.html#form)。
* `inputText`: 渲染[文本输入](http://www.w3.org/TR/html-markup/input.text.html)。
* `inputPassword`: 渲染[密码输入](http://www.w3.org/TR/html-markup/input.password.html#input.password)。
* `inputDate`: 渲染[日期输入](http://www.w3.org/TR/html-markup/input.date.html)。
* `inputFile`: 渲染[文件输入](http://www.w3.org/TR/html-markup/input.file.html)。
* `inputRadioGroup`: 渲染[单选输入](http://www.w3.org/TR/html-markup/input.radio.html#input.radio)。
* `select`: 渲染[下拉列表](http://www.w3.org/TR/html-markup/select.html#select)。
* `textarea`: 渲染[文本域](http://www.w3.org/TR/html-markup/textarea.html#textarea)。
* `checkbox`: 渲染[复选框](http://www.w3.org/TR/html-markup/input.checkbox.html#input.checkbox)。
* `input`: 渲染通用输入（需要显示声明参数）。

在 `form` helper 中，你可以通过指定额外的参数，将其添加到生成的 HTML中：

```scala
@helper.inputText(userForm("name"), 'id -> "name", 'size -> 30)
```

上面这个通用的 `input` helper 能让你编写你想要的 HTML 代码：

```scala
@helper.input(userForm("name")) { (id, name, value, args) =>
    <input type="text" name="@name" id="@id" @toHtmlArgs(args)>
}
```

**注意：**所有额外的参数都会被添加到生成的 HTML 中，除非他们以 `_` 字符开头。以 `_` 开头的参数是[域构造器参数](ScalaCustomFieldConstructors.md)。

对于复杂表单元素，你同样可以自定义视图 helper （在 `views` 页面中使用 scala 类）并自定义[域构造器](ScalaCustomFieldConstructors.md)。

### 在视图模板中显示错误

表单中的错误类型为 `Map[String, FormError]`，其中 `FormError`中有：

* `key`：应和表单域相同。
* `message`：一段错误提示信息或是对应于该信息的键。
* `args`：错误提示信息中用到的一组参数。

绑定的表单实例中可以获得如下错误：

* `errors`：返回所有错误，类型为 `Seq[FormError]`。
* `globalErrors`：返回所有错误，类型为没有键的 `Seq[FormError]`。
* `error("name")`：返回第一个绑定了该键的错误，类型为 `Option[FormError]`。
* `errors("name")`：返回所有绑定了该键的错误，类型为 `Option[FormError]`。

使用表单 helper 可以自动渲染绑定于某表单域的错误，如 `@helper.inputText` 的错误显示如下：

```scala
<dl class="error" id="age_field">
    <dt><label for="age">Age:</label></dt>
    <dd><input type="text" name="age" id="age" value=""></dd>
    <dd class="error">This field is required!</dd>
    <dd class="error">Another error</dd>
    <dd class="info">Required</dd>
    <dd class="info">Another constraint</dd>
</dl>
```

没有绑定于任一键的全局错误（Global errors）不会有相应的 helper，且必须显示的定义在页面中：

```scala
@if(userForm.hasGlobalErrors) {
  <ul>
  @for(error <- userForm.globalErrors) {
    <li>@error.message</li>
  }
  </ul>
}
```

### 使用元组做映射

你可以在表单域中使用元组，而非样本类（case class）：

```scala
val userFormTuple = Form(
  tuple(
    "name" -> text,
    "age" -> number
  ) // tuples come with built-in apply/unapply
)
```

有时候使用元组会比定义样本类（case class）更方便，尤其在元组的元数（arity）很小时：

```scala
val anyData = Map("name" -> "bob", "age" -> "25")
val (name, age) = userFormTuple.bind(anyData).get
```

### 使用 single 做映射

元组只有在有多个值的时候才有用。如果只有一个表单域，可以用 `Forms.single` 来映射单一值，避免了使用样本类（case class）或是元组的额外开销：

```scala
val singleForm = Form(
  single(
    "email" -> email
  )
)
```

```scala
val email = singleForm.bind(Map("email", "bob@example.com")).get
```

### 填值

有时候，你想要在表单中预置一些值，通常用于编辑数据：

```scala
val filledForm = userForm.fill(UserData("Bob", 18))
```

当配合视图 helper 使用时，该元素会填上预置的值：

```scala
@helper.inputText(filledForm("name")) @* will render value="Bob" *@
```

填值在 helper 需要一列值或是键值对时尤其有用，比如 `select` 和 `inputRadioGroup` helper 。使用 `options` 可以为 helper 填入列表，键值对和対值（pair）。

### 嵌套值

表单映射同样可以在已有的映射中使用 `Forms.mapping` 来实现嵌套值：

```scala
case class AddressData(street: String, city: String)

case class UserAddressData(name: String, address: AddressData)
```

```scala
val userFormNested: Form[UserAddressData] = Form(
  mapping(
    "name" -> text,
    "address" -> mapping(
      "street" -> text,
      "city" -> text
    )(AddressData.apply)(AddressData.unapply)
  )(UserAddressData.apply)(UserAddressData.unapply)
)
```

**注意：**当你使用这种方法来嵌套数据时，浏览器传来的表单值必须命名为 `address.street`，`address.city`...

```scala
@helper.inputText(userFormNested("name"))
@helper.inputText(userFormNested("address.street"))
@helper.inputText(userFormNested("address.city"))
```

### 重复值

表单映射可以使用 `Forms.list` 或 `Forms.seq` 来定义重复值：

```scala
case class UserListData(name: String, emails: List[String])
```

```scala
val userFormRepeated = Form(
  mapping(
    "name" -> text,
    "emails" -> list(email)
  )(UserListData.apply)(UserListData.unapply)
)
```

当你这么使用重复数据时，浏览器传入的表单值必须被命名为 `emails[0]`，`emails[1]`，`emails[2]`...

使用 `repeat` helper 来生成和表单 `emails` 域相同数目的输入：

```scala
@helper.inputText(myForm("name"))
@helper.repeat(myForm("emails"), min = 1) { emailField =>
    @helper.inputText(emailField)
}
```

`min` 参数允许你在表单数据为空时，定义最少显示多少个表单域。

### 可选值

使用 `Forms.optional` 来定义可选值：

```scala
case class UserOptionalData(name: String, email: Option[String])
```

```scala
val userFormOptional = Form(
  mapping(
    "name" -> text,
    "email" -> optional(email)
  )(UserOptionalData.apply)(UserOptionalData.unapply)
)
```

这样做会输出一个 `Option[A]`，在没有找到任何表单值的情况下则返回 `None`。

### 默认值

你可以使用 `Form#fill` 来初始化值：

```scala
val filledForm = userForm.fill(User("Bob", 18))
```

你也可以通过 `Forms.default` 来定义默认值：

```scala
Form(
  mapping(
    "name" -> default(text, "Bob")
    "age" -> default(number, 18)
  )(User.apply)(User.unapply)
)
```

### 忽略值

如果你想定义某个表单域为一个静态值，使用 `Forms.ignored`：

```scala
val userFormStatic = Form(
  mapping(
    "id" -> ignored(23L),
    "name" -> text,
    "email" -> optional(email)
  )(UserStaticData.apply)(UserStaticData.unapply)
)
```

## 合并起来

这个例子演示了使用模型和控制器来处理一个实体：

样本类 `Contact`：

```scala
case class Contact(firstname: String,
                   lastname: String,
                   company: Option[String],
                   informations: Seq[ContactInformation])

object Contact {
  def save(contact: Contact): Int = 99
}

case class ContactInformation(label: String,
                              email: Option[String],
                              phones: List[String])
```

需要注意的是，`Contact` 里有一个包含了 `ContactInformation` 元素的 `Seq` 和一个含有 `String` 的 `List`。在这个例子中，我们组合了嵌套映射和重复映射（分别由 `Forms.seq` 和 `Forms.list` 定义）。

```scala
val contactForm: Form[Contact] = Form(

  // Defines a mapping that will handle Contact values
  mapping(
    "firstname" -> nonEmptyText,
    "lastname" -> nonEmptyText,
    "company" -> optional(text),

    // Defines a repeated mapping
    "informations" -> seq(
      mapping(
        "label" -> nonEmptyText,
        "email" -> optional(email),
        "phones" -> list(
          text verifying pattern("""[0-9.+]+""".r, error="A valid phone number is required")
        )
      )(ContactInformation.apply)(ContactInformation.unapply)
    )
  )(Contact.apply)(Contact.unapply)
)
```

这段代码演示了一条已存在的 `Contact` 如何通过填入数据在表单中显示出来：

```scala
def editContact = Action {
  val existingContact = Contact(
    "Fake", "Contact", Some("Fake company"), informations = List(
      ContactInformation(
        "Personal", Some("fakecontact@gmail.com"), List("01.23.45.67.89", "98.76.54.32.10")
      ),
      ContactInformation(
        "Professional", Some("fakecontact@company.com"), List("01.23.45.67.89")
      ),
      ContactInformation(
        "Previous", Some("fakecontact@oldcompany.com"), List()
      )
    )
  )
  Ok(views.html.contact.form(contactForm.fill(existingContact)))
}
```

最后是表单提交：

```scala
def saveContact = Action { implicit request =>
  contactForm.bindFromRequest.fold(
    formWithErrors => {
      BadRequest(views.html.contact.form(formWithErrors))
    },
    contact => {
      val contactId = Contact.save(contact)
      Redirect(routes.Application.showContact(contactId)).flashing("success" -> "Contact saved!")
    }
  )
}
```