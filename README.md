![img](https://github.com/Qasak/a-proxy/blob/master/notes/proxy%E5%9C%B0%E5%9D%80%E4%B8%8E%E7%AB%AF%E5%8F%A3.png)

![img](https://github.com/Qasak/a-proxy/blob/master/notes/socket%E6%8E%A5%E5%8F%A3-echo%E6%9C%8D%E5%8A%A1%E5%99%A8.png)

## Summary

+ 1：顺序代理
  + 非常适合嵌入样式的简单文本页面

+ 2：并发代理
  + 多线程

+ 3:缓存Web对象
  + 缓存单个对象，而不是整个页面
  + 使用LRU逐出策略
  + 缓存系统必须允许并发读取，同时保持一致性。并发性？共享资源？

## 第1部分：实现顺序`webproxy`

第一步是实现一个处理HTTP/1.0get请求的基本顺序proxy。其他请求类型（例如POST）是严格可选的。

启动时，proxy应该侦听端口号将被指定的传入连接

在命令行上。一旦建立了连接，你的proxy应该读取整个请求，并解析请求。它应该确定客户端是否发送了一个有效的HTTP请求；

如果是这样，它就可以建立自己与相应web服务器的连接，然后向客户端请求对象

明确规定。最后，proxy应该读取服务器的响应并将其转发给客户端

### HTTP/1.0 GET 请求

当最终用户输入诸如http://www.cmu.edu/hub/index.html进入地址

在web浏览器中，浏览器将向proxy发送一个HTTP请求，该请求以可能如下所示：

```c
GET http://www.cmu.edu/hub/index.html HTTP/1.1
```

在这种情况下，proxy至少应将请求解析为以下字段：主机名，`www.cmu.edu`

以及路径或查询及其后面的所有内容，`/hub/index.html `  这样，proxy可以确定它应该打开到的连接`www.cmu.edu`并发送一个自己的HTTP请求，以以下形式的一行开始：

```c
GET /hub/index.html HTTP/1.0
```

请注意，HTTP请求中的所有行都以回车符“\r”结尾，后跟新行“\n”。同样重要的是，每个HTTP请求都以一个空行终止：“\r\n”。

你应该注意到，在上面的示例中，web浏览器的请求行以HTTP/1.1结尾，而proxy的请求行以HTTP/1.0结尾。现代web浏览器将生成HTTP/1.1请求，但proxy应该处理这些请求并将其作为HTTP/1.0请求转发。

> HTTP 1.0/1.1区别
>
> 1. **缓存处理**，在HTTP1.0中主要使用header里的If-Modified-Since,Expires来做为缓存判断的标准，HTTP1.1则引入了更多的缓存控制策略例如Entity tag，If-Unmodified-Since, If-Match, If-None-Match等更多可供选择的缓存头来控制缓存策略。
> 2. **带宽优化及网络连接的使用**，HTTP1.0中，存在一些浪费带宽的现象，例如客户端只是需要某个对象的一部分，而服务器却将整个对象送过来了，并且不支持断点续传功能，HTTP1.1则在请求头引入了range头域，它允许只请求资源的某个部分，即返回码是206（Partial Content），这样就方便了开发者自由的选择以便于充分利用带宽和连接。
> 3. **错误通知的管理**，在HTTP1.1中新增了24个错误状态响应码，如409（Conflict）表示请求的资源与资源的当前状态发生冲突；410（Gone）表示服务器上的某个资源被永久性的删除。
> 4. **Host头处理**，在HTTP1.0中认为每台服务器都绑定一个唯一的IP地址，因此，请求消息中的URL并没有传递主机名（hostname）。但随着虚拟主机技术的发展，在一台物理服务器上可以存在多个虚拟主机（Multi-homed Web Servers），并且它们共享一个IP地址。HTTP1.1的请求消息和响应消息都应支持Host头域，且请求消息中如果没有Host头域会报告一个错误（400 Bad Request）。
> 5. **长连接**，HTTP 1.1支持持续连接（Persistent Connection）和请求的流水线（Pipelining）处理，在一个TCP连接上可以传送多个HTTP请求和响应，减少了建立和关闭连接的消耗和延迟，在HTTP1.1中默认开启Connection： keep-alive，一定程度上弥补了HTTP1.0每次请求都要创建连接的缺点。

重要的是要考虑到HTTP请求，即使只是HTTP/1.0 get请求的子集，也可能非常复杂。教科书描述了HTTP事务的某些细节，但是完整的HTTP/1.0规范应该参考`RFC1945`。理想情况下，根据`RFC1945`的相关部分，你的HTTP请求解析器将是完全健壮的，除了一个细节：虽然规范允许多行请求字段，但是你的proxy不需要正确地处理它们。当然，你的proxy绝不应因格式错误的请求而过早中止。

### 请求报头(request header)

lab中的重要请求报头是

`Host, User-Agent, Connection, Proxy-Connection`



+ 始终发送`Host`报头。虽然这种行为在技术上不受HTTP/1.0的许可规范，有必要诱使某些Web服务器做出合理的响应，尤其是那些使用虚拟主机。

  主机头描述终端服务器的主机名。例如，访问`http://www.cmu.edu/hub/index.html`，则proxy将发送以下报头：

  	`Host：www.cmu.edu`

  web浏览器可能会将自己的`Host`附加到HTTP请求中。如果是的话在这种情况下，proxy应该使用与浏览器相同的主机头。

+ 你*可以*选择始终发送以下`User-Agent`报头：

  `User-Agent：Mozilla/5.0 (X11; Linux x86_64; rv:10.0.3) Gecko/20120305 Firefox/10.0.3`

  *这个header是单行的*

`User-Agent`标识客户端（根据操作系统等参数和浏览器），web服务器经常使用标识信息来操作它们的内容

用telnet-style测试可以得到不同User-Agent的不同反馈

+ 始终发送以下`Connection` 报头：

  `Connection: close`

+ 始终发送以下`Proxy-Connection`报头：

  `Proxy-Connection: close`

  `Connection`和`Proxy-Connection`报头用于指定连接是否将在第一个请求/响应交换完成后保持活动状态。完全可以接受（和建议）让你的proxy为每个请求打开一个新的连接。为这些标头的值指定`close`会提醒web服务器，你的proxy打算在首次请求/响应交换后关闭连接

为了方便起见，在`proxy.c`中，所描述的`User-Agent`报头的值作为字符串常量提供给你。

最后，如果浏览器作为HTTP请求的一部分发送任何额外的请求头，那么proxy应该不加更改地转发它们。

### 端口号

lab中有两类重要的端口号：HTTP请求端口和proxy的侦听端口。

HTTP请求端口是HTTP请求的URL中的可选字段。也就是说，URL可以是如下形式

`http://www.cmu.edu:8080/hub/index.html`，在这种情况下，proxy应该连接到主机`www.cmu.edu`的端口8080上，而不是默认的HTTP端口，即端口80。无论URL中是否包含端口号，proxy都必须正常工作。

侦听端口是proxy应该侦听传入连接的端口。proxy应该接受一个命令行参数，指定proxy的侦听端口号。例如，使用以下命令，proxy应侦听端口15213上的连接：

```
`linux>./proxy 15213`
```

你可以选择任何非特权侦听端口（大于1024小于65536），只要它不被其他进程使用。因为每个proxy都必须使用一个唯一的侦听端口，而且许多人都会在每台机器上同时工作，脚本`port-for-user`可以帮助你选择自己的个人端口号。使用它可以根据你的用户ID生成端口号：

```
`linux>./port-user.pl qasak`

`qasak: 45806`
```

由`port-for-user`返回的端口p总是偶数。因此，如果你需要一个额外的端口号，比如对于`Tiny server`，你可以安全地使用端口p和p+1。

请不要随意选择端口。如果你这样做，你就冒着干扰其他用户的风险



## 第二部分：处理并发需求

一旦有了一个工作的顺序proxy，就应该将其更改为同时处理多个请求。

实现并发服务器的最简单方法是生成一个新线程来处理每个新连接请求。也可以进行其他设计

+ 请注意，线程应以分离模式运行，以避免内存泄漏。

+ CS:APP3e教科书中描述的`open_clientfd`和`open_listenfd`函数是基于现代的和协议无关的`getaddrinfo`函数，因此是线程安全的。



## 第三部分：缓存web对象

在实验的最后一部分，你将向proxy添加一个缓存，在内存中存储最近使用的Web对象。HTTP实际上定义了一个相当复杂的模型，通过这个模型，web服务器可以指示如何缓存它们所服务的对象，客户端可以指定应该如何代表它们使用缓存。但是，你的proxy将采用简化的方法。

当proxy从服务器接收到web对象时，它应该在将对象传输到客户端时将其缓存在内存中。如果另一个客户机从同一个服务器请求相同的对象，则你的proxy无需重新连接，只需重新发送缓存对象。

显然，如果你的proxy要缓存所请求的每个对象，它将需要无限量的内存。此外，由于某些web对象比其他对象大，因此可能会出现这样的情况：一个巨大的对象将消耗整个缓存，从而根本无法缓存其他对象。为了避免这些问题，proxy应该同时具有最大缓存大小和最大缓存对象大小。

### 最大缓存大小

proxy的整个缓存应具有以下最大大小：

```
`MAX_CACHE_SIZE = 1 MiB `
```

当计算缓存的大小时，proxy必须只计算用于存储实际web对象的字节数；

任何无关的字节，包括元数据，都应该被忽略。

### 最大对象大小

proxy应仅缓存不超过以下最大大小的web对象：

```
`MAX_OBJECT_SIZE = 100 KiB  `
```

为了方便起见，这两个大小限制都是作为`proxy.c`中的宏提供的。

实现正确缓存的最简单方法是为每个活动连接分配一个缓冲区，并在从服务器接收到数据时积累数据。如果缓冲区的大小超过了最大对象大小，则可以丢弃缓冲区。如果在超过最大对象大小之前读取了web服务器的全部响应，则可以缓存该对象。使用此方案，proxy将用于web对象的最大数据量如下所示，其中T是活动连接的最大数量：

```
`MAX_CACHE_SIZE + T * MAX_OBJECT_SIZE  `
```

### 驱逐政策

你的proxy缓存应该使用一个收回策略，该策略近似于最近最少使用（LRU）的收回策略。它不一定是严格的LRU，但它应该是一个相当接近的东西。请注意，读一个对象和写它都算作使用该对象。

### 鲁棒性

和往常一样，你必须交付一个对错误、甚至是错误或恶意输入具有健壮性的程序。

服务器通常是长时间运行的进程，webproxy也不例外。仔细考虑长时间运行的进程应该如何应对不同类型的错误。对于许多类型的错误，proxy立即退出显然是不合适的。

健壮性还意味着其他要求，包括对诸如分段错误和缺少内存泄漏和文件描述符泄漏之类的错误情况的抗毁性。

### 为什么是多线程缓存？

+ 顺序缓存会对并行代理造成瓶颈

+ 多个线程可以安全地读取缓存的内容
  + 在缓存中搜索正确的数据并返回
  + 两个线程可以从同一个缓存块中读取

+ 但是写内容呢？
  + 在另一个线程读取时覆盖块？
  + 两个线程写入同一个缓存块？



## 测试与debugging

除了简单的autograder，你将没有任何样本输入或测试程序来测试你的实现。你将不得不提出你自己的测试，甚至你自己的测试工具来帮助你调试你的代码和决定什么时候你有一个正确的实现。在现实世界中，这是一项很有价值的技能，在现实世界中，几乎不知道确切的操作条件，也常常无法获得参考解决方案。

幸运的是，有许多工具可以用来调试和测试proxy。一定要练习所有的代码路径并测试一组有代表性的输入，包括基本情况、典型情况和边缘情况。

### Tiny web server

你的讲义目录CS:APP Tiny web服务器的源代码。虽然没有`thttpd`强大，

在`CSAPP: tinyweb`服务器将很容易为你修改你认为合适的。这也是proxy代码的合理起点。它是驱动程序代码用来获取页面的服务器

### telnet

你可以使用telnet打开与proxy的连接并向其发送HTTP请求

### curl

你可以使用curl生成对任何服务器的HTTP请求，包括你自己的proxy。它是一个非常有用的调试工具。例如，如果你的proxy和Tiny都在本地机器上运行，Tiny监听端口15213，proxy监听端口15214，那么你可以使用以下curl命令通过proxy从Tiny请求页面：

> linux> curl -v --proxy http://localhost:15214 http://localhost:15213/home.html
> \* About to connect() to proxy localhost port 15214 (#0)
> \* Trying 127.0.0.1... connected
> \* Connected to localhost (127.0.0.1) port 15214 (#0)
> \> GET http://localhost:15213/home.html HTTP/1.1
> \> User-Agent: curl/7.19.7 (x86_64-redhat-linux-gnu)...
> \> Host: localhost:15213
> \> Accept: */*
> \> Proxy-Connection: Keep-Alive
> \> *
> HTTP 1.0, assume close after body
> < HTTP/1.0 200 OK
> < Server: Tiny Web Server
> < Content-length: 120
> < Content-type: text/html
> <
> <html>
>
> <head><title>test</title></head>
>
> <body>
> \<img align="middle" src="godzilla.gif">
> Dave O’Hallaron
> </body>
> </html>
> \* Closing connection #0  

### netcat

netcat，也称为nc，是一个多功能的网络实用程序。你可以像telnet一样使用netcat来打开与服务器的连接。因此，假设你的proxy使用端口12345在catshark上运行，你可以执行以下操作来手动测试proxy：

> sh> nc catshark.ics.cs.cmu.edu 12345
> GET http://www.cmu.edu/hub/index.html HTTP/1.0
> HTTP/1.1 200 OK
> ...  

除了能够连接到Web服务器之外，netcat还可以作为服务器本身来运行。使用以下命令，你可以将netcat作为监听端口12345的服务器运行：

```
`sh> nc -l 12345  `
```

一旦你设置了一个netcat服务器，你就可以通过你的proxy生成一个对它的虚假对象(phony object)的请求，你就可以检查你的proxy发送给netcat的确切请求。

### 浏览器

最后，你应该使用最新版本的Mozilla Firefox测试proxy。访问`About Firefox`会自动将你的浏览器更新到最新版本。

![img](https://github.com/Qasak/a-proxy/blob/master/notes/firefox%E4%BB%A3%E7%90%86%E8%AE%BE%E7%BD%AE.png)



要配置Firefox以使用proxy，请访问

```
`Preferences>Advanced>Network>Settings  `
```

看到你的proxy通过一个真正的网络浏览器工作是非常令人兴奋的。虽然proxy的功能会受到限制，但你会注意到你可以通过proxy浏览绝大多数网站。

一个重要的警告是，在使用Web浏览器测试缓存时必须非常小心。所有现代Web浏览器都有自己的缓存，在尝试测试proxy缓存之前，应该禁用缓存。

## 提示

+ 如教科书第10.11节所述，使用标准I/O函数进行套接字输入和输出是一个问题。相反，我们建议你使用`robust I/O(RIO)`包，它在讲义目录的`csapp.c`文件中提供。

  > https://github.com/Qasak/a-proxy/blob/master/notes/UNIX%20IO-%E6%A0%87%E5%87%86IO-RIO.png
  >
  > 
  >
  > 

+ csapp.c中提供的错误处理函数不适合你的proxy，因为一旦服务器开始接受连接，它就不应该终止。你需要修改它们或者自己写。

+ 你可以随意修改讲义目录中的文件。例如，为了实现良好的模块化，你可以在名为cache.c和cache.h的文件中实现缓存功能，当然，添加新文件将需要你更新提供的Makefile。

+ 正如CS:APP3e文本第964页的旁白所讨论的，你的proxy必须忽略SIGPIPE信号，并且应该优雅地处理返回EPIPE错误的`write`操作。

+ 有时，调用`read`从已提前关闭的套接字接收字节将导致`read`返回-1，`errno`设置为`ECONNRESET`。你的proxy不应因此而终止

+ 请记住，并非web上的所有内容都是ASCII文本。网络上的很多内容都是二进制数据，比如图像和视频。确保在选择和使用网络I/O函数时考虑二进制数据

+ 将所有请求作为HTTP/1.0转发，即使原始请求是HTTP/1.1。