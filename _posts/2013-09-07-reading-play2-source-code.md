---
layout: post
title: "Reading Play2 Source Code"
description: ""
category: dev
tags: [play2,scala]
---
{% include JB/setup %}

## 前言
[PlayFramework] （以下简称 Play）是个主要使用 [Scala] 语言开发的全栈式 Web 开发框架，开发者可以使用 Java 或者 Scala 开发自己的程序。Play 为了支持 Java，还包含了一些 Java 代码，这里不再阅读其 Java 代码，只阅读 Scala 代码。

代码基线：以 Play2.2 主干为准。

理解有一个过程，因此会根据新的理解随时刷新本文。

[PlayFramework]: http://wwww.playframework.com PlayFramework 官网
[Scala]: http://www.scala-lang.org Scala

## 总体架构
Play 模块划分很清晰，也提供了插件机制，用于扩展。例如 i18n、db 等功能都是插件方式提供的，但一些其他功能还没有插件化，依旧作为 Play 的子模块提供的，Play 团队也意识到了这一点，会持续改进插件机制，以便更多功能插件化。这个工作会在 Play 2.3 版本中完成。

若要用数学表达式描述 Play 的构成，则可以使用下面这个：

    Play = Core + API + N * Plugin
Core 主要是底层通信协议，使用 Netty 实现服务器通信；而 API 是实现了各种功能时也对外提供 API 能力，用户可以使用 API 的功能；Plugin 就是各种扩展。这里只先阅读 Core + API 的部分代码。

## 启动
任何程序都有源头，那么我们就从程序启动开始看吧。

在新建应用中运行下面命令到底发生了什么？

    play run

这个命令是在工程目录下运行的，它会启动 SBT：

    if [ -n "$1" ]; then
        JPDA_PORT="${JPDA_PORT}" $dir/framework/build "$@"
    else
        JPDA_PORT="${JPDA_PORT}" $dir/framework/build play
    fi
由于携带了 run 参数，因此会走第一个分支，否则会直接调用 build play. 有上述代码可以看出，运行 play 和 play play 效果是一样的。

这里啰嗦一句，如果是在非工程目录，例如新建 play 应用，那么会调用下面语句，通过 play.console.Console 类加载:

    "$JAVA" -Dplay.home=$dir/framework -Dsbt.boot.properties=$dir/framework/sbt/play.boot.properties ${PLAY_OPTS} -jar $dir/framework/sbt/sbt-launch.jar "$@"

这里说明一下两个 SBT 启动环境，一个是 **play.boot.properties**，用于新建 play 应用；另一个是 **sbt.boot.properties**, 用于编译、启动 play 应用。后者所有的命令都在 sbt-plugin 工程中，所以在每个 play 应用的 plugins.sbt 文件中，都定义以下内容，就是用于加载 play 的 sbt-plugin:

    addSbtPlugin("com.typesafe.play" % "sbt-plugin" % "2.2-SNAPSHOT")

