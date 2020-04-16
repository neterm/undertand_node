# 异步编程

## 4.2 异步编程的优势与难点

曾经的单线程模型在同步I/O的影响下，由于I/O调用缓慢，在应用层面导致CPU与I/O无法重叠进行。为了照顾编程人员的阅读思维习惯，同步I/O盛行了很多年。在日新月异的技术大潮面签，性能问题摆在了编程人员面签。提升性能的方式过去多用多线程的方式解决，但是多线程的引入在业务逻辑方面制造的麻烦也不少。从操作系统调度多线程的上下文切换开销，到实际编程里的锁、同步等问题，让开发人员头疼的时候也并不少。另一个解决I/O性能的方案是通过C/C++调用操作系统底层接口，自己手工完成异步I/O，这个能够达到很高的性能，但是调试和开发的门槛却很高，在帮助业务解决问题上，需要花费较大的精力。Node利用JavaScript及其内部异步库，将异步直接提升到业务层面，这是一种创新。

### 4.2.1 优势

Node带来的最大特性莫过于基于事件驱动的非阻塞I/O模型，这是它的灵魂所在。非阻塞I/O可以使CPU与I/O并不互相依赖等待，让资源得到更好的利用。对于网络应用而言，并行带来的想象空间更大，延展而开的是分布式和云。并行使得各个单点之间能够更有效地组织起来，这也是Node在云计算厂商中广受青睐地原因。

如果采用传统的同步I/O模型，分布式计算中性能的折扣很明显。

通过学习Node实现异步I/O的原理，利用事件循环的方式，JavaScript线程像一个分配任务和处理结果的大管家，I/O线程池里的各个I/O都是小二，负责兢兢业业地完成分配来地任务，小二与管家之间互不依赖，所以可以保证整体地高效率。这个利用事件循环地经典调度方式在很多地方都存在应用，最典型的是UI编程，如iOS应用开发等。

这个模型的缺点则在于管家无法承担过多细节性的任务，如果承担太多，则会影响到任务的调度，管家忙个不停，而小二却得不到活干，结局则是整体效率的降低。

换言之，Node是为了解决编程模型中阻塞I/O的性能问题的，采用了单线程模型，这导致Node更像一个处理I/O密集问题的能手，而CPU密集型则取决于管家的能耐如何。

在第一章中，从斐波那契数列数列计算的测试结果中可以看出，这个管家的具体能力如何。如果形象地去评判的话，C语言的性能至尊，得益于V8性能的Node则是一流武林高手，在具备武功秘籍的情况下（调用C/C++扩展模块），Node的能力可以逼近顶尖之列。

由于事件循环模型需要应对海量请求，海量请求同时作用在单线程上，就需要防止任何一个计算耗费过多CPU时间片。至于计算密集型，还是I/O密集型，只要计算不影响异步I/O的调度，那就构不成问题。建议对CPU的好用不要超过10ms，或者将大量的计算分解成诸多小量计算，通过`setImmediate()`进行调度。只要合理利用Node的异步模型与V8的高性能，就可以充分发挥CPU和I/O资源的优势。

### 4.2.2 难点

Node令异步编程如此风行，这也是异步编程首次大规模出现在业务层面。它借助异步I/O模型及V8高性能引擎，突破单线程的性能瓶颈，让JavaScript在后端达到实用价值。另一方面，它也同意了前后端JavaScript的编程模型。对于异步编程带来的新鲜感与不适感，开发者们有着不同程度的感受。接下来，我们梳理下异步编程的难点，以便更好的利用Node。

#### 1. 难点1：异常处理

过去我们处理异常时，通常实用类Java的`try/catch/final`语句块进行异常捕获，示例代码如下：

```javascript
try {
  JSON.parse(json);
} catch (e) {
  // TODO
}
```

