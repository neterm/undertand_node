# 异步编程

---

- [4.1 函数式编程](#41-函数式编程)
- [4.2 异步编程的优势与难点](#42-异步编程的优势与难点)
- [4.3 异步编程解决方案](#43-异步编程解决方案)
- [4.4 异步并发控制](#44-异步并发控制)
- [4.5 总结](#45-总结)

---

有异步 I/O，必有异步编程。

上一章节描述了 Node 如何通过事件循环实现异步，包括与各种 I/O 多路复用搭配实现的异步 I/O 以及与 I/O 无关的异步。Node 是首个将异步大规模带到应用层的平台，它从内在运行机制到 API 的涉及，无不透露出异步的气息来。异步的高性能为它带来了高度的赞誉，而异步编程也为其带来了部分的诋毁。

前述章节中亦描述过异步 I/O 在应用层面不流行的原因，那便是异步编程在流程控制中，业务表达不适合自然语言的线性思维习惯。较少人能适应直接面对事件驱动进行编程，维度对它熟悉的主要是 GUI 开发者，如前端工程师或 GUI 工程师。前端工程师习以为常并能够娴熟地处理各种 DOM 事件和浏览器中地事件。Ryan Dahl 偏好事件驱动，而 JavaScript 在浏览器中也正契合事件驱动地执行过程，这也使得前后端的 JavaScript 在执行原理和风格上都趋于一致。虽然语言执行在不同的环境，但除了宿主提供的 API 有所不同外，并不让人觉得是一门新语言。

V8 和异步 I/O 在性能上带来的提升，前后端 JavaScript 编程风格一致，是 Node 能够迅速成功并流行起来的主要原因。

## 4.1 函数式编程

在开始异步编程之前，先得知晓 JavaScript 现今的回调函数和深层嵌套的来龙去脉。在 JavaScript 中，函数（function）作为一等公民，使用上非常自由，无论调用它，或者作为参数，或者作为返回值均可。函数的灵活性是 JavaScript 比较吸引人的地方之一，它与古老的 Lisp 语言颇具渊源。JavaScript 在诞生之前，Brend Eich 借鉴了 Scheme 语言（Scheme 作为 Lisp 的派生），吸收了函数式编程的精华，将函数作为一等公民便是经典案例。

鉴于函数时编程在近年重新火热，而前端类图书中较少述及这部分知识，这里稍做补充，因为它是 JavaScript 异步编程的基础。

### 4.1.1 高阶函数

在通常的语言中，函数的参数只接收基本的数据类型或对象引用，返回值也只是基本数据类型和对象引用。下面的代码为常规的参数传递和返回：

```js
function foo(x) {
  return x;
}
```

高阶函数则是可以把函数作为参数，或是将函数作为返回值的函数，如下面的代码所示：

```js
function foo(x) {
  return function () {
    return x;
  };
}
```

高阶函数可以将函数作为输入或返回值，这个变化虽然细小，但是对于 C/C++语言而言，通过指针也可以达到相同的效果。但对于程序编写，高阶函数则比普通的函数要灵活许多。除了通常意义的函数调用返回外，还形成了一种后续传递风格（continuation Passing Style）的结果接收方式，而非单一的返回值形式。后续传递风格的程序编写将函数的业务重点从返回值转移到了回调函数中：

```js
function foo(x, bar) {
  return bar(x);
}
```

以上面的代码为例，对于相同的`foo()`函数，传入的`bar`参数不同，则可以得到不同的结果。一个经典的例子便是数组的`sort()`方法，它是一个货真价实的高阶函数，可以接收一个方法作为参数参与运算排序：

```js
var points = [40, 100, 1, 5, 25, 10];
points.sort(function (a, b) {
  return a - b;
});

// => [1, 5, 10, 25, 40, 100]
```

通过改动`sort()`方法的参数可以决定不同的排序方式，从这里可以看出高阶函数的灵活性来。结合 Node 提供的最基本的事件模块可以看出，事件的处理方式正是基于高阶函数的特性来完成的。在自定义事件实例中，通过为相同事件注册不同的回调函数，可以很灵活地处理业务逻辑。

```js
var emitter = new events.EventEmitter();
emitter.on('event_foo', function () {
  // TODO
});
```

本书时长提到事件可以十分方便地进行复杂业务逻辑地解耦，它其实受益于高阶函数。

高阶函数在 Javascript 中比比皆是，其中 ECMAScript 5 中提供地一些数组方法（forEach()、map()、reduce()、reduceRight()、filter()、every()、some()）十分经典。

### 4.1.2 偏函数用法

偏函数用法是指创建一个调用另外一个部分——参数或变量已经预置的函数——的函数的用法。这句话相对比较拗口，下面我们用实例来说明：

```js
var toString = Object.prototype.toString();

var isString = function (obj) {
  return toString.call(obj) === '[object String]';
};

var isFunction = function (obj) {
  return toString.call(obj) === '[object Function]';
};
```

在 JavaScript 中进行类型判断时，我们通常会进行类似于上述代码的方法定义。这段代码固然不复杂，只有两个函数的定义，但是里面存在的问题时我们需要重复去定义的一些相似的函数，如果有更多的`isXXX()`，就会出现更多的冗余代码。为了解决重复定义的问题，我们引入一个新函数，这个新函数可以如工厂一样批量创建一些类似的函数。在下面代码中，我们通过`isType()`函数预先指定 type 的值，然后返回一个新的函数：

```js
var isType = function (type) {
  return function (obj) {
    return toString.call(obj) === '[object ' + type + ']';
  };
};

var isString = isType('String');
var isFunction = isType('Function');
```

可以看出，引入`isType()`函数后，创建`isString()`、`isFunction()`函数就变得简单多了。这中通过指定部分参数来产生一个新的定制函数的形式就是偏函数。

偏函数的应用在异步编程中也十分常见，著名类库 Underscore 提供的`after()`方法即是偏函数应用，其定义如下：

```js
_.after = function (times, func) {
  if (times <= 0) return func();

  return function () {
    if (--times < 1) return func.apply(this, arguments);
  };
};
```

这个函数可以根据传入的 times 参数和具体方法，生成一个需要调用多次才真正执行实际函数的函数。

## 4.2 异步编程的优势与难点

曾经的单线程模型在同步 I/O 的影响下，由于 I/O 调用缓慢，在应用层面导致 CPU 与 I/O 无法重叠进行。为了照顾编程人员的阅读思维习惯，同步 I/O 盛行了很多年。在日新月异的技术大潮面签，性能问题摆在了编程人员面签。提升性能的方式过去多用多线程的方式解决，但是多线程的引入在业务逻辑方面制造的麻烦也不少。从操作系统调度多线程的上下文切换开销，到实际编程里的锁、同步等问题，让开发人员头疼的时候也并不少。另一个解决 I/O 性能的方案是通过 C/C++调用操作系统底层接口，自己手工完成异步 I/O，这个能够达到很高的性能，但是调试和开发的门槛却很高，在帮助业务解决问题上，需要花费较大的精力。Node 利用 JavaScript 及其内部异步库，将异步直接提升到业务层面，这是一种创新。

### 4.2.1 优势

Node 带来的最大特性莫过于基于事件驱动的非阻塞 I/O 模型，这是它的灵魂所在。非阻塞 I/O 可以使 CPU 与 I/O 并不互相依赖等待，让资源得到更好的利用。对于网络应用而言，并行带来的想象空间更大，延展而开的是分布式和云。并行使得各个单点之间能够更有效地组织起来，这也是 Node 在云计算厂商中广受青睐地原因。

如果采用传统的同步 I/O 模型，分布式计算中性能的折扣很明显。

通过学习 Node 实现异步 I/O 的原理，利用事件循环的方式，JavaScript 线程像一个分配任务和处理结果的大管家，I/O 线程池里的各个 I/O 都是小二，负责兢兢业业地完成分配来地任务，小二与管家之间互不依赖，所以可以保证整体地高效率。这个利用事件循环地经典调度方式在很多地方都存在应用，最典型的是 UI 编程，如 iOS 应用开发等。

这个模型的缺点则在于管家无法承担过多细节性的任务，如果承担太多，则会影响到任务的调度，管家忙个不停，而小二却得不到活干，结局则是整体效率的降低。

换言之，Node 是为了解决编程模型中阻塞 I/O 的性能问题的，采用了单线程模型，这导致 Node 更像一个处理 I/O 密集问题的能手，而 CPU 密集型则取决于管家的能耐如何。

在第一章中，从斐波那契数列数列计算的测试结果中可以看出，这个管家的具体能力如何。如果形象地去评判的话，C 语言的性能至尊，得益于 V8 性能的 Node 则是一流武林高手，在具备武功秘籍的情况下（调用 C/C++扩展模块），Node 的能力可以逼近顶尖之列。

由于事件循环模型需要应对海量请求，海量请求同时作用在单线程上，就需要防止任何一个计算耗费过多 CPU 时间片。至于计算密集型，还是 I/O 密集型，只要计算不影响异步 I/O 的调度，那就构不成问题。建议对 CPU 的好用不要超过 10ms，或者将大量的计算分解成诸多小量计算，通过`setImmediate()`进行调度。只要合理利用 Node 的异步模型与 V8 的高性能，就可以充分发挥 CPU 和 I/O 资源的优势。

### 4.2.2 难点

Node 令异步编程如此风行，这也是异步编程首次大规模出现在业务层面。它借助异步 I/O 模型及 V8 高性能引擎，突破单线程的性能瓶颈，让 JavaScript 在后端达到实用价值。另一方面，它也同意了前后端 JavaScript 的编程模型。对于异步编程带来的新鲜感与不适感，开发者们有着不同程度的感受。接下来，我们梳理下异步编程的难点，以便更好的利用 Node。

#### 1. 难点 1：异常处理

过去我们处理异常时，通常实用类 Java 的`try/catch/final`语句块进行异常捕获，示例代码如下：

```javascript
try {
  JSON.parse(json);
} catch (e) {
  // TODO
}
```

但是这对于异步编程而言并不一定适用。第三章中提到过，异步 I/O 的实现主要包含两个阶段：提交请求和处理请求。这两个阶段中间有事件循环的调度，两者彼此不关联。异步方法则通常在第一个阶段提交请求后立即返回，因为异常并不一定发生在这个阶段，`try/catch`的功效在此处不会发挥任何作用。异步方法的定义如下：

```js
var async = function (callback) {
  process.nextTick(callback);
};
```

调用`async()`方法后，`callback`被存放起来，知道下一个事件循环（Tick）才会取出来执行。尝试对异步方法进行`try/catch`操作只能捕获当次事件循环内的异常，对`callback`执行时抛出的异常将无能为力。

```js
try {
  async(callback);
} catch (e) {
  // TODO
}
```

Node 在处理异常形成了一种约定，将异常作为回调函数的第一个参数传回，如果为空值，则表明异步调用没有异常抛出：

```js
async(function (err, results) {
  //TODO
});
```

在我们自行编写的异步方法上，也需要去遵循这样一些原则：

- 原则一：必须执行调用者传入的回调函数；

- 原则二：正确传递异常供调用者判断。

```javascript
var async = function (callback) {
  var async = function (callback) {
    process.nextTick(function () {
      var results = something;
      if (error) {
        return callback(error);
      }
      callback(null, resutls);
    });
  };
};
```

在异步编程方法的编写中，另一个容易犯的错误是对用户传递的回调函数进行异常捕获，示例代码：

```js
try {
  req.body = JSON.parse(buf, options.reviver);
  callback();
} catch (err) {
  err.body = buf;
  err.status = 400;
  callback(err);
}
```

上述代码的意图是捕获`JSON.parse()`中可能出现的异常，但是却不小心包含了用户传递的回调函数。这意味着如果回调函数中有异常抛出，将会进入`catch()`代码块中执行，于是回调函数将被执行两次。这显然不是预期的情况，可能导致业务混乱。正确的捕获应当为：

```js
try {
  req.body = JSON.parse(buf, options.reviver);
} catch (err) {
  err.body = buf;
  err.status = 400;
  return callback(err);
}
callback();
```

在编写异步方法时，只要将异常正确地传递给用户地回调方法即可，无需过多处理。

#### 2. 函数嵌套过深

这或许时 Node 被人诟病最多地地方。在前端开发中，DOM 事件相对而言不会存在互相依赖或需要多个事件一起协作地场景，较少存在异步多级依赖地情况。下面地代码为彼此独立地 DOM 事件绑定：

```js
$(selector).click(function (event) {
  // TODO
});
$(selector).change(function (event) {
  // TODO
});
```

但是对于 Node 而言，事务中存在多个异步调用地场景比比皆是。比如一个遍历目录地操作，其代码：

```js
fs.readdir(path.join(__dirname, '..'), function (err, files) {
  files.forEach(function (filename, index) {
    fs.readFile(filename, 'utf8', function (err, file) {
      // TODO
    });
  });
});
```

对于上述场景，两次操作存在依赖关系，函数嵌套地行为也许情有可原。那么，在网页渲染地过程中，通常需要数据、模板、资源文件，这三者互相之间并不依赖，但是最终渲染结果三者缺一不可。如果采用默认地异步方法调用，程序也许是：

```js
fs.readFile(template_path, 'utf8', function (err, template) {
  db.query(sql, function (err, data) {
    l1On.get(function (err, resources) {
      // TODO
    });
  });
});
```

这样在结果的保证上是没有问题的，问题在于这并没有利用好异步 I/O 带来的并行优势。这是异步编程的典型问题，为此有人曾说，因为嵌套的深度，未来最难看的代码必将从 Node 诞生。但是实际情况没有想象得那么糟糕，且看后面如何解决该问题。

#### 3. 阻塞代码

对于进入 JavaScript 世界不久得开发者，比较纳闷这门编程语言竟然没有`sleep()`这样得线程沉睡功能，唯独能用于延时操作得只有`setInterval()`和`setTimeout`这两个函数。但是让人惊讶的是，这两个函数并不能阻塞后续代码的继续执行。所以，有多半的开发者会写下下述这样的代码来实现`sleep(1000)`的效果：

```js
var start = new Date();
while (new Date() - start < 1000) {
  // TODO
}
// 需要阻塞的代码
```

但是事实是糟糕的，这段代码会持续占用 CPU 进行判断，与真正的线程沉睡相去甚远，完全破环了事件循环的调度。由于 Node 是单线程的原因，CPU 资源全部会用于为这段代码服务，导致其余任何请求都会得不到响应。

遇见这样的需求，在统一规划业务逻辑后，调用`setTimeout()`的效果更好。

#### 4. 多线程编程

我们谈论 JavaScript 的时候，通常谈的是单一线程上执行的代码，这在浏览器中指的是 JavaScript 执行线程与 UI 渲染共用的一个线程；在 Node 中，只是没有 UI 渲染的部分，模型基本相同。对于服务器而言，如果服务器是多核 CPU，单个 Node 进程实质上是没有充分利用多核 CPU 的。随着现今业务的复杂度，对于多核 CPU 利用的要求也越来越高。浏览器提出了 Web Workers，它通过将 JavaScript 执行与 UI 渲染分离，可以很好地利用多核 CPU 为大量计算服务。同时前端 Web Workers 也是一个利用消息机制合理适用多核 CPU 的理想模型。

遗憾在于前端浏览器存在对标准的滞后性，Web Workers 并没有广泛应用起来。另外 Web Workers 能解决利用 CPU 和减少阻塞 UI 渲染，但是不能解决 UI 渲染的效率问题。Node 借鉴了这个模式，`child_process`是其基础 API，`cluster`模块是更深层次的应用。借助 Web Workers 的模式，开发人员要更多地去面临跨线程的编程，这对于以往的 JavaScript 编程经验是较少考虑的。在第九章中，我们将详细分析 Node 的进程，以展开这部分内容。

#### 5. 异步转同步

习惯异步编程的同学，也许能够从容面对异步编程带来的副产品，比如嵌套回调、业务分散等问题。Node 提供给了绝大部分的异步 API 和少量的同步 API，偶尔出现的同步需求将会因为没有同步 API 让开发者无所适从。目前，Node 中视图同步式编程，但并不能得到原生支持，需要借助库或者编译等手段来实现。但对于异步调用，通过良好的流程控制，还是能够将逻辑梳理称顺序式的形式。

## 4.3 异步编程解决方案

前面列举了因异步编程带来的一些问题，与异步编程提升性能成果相比，编程过程看起来似乎没有想象中那么美好，但是事实却也没有那么糟糕。与问题相比，解决问题的方案总是更多，本节将展开各个典型的解决方案。

目前异步编程主要解决方案有如下 3 种：

- 事件发布/订阅模式
- Promise/Deferred 模式
- 流程控制库

### 4.3.1 事件发布/订阅模式

事件监听器模式是一种广泛用于异步编程的模式，是回调函数的事件化，又称发布/订阅模式。Node 自身提供的[events 模块](http://nodejs.org/docs/latest/api/events.html)是发布/订阅模式的一个简单实现，Node 种部分模块都继承自它，这个模块比前端浏览器种的大量 DOM 事件简单，不存在事件冒泡，也不存在`preventDefault()`、`stopPropagation()`和`stopImmediatePropagation()`等控制事件传递的方法。它具有`addListener/on()`、`once()`、`removeListener()`、`removeAllListeners()`和`emit()`等基本的事件监听模式的方法实现。事件发布/订阅模式的操作极其简单，示例代码如下：

```js
// 订阅
emitter.on('event1', function (message) {
  console.log(message);
});

// 发布
emitter.emit('event1', 'I am Message!');
```

可以看到，订阅事件就是一个高阶函数的应用。事件发布/订阅模式可以实现一个事件与多个回调函数的关联，这些回调函数又称为事件监听器。通过`emit()`发布事件后，消息会立即传递给当前事件的所有侦听器执行。侦听器可以很灵活地添加和删除，使得事件和具体处理逻辑之间可以很轻松地关联和解耦。

事件发布/订阅模式自身并无同步和异步调用问题，但是在 Node 种，`emit()`调用多半是伴随事件循环而异步触发地，所以我们说事件发布/订阅广泛应用于异步编程。

事件发布/订阅模式常常用来解耦业务逻辑，事件发布者无需关注订阅者地监听器如何实现业务逻辑，甚至不用关注有多少个侦听器存在，数据通过消息地方式可以很灵活地传递。在一些经典场景中，可以通过事件发布/订阅模式进行组件封装，将不变的部分封装在组件内部，将容易变化、需要自定义的部分通过事件暴露给外部处理，这是一种典型的逻辑分离方式。在这种事件发布/订阅式组件中，事件的设计非常重要，因为它关乎外部调用组件时是否优雅，从某种角度来说事件的设计就是组件的接口设计。

从另一个角度来看，事件监听器模式也是一种钩子（hook）机制，利用钩子导出内部数据或状态给外部调用者。Node 中很多对象大多具有黑盒的特点，功能点少，如果不通过事件钩子的形式，我们就无法获得对象在运行期间的中间值或内部状态。这种通过事件钩子的方式，可以适编程者不用关注组件是如何执行的，只需要关注在需要的事件点上即可。下面的 HTTP 请求是典型场景：

```js
var options = {
  host: 'hq.sinajs.cn',
  port: '/?list=sh601006',
  method: 'GET',
};
var req = http.request(options, function (res) {
  console.log('STATUS: ' + res.statusCode);
  console.log('HEADERS: ' + JSON.stringify(res.headers));
  res.setEncoding('utf8');
  res.on('data', function (chunk) {
    console.log('BODY: ' + chunk);
  });
  res.on('end', function () {
    // TODO
  });
});

req.on('error', function (e) {
  console.log('problem with request: ' + e.message);
});

req.write('data \n');
req.write('data \n');
req.end();
```

在这段 HTTP 请求的代码中，程序员只需要将实现放在`error`、`data`、`end`这些业务事件点上即可，至于内部的流程如何，无需过于关注。

值得一提的是，Node 对事件发布/订阅的机制做了一些额外的处理，这大多是基于健壮性而考虑的。下面为两个具体细节点：

- 如果对一个事件添加超过 10 个侦听器，将会得到一条警告。这一处设计与 Node 自身单线程运行有关，设计者认为侦听器太多可能导致内存泄漏，所以存在这样一条警告。调用`emitter.setMaxLinsteners(0);`可以将这个限制去掉。另一方面，由于事件发布会引起一系列侦听器执行，所以事件相关的侦听器过多，可能存在过多占用 CPU 的场景。
- 为了处理异常，`EventEmitter`对象对`error`事件进行了特殊对待。如果运行期间的错误触发了`error`事件，`EventEmitter`会检查是否有对`error`事件添加过侦听器。如果添加了，这个错误将会交由该侦听器处理，否则这个错误会作为异常抛出。如果外部没有捕获这个异常，将会引起线程退出。一个健壮的`EventEmitter`实例应该对`error`事件做处理。

#### 1.继承 events 模块

实现一个继承`EventEmitter`的类是十分简单的，以下代码是 Node 中`Stream`对象继承`EventEmitter`的例子：

```js
var events = require('events');

function Stream() {
  events.EventEmitter.call(this);
}
util.inherits(Stream, events.EventEmitter);
```

Node 在`util`模块中封装了继承的方法，所以此处可以很便利地调用。开发者可以通过这样地方式轻松继承`EventEmitter`类，利用事件机制解决业务问题。在 Node 提供地核心模块中，有近半数都继承自`EventEmitter`。

#### 2.利用事件队列解决雪崩问题

在事件订阅/发布模式中，通常也有一个`once()`方法，通过它添加的侦听器只能执行一次，在执行之后就会将它与事件的关联移除。这个特性常常可以帮助我们过滤一些重复性的事件响应。下面我们介绍下如何采用`once`解决雪崩问题。

在计算机中，缓存由于放在内存中，访问速度十分快，常常用于加倍数据访问，让绝大多数的请求不必重复去做一些低效的数据读取。所谓雪崩问题，就是在高访问量、大量并发量的情况下，缓存失效的情景，此时大量的请求同时涌入数据库中，数据库无法同时承受如此大的查询请求，进而往前影响网站整体的响应速度。

```js
var select = function (callback) {
  db.select('SQL', function (results) {
    callback(results);
  });
};
```

如果站点刚好启动，这是缓存中是不存在数据的，而访问量巨大，同一句 SQL 会被发送到数据库中反复查询，会影响服务的整体性能。一种改进方案是添加一个状态锁，相关代码：

```js
var status = 'ready';
var select = function (callback) {
  if (status == 'ready') {
    status = 'pending';
    db.select('SQL', function (results) {
      status = 'ready';
      callback(results);
    });
  }
};
```

但在这种情况下，连续地多次调用`select`时，只有第一次调用是生效地，后续地`select()`是没有数据服务的，这个时候可以引入事件队列，相关代码：

```js
var proxy = new events.EventEmitter();
var status = 'ready';
var select = function (callback) {
  proxy.once('selected', callback);
  if(status == 'ready') {
    status = 'pending';
    db.select('SQL', function (results) {
      proxy.emit('selected', results);
      status = 'ready';
    });
};
```

这里我们利用了`once()`方法，将所有请求地回调压入事件队列中，利用其执行一次就会将监视器移除的特点，保证每一个回调都只会被执行一次。对于相同的 SQL 语句，保证在同一个查询开始到结束的过程中永远只有一次。SQL 在进行查询时，新到来的相同调用只需要在队列中等待数据就绪即可，一旦查询结束，得到的结果可以被这些调用共同使用。这种方式能节省重复的数据库调用产生的开销。由于 Node 单线程执行的原因，此处无需担心状态同步问题。这种方式其实也可以应用到其它远程调用的场景中，即使外部没有缓存策略，也能有效节省重复开销。

慈湖可能因为存在侦听器过多引发的警告，需要调用`setMaxListeners(0);`移除警告，或者设置更大的警告阈值。`once`方法产生的效果，也可以在著名的 Gearman 异步应用框架中实现。但在 JavaScript 中实现这个效果十分容易。

#### 3.多异步之间的协作方案

事件发布/订阅模式有着它的优点。利用高阶函数的优势，侦听器作为回调函数可以随意添加和删除，它帮助开发者轻松处理随时可能添加的业务逻辑。也可以隔离业务逻辑，保持业务逻辑单元的职责单一。一般而言，事件与侦听器的关系是一对多，但是在异步编程中，也会出现事件与侦听器多对一的情况，也就是说一个业务逻辑可以依赖两个通过回调函数或事件传递的结果。前面提及嵌套过深的原因即是如此。

这里我们尝试通过原生代码解决“难点 2”中为了最终结果的处理而导致可以并行调用但实际只能串行执行的问题。我们的目标是既要享受异步 I/O 带来的性能提升，也要保持良好的编码风格。这里以渲染页面所需要的模板读取、数据读取和本地化资源读取为例，简单介绍下，相关代码：

```js
var count = 0;
var results = {};
var done = function (key, vaue) {
  results[key] = value;
  count++;
  if (count === 3) {
    // 渲染页面
    render(results);
  }
};

fs.readFile(template_path, 'utf8', function (err, template) {
  done('template', template);
});
db.query(sql, function (err, data) {
  done('data', data);
});
l1On.get(function (err, resources) {
  done('resources', resources);
});
```

由于多个异步场景中回调函数的执行并不能保证顺序，且回调函数之间互相没有任何交集，所以需要借助一个第三方函数和第三方变量来处理异步协作的结果。通常我们把这个用于检测次数的变量叫**哨兵变量**，聪明的你也许已经想到利用偏函数来处理哨兵变量与第三方函数的关系了：

```js
var after = function (times, callback) {
  var count = 0,
    results = {};
  return function (key, value) {
    results[key] = value;
    count++;
    if (count === times) {
      callback(results);
    }
  };
};
var done = after(times, render);
```

这种方案实现了多对一的目的。如果业务继续增长，我们依然可以继续利用发布/订阅方式来完成多对多的方案，相关代码：

```js
var emitter = new events.Emitter();
var done = after(times, render);
emitter.on('done', done);
emitter.on('done', other);

fs.readFile(template_path, 'utf8', function (err, template) {
  emitter.emit('done', 'template', template);
});
db.query(sql, function (err, data) {
  emitter.emit('done', 'data', data);
});
l1On.get(function (err, resources) {
  emitter.emit('done', 'resources', resources);
});
```

这种方案结合了前者用简单的偏函数完成多对一的收敛和事件订阅/发布模式中一对多的发散。在上面的方法中，有一个令调用者不那么舒服的问题，就是调用者要去准备这个`done()`函数，以及在回调函数中把数据一个个提取出来，再进行处理。

另一个方案则是由笔者自己写的 EventProxy 模块，它是对事件订阅/发布模式的扩充，可以自由订阅组合事件。由于依旧采用的事件订阅/发布模式，与 Node 十分契合，相关代码：

```js
var proxy = new EventProxy();
proxy.all('template', 'data', 'resources', function (template, data, resources) {
  // TODO
});
fs.readFile(template_path, 'utf8', function (err, template) {
  proxy.emit('template', template);
});
db.query(sql, function (err, resources) {
  proxy.emit('template', resources);
});
l1On.get(function (err, resources) {
  proxy.emit('resources', resources);
});
```

EventProxy 提供了一个`all()`方法来订阅多个事件，当每个事情都被触发后，侦听器才会执行。另外的一个方法是`tail()`方法，它与`all()`方法的区别在于`all()`方法的侦听器在满足条件之后只会执行一次，`tail()`方法的侦听器则在满足条件时执行一次之后，如果组合事件中的某个事件中的某个事件被再次触发，侦听器回用最新的数据继续执行。

`all()`方法带来的另一个改进则是：在侦听器中返回数据的参数列表与订阅组合事件的事件列表是一致对应的。

除此之外，在异步的场景下，我们常常需要从一个接口多次读取数据，此时触发的事件名或许是相同的。EventProxy 提供了`after()`方法来实现事件在执行多少次后执行侦听器的单一事件组合订阅方式，示例代码：

```js
var proxy = new EventProxy();
proxy.after('data', 10, function (datas) {
  // TODO
});
```

这段代码表示执行 10 次 data 事件后，执行侦听器。这个侦听器得到的数据为 10 次按事件触发次序排序的数组。EventProxy 模块除了可以应用于 Node 中之外，还可以用在前端浏览器中。

#### 4.EventProxy 原理

EventProxy 来自于 Backnone 的事件模块，Backnone 的事件模块是 Model、View 模块的基础功能，在前端有广泛使用。它在每个非 all 事件触发时都会触发一次 all 事件，相关代码：

```js
// Trigger an event, firing all bound callbacks. Callbacks are passed the
// Same arguments as `trigger` is, apart from the event name.
// Listening for `"all"` passes the true event name as the first argument
trigger: function(eventName) {
  var list, calls, ev, callback, args;
  var both = 2;
  if(!(calls = this._callbacks)) return this;
  while(both--){
    ev = both ? eventName : 'all';
    if(list = calls[ev]) {
      for(var i = 0, l = list.length; i<l; i++) {
        if(!(callback = list[i])) {
          list.splice(i, 1);
          i--;
          l--;
        }else{
          args = both ? Array.prototype.slice.call(arguments, 1) : argumets;
          callback[0].apply(callback[1] || this, args);
        }
      }
    }
  }
  return this;
}
```

EventProxy 则是一个将 all 当作一个事件流的拦截层，在其中注入一些业务来处理单一事件无法解决的异步处理问题。类似的扩展还有`all()`、`tail()`、`after()`、`not()`和`any()`等。

#### 5.EventProxy 的异常处理

EventProxy 在事件发布/订阅模式的基础上还完善了异常处理。在异步方法中，异常处理需要占用一定比例的精力。在过去一段时间内，我们口试通过额外添加 error 事件来进行异常统一处理，大致代码：

```js
exports.getContent = function (callback) {
  var ep = new EventProxy();
  ep.all('tpl', 'data', function (tpl, data) {
    // 成功回调
    callback(null, {
      template: tpl,
      data: data,
    });
  });
  // 侦听error事件
  ep.bind('error', function (err) {
    // 卸载掉所有处理函数
    ep.unbind();
    // 异常回调
    callback(err);
  });

  fs.readFile('template.tpl', 'utf8', function (err, content) {
    if (err) {
      // 一旦发生异常，一律交给error事件的处理函数处理
      return ep.emit('error', err);
    }
    ep.emit('tpl', content);
  });
  db.get(sql, function (err, result) {
    if (err) {
      // 一旦发生异常，一律交给 error事件的处理函数处理
      return ep.emit('error', err);
    }
    ep.emit('data', result);
  });
};
```

因为异常处理的原因，代码量一下子躲起来了，而 EventProxy 在实践过程中改进了这个问题，相关代码：

```js
exports.getContent = function (callback) {
  var ep = new EventProxy();
  ep.all('tpl', 'data', function (tpl, data) {
    // 成功回调
    callback(null, {
      template: tpl,
      data: data,
    });
  });
  // 绑定错误处理函数
  ep.fail(callback);

  fs.readFile('template.tpl', 'utf8', dp.done('tpl'));
  db.get(sql, ep.done('data'));
};
```

在上述代码中，EventProxy 提供了`fail()`和`done()`这两个实例方法来优化处理异常，使得开发者将精力关注在业务部分，而不是在异常捕获上。

关于`faii()`方法的实现，可以参见以下变换：

```js
ep.fail(callback);
// 等价于
ep.fail(function (err) {
  callback(err);
});
// 又等价于
ep.bind('error', function (err) {
  // 卸载所有处理函数
  ep.unbind();
  // 异常回调
  callback(err);
});
```

而`done()`方法的实现，也可以参见以下变换：

```js
ep.done('tpl');

// 等价于
function (err, content) {
  if(err) {
    // 一旦发生异常，一律交给error事件的处理函数处理
    return ep.emit('error', err);
  }
  ep.emit('tpl',content);
}

// 同时，done() 方法也接收一个函数作为参数
ep.done(function (content) {
  // TODO
  // 这里无需考虑异常
  ep.emit('tpl',content);
});

// 等价于
function (err, content) {
  if(err) {
    // 一旦发生异常，一律交给 error 事件的处理函数处理
    return ep.emit('error', err);
  }
  (function (content) {
    // TODO
    // 这里无需考虑异常
    ep.emit('tpl',content);
  }(content));
}
```

当之传入一个回调函数时，需要手工调用`emit()`触发事件。另一个改进是同时传入事件名和回调函数，相关代码：

```js
ep.done('tpl', function (content) {
  return content;
});
```

在这种方式下，我们无须再回调函数中处理事件的触发，只需将处理过的数据返回即可。返回的结果将在`done()`方法中用作事件的数据而触发。

这里的`fail()`和`done()`十分类似于 Promise 模式中的`fail()`和`done()`。换句话而言，这可以算是事件发布/订阅模式向 Promise 模式的借鉴。这样的完善既提升了程序的健壮性，同时也降低了代码量。

### 4.3.2 Promise/Deferred 模式

使用事件的方式时，执行流程需要被预先设定。即便是分支，也需要预先设定，这是由发布/订阅模式的运行机制决定的。下面为普通的 Ajax 调用：

```js
$.get('/api', {
  success: onSuccess,
  error: onError,
  complete: onComplete,
});
```

再上面的异步调用中，必须严谨地设置目标。那么是否有一种先执行异步调用，延迟传递处理地方式呢？答案是 Promise/Deferred 模式。

Promise/Deferred 模式在 JavaScript 框架中最早出现于 Dojo 的代码中，被广为所知则是来自于 jQuery 1.5 版本，该版本几乎重写了 Ajax 部分，使得调用 Ajax 时可以通过如下方式进行：

```js
$.get('/api').success(onSuccess).error(onError).complete(onComplete);
```

这使得即使不调用`sucess()`、`error()`等方法，Ajax 也会执行，这样的调用方式比预先传入回调让人觉得舒适一些。在原始的 API 中，一个事件只能处理一个回调，而通过 Deferred 对象，可以对使劲按加入任意的业务处理逻辑，示例代码：

```js
$.get('/api').success(onSuccess1).success(onSuccess2);
```

Promise/Deferred 模式在 2009 年时被 Kris Zyp 抽象为一个提议草案，发布在 CommonJS 规范中。随着使用 Promise/Deferred 模式的应用逐渐增多，CommonJS 草案目前已经抽象出了 Promises/A、Promises/B、Promises/D 这样经典的异步 Promise/Deferred 模型，这使得异步操作可以以一种优雅的方式出现。

异步的广度使用使得回调、嵌套的出现，但是一旦出现深度的嵌套，就会让编程的体验变得不愉快，而 Promise/Deferred 模式在一定的程度上缓解了这个问题。这里我们将着重介绍 Promise/A 来以点代面介绍 Promise/Deferred 模式。

#### 1. Promises/A

Promise/Deferred 模式其实包含两部分，即 Promise 和 Deferred。这里暂且不提两者的区别是什么，先来看看 Promises/A 的行为把。

Promises/A 提议对单个异步操作做出了这样的抽象定义，具体如下

- Promise 操作只会处于 3 中状态中的一种：未完成态、完成态和失败态。
- Promise 的状态只会出现从未完成态向完成态或失败态转化，不能逆反。完成态和失败态不能相互转换。
- Promise 的状态一旦转化，将不能再被更改。

在 API 的定义上，Promises/A 提议是比较简单的。一个 Promise 对象只要具备`then()`方法即可。但是对于`then()`方法，有以下简单的要求。

- 接收完成态、失败态的回调方法。在操作完成或出现错误时，将会调用对应方法。
- 可选地支持 progress 事件回调作为第三个方法。
- `then()`方法只接收`function`对象,其余对象将被忽略。
- `then()`方法继续返回`Promise`对象，以实现链式调用。

`then()`方法定义如下：

```js
then(fulfilledHandler, errorHandler, progressHandler);
```

为了演示 Promises/A 提议，这里我们尝试通过继承 Node 地 events 模块来完成一个简单地实现，代码如下：

```js
var Promise = function () {
  EventEmitter.call(this);
};
util.inherits(Promise, EventEmitter);

Promise.prototype.then = function (fulfilledHandler, errorHandler, progressHandler) {
  if (typeof fulfilledHandler === 'function') {
    // 利用once()方法，保证成功回调只执行一次
    this.once('success', fulfilledHandler);
  }
  if (typeof errorHandler === 'function') {
    // 利用once方法，保证异常回调只执行一次
    this.once('error', errorHandler);
  }
  if (typeof progressHandler === 'function') {
    this.on('progress', progressHandler);
  }
  return this;
};
```

这里看到`then()`方法所做的事情是将回调函数存放起来。为了完成整个流程，还需要触发执行这些回调函数的地方，实现这个功能地对象被称为 Deferred，即延迟对象，示例代码：

```js
var Deferred = function () {
  this.state = 'unfulfilled';
  this.promise = new Promise();
};

Deferred.prototype.resolve = function (obj) {
  this.state = 'fulfilled';
  this.promise.emit('success', obj);
};

Deferred.prototype.reject = function (err) {
  this.state = 'failed';
  this.promis.emit('error', err);
};

Deferred.prototype.progress = function (data) {
  this.promise.emit('progress', data);
};
```

利用 Promiese/A 提议的模式，我们可以对一个典型的响应对象进行封装，相关代码：

```js
res.setEncoding('utf8');
res.on('data', function (chunk) {
  console.log('BODY: ' + chunk);
});
res.on('end', function () {
  // Done
});
res.on('error', function (err) {
  // Error
});
// 上述代码转换为如下简略形式
res.then(
  function () {
    // Done
  },
  function (err) {
    // Error
  },
  function (chunk) {
    console.log('BODY: ' + chunk);
  }
);
```

要实现如此简单地 API，只需要简单改造以下即可：

```js
var promisify = function (res) {
  var deferred = new Deferred();
  var result = '';
  res.on('data', function (chunk) {
    result += chunk;
    deferred.progress(chunk);
  });
  res.on('end', function () {
    promise.resolve(result);
  });
  res.on('error', function () {
    promise.reject(err);
  });
  return deferred.promise;
};
```

如此就得到了简单的结果。这里返回`deferred.promise`的目的时为了不让外部程序调用`resolve()`和`reject()`方法，更改内部状态的行为交给定义者处理。下面为定义好 Promise 后的调用示例：

```js
promisify(res).then(
  function () {
    // Done
  },
  function (err) {
    // Error
  },
  function (chunk) {
    console.log('BODY: ' + chunk);
  }
);
```

这里回到 Promise 和 Deferred 的差别上。从上面的代码可以看出，Deferred 主要用于内部，用户维护异步模型的状态；Promise 则作用于外部，通过`then()`方法暴露给外部以添加自定义逻辑。

与事件发布/订阅模式相比，Promise/Deferred 模式的 API 接口和抽象模型都十分简洁。它将业务中不可变的部分封装再 Deferred 中，将可变的部分交给 Promise。此时问题就来了，对于不同的场景，都需要去封装和改造起 Deferred 部分，然后才能得到整洁的接口。如果场景中不常用，封装花费的事件带来的整洁相比并不划算。

Promise 是高级接口，事件是低级接口。低级接口可以构成更多更复杂的场景，高级接口一旦定义，不太容易变化，不再有低级接口的灵活性，但对于解决典型问题非常有效。Promises/A 的模型抽象在几种 Promise 提议中相对简洁。

这里再介绍一个 Q。Q 模块是 Promises/A 规范的一个实现，可以通过`npm install q`进行安装使用。它对 Node 中常见回调函数的 Promise 实现如下：

```js
defer.prototype.makeNodeResolver = function () {
  var self = this;

  return function (error, value) {
    if (error) {
      self.reject(error);
    } else if (arguments.length > 2) {
      self.resolve(array_slice(arguments, 1));
    } else {
      self.resolve(value);
    }
  };
};
```

可以看出这里是一个高阶函数的使用，`makeNodeResolver`返回一个 Node 风格的回调函数。对于`fs.readFile()`的调用，将会演化成：

```js
var readFile = function (file, encoding) {
  var deferred = Q.defer();
  fs.readFile(file, encoding, deferred.makeNodeResolver());
  return deferred.promise;
};

// 定义之后的调用：
readFile('foo.txt', 'utf-8').then(
  function (data) {
    // Success case
  },
  function (err) {
    // Failed case
  }
);
```

Promise 通过封装异步调用，实现了正向用例和反向用例的分离以及逻辑处理延迟，使得回调函数相对优雅。

前面分析了 Q 对 Node 异步回调的处理。事实上，异步编程中需要花费很多精力进行异常的判断和处理，为了分离异常和正常情况，我写了一个模块 memeda 用于处理`makeNodeResolver`相似的事情。下面调用示例可以看出，正常解雇哦和异常结果分离到两个函数中：

```js
var failing = require('memeda').failing;
fs.readFile(
  file,
  encoding,
  failing(function (err) {
    // TODO
  }).passing(function (data) {
    // TODO
  })
);
```

我们可以对 Q 和 memeda 模块略做比较。两者相似之处在于分离逻辑，使开发者侧重关注正常情况。不同之处在于 Q 通过`promise()`可以实现延迟处理，以及通过多次调用`then()`附加更多结果处理逻辑。可以看出，Promise 需要封装，但是很强大，具备很强的侵入性；纯粹的函数则较轻量，但功能相对弱小。

#### 2. Promise 中的多异步协作

在 Promise 的极少中说过，主要解决的是单个异步操作中存在的问题。回到我们的难点，当我们需要处理多个异步调用时，又该如何处理呢？

类似于 EventProxy，这里给出一个简单的原型实现，相关代码如下：

```js
Deferred.prototype.all = function (promises) {
  var count = promises.length;
  var that = this;
  var results = [];
  promises.forEach(function (promise, i) {
    promise.then(
      function (data) {
        count--;
        results[i] = data;
        if (count === 0) {
          that.resolve(results);
        }
      },
      function (err) {
        that.reject(err);
      }
    );
  });
  return this.promise;
};
```

对于多次文件的读写场景，以下面代码为例，`all()`方法将两个单独的 Promise 重新抽象组合成一个新的 Promise：

```js
var promise1 = readFile('foo.txt', 'utf-8');
var promise2 = readFile('bar.txt', 'utf-8');

var deferred = new Deferred();
deferred.all([promise1, promise2]).then(
  function (results) {
    // TODO
  },
  function (err) {
    // TODO
  }
);
```

这里通过`all()`方法抽象多个异步操作。只有所有异步操作成功，这个异步操作才算成功，一旦其中一个异步操作失败，整个异步操作就失败了。

#### 3.Promise 的进阶知识

在 API 的暴露上，Promise 模式比原始的事件侦听和触发略为优美，它的缺陷则是需要为不同的场景封装不同的 API，没有直接的原生事件那么灵活。但对于经典的场景，封装出 API 的成本也并不高，值得一做。

Promise 的秘诀其实在于对队列的操作。这里介绍一个实际的案例，我们处理自动化测试时，要跟远程服务器之间进行多次指令发送，这些指令时按顺序依次进行的。在 Node 中，网络库是完全异步的，无法在编程层面实现像其他语言那般的同步调用。由于网站界面通常都是由前端工程师完成的，用 JavaScript 编写自动化测试可以减轻他们切换环境的痛苦，所以不能因为无法同步调用就放弃 Node。解决同步调用问题的答案也就是采用 Deferred 模式。

现在有一组纯异步的 API，为了完成一串事情，我们代码大致如下：

```js
obj.api1(function (value1) {
  obj.api2(function (value2) {
    obj.api3(function (value3) {
      obj.api4(function (value4) {
        callback(value4);
      });
    });
  });
});
```

由于有按每个步骤依次执行的需要，所以必须嵌套执行。但那样我们会得到很难看的嵌套，超过 10 个连续嵌套就会让代码十分难看。于是我们得到了“Pyramid of Doom”，译为中文，是谓“恶魔金字塔”。相信初入 Node 世界的人，也写过不少类似的代码。

下面我们通过普通的函数将上面的代码尝试展开：

```js
var handler1 = function (value1) {
  obj.api2(value1, handler2);
};
var handler2 = function (value2) {
  obj.api3(value2, handler3);
};
var handler3 = function (value3) {
  obj.api4(value3, handler3);
};

var handler4 = function (value4) {
  callback(value4);
};

obj.api1(handler1);
```

对于喜欢利用事件的开发者，我们展开后的代码又将会是怎样的情况呢？具体如下所示：

```js
var emitter = new event.Emitter();

emitter.on('step1', function () {
  obj.api1(function (value1) {
    emitter.emit('step2', value1);
  });
});

emitter.on('step2', function (value1) {
  obj.api2(value1, function (value2) {
    emitter.emit('step3', value2);
  });
});

emitter.on('step3', function (value2) {
  obj.api3(value2, function (value3) {
    emitter.emit('step4', value3);
  });
});

emitter.on('step4', function (value3) {
  obj.api4(value3, function (value4) {
    callback(value4);
  });
});

emitter.emit('step1');
```

利用事件展开后的效果变得越来越糟糕了。与纯粹嵌套相比，代码量明显增加了，这显然不会带来良好的编程体验。为此，我们需要一种更好的方式。

- 支持序列执行的 Promise

  理想的编程体验应当是前一个调用结果作为下一个调用的开始，是传说中的链式调用，相关代码如下：

  ```js
  promise()
    .then(obj.api1)
    .then(obj.api2)
    .then(obj.api3)
    .then(obj.api4)
    .then(
      function (value4) {
        // Do something
      },
      function (error) {
        // Handle any error from step1 through step4
      }
    )
    .done();
  ```

  尝试改造下代码实现链式调用：

  ```js
  var Deferred = function () {
    this.promise = new Promise();
  };

  // 完成态
  Deferred.prototype.resolve = function (obj) {
    var promise = this.promise;
    var handler;

    while ((handler = promise.queue.shift())) {
      if (handler && handler.fulfilled) {
        var ret = handler.fulfilled(obj);
        if (ret && ret.isPromise) {
          ret.queue = promise.queue;
          this.promise = ret;
          return;
        }
      }
    }
  };

  // 失败态
  Deferred.prototype.reject = function (err) {
    var promise = this.promise;
    var handler;

    while ((handler = promise.queue.shift())) {
      if (handler && handler.error) {
        var ret = handler.error(err);
        if (ret && ret.isPromise) {
          ret.queue = promise.queue;
          this.promise = ret;
          return;
        }
      }
    }
  };

  // 生成回调函数
  Deferred.prototype.callback = function () {
    var that = this;
    return function (err, file) {
      if (err) {
        return that.reject(err);
      }

      that.resolve(file);
    };
  };

  var Promise = function () {
    // 队列用于存储待执行的回调函数
    this.queue = [];
    this.isPromise = true;
  };

  Promise.prototype.then = function (fulfilledHandler, errorHandler, progressHandler) {
    var handler = {};
    if (typeof fulfilledHandler === 'function') {
      handler.fulfilled = fulfilledHandler;
    }

    if (typeof errorHandler === 'function') {
      handler.error = errorHandler;
    }

    this.queue.push(handler);
    return this;
  };
  ```

  这里我们以两次文件读取作为例子，以验证该设计的可行性。这里假设读取第二个文件是依赖于第一个文件的内容，相关代码如下：

  ```js
  var readFile1 = function (file, encoding) {
    var deferred = new Deferred();
    fs.readFile(file, encoding, deferred.callback());
    return deferred.promise;
  };

  var readFile2 = function (file, encoding) {
    var deferred = new Deferred();
    fs.readFile(file, encoding, deferred.callback());
    return deferred.promise;
  };

  readFile1('template.txt', 'utf8')
    .then(function (file1) {
      return readFile2(file1.trim(), 'utf8');
    })
    .then(function (file2) {
      console.log(file2);
    });
  ```

  将这段代码存为`sequence.js`。执行该代码，将会得到：

  ```sh
  node sequence.js
  I am file2
  ```

  要让 Promise 支持链式执行，主要通过以下两个步骤。
  （1）将所有的回调都存到队列中。
  （2）Promise 完成时，逐个执行回调，一旦检测到返回了新的对象，停止执行，然后将当前 Deferred 对象的 Promise 引用改变为新的 Promise 对象，并将队列中余下的回调交给它。

  写道这里，你是否明白了恶魔金字塔该如何优化？

  再次重申，这里的代码主要用于 Promise 的实现原理。在更多细节优化方面，Q 或者 when 等 Promise 库做的更好，实际应用时请采用这些成熟库。

- 将 API Promise 化

  这里仍然会发现，为了体验更好的 API，需要做较多的准备工作。这里提供了一个方法可以批量将方法 Promise 化，相关代码如下：

  ```js
  var smooth = function (method) {
    return function () {
      var deferred = new Deferred();
      var args = Array.prototype.slice.call(arguments, 1);
      args.push(deferred.callback());
      method.apply(null, args);
      return deferred.promise;
    };
  };
  ```

  于是前面的文件读取的构造：

  ```js
  var readFile1 = function (file, encoding) {
    var deffered = new Deferred();
    fs.readFile(file, encoding, deferred.callback());
    return deffered.promise;
  };
  ```

  可以简化为：

  ```js
  var readFile = smooth(fs.readFile);
  ```

  要实现同样的效果，代码量将会锐减。

### 4.3.3 流程控制库

前面叙述了最为主流的模式——事件发布/订阅模式和 Promise/Deferred 模式，这些是经典的模式或者是写进规范里的解决方案，但一旦涉及模式或者规范，就需要为他们做较多的准备工作。这一节，将会介绍一些非模式化的应用，虽非规范，但更灵活。

#### 1.尾触发与 Next

除了事件和 Promise 外，还有一类方法是需要手工调用才能持续执行后续调用的，我们将该类方法叫做尾触发，常见的关键字是`next`。事实上，尾触发目前应用最多的地方是 Connect 的中间件。

这里我们暂且不关注 Connect 的具体应用，先看下 Connect 的 API 暴露方式，相关代码如下：

```js
var app = connect();

app.use(connect.staticCache());
app.use(connect.static(__dirname + '/public'));
app.use(connect.cookieParser());
app.use(connect.session());
app.use(connect.query());
app.use(connect.bodyParser());
app.use(connect.csrf());
app.listen(3001);
```

在通过`use()`方法注册好一系列中间件后，监听端口上的请求。中间件利用了尾触发的机制，最简单的中间件如下：

```js
function(req, res, next) {
  // 中间件
}
```

每个中间件传递请求对象、响应对象和尾触发函数，通过队列形成一个处理流。

中间件机制使得在处理网络请求时，可以像面向切面编程一样进行过滤、验证、日志等功能，而不与具体业务逻辑产生关联，以致产生耦合。

下面我们来 Connect 的核心实现，相关代码如下：

```js
function createServer() {
  // 通过如下代码创建了HTTP服务器的request事件处理函数
  function app(req, res) {
    app.handle(req, res);
  }

  utils.merge(app, proto);
  utils.merge(app, EventEmitter.prototype);

  app.route = '/';
  // 真正核心代码是 下面一句
  // stack属性是这个服务器内部维护的中间件队列
  // 通过调用`use()`方法我们可以将中间件放进队列中
  app.stack = [];

  for (var i = 0; i < arguments.length; i++) {
    app.use(arguments[i]);
  }

  return app;
}

// 下面代码为`use()`方法的重要部分：
app.use = function (route, fn) {
  // some code
  this.stack.push({ route: route, handle: fn });

  return this;
};

// 此时就建好了处理模型。接下来，结合Node原生http模块实现监听即可。监听函数的实现：
app.listen = function () {
  var server = http.createServer(this);
  return server.listen.apply(server, arguments);
};

// 最终回到 `app.handle()`方法，每一个监听到的网络请求都从这里开始处理：
app.handle = function (req, res, next) {
  // some code
  next();
};
```

原始的`next()`方法较为复杂，下面是简化后的内容，其原理十分简单，取出队列中的中间件，并执行，同时传入当前方法以实现递归调用，达到持续触发的目的：

```js
function next(err) {
  // some code
  // next callback

  layer = stack[index++];

  layer.handle(req, res, next);
}
```

所有嫌弃异步编程复杂的开发者，均可以参考 Connect 的流式处理，这对于划分业务逻辑、逐步处理均有效。

值得提醒的是，尽管中间件这种尾触发模式并不要求每个中间方法都异步的，但是如果每个步骤都采用异步来处理，实际上只是串行化的处理，没办法通过并行的异步调用来提升业务的处理效率。流式处理可以将一些串行的逻辑扁平化，但是并行逻辑处理还是需要搭配事件或者 Promise 完成的，这样业务在纵向和横向都能够各自清晰。

在 Connect 中，尾触发十分适合处理网络请求的场景。将复杂的处理逻辑拆解为简洁、单一的处理单元，逐层次地处理请求对象和响应对象。

#### 2.async

接下来，我们要介绍最知名地流程控制模块 async。async 长期占据 NPM 依赖榜地前三名，可见在 Node 开发中，流程控制是开发过程中地基本需求。async 模块提供了 20 多个方法用于处理异步和各种协作模式，这里我们介绍几种典型的用法。

- 异步的串行执行

  这里我们依旧采用前面读取两个文件的例子，看一下 async 是如何解决“恶魔金字塔”问题的。

  async 提供了`series()`方法来实现一组任务的串行执行，示例代码：

  ```js
  async.series(
    [
      function (callback) {
        fs.readFile('file1.txt', 'utf8', callback);
      },
      function (callback) {
        fs.readFile('file2.txt', 'utf8', callback);
      },
    ],
    function (err, resutls) {
      // results => [file1.txt, file2.txt]
    }
  );
  // 这段代码等价于：
  fs.readFile('file1.txt', 'utf8', function (err, content) {
    if (err) return callback(err);
    fs.readFile('file2.txt', 'utf8', function (err, data) {
      if (err) return callback(err);
      callback(null, [content, data]);
    });
  });
  ```

  这段代码值得玩味的是回调函数。可以发现，`series()`方法中传入的函数`callback()`并非由使用者指定。事实上，此处的回调函数由 async 通过高阶函数的方式注入，这里隐含了特殊的逻辑。每个`callback()`执行时将会将结果保存起来，然后执行下一个调用，直到结束所有调用。最终的回调函数执行时，队列的异步调用保存的结果以数组的方式传入。这里的异常处理规则是一旦出现异常，就结束所有调用，并将一场传递给最终回调函数的第一个参数。

- 异步的并行执行

  当我们需要通过并行来提升性能是，async 提供了`parallel()`方法，用以并行执行一些异步操作。以下为读取两个文件的并行版本：

  ```js
  async.parallel(
    [
      function (callback) {
        fs.readFile('file1.txt', 'utf8', callback);
      },
      function (callback) {
        fs.readFile('file2.txt', 'utf8', callback);
      },
    ],
    function (err, results) {
      // results => [file1.txt, file2.txt]
    }
  );
  ```

  上面这段代码等价于下面的代码：

  ```js
  var counter = 2;
  var results = [];
  var done = function (index, value) {
    results[index] = value;
    counter--;
    if (counter === 0) {
      callback(null, results);
    }
  };

  // 值传递一个异常
  var hasErr = false;
  var fail = function (err) {
    if (!hasErr) {
      hasErr = true;
      callback(err);
    }
  };

  fs.readFile('file1.txt', 'utf8', function (err, content) {
    if (err) return fail(err);
    done(0, content);
  });

  fs.readFile('file2.txt', 'utf8', function (err, data) {
    if (err) return fail(err);
    done(1, data);
  });
  ```

  同样，通过 async 编写的代码既没有深度的嵌套，也没有复杂的状态判断，它的诀窍依然来自于注入的回调函数。`parallel()`方法对于异常的判断依然是一旦某个异步调用产生了异常，就会将异常作为第一个参数传入给最终的回调函数。只有所有异步调用都正常完成时，才会将结果以数组的方式传入。

  也许你还记得 EventProxy 的方案：

  ```js
  var EventProxy = require('eventproxy');

  var proxy = new EventProxy();
  proxy.all('content', 'data', function (content, data) {
    callback(null, [content, data]);
  });
  proxy.fail(callback);

  fs.readFile('file1.txt', 'utf8', proxy.done('content'));
  fs.readFile('file2.txt', 'utf8', proxy.done('data'));
  ```

  与通过 async 编写所产生的代码量相差不大。EventProxy 虽然基于事件发布/订阅模式而设计，但也用到了与 async 相同的原理，通过特殊的回调函数来隐含返回值的处理。所不同的是，在 async 的框架模式下，这个回到函数由 async 封装后传递出来，而 EventProxy 则通过 done()和 fail()方法来生成新的回调函数。这两种实现方式都是高阶函数的应用。

- 异步调用的依赖处理

  `series()`适合无依赖的异步串行执行，但当前一个结果是后一个调用的输入时，`series()`方法就无法满足需求了。所幸，这种典型场景的需求，async 提供了`waterfall()`方法来满足，相关代码：

  ```js
  async.waterfall(
    [
      function (callback) {
        fs.readFile('file1.txt', 'utf8', function (err, content) {
          callback(err, content);
        });
      },
      function (arg, callback) {
        // arg1 => file2.txt
        fs.readFile(arg, 'utf8', function (err, content) {
          callback(err, content);
        });
      },
      function (arg, callback) {
        // arg => file3.txt
        fs.readFile(arg, 'utf8', function (err, content) {
          callback(err, content);
        });
      },
    ],
    function (err, results) {
      console.log(err, results);
    }
  );
  ```

  这段代码等价于如下代码：

  ```js
  fs.readFile('file1.txt', 'utf8', function (err, data1) {
    if (err) return callback(err);
    fs.readFile(data1, 'utf8', function (err, data2) {
      if (err) return callback(err);
      fs.readFile(data2, 'utf8', function (err, data3) {
        if (err) return callback(err);
        callback(null, data3);
      });
    });
  });
  ```

- 自动依赖处理
  在现实的业务环境中，具有很多复杂的依赖关系，这些业务或是异步，或是同步。这种混杂的编程环境经常让人理不清顺序的情况。为此，async 提供了一个强大的方法`auto()`实现复杂业务处理。

  假设我们的业务场景如下：

  1. 从磁盘读取配置文件。
  2. 根据配置文件连接 MongoDB。
  3. 根据配置文件连接 Redis。
  4. 编译静态文件。
  5. 上传静态文件到 CDN。
  6. 启动服务器。

  简单映射下上述内容：

  ```js
  {
    readConfig: function () {},
    connectMongoDB: function () {},
    connectRedis: function () {},
    compileAsserts: function () {},
    uploadsAsserts: functin () {},
    startup: function () {}
  }
  ```

  接下来分析下依赖关系。可以看出，`connectMongoDB`和`connectRedis`依赖`readConfig`，`uploadsAsserts`依赖`compileAsserts`，`startup`则依赖所有完成。依赖关系如下：

  ```js
  var deps = {
    readConfig: function (callback) {
      // read config file
      callback();
    },
    connectMongoDB: [
      'readConfig',
      function (callback) {
        // connect to mongodb
        callback();
      },
    ],
    connectRedis: [
      'readConfig',
      function (callback) {
        //connet to redis
        callback();
      },
    ],
    compileAsserts: function (callback) {
      // complile asserts
      callback();
    },
    uploadsAsserts: [
      'compileAsserts',
      function (callback) {
        // upload to assert
        callback();
      },
    ],
    startup: [
      'connectMongoDB',
      'connectRedis',
      'uploadAsserts',
      function (callback) {
        // startup
      },
    ],
  };

  // `auto()`方法能根据依赖关系自动分析，以最佳的顺序执行以上业务：
  async.auto(deps);
  ```

  如果用 EventProxy 实现，则需要更细腻的事件分配，相关代码：

  ```js
  proxy
    .assp('readtheconfig', function () {
      // read config file
      proxy.emit('readConfig');
    })
    .on('readConfig', function () {
      // connect to mongodb
      proxy.emit('connectMongoDB');
    })
    .on('readConfig', function () {
      // connect to redis
      proxy.emit('connectRedis');
    })
    .assp('compiletheasserts', function () {
      // compile asserts
      proxy.emit('compileAsserts');
    })
    .on('compileAsserts', function () {
      proxy.emit('uploadsAsserts');
    })
    .all('connectMongoDB', 'connectRedis', 'uploadAsserts', function () {
      // startup
    });
  ```

- 小结

  本节主要介绍[async](https://github.com/callan/async 'async')的几种常见用法。此外，async 还提供了 forEach、map 等类似 ECMAScript 5 中数组的方法。

#### 3.Step

另一个知名的流程控制库是 Tim Caswell 的 Step，它比 async 更轻量，在 API 的暴露上也更具备一致性，因为它只有一个接口 Step。通过`npm install step`即可安装使用：

```js
Step(task1, task2, task3);
```

Step 接受任意数量的任务，所有的任务都将会串行依次执行。下面的示例代码将依次读取文件：

```js
Step(
  function readFile1() {
    fs.readFile('file1.txt', 'utf8', this);
  },
  function readFile2() {
    fs.readFile('file2.txt', 'utf8', this);
  },
  function done(err, content) {
    console.log(content);
  }
);
```

可以看到，Step 与前面介绍的事件模式。Promise 模式甚至 async 都不同的一点在于 Step 用到了 this 关键字。事实上，它是 Step 内部的一个 next()方法，将异步调用的结果传递给下一个任务作为参数，并调用执行。

- 并行任务执行

  那么，Step 如何实现多个异步任务并行执行呢？this 具有一个`parallel()`方法，它告诉 Step，需要等待所有任务完成时才进行下一个任务：

  ```js
  Step(
    function readFile1() {
      fs.readFile('file1.txt', 'utf8', this.parallel());
      fs.readFile('file2.txt', 'utf8', this.parallel());
    },
    function done(err, content1, content2) {
      // content1 => file1
      // content2 => file2
      console.log(arguments);
    }
  );
  ```

  使用`parallel()`的时候需要小心的是，如果异步方法的结果传回的是多个参数，Step 将只会取前两个参数，相关代码如下：

  ```js
  var asyncCall = function (callback) {
    process.nextTick(function () {
      callback(null, 'result1', 'result2');
    });
  };
  // 在调用`parallel()`时，result2将会丢弃。
  ```

  Step 的 parallel()方法的原理时每次执行时，内部计数器加 1，然后返回一个回调函数，这个回调哈数在异步调用结束时才执行。当回调函数执行时，将计数器减 1。当计数器为 0 时，告知 Step 所以异步调用结束了，Step 会执行下一个方法。

  Step 与 async 相同的是异常处理，一旦有一个异常产生，这个异常会作为下一个方法的第一个参数传入。

- 结果分组

  Step 提供的另一个方法是`group()`，它类似于`parallel()`的效果，但是在结果传递上略有不同。下面的代码用于读取一个目录，然后迭代其中文件的操作：

  ```js
  Step(
    function readDir() {
      fs.readdir(__dirname, this);
    },
    function readFiles(err, results) {
      if (err) throw err;

      // Create a new group
      var group = this.group();
      results.forEach(function (filename) {
        if (/\.js$/.test(filename)) fs.readFile(__dirname + '/' + filename, 'utf8', group());
      });
    },
    function showAll(err, files) {
      if (err) throw err;
      console.dir(files);
    }
  );
  ```

  我们注意到有两次`group()`的调用。第一次调用是告知 Step 要并行执行，第二次调用的结果将会生成一个回调函数，而回调函数接受的返回值将会按组存储。`parallel()`传递给下一个任务的结果是如下形式：

  ```js
  function (err, result1, result2, ...);
  ```

  `group()`传递的结果是：

  ```js
  function (err, results);
  ```

  这个函数返回的数据保存在数组中。

#### 4.wind

这里还要介绍一种思路完全不同的异步编程方案[wind](https://github.com/JefferyZhao/wind 'wind')。它的前身为 Jscex，由国内知名码农赵劼完成开发。它为 JavaScript 语言提供了一个 monadic 扩展，能够显著提高一些常见场景下的异步编程体验。

异步编程有时需要面临的场景非常特殊，下面我们由一个冒泡排序来了解 wind 的特殊之处：

```js
var compare = function (x, y) {
  return x - y;
};

var swap = function (a, i, j) {
  var t = a[i];
  a[i] = a[j];
  a[j] = t;
};

var bubbleSort = function (array) {
  for (var i = 0; i < array.length; i++) {
    for (var j = 0; j < array.length; j++) {
      if (compare(array[j], array[j + 1]) > 0) swap(array, j, j + j);
    }
  }
};
```

现在我们要添加的需求是，将这个冒泡排序动画起来。这意味着在`swap()`方法中需要添加动画逻辑，这在 JavaScript 中并不是一件难事，困难的地方在于动画需要延时的方式完成。但在 JavaScript 中只有 setTimeout()能够实现延时功能（用 while 判断时间的方式不可取，这在前面有所描述）。我们直到，`setTimeout()`是一个异步方法，在执行后，将立即返回。所以，难点出现在：

- 动画执行时无法停止排序算法的执行；
- 排序算法的继续执行将会启动更多动画。

因此，逐步骤的动画将难以实现，而 wind 在解决这个问题上体现出了它的独特魅力之处，相关代码如下：

```js
var compare = function (x, y) {
  return x - y;
}

var swapAsync = eval(Wind.compile('async', function (a, i, j) {
  $await(Wind.Async.sleep(20));// 暂停20ms
  var t = a[i];
  a[i] = a[j];
  a[j] = t;
  paint(a); // 重绘数组
}))；

var bubbleSort = eval(Wind.compile('async', function (array) {
  for (var i = 0; i < array.length; i++) {
    for (var j = 0; j < array.length-i-1; i++) {
      if (compare(array[j], array[j+1]) > 0) {
        $await(swapAsync(array, j, j+1));
      }
    }
  }
}));
```

上述代码实现了暂停 20ms、绘制动画、继续排序的效果。从代码的角度来看，这里虽然接入了异步方法，但是并没有如同其它异动流程控制库那样变得异步化，逻辑并没有因为异步被拆分。同时可以注意到，我们的代码引入了一些新的东西：

- eval(Wind.compile('async', function () {}));
- \$await();
- Wind.Async.sleep(20);

下面我们将详细介绍以上 3 行代码的特异之处。

- 异步任务定义

  `eval()`函数在业界一向是一个需要谨慎对待的函数，Douglas Crockford 更是深恶痛觉地将其称为魔鬼，因为他能访问上下文和编译器，导致上下文混乱。大多数利用`eval()`函数地人都不能把握好它的用法，导致 Douglas Crockford 认为它是 JavaScript 可有可无地功能。

  但是在 wind 的世界里，恰好反其道而行之，巧妙地利用了`eval()`访问上下文的特性。`Wind.compile()`会将普通的函数进行编译，然后交给`eval()`执行。换言之，`eval(Wind.compile('async', function () {}));`定义了异步任务。`Wind.Async.sleep()`则内置了对`setTimeout()`的封装。

- \$await()与任务模型

  在定义完异步方法后，wind 提供了`$await()`方法实现等待完成异步方法。但事实上，它并不是一个方法，也不存在于上下文中，只是一个等待的占位符，告之编译器这里需要等待。

  `$await()`接受的参数是一个任务对象，表示等待任务结束后才会执行后续操作。每一个异步任务都可以转化为一个任务，wind 正是基于任务模型实现的。下面的代码用于将`fs.readFile()`调用转化为一个任务模型：

  ```js
  var Wind = require('wind');
  var Task = Wind.Async.Task;

  var readFileAsync = function (file, encoding) {
    return Task.create(function (t) {
      fs.readFile(file, encoding, function (err, file) {
        if (err) {
          t.complete('failure', err);
        } else {
          t.complete('success', file);
        }
      });
    });
  };
  ```

  出了通过`eval(Wind.compile('async', function () {}))`定义任务外，正式的任务创建方法为`Task.create()`。执行`readFileAsync()`进行偏函数转换得到真正的任务。异步方法在执行结束时，可以通过`complete()`传递 failure 或 success 信息，告知任务执行完毕。如果是 failure 则可以通过 try/catch 捕获异常。这略微有些打破前述 try/catch 无法捕获回调函数中异常的结论。下面的代码为调用`readFileAsync()`得到一个任务的示例：

  ```js
  var task = readFileAsync('file1.txt', 'utf8');
  ```

  下面我们如同介绍 async 或 Step 的串行示例一样，尝试感受下 wind 的风采：

  ```js
  var serial = eval(
    Wind.compile('async', function () {
      var file1 = $await(readFileAsync('file1.txt', 'utf8'));
      console.log(file1);
      var file2 = $await(readFileAsync('file2.txt', 'utf8'));
      console.log(file2);

      try {
        var file3 = $await(readFileAsync('file3.txt', 'utf8'));
      } catch (err) {
        console.log(err);
      }
    })
  );

  serial().start();
  ```

  执行上述代码，将得到如下输出：

  ```sh
  file1
  file2
  { [Error: ENOENT, open 'file3.txt'] errno: 34, code: 'ENOENT', path: 'file3.txt' }
  ```

  异步方法在 JavaScript 中通常会立即返回，在 wind 中做到了步阻塞 CPU 但阻塞代码的目的。接下来我们尝试下并行的效果，相关代码如下：

  ```js
  var parallel = eval(
    Wind.compile('async', function () {
      var result = $await(
        Task.whenAll({
          file1: readFileAsync('file1.txt', 'utf8'),
          file2: readFileAsync('file2.txt', 'utf8'),
        })
      );
    })
  );
  parallel().start();
  ```

  wind 提供了`whenAll()`来处理并发，通过\$await 关键字将等待配置的所有任务完成后才继续向下执行。

- 异步方法转换辅助函数

  可以看到，除了`eval(Wind.compile('async', function () {}))`在代码中稍显冗长外，异步调用在代码层面上已经与同步调用相差无几。这十分适合从已有的采用同步编写方式的代码向 Node 迁移，可以省略掉重写代码的开销。

  如同 Promise/Deferred 模式可以让异步编程模型变得简单，这种近同步变成的体验需要我们额外或者提前完成的事情是：将异步方法任务化。这种任务化的过程可以看作是 Promise/Deferred 的封装。如果每个方法都如`readFileAsync`一般去定义，将会是一个庞大的工作量。wind 提供了两个方法来辅助转换：

  - Wind.Async.Binding.fromCallback
  - Wind.Async.Binding.fromStandard

  在 Node 异步方法的回调传值有两种，一种是无异常的调用，通常只有一个参数返回，如下：

  ```js
  fs.exists('/etc/passwd', function (exists) {
    // exists 参数表示是否存在
  });

  // 而fromCallback用于转换这类异步调用为wind中的任务
  ```

  另一类是带异常的调用，遵循规范将返回参数列表的第一个参数作为异常标识，如下所示：

  ```js
  fs.readFile('file1.txt', function (err, data) {
    // err 表示异常
  });
  // fromStandard 用于转换这类异常调用到wind中的任务
  ```

  所以，`readFileAsync`的定义其实只要一行代码即可实现：

  ```js
  var readFileAsync = Wind.Async.Binding.fromStandard(fs.readFile);
  ```

#### 5.流程控制小结

从本书介绍的各个流程控制案例来看，从解决“恶魔金字塔”到解决异步协作的方法有很多种，几个类库几乎各显神通。异步编程虽然相对复杂，但并非难事，相同的问题通过各种技巧依然能将复杂的事情简单化。

这里简单的对比下几种方案的区别：事件发布/订阅模式相对算是一种较为原始的方式，Promise/Deferred 模式贡献了一个非常不错的异步任务模型的抽象。而上述的这些异步流程控制方案与 Promise/Deferred 模式的思路不同，Promise/Deferred 的重头在于封装异步的调用部分，流程控制库则显得没有模式，将处理重点放置在回调函数的注入商。从自由度来讲，async、Step 这类的流控库要相对灵活得多。EventProxy 库则主要借鉴事件发布/订阅模式和流程控制库通过高阶函数生成回调函数的方式实现。

除了 async、Step、EventProxy、wind 等方案外，还有一类通过源代码编译的方案来实现流程控制简化，streamline 是一个典型的例子。这类例子并不在本章的讨论范围，如果读者有兴趣，可以自行查阅相关资料。

## 4.4 异步并发控制

在陆续介绍的各种异步编程方案里，解决的问题无外乎保持异步的性能优势，提升编程体验，但是这里有一个过犹不及的案例。

在 Node 中，我们可以十分方便地利用异步发起并行调用。使用下面的代码，我们可以轻松发起 100 次异步调用：

```js
for (var i = 0; i < 100; i++) {
  async();
}
```

但是如果并发量过大，我们的下层服务器将会吃不消。如果是对文件系统大量并发调用，操作系统的文件毛舒服将会被瞬间用光，则抛出如下错误：

```sh
Error: EMFILE, too many open files
```

可以看出，异步 I/O 与同步 I/O 的显著差距：同步 I/O 因为每个 I/O 都彼此阻塞，在循环体中，总是一个接着一个调用，不会出现好用文件描述符太多的情况，同时性能也是低下的；对于异步 I/O，虽然并发容易实现，但是由于太容易实现，依然需要控制。换言之，尽管是要压榨底层系统性能，但还是需要给予一定的过载保护，以防止过犹不及。

### 4.4.1 bagpipe 的解决方案

如何对既有的异步 API 添加过载保护，我们期望的当然不是去改动 API。那么如何实现呢？我写的 bagpipe 模块的解决思路是这样的。

- 通过一个队列来控制并发量。
- 如果当前活跃（指调用发起但未执行回调）的异步调用量小于限定值，从队列中取出执行。
- 如果活跃调用达到限定值，调用暂时存放在队列中。
- 每个异步调用结束时，从队列中取出新的异步调用执行。

bagpipe 的 API 主要暴露了一个`push()`方法和 full 事件，示例代码：

```js
var Bagpipe = require('bagpipe');

// 设定最大并发数为10
var bagpipe = new Bagpipe(10);

for (var i = 0; i < 100; i++) {
  bagpipe.push(async, function () {
    // 异步回调执行
  });
}

bagpipe.on('full', function (length) {
  console.warn('底层系统处理不能及时完成，队列拥堵，目前队列长度为：' + length);
});
```

这里的实现细节类似于前文的`smooth()`。`push()`方法依然是通过函数变换的方式实现，假设第一个参数是方法，最后一个参数是回到函数，其余为其它参数，其核心实现是：

```js
Bagpipe.prototype.push = function (method) {
  var args = [].slice.call(arguments, 1);
  var callback = args[args.length - 1];

  if (typeof callback === 'function') {
    args.push(function () {});
  }

  if (this.options.disabled || this.limit < 1) {
    method.apply(null, args);
    return this;
  }

  // 队列长度也超过限制值时
  if (this.queue.length < this.queueLength || !this.options.refuse) {
    this.queue.push({
      method: method,
      args: args,
    });
  } else {
    var err = new Error('Too much async call in queue');
    err.name = 'TooMuchAsyncCallError';
    callback(err);
  }

  if (this.queue.length > 1) this.emit('full', this.queue.length);

  this.next();
  return this;
};
// 将调用推入队列后，调用依次`next()`方法尝试触发。
// next() 方法定义如下：
Bagpipe.prototype.next = function () {
  var that = this;
  if (that.active < that.limit && that.queue.length) {
    var req = that.queue.shift();
    that.run(req.method, req.args);
  }
};

// next()方法主要判断活跃调用的数量，
// 如果正常，将调用内部方法 run() 来执行真正的调用。
// 这里为了判断回调函数执行，采用了一个注入代码的技巧：
Bagpipe.prototype.run = function (method, args) {
  var that = this;
  that.active++;
  var callback = args[args.length - 1];
  var timer = null;
  var called = false;

  // inject logic
  arg[args.length - 1] = function (err) {
    // anyway, clear the timer
    if (timer) {
      clearTimeout(timer);
      timer = null;
    }

    if (!called) {
      that._next();
      callback.apply(null, arguments);
    } else {
      // pass the outdated error
      if (err) {
        that.emit('outdated', err);
      }
    }
  };

  var timeout = that.options.timeout;
  if (timeout) {
    timer = setTimeout(function () {
      // set called as true
      called = true;
      that._next();

      // pass the exception
      var err = new Error(timeout + 'ms timeout');
      err.name = 'BagpipeTimeoutError';
      err.data = {
        name: method.name,
        method: method.toString(),
        args: args.slice(0, -1),
      };
      callback(err);
    }, timeout);
  }
  method.apply(null, args);
};
```

用户传入的回到函数被真正执行前，被封装替换过。这个封装的回到函数内部的逻辑将活跃值的计数器减 1 后，主动调用`next()`执行后续等待的异步调用。

bagpipe 类似于打开了一道窗口，允许异步调用并行进行，但是严格设定上限。仅仅在调用`push()`时分开传递，并不对原有 API 有任何侵入。

- 拒绝模式

  事实上，bagpipe 还有一些深度的使用方式。对于大量的异步调用，也需要分场景进行区分，因为涉及并发控制，必然会造成部分调用需要进行等待。如果调用有实时方面的需求，那么需要快速返回，因为等到方法被真正执行时，可能已经超过了等待事件，即使返回了数据，也没有意义了。这种场景下需要快速失败，让调用方尽早返回，而不用浪费不必要的等待时间。bagpipe 为此支持了拒绝模式。

  ```js
  // 拒绝模式的使用只要设置下参数即可，相关代码如下：
  var bagpipe = new Bagpipe(10, {
    refuse: true,
  });
  // 在拒绝模式下，如果等待的调用队列也满了之后，新来的调用就直接返回给它一个队列太忙的拒绝异常。
  ```

- 超时控制

  造成队列拥塞的主要原因是异步调用耗时太久，调用产生的速度远远高于执行的速度。为了防止某些异步调用使用了太多的事件，我们需要设置一个时间蓟县，将那些执行时间太久的异步调用清理出活跃队列，让派对中的异步调用尽快执行。否则在拒绝模式下，会有太多的调用因为某个执行很慢，导致得到拒绝异常。相对而言，这种场景下得到拒绝异常显得比较无辜。为了公平地对待在实时需求场景下的每个调用，必须要控制每个调用的执行时间，将那些害群之马踢出队伍。

  为此，bagpipe 也提供了超时控制。超时控制是为了异步调用设置一个时间阈值，如果异步调用没有在规定时间内完成，我们先执行用户传入的回调函数，让用户得到一个超时异常，以尽早返回。然后让下一个等待队列中的调用执行。

  ```js
  // 超时设置如下：
  // 设定最大并发为10
  var bagpipe = new Bagpipe(10, { timeout: 3000 });
  ```

- 小结

  异步调用的并发限制在不同场景的需求不同：非实时场景下，让超出限制的并发暂时等待执行已经可以满足需求；但在实时场景下，需要更细粒度、更合理的控制。

### 4.4.2 async 的解决方案

无独有偶，async 也提供了一个方法用于处理异常调用的限制：`parallelLimit()`。如下是 async 的示例代码：

```js
async.parallelLimit(
  [
    function (callback) {
      fs.readFile('file1.txt', 'utf8', callback);
    },
    function (callback) {
      fs.readFile('file2.txt', 'utf8', callback);
    },
  ],
  1,
  function (err, results) {
    // TODO
  }
);
```

`parallelLimit()`与`parallel()`类似，但是多了一个用于限制并发数量的参数，使得任务只能同时并发一定数量，而不是无限制并发。

`parallelLimit()`方法的缺陷在于无法动态地增加并行任务。为此，async 提供了`queue()`方法来满足该续期，这对于遍历文件目录等操作十分有效。以下是`queue()`的示例代码：

```js
var q = async.queue(function (file, callback) {
  fs.readFile(file, 'utf8', callback);
}, 2);

q.drain = function () {
  // 完成了队列中的所有任务
};
fs.readdirSync('.').forEach(function (file) {
  q.push(file, function (err, data) {
    // TODO
  });
});
```

尽管`queue()`实现了动态添加并行任务，但是相比`parallelLimit()`，由于`queue()`接收的参数是固定的，它丢失了`parallelLimit()`的多样性，我死心地认为 bagpipe 更灵活，可以添加任意类型的异步任务，也可以动态添加异步任务，同时还能够在实时处理场景中加入拒绝模式和超时控制。在实际应用中，开发者可以根据场景进行取舍。

## 4.5 总结

在接触 Node 的过程中，很多人粗略地接触了几个回调函数之后就放弃了。尽管异步编程略微艰难，但是并非一无是处，一旦习惯，就显得自然能。从社区和过往地经验而言，JavaScript 异步编程的难题已经基本解决，无论是通过事件，还是通过 Promise/Deferred 模式或者流程控制库。相信在账务以上技巧之后，异步编程不是难事，习惯异步变成后，将会收获许多值得享受的编程体验。

本章主要介绍了主流的几种异步编程解决方案，这是目前 JavScript 中主要使用的方案。但对于其它语言而言，还有协程（coroutine)等方式。但是由于 Node 基于 V8 的原因，在目前 ECMAScript5 的是线下，还不支持协程。这些标准和规范还在丁志忠，所以暂时不作介绍。未来的 v8 如果支持 Generator,也将在 Node 中直接使用。

最后，因为总是习惯性地以线性方式进行思考，一只鱼异步编程相对较难以掌握。这个世界以一部运行地本质是不会因为大家线性的惯性思维改变。就像日出月落不会因为你的心情而改变其自有的运行轨迹。
