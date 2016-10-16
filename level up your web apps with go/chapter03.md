# HTTP

发送和接受`HTTP`请求是每一个`web`应用程序的核心工作。Go的标准库提供了很多包让我们轻松的驾驭http请求和响应。本章中，将会涉及响应请求，使用中间件（middleware）路由请求，创建`HTML`模板和响应`JSON`字串，所有这些功能都只依赖Go的标准库完成。

## 响应请求

Go创建web服务器相当简单。在多数其他语言，创建一个web服务器通常还需要一个额外的服务器软件，例如`PHP`需要`Apache`。Go的标准库提供了`http`包，我们无效依赖第三方包，仅使用`http`就能创建web服务。

有多种方式创建GOhttp服务，最简单的方式就是指定一个`url`模式（一下简称模式或URL模式），并注册一个**处理函数**，这个函数的签名是`func(w http.ResponeWriter, r *http.Request)`。一旦url匹配了模式，就会调用注册的函数处理请求。接下来可以调用`http.ListenAndServer`启动服务监听：

```
package main

)

```

这个示例中，当请求访问端口3000的任何路径时，注册的函数就会执行。函数中使用了`fmt.Fprintf`往`response writer`对象写入字串`“Hello Gpher”`来返回给客户端的响应。

> 路径模式匹配
> 
> 模式`/`将匹配任何路径的请求。关于url路径的模式匹配需要一点解释，后面的章节再详细阐述。


运行上述代码，使用浏览器访问`http://127.0.0.1:3000`，将会看到如下的显示：

![http hello world](./images/3.1.png)

### 一探究竟

处理函数（handler function）的两个参数提供了给任何请求响应所需要的全部功能。

第一个参数是`http.ResponseWriter`的实例，我们将返回给浏览器的信息写入这个对象中。响应主要由页面的主体（body）组成，不过也可以提供对其他信息其他例如响应头（headers）和状态码（status code）的访问。

> 所谓浏览器
> 
> 本书中发送任何方法的请求所用的“浏览器”不仅仅是浏览器软件，还可以是命令行工具例如`cURL`
> 如果安装了`cURL`， 可以使用命令`curl -i 127.0.0.1:3000`替代浏览器软件。

第二个参数是`http.Request`的实例，它涵盖我们期望看到的所有请求的信息，包括主体数据（data body），查询字串（query string）和请求头（headers）。

### 更多细节

现在的例子响应的信息并不多。实际上，真正的HTTP（raw HTTP） 响应也很小：

```
HTTP/1.1 200 OK

```

第一行的含义是告知接受一个处理成功的HTTP响应（OK HTTP）。然后是少许`header`信息，响应的日期`date`和`content`内容。注意`Content-Type`的值是`text/plain`。Go会读取响应主体的前512个字节并嗅探出响应的类型。例子中并没有明细的类型，因此返回了基本的text类型。除了`header`之外，剩下的内容为响应的主体。

要提供更多信息，我们需要操作处理函数中的`ResponseWriter`对象。通过一个命名贴切的`Header`方法返回一个`Header`结构的图来修改响应头`header`。可以通过其`Add`方法给响应头追加内容：

```
w.Header().Add("Server", "Go Server")
```

写入一些`HTML`标签，将会触发go嗅探出返回的`content-type`为`text/html`类型：

