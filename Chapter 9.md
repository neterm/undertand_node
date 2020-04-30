# 玩转进程

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
var http = require('http');
http
  .createServer(function (req, res) {
    res.writeHead(200, { 'Content-Type': 'text/plain' });
    res.end('Hello World\n');
  })
  .listen(Math.round((1 + Math.random()) * 1000), '127.0.0.1');
```

通过`node worker.js`启动它，竟会侦听 1000 到 2000 之间的一个随机端口。将以下代码存为 master.js，并通过`node master.js`启动它：

```js
var fork = require('child_process').fork;
var cpus = require('os').cpus();
for (var i = 0; i < cpus.length; i++) {
  fork('./worker.js');
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
var sp = require('child_process');

cp.spawn('node', ['worker.js']);
cp.exec('node worker.js', function (err, stdout, stderr) {
  // some code
});
cp.execFile('worker.js', function (err, stdout, stderr) {
  // some code
});
cp.fork('./worker.js');
```

以上 4 个方法在创建子进程之后均会返回子进程对象。他们的差别在于：

| 类型       | 回调/异常 | 进程类型 | 执行类型        | 可设置超时 |
| ---------- | --------- | -------- | --------------- | ---------- |
| spawn()    | no        | 任意     | 命令            | no         |
| exec()     | yes       | 任意     | 命令            | yes        |
| execFile() | yes       | 任意     | 可执行文件      | yes        |
| fork()     | no        | Node     | JavaScript 文件 | no         |

这里的可执行文件是指可以直接执行的文件，如果是JavaScript文件通过execFile()运行，它的首航必须添加如下代码：

```sh
#!/usr/bin/env node
```

尽管4种创建进程的方式有些差别，但事实上后三种方法都是spawn()的延伸应用。

### 9.2.2 进程间通信

在Master-Worker模式中，要实现主进程管理和调度工作进程的功能，需要主进程和工作进程之间的通信。对于child_process模块，创建好了子进程，然后与父子进程间通信是十分容易的。

在前端浏览器种，JavaScript主进程与UI渲染共用同一个进程。执行JavaScript的时候UI渲染是停滞的，渲染UI时，JavaScript是停滞的，两者互相阻塞。长时间执行JavaScript将会造成UI停顿不响应。为了解决这个问题，HTML5提供了WebWorker API。WebWorker允许创建工作线程并在后台运行，使得一些阻塞较为严重的计算不影响主线程上的UI渲染。它的API如下所示：

```js
var worker = new Worker('worker.js');
worker.onmessage = function (event) {
  document.getElementById('result').textContent = event.data;
};

// 其中， worker.js如下：
var n = 1;
search: while (true) {
  n += 1;
  for (var i = 2; i <= Math.sqrt(n); i++) {
    if(n % i == 0) {
      continue search;
      // found a prime
      postMessage(n);
    }
  }
}
```

主进程与工作进程之间通过`onmessage()`和`postMessage()`进行通信，子进程对象则由`send()`方法实现主进程向子进程发送数据，message事件实现收听子进程发来的数据，与API在一定程度上相似。通过消息传递内容，而不是共享或直接操作相关资源，这时较为轻量和无依赖的做法。Node种对应示例如下：

```js
// parent.js

var cp = require('child_process');
var n = cp.fork(__dirname + './sub.js');

n.on('message', function (m) {
  console.log('PARENT go message:', m);
});

n.send({hello: 'world'});

// sub.j
process.on('message', function (m) {
  console.log('CHILD got message: ', m);
});

process.send({foo: 'bar'});
```

通过fork()或者其它APiece，创建子进程之后，为了实现父子进程之间的通信，父进程与子进程之间将会创建IPC通道。通过IPC通道，父子进程才能通过message和send()传递消息。

- 进程间通信原理

IPC的全称是Inter-Process Communication，即进程间通信。进程间通信的目的是为了让不同的进程能够互相访问资源并进行协调工作。实现进程间通信的技术由很多，如命名管道、匿名管道、socket、信号量、共享内存、消息队列、Domain Socket等。Node种实现IPC通道的是管道（pipe)技术。但此管道非彼管道，在Node种管道是个抽象层的称呼，具体实现细节由libuv提供，在Window下由命名管道（named pipe）实现，\*nix系统则采用Unix Domain Socket实现。表现在应用蹭上的进程间通信只有简单的message事件和send()方法，接口十分简洁和消息化。

父进程在实际创建子进程之前，会创建IPC通道并监听它，然后才真正创建出子进程，并通过环境变量（NODE_CHANNEL_FD）告诉子进程这个IPC通道的文件描述符。子进程在启动的过程中，根据文件描述符去连接这个已经存在的IPC通道，从而完成父子进程的连接。

建立连接之后，父子进程就可以自由地通信了。由于IPC通道是用命名管道或Domain Socket创建的，它们与网络socket的行为比较类似，属于双向通信。不同的是它们在系统内核种就完成了进程间的通信，而不用经过实际的网络层，非常高效。在Node中，IPC通道被抽象为Stream对象，在调用send()时发送数据（类似于write()），接受到的消息会通过message事件（类似于data）触发给应用层。

> 只有启动的子进程是Node进程时，子进程才会根据环境变量去连接IPC通道，对于其它类型的子进程则无法实现进程间的通信，除非其它进程也按约定去连接这个已经创建好的IPC通道。

### 9.2.3 句柄传递

建立好进程之间的IPC后，如果仅仅只用来发送一些简单的数据，显然不够我们的实际应用使用。还记得本章第一部分代码需要将启动的服务器分别监听各自的端口么，如果让服务都监听到相同的端口，将会有什么样的结果？

```js
var http = require('http');
http.createServer(function (req, res) {
  res.writeHead(200, {'Content-Type': 'text/plain'});
  res.end('Hello World\n');
}).listen(8888,'127.0.0.1');
```

再次启动master.js，将报错。这时只有一个工作进程能够监听到该端口，其余的进程在监听的过程中都抛出了EADDRINUSE异常，这是端口被专用的情况，新的进程不能继续监听该端口了。这个问题破坏了我们将多个进程监听同一个端口的想法。要解决这个问题，通常的做法是让每个进程监听不同的端口，其中主进程监听著端口（如80），主进程堆外接受所有的网络请求，再将这些请求分别代理到不同的端口的进程上。

通过代理，可以避免端口不能重复监听的问题，甚至可以在代理进程上做适当的负载均衡，使得每个子进程可以较为均衡地执行任务。由于进程每接收到一个请求，将会用掉一个文件描述符，因此代理方案中客户端连接到代理进程，代理进程连接到工作进程的过程需要用掉两个文件描述符。操作系统的文件描述符是有限的，代理方案浪费掉一倍数量的文件描述符的做法影响了系统的扩展能力。

为了解决上述这样的问题，Node在版本v0.5.9引入了进程间发送句柄的功能。send()方法除了能通过IPC发送数据外，还能发送句柄，第二个可选参数就是句柄：`child.send(message[, sendHanle])`。

那什么是句柄？句柄是一种可以用来标识资源的引用，它的内部包含了指向对象的文件描述符。比如句柄可以用来标识一个服务器端的socket对象、一个客户端soket对象、一个UDP套接字、一个管道等。

发送句柄意味着什么？在前一个问题中，我们可以去掉代理这种方案，使主进程接受到socket请求后，将这个socket直接发送给工作进程，而不是重新与工作进程之间建立新的socket连接来转发数据。文件描述符浪费的问题可以通过这样的方式轻松解决。来看看我们的示例代码：

```js
//主进程代码
var child = require('child_process').fork('child.js');

// open up the server object and send the handle
var server = require('net').createServer();
server.on('connection', function (soket) {
  socket.end('handled by parent\n');
});

server.listen(1337, function () {
  child.send('server', server);
});

// 子进程代码：
process.on('message', function (m, server) {
  if(m === 'server') {
    server.on('connection', function (socket) {
      soket.end('handled by child\n');
    });
  }
});
```

这个示例中，直接将一个TCp服务器发送给子进程。这是看起来不可思议的事情，我们先来测试一番，看看效果如何：

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
var cp = require('child_process');

var child1 = cp.fork('child.js');
var child2 = cp.fork('child.js');

// Open up server Object and send the handle

var server = require('net').createServer();
server.on('connection', function (socket) {
  socket.end('handled by parent\n');
});

server.listen(1337, function () {
  child1.send('server', server);
  child2.send('server', server);
});

// 然后在子进程中将进程ID打印出来：

// child.js
process.on('message', function (m, server) {
  if(m === 'server') {
    server.on('connection', function (socket) {
      socket.end('handled by child, pid is ' + process.pid + '\n');
    });
  }
});
```

再用curl测试我们的服务：

```sh
curl 'http://127.0.0.1:1337/'
# => handled by child pid is 24673
curl 'http://127.0.0.1:1337'
# => handled by parent
curl 'http://127.0.0.1:1337'
# => handled by child, pid is 24672

```

测试的结果是每次出现的结果都可能不同，如果可能被父进程处理，也可能被不同的子进程处理。并且这是在TCP层面上完成的事情，我们尝试将其转换到HTTP层面来实时。对于主进程而言，我们甚至可以让他更轻量一点，那么是否将服务器句柄发送给子进程之后，就可以关闭服务器的监听，让子进程来处理请求呢？

我们对主进程进行改动：

```js
var cp = require('child_process');

var child1 = cp.fork('child.js');
var child2 = cp.fork('child.js');

// Open up the server object and send the handle
var server = require('net').createServer();
server.listen(1337, function () {
  child1.send('server', server);
  child2.send('server', server);
  // 关闭
  server.close();
});

// 子进程进行改动：
var http = require('http');
var server = http.createServer(function (req, res) {
  res.writeHead(200, {'Content-Type': 'text/plain'});
  res.end('handled by child, pid is ' + process.pid + '\n');
});

process.on('message', function (m, tcp) {
  if(m === 'server') {
    tcp.on('connection', function(socket) {
      server.emit('connection', soket);
    });
  }
});
```

重启parent.js后，再次测试：

```sh
curl "http://127.0.0.1:1337/"
# => handled by child, pid = 24852
curl "http://127.0.0.1:1337/"
# => handled by child, pid = 24851
```

这样一来，所有的请求都由子进程处理了。整个过程中，服务的过程发生了依次改变。

我们神奇地发现，多个子进程可以同时监听相同端口，在没有EADDRINUSE异常发生了。

#### 1.句柄发送与还原

上文介绍地虽然是句柄发送，但是仔细看看，句柄发送跟我们直接将服务器对象发送给子进程有没有差别？它是否真的将服务器对象发送给子进程？为什么它可以发送到多个子进程中？发送给子进程为什么父进程中还存在这个对象？本节将解开这些秘密地所在。

目前子进程对象send()方法可以发送地句柄类型包括以下几种：

- net.Socket。TCP套接字。
- net.Server。TCP服务器，任意建立在TCP服务上的应用层服务都可以享受到它带来的好处。
- net.Native。C++层面的TCP套接字或IPC管道。
- dgram.Socket。UDP套接字。
- dgram.Native。C++层面的UDP套接字。

send()方法在将消息发送到IPC管道前，将消息组装成两个对象，一个参数是handle，另一个参数是message。message参数如下：

```js
{
  cmd: 'NODE_HANDLE',
  type: 'net.Server',
  msg: message
}
```

发送到IPC管道中的实际上是我们要发送的句柄文件描述符，文件描述符实际上是一个整数值。这个message对象在写入到IPC管道时也会通过JSON.stringify()进行序列化。所以最终发送到IPC通道的信息都是字符串，send()方法能发送消息和句柄并不意味着他能发送任意对象。

连接IPC通道的子进程可以读取到父进程发来的消息，将字符串通过JSON.parse()解析还原为对象后，才触发message事件将消息体传递给应用层使用。在这个过程中，消息对象还被进行过滤处理，message.cmd的值如果以NODE_为前缀，它将响应一个内部事件internalMEssage。


测试