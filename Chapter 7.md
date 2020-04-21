# 网络编程

---

- [7.1 构建 TCP 服务](#71-构建-tcp-服务)
- [7.2 构建 UDP 服务](#72-构建-udp-服务)
- [7.3 构建 HTTP 服务](#73-构建-http-服务)
- [7.4 构建 WebSocket 服务](#74-构建-websocket-服务)
- [7.5 网络服务和安全](#75-网络服务和安全)
- [7.6 总结](#76-总结)

---

Node 是一个面向网络而生的平台，它具有事件驱动、无阻塞、单线程等特性，具备良好的可伸缩性，使得它十分清凉，适合在分布式网络中扮演各种各样的角色。同时 Node 提供的 API 十分贴合网络，适合用它基础的 API 构建灵活的网络服务。从本章其，我们将介绍 Node 在网络服务器方面的具体能力。

利用 Node 可以十分方便的搭建网络服务器。在 Web 领域，大多数的编程语言需要专门的 Web 服务器作为容器，如 ASP、ASP.NET 需要 IIS 作为服务器，PHP 需要搭载 Apache 或 Nginx 环境等，JSP 需要 Tomcat 服务器等。但对于 Node 而言，只需要几行代码即可构建服务器，无需额外的容器。

Node 提供了 net、dgram、http、https 这 4 个模块，分别用于处理 TCP、UDP、HTTP、HTTPS，适用于服务器端和客户端。

## 7.1 构建 TCP 服务

TCP 服务在网络应用中十分常见，目前大多数应用都是基于 TCP 搭建而成的。

### 7.1.1 TCP

TCP 全名为传输控制协议，在 OSI 模型（由七层组成，分别为物理层、数据链路层、网络层、传输层、会话层、表示层、应用层）中输入传输层协议。许多应用层协议都是基于 TCP 构建，典型的 HTTP、SMTP、IMAP 等协议。

TCP 是面向连接的协议，其显著的特征是在传输之前需要 3 次握手形成会话。

只有会话形成之后，服务器端和客户端之间才能互相发送数据。在创建会话的过程中，服务器端和客户端分别提供了一个套接字，这两个套接字共同形成一个连接。服务器端与客户端则通过套接字实现两者之间连接的操作。

### 7.1.2 创建 TCP 服务器端

在基本了解 TCP 的工作原理之后，我们可以开始创建一个 TCP 服务器端来接收网络请求，代码如下：

```js
var net = require('net');

var server = net.createServer(function (socket) {
  // 新连接
  soket.on('data', function (data) {
    socket.write('您好！');
  });

  soket.on('end', function () {
    console.log('断开连接');
  });

  socket.write('欢迎光临《深入浅出node.js》示例：\n');
});

server.listen(8124, function () {
  console.log('server bound');
});
```

我们通过`net.createServer(listener)`即可创建一个 TCP 服务器，listener 是连接事件 connection 的监听器，也可以采用如下方式进行侦听：

```js
var server = net.createServer();

server.on('connect', function (socket) {
  // 新连接
});
server.listen(8124);
```

我们可以利用 Telnet 工具作为客户端对刚才创建的简单服务器进行会话交流：

```shell
$ telnet 127.0.0.1 8124
Trying 127.0.0.1...
Connected to localhost.
Escap character is '^]'.
欢迎光临《深入浅出node.js》示例：
hi
您好!
```

除了端口外，我们同样也可以对 Domain Socket 进行监听，代码：

```js
server.listen('/tmp/echo.sock');
```

通过 nc 工具进行会话，测试上面构建的 TCP 服务的代码：

```shell
$ nc -U /tmp/echo.sock
...
```

通过 net 模块自行构造客户端进行会话，测试上面构建的 tcp 服务的代码：

```js
var net = require('net');
var client = net.connect({ port: 8124 }, function () {
  console.log('client connected');
  client.write('world!\r\n');
});

client.on('data', function (data) {
  console.log(data.toString());
  client.end();
});

client.on('end', function () {
  console.log('client disconnected');
});
```

将以上客户端代码存为 client.js，并执行。其结果与使用 Telnet 或 nc 的会话结果并无差别。如果是 Domain Socket，在填写选项时，填写 path 即可：

```js
var client = net.connect({ path: '/tmp/echo.sock' });
```

### 7.1.3 TCP 服务的事件

在上述的示例中，代码分为服务器事件和连接事件。

#### 1. 服务器事件

对于通过`net.createServer()`创建的服务器而言，它时一个 EventEmitter 实例，它的自定义事件有如下几种：

- listening: 在调用 `server.listen()`绑定端口或者 Domain Socket 后触发，简洁写法为`server.listen(port, listeningListener)`，通过`listen()`方法的第二个参数传入。

- connection：每个客户端套接字连接到服务器端时触发，简洁写法通过`net.createServer()`，最后一个参数传入。

- close：当服务器关闭时触发，在调用`server.close()`后，服务器将停止接收新的套接字连接，但保持当前存在的连接，等待所有连接都断开后，会触发该事件。

- error：当服务器发生异常时，将会触发该事件。比如侦听一个使用中的端口，将会触发一个异常，如果不侦听 error 事件，服务器将会抛出异常。

#### 2. 连接事件

服务器可以同时与多个客户端保持连接，对于每个连接而言，是典型的可写可读 Stream 对象。Stream 对象可以用于服务器端和客户端之间的通信，既可以通过 data 事件从一端读取另一端发来的数据，也可以通过`write()`方法从一段向另一端发送数据。它具有如下自定义事件：

- data：当一端调用`write()`发送数据时，另一端会触发 data 事件，事件传递的数据即是`write()`发送的数据。

- end：当连接中的任意一端发送了 FIN 数据时，将会触发该事件。

- connect：该事件用于客户端，当套接字与服务器端连接成功时会被触发。

- drain：当任意一端调用 write()发送数据时，当前这段会触发该事件。

- error：当异常发生时，触发该事件。

- close：当套接字完全关闭时，触发该事件。

- timeout：当一定时间后连接不再活跃时，该事件将会被触发，通知用户该链接已经被闲置了。

另外，由于 TCP 套接字是可写可读的 Stream 对象，可以利用`pipe()`方法巧妙地实现管道操作，如下代码实现了一个 echo 服务器：

```js
var net = require('net');

var server = net.createServer(function (socket) {
  socket.write('Echo Server \r\n');
  socke.pipe(socket);
});

server.listen(1337, '127.0.0.1');
```

值得注意的是，TCP 针对网络的小数据包有一定的优化策略：Nagle 算法。如果每次只发送一个字节的内容而不优化，网络中将充满只有极少数有效数据的数据包，将十分浪费网络资源。Nagle 算法针对这种情况，要求缓冲区的数据达到一定数量或者一定时间后才将其发出，所以小数据包将会被 Nagle 算法合并，以此来优化网络。这种优化虽然使网络带宽被有效地使用，但是数据有可能被延迟发送。

在 Node 中，由于 TCP 默认启用了 Nagle 算法，可以调用 socket.setNodelay(true)去掉 Nagle 算法，使得 write()可以立即发送数据到网络中。

另一个需要注意的是，尽管在网络的一端调用`write()`会触发另一端的 data 事件，但是并不意味着每次`write()`都会触发一次 data 事件，在关闭掉 Nagle 算法后，另一端可能会将接收到的多个小数据包合并，然后值触发一次 data 事件。

## 7.2 构建 UDP 服务

UDP 又称为用户数据包协议，与 TCP 一样同属于与网络传输层。UDP 与 TCP 最大的不同就是 UDP 不是面向连接的。TCP 中连接一旦建立，所有的会话都基于连接完成，客户端如果要与另一个 TCP 服务通信，需要另外创建一个套接字来完成连接。但在 UDP 中，一个套接字可以与多个 UDP 服务通信，它虽然提供面向事务的简单不可靠信息传输服务，在网络差的情况下存在丢包严重的问题，但是由于它无须连接，资源消耗低，处理快速且灵活，所以常常应用在那种偶尔丢一两个数据包也不会产生重大影响的场景，比如音频、视频等。UDP 目前应用很广泛，DNS 服务即是基于它实现的。

### 7.2.1 创建 UDP 套接字

创建 UDP 套接字十分简单，UDP 套接字一旦创建，既可以作为客户端发送数据，也可以作为服务器端接收数据。下面的代码创建一个 UDP 套接字：

```js
var dgram = require('dgram');

var socket = dgram.createSocket('udp4');
```

### 7.2.2 创建 UDP 服务器端

若要让 UDP 套接字接收网络消息，只要调用`dgram.bind(port[, address])`方法对网卡和端口进行绑定即可。以下为一个完整的服务器端示例：

```js
var dgram = require('dgram');

var server = dgram.createSocket('udp4');

server.on('message', function (msg, rinfo) {
  console.log('server got: ' + msg + 'from ' + rinfo.address + ':' + rinfo.port);
});

server.on('listening', function () {
  var address = server.address();
  console.log('server listening ' + address.address + ':' + address.port);
});

server.bind(41234);

// 该套接字将接收所有网卡上41234端口上的消息。在绑定完成后，将触发listening事件。
```

### 7.2.3 创建 UDP 客户端

接下来我们创建一个客户端与服务器端进行对话，代码如下：

```js
var dgram = require('dgram');

var message = new Buffer('深入浅出Node.js');
var client = dgram.createSocket('udp4');
client.send(message, 0, message.length, 41234, 'localhost', function (err, bytes) {
  client.close();
});
// 保存为client.js并执行
```

当套接字对象在客户端时，可以调用`send()`方法发送消息到网络中。`send()`方法的参数如下：

```js
socket.send(buf, offset, length, port, address[, callback]);
```

这些参数分别为要发送的 Buffer、Buffer 的偏移、Buffer 的长度、目标端口、目标地址、发送完成后的回调。与 TCP 套接字的`write()`相比，`send()`方法的参数列表相对复杂，但是它更灵活的地方在于可以随意发送数据到网络中的服务器端，而 TCP 如果要发送数据到另一个服务器端，则需要重新通过套接字构造新的连接。

### 7.2.4 UDP 套接字事件

UDP 套接字相对于 TCP 套接字使用起来更简单，它只是一个 EventEmitter 的实例，而非 Stream 的实例。它具备如下自定义事件：

- message：当 UDP 套接字侦听网卡端口后，接收到消息时触发该事件，触发携带的数据为消息 Buffer 对象和一个远程地址信息。

- listening：当 UDP 套接字开始侦听时，触发该事件。

- close：调用 close()方法时触发该事件，并不在触发 message 事件。如需再次触发 message 事件，重新绑定即可。

- error：当异常发生时触发该事件，如果不震霆，异常将直接抛出，使进程退出。

## 7.3 构建 HTTP 服务

TCP 与 UDP 都属于网络传输层协议，如果要构造高效的网络应用，就应该从传输层进行着手。但是对于经典的应用场景，则无需从传输层协议入手构造自己的应用，比如 HTTP 或 SMTP 等，这些经典的应用协议对于普通应用而言绰绰有余。Node 提供了基本的 http 和 https 模块用于 HTTP 和 HTTPS 的封装，对于其它应用层协议的封装，也能从社区轻松找到其实现。

在 Node 中构建 HTTP 服务极其容易，Node 官网上的经典例子就展示了如何用寥寥几行代码实现一个 HTTP 服务器，代码如下：

```js
var http = require('http');
http
  .createServer(function (req, res) {
    res.writeHead(200, { 'Content-Type': 'text/plain' });
    res.end('Hello World!\n');
  })
  .listen(1337, function () {
    console.log('Server running at http://127.0.0.1:1337/');
  });
```

尽管这个 HTTP 服务器简单到只能回复 Hello World，但是它能位置的并发量和 QPS 都是不容小觑的，其背后的原因在第三章中有叙述，此处我们不再探讨。这里我们抛开性能，只对其 HTTP 服务在应用层的实现原理进行展开、讨论和研究。

### 7.3.1 HTTP

#### 1.初识 HTTP

HTTP 的全称时超文本传输协议，英文写作 HyperText Transfer Protocol。欲了解 Web，先了解 HTTP 将会极大地提高我们对 Web 地认知。HTTP 构建在 TCP 之上，属于应用层协议。在 HTTP 地两端是服务器和浏览器，即著名的 B/S 模式，如今精彩纷呈的 Web 即是 HTTP 应用。

HTTP 得以发展是 W3C 和 IETF 两个组织合作的结果，它们最终发布了一系列 RFC 标准，目前最知名的 HTTP 标准为 RFC 2616。

#### 2.HTTP 报文

为了详细解释 HTTP 的报文，在启动上述服务器端代码后，我们对经典示例代码进行一次报文的获取，这里采用的工具是 curl，通过`-v`选项，可以显示这次网络通信的所有报文信息。

```shell
λ curl -v http://127.0.0.1:1337/
*   Trying 127.0.0.1...
* TCP_NODELAY set
* Connected to 127.0.0.1 (127.0.0.1) port 1337 (#0)
> GET / HTTP/1.1
> Host: 127.0.0.1:1337
> User-Agent: curl/7.55.1
> Accept: */*
>
< HTTP/1.1 200 OK
< Content-Type: text/plain
< Date: Tue, 21 Apr 2020 09:13:39 GMT
< Connection: keep-alive
< Transfer-Encoding: chunked
<
Hello World
* Connection #0 to host 127.0.0.1 left intact
```

从上述信息中我们可以看到这次网络通信的报文信息分为几个部分，第一部分内容为经典的 TCP 的 3 次握手过程：

```shell
*   Trying 127.0.0.1...
* TCP_NODELAY set
* Connected to 127.0.0.1 (127.0.0.1) port 1337 (#0)
```

第二部分是在完成握手之后，客户端向服务器端发送请求报文，如下所示：

```shell
> GET / HTTP/1.1
> Host: 127.0.0.1:1337
> User-Agent: curl/7.55.1
> Accept: */*
>
```

第三部分是服务器端完成处理后，向客户端发送相应内容，包括响应头和响应体：

```shell
< HTTP/1.1 200 OK
< Content-Type: text/plain
< Date: Tue, 21 Apr 2020 09:13:39 GMT
< Connection: keep-alive
< Transfer-Encoding: chunked
<
Hello World
```

最后部分是结束会话的信息：

```shell
* Connection #0 to host 127.0.0.1 left intact
```

从上述报文信息中可以看出 HTTP 的特点，它是基于请求响应式的，以一问一答的方式实现服务，虽然基于 TCP 会话，但是本身却无会话的特点。从协议的角度来说，现在的应用，如浏览器，其实是一个 HTTP 的代理，用户的行为会通过它转化成 HTTP 请求报文发送给服务器端，服务器端在处理请求后，发送响应报文给代理，代理在解析报文后，将用户需要的内容呈现在界面上。以浏览器打开图片地址为例：首先，浏览器构造 HTTP 报文发向图片服务器端；然后，服务器端判断报文中的要请求的地址，将磁盘中的图片文件以报文的形式发送给浏览器；浏览器接受完图片后，调用渲染引擎将其显示给用户。简而言之，HTTP 服务只做两件事，处理 HTTP 请求和发送 HTTP 响应。

无论是 HTTP 请求报文还是 HTTP 响应问问，报文内容都包含两个部分：报文头和报文体。
上文的报文代码中`>`和`<`部分属于报文的头部，由于是 GET 请求，请求报文中没有包含报文体，响应报文中的 Hello World 即是报文体。

### 7.3.2 http 模块

Node 的 http 模块包含对 HTTP 处理的封装。在 Node 中，HTTP 服务继承自 TCP 服务器（net 模块），它能够与多个客户端保持连接，由于其采用事件驱动的形式，并不为每一个连接创建额外的线程或进程，保持很低的内存占用，所以能实现高并发。HTTP 服务于 TCP 服务模型有区别的地方在于，在开启 keep-alive 之后，一个 TCP 会话可以用于多次请求和响应。TCP 服务以 connection 为单位进行服务，HTTP 服务以 request 为单位进行服务。http 模块即是将 connection 到 request 的过程进行了封装。

除此之外，http 模块将连接所用套接字的读写抽象为 ServerRequest 和 ServerResponse 对象，它们分别对应请求和响应操作。在请求产生的过程中，http 模块拿到连接中传来的数据，调用二进制模块 http_parser 进行解析，在解析完请求报文的报头后，触发 request 事件，调用用户的业务逻辑。

#### 1. HTTP 请求

对于 TCP 连接的读操作，http 模块将其封装成 ServerRequest 对象。让我们再次查看前面的请求报文，报文头部将会通过 http_parser 进行解析。

报文头第一行`GET / HTTP/1.1`被解析之后分解为如下属性：

- req.method 属性：值为 GET，是为请求方法，常见的请求方法有 GET、POST、DELETE、PUT、CONNECT 等几种。

- req.url 属性：值为`/`。

- req.httpVersion 属性：值为`1.1`。

其余报头是很规律的`key: value`格式，被解析后放置在`req.headers`属性上传递给业务逻辑以供调用。

报文体部分则抽象为一个只读流对象，如果业务逻辑需要读取报文体中的数据，则要在这个数据流结束后才能进行操作：

```js
function (req, res) {
  var buffers = [];
  req.on('data', function (trunk) {
    buffers.push(trunk);
  }).on('end',function () {
    var buffer = Buffer.concat(buffers);
    // TODO
    res.end('Hello world');
  });
}
```

HTTP 请求对象和 HTTP 响应对象是相对较底层的封装，现行的 Web 框架如 Connect 和 Express 都是在这两个对象的基础上进行高层封装完成的。

#### 2. HTTP 响应

再来看 HTTP 响应对象。HTTP 响应相对简单一些，它封装了对底层连接的写操作，可以将其看成一个可写的流对象。它影响响应报文头部信息的 API 为`res.setHeader()`和`res.writeHead()`。

```js
res.writeHead(200, { 'Content-Type': 'text/plain' });
```

其分为`setHeader()`和`writeHead()`两个步骤。它在 http 模块的封装下，实际生成如下报文：

```shell
< HTTP/1.1 200 OK
< Content-Type: text/plain
```

我们可以调用`setHeader`进行多次设置，但只有调用`writeHead`后，报头才会写入到连接中。除此之外，http 模块会自动帮你设置一些头信息。

报文体部分则是调用`res.write()`和`res.end()`方法实现，后者与前者的差别在于`res.end()`会先调用`write()`发送数据，然后发送信号告知服务器这次响应结束。

响应结束后，HTTP 服务器可能会将当前的连接用于下一个请求，或者关闭请求。值得注意的是，报头是在报文体发送前发送的，一旦开始了数据的发送，`writeHead()`和`setHeader()`将不再生效。这由协议的特性决定。

另外，无论服务器端在处理业务逻辑时是否发生异常，务必在结束时调用`res.end()`结束请求，否则客户端将一直处于等待的状态。当然，也可以通过延迟`res.end()`的方式实现客户端与服务器端之间的长连接，但结束时无比关闭连接。

### 3.HTTP 服务的事件

如同 TCP 服务一样，HTTP 服务器也抽象了一些事件，以供应用层使用，同样典型的是，服务器也是一个 EventEmitter 实例。

- connection 事件： 在开始 HTTp 请求和响应前，客户端与服务器端需要建立底层 TCP 连接，这个连接可能因为开启了`keep-alive`，可以在多次请求响应之间使用；当这个连接建立时，服务器触发一次 connnection 事件。

- request 事件：建立 TCP 连接后，http 模块底层将在数据流中抽象出 HTTP 请求和 HTTP 响应，当请求数据发送到服务器端，在解析出 HTTP 请求头后，将会触发该事件；在`res.end()`后，TCP 连接可能将用于下一次请求响应。

- close 事件：与 TCP 服务器的行为一致，调用`server.close()`方法停止接受新的连接，当已有的连接都断开时，触发该事件；可以给 server.close()传递一个回调函数来快速注册该事件。

- checkContinue 事件：某些客户端在发送较大数据时，并不会将数据直接发送，而是先发送一个头部带`Expect: 100-continue`的请求到服务器，服务器将会触发 checkContinue 事件；如果没有为服务器监听该事件，服务器将会自动响应客户端 100 Continue 的状态码，表示接受数据上传；如果不接受数据的较多时，响应客户端 400 Bad Request 拒绝客户端继续发送数据即可。需要注意的是，当该事件发送时，不会触发 request 事件，两个事件之间相互排斥。当客户端收到 100 Continue 后重新发起请求时，才会触发 request 事件。

- connect 事件：当客户端发起 CONNECT 请求时触发，而发起 CONNECT 请求通常在 HTTP 代理时出现；如果不见听该事件，发起该请求的连接将会关闭。

- upgrade 事件：当客户端要求升级连接的协议时，需要和服务器端协商，客户端会在请求头中带上 Upgrade 字段，服务器端会在接收到这样的请求时触发该事件。这在后文的 WebSocket 部分由详细流程介绍。如果不见听该事件，发起该请求的连接会关闭。

- clientError 事件：连接客户端触发 error 事件时，这个错误会传递到服务器端，此时触发该事件。

### 7.3.3 HTTP 客户端

在对服务器端的实现进行了描述后，HTTP 客户端的原理几乎不用再描述，因为它就是服务器端服务模型的另一部分，处于 HTTP 的另一端，在整个报文的参与中，报文头和报文体由它产生。同时 http 模块提供了一个底层 API：`http.request(options, connect)`，用于构造 HTTP 客户端。

下面的示例与上文的 curl 命令大致相同：

```js
var optioons = {
  hostname: '127.0.0.1',
  port: 1334,
  path: '/',
  method: 'GET',
};

var req = http.request(options, function (res) {
  console.log('Status: ' + res.statusCode);
  console.log('Headers: ' + JSON.stringify(res.headers));
  res.setEncoding('utf8');
  res.on('data', function (chunk) {
    console.log(chunk);
  });
});
req.end();
```

其中，options 参数决定了这个 HTTP 请求头中的内容，它的选项有如下：

- host：服务器的域名或 ip 地址，默认为 localhost。
- hostname：服务器名称。
- port：服务器端口，默认为 80。
- localAdress：建立网络连接的本地网卡
- socketPath：Domain 套接字路径。
- method：HTTP 请求方法，默认为 GET。
- path：请求路径，默认为`/`。
- headers：请求头对象。
- auth：Basic 认证，这个值将被计算成请求头中的 Authorization 部分。

报文体的内容由请求对象的`write()`和`end()`方法实现：通过`write()`方法向连接中写入数据，通过 end()方法告知报文结束。它与浏览器的 Ajax 调用几近相同，Ajax 的实质就是一个异步的网络 HTTP 请求。

#### 1. HTTP 响应

HTTP 客户端的响应对象和服务器端较为类似，在 ClientRequest 对象中，它的事件叫做 response。ClientRequest 在解析响应报文时，一解析完响应头就触发 response 事件，同时传递一个响应对象以供操作 ClientResponse。后续响应报文提以只读流的方式提供。

由于从响应读取数据与服务器端 ServerRequest 读取数据的行为较为类似，此处不再赘述。

#### 2. HTTP 代理

如同服务器端的实现一般，http 提供的 ClientRequest 对象也是基于 TCP 层实现的，在 keep-alive 的情况下，一个底层会话连接可以多次用于请求。为了重用 TCP 连接，http 模块包含一个默认的客户端代理对象`http.globalAgent`。它对每个服务器端（host+port）创建的连接进行了管理，默认情况下，通过 ClientRequest 对象对同一个服务器端发起 HTTP 请求最多可以创建 5 个连接。它的实质是一个连接池。

调用 HTTP 客户端同时对一个服务器发起 10 次 HTTP 请求时，其实质只有 5 个请求处于并发状态，后续的请求需要等待某个请求完成后才真正发出。这与浏览器对同一个域名有下载连接的限制时相同的行为。

如果你在服务器端通过 ClientRequest 调用网络中的其它 HTTP 服务，记得关注代理对象对网络请求的限制。一旦请求量过大，连接限制将会限制服务性能。如需癌变，可以在 options 中传递 agent 选项。默认情况下，请求会采用全局的代理对象，默认连接数量限制为 5。我们可以自行构造代理对象：

```js
var agent = new http.Agent({ maxSockets: 10 });
var options = {
  hostname: '127.0.0.1',
  port: 1334,
  path: '/',
  method: 'GET',
  agent: agent,
};
```

也可以设置 agent 选项为 false，以脱离连接池的管理，使得请求不受并发的限制。

Agent 对象的 sockets 和 requests 属性分别表示当前连接池中的连接数和处于等待状态的请求书，在业务中监视这两个值有助于发现业务状态的繁忙程度。

#### 3.HTTP 客户端事件

与服务器端对应，HTTP 客户端也有响应的事件。

- response：与服务器端的 request 事件对应的客户端在请求发出后得到服务器端响应时，触发该事件。

- socket：当底层连接池中建立的连接分配给当前请求对象时，触发该事件。

- connect：当客户端向服务器端发起 CONNECT 请求时，如果服务器端响应了 200 状态码，客户端将触发此事件。

- upgrade：客户端向服务器端发起 Upgrade 请求时，如果服务器端响应了 101 Switching Protocols，状态，客户端将会触发该事件。

- continue：客户端向服务器端发起`Expect: 100-continue`头信息，以视图发送给较大数据量，如果服务器端响应 100 Continue 状态，客户端将触发该事件。

## 7.4 构建 WebSocket 服务

提到 Node，不能错过的时 WebSocket 协议。它与 Node 之间的配合堪称完美，其理由有两条。

- WebSocket 客户端基于事件的编程模型与 Node 中自定义事件相差无几。
- WebSocket 实现了客户端与服务器之间的长连接，而 Node 事件驱动的方式十分擅长于大量的客户端保持高并发连接。

除此之外，WebSocket 与传统 HTTP 有如下好处：

- 客户端与服务器端只需要建立一个 TCP 连接，可以使用更少的连接。
- WebSocket 服务器端可以推送数据到客户端，这远比 HTTP 请求响应模型更灵活、更有效。
- 有更轻量级的协议头，减少数据传送量。

WebSocket 最早时作为 HTML5 重要特性而出现的，最终在 W3C 和 IETF 的推动下，形成 RFC 6455 规范。现代浏览器大多都支持 WebSocket 协议，接下来我们用一端代码来展现 WebSocket 在客户端的应用示例：

```js
var socket = new WebSocket('ws://127.0.0.1:12010/updates');
socket.onopen = function () {
  setInterval(function () {
    if (socket.bufferedAmount == 0) socket.send(getUpdateData());
  }, 50);
};

socket.onmessage = function () {
  // TODO event.data
};
```

上述代码中，浏览器与服务器端创建 WebSocket 协议请求，在请求完成后里连接打开，每 50ms 向客户端发送一次数据，同时可以通过 onmessage()方法接收服务器端传来的数据。这行为与 TCP 客户端十分相似，相较于 HTTP，它能够双向通信。浏览器一旦能够使用 WebSocket，可以想象应用的使用空间极大。

在 WebSocket 之前，网页客户端与服务器端进行通信最高效的是 Comet 技术。实现 Comet 基数的细节时采用长轮询或 iframe 流。长轮询的原理是客户端向服务器端发起请求，服务器端只在超时或有数据响应时断开连接（res.end()）；客户端在收到数据或者超时后重新发起请求。这个请求行为拖着长长的尾巴，是故用 Comet（彗星）来命名它。

使用 WebSocket 的话，网页客户端只需一个 TCP 连接即可完成双向通信，在服务器端与客户端频繁通信时，无需频繁断开连接和重发请求。连接可以得到高效应用，编程模型也十分简洁。

前文也或多或少提到了 WebSocket 与 HTTP 的区别，相比 HTTp，WebSocket 更接近于传输层协议，它并没有在 HTTP 的基础上模拟服务器端的推送，而是在 TCP 上定义独立的协议。让人迷惑的部分在于 WebSocket 的握手部分是由 HTTP 完成的，使人觉得它可能时基于 HTTP 实现的。

WebSocket 协议主要分为两个部分：握手和数据传输。下面我们来详细说一说这两个部分。

### 7.4.1 WebSocket 握手

客户端建立连接时，通过 HTTP 发起请求报文：

```shell
GET /chat HTTP/1.1
Host: server.example.com
Upgrade: websocket
Connect: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Protocol: chat, superchat
Sec-WebSocket-Version： 13
```

与普通的 HTTp 请求协议略有区别的部分在于如下这些协议头：

```shell
Upgrade: websocket
Connection: Upgrade
```

上述这两个字段表示请求服务器端升级协议为 WebSocket。其中 Sec-WebSocket-Key 用于安全校验。Sec-WebSocket-Key 的值是随机生成的 Base64 编码的字符串。服务器端接收到之后将其与字符串 258EAFA5-E914-47DA-95CA-C5ABODC85B11 相连，然后通过 sha1 安全散列算法计算出结果后，再进行 Base64 编码，最后返回给客户端。算法如下：

```js
var crypto = require('crypto');
var val = crypto.createHash('sha1').update(key).digest('base64');
```

另外，下面两个字段指定子协议和版本号：

```shell
Sec-WebSocket-Protocol: chat, superchat
Sec-WebSocket-Version: 13
```

服务器端在处理完请求后，响应如下报文：

```shell
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbk+xOo=
Sec-WebSocket-Protocol: chat
```

上面的报文告之客户端正在更换协议，更新应用层协议为 WebSocket 协议，并在当前的套接字连接上应用新的协议。剩余的字段分别表示服务器端基于 Sec-WebSocket-Key 生成的字符串和选中的子协议。客户端将会校验 Sec-WebSocket-Accept 的值，如果成功，将开始接下来的数据传输。

我们在 Node 模拟浏览器发起协议切换的行为：

```js
var WebSocket = function (url) {
  // 伪代码，解析 ws://127.0.0.1:12010/updates，用于请求
  this.options = parseUrl(url);
  this.connect();
};

WebSocket.prototype.setSocket = function (socket) {
  this.socket = socket;
};

WebSocket.prototype.connect = function () {
  var this = that;
  var key = new Buffer(this.options.protocolVersion + '-' + Date.now()).toString('base64');
  var shasum = crypto.createHash('sha1');

  var expected = shasum.update(key + '258EAFA5-E914-47DA-95CA-C5ABODC85B11').disget('base64');

  var options = {
    port: this.options.port, //12010
    host: this.options.hostname, // 127.0.0.1
    headers: {
      Connection: 'Upgrade',
      Upgrade: 'websocket',
      'Sec-WebSocket-Version': this.options.protocolVersion,
      'Sec-WebSocket-Key': key,
    },
  };

  var req = http.request(options);
  req.end();

  req.on('upgrade', function (res, socket, upgradeHead) {
    // 连接成功
    that.setSocket(socket);

    // 触发open事件
    that.onopen();
  });
};
```

下面是服务器端的响应行为：

```js
var server = http.createServer(function (req, res) {
  res.writeHead(200, { 'Content-Type': 'text/plain' });
  res.end('Hello World\n');
});
server.listen(12010);

// 在收到upgrade请求后，告知客户端允许切换协议
server.on('upgrade', function (req, socket, upgradeHead) {
  var head = new Buffer(upgradeHead.length);
  upgradeHead.copy(head);

  var key = req.headers['sec-websocket-key'];
  var shasum = crypto.createHash('sha1');
  key = shasum.update(key + '258EAFA5-E914-47DA-95CA-C5ABODC85B11').disget('base64');

  var headers = [
    'HTTP/1.1 101 Switching Protocols',
    'Upgrade: websocket',
    'Connection: Upgrade',
    'Sec-WebSocket-Accept: ' + key,
    'Sec-WebSocket-Protocol: ' + protocol,
  ];

  // 让数据立即发送
  socket.setNoDelay(true);
  socket.write(headers.concat('', '').join('\r\n'));

  // 建立服务器端WebSocket连接
  var WebSocket = new WebSocket();
  websocket.setSocket(socket);
});
```

一旦 WebSocket 握手成功，服务器端与客户端将会呈现对等的效果，都能接收和发送消息。

### 7.4.2 WebSocket 数据传输

在握手顺利完成后，当前连接不再进行 HTTP 的交互，而是开始 WebSocket 的数据帧协议。实现客户端与服务器端的数据交换。

握手完成后，客户端的`onopen()`将会被触发执行,代码如下：

```js
socket.onopen() = function () {
  // TODO: opened()
};
```

服务器端则没有`onopon()`方法科研。为了完成 TCP 套接字事件到 WebSocket 事件的封装，需要在接收数据时进行处理，WebSockt 的数据帧协议即在底层 data 事件上封装完成：

```js
WebSocket.prototype.setSocket = function (socket) {
  this.socket = socket;
  this.socket.on('data', this.receiver);
};

// 同样的数据发送时，也需要做封装操作，代码如下：
WebSocket.prototype.send = function (data) {
  this._send(data);
};
```

当客户端调用`send()`发送数据时，服务器端触发`onmessage()`；当服务器端调用`send()`发送数据时，客户端的`onmessage()`触发。当我们调用`send()`发送一条数据时，协议可能将一个数据封装为一帧或多帧数据，然后逐帧发送。

为了安全考虑，客户端需要对发送的数据帧进行掩码处理，服务器一旦收到无掩码帧（比如中间拦截破坏），连接将关闭。而服务器发送到客户端的数据帧则无需做掩码处理，同样，如果客户端收到带掩码的数据帧，连接也将关闭。

我们以客户端发送 hello world!到服务器，服务器端回以 yakexi 作为一个流程来研究数据帧协议的实现过程。

WebSocket 数据帧，每 8 位为一列，也即 1 个字节。其中每一位都有它的意义：

- fin：如果这个数据帧时最后一帧，这个 fin 位为 1，其余情况为 0。当一个数据没有被分为多帧时，它既是第一帧也是最后一帧。

- rsv1、rsv2、rsv3：各为 1 位长，3 各标识用于扩展，当有已协商的扩展时，这些值可能为 1，其余情况为 0。

- opcode：长为 4 位的操作码，可以用来表示 0 到 15 的值，用于解释当前数据帧。0 表示附加数据帧，1 表示文本数据帧，2 表示二进制数据帧，8 表示发送一个连接关闭的数据帧，9 表示 ping 数据帧，10 表示 pong 数据帧，其余值暂时没有定义。ping 数据帧和 pong 数据帧用于心跳检测，当一段发送 ping 数据帧时，另一端必须发送 pong 数据帧作为响应，告知对方仍然处于响应状态。

- masked：表示是否进行掩码处理，长度为 1。客户端发送给服务器端时为 1，服务器端发送给客户端时为 0。

- payload length：一个 7、7+16 或 7+64 长的数据位，标识数据的长度的数据位，标识数据的长度，如果值在`0~125`之间，那么该值就是数据的真实长度；如果值是 126，则后面后面 16 为的值是数据的真实长度；如果值是 127，则后面 64 为的值是数据的真实长度。

- masking key：当 masked 为 1 时存在，是一个 32 位长的数据为，用于解密数据。

- payload data：我们的目标数据，位数为 8 的倍数。

客户端发送消息时，需要构造一个或多个数据帧协议报文。由于 hello world!较短，不存在分割多个数据帧的情况，又由于 hello world!会以文本的方式发送，它的 payload length 长度为 96（12 字节 x8 位/字节），二进制标识为 1100000。所以报文应当如下：

```shell
fin(1) + res(000) + opcode(0001) + payload length(1100000) + masking key(32位) + payload data(hello world!加密后的二进制)
```

当以文本方式发送时，文本的编码编码 UTF-8，由于这里发送的不存在中文，所以一个字符占一个字节，即 8 位。

客户端发送消息后，服务器端在 data 事件中接收到这些编码数据，然后解析为相应的数据帧，再以数据帧的格式，通过掩码将真正的数据解密出来，然后触发 onmessage()执行，如下所示：

```js
socket.onmessage = function (event) {
  // TODO: event.data
};
```

服务器端再回复 yakexi 的时候，剩下的事情就无需掩码，其余相同，如下所示：

```shell
fin(1) + res(000) + opcode(0001) + masked(0) + payload length(1100000) + payload data(yakexi的二进制)
```

这里的行为与纯 TCP 连接的行为十分类似，近似地可以理解为 TCP 客户端套接字地 connect 事件和 data 事件。

至此，WebSocket 的原理介绍完毕，具体如何解析数据帧和触发 onmessage()，请参考 ws 模块的实现，由于其有过多细节，这里就不再展开描述。

### 7.4.3 小结

再所有的 WebSocket 服务器端视线中，没有比 Node 更贴近 WebSocket 的使用方式了。它们共性有以下内容：

- 基于事件的编程接口。
- 基于 JavaScript，以封装良好的 WebSocket 实现，API 与客户端可以高度相似。

另外，Node 基于事件驱动的方式使得它对应 WebSocket 这类长连接的应用场景可以轻松地处理大量并发请求。尽管 Node 没有内置 WebSocket 的库，但是社区的 ws 模块封装了 WebSocket 的底层实现。socket.io 即是再它的基础上构建实现的。

## 7.5 网络服务和安全

在网络中，数据在服务器端和客户端之间传递，由于是明文传递的内容，一旦在网络被人监控，数据可能一览无余地展现在中间地窃听者面前。为此我们需要将数据加密后再进行网络传输，这样即是数据被截获和窃听，窃听者也无法知道数据地真实内容是什么。但是对于我们地应用程协议而言，如 HTTP、FTP 等，我们仍然希望能够透明地处理数据，而无需担心网络传输过程中的安全问题。再网景公司的 NetSpace 浏览器推出之初就提出了 SSL（Secure Sockets Layer，安全套接层）。SSL 作为一种安全协议，他在传输层提供对网络连接加密的功能。对于应用层而言，它是透明的，数据再传输到应用层之前就已经完成了加密和解密的过程。最初的 SSL 应用在 Web 上，被服务器端和浏览器端同时支持，随后 IETF 将其标准化，称为 TLS（Transport Layer Security，安全传输层协议）。

Node 在网络安全上提供了 3 个模块，分别为 crypto、tls、https。其中 crypto 主要用于加密解密，SHA1、MD5 等加密算法都在其中有体现，在这里我们不用再提。真正用于网络的是另外两个模块，tls 模块提供了与 net 模块类似的功能，区别在于它建立在 TLS/SSL 加密的 TCP 连接上。对于 https 而言，它完全与 http 模块接口一致，区别也仅仅在于它建立于安全的连接之上。

### 7.5.1 TLS/SSL

#### 1.密钥

TLS/SSL 是一个公钥/私钥的结构，它是一个非对称的结构，每个服务器端和客户端都有自己的公私钥。公钥用来加密要传输的数据，私钥用来解密接收到的数据。公钥和私钥是配对的，通过公钥加密的数据，只有通过私钥才能解密，所以在建立安全传输之前，客户端和服务器之间需要互换公钥。客户端发送数据时要通过服务器端的公钥进行加密，服务器端发送数据时则需要客户端的公钥进行加密，如此才能完成加密解密的过程。

Node 在底层采用的是 openssl 实现 TLS/SSL 的，为此要生成公钥和私钥可以通过 openssl 完成。我们分别为服务器和客户端生成私钥。

```shell
# 生成服务器端私钥
$ openssl genrsa -out server.key 1024
# 生成客户端私钥
$ openssl genrsa -out client.key 1024
```

上述命令生成了两个 1024 位长的 RSA 私钥文件，我们可以通过它继续生成公钥：

```shell
$ openssl rsa -in server.key -pubout -out server.pem
$ openssl rsa -in client.key -pubout -out client.pem
```

公私钥的非对称加密虽然好，但是网络中依然可能存在窃听的情况，典型的例子是中间人攻击。客户端和服务器端在交换公钥的过程中，中间人对客户端扮演服务器端的角色，对服务器端扮演客户端的角色，因此客户端和服务器端几乎感受不到中间人的存在。为了解决这个问题，数据传输过程中还需要对得到的公钥进行认证，以确认得到的公钥来自目标服务器。如果不能保证这种认证，中间人可能会将伪造的站点响应给用户，从而造成经济损失。

为了解决这个问题，TLS/SSL 引入了数字证书来进行认证。与直接用公钥不同，数字证书中包含了服务器的名字和主机名、服务器的公钥、签名颁发机构名称、来自签名颁发机构的签名。在连接建立前，会通过整数中的签名确认收到的公钥是来自目标服务器的，从而产生信任关系。

#### 2. 数字证书

为了确保我们的数据安全，现在我们引入一个第三方：CA（Certificate Authority，数字整数认证中心）。CA 的作用是为站点颁发整数，而这个证书中具有 CA 通过自己的公钥和私钥实现的签名。

为了得到签名证书，服务器端需要通过自己的私钥生成 CSR（Certificate Signing Request，证书签名请求）文件。CA 机构将通过这个文件颁发属于该服务器端的签名证书，只要通过 CA 机构就能验证证书是否合法。

通过 CA 机构颁发证书通常是一个繁琐的过程，需要付出一定的经历和费用。对于中小型企业而言，多半是采用自签名证书来构建安全网络的。所谓自签名证书，就是自己扮演 CA 机构，给自己服务器端颁发签名证书。以下为生成私钥、生成 CSR 文件、通过私钥自签名生成证书的过程：

```shell
$ openssl genrsa -out ca.key 1024
$ openssl req -new -key ca.key -out ca.csr
$ openssl x509 -req in ca.csr -signkey ca.key -out ca.crt
```

上述步骤完成了扮演 CA 角色需要的文件。接下来回到服务器端，服务器端需要向 CA 机构申请签名证书。在申请签名证书之前依然要创建自己的 CSR 文件。值得注意的是，这个过程中的 Common Name 要匹配服务器域名，否则在后续的认证过程中会出错。如下是生成 CSR 文件所用命令：

```shell
$ openssl req -new -key server.key -out server.csr
```

得到 CSR 文件后，向我们自己的 CA 机构申请签名吧。签名过程需要 CA 的证书和私钥参与，最终颁发一个带有 CA 签名的证书：

```shell
$ openssl x509 -req -CA ca.crt -CAkey ca.key -CAcreateserial in server.csr -out server.crt
```

客户端在发起安全连接前回去获取服务器端的证书，并通过 CA 的证书验证服务器端证书的真伪。除了验证真伪外，通常还含有对服务器名称、IP 地址等进行验证的过程。

CA 机构将证书颁发给服务器端后，证书在请求过程中会被发送给客户端，客户端需要通过 CA 证书验证真伪。如果是知名的 CA 机构，它们的证书一般预装在浏览器中。如果是自己扮演 CA 机构，颁发自由签名则不能享受这个福利，客户端需要获取到 CA 的证书才能进行验证。

上述的过程中可以看出，签名证书是一环一环地颁发，但是在 CA 那里的证书是不需要上级证书参与签名的，这个证书我们称为根证书。

### 7.5.2 TLS 服务

#### 1. 创建服务器端

将构建服务所需要的证书都备齐子厚，我们通过 Node 的 tls 模块来创建一个安全的 TCP 服务，这个服务是一个简单的 echo 服务器：

```js
var tls = require('tls');
var fs = require('fs');
var options = {
  key: fs.readFileSync(__dirname + '/server.key'),
  cert: fs.readFileSync(__dirname + '/server.crt'),
  requestCert: true,
  ca: [fs.readFileSync(__dirname + '/ca.crt')],
};

var server = tls.createServer(options, function (stream) {
  console.log('server connected', stream.authorized ? 'authorized' : 'unauthorized');
  stream.write('welcome!\n');
  stream.setEncoding('utf8');
  stream.pipe(stream);
});

server.listen(8000, function () {
  console.log('server bound');
});
```

启动上述服务后，通过下面的命令可以测试证书是否正常：

```sh
openssl s_client -connect 127.0.0.1:8000
```

#### 2.TLS 客户端

为了完善整个体系，接下来我们用 Node 来模拟客户端，如果 net 模块一样，tls 模块也提供了`connect()`方法来构建客户端。在构建我们的客户端之前，需要为客户端生成属于自己的私钥和签名，代码如下：

```sh
# 创建私钥
openssl genrsa -out client.key 1024
# 生成csr
openssl req -new -key client.key -out client.csr
# 生成签名证书
openssl x509 -req -CA ca.crt -CAkey ca.key -CAcreateserial -in client.csr -out client.crt
```

并创建客户端：

```js
var tls = require('tls');
var fs = require('fs');

var options = {
  key: fs.readFileSync('./keys/client.key'),
  cert: fs.readFileSync('./keys/client.crt'),
  ca: [fs.readFileSync('./keys/ca.crt')],
};

var stream = tls.connect(8000, options, function () {
  consoe.log('client connected', stream.authorized ? 'authorized' : 'unauthorized');
  process.stdin.pipe(stream);
});

stream.setEncoding('utf8');
stream.on('data', function (data) {
  console.log(data);
});

stream.on('end', function () {
  server.close();
});
```

启动客户端过程中，用到了客户端生成的私钥、证书、CA 证书。客户端启动之可以在输入流中输入数据，服务端将会回应相同的数据。

至此我们完成了 TLS 的服务器端和客户端的创建。与普通的 TCP 服务器端和客户端相比，TLS 的服务器端和客户端仅仅只在证书的配置上有差别，其余部分基本相同。

### 7.5.3 HTTPS 服务

HTTPS 服务就是工作在 TLS/SSL 上的 HTTP。在了解了 TLS 服务后，创建 HTTPS 服务就是再简单不过的事情。

#### 1. 准备证书

HTTPS 服务需要用到私钥和签名证书，我们可以直接用上文生成的私钥和证书。

#### 2. 创建 HTTPS 服务

创建 HTTPS 服务只比 HTTP 服务多一个选项配置，其余地方几乎相同，代码如下：

```js
var https = require('https');
var fs = require('fs');

var options = {
  key: fs.readFileSync(__dirname + 'server.key'),
  cert: fs.readFileSync(__dirname + 'server.pem'),
};

https
  .createServer(options, function (req, res) {
    res.writeHead(200);
    res.end('hello world!');
  })
  .listen(8000);
```

启动后通过 curl 进行测试：

```sh
curl https://localhost:8000
```

提示证书无法验证通过，由于是自签名证书，curl 工具无法验证服务器端证书是否正确，所以抛出错误。解决问题的方式有两种。一种是加`-k`选项，让 curl 工具忽略证书的签名，这样的结果是数据依然会通过公钥加密传输，但是无法保证对方是可靠的，会存在中间人攻击的潜在风险。另一种解决方式给 curl 设置`--cacert`，告知 CA 证书使之完成对服务器证书的验证。

#### 3.HTTPS 客户端

对应的，我们也会用 Node 来实现 HTTPS 的客户端，与 HTTP 客户端相差不大，除了指定证书相关参数外：

```js
var https = require('https');
var fs = requrie('fs');

var options = {
  hostname: 'localhost',
  port: 8000,
  path: '/',
  method: 'GET',
  key: fs.readFileSync(__dirname + '/client.key'),
  cert: fs.readFileSync(__dirname + '/client.crt'),
  ca: [fs.readFileSync(__dirname + '/ca.crt')],
};

options.agent = new https.Agent(options);
var req = https.request(options, function (res) {
  res.setEncoding('utf8');
  res.on('data', function (d) {
    console.log(d);
  });
});

req.end();

req.on('error', function (e) {
  console.log(e);
});
```

如果不设置 ca 选项，将会得到如下异常：

```sh
[Error: UNABLE_TO_VERIFY_LEAF_SIGNATURE]
```

解决该异常的方案是添加选项属性`rejectUnauthorized`为`false`，它的 i 熬过与 curl 工具加`-k`一样，都会再数据传输过程中加密，但是无法保证服务器端的证书是不是伪造的。

## 7.6 总结

Node 基于事件驱动和非阻塞设计，在分布式环境中尤其能发挥出它的特长，基于事件驱动可以实现与大量的客户端进行连接，非阻塞设计则让它可以更好的提升网络的相应吞吐。Node 提供了相对底层的网络调用，以及基于事件的编程接口，使得开发者在这些模块上十分轻松地构建网络应用。