```
fmt.Fprintf(w, `<html>
```
再次运行服务器代码，将会收到新的`Server`响应头和`Content-type`：

```
HTTP/1.1 200 OK
```

### 深入URL模式匹配

因为所有的请求是 `http.ListenAndServe`所监听的端口接受处理。因此我们需要一种将不同资源的请求路由到我们代码的不同部分的方法。幸运的是Go提供了`http.ServeMux`结构原生支持这样的方式。`ServeMux`是一个请求多路复用器，它将请求的`URL`与模式中中`url`进行匹配，并执行最匹配模式的处理函数。

`http`包提供了一个默认的`DefaultServeMux`实例，并且`http.HandleFunc`也是`DefaultServeMux`方法`(*ServeMux) HandleFunc`的包装。

当你创建`ServeMux`的时候，你可以为不同的URL模式注册不同处理函数。模式（Pattern）不必完全匹配路径（Path）。 有两种类型的模式：路径（path）和子树（subtree）。路径的结尾没有斜杠`/`，匹配严格的路径。子树的结尾包含斜杠`/`，并且匹配所有以子树开头的url。因此，在下一个例子中，请求`/articles/latest`将会返回`“Hello from /articles/”`，因为url`/articles/latest`与模式中的`/articles/`子树匹配。可是访问`/users/latest`将会返回一个404错误，因为模式`/users`缺少尾部的`/`，需要完全匹配（译者注：下面注释部分为源代码，可是是有问题的用法，参考上下文和go源代码，正确的用法为非注释的地方）：

```
/*
func main() {

	mux := http.NewServeMux()
	mux.HandleFunc("/articles/", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintf(w, "Hello from /articles/")
	})
	mux.HandleFunc("/users", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintf(w, "Hello from /users")
	})
	http.ListenAndServe(":3000", mux)
}
*/

func main() {

	mux := http.NewServeMux()
	mux.HandleFunc("/articles/", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintf(w, "Hello from /articles/")
	})
	mux.HandleFunc("/users", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintf(w, "Hello from /users")
	})
	http.ListenAndServe(":3000", mux)
}

```

模式的长度也很重要。模式越长的优先级越高。例如，模式`/articles/latest/`要比`/articles/`优先级高。因为有了顺序优先级，所以与添加路由的顺序没有区别。添加在`/articles/latest/`的路由规则在`/articles/`的处理程序之前，实际访问的时候也和预期绝对一致：

```
func main() {
	mux := http.NewServeMux()
	mux.HandleFunc("/articles/", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintf(w, "Hello from /articles/")
	})
	mux.HandleFunc("/articles/latest/", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintf(w, "Hello from /articles/latest/")
	})
	mux.HandleFunc("/users", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintf(w, "Hello from /users")
	})
	http.ListenAndServe(":3000", mux)
}

