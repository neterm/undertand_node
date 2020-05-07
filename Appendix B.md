# 调试 Node

---

- [B.1 Debugger](#b1-debugger)
- [B.2 Node Inspector](#b2-node-inspector)
- [B.3 总结](#b3-总结)

---

JavaScript 作为 Node 的主要编程语言。在大多数的脚本语言中，调试是一项比较麻烦的事情，JavaScript 也不例外。在 Firefox 浏览器的 Firebug 插件出现之前，主流的 JavaScript 调试方式是在代码中编写 alert()，这种糟糕的调试体验之前存在了很久。对于 Node 而言，调试的方式则不会像早期 Web 开发那么糟糕。这篇附录将会介绍 Node 开发中主要的几种调试方式。

## B.1 Debugger

Node 的调试直接受益于 V8。V8 提供了标准的调试 API，使得可以从进程内部进行调试。同时还提供了基于该 API 的 TCP 调试协议，使得通过调试协议，可以从进程外进行代码调试。Node 内建了调试协议的客户端，所以在启动时带上 debug 参数就可以实现对 JavaScript 代码的调试。

在进行调试前，需要通过`debugger;`语句在代码中设置断点，这样在执行时代码就会形成中断。以下为断点设置示例：

```js
// myscript.js

x = 5
setTimeout(function () {
  debugger
  console.log('world')
}, 1000)

console.log('hello')
```

执行上述代码时，在命令行中加入 debug。添加 debug 在命令中后，Node 会开启调试功能，内建的客户端会与 V8 建立连接。

Node 的调试客户端并没有支持 V8 的所有命令，只有简单的步进和检查的命令。

其中步进指令主要有如下几个：

- cont 或 c。继续执行
- next 或 n。执行到下一个断点。
- step 或 s。步进到函数内部。
- out 或 o。从函数内部跳出。
- pause。暂停执行。

通过断点进入交互提示后，可以通过步进指令指令逐方法地调试。

通过步进指令，还可以继续设置断点。V8 提供了如下几种设置断点和清除断点的方法：

- setBreakpoint()或 sb()。在当前行设置断点。
- setBreakpoint(line)或 sb(line)。在指定行设置断点。
- setBreakpoint('fn()')或 sb(...)。在函数体的第一个声明处设置断点。
- setBreakpoint('script.js', 1)或 sb(...)。在指定文件的第 n 行设置断点。
- clearBreakpoint()或者 cb()。清除断点。

除了设置断点外，在中断后进行调试时，还可以查看一些信息。这些信息指令如下：

- backtrace 或 bt。打印当前执行情况下的堆栈信息。
- list(5)。列出当前上下文前后 5 行源代码。
- watch(expr)。添加表达式到观察列表，进行观察。
- unwatch(expr)。从观察列表移除对表达式观察。
- watchers。列出所有观察的表达式和值。
- repl。打开调试的交互，用于执行调试脚本的上下文。

V8 的调试功能除了命令行中通过 debug 可以启用外，对于已经运行的进程，可以通过向其发送 SIGUSR1 信号启用调试。假设通过如下命令启动了一个服务进程：

```js
node server.js
```

通过 ps 命令找出进程的 ID，然后对这个运行中的进程发送 SIGUSR1 信号：`kill -s USR1 10093`；在原有的进程下，可以看到接收到信号并启动调试客户端的提示信息。调试客户端启动后，可以通过浏览器访问`http://localhost:5858/`来进行调试。浙江引入我们下一个调试工具介绍——Node Inspector 工具就是在这个基础上实现的图形界面调试。

## B.2 Node Inspector

Node Inspector 工具是基于 Debugger 和 Blink 开发者工具创建的调试界面。在代码的调试功能方面，源自 Node 为 V8 内建的调试代理，界面交互功能则来自 Blink 的开发者工具。带有 Blink 开发者工具的浏览器有 Chrome、Opera。这意味着我们可以像调试浏览器中的 JavaScript 代码一样调试 Node 的 JavaScript 代码。

### B.2.1 安装 Node Inspector

在使用 Node Inspector 之前，需要通过 NPM 工具安装它为全局命令行工具，安装命令如下：

```sh
npm install -g node-inspector
```

### B.2.2 错误堆栈

使用 Node Inspector 必须线启用 Node 进程的调试模式。启用调试模式的方式在前文有过介绍，在命令行中使用 debug 或者通过发送 SIGUSR1 给 Node 进程即可启用调试模式。

启动 Node 进程调试后，就可以启动 Node Inspector 工具。Node Inspector 工具相当于在 Blink 开发者工具与 Node 进程的调试代理之间建立了联系。

命令行中输出了一些信息，这时可以打开带 Blink 开发者工具的浏览器访问`http://127.0.0.1:8080/debug?port=5858`开始真正的调试。

在 Sources 面板中可以选择具体的 JavaScript 脚本设置断点，后续的调试过程就跟在浏览器中调试 JavaScript 一样。

## B.3 总结

由于 Node 主要运行在服务器中，调试会引起执行终端，进而中断服务，不利于在有大访问量的情况下进行。调试只适合与开发阶段，并且由于过程略麻烦，不宜在开发阶段过于依赖。更好的方式是编写良好的单元测试和做合理的日志记录，这对于程序开发来说更轻量，信赖度更高。
