# Custom Field Constructors

一个表单域的渲染并不仅仅需要包含 `<input>` 标签，还有 `<label>` 和其他一些标签可能被你的 CSS 框架用来装饰表单域。

所有的输入 helper 都会接受一个隐式（implicit）的 `FieldConstructor` 来处理这一部分。[默认的构造器](https://www.playframework.com/documentation/2.3.x/api/scala/index.html#views.html.helper.defaultFieldConstructor$)（在作用域内没有指定其他域构造器时会被调用），生成的 HTML 类似于：

```scala
<dl class="error" id="username_field">
    <dt><label for="username">Username:</label></dt>
    <dd><input type="text" name="username" id="username" value=""></dd>
    <dd class="error">This field is required!</dd>
    <dd class="error">Another error</dd>
    <dd class="info">Required</dd>
    <dd class="info">Another constraint</dd>
</dl>
```

这个默认的域构造器支持传入额外的选项到输入 helper：

```scala
'_label -> "Custom label"
'_id -> "idForTheTopDlElement"
'_help -> "Custom help"
'_showConstraints -> false
'_error -> "Force an error"
'_showErrors -> false
```

## 自定义域构造器

通常，你需要自定义域构造器。首先要写一个模板：

```scala
@(elements: helper.FieldElements)

<div class="@if(elements.hasErrors) {error}">
    <label for="@elements.id">@elements.label(elements.lang)</label>
    <div class="input">
        @elements.input
        <span class="errors">@elements.errors(elements.lang).mkString(", ")</span>
        <span class="help">@elements.infos(elements.lang).mkString(", ")</span>
    </div>
</div>
```

**注意：** 这只是一个例子。你想要实现多复杂的功能都可以。你同样还能使用 `@elements.field` 来获取原始的表单域。

现在使用模板函数来创建 `FieldConstructor`:

```scala
object MyHelpers {
  import views.html.helper.FieldConstructor
  implicit val myFields = FieldConstructor(html.myFieldConstructorTemplate.f)
}
```

还可以让表单 helper 来调用他，只需要将其导入到你的模板中：

```scala
@import MyHelpers._
```

```scala
@helper.inputText(myForm("username"))
```

模板会使用你的域构造器来渲染输入文本。

你也可以为 `FieldConstructor` 设置一个隐式（implicit）的值：

```scala
@implicitField = @{ helper.FieldConstructor(myFieldConstructorTemplate.f) }
```

```scala
@helper.inputText(myForm("username"))

```