```
> 主机名匹配
> 
> 路由匹配的时候也可以从主机名开始，并且只有匹配的主机名才算真正的匹配。主机名匹配的模式拥有最高的优先级。

### 返回错误

有时候事与愿违，程序并不会正常工作，此时就需要返回一个错误，至少需要返回与`200 ok`不一样的响应。Web开发中状态码至关重要；如果没有状态码，访问不存在的资源的时候，或者网页的移动了，又或者遇了问题的时候希望能够回退。

常见的几个状态码：

■ 301 Moved Permanently― 重定向页面

`http`包提供另一个`Error`函数用于返回错误，它需要两个参数，一个是`ResponseWriter`，另一个是整型的状态码。例如，500的错误返回如下：

```
http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
```

每个状态代码都有语义，所以你应该努力使用最合适的。如果有疑问，请记住，你并不是第一个吃螃蟹的人，在你试图找到合适的状态码之前，可能有无数的人已经尝试做了。Google是你的好朋友。完整的状态代码的请查看维基百科文章：[List of HTTP status codes](https://en.wikipedia.org/wiki/List_of_HTTP_status_codes)

> 状态码常量
> 
> Go的http包提供了很多HTTP状态码的常量。你可以直接使用这些常量而不是自己声明整数的code。`http.StatusBadRequest`就比 400 更容易理解。
> 
> 几个常见的的`Helper函数`：
> `http.NotFound` 可能是最普遍错误码是 `404 Not Found`。`http.NotFound`函数的参数为`ResponseWriter` 和`Request`类型的实例。（`http.NotFound(w, r)`）
> 
> `http.Redirect` 它的参数除了是`ResponseWriter`和`Request`的实例之外还有两个参数，一个是字串类型的参数，表示需要重定向的`url`；另外一个是整型的数字状态码。重定向的状态码有好几个。有多种类型的重定向状态代码。 最常见的是301用于永久移动的资源，302用于临时移动的资源。（` http.Redirect(w, r, "http://google.com", http.StatusMovedPermanently)`）

### Handler接口

还有另一种方式注册函数来处理`HTTP`请求。任何实现接口`http.Handler`的类型都可以和模式字串一起传递到`http.Handle`函数。 正如我们之前使用`HandleFunc`函数看到的，这是使用默认ServeMux实例的快捷方式。

`http.Handler`接口只有一个函数，这个函数的签名和`http.HandleFunc`函数一致。

```
type Handler interface {
```
我们可以给任何类型实现`ServeHTTP`方法。如果我们想创建一个handler函数用来处理返回服务器当前的时间的响应。例如定义一个类型`UptimeHandler`并实现`Handler`接口的`ServeHTTP`方法：

```
package main
   return UptimeHandler{ Started: time.Now() }
```

使用浏览器访问`http://127.0.0.1:3000`，就能看见服务器返回的当前时间。`time.Since`函数返回一个`time.Duration`结构对象，表示自服务器启动以来所经过的时间。然后`time.Duration`类型被格式化为人类可读的字串; 例如，`"Current Uptime: 4m20.843867792s"`。

### 中间件

在Web开发中，**中间件**（Middleware）是指一系列处理程序，它们围绕Web应用程序，添加额外的功能。这是一个常见的概念，虽然经常被忽略，可是一旦正确的使用就非常强大。中间件可用于验证用户，压缩响应内容，限制请求速率，捕获应用程序异常等等。

使用`http.Handlers`创建中间件很普通，也很容易。例如，我们可以创建一个中间件，他的功能就是检查请求中查询字串的`token`是否合法，否则则返回一个`404 Not Found`的响应错误。

```
// SecretTokenHandler 检查 secret token

// SecretTokenHandler 实现了 http.Handler 接口的 ServeHTTP方法.
    // 检查 查询字串的 token
        // 检查token合法，调用下一个handler函数

```

curl -i 127.0.0.1:3000
 
HTTP/1.1 404 Not Found
```

如果添加了token，将会返回下一个处理函数的处理响应结果：

```
curl -i 127.0.0.1:3000/?secret_token=MySecret
```

## HTML 模板

到目前为止，我们看过的例子是微不足道的，旨在检查一些特定的用例。如果我们想要开始返回更复杂的响应，该怎么办呢？入金仅仅使用`fmt`包来来生成`HTML`将会很棘手。

幸运的是Go提供了原生的`HTML`模板处理包`html/template`。该包不仅能让我们轻而易举格式化来自Go数据的`HTML`页面，还能正确处理`escaping`转义字符的`HTML`输出。大多数情况下，都无需担心传入模板数据的escaping转义问题，Go将会帮你处理。

> Escaping转义
> 
> Escaping input on an HTML page is extremely important for both layout and se- curity reasons. Escaping is the process of turning characters that have special meaning in HTML into HTML entities. For example, an ampersand is used to start an HTML entity, so if you wanted to display an ampersand correctly, you would have to write the HTML entity for one instead: &amp;. If you don’t escape any data you try to display in your templates, then at best you might allow people to accidentally break your page layout, and at worst you open up your pages to ma- licious attackers.
> 




`html/template`包将分两步工作。首先需要需要将HTML字符模板解析成为`Template`类型。然后执行注入模板的数据结构来生成`HTML`字串。

无论是从文件中载入还是直接在Go代码中定义，模板都是从纯文件字符串中创建。变量替换还是被称之为**action**控制结构，都是通过花括号`{{` 和 `}}`对包裹。任何它们之外的字符都不会被修改。

一个简单的模板输出的例子看起来是这样的：

```
package main
```
每一个模板都需要命名。原因稍后解释，目前都可以命名为“Foo”。

如你所见，例子中的`{{.}}`称之为**点action**，它指的是传递到模板中的数据。因为我们只是传递一个字符串，所有我们需要做的是直接引用数据，但点可以赋值多次不同的数据; 例如，当循环数据时，`.`将分配当前迭代的值。我们将会看到在模板包中点将会被重复使用。

### 访问模板数据

模板中访问更复杂的数据类型也相当简单。虽然在Go访问结构的字段，图的key或者简单的函数不一样，但是在模板中访问方式却很相似。如果我们在渲染模板的时候注入一个结构体，可以通过它们的字段名访问其可以导出的字段。下面例子将会打印输出：`“The Go html/template package” by Mal Curtis`：

```
type Article struct{
```
上述规则同样适用与图，即通过图的key访问其值。与结构体不一样，key无需以大写字母开头：

```
func main(){
    err = tmpl.Execute(os.Stdout, article)
```


不带参数的函数或方法也可以以相同的方式调用（在Go文档中称为`niladic`函数）。 这个例子将输出`"Written by Mal Curtis"`：

```
type Article struct{
```

### 模板中的if else 条件判断

现在我们介绍了如何访问模板中不同类型数据结构，接下来看看是如何通过条件和循环结构控制模板中的执行流程。

就像Go代码一样，模板通过`if`和`else`语句控制模板的条件流程。这没有什么难度，它将为模板提供大量的限制逻辑的部分。


就像在Go代码中一样，我们可以通过使用If和Else语句来访问条件流。 这里没有太多的困难 - 它将弥补模板有限的逻辑的大部分。 因为我们不能使用括号来指定属于`if`语句的内容，所以我们使用`end`语句来表示结束边界。 如果文章尚未发布，此示例会将`"Draft"`一词附加到标题中：

```
type Article struct{
    Draft bool 
}

```
你也可以在模板中使用可选的`{{ else }}`语句来执行条件不满足的情况，`{{if .Draft}} (Draft){{else}} (Published){{end}}`。

### 循环

 可以使用`range`来迭代切片或者图。range也可以提供一个可选的`{{ else }}`语句，当循环结束的时候，没有可以循环的项的时候使用else语句对列出一些消息佷有用。。

在每个循环中，`.`被设置为循环变量的值。如果你在一个循环中调用点动作`{{.}}`，它将在循环的每次迭代中设置为不同的值。在这个例子中，我们将获得`Article`的`Name`和`AuthorName`的列表，或者一条消息`"No published articles yet"`：

```
func main(){

    if err != nil { panic(err) }

```

### Multiple 模板

当解析模板的时候，实际上可以同时定义多个模板。这样就在运行代码的时候，也可以选择使用哪一个模板，或者在一个模板中调用另外一个模板。这给模板提供了强大的功能，鼓励模板代码的重用---普遍的做法是创建多个模板块。

> 添加模板
> 
> 你不必在一次`Parse`调用的时候添加多个模板，而是可以多次调用`Parse`，或者使用`ParseFile`和`ParseGlob`方面从文件中载入模板。后续的章节将会介绍`Glob`模块。



### 管道过滤器（Piplines）
### 模板变量

模板变量并不想之前介绍的概念那么重要，当然你应该知道你不必每次都访问传入到模板的数据值。你可以在模板中定义变量。之所以这样做的原因是当你想在模板多个地方以相同的方式格式化一个值的时候。它更容易和更快地把格式化的值赋给一个变量，并在多个地方输出。

变量作为管道的结果，看起来与常规Go变量略有不同，因为它们必须以美元符号`$`开头。 我们可以用这种方式重写前面的示例的模板：

```
{{$total := multiply .Price .Quantity}}
```

> 变量作用域（Scope）
> 
> 变量的作用域只在其定义的代码块中。`if`或者`range`语句里定义的变量，在其外面的代码块将无效。



## 渲染 JSON
### 序列化（Marshling）
### 序列化结构体
### 自定义JSON字段名
### 内嵌类型
### 反序列化（Unmarshling）
### 未知的JSON结构处理
## 总结


