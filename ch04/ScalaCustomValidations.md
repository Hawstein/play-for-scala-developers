# Custom Validations

[验证包](https://www.playframework.com/documentation/2.3.x/api/scala/index.html#play.api.data.validation.package) 允许你调用 `verifying` 方法来创建专门的约束。Play 还提供了用 `Constraint` 样本类的方法来自定义约束。

这里，我们会实现一个简单的密码强度约束，通过正则表达式来验证密码不是由全字母或是全数字组成。`Constraint` 接受一个返回 `ValidationResult` 的函数，我们使用这个函数来返回密码验证的结果：

```scala
val allNumbers = """\d*""".r
val allLetters = """[A-Za-z]*""".r

val passwordCheckConstraint: Constraint[String] = Constraint("constraints.passwordcheck")({
  plainText =>
    val errors = plainText match {
      case allNumbers() => Seq(ValidationError("Password is all numbers"))
      case allLetters() => Seq(ValidationError("Password is all letters"))
      case _ => Nil
    }
    if (errors.isEmpty) {
      Valid
    } else {
      Invalid(errors)
    }
})
```

**注意：** 这个例子是为了演示自定义约束而故意设计的。关于正确的密码安全设计，请参考 [OWASP 指南](https://www.owasp.org/index.php/Authentication_Cheat_Sheet#Implement_Proper_Password_Strength_Controls)

我们还可以结合 `Constraints.min` 来给密码添加额外的一层验证、

```scala
val passwordCheck: Mapping[String] = nonEmptyText(minLength = 10)
  .verifying(passwordCheckConstraint)
```