好了，有关 SBT 的介绍就是这些，SBT 就像它的名字一样有一点 BT，因为确实比较复杂，可以参考[官方文档](http://www.scala-sbt.org/)学习学习。

言归正传，进入 Play 的入口。先看下面的配置

    mainClass in (Compile, run) := Some("play.core.server.NettyServer"),

非常清楚的指明 Play 的启动类就是 play.core.server.NettyServer，此类位于 Play 的 Core 中。找到如下 main 入口函数：

    {% highlight scala %}
      def main(args: Array[String]) {
        args.headOption
          .orElse(Option(System.getProperty("user.dir")))
          .map { applicationPath =>
            val applicationFile = new File(applicationPath)
            if (!(applicationFile.exists && applicationFile.isDirectory)) {
              println("Bad application path: " + applicationPath)
            } else {
              createServer(applicationFile).getOrElse(System.exit(-1))
            }
          }.getOrElse {
            println("No application path supplied")
          }
      }
    {% endhighlight %}

此处代码就是为了创建 Netty Server，下面就是 createServer 的局部代码：

    {% highlight scala %}
    val server = new NettyServer(
        new StaticApplication(applicationPath),
        Option(System.getProperty("http.port")).fold(Option(9000))(p => if (p == "disabled") Option.empty[Int] else Option(Integer.parseInt(p))),
        Option(System.getProperty("https.port")).map(Integer.parseInt(_)),
        Option(System.getProperty("http.address")).getOrElse("0.0.0.0")
    )
    {% endhighlight %}
    
上述代码虽然普通，但阐述了两个非常重要的概念：

* 创建了 **StaticApplication** 实例。它是 **ApplicationProvider** 继承类，该类虽然简单，但实际上非常重要的，它加载了 **Application**， Application 是 Play 的一个核心概念，很多地方都会涉及到。

      {% highlight scala %}
      class StaticApplication(applicationPath: File) extends ApplicationProvider {
        
        val application = new DefaultApplication(applicationPath,     
          this.getClass.getClassLoader, None, Mode.Prod)
        
        Play.start(application)
        
        def get = Success(application)
        def path = applicationPath
      }
      {% endhighlight %}
      
    代码中的 Play.start(applicaiton) 就是启动了 Application。

* 启动 Netty 服务器。熟悉 Netty 的同学知道，Netty 采用管线的思想处理 Request 和 Response 的，而 Netty 已经默认支持处理 HTTP 请求了，因此 Play 只需在管线中嵌入自己的处理逻辑，那么就自然而然的进入了 Play。见下列代码：

      {% highlight scala %}
      newPipeline.addLast("decoder", new HttpRequestDecoder(maxInitialLineLength, maxHeaderSize, maxChunkSize))
      newPipeline.addLast("encoder", new HttpResponseEncoder())
      newPipeline.addLast("decompressor", new HttpContentDecompressor())
      newPipeline.addLast("http-pipelining", new HttpPipeliningHandler())
      newPipeline.addLast("handler", defaultUpStreamHandler)
      {% endhighlight %}

  其中 defaultUpStreamHandler 就是 **PlayDefaultUpstreamHandler**。它的核心功能就是实现 Netty 的 **messageReceived** 方法，这是个巨型函数，主要完成以下功能：

    * 将 HTTP 请求转化成 Play 的 RequestHeader 并交给 Handler 处理
    * 处理 HTTP 的 body 信息
    * 返回 Result 给浏览器

  这里不对实现做具体分析(涉及到 Play 高级部分，可以后续再分析)，主要还是阅读 Play 逻辑部分的代码。

## 接收请求

Play 把一个请求分为两部分：**RequestHeader** 和 **Body**，前者包含了 HTTP 请求的基本信息，后者就是请求的具体内容了。

程序启动后，就开始监听端口接收请求了。Netty 会把请求传递给 Play 的 **Application** 的 
**GlobalSettings**

    {% highlight scala %}
    trait Application {
        ……
        /**
        * `Dev`, `Prod` or `Test`
        */
        def mode: Mode.Mode
    
        def global: GlobalSettings
        def configuration: Configuration
        def plugins: Seq[Plugin]
        ……
    }
    {% endhighlight %}
    
每个 Application 都有 *global* 属性，可以在 application.conf 中指定，若没有指定，则使用默认的。其实所有的请求信息都会首先进入 **global.onRequestReceived** 方法。

    {% highlight scala %}
    def onRequestReceived(request: RequestHeader): (RequestHeader, Handler) = {
      onRouteRequest(request)
        .map {
          case handler: RequestTaggingHandler => (handler.tagRequest(request), handler)
          case handler => (request, handler)
        }
        .map {
          case (taggedRequest, handler) => (taggedRequest, doFilter(rh => handler)(taggedRequest))
        }
        .getOrElse {
          (request, Action.async(BodyParsers.parse.empty)(_ => this.onHandlerNotFound(request)))
        }
    }
    {% endhighlight %}

这个函数主要完成以下任务：

* 找到路由. onRouteRequest(request) 就是根据 request 找到对应的处理函数（Action）
* 调用 doFilter 执行具体的 Handler 函数

下面是 doFilter 函数

    {% highlight scala %}
    def doFilter(next: RequestHeader => Handler): (RequestHeader => Handler) = {
      (request: RequestHeader) =>
        {
          next(request) match {
            case action: EssentialAction => doFilter(action)
            case handler => handler
          }
        }
    }
    {% endhighlight %}

总结处理流程:

    Netty -> PlayDefaultUpstreamHandler -> Applicaiton -> GlobalSettings -> Action

注: Action 是 Handler 的继承类。
    
               
## 处理请求
进入 Handler 后就要开始处理请求了。Play 是一个异步 Web 框架，所有的调用都是异步的，因此在处理请求时与其他 Web 框架有一些不同。    

先看一下 Play 处理请求的流程，依旧非常的简单：

    RequestHeader -> Array[Byte] -> Result 

获取到 HTTP 的请求头信息（RequestHeader），再获取内容信息（Array[Byte]）,最后生成结果（Result）。

可以参考官方文档的详细解释：[HttpApi](http://www.playframework.com/documentation/2.2.0-RC1/HttpApi)

### 核心类 EssentialAction
    {% highlight scala %}
    trait EssentialAction extends (RequestHeader => Iteratee[Array[Byte], SimpleResult]) with Handler {
    
      /**
       * Returns itself, for better support in the routes file.
       *
       * @return itself
       */
      def apply() = this
    
    }
    {% endhighlight %}
    
所有 Action 都从 EssentialAction 继承, 从类名就可以看出其重要性了。用于在 controller 中实现的各种 GET、POST 方法，归根结底都是要生成 Action。

该 trait 很简单，就是实现了从 RequestHeader 到 Iteratee\[Array\[Byte]]，再到 SimpleResult 的转换过程, 其实它也是调用了 BodyParser 的方法完成这个转换的。看看下面代码就很清楚了：
  
    {% highlight scala %}
    def apply(rh: RequestHeader): Iteratee[Array[Byte], SimpleResult] = parser(rh).mapM {
    case Left(r) =>
      Play.logger.trace("Got direct result from the BodyParser: " + r)
      Future.successful(r)
    case Right(a) =>
      val request = Request(rh, a)
      Play.logger.trace("Invoking action with request: " + request)
      Play.maybeApplication.map { app =>
        play.utils.Threads.withContextClassLoader(app.classloader) {
          apply(request)
        }
      }.getOrElse(Future.successful(Results.InternalServerError))
    }(executionContext)
    {% endhighlight %}
  
* parser 就是 BodyParser（下面有介绍），parser 分析 Http Content 后返回具体的内容
* 分析后组装成 Request，记住这个等式：Request = RequestHeader + Body，代码中的变量 a 就是 Body 的内容
* 创建好了 Request 后就调用用户定义的 Action 代码了

EssentialAction 不是从一个类继承而来，而是从一个**函数**继承的，刚开始我也很奇怪，后来才知道这也是 [Scala] 的一种特性，如果是从方法继承的，则需要在其 apply 方法中传入入参，并返回出参，具体到 EssentialAction 类，就是传入 RequestHeader，返回 Iteratee\[Array\[Byte]]。

实际上 EssentialAction 就是一个方法类了, RequestHeader => Iteratee\[Array\[Byte], SimpleResult] 可以换成 Function1\[RequestHeader, Iteratee\[Array\[Byte], SimpleResult]]。

### BodyParser
以前说过一个完整的 HTTP 请求是由 Header 和 Body 组成的，通过分析 Header 和 Body 才能生成最终的 Result，因此这里需要一个 BodyParser，它就是来负责分析处理 Body。

EssentialAction 继承类 Action 中就有一个属性为 BodyParser。下面是 BodyParser 的声明：

    {% highlight scala %}
    trait BodyParser[+A] extends Function1[RequestHeader, Iteratee[Array[Byte], Either[SimpleResult, A]]]
    {% endhighlight %}
    
和 EssentialAction 类似，BodyParser 也是个方法类, 负责分析并获取 Body 内容。


### 核心类 ActionBuilder
顾名思义，ActionBuilder 就是负责创建 Action 实例的，当然必要条件是 RequestHeader 和 BodyParser 都已经准备好了。

整个思路就是这样的： ActionBuilder 的目的是创建 Action，而 Action 需要 RequestHeader 和 BodyParser，就需要创建后两者，而 RequestHeader 来自于 Netty 分解，而 BodyParser 由 Play 默认提供，当然用户可以自定义 BodyParser。

这里以其中一个方法为例(删除了一些代码)：

    {% highlight scala %}
    trait ActionBuilder {
      def apply(bodyParser: BodyParser)(block: Request => Result): Action = async(bodyParser) { req: Request =>
        block(req) match {
          case simple: SimpleResult => Future.successful(simple)
          case async: AsyncResult => async.unflatten
        }
      }
    
    final def async[A](bodyParser: BodyParser[A])(block: R[A] => Future[SimpleResult]): Action[A] = composeAction(new Action[A] {
      def parser = composeParser(bodyParser)
      def apply(request: Request[A]) = try {
        invokeBlock(request, block)
      } catch {
        // NotImplementedError is not caught by NonFatal, wrap it
        case e: NotImplementedError => throw new RuntimeException(e)
        // LinkageError is similarly harmless in Play Framework, since automatic reloading could easily trigger it
        case e: LinkageError => throw new RuntimeException(e)
      }
      override def executionContext = ActionBuilder.this.executionContext
    })
    }
    {% endhighlight %}
    
这个简化版的 ActionBuilder 传入参数有：

* BodyParser 用于分析 Body
* block 就是用户实现的方法，其输入参数就是 Request
* 这里没有 RequestHeader，是在执行 Action 时由 Play 当成参数传入给 Action，回想一下 onRequestReceived 方法的入参
* Request = RequestHeader + Body
* async 是真正调用的终极方法，最终创建好 Action 实例，然后在 Action 中执行用户的代码

例如下面就是一个例子：

    {% highlight scala %}
    def index = Action(parse.anyContent) { request =>
      Ok("Got request [" + request + "]")
    }
    {% endhighlight %}
    
* 这里 Action 就是 ActionBuilder 的继承类（object 对象），用于生成 Action
* parse.anyContent 是 BodyParser，对应 ActionBuilder 的 apply 方法中的第一个参数 bodyParser
* 方法里的代码就是对应于 apply 方法中的第二个参数 block，其入参就是 Request，再回忆一下 Request 是如何产生的。

### 串联起来
分析了这些类后，需要将它们串联起来：

* Netty 接收到 HTTP 请求，分解出 RequestHeader
* Play 调用 GlobalSettings 的 onRequestReceived(RequestHeader)
* onRequestReceived 查找 Router 找到对应的 Controller 的某个方法
* 这个方法是通过 ActionBuilder 创建 Action:
  * 传入的参数有 BodyParser (可选)和 Request => Result, 最终会转化成 Future\[SimpleResult]，后者就是用户自定义的实现
  * Action 接收 RequestHeader 入参，并调用 BodyParser 分析 HTTP Content，随后会生成 Request
  * Action 会执行用户自定义的实现，入参就是上述 Request，并返回 Future\[SimpleResult]     
* Router 找到这个 Action 后，就执行它，传入的参数就是 RequestHeader

执行 Action 后返回的 SimpleResult 干什么用的？接下来继续分析。

## 生成结果

上述都讲的是如何处理请求，在处理完请求后，就需要返回结果了。

    {% highlight scala %}
    sealed trait Result extends NotNull with WithHeaders[Result]
    case class SimpleResult(header: ResponseHeader, body: Enumerator[Array[Byte]],
          connection: HttpConnection.Connection = HttpConnection.KeepAlive) extends 
          PlainResult
    class Status(status: Int) extends SimpleResult(header = ResponseHeader(status), 
          body = Enumerator.empty, connection = HttpConnection.KeepAlive)
    {% endhighlight%}
    
在上面的分析中经常看到 Future\[SimpleResult]，就是这个定义。SimpleResult 中有一个 Enumerator 对象，它就是为了像客户端写入返回的内容的，就是异步写入内容。试想一下一个页面很大，不可能全部装进内容里，再一次返回，Play 采用异步写入的方式，不会浪费内存的。不过这块的分析超出了本文的范畴，所以 Play 最底层机制，以后再做分析吧。

为了简化使用，Play 也定义了一个友好类 Status，它的构造参数是一个整型，没错，就是 HTTP 的返回码，如 OK 就是 200

    val Ok = new Status(OK)

Status 类定义了一个 apply 方法：

    {% highlight scala %}
    /**
     * Set the result's content.
     *
     * @param content The content to send.
     */
    def apply[C](content: C)(implicit writeable: Writeable[C]): SimpleResult = {
      SimpleResult(
        ResponseHeader(status, writeable.contentType.map(ct => Map(CONTENT_TYPE -> ct)).getOrElse(Map.empty)),
        Enumerator(writeable.transform(content))
      )
    }
    {% endhighlight%}
    
这个也是 Scala 的一个特性：apply 方法可以直接在类实例调用，省略掉 apply。所以就有了下面的调用:

    Ok("Return")
    
除了 apply 方法外， Status 也提供了丰富的封装函数，如 sendFile、chunked、feed 等，这些方法都是为了生成另外一个 SimpleResult（心中默念 immutable 三次就会明白了）。

好了，SimpleResult 都已经生成好了，那么就剩下将它返回给调用者。SimpleResult 中包含了返回码、Headers 以及 Enumerator，前面两个很好理解，而 Enumerator 比较复杂，它把页面内容（也可能是文件）通过异步方式返回给调用者。

## 总结
从上述分析看出，Play 的处理流程还是很清晰与简单的，其实这简单的背后是复杂的机制。Play 是完全异步的 Web 架构，因此在处理流程上会比较复杂，只是在本文中并没有涉及这块（比较头大），等把上层理顺后再分析底层机制吧。