但是这对于异步编程而言并不一定适用。第三章中提到过，异步I/O的实现主要包含两个阶段：提交请求和处理请求。这两个阶段中间有事件循环的调度，两者彼此不关联。异步方法则通常在第一个阶段提交请求后立即返回，因为异常并不一定发生在这个阶段，`try/catch`的功效在此处不会发挥任何作用。异步方法的定义如下：

```js
var async = function(callback) {
  process.nextTick(callback);
}
```

调用`async()`方法后，`callback`被存放起来，知道下一个事件循环（Tick）才会取出来执行。尝试对异步方法进行`try/catch`操作只能捕获当次事件循环内的异常，对`callback`执行时抛出的异常将无能为力。

```js
try {
  async(callback);
} catch (e) {
  // TODO
}
```

Node在处理异常形成了一种约定，将异常作为回调函数的第一个参数传回，如果为空值，则表明异步调用没有异常抛出：

```js
async (function(err,results) {
  //TODO
});
```

在我们自行编写的异步方法上，也需要去遵循这样一些原则：

- 原则一：必须执行调用者传入的回调函数；

- 原则二：正确传递异常供调用者判断。

  ```javascript
  var async = function(callback) {
    var async = function(callback) {
      process.nextTick(function() {
        var results = something;
        if(error) {
          return callback(error); 
        }
        callback(null, resutls);
      });
    }
  }
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

这或许时Node被人诟病最多地地方。在前端开发中，DOM事件相对而言不会存在互相依赖或需要多个事件一起协作地场景，较少存在异步多级依赖地情况。下面地代码为彼此独立地DOM事件绑定：

```js
$(selector).click(function(event) {
  // TODO
});
$(selector).change(function(event) {
  // TODO
});
```

但是对于Node而言，事务中存在多个异步调用地场景比比皆是。比如一个遍历目录地操作，其代码：

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

这样在结果的保证上是没有问题的，问题在于这并没有利用好异步I/O带来的并行优势。这是异步编程的典型问题，为此有人曾说，因为嵌套的深度，未来最难看的代码必将从Node诞生。但是实际情况没有想象得那么糟糕，且看后面如何解决该问题。

#### 3. 阻塞代码 

对于进入JavaScript世界不久得开发者，比较纳闷这门编程语言竟然没有`sleep()`这样得线程沉睡功能，唯独能用于延时操作得只有`setInterval()`和`setTimeout`这两个函数。但是让人惊讶的是，这两个函数并不能阻塞后续代码的继续执行。所以，有多半的开发者会写下下述这样的代码来实现`sleep(1000)`的效果：

```js
var start = new Date();
while(new Date - start < 1000) {
  // TODO
}
// 需要阻塞的代码
```

但是事实是糟糕的，这段代码会持续占用CPU进行判断，与真正的线程沉睡相去甚远，完全破环了事件循环的调度。由于Node是单线程的原因，CPU资源全部会用于为这段代码服务，导致其余任何请求都会得不到响应。

遇见这样的需求，在统一规划业务逻辑后，调用`setTimeout()`的效果更好。

#### 4. 多线程编程

我们谈论JavaScript的时候，通常谈的是单一线程上执行的代码，这在浏览器中指的是JavaScript执行线程与UI渲染共用的一个线程；在Node中，只是没有UI渲染的部分，模型基本相同。对于服务器而言，如果服务器是多核CPU，单个Node进程实质上是没有充分利用多核CPU的。随着现今业务的复杂度，对于多核CPU利用的要求也越来越高。浏览器提出了Web Workers，它通过将JavaScript执行与UI渲染分离，可以很好地利用多核CPU为大量计算服务。同时前端Web Workers也是一个利用消息机制合理适用多核CPU的理想模型。

遗憾在于前端浏览器存在对标准的滞后性，Web Workers并没有广泛应用起来。另外Web Workers能解决利用CPU和减少阻塞UI渲染，但是不能解决UI渲染的效率问题。Node借鉴了这个模式，`child_process`是其基础API，`cluster`模块是更深层次的应用。借助Web Workers 的模式，开发人员要更多地去面临跨线程的编程，这对于以往的JavaScript编程经验是较少考虑的。在第九章中，我们将详细分析Node的进程，以展开这部分内容。

#### 5. 异步转同步

习惯异步编程的同学，也许能够从容面对异步编程带来的副产品，比如嵌套回调、业务分散等问题。Node提供给了绝大部分的异步API和少量的同步API，偶尔出现的同步需求将会因为没有同步API让开发者无所适从。目前，Node中视图同步式编程，但并不能得到原生支持，需要借助库或者编译等手段来实现。但对于异步调用，通过良好的流程控制，还是能够将逻辑梳理称顺序式的形式。

## 4.3 异步编程解决方案

前面列举了因异步编程带来的一些问题，与异步编程提升性能成果相比，编程过程看起来似乎没有想象中那么美好，但是事实却也没有那么糟糕。与问题相比，解决问题的方案总是更多，本节将展开各个典型的解决方案。

目前异步编程主要解决方案有如下3种：

- 事件发布/订阅模式
- Promise/Deferred模式
- 流程控制库

### 4.3.1 事件发布/订阅模式

事件监听器模式是一种广泛用于异步编程的模式，是回调函数的事件化，又称发布/订阅模式。Node自身提供的[events模块](http://nodejs.org/docs/latest/api/events.html)是发布/订阅模式的一个简单实现，Node种部分模块都继承自它，这个模块比前端浏览器种的大量DOM事件简单，不存在事件冒泡，也不存在`preventDefault()`、`stopPropagation()`和`stopImmediatePropagation()`等控制事件传递的方法。它具有`addListener/on()`、`once()`、`removeListener()`、`removeAllListeners()`和`emit()`等基本的事件监听模式的方法实现。事件发布/订阅模式的操作极其简单，示例代码如下：

```js
// 订阅
emitter.on('event1', function (message) {
  console.log(message);
});

