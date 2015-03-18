# Common use cases

模板作为简单的函数，可以由任意的方式构成。下面是一些常见的使用情况。

## 页面布局（layout）

让我们来声明一个 `views/main.scala.html` 模板来作为主要的布局模板：

```html
@(title: String)(content: Html)
<!DOCTYPE html>
<html>
  <head>
    <title>@title</title>
  </head>
  <body>
    <section class="content">@content</section>
  </body>
</html>
```

正如你所见，这个模板接收两个参数：标题（title）和 HTML 内容块（content）。下面我们在另一个模板（`views/Application/index.scala.html`）中使用它：

```html
@main(title = "Home") {

  <h1>Home page</h1>

}
```

> 注意：有时候我们使用命名参数，如 `@main(title = "Home")`，而不是 `@main("Home")`。这个视具体情况，选择一个表述清楚的即可。

有时候，你需要另一个页面内容来作为侧边栏（sidebar），这时你可以添加一个额外的参数来做到：

```html
@(title: String)(sidebar: Html)(content: Html)
<!DOCTYPE html>
<html>
  <head>
    <title>@title</title>
  </head>
  <body>
    <section class="sidebar">@sidebar</section>
    <section class="content">@content</section>
  </body>
</html>
```

在我们的 `index` 模板中使用：

```html
@main("Home") {
  <h1>Sidebar</h1>

} {
  <h1>Home page</h1>

}
```

或者，我们可以单独声明侧边栏：

```html
@sidebar = {
  <h1>Sidebar</h1>
}

@main("Home")(sidebar) {
  <h1>Home page</h1>

}
```

## 标签（它们也是函数）

下面我们来写一个简单的标签 `views/tags/notice.scala.html`，用于显示 HTML 通知：

```html
@(level: String = "error")(body: (String) => Html)

@level match {

  case "success" => {
    <p class="success">
      @body("green")
    </p>
  }

  case "warning" => {
    <p class="warning">
      @body("orange")
    </p>
  }

  case "error" => {
    <p class="error">
      @body("red")
    </p>
  }

}
```

现在，我们在另一个模板中使用它：

```html
@import tags._

@notice("error") { color =>
  Oops, something is <span style="color:@color">wrong</span>
}
```

## Includes

同样没有任何特别之处，你可以调用任意其它模板（事实上可以调用任意地方的任意函数）：

```html
<h1>Home</h1>

<div id="side">
  @common.sideBar()
</div>
```

## moreScripts 与 moreStyles 等价物

想在 Scala 模板中定义 moreScripts 和 moreStyles 等价物（像在 Play! 1.x 那样），你只需像下面那样在主模板中定义一个变量：

```html
@(title: String, scripts: Html = Html(""))(content: Html)

<!DOCTYPE html>

<html>
    <head>
        <title>@title</title>
        <link rel="stylesheet" media="screen" href="@routes.Assets.at("stylesheets/main.css")">
        <link rel="shortcut icon" type="image/png" href="@routes.Assets.at("images/favicon.png")">
        <script src="@routes.Assets.at("javascripts/jquery-1.7.1.min.js")" type="text/javascript"></script>
        @scripts
    </head>
    <body>
        <div class="navbar navbar-fixed-top">
            <div class="navbar-inner">
                <div class="container">
                    <a class="brand" href="#">Movies</a>
                </div>
            </div>
        </div>
        <div class="container">
            @content
        </div>
    </body>
</html>
```

对于一个需要额外脚本的扩展模板，用法如下：

```html
@scripts = {
    <script type="text/javascript">alert("hello !");</script>
}

@main("Title",scripts){

   Html content here ...

}
```

如果扩展模板不需要额外脚本，则：

```html
@main("Title"){

   Html content here ...

}
```
