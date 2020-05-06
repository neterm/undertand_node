# 玩转进程

---

- [9.1 服务模型的变迁](#91-服务模型的变迁)
- [9.2 多进程架构](#92-多进程架构)
- [9.3 集群稳定之路](#93-集群稳定之路)
- [9.4 Cluster 模块](#94-cluster-模块)
- [9.5 总结](#95-总结)

---

Node 在选型时决定在 V8 引擎之上构建，也就意味着它的木星于浏览器类似。我们的 JavaScript 将会运行在单个进程的单个线程上。它带来的好处是：程序状态是单一的，在没有多线程的情况下没有锁、线程同步问题，操作系统在调度时也因为较少上下文的切换，可以很好地提高 CPU 的使用率。

但是单进程单线程并非完美的结构，如今 CPU 基本均是多核的，真正的服务器（非 VPS）往往还有多个 CPU。一个 Node 进程只能利用一个核，这将抛出 Node 实际应用的第一个问题：**如何充分利用多核 CPU 服务器？**

另外，由于 Node 执行在单进程上，一旦单线程上抛出的异常没有捕获，将会引起整个进程奔溃。这给 Node 的实际应用抛出了第二个问题：**如何保证进程的健壮性和稳定性？**

从严格的意义上而言，Node 并非真正的单线程架构，在第三章中我们有叙述过 Node 自身还有一定的 I/O 线程存在，这些 I/O 线程由底层 libuv 处理，这部分线程对于 JavaScript 开发者而言是透明的，只在 C++扩展开发时才会关注到。JavaScript 代码永远运行在 V8 上，是单线程的。本章将围绕 JavaScript 部分展开，所以屏蔽底层细节的讨论。

## 9.1 服务模型的变迁

从“古”至今，Web 服务器的架构已经经历了几次变迁。服务器处理客户端请求的并发量，就是每个里程碑的见证。

### 9.1.1 石器时代：同步

最早的服务器，其执行模型是同步的，它的服务模式是一次只为一个请求服务，所以请求都得按次序等待服务。这意味着除了当前得请求被处理外，其余请求都处于耽误状态。它得处理能力相当低下，假设每次响应服务好用得事件稳定为 N 秒，这类服务得 QPS 为 1/N。

这类架构如今已基本被淘汰，只在一些无并发要求的应用中存在。

### 9.1.2 青铜时代：复制进程

为了解决同步架构的并发问题，一个简单的改进是通过进程的复制同时服务更多的请求和用户。这样每个连接都需要一个进程来服务，即 100 个连接需要启动 100 个进程来进行服务，这是非常昂贵的代价。在进程复制的过程中，需要复制进程内部的状态，对于每个连接都进行这样复制的话，相同的状态将会在内存中存在很多份，造成浪费。并且这个过程由于要复制较多的数据，启动是较为缓慢的。

为了解决启动缓慢的问题，预复制（prefork）被引入服务模型中，即预先复制一定数量的进程。同时将进程复用，避免进程创建、销毁带来的开销。但是这个模型并不具备伸缩性，一旦并发请求过高，内存使用随着进程数的增长将会被耗尽。

假设通过进行复制和预复制的方式搭建的服务器有资源的限制，且进程数上限为 M，那么这类服务器的 QPS 为 M/N。

### 9.1.3 白银时代：多进程

为了解决进程复制中的浪费问题，多线程被引入服务模型，让一个线程服务一个请求。线程相对进程的开销要小许多，并且线程之间可以共享数据，内存浪费的问题可以得到解决，并且利用线程池可以减少创建和销毁线程的开销。但是多线程面临的问题只能说比多进程略好，因为每个线程都拥有自己独立的堆栈，这个堆栈都需要占用一定的内存空间。另外，由于一个 CPU 核心在一个时刻只能做一件事情，操作系统只能通过将 CPU 切分为时间片的方法，让线程可以较为均匀地使用 CPU 资源，但是操作系统内核在切换线程的同时，也要切换线程的上下文，当线程数量过多时，时间将会被耗用在上下文切换中。所以在大量并发量时，多线程结构还是无法做到强大的伸缩性。

如果忽略掉多线程上下文切换的开销，假设线程所占用的资源为进程的 1/L，受资源上限的影响，它的 QPS 则为 M\*L/N。

### 9.1.4 黄金时代：事件驱动

多线程的服务模型服役了很长一段时间，Apache 就是采用多线程/多进程模型实现的，当并发量增长到上万时，内存耗用的问题将会暴露出来，这即是著名的 C10k 问题。

为了解决高并发问题，基于事件驱动的服务模型出现了，向 Node 与 Nginx 均是基于事件驱动的方式实现的，采用单线程避免了不必要的内存开销和上下文切换开销。

基于事件的服务模型存在的问题即是本章起始时提及的两个问题：CPU 的利用率和进程的健壮性。单线程的架构并不少，其中尤以 PHP 最为知名——在 PHP 中没有线程的支持。它的健壮性是由它给每个请求都建立独立的上下文来实现的。但是对于 Node 来说，所有请求的上下文都是统一的，它的稳定性是亟需解决的问题。

由于所有处理都在单线程上进行，影响事件驱动服务模型性能的点在于 CPU 的计算能力，它的上限绝对这类服务模型的上限，但它不受多进程或多线程模式中资源上限的影响，可伸缩性远比前两者高。如果解决掉多核 CPU 的利用问题，带来的性能提升是客观的。

## 9.2 多进程架构

面对单进程单线程对多核使用不足的问题，前人的经验是启动多进程即可。理想状态下每个进程各自利用一个 CPU，依次实现多核 CPU 的利用。所幸，Node 提供了 child_process 模块，并且提供了 child_process.fork()函数，供我们实现进程的复制。

我们再一次将经典的示例代码存为 worker.js 文件：

```js
var http = require('http')
http
  .createServer(function (req, res) {
    res.writeHead(200, { 'Content-Type': 'text/plain' })
    res.end('Hello World\n')
  })
  .listen(Math.round((1 + Math.random()) * 1000), '127.0.0.1')
```

通过`node worker.js`启动它，竟会侦听 1000 到 2000 之间的一个随机端口。将以下代码存为 master.js，并通过`node master.js`启动它：

```js
var fork = require('child_process').fork
var cpus = require('os').cpus()
for (var i = 0; i < cpus.length; i++) {
  fork('./worker.js')
}
```

这段代码将会根据机器上的 CPU 数量复制出对应 Node 进程数。在\*nix 系统下可以通过`ps aux |grep worker.js`查看进程数量：

```sh
ps aux | grep worker.js
```

这就是著名的 Master-Worker 模式，又称为主从模式。在主从模式中，进程分为两种：主进程和工作进程。这时典型的分布式架构中用于并行处理业务的模式，具备较好的可伸缩性和稳定性。主进程不负责具体的业务处理，而是负责调度或管理工作进程，它是趋向于稳定的。工作进程负责具体的业务处理，因为业务的多种多样，甚至一项业务由多人开发完成，所以工作进程的稳定性值得开发者关注。

通过 fork()复制的进程都是一个独立的进程，这个进程中有着独立而全新的 V8 实例。它需要至少 30ms 的启动时间和至少 10MB 的内存。尽管 Node 提供了 fork()供我们复制进程使每个 CPU 内核都四用上，丹斯依然要切记 fork()进程是昂贵的。好在 Node 通过事件驱动的方式在单线程上解决了大并发的问题，这里启动多个进程只是为了充分将 CPU 资源利用起来，而不是为了解决并发问题。

### 9.2.1 创建子进程

child_process 模块给予 Node 可以随意创建子进程（child_process）的能力。它提供了 4 个方法用于创建子进程。

- spawn()：启动一个子进程来执行命令。
- exec()：启动一个子进程来执行命令，与 spawn()不同的是其接口不同，它有一个回调函数获知子进程的状况。
- execFile()：启动一个子进程来执行可执行文件。
- fork()：与 spawn()类似，不同点在于它创建 Node 的子进程只需要指定执行的 JavaScript 文件模块即可。

spawn()与 exec()、execFile()不同的是，后两者创建时可以指定 timeout 属性设置超时时间，一旦创建的进程运行超过设定的时间将会被杀死。

exec()与 execFile()不同的是，exec()适合执行已有命令，execFile()适合执行文件。这里我们以一个寻常命令为例，`node worker.js`分贝用上述 4 种方法实现：

```js
var sp = require('child_process')

cp.spawn('node', ['worker.js'])
cp.exec('node worker.js', function (err, stdout, stderr) {
  // some code
})
cp.execFile('worker.js', function (err, stdout, stderr) {
  // some code
})
cp.fork('./worker.js')
```

以上 4 个方法在创建子进程之后均会返回子进程对象。他们的差别在于：

| 类型       | 回调/异常 | 进程类型 | 执行类型        | 可设置超时 |
| ---------- | --------- | -------- | --------------- | ---------- |
| spawn()    | no        | 任意     | 命令            | no         |
| exec()     | yes       | 任意     | 命令            | yes        |
| execFile() | yes       | 任意     | 可执行文件      | yes        |
| fork()     | no        | Node     | JavaScript 文件 | no         |

这里的可执行文件是指可以直接执行的文件，如果是 JavaScript 文件通过 execFile()运行，它的首航必须添加如下代码：

```sh
#!/usr/bin/env node
```

尽管 4 种创建进程的方式有些差别，但事实上后三种方法都是 spawn()的延伸应用。

### 9.2.2 进程间通信

在 Master-Worker 模式中，要实现主进程管理和调度工作进程的功能，需要主进程和工作进程之间的通信。对于 child_process 模块，创建好了子进程，然后与父子进程间通信是十分容易的。

在前端浏览器种，JavaScript 主进程与 UI 渲染共用同一个进程。执行 JavaScript 的时候 UI 渲染是停滞的，渲染 UI 时，JavaScript 是停滞的，两者互相阻塞。长时间执行 JavaScript 将会造成 UI 停顿不响应。为了解决这个问题，HTML5 提供了 WebWorker API。WebWorker 允许创建工作线程并在后台运行，使得一些阻塞较为严重的计算不影响主线程上的 UI 渲染。它的 API 如下所示：

```js
var worker = new Worker('worker.js')
worker.onmessage = function (event) {
  document.getElementById('result').textContent = event.data
}

// 其中， worker.js如下：
var n = 1
search: while (true) {
  n += 1
  for (var i = 2; i <= Math.sqrt(n); i++) {
    if (n % i == 0) {
      continue search
      // found a prime
      postMessage(n)
    }
  }
}
```

主进程与工作进程之间通过`onmessage()`和`postMessage()`进行通信，子进程对象则由`send()`方法实现主进程向子进程发送数据，message 事件实现收听子进程发来的数据，与 API 在一定程度上相似。通过消息传递内容，而不是共享或直接操作相关资源，这时较为轻量和无依赖的做法。Node 种对应示例如下：

```js
// parent.js

var cp = require('child_process')
var n = cp.fork(__dirname + './sub.js')

n.on('message', function (m) {
  console.log('PARENT go message:', m)
})

n.send({ hello: 'world' })

// sub.j
process.on('message', function (m) {
  console.log('CHILD got message: ', m)
})

process.send({ foo: 'bar' })
```

通过 fork()或者其它 APiece，创建子进程之后，为了实现父子进程之间的通信，父进程与子进程之间将会创建 IPC 通道。通过 IPC 通道，父子进程才能通过 message 和 send()传递消息。

- 进程间通信原理

IPC 的全称是 Inter-Process Communication，即进程间通信。进程间通信的目的是为了让不同的进程能够互相访问资源并进行协调工作。实现进程间通信的技术由很多，如命名管道、匿名管道、socket、信号量、共享内存、消息队列、Domain Socket 等。Node 种实现 IPC 通道的是管道（pipe)技术。但此管道非彼管道，在 Node 种管道是个抽象层的称呼，具体实现细节由 libuv 提供，在 Window 下由命名管道（named pipe）实现，\*nix 系统则采用 Unix Domain Socket 实现。表现在应用蹭上的进程间通信只有简单的 message 事件和 send()方法，接口十分简洁和消息化。

父进程在实际创建子进程之前，会创建 IPC 通道并监听它，然后才真正创建出子进程，并通过环境变量（NODE_CHANNEL_FD）告诉子进程这个 IPC 通道的文件描述符。子进程在启动的过程中，根据文件描述符去连接这个已经存在的 IPC 通道，从而完成父子进程的连接。

建立连接之后，父子进程就可以自由地通信了。由于 IPC 通道是用命名管道或 Domain Socket 创建的，它们与网络 socket 的行为比较类似，属于双向通信。不同的是它们在系统内核种就完成了进程间的通信，而不用经过实际的网络层，非常高效。在 Node 中，IPC 通道被抽象为 Stream 对象，在调用 send()时发送数据（类似于 write()），接受到的消息会通过 message 事件（类似于 data）触发给应用层。

> 只有启动的子进程是 Node 进程时，子进程才会根据环境变量去连接 IPC 通道，对于其它类型的子进程则无法实现进程间的通信，除非其它进程也按约定去连接这个已经创建好的 IPC 通道。

### 9.2.3 句柄传递

建立好进程之间的 IPC 后，如果仅仅只用来发送一些简单的数据，显然不够我们的实际应用使用。还记得本章第一部分代码需要将启动的服务器分别监听各自的端口么，如果让服务都监听到相同的端口，将会有什么样的结果？

```js
var http = require('http')
http
  .createServer(function (req, res) {
    res.writeHead(200, { 'Content-Type': 'text/plain' })
    res.end('Hello World\n')
  })
  .listen(8888, '127.0.0.1')
```

再次启动 master.js，将报错。这时只有一个工作进程能够监听到该端口，其余的进程在监听的过程中都抛出了 EADDRINUSE 异常，这是端口被专用的情况，新的进程不能继续监听该端口了。这个问题破坏了我们将多个进程监听同一个端口的想法。要解决这个问题，通常的做法是让每个进程监听不同的端口，其中主进程监听著端口（如 80），主进程堆外接受所有的网络请求，再将这些请求分别代理到不同的端口的进程上。

通过代理，可以避免端口不能重复监听的问题，甚至可以在代理进程上做适当的负载均衡，使得每个子进程可以较为均衡地执行任务。由于进程每接收到一个请求，将会用掉一个文件描述符，因此代理方案中客户端连接到代理进程，代理进程连接到工作进程的过程需要用掉两个文件描述符。操作系统的文件描述符是有限的，代理方案浪费掉一倍数量的文件描述符的做法影响了系统的扩展能力。

为了解决上述这样的问题，Node 在版本 v0.5.9 引入了进程间发送句柄的功能。send()方法除了能通过 IPC 发送数据外，还能发送句柄，第二个可选参数就是句柄：`child.send(message[, sendHanle])`。

那什么是句柄？句柄是一种可以用来标识资源的引用，它的内部包含了指向对象的文件描述符。比如句柄可以用来标识一个服务器端的 socket 对象、一个客户端 soket 对象、一个 UDP 套接字、一个管道等。

发送句柄意味着什么？在前一个问题中，我们可以去掉代理这种方案，使主进程接受到 socket 请求后，将这个 socket 直接发送给工作进程，而不是重新与工作进程之间建立新的 socket 连接来转发数据。文件描述符浪费的问题可以通过这样的方式轻松解决。来看看我们的示例代码：

```js
//主进程代码
var child = require('child_process').fork('child.js')

// open up the server object and send the handle
var server = require('net').createServer()
server.on('connection', function (soket) {
  socket.end('handled by parent\n')
})

server.listen(1337, function () {
  child.send('server', server)
})

// 子进程代码：
process.on('message', function (m, server) {
  if (m === 'server') {
    server.on('connection', function (socket) {
      soket.end('handled by child\n')
    })
  }
})
```

这个示例中，直接将一个 TCp 服务器发送给子进程。这是看起来不可思议的事情，我们先来测试一番，看看效果如何：

```sh
# 先启动服务器
node parent.js

# 然后开一个新的命令行窗口，用上curl工具
curl 'http://127.0.0.1:1337/'
# => handled by parent
curl 'http://127.0.0.1:1337/'
# => handled by child
curl 'http://127.0.0.1:1337/'
# => handled by child
curl 'http://127.0.0.1:1337/'
# => handled by parent
```

命令行中的响应结果也是很不可思议的，这里的子进程和父进程都有可能处理我们客户端发起的请求。实时将服务发送给多个子进程：

```js
// parent.js
var cp = require('child_process')

var child1 = cp.fork('child.js')
var child2 = cp.fork('child.js')

// Open up server Object and send the handle

var server = require('net').createServer()
server.on('connection', function (socket) {
  socket.end('handled by parent\n')
})

server.listen(1337, function () {
  child1.send('server', server)
  child2.send('server', server)
})

// 然后在子进程中将进程ID打印出来：

// child.js
process.on('message', function (m, server) {
  if (m === 'server') {
    server.on('connection', function (socket) {
      socket.end('handled by child, pid is ' + process.pid + '\n')
    })
  }
})
```

再用 curl 测试我们的服务：

```sh
curl 'http://127.0.0.1:1337/'
# => handled by child pid is 24673
curl 'http://127.0.0.1:1337'
# => handled by parent
curl 'http://127.0.0.1:1337'
# => handled by child, pid is 24672

```

测试的结果是每次出现的结果都可能不同，如果可能被父进程处理，也可能被不同的子进程处理。并且这是在 TCP 层面上完成的事情，我们尝试将其转换到 HTTP 层面来实时。对于主进程而言，我们甚至可以让他更轻量一点，那么是否将服务器句柄发送给子进程之后，就可以关闭服务器的监听，让子进程来处理请求呢？

我们对主进程进行改动：

```js
var cp = require('child_process')

var child1 = cp.fork('child.js')
var child2 = cp.fork('child.js')

// Open up the server object and send the handle
var server = require('net').createServer()
server.listen(1337, function () {
  child1.send('server', server)
  child2.send('server', server)
  // 关闭
  server.close()
})

// 子进程进行改动：
var http = require('http')
var server = http.createServer(function (req, res) {
  res.writeHead(200, { 'Content-Type': 'text/plain' })
  res.end('handled by child, pid is ' + process.pid + '\n')
})

process.on('message', function (m, tcp) {
  if (m === 'server') {
    tcp.on('connection', function (socket) {
      server.emit('connection', soket)
    })
  }
})
```

重启 parent.js 后，再次测试：

```sh
curl "http://127.0.0.1:1337/"
# => handled by child, pid = 24852
curl "http://127.0.0.1:1337/"
# => handled by child, pid = 24851
```

这样一来，所有的请求都由子进程处理了。整个过程中，服务的过程发生了依次改变。

我们神奇地发现，多个子进程可以同时监听相同端口，在没有 EADDRINUSE 异常发生了。

#### 1.句柄发送与还原

上文介绍地虽然是句柄发送，但是仔细看看，句柄发送跟我们直接将服务器对象发送给子进程有没有差别？它是否真的将服务器对象发送给子进程？为什么它可以发送到多个子进程中？发送给子进程为什么父进程中还存在这个对象？本节将解开这些秘密地所在。

目前子进程对象 send()方法可以发送地句柄类型包括以下几种：

- net.Socket。TCP 套接字。
- net.Server。TCP 服务器，任意建立在 TCP 服务上的应用层服务都可以享受到它带来的好处。
- net.Native。C++层面的 TCP 套接字或 IPC 管道。
- dgram.Socket。UDP 套接字。
- dgram.Native。C++层面的 UDP 套接字。

send()方法在将消息发送到 IPC 管道前，将消息组装成两个对象，一个参数是 handle，另一个参数是 message。message 参数如下：

```js
{
  cmd: 'NODE_HANDLE',
  type: 'net.Server',
  msg: message
}
```

发送到 IPC 管道中的实际上是我们要发送的句柄文件描述符，文件描述符实际上是一个整数值。这个 message 对象在写入到 IPC 管道时也会通过 JSON.stringify()进行序列化。所以最终发送到 IPC 通道的信息都是字符串，send()方法能发送消息和句柄并不意味着他能发送任意对象。

连接 IPC 通道的子进程可以读取到父进程发来的消息，将字符串通过 JSON.parse()解析还原为对象后，才触发 message 事件将消息体传递给应用层使用。在这个过程中，消息对象还被进行过滤处理，message.cmd 的值如果以 NODE\_为前缀，它将响应一个内部事件 internalMessage。如果 message.cmd 的值为 NODE_HANDLE，它将取出 message.type 的值和得到的文件描述符一起还原出一个对应的对象。

以发送的 TCP 服务器的句柄为例，子进程收到消息后的还原过程如下：

```js
function (message, handle, emit) {
  var self = this;

  var server = new net.Server();
  server.listen(handle, function () {
    emit(server);
  });
}
```

上面的代码中，子进程根据 message.type 创建对应的 TCP 服务器对象，然后监听到文件描述符上。由于底层细节不被应用层感知，所以在子进程中，开发者会有一种服务器就是从父进程中直接传递过来的错觉。值得注意的是，Node 进程之间只有消息传递，不会真正地传递对象，这种错觉是抽象封装地结果。

目前 Node 只支持上述提到的几种句柄，并非任意类型的句柄都能在进程之间传递，除非它有完整的发送和还原的过程。

#### 2. 端口共同监听

在了解了句柄传递背后的原理后，我们继续探究为何通过发送句柄后，多个进程可以监听到相同的端口而不引起 EADDRINUSE 异常。其答案也很简单，我们独立启动的进程中，TCP 服务器端 socket 套接字的文件描述符并不相同，导致监听到相同的端口时会抛出异常。

Node 底层对每个端口监听都设置了 SO_REUSEADDR 选项，这个选项的含义是不同进程可以就相同的网卡和端口进行监听，这个服务器端套接字可以被不同的进程复用：

```c++
setsocket(tcp->io_watcher.fd, SOL_SOCKET, SO_REUSEADDR, &on, sizeof(on));
```

由于独立启动的进程互相之间并不知道文件描述符，所以监听相同端口时就会失败。但对于 send()发送的句柄还原出来的服务而言，他们的文件描述符是相同的，所以监听相同端口不会引起异常。

多个应用监听相同端口时，文件描述符同一时间只能被某个进程所用。换言之就是网络请求向服务器端发送时，只有一个幸运的进程能够抢到连接，也就是说只有它能为这个请求进行服务。这些进程服务是抢占式的。

### 9.2.4 小结

至此，我们介绍了创建子进程、进程间通信的 IPC 通道实现、句柄在进程间的发送和还原、端口共用等细节。通过这些基础技术，用 child_process 模块在单机上搭建 Node 集群是件相对容易的事情。因此在多核 CPU 的环境下，让 Node 进程能够充分利用资源不再是难题。

## 9.3 集群稳定之路

搭建好了集群，充分利用了多核 CPU 资源，似乎就可以迎接客户端大量的请求了。但请等等，我们还有一些细节需要考虑：

- 性能问题。
- 多个工作进程的存活状态管理。
- 工作进程的平滑启动。
- 配置或静态数据的动态重新载入。
- 其它细节。

是的，虽然我们创建了很多工作进程，但每个工作进程依然是单线程上执行的，它的稳定性还不能得到完全的保障。我们需要建立起一个健全的机制来保障 Node 应用的健壮性。

### 9.3.1 进程事件

再次回归到子进程对象上，除了引人关注的 send()方法和 message 事件外，子进程还有些什么呢？首先除了 message 事件外，Node 还有如下事件：

- error：当子进程无法被复制创建、无法被杀死、无法发送消息时会触发该事件。
- exit：子进程退出时触发该事件，子进程如果是正常退出这个事件的第一个参数为退出码，否则为 null。如果进程是通过 kill()方法杀死的，会得到第二个参数，它标识杀死进程时的信号。
- close：在子进程的标准输入输出流中止时触发该事件，参数与 exit 相同。
- disconnect：在父进程或子进程中调用 disconnect()方法时触发该事件，在调用该方法时将关闭监听 IPC 通道。

上述这些事件是父进程能监听到的与子进程相关的事件。除了 send()外，还能通过 kill()方法给子进程发送消息。kill()方法并不能真正地将通过 IPC 相连的子进程杀死，它知识给子进程发送了一个系统信号。默认情况下，父进程将通过 kill()方法给子进程发送一个 SIGTERM 信号。它与进程默认的 kill()方法类似：

```js
// 子进程
child.kill([signal])
// 当前进程
process.kill(pid[, signal])
```

他们一个发送给子进程，一个发送给目标进程。在 POSIX 标准中，一套完备的信号系统，在命令中执行 kill -l 可以看到详细的信号列表：

```sh
kill -l
 1) SIGHUP       2) SIGINT       3) SIGQUIT      4) SIGILL       5) SIGTRAP
 6) SIGABRT      7) SIGBUS       8) SIGFPE       9) SIGKILL     10) SIGUSR1
11) SIGSEGV     12) SIGUSR2     13) SIGPIPE     14) SIGALRM     15) SIGTERM
16) SIGSTKFLT   17) SIGCHLD     18) SIGCONT     19) SIGSTOP     20) SIGTSTP
21) SIGTTIN     22) SIGTTOU     23) SIGURG      24) SIGXCPU     25) SIGXFSZ
26) SIGVTALRM   27) SIGPROF     28) SIGWINCH    29) SIGIO       30) SIGPWR
31) SIGSYS      34) SIGRTMIN    35) SIGRTMIN+1  36) SIGRTMIN+2  37) SIGRTMIN+3
38) SIGRTMIN+4  39) SIGRTMIN+5  40) SIGRTMIN+6  41) SIGRTMIN+7  42) SIGRTMIN+8
43) SIGRTMIN+9  44) SIGRTMIN+10 45) SIGRTMIN+11 46) SIGRTMIN+12 47) SIGRTMIN+13
48) SIGRTMIN+14 49) SIGRTMIN+15 50) SIGRTMAX-14 51) SIGRTMAX-13 52) SIGRTMAX-12
53) SIGRTMAX-11 54) SIGRTMAX-10 55) SIGRTMAX-9  56) SIGRTMAX-8  57) SIGRTMAX-7
58) SIGRTMAX-6  59) SIGRTMAX-5  60) SIGRTMAX-4  61) SIGRTMAX-3  62) SIGRTMAX-2
63) SIGRTMAX-1  64) SIGRTMAX
```

Node 提供了这些信号对应的信号事件，每个进程都可以监听这些信号事件。这个信号事件是用来通知进程的，每个信号事件有不同的含义，进程在收到响应信号时，应当做出约定的行为，如 SIGTERM 是软件终止信号，进程收到该信号时应当退出：

```js
process.on('SIGTERM', function () {
  console.log('Got a SIGTERM, exiting...')
  process.exit(1)
})

console.log('server running with PID: ', process.pid)
process.kill(process.pid, 'SIGTERM')
```

### 9.3.2 自动重启

有了父子进程之间的相关事件之后，就可以在这些关系之间创建出需要的机制了。知少我们能够通过监听子进程的 exit 事件来获知其退出的信息，接着前文的多进程架构，我们在主进程上要加入一些子进程管理的机制，比如重新启动一个工作进程来继续服务。

```js
// master.js
var fork = require('child_process').fork
var cpus = require('os').cpus()

var server = require('net').createServer()
server.listen(1337)

var workers = {}
var createWorker = function () {
  var worker = fork(__dirname + '/worker.js')
  // 退出时重新启动新的进程
  worker.on('exit', function () {
    console.log('Worker ' + worker.pid + 'exited.')
    delete workers[worker.pid]
    createWorker()
  })

  // 句柄转发
  worker.send('server', server)
  worders[worker.pid] = worker
  console.log('Create worker. pid: ' + worker.pid)
}

for (var i = 0; i < cpus.length; i++) {
  createWorker()
}

// 进程自己退出时，让所有工作进程退出
process.on('exit', function () {
  for (var pid in workers) {
    workers[pid].kill()
  }
})

// 测试以上代码：
// node master.js
// => Create worker. pid: 30504
// => Create worker. pid: 30505
// => Create worker. pid: 30506
// => Create worker. pid: 30507
```

通过 kill 命令杀死某个进程 `kill 30506`，结果是 30506 进程退出后，自动启动了一个新的工作进程 30518，总体进程数量并没有发生改变。

这个场景中，我们主动杀死了一个进程，在实际业务中，可能有隐藏的 bug 导致工作进程退出，那么我们需要仔细地处理这种异常：

```js
// worker.js
var http = require('http')
var server = http.createServer(function (req, res) {
  res.writeHead(200, { 'Content-Type': 'text/plain' })
  res.end('handled by child, pid is ' + process.pid + '\n')
})

var worker
process.on('message', function (m, tcp) {
  if (m === 'server') {
    worker = tcp
    worker.on('connection', function (socket) {
      server.emit('connection', socket)
    })
  }
})

process.on('uncaughtException', function () {
  // 停止接收新的连接
  worker.close(function () {
    // 所有已有连接断开后，退出进程
    process.exit(1)
  })
})
```

上述代码的处理流程是，一旦有未捕获的异常出现，工作进程就会立即停止接收新的连接；当所有连接断开后，退出进程。主进程在侦听到工作进程的 exit 后，将会立即启动新的进程服务，以保证整个集群中总是有进程在为用户服务的。

#### 1. 自杀信号

当然上述代码存在的问题是要等到已有的所有连接断开后进程才退出，在极端的情况下，所有工作进程都停止接受新的连接，全处在等待退出的状态。但在等到进程完全退出才重启的过程中，所有新来的请求可能存在没有工作进程为新用户服务的场景，这会丢掉大部分请求。

为此需要改进这个过程，不能等到工作进程推出后才重启新的工作进程。当然也不能暴力退出进程，因为这样会导致得知要退出时，向主进程发送一个自杀信号，然后才停止接收新的连接，当所有连接都断开后才退出。主进程在接收到自杀信号后，立即创建新的工作进程服务：

```js
// worker.js
process.on('uncaughtException', function (err) {
  process.send({act: 'suicide'})
  // 停止接收新的连接
  worker.close(function () {
    // 所有已有连接断开后，退出进程
    process.exit(1)
  })
})

// 主进程将重启工作进程的任务，从exit事件的处理函数中转移到message事件的处理函数中：
var createWorker = function () {
  var worker = for(__dirname + '/worker.js')
  // 启动新进程
  worker.on('message', function (message) {
    if(message.act === 'suicide') {
      createWorker()
    }
  })

  worker.on('exit', function () {
    console.log('Worker ' + worker.pid + 'exited.')
    delete workers[worker.pid]
  })

  worker.send('server', server)
  workers[worker.pid] = worker;
  console.log('Create worker. pid: ' + worker.pid)
}

// 为了模拟未捕获的异常，我们将工作进程的处理代码改为抛出异常，一旦用户请求，将会有一个可怜的工作进程退出：

var server = http.createServer(function (req, res) {
  res.writeHead(200, {'Content-Type': 'text/plain'})
  res.end('handled by child, pid is ' + process.pid + '\n')
  throw new Error('throw exception')
})
```

启动所有进程后，用 curl 工具测试效果。

与前一种方案相比，创建新工作进程在前，退出异常进程在后。在这个可怜的异常进程退出之前，总是有新的工作进程来提上它的岗位。至此我们完成了进程的平滑启动，一旦有一场出现，主进程会创建新的工作进程来为用户服务，旧的进程一旦处理完已有链接就自动断开。整个过程使得我们的应用的稳定性和健壮性大大提高。

这里存在的问题是有可能我们的连接是长连接，不是 HTTP 服务的这种短链接，等待长连接断开可能需要较久的事件。为此为已有链接的断开设置一个超时时间是有必要的，在限定时间里强制退出设置如下：

```js
process.on('uncaughtException', function (err) {
  process.send({ act: 'suicide' })
  // 停止接收新的连接
  worker.close(function () {
    // 所有已有连接断开后，退出进程
    process.exit(1)
  })

  // 5s后退出进程
  setTimeout(function () {
    process.exit(1)
  }, 5000)
})
```

#### 2. 限量重启

通过自杀信号告知主进程可以使得新连接总是有进程服务，但是依然还是有极端的情况。工作进程不能无限制地被重启，如果启动地过程中就发生了错误，或者启动后接到链接就收到错误，会导致工作进程被频繁重启，这种频繁重启不属于我们捕获未知异常地情况，因为这种短时间内频繁重启已经不符合预期地设置，极有可能是程序编写的错误。

为了消除这种无意义的重启，在满足一定规则的限制下，不应当反复重启。比如在单位时间内规定只能重启多少次，超过限制就会触发 giveup 事件，告知放弃重启工作进程这个重要事件。

为了完成限量重启的统计，我们引入一个队列来做标记，在每次重启工作进程之间进行打点并判断重启是否太过频繁：

```js
// 重启次数
var limit = 10

// 时间单位
var during = 60000
var restart = []
var isTooFrequently = function () {
  // 记录重启时间
  var time = Date.now()
  var length = restart.push(time)
  if (length > limit) {
    // 取出最后10个记录
    restart = restart.slice(limit * -1)
  }

  // 最后一次重启到前10此重启之间的时间间隔
  return restart.length >= limit && restart[restart.length - 1] - restart[0] < during
}

var workers = []
var createWorker = function () {
  // 检查是否太过频繁
  if (isTooFrequently()) {
    // 触发giveup事件后，不再重启
    process.emit('giveup', length, during)
    return
  }

  var worker = fork(__dirname + '/worker.js')
  worker.on('exit', function () {
    console.log('Worker ' + worker.pid + 'exited.')
    delete workers[worker.pid]
  })

  // 重新启动新进程
  worker.on('message', function (message) {
    if (message.act === 'suicide') {
      createWorker()
    }
  })

  // 句柄转发
  worker.send('server', server)
  workers[worker.pid] = worker
  console.log('Create worker. pid: ' + worker.pid)
}
```

giveup 事件是比 uncaughtExpection 更严重的异常事件。uncaughtExpection 只代表集群中某个工作进程退出，在整体性保证下，不会出现用户得不到服务的情况，但是这个 giveup 事件则表示集群中没有任何进程服务了，十分危险。为了健壮性考虑，我们应该在 giveup 事件中添加重要日志，并让监控系统监视这个严重错误，进而警报。

### 9.3.3 负载均衡

在多个进程之间监听相同的端口，使得用户请求能够分散在多个进程上进行处理，这带来的好处是可以见 CPU 资源都调用起来。这犹如饭店将客人的点单分发给多个厨师进行餐点制作。既然涉及多个厨师共同处理所有菜单，那么保证每个厨师的工作量是一门学位，既不能让一些厨师忙不过来，也不能让一些厨师闲着，这种保证多个处理单元工作量公平的策略叫负载均衡。

Node 默认提供的机制是采用操作系统的抢占式策略。所谓的抢占式策略就是在一堆工作进程中，闲着的进程相对来的请求进行争抢，谁抢到了谁服务。

一般而言，这种抢占式策略对大家是公平的，各个进程可以根据自己的繁忙都来进行抢占。但是对于 Node 而言，要分清的是它的繁忙是由 CPU、I/O 两个部分构成，影响抢占是 CPU 的繁忙都。不同的业务，可能存在 I/O 频繁，而 CPU 较为空闲的情况，这个可能造成某个进程能够抢的更多请求，形成负载不均衡的情况，这可能造成某个进程能够强盗较多请求，形成负载不均衡的情况。

为此 Node 在 v 0.11 中提出了一种新的策略使得负载均衡更合理，这种新的策略叫 Round-Robin，又叫轮叫调度。轮叫调度的工作方式是由主进程接收连接，将其一次分发给工作进程。分发的策略是在 N 个工作进程中，每次选择第`i = (i + 1)mod`个进程来发送连接。在 cluster 模块中启用他的方式如下：

```js
// 启用Round-Robin
cluster.schedulingPolicy = cluster.SCHED_RR
// 不启用Round-Robin
cluster.schedulingPolicy = cluster.SCHED_NONE

// 或者在环境变量中设置NODE_CLUSTER_POLICY的值

export NODE_CLUSTER_SCHED_POLICY=rr
export NODE_CLUSTER_SCHED_POLICY=none
```

Round-Robin 非常简单，可以避免 CPU 和 I/O 繁忙差异导致的负载不均衡。Round-Robin 策略也可以通过代理服务器来实现，但是它会导致服务器上小号的文件描述符是平常方式的两倍。

### 9.3.4 状态共享

在第五章中，我们提到在 Node 进程中不宜存放太多数据，因为它会家中垃圾回收的负担，进而影响性能。同时，Node 也不允许在多个进程之间共享数据。但在实际的业务中，往往需要共享一些数据，譬如配置数据，这在多个进程中应当是一致的。为此，在不允许共享数据的情况下，我们需要一种方案和机制来实现数据在多个进程之间的共享。

#### 1. 第三方数据存储

解决数据共享最直接、最简单的方式就是通过第三方进行数据存储，比如将数据存放到数据库、磁盘文件、缓存服务（如 Redis）中，所有工作进程启动时将其读取进内存中。但这种方式存在的问题是如果数据发生改变，还需要一种机制通知到各个子进程，使得它们的内部状态也得到更新。

实现状态同步的机制有两种，一种在各个子进程去向第三方进行定时轮询。

定时轮询带来的问题是轮询时间不能过密，如果子进程过多，就会形成并发处理，如果数据并没有发生改变，这些轮询会没有意义，白白增加查询状态的开销。如果轮询时间国常，数据发生改变时，不能及时更新到子进程中中，会有一定延迟。

#### 2. 主动通知

一种改进的方式是当数据发生更新时，主动通知子进程。当然即使是主动通知，也需要一种机制来即使获取数据的改变。这个过程仍然不能脱离轮询，但我们可以减少轮询的进程数量，我们将这种用来发送通知和查询状态是否更改的进程叫做通知进程。为了不混合业务逻辑，可以将这个进程设计为之轮询和通知，不处理任何业务逻辑。

这种推送机制如果按进程间信号传递，在跨多台服务器时会无效，是故可以考虑采用 TCP 或 UDP 的方案。进程在启动时从通知服务处除了读取第一数据外，还将进程信息注册到通知服务处。一旦通过轮询发现有数据更新后，根据注册信息，将更新后的数据发送给工作进程。由于不涉及太多进程去向同一地方进行状态查询，状态响应处的压力不至于太过巨大，单一的通知服务轮询带来的压力并不大，所以可以将轮询时间调整得较短，一旦发现更新，就能实时地推送到各个子进程中。

## 9.4 Cluster 模块

前文介绍了 child_process 模块中地大多数细节，以及如何通过这个模块构建强大地单机集群。如果熟知 Node，也许你会惊讶为何迟迟不谈 cluster 模块。上述提及地问题，Node 在 v 0.8 版本时新增地 cluster 模块就能解决。在 v0.8 本本之前，实现多进程架构必须通过 child_process 来实现，要创建单机 Node 集群，由于有那么多细节需要处理，对普通工程师而言是一件相对较难的工作，于是 v0.8 时直接引入了 cluster 模块，用以解决多核 CPU 的利用率问题，同时也提供了较完善的 API，用以处理进程的健壮性问题。

对于文章开头提到的创建 Node 进程集群，cluster 实现起来也是很轻松的事情：

```js
// cluster.js
var cluster = require('cluster')

cluster.setupMaster({
  exec: 'worker.js',
})

var cpus = require('os').cpus()
for (var i = 0; i < cpus.length; i++) {
  cluster.fork()
}
```

执行 `node cluster.js`将会得到与前文创建子进程集群的效果相同。就官方的文档而言，它更喜欢如下形式作为示例：

```js
var cluster = require('cluster')
var http = require('http')
var numCPUs = require('os').cpus().length

if (cluster.isMaster) {
  // Fork workers
  for (var i = 0; i < numCPUs; i++) {
    cluster.fork()
  }

  cluster.on('exit', function (worker, code, signal) {
    console.log('worker ' + worker.process.pid + ' died.')
  })
} else {
  // Workers can share any TCP connection
  // In this case its a HTTP server
  http
    .createServer(function (req, res) {
      res.writeHead(200)
      res.end('hello world')
    })
    .listen(8000)
}
```

在进程中判断是主进程还是工作进程，主要取决于环境变量中是否有 NODE_UNIQUE_ID，如下：

```js
cluster.isWorker = 'NODE_UNIQUE_ID' in process.env
cluster.isMaster = cluster.isWorker === false
```

但是官方示例中，忽而判断 cluster.isMaster，忽而判断 cluster.isWorker，对于代码的可读性十分差。我建议用 cluster.setupMaster()这个 API，将主进程和工作进程从代码上完全剥离，如同 send()方法可以看起来直接将服务器从主进程发送到子进程那样神奇，玻璃代码后，甚至都感觉不到主进程中有任何服务器相关的代码。

通过 cluster.setupMaster()创建子进程而不是使用 cluster.fork()，程序结构不再凌乱，逻辑分明，代码的可读性和可维护性较好。

### 9.4.1 Cluster 工作原理

事实上 cluster 模块就是 child_process 和 net 模块的组合应用。cluster 启动时，如同我们在 9.2.3 节里的代码一样，它会在内部启动 TCP 服务器，在 cluster.fork()子进程时，将这个 TCP 服务器端的 socket 的文件描述符发送给工作进程。如果进程时通过 cluster.fork()复制出来的，那么它的环境变量里就存在 NODE_UNIQUE_ID，如果工作进程中存在 listen()侦听网络端口的调用，它将拿到该文件描述符，通过 SO_REUSEADDR 端口重用，从而实现多个子进程共享端口。对于普通方式启动的过程，则不存在文件描述符传递共享等事情。

在 cluster 内部隐式创建 TCP 服务器的方式对于使用者来说十分透明，但也正式这种方式使得它无法如直接使用 child_process 那样灵活。在 cluster 模块应用中，一个主进程只能管理一组工作进程。

对于自行通过 child_process 来操作时，则可以更灵活地控制工作进程，甚至控制多组工作进程。其原因在于自行通过 child_process 操作紫禁城是，可以隐式地创建多个 TCP 服务器，使得子进程可以共享多个服务器端 socket。

### 9.4.2 Cluster 事件

对于健壮性处理，cluster 模块也暴露了相当多的事件。

- fork: 复制一个工作进程后，触发该事件。
- online：复制好一个工作进程后，工作进程主动发送一条 online 消息给主进程，主进程收到消息后，触发该事件。
- listening：工作进程中调用 listen()（共享了服务器端 socket）后，发送一条 listening 消息给主进程，主进程收到消息后，触发该事件。
- disconnect：主进程和工作进程之间 IPC 通道断开后会触发该事件。
- exit：有工作进程退出时触发该事件。
- setup：cluster.setupMaster()执行后触发该事件。

这些事件大多数跟 child_process 模块的事件有关，在进程间消息传递的基础上完成封装。这些事件对于增强应用的健壮性已经足够了。

## 9.5 总结

尽管 Node 从单线程的角度来讲它有够脆弱的：既不能充分利用多核 CPU 资源，稳定性也无法得到保障。但是群体的力量时强大的，通过简单的主从模式，就可以将应用的质量提升一个档次。在实际的复杂业务中，我们可能要启动很多子进程来处理任务，结构甚至远比主从模式复杂，但是每个子进程应当是简单到只做一件事，并做好一件事，将复杂分解为简单，将简单组合成强大。

尽管通过 child_process 模块可以大幅度提升 Node 的稳定性，但是一旦主进程出现问题，所有子进程将失去管理。在 Node 的进程管理之外，还需要用监听进程数量或监听日志的方式确保整个系统的稳定性，即使主进程出错退出时，也能及时得到监控警报，使得开发者可以及时处理故障。