// 发布
emitter.emit('event1', 'I am Message!');
```

可以看到，订阅事件就是一个高阶函数的应用。事件发布/订阅模式可以实现一个事件与多个回调函数的关联，这些回调函数又称为事件监听器。通过`emit()`发布事件后，消息会立即传递给当前事件的所有侦听器执行。侦听器可以很灵活地添加和删除，使得事件和具体处理逻辑之间可以很轻松地关联和解耦。

事件发布/订阅模式自身并无同步和异步调用问题，但是在Node种，`emit()`调用多半是伴随事件循环而异步触发地，所以我们说事件发布/订阅广泛应用于异步编程。

事件发布/订阅模式常常用来解耦业务逻辑，事件发布者无需关注订阅者地监听器如何实现业务逻辑，甚至不用关注有多少个侦听器存在，数据通过消息地方式可以很灵活地传递。在一些经典场景中，可以通过事件发布/订阅模式进行组件封装，将不变的部分封装在组件内部，将容易变化、需要自定义的部分通过事件暴露给外部处理，这是一种典型的逻辑分离方式。在这种事件发布/订阅式组件中，事件的设计非常重要，因为它关乎外部调用组件时是否优雅，从某种角度来说事件的设计就是组件的接口设计。

从另一个角度来看，事件监听器模式也是一种钩子（hook）机制，利用钩子导出内部数据或状态给外部调用者。Node中很多对象大多具有黑盒的特点，功能点少，如果不通过事件钩子的形式，我们就无法获得对象在运行期间的中间值或内部状态。这种通过事件钩子的方式，可以适编程者不用关注组件是如何执行的，只需要关注在需要的事件点上即可。下面的HTTP请求是典型场景：

```js
var options = {
  host: 'www.google.com',
  port: '/upload',
  method: 'POST'
};
var req = http.request(options,function (res) {
  console.log('STATUS: ' + res.statusCode);
  console.log('HEADERS: ' + JSON.stringify(res.headers));
  res.setEncoding('utf8');
  res.on('data', function (chunk) {
    console.log('BODY: ' + chunk);;
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

在这段HTTP请求的代码中，程序员只需要将实现放在`error`、`data`、`end`这些业务事件点上即可，至于内部的流程如何，无需过于关注。

值得一提的是，Node对事件发布/订阅的机制做了一些额外的处理，这大多是基于健壮性而考虑的。下面为两个具体细节点：

- 如果对一个事件添加超过10个侦听器，将会得到一条警告。这一处设计与Node自身单线程运行有关，设计者认为侦听器太多可能导致内存泄漏，所以存在这样一条警告。调用`emitter.setMaxLinsteners(0);`可以将这个限制去掉。另一方面，由于事件发布会引起一系列侦听器执行，所以事件相关的侦听器过多，可能存在过多占用CPU的场景。
- 为了处理异常，`EventEmitter`对象对`error`事件进行了特殊对待。如果运行期间的错误触发了`error`事件，`EventEmitter`会检查是否有对`error`事件添加过侦听器。如果添加了，这个错误将会交由该侦听器处理，否则这个错误会作为异常抛出。如果外部没有捕获这个异常，将会引起线程退出。一个健壮的`EventEmitter`实例应该对`error`事件做处理。

#### 1.继承events模块

实现一个继承`EventEmitter`的类是十分简单的，以下代码是Node中`Stream`对象继承`EventEmitter`的例子：

```js
var events = require('events');

function Stream() {
  events.EventEmitter.call(this);
}
util.inherits(Steam, events.EventEmitter);
```

Node在`util`模块中封装了继承的方法，所以此处可以很便利地调用。开发者可以通过这样地方式轻松继承`EventEmitter`类，利用事件机制解决业务问题。在Node提供地核心模块中，有近半数都继承自`EventEmitter`。

#### 2.利用事件队列解决雪崩问题

在事件订阅/发布模式中，通常也有一个`once()`方法，通过它添加的侦听器只能执行一次，在执行之后就会将它与事件的关联移除。这个特性常常可以帮助我们过滤一些重复性的事件响应。下面我们介绍下如何采用`once`解决雪崩问题。

在计算机中，缓存由于放在内存中，访问速度十分快，常常用于加倍数据访问，让绝大多数的请求不必重复去做一些低效的数据读取。所谓雪崩问题，就是在高访问量、大量并发量的情况下，缓存失效的情景，此时大量的请求同时涌入数据库中，数据库无法同时承受如此大的查询请求，进而往前影响网站整体的响应速度。

```js
var select = function (callback) {
  db.select("SQL", function (results) {
    callback(results);
  });
}
```

如果站点刚好启动，这是缓存中是不存在数据的，而访问量巨大，同义句SQL会被发送到数据库中反复查询，会影响服务的整体性能。一种改进方案是添加一个状态锁，相关代码：

```js
var status = 'ready';
var select = function (callback) {
  if(status == 'ready') {
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

这里我们利用了`once()`方法，将所有请求地回调压入事件队列中，利用其执行一次就会将监视器移除的特点，保证每一个回调都只会被执行一次。对于相同的SQL语句，保证在同一个查询开始到结束的过程中永远只有一次。SQL在进行查询时，新到来的相同调用只需要在队列中等待数据就绪即可，一旦查询结束，得到的结果可以被这些调用共同使用。这种方式能节省重复的数据库调用产生的开销。由于Node单线程执行的原因，此处无需担心状态同步问题。这种方式其实也可以应用到其它远程调用的场景中，即使外部没有缓存策略，也能有效节省重复开销。

慈湖可能因为存在侦听器过多引发的警告，需要调用`setMaxListeners(0);`移除警告，或者设置更大的警告阈值。`once`方法产生的效果，也可以在注明的Gearman异步应用框架中实现。但在JavaScript中实现这个效果十分容易。

#### 3.多异步之间的协作方案

事件发布/订阅模式有着它的优点。利用高阶函数的优势，侦听器作为回调函数可以随意添加和删除，它帮助开发者轻松处理随时可能添加的业务逻辑。也可以隔离业务逻辑，保持业务逻辑单元的职责单一。一般而言，事件与侦听器的关系是一对多，但是在异步编程中，也会出现事件与侦听器多对一的情况，也就是说一个业务逻辑可以依赖两个通过回调函数或事件传递的结果。前面提及嵌套过深的原因即是如此。

这里我们尝试通过原生代码解决“难点2”中为了最终结果的处理而导致可以并行调用但实际只能串行执行的问题。我们的目标是既要享受异步I/O带来的性能提升，也要保持良好的编码风格。这里以渲染页面所需要的模板读取、数据读取和本地化资源读取为例，简单介绍下，相关代码：

```js
var count = 0;
var results = {};
var done = function(key, vaue) {
  results[key] = value;
  count++;
  if(count === 3) {
    // 渲染页面
    render(results);
  }
};

fs.readFile(template_path, 'utf8', function (err, template) {
  done('template', template);
});
db.query(sql, function (err, data) {
  done('data',data);
});
l1On.get(function (err, resources) {
  done('resources', resources);
});
```

由于多个异步场景中回调函数的执行并不能保证顺序，且回调函数之间互相没有任何交集，所以需要借助一个第三方函数和第三方变量来处理异步协作的结果。通常我们把这个用于检测次数的变量叫**哨兵变量**，聪明的你也许已经想到利用偏函数来处理哨兵变量与第三方函数的关系了：

```js
var after = function (times, callback) {
  var count = 0, results = {};
  return function (key, value) {
    results[key] = value;
    count++;
    if(count === times) {
      callback(results);
    }
  }
}
var done = after(times,render);
```

这种方案实现了多对一的目的。如果业务继续增长，我们依然可以继续利用发布/订阅方式来完成多对多的方案，相关代码：

```js
var emitter = new events.Emitter();
var done = after(times, render);
emitter.on('done', done);
emitter.on('done', other);

fs.readFile(template_path, 'utf8', function (err, template) {
  emitter.emit('done', 'template', template);
})
db.query(sql, function (err, data) {
  emitter.emit('done', 'data', data);
});
l1On.get(function (err, resources) {
  emitter.emit('done', 'resources', resources);
});
```

这种方案结合了前者用简单的偏函数完成多对一的收敛和事件订阅/发布模式中一对多的发散。在上面的方法中，有一个令调用者不那么舒服的问题，就是调用者要去准备这个`done()`函数，以及在回调函数中把数据一个个提取出来，再进行处理。

另一个方案则是由笔者自己写的EventProxy模块，它是对事件订阅/发布模式的扩充，可以自由订阅组合事件。由于依旧采用的事件订阅/发布模式，与Node十分契合，相关代码：

```js
var proxy = new EventProxy();
proxy.all('template', 'data', 'resources',function (template,data,resources) {
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
})
```

EventProxy提供了一个`all()`方法来订阅多个事件，当每个事情都被触发后，侦听器才会执行。另外的一个方法是`tail()`方法，它与`all()`方法的区别在于`all()`方法的侦听器在满足条件之后只会执行一次，`tail()`方法的侦听器则在满足条件时执行一次之后，如果组合事件中的某个事件中的某个事件被再次触发，侦听器回用最新的数据继续执行。

`all()`方法带来的另一个改进则是：在侦听器中返回数据的参数列表与订阅组合事件的事件列表是一致对应的。

除此之外，在异步的场景下，我们常常需要从一个接口多次读取数据，此时触发的事件名或许是相同的。EventProxy提供了`after()`方法来实现事件在执行多少次后执行侦听器的单一事件组合订阅方式，示例代码：

```js
var proxy = new EventProxy();
proxy.after('data', 10, function (datas) {
  // TODO
});
```

这段代码表示执行10次data事件后，执行侦听器。这个侦听器得到的数据为10次按事件触发次序排序的数组。EventProxy模块除了可以应用于Node中之外，还可以用在前端浏览器中。

#### 4.EventProxy原理

EventProxy 来自于Backnone的事件模块，Backnone的事件模块是Model、View模块的基础功能，在前端有广泛使用。它在每个非all事件触发时都会触发一次all事件，相关代码：

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

EventProxy则是一个将all当作一个事件流的拦截曾，在其中注入一些业务来处理单一事件无法解决的异步处理问题。类似的扩展还有`all()`、`tail()`、`after()`、`not()`和`any()`等。

#### 5.EventProxy的异常处理

EventProxy在事件发布/订阅模式的基础上还完善了异常处理。在异步方法中，异常处理需要占用一定比例的精力。在过去一段时间内，我们口试通过额外添加error事件来进行异常统一处理，大致代码：

```js
export.getContent = function (callback) {
  var ep = new EventProxy();
  ep.all('tpl', 'data', function (tpl, data) {
    // 成功回调
    callback(null, {
      template: tpl,
      data: data
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
    if(err) {
      // 一旦发生异常，一律交给error事件的处理函数处理
      return ep.emit('error',err);
    }
    ep.emit('tpl', content);
  });
  db.get(sql, function (err, result) {
    if(err) {
      // 一旦发生异常，一律交给 error事件的处理函数处理
      return ep.emit('error', err);
    }
    ep.emit('data', result);
  });
};
```

因为异常处理的原因，代码量一下子躲起来了，而EventProxy在实践过程中改进了这个问题，相关代码：

```js
exports.getContent = function (callback) {
  var ep = new EventProxy();
  ep.all('tpl', 'data', function (tpl, data) {
  	// 成功回调
    callback(null, {
      template: tpl,
      data: data
    });
  });
  // 绑定错误处理函数
  ep.fail(callback);
  
  fs.readFile('template.tpl', 'utf8', dp.done('tpl'));
  db.get(sql, ep.done('data'));
};
```

在上述代码中，EventProxy提供了`fail()`和`done()`这两个实例方法来优化处理异常，使得开发者将精力关注在业务部分，而不是在异常捕获上。

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

这里的`fail()`和`done()`十分类似于Promise模式中的`fail()`和`done()`。换句话而言，这可以算是事件发布/订阅模式向Promise模式的借鉴。这样的完善既提升了程序的健壮性，同时也降低了代码量。

### 4.3.2 Promise/Deferred模式

使用事件的方式时，执行流程需要被预先设定。即便是分支，也需要预先设定，这是由发布/订阅模式的运行机制决定的。下面为普通的Ajax调用：

```js
$.get('/api', {
  success: onSuccess,
  error: onError,
  complete: onComplete
});
```

再上面的异步调用中，必须严谨地设置目标。那么是否有一种先执行异步调用，延迟传递处理地方式呢？答案时Promise/Deferred模式。

Promise/Deferred模式再JavaScript框架中最早出现于Dojo的代码中，被广为所知则是来自于jQuery 1.5 版本，该版本几乎重写了Ajax部分，使得调用Ajax时可以通过如下方式进行：

```js
$.get('/api')
	.success(onSuccess)
	.error(onError)
	.complete(onComplete);
```

这使得及时不调用`sucess()`、`error()`等方法，Ajax也会执行，这样的调用方式比预先传入回调让人觉得舒适一些。在原始的API中，一个事件只能处理一个回调，而通过Defferred对象，可以对使劲按加入任意的业务处理逻辑，示例代码：

```js
$.get('/api')
	.success(onSuccess1)
	.success(onSuccess2);
```

Promise/Deferred模式在2009年时被Kris Zyp抽象为一个提议草案，发布在CommonJS规范中。随着使用Promise/Deferred模式的应用逐渐增多，CommonJS草案目前已经抽象出了Promises/A、Promises/B、Promises/D这样经典的异步Promise/Deferred模型，这使得异步操作可以以一种优雅的方式出现。

异步的广度使用使得回调、嵌套的出现，但是一旦出现深度的嵌套，就会让编程的体验变得不愉快，而Promise/Deferred模式在一定的程度上缓解了这个问题。这里我们将着重介绍Promise/A来以点代面介绍Promise/Deferred模式。

#### 1. Promises/A

Promise/Deferred模式其实包含两部分，即Promise和Deferred。这里暂且不提两者的区别是什么，先来看看Promises/A的行为把。

Promises/A提议对单个异步操作做出了这样的抽象定义，具体如下

- Promise操作只会处于3中状态中的一种：未完成态、完成态和失败态。
- Promise的状态只会出现从未完成态向完成态或失败态转化，不能逆反。完成态和失败态不能相互转换。
- Promise的状态一旦转化，将不能再被更改。

在API的定义上，Promises/A提议是比较简单的。一个Promise对象只要具备`then()`方法即可。但是对于`then()`方法，有以下简单的要求。

- 接收完成态、失败态的回调方法。在操作完成或出现错误时，将会调用对应方法。
- 可选地支持progress事件回调作为第三个方法。
- `then()`方法只接收`function`对象,其余对象将被忽略。
- `then()`方法继续返回`Promise`对象，以实现链式调用。

`then()`方法定义如下：

```js
then(fulfilledHandler, errorHandler, progressHandler)
```

为了演示Promises/A提议，这里我们尝试通过继承Node地events模块来完成一个简单地实现，代码如下：

```js
var Promise = function () {
  EventEmitter.call(this);
};
util.inherits(Promise, EventEmitter);

Promise.prototype.then = function (fulfilledHandler, errorHandler, progressHandler) {
  if(typeof fulfilledHandler === 'function') {
    // 利用once()方法，保证成功回调只执行一次
    this.once('success', fulfilledHandler);
  }
  if(typeof errorHandler === 'function') {
    // 利用once方法，保证异常回调只执行一次
    this.once('error', errorHandler);
  }
  if(typeof progressHandler === 'function') {
    this.on('progress', progressHandler);
  }
  return this;
}
```

这里看到`then()`方法所做地事情是将回调函数存放起来。为了完成整个流程，还需要触发执行这些回调函数地地方，实现这个功能地对象被称为Deferred，即延迟对象，示例代码：

```js
var Deferred = function () {
  this.state = 'unfulfilled';
  this.promise = new Promise();
}

Deferred.prototype.resolve = function (obj) {
  this.state = 'fulfilled';
  this.promise.emit('success',obj);
};

Deferred.prototype.reject = function (err) {
  this.state = 'failed';
  this.promis.emit('error', err);
};

Deferred.prototype.progress = function (data) {
  this.promise.emit('progress', data);
}
```

利用Promiese/A提议地模式，我们可以对一个典型地响应对象进行封装，相关代码：

```js
res.setEncoding('utf8');
res.on('data',function (chunk) {
  console.log('BODY: ' + chunk);
});
res.on('end', function () {
  // Done
});
res.on('error', function (err) {
  // Error
});
// 上述代码转换为如下简略形式
res.then(function () {
  // Done
}, function (err) {
  // Error
}, function (chunk) {
  console.log('BODY: ' + chunk);
});
```

要实现如此简单地API，只需要简单改造以下即可：

```js
var promisify = function (res) {
  var deferred = new Deffered();
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

如此就得到了简单的结果。这里返回`deferred.promise`的目的时为了不让外部程序调用`resolve()`和`reject()`方法，更改内部状态的行为交给定义者处理。下面为定义好Promise后的调用示例：

```js
promisify(res).then(function () {
  // Done
}, function (err) {
  // Error
}, function (chunk) {
  console.log('BODY: ' + chunk);
});
```

这里回到Promise和Deferred的差别上。从上面的代码可以看出，Deferred主要用于内部，用户维护异步模型的状态；Promise则作用于外部，通过`then()`方法暴露给外部以添加自定义逻辑。





