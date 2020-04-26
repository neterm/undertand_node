# 模块机制

---

- [2.1 CommonJS 规范](#21-commonjs-规范)
- [2.2 Node 的模块实现](#22-node的模块实现)
- [2.3 核心模块](#23-核心模块)
- [2.4 C/C++扩展模块](#24-cc扩展模块)
- [2.5 模块调用栈](#25-模块调用栈)
- [2.6 包和 NPM](#26-包和npm)
- [2.7 前后端共用模块](#27-前后端共用模块)
- [2.8 总结](#28-总结)

---

JavaScript 自诞生以来，曾经没有人拿它当成一门真正的编程语言，认为它不过是一种网页小脚本而已，在 web 1.0 时代，这种脚本语言在网络中主要有两个作用广为流传，一个是表单校验，另一个是网页特效。另一方面，由于仓促地被创造出来，所以它自身地各种缺陷也被各种编程人员广为诟病。知道 web 2.0 时代，前端工程师利用它大大提升了网页上地用户体验。在这个过程中，B/S 应用展现出比 C/S 应用优越地地方。因此 JavaScirpt 被广泛重视起来。

在 web 2.0 流行的过程中，各种前端库和框架被开发出来，他们最初用于兼容各个版本的浏览器，随后随着更多的用户需求在前端被实现，JavaScript 也从表单校验迁到应用开发级别上。在这个过程中，它大致经历了工具类库、组件库、前端框架、前端应用的变迁。

经历了常常的后天努力过程，JavaScript 不断被类聚和抽象，以更好的组织业务逻辑。从另一个角度而言，它也道出了 JavaScript 先天就缺乏的一项功能：模块。

在其它高级语言中，Java 有类文件，Python 有`import`机制，Ruby 有`require`，PHP 有`include`和`require`。而 JavaScript 通过`<script>`标签引入代码的方式显得杂乱无章，语言自身毫无组织和约束能力。人们不得不用命名空间等方式认为约束代码，以求达到安全和易用的目的。

但是看起来凌乱的 JavaScript 编程现状并不代表社区没有进步，JavaScript 的本地化编程之路一直在探索中。在 Node 出现之前，服务端 JavaScript 基本没有市场，与欣欣向荣的前端 JavaScript 相比，Rhino 等后端 JavaScript 运行环境基本只是用于小工具，但是经历十多年发展后，社区也为 JavaScript 制定了响应的规范，其中 CommonJS 规范的提出算是最为重要的里程碑。

## 2.1 CommonJS 规范

CommonJS 规范为 JavaScript 制定了一个美好的愿景——希望 JavaScript 能够在任何地方运行。

### 2.1.1 CommonJS 的出发点

在 JavaScript 的发展历程中，它主要在浏览器前端发光发热。由于官方规范（ECMAScript)规范化的时间较早，规范涵盖的范畴较小。这些规范中包含语法、类型、上下文、表达式、声明、方法、对象等语言的基本要素。在实际应用中，JavaScript 的表现能力取决于宿主环境的 API 支持程度。在 web 1.0 时代，只有对 DOM、BOM 的基本支持。随着 web 2.0 的推进，HTML5 崭露头角，它将 web 网页应用带进 web 应用时代，在浏览器中出现了更多、更强大的 API 供 JavaScript 调用，这得感谢 W3C 组织对 HTML5 规范的推进以及各大浏览器厂商对规范的大力支持。但是，web 在发展，浏览器出现了更多的标准 API，这些过程发生在前端，后端 JavaScript 的规范却远远落后。对于 JavaScript 自身而言，他的规范依然是薄弱的，还有以下缺陷。

- **没有模块系统**
- **标准库较少**。ECMAScript 仅定义了部分核心库，对于文件系统，I/O 流等常见需求却没有标准的 API。就 HTML5 的发展状况而言，W3C 标准化在一定意义上就是在推进这个过程，但是它仅限于浏览器。
- **没有标准接口**。在 JavaScript 中，几乎没有定义过如 web 服务器或者数据库之类的标准统一接口。
- **缺乏包管理系统**。这导致 JavaScript 应用中基本没有自动浇在和安装依赖的能力。

CommonJS 规范的提出，主要是为了弥补当前 JavaScript 没有标准的缺陷，以达到像 python、Ruby 和 Java 等具备开发大型应用的基础能力，而不是停留在小脚本程序的阶段。他们期望那些用 CommonJS API 写出的应用可以具备跨宿主环境执行的能力，这样不仅可以利用 JavaScript 开发富客户端应用，而且还可以编写以下应用。

- 服务端 JavaScrip 应用程序。
- 命令行工具。
- 桌面图形界面应用程序。
- 混合应用（Titanium 和 Adobe AIR 等形式的应用）。

如今，CommonJS 中的大部分规范虽然依旧是草案，但是已经初显成效，为 JavaScript 开发大型应用程序指明了一条非常棒的道路。目前，它依旧在成长中，这些规范涵盖了模块、二进制、Buffer、字符集、I/O 流、进程环境、文件系统、套接字、单元测试、Web 服务器网关接口、包管理等。

理论和实践总是相互影响和促进的，Node 能以一种比较成熟的姿态出现，离不开 CommonJS 规范的影响。在服务器端，CommonJS 能以一种寻常的姿态写进各个公司的项目代码中，离不开 Node 优异的表现。实现的优良表现离不开规范最初优秀的设计，规范因实现的推广而得以普及。

Node 借助 CommonJS 的 Modules 规范实现了一套非常易用的模块系统，NPM 对 Packages 规范的完好支持使得 Node 应用在开发过程中事半功倍。

### 2.1.2 CommonJS 的模块规范

CommonJS 对模块的定义十分简单，主要分为模块引用、模块定义和模块标识 3 个部分。

#### 1.模块引用

模块引用的示例代码如下：

```javascript
var math = require('math');
```

在 CommonJS 规范中，存在`require()`方法，这个方法接受模块标识，以此引入一个模块的 API 到当前上下文中。

#### 2. 模块定义

在模块中，上下文提供`require()`方法来引入外部模块。对应引入的功能，上下文提供了`exports`对象用于导出当前模块的方法或变量，并且它是唯一导出的出口。在模块中，还存在一个`Module`对象，它代表模块自身，而`exports`是`Module`的属性。在 Node 中，一个文件就是一个模块，将方法挂载在`exports`对象上作为属性即可定义导出的方式：

```javascript
// math.js
exports.add = function () {
  var sum = 0,
    i = 0,
    args = arguments,
    l = args.length;
  while (i < l) {
    sum += args[i++];
  }
  return sum;
};
```

在另一个文件中，我们通过`require()`方法引入模块后，就能调用定义的属性和方法了：

```javascript
//program.js
var math = require('math');
exports.increment = function (val) {
  return math.add(val, 1);
};
```

#### 3. 模块标识

模块标识其实就是传递给`require()`方法的参数，它必须是符合小驼峰命名的字符串，或者以`.`、`..`开头的相对路径或绝对路径。它可以没有文件后缀`.js`。

模块的定义十分简单，接口也十分简洁。它的意义在于将类聚的方法和变量等限定在私有的作用域中，同时支持引入和导出功能以顺畅地连接上下游依赖。每个模块具有独立地空间，他们互不干扰，在引用时，也显得干净利落。

CommonJS 构建地这套模块导出和引入机制使得用户完全不必考虑变量污染，命名空间等方案与之相比相形见绌。

## 2.2 Node 的模块实现

Node 在实现中并非完全按照规范实现的，而是对模块规范进行了一定的取舍，同时也增加了少许自身需要的特性。尽管规范中`exports`、`require`和`module`听起来十分简单，但是 Node 在实现它们过程中究竟经历了什么，这个过程需要知晓。

在 Node 中引入模块，需要经历如下三个步骤：

- （1）路径分析
- （2）文件定位
- （3）编译执行

Node 中的模块分为两类：一类是 Node 提供的模块，称为核心模块；另一类是用户编写的模块，称为文件模块。

- 核心模块部分在 Node 的源代码编译过程中，编译进了二进制执行文件。在 Node 进程启动的时候，部分核心模块就被直接加载进内存中，所以这部分核心模块引入时，文件定位和编译执行这两个步骤都可以省略掉，并且在路径分析中优先判断，所以它的加载速度是最快的。
- 文件模块则是在运行时动态加载，需要完整的路径分析、文件定位、编译执行过程，速度比核心模块慢。

接下来我们展开详细的模块加载过程

### 2.2.1 优先从缓存加载

展开介绍路径分析和文件定位之前，我们需要知晓，与前端浏览器缓存静态脚本用于提高性能一样，Node 对引入过的模块都会进行缓存，以减少二次引入时的开销。不同的是，浏览器仅仅缓存文件，而 Node 缓存的是编译和执行之后的对象。

不论是核心模块还是文件模块，`require()`方法对相同模块的二次加载都一律采用缓存优先的方式，这是`第一级优先`的。不同之处在于核心模块的缓存检查先于文件模块的缓存检查。

### 2.2.2 路径分析和文件定位

因为标识符有几种形式，对于不同的标识符，模块的查找和定位有不同程度上的差异。

#### 1. 模块标识符分析

前面提到过，`require()`方法接收一个标识符作为参数。在 Node 实现中，正是基于这样一个标识符进行模块查找的。模块标识符在 Node 中主要分为以下几类。

- 核心模块，如`http`、`fs`、`path`等。
- `.`或`..`开始的相对路径文件模块。
- 以`/`开始的绝对路径文件模块。
- 非路径形式的文件模块，如自定义的`connect`模块。

##### 核心模块

核心模块的优先级仅次于缓存加载，它在 Node 的源代码编译过程中已经编译为二进制代码，其加载过程最快。

如果试图加载一个与核心模块标识符相同的自定义模块，那是不会成功的。如果自己编写了一个`http`用户模块，想要加载成功，必须选择一个不同的标识符或者使用路径的方式。

##### 路径形式的文件模块

以`.`、`..`和`/`开始的标识符，这里都被当作文件模块来处理。在分析路径模块时，`require()`方法会将路径转换为真实的路径，并以真实路径作为索引，将编译执行后的结果存放到缓存中，以使二次加载更快。

由于文件模块给 Node 指明了确切的文件位置，所以查找过程中可以节省大量时间，其加载速度慢于核心模块。

##### 自定义模块

自定义模块指的是非核心模块，也不是路径形式的标识符。它是一种特殊的文件模块，可能时一个文件或者包的形式。这类模块的查找是最费时的，也是所有方式中最慢的一种。

在介绍自定义模块的查找方式之前，我们需要先介绍下`模块路径`这个概念。

模块路径是 Node 在定位文件模块的具体文件时指定的查找策略，具体表现为一个路径组成的数组。关于这个路径的生成规则，我们可以手动尝试一番。

- （1）创建`module_path.js`文件，其内容为`console.log(module.paths)`；
- （2）将其放到任意一个目录中然后执行 `node module_path.js`。

在 Linux 下，你可能得到的是这样一个数组输出：

```javascript
[
  '/home/jackson/research/node_modules',
  '/home/jackson/node_modules',
  '/home/node_modules',
  '/node_modules',
];
```

而在 Windows 下，也许是这样：

```javascript
['c:\\nodejs\\node_modules', 'c:\\node_modules'];
```

可以看出，模块路径的生成规则：

- 当前目录下的`node_modules`目录
- 父目录下的`node_modules`目录
- 父目录的父目录下的`node_modules`目录
- 沿路径向上逐级递归，直到根目录下的`node_modules`目录。

它的生成方式与 JavaScript 的原型链或作用域链的查找方式十分类似。在加载过程中，Node 会逐个尝试模块路径中的路径，直到找到目标文件为止。可以看出，当前文件的路径越深，模块查找耗时会越多，这是自定义模块的加载速度最慢的原因。

#### 2. 文件定位

从缓存加载的优化策略使得二次引入时不需要路径分析、文件定位和编译执行的过程，大大提高了再次加载模块时的效率。

但在文件的定位过程中，还有一些细节需要注意，这主要包括文件扩展名的分析、目录和包的处理。

##### 文件扩展名分析

`require()`在分析标识符的过程中，会出现标识符中不包含文件扩展名的情况。CommonJS 模块规范也允许在标识符中不包括文件扩展名，在这种情况下，Node 会按照`.js`、`.node`、`.json`的次序补足扩展名，依次尝试。

在尝试过程中，需要调用 fs 模块同步阻塞式地判断文件是否存在。因为 Node 是单线程地，所以这里是一个会引起性能问题地地方。小诀窍是：如果是`.node`和`.json`，在传递给`require()`的标识符中带上扩展名，会加快一点速度。另一个诀窍是：同步配合缓存，可以大幅度缓解 Node 单线程中阻塞式调用的缺陷。

##### 目录分析和包

在分析标识符过程中，`require()`通过分析文件扩展名之后，可能没有查找到对应文件，但却得到一个目录，这在引入自定义模块和逐个模块路径进行查找是经常会出现，此时 Node 会将目录当成一个包来处理。

在这个过程中，Node 对 CommonJS 包规范进行了一定程度的支持。首先，Node 在当前目录下查找`package.json`（CommonJS 包规范定义的包描述文件），通过 JSON.parse()解析出包描述对象，从中去除`main`属性指定的文件名进行定位。如果文件名缺少扩展名，将会进入扩展名分析的步骤。

而如果 main 属性指定的文件名错误，或者压根没有`package.json`文件，Node 会将 index 当作默认文件名，然后依次检查`index.js`、`index.node`、`index.json`。

如果目录分析的过程中没有定位成功任何文件，则自定义模块进入下一个模块路径进行查找。如果模块路径数组都被遍历完毕，依然没有找到目标文件，则会抛出查找失败的异常。

## 2.2.3 模块编译

在 Node 中，每个文件模块都是一个对象，它的定义如下：

```javascript
function Module(id, parent) {
  this.id = id;
  this.exports = {};
  this.parent = parent;
  if (parent && parent.children) {
    parent.children.push(this);
  }

  this.filename = null;
  this.loaded = false;
  this.children = [];
}
```

编译和执行是引入文件模块的最后一个阶段。定位到具体的文件后，Node 会新建一个模块对象，然后根据路径载入并编译。对于不同的文件扩展名，其载入的方法也有所不同：

- `.js`文件。通过 fs 模块同步读取文件后编译执行。
- `.node`文件。这时 C/C++编写的扩展文件，通过`dlopen()`方法加载最后编译生成文件。
- `.json`文件。通过 fs 模块同步读取文件后，用`JSON.parse()`返回结果。
- 其余扩展名文件。它们都被当作称`.js`文件载入。

每一个编译成功的模块都会将其文件路径作为索引缓存在 Module.\_cache 对象上，以提高二次引入的性能。

根据不同的文件扩展名，Node 调用不同的读取方式，如`.json`文件的调用如下：

```javascript
// Native extension for .json
Module._extensions['.json'] = function (module, filename) {
  var content = NativeModule.require('fs').readFileSync(filename, 'utf8');
  try {
    module.exports = JSON.parse(stripBOM(content));
  } catch (err) {
    err.message = filename + ': ' + err.message;
    throw err;
  }
};
```

其中，`Module._extensions`会被赋值给`require()`的`extensions`属性，所以通过代码中访问`require.extensions`可以知道系统中已有的扩展加载方式。编写如下代码测试下：

```javascript
console.log(require.extensions);
```

得到的执行结果如下：

```javascript
{'.js':[Function], '.json': [Function], '.node': [Function]}
```

如果相对自定义的扩展名进行特殊的加载，可以通过类似`require.extensions['.ext']`的方式实现。早期的 CoffeeScript 文件就是通过添加`require.extensions['.coffee']`扩展的方式来实现加载的。但是从 v0.10.6 版本开始，官方不鼓励通过这种方式来进行自定义扩展名的加载，而是期望先将其他语言或文件编译成 JavaScript 文件后再加载，这样做的好处在于不将繁琐的编译加载等过程引入 Node 的执行过程中。

再确定文件的扩展名之后，Node 将调用具体的编译方式来将文件执行后返回给调用者。

### 1. JavaScript 模块的编译

回到 CommonJS 模块规范，我们知道每一个模块文件中存在着`require`、`exports`、`module`这三个变量，但是他们在模块文件中并没有定义，是从何而来呢？甚至 Node 的 API 文档中，我们知道每个模块中还有`__filename`、`__dirname`这两个变量的存在，它们又是从何而来呢？我们把直接定义模块的过程放诸到浏览器端，就会存在污染全局变量的情况。

事实上，在编译过程中，Node 对获取的 JavaScript 文件内容进行了头尾包装。在头部添加了`(function (exports, require, module, __filename, __dirname){\n`，在尾部添加了`});`。最终一个正常的 JavaScript 就被包装成了如下样子：

```javascript
(function (exports, require, module, __filename, __dirname) {
  var math = require('math');
  exports.area = function (radius) {
    return Math.PI * radius * radius;
  };
});
```

这样每个模块之间都进行了作用域隔离。包装之后的代码会通过`vm`原生模块中的`runInThisContext()`方法执行（类似`eval`，只是具有明确上下文，不污染全局），返回一个具体的`function`对象。最后将当前模块对象的`exports`属性、`require()`方法、`module`（模块对象自身），以及在文件定位中得到的完整文件路径和文件目录作为参数传递给这个`function()`执行。

这就是这些变量并没有定义在每个模板文件中却存在的原因。在执行后，模块的`exports`属性被返回给调用方。`exports`属性上的任何方法和属性都可以被外部调用到，但是模块中的其余变量或属性则不可直接被调用。

至此，`require`、`exports`、`module`的流程已经完整，这就是 Node 对 CommonJS 模块规范的实现。

此外，许多初学者都曾经纠结过为何存在`exports`的情况下，还存在`module.exports`。理想情况，只要赋值给`exports`即可：

```javascript
exports = function () {
  // My Class
};
```

但是通常会得到一个失败的结果。其原因在于，`exports`对象是通过形参的方式传入的，直接赋值形参会改变形参的引用，但并不能改变作用域外的值。

```javascript
var change = function (a) {
  a = 100;
  console.log(a); // => 100
};

var a = 10;
change(a);
console.log(a); // => 10
```

如果要达到`require`引入一个类的效果，请赋值给`module.exports`对象。这个迂回方案不改变形参的引用。

### 2. C/C++模块的编译

Node 调用`process.dlopen()`方法进行加载和执行。在 Node 的架构下，`dlopen()`方法在 Windows 和\*nix 平台下分别有不同的实现，通过`libuv`兼容层进行了封装。

实际上，`.node`的模块文件不需要编译，因为它是编写 C/C++模块之后编译产生的，所以这里只需要加载和执行的过程。在执行过程中，模块的`exports`对象与`.node`模块产生联系，然后返回给调用者。

C/C++模块给 Node 使用者带来的优势主要是执行效率方面的，劣势则是 C/C++模块的编写门槛比 JavaScript 高。

### 3.JSON 文件的编译

`.json`文件的编译是三种编译方式中最简单的。Node 利用` fs``模块同步读取JSON文件的内容之后，调用 `JSON.parse`方法得到对象，然后将它赋值给模块对象的`exports`,供外部使用

JSON 文件在用作项目的配置文件时比较有用，如果你定义了一个 JSON 文件作为配置，那就不必要调用`fs`模块去异步读取数据和解析，直接调用`require()`引入即可。此外，你还可以享受模块缓存的便利，并且二次引入时也没有性能影响。

这里指的模块编译都是只文件模块，即用户自己编写的模块。

## 2.3 核心模块

前面提到，Node 的核心模块在编译成可执行文件的过程中被编译称二进制文件。核心模块其实分为 C/C++编写的和 JavaScript 编写的两部分，其中 C/C++文件存放在 Node 项目的`src`目录下，JavaScript 文件存放在`lib`目录下。

### 2.3.1 JavaScript 核心模块编译过程

在编译所有 C/C++文件之前，编译程序将所有的 JavaScript 模块文件编译为 C/C++代码，此时是否直接将其编译为可执行代码了呢？其实不是。

#### 1. 转存为 C/C++代码

Node 采用了 V8 附带的 js2c.py 工具，将所有的内置的 JavaScript 代码（`src/node.js`和`lib/*.js`）转换称 C++里的数组，生成`node_natives.h`头文件，相关代码如下：

```c++
namespace code {
  const char node_native[] = {47, 47, ..};
  const char dgram_native[] = {47, 47, ..};
  const char console_native[] = {47, 47, ..};
  const char buffer_native[] = {47, 47, ..};
  const char querystring_native[] = {47, 47, ..};
  const char punycode_native[] = {47, 42, ..};
  ...
  struct _native {
    const char* name;
    const char* source;
    size_t source_len;
  };

  static const struct _native natives[] = {
    {"node", node_native, sizeof(node_native) - 1},
    {"node", dgram_native, sizeof(dgram_native) - 1},
    ...
  };
}
```

在这个过程中，JavaScript 代码以字符串的形式存储在 node 命名空间中，是不可直接执行的。在启动 Node 进程时，JavaScript 代码直接加载进内存中，在加载过程中，JavaScript 核心模块经历标识符分析后直接定位到内存中，比普通的文件模块从磁盘中一处一处查找要快得多。

#### 2. 编译 JavaScript 核心模块

`lib`目录下的所有模块文件也没有定义`require`、`module`、`exports`这些变量。在引入 JavaScript 核心模块的过程中，也经历了头尾包装的过程，然后才执行和导出了`exports`对象。与文件模块的区别在于：获取源代码的方式（核心模块是从内存中加载的）以及缓存执行结果的位置。

JavaScript 核心模块的定义如下面代码所示，源文件通过`process.binding('natives')`取出，编译成功的模块缓存到`NativeModule._cache`对象上，文件模块则缓存到`Module._cache`对象上：

```javascript
function NativeModule(id) {
  this.filename = id + '.js';
  this.id = id;
  this.exports = {};
  this.loaded = false;
}
NativeModule._source = process.binding('natives');
NativeModule._cache = {};
```

### 2.3.2 C/C++核心模块的编译过程

在核心模块中，有些模块全部由 C/C++编写，有些模块则由 C/C++完成核心部分，其它部分则由 JavaScript 实现包装或向外导出，以满足性能需求。后面这种 C++模块主要完成核心，JavaScript 主外实现封装的模式时 Node 能够提高性能的常见方式。通常，脚本语言的开发速度优于静态语言，但是其性能则弱于静态语言。而 Node 的这种符合模式可以在开发速度和性能之间找到平衡点。

这里我们将那些由纯 C/C++编写的部分统一称为内建模块，因为它们通常不被用户直接调用。Node 的 buffer、crypto、evals、fs、os 等模块都是部分通过 C/C++编写的。

#### 1. 内建模块的组织形式

在 Node 中，内建模块的内部结构定义如下：

```c++
struct node_module_struct {
  int version;
  void *dso_handle;
  const (*register_func) (v8::Handle<v8::Object> target);
  const char *modname;
}
```

每一个内建模块在定义之后，都通过`NODE_MODULE`宏将模块定义到 node 命名空间中，模块的具体初始化方法挂载为结构的`register_func`成员：

```c++
#define NODE_MODULE(modname, regfunc)                               \
  extern "C"{                                                     \
    NODE_MODULE_EXPORT node::node_module_struct modname ## _module =   \
    {                                                             \
      NODE_STANDARD_MODULE_STUFF,                                   \
      regfunc,                                                    \
      NODE_STRINGIFY(modname)                                      \
    };
  }
```

`node_extensions.h`文件将这些散列的内建模块统一放进了一个叫`node_module_list`的数组中，这些模块有：

- `node_buffer`
- `node_crypto`
- `node_evals`
- `node_fs`
- `node_http_parser`
- `node_os`
- `node_zlib`
- `node_timer_wrap`
- `node_tcp_wrap`
- `node_udp_wrap`
- `node_pipe_wrap`
- `node_cares_wrap`
- `node_tty_wrap`
- `node_process_wrap`
- `node_fs_event_wrap`
- `node_signal_watcher`

这些内建模块的取出也十分简单。Node 提供了`get_builtin_module()`方法从`node_module_list`数组中取出这些模块。

内建模块的优势在于：首先，它们本身由 C/C++编写，性能上优于脚本语言；其次，在进行文件编译时，它们被编译进二进制文件。一旦 Node 开始执行，它们被直接加载进内存中，无需再次做标识符定位、文件定位、编译等过程，直接就可执行。

#### 2. 内建模块的导出

在 Node 的所有模块类型中，存在着一种依赖层级关系，即文件模块可能会依赖核心模块，核心模块可能会依赖内建模块。

通常，不推荐文件模块直接调用内建模块。如需调用，直接调用核心模块即可，因为核心模块中基本都封装了内建模块。那么内建模块时如何将内部变量或方法导出，以供外部 JavaScript 核心模块调用的呢？

Node 在启动的时候，会生成一个全局变量`process`，并提供`Binding()`方法来协助加载内建模块。`Binding()`的实现代码在`src/node.cc`中，具体如下所示：

```c++
static Handle<Value> Binding(const Arguments& args) {
  HandleScope scope;

  Local<String> module = args[0]->ToString();
  String::Utf8Value module_v(module);
  node_module_struct* modp;

  if(binding_cache.IsEmpty()) {
    binding_cache = Persistent<Object>::New(Object::New());
  }

  Local<Object> exports;

  if(binding_cache->Has(module)) {
    exports = binding_cache->Get(module)->ToObject();
    return scope.Close(exports);
  }

  // Append a string to process.moduleLoadList
  char buf[1024];
  snprintf(buf, 1024, "Binding %s", *module_v);
  uint32_t l = module_load_list->Length();
  module_load_list->Set(l, String::New(buf));

  if((modp = get_bultin_module(*module_v)) != NULL) {
    exports = Object::New();
    modp->register_func(exports);
    binding_cache->Set(module, exports);
  }else if(!strcmp(*module_v, "constants")) {
    exports = Object::New();
    DefineConstants(exports);
    binding_cache->Set(module, exports);

  #ifdef __POSIX__
  } else if(!strcmp(*module_v, "io_Watcher")) {
    exports = Object::New();
    IOWatcher::Initialize(exports);
    binding_cache->Set(module, exports);
  #endif
  } else if(!strcmp(*module_v, "natives")) {
    exports = Object::New();
    DefineJavaScript(exports);
    binding_cache->Set(module, exports);
  } else {
    return ThrowException(Exception::Error(String::New("No such module")));
  }
  return scope.Close(exports);
}
```

在加载内建模块时，我们先创建一个`exports`空对象，然后调用`get_builtin_module()`方法取出内建模块对象，通过执行`register_func()`填充`exports`对象，最后将`exports`对象按模块名缓存，并返回给调用方完成导出。

这个方法不仅可以导出内建方法，还能导出一些别的内容。前面提到的 JavaScript 核心文件被转换成 C/C++数组存储后，便是通过`process.binding('natives')`取出放置在`NativeModule._source`中的：

```javascript
NativeModule._source = process.binding('natives');
```

该方法将通过`js2c.py`工具转换出的字符串数组取出，然后重新转换为普通字符串，以对 JavaScript 核心模块进行编译和执行。

### 2.3.3 核心模块的引入流程

前面讲述了核心模块的原理，也解释了核心模块的引入速度为何是最快的。

为了符合 CommonJS 模块规范，从 JavaScript 到 C/C++的过程是相当复杂的，它要经历 C/C++层面的内建模块定义、（JavaScript）核心模块的定义和引入以及（JavaScript）文件模块层面的引入。但是对于用户而言，`require()`十分简洁、友好。

### 2.3.4 编写核心模块

核心模块被编译二进制文件需要遵循一定规则。作为 Node 的使用者，尽管几乎没有机会参与核心模块的开发，但是了解如何开发核心模块有助于我们更加深入地了解 Node。

核心模块中地 JavaScript 部分几乎与文件模块地开发相同，遵循 CommonJS 模块规范，上下文除了拥有`require`、`module、`exports`外，还可以调用 Node 中的一些全局变量，这里不做描述。

下面我们以 C/C++模块为例演示如何编写内建模块。为了便于理解，我们先编写一个极其简单的 JavaScript 版本的原型，这个方法返回一个 Hello Wolrd!字符串：

```javascript
exports.sayHello = function () {
  return 'Hello World!';
};
```

编写内建模块通常分两步完成：编写头文件和编写 C/C++文件。

（1）将以下代码保存为`node_hello.h`，存放在 Node 的`src`目录下：

```C++
#ifndef NODE_HELLO_H_
#define NODE_HELLO_H_
#include <v8.h>

namespace node {
  // 预定义方法
  v8::Handle<v8::Value> SayHello(const v8::Arguments& args);
}
#endif
```

（2）编写`node_hello.cc`,并存储到`src`目录下：

```C++
#include <node.h>
#include <node_hello.h>
#include <v8.h>

namespace node {
  using namespace v8;
  // 实现预定义的方法
  Handle<Value> SayHello(const Arguments& args) {
    HandleScope scope;
    return scope.Close(String::New("Hello World!"));
  }

  // 给传入的目标对象添加sayHello方法
  void Init_Hello(Handle<Object> target) {
    target->Set(String::NewSymbol("sayHello"), FunctionTemplate::New(SayHello)->GetFunction());
  }
}

// 调用NODE_MODULE()将注册方法定义到内存中
NODE_MODULE(node_hello,node::Init_Hello);
```

以上两步完成了内建模块的编写，但是真正要让 Node 认为它是内建模块，还需要更改`src/node_extensions.h`，在`NODE_EXT_LIST_END` 前添加`NODE_EXT_LIST_ITEM(node_hello)`，以将`node_hello`模块添加进`node_module_list`数组中。

其次，还需要让编写的两份代码编译进执行文件，同时需要更改 Node 的项目生成文件`node.gyp`，并在`'target_name': 'node'`节点的`sources`中添加新编写的两个文件。然后编译整个 Node 项目，具体的编译步骤请参见附录 A。

编译和安装后，直接在命令行中运行以下代码，将会得到期望的效果：

```javascript
var hello = process.binding('hello');
hello.sayHello(); // => Hello Wolrd!
```

至此，原生编写过程中需要注意的细节都已表述过了。可以看出，简单的模块通过 JavaScript 来编写可以大大提高生产效率。这里我们写作本届的目的是希望有能力的读者可以深入 Node 的核心模块，去学习它或者改进它。

## 2.4 C/C++扩展模块

对于前端工程师来说，C/C++扩展模块或许比较生疏和晦涩，但是如果你了解了它，在模块出现性能瓶颈时将会对你有极大的帮助。

JavaScript 的典型弱点就是位运算。JavaScript 的位运算符参照 Java 的位运算符实现，但是 Java 位运算是在 int 型数字的基础上进行的，而 JavaScript 只有 double 的数据类型，在进行位运算的过程中，需要将 double 型转换为 int 型，然后再进行。所以，在 JavaScript 层面做位运算的效率不高。

在应用中，会频繁出现位运算的需求，包括转码、编码等过程，如果通过 JavaScript 来实现，CPU 资源将会耗费很多，这时编写 C/C++扩展模块提升性能的机会就来了。

C/C++扩展模块属于文件模块中的一类。前面讲述文件模块的编译部分时提到，C/C++模块通过预先编译为`.node`文件，然后调用`process.dlopen()`方法加载执行。在这一节中，我们将分析整个 C/C++扩展模块的编写、编译、加载、导出的过程。

在开始编写扩展模块前，需要强调一点的是，Node 的原生模块一定程度上都可以跨平台的，其前提条件是源代码可以支持在\*nix 和 Windows 上编译，其中\*nix 下通过`g++/gcc`等编译器编译为动态链接共享对象文件（.so），在 Windows 下则需要通过 Visual C++的编译器编译为动态链接库文件（.dll）。这里有一个让人迷惑的地方，那就是引用加载时确实`.node`文件。其实`.node`的扩展名只是为了看起来更自然一点，不会因为平台差异而产生不同的感觉。实际上，在 Windows 下它是一个`.dll`文件，而在\*nix 下则是一个`.so`文件。为了实现跨平台，`dlopen()`方法在内部实现时区分了平台，分别用的时加载`.so`和`.dll`的方式。

值得注意的时，一个平台下的`.node`文件在另一个平台下时无法加载执行的，必须重新各自平台下的编译器编译为正确的`.node`文件。

### 2.4.1 前提条件

如果想要编写高质量的 C/C++扩展模块，还需要深厚的 C/C++编程功底才行。除此之外，以下这些条目都是不能避开的，在了解它们之后，可以让你在编写过程中事半功倍。

- **GYP 项目生成工具**。GYP 工具，即“Generate Your Projects”短句的缩写。它的好处在于，可以帮助你生成各个平台下的项目文件，比如 Windows 下的 Visual Studio 解决方案文件（.sln）、Mac 下的 XCode 项目配置文件以及 Scons 工具。在这个基础上，再动用各自平台下的编译器编译项目。这大大减少了跨平台模块在项目组织上的精力投入。

  Node 源码中一度出现过各种项目文件，后来均统一为 GYP 工具。这除了可以减少编写跨平台项目文件的工作量外，另一个简单的原因就是 Node 自身的源码就是通过 GYP 编译的。为此，Nathan Rajlich 基于 GYP 为 Node 提供了一个专有的扩展构建工具`node_gyp`，这个工具通过`npm install -g node-gyp`命令即可安装。

- **V8 引擎 C++库**。V8 是 Node 自身的动力来源之一。它自身由 C++携程，可以实现 JavaScript 与 C++的相互调用。

- **libuv 库**。它是 Node 自身动力来源之二。Node 能够实现跨平台的诀窍就是它的 libuv 库，这个库是跨平台的一层封装，通过它去调用一些底层操作，比自己在各个平台下编写实现要高效得多。libuv 封装得功能包括事件循环、文件操作等。

- **Node 内部库**。在写 C++模块时，免不了要做一些面向对象的编程工作，而 Node 自身提供了一些 C++代码，比如：`node::ObjectWrap`类可以用来包装你自定义类，它可以帮助实现对象回收工作。

- **其它库**。其它存在 deps 目录下的库，在编写扩展模块时也许可以帮助你，比如`zlib`、`openssl`、`http_parser`等。

### 2.4.2 C/C++扩展模块的编写

在介绍 C/C++内建模块时，其实已经介绍了 C/C++模块的编写方式。普通的扩展模块与内建模块的区别在于无须将源代码编译进 Node，而是通过`dlopen()`方法动态加载。所以在编写普通扩展模块时，无需将源代码写进 node 命名空间，也不需要提供头文件。下面我们将采用同一例子来介绍 C/C++扩展模块的编写。

它的 JavaScript 原型代码与前面的例子一样：

```javascript
exports.sayHello = function () {
  return 'Hello World!';
};
```

新建 hello 目录作为自己的项目位置，编写`hello.cc`并将其存储到`src`目录下，相关代码如下：

```c++
#include <node.h>
#include <v8.h>

using namespace vu;
// 实现预定义方法
Handle<Value> SayHello(const Arguments& args) {
  HandleScope scope;
  return scope.Colose(String::New("Hello World!"));
}
// 给传入的目标对象添加sayHello()方法
void Init_Hello(Handle<Object> target) {
  target->Set(String::NewSymbol("sayHello"), FunctionTemplate::New(SayHello)->GetFunction());
}
// 调用NODE_MODULE()方法将注册方法定义到内存中
NODE_MODULE(hello,Init_Hello);
```

C/C++扩展模块与内建模块的套路一样，将方法挂载在`target`对象上，然后通过`NODE_MODULE`声明即可。

由于不像编写内建模块那样将对象声明到`node_module_list`链表中，所以无法被认作时一个原生模块，只能通过`dlopen()`来动态加载，然后导出给 JavaScript 调用。

### 2.4.3 C/C++扩展模块的编译

在 GYP 工具的帮助下，C/C++扩展模块的编译是一件省心的事情，无需为每个平台编写不同的项目编译文件。写好`.gyp`项目文件是除编码外的头等大事，然而你也无需担心此事太难，因为`.gyp`项目文件是足够简单的。`node-gyp`约定`.gyp`文件为`binding.gyp`，其内容如下：

```javascript
{
  'targets': [
    {
      'target_name': 'hello',
      'sources': [
        'hello.cc'
      ],
      'conditions': [
        [
          'OS == "win"',
          {
            'libraries': ['-lnode.lib']
          }
        ]
      ]
    }
  ]
}
```

然后调用：

```sh
node-gyp configure
```

`node-gyp configure`这个命令会在当前目录中创建`build`目录，并生成系统相关的项目文件。在\*nix 下，`build`目录会出现`Makefile`等文件；在 Windows 下，则会生成`vcxproj`等文件。

继续执行如下代码：

```sh
node-gyp build
```

编译过程会根据平台不同，分别通过`make`或`vcbuild`进行编译。编译完成后，`hello.node`文件会生成在`build/Release`目录下。

### 2.4.4 C/C++扩展模块的加载

得到 hello.node 结果文件后，如果调用扩展模块其实在前面已经提及。`require()`方法通过解析标识符、路径分析、文件定位，然后加载执行即可。下面代码引入前面编译得到的`.node`文件，并调用执行其中方法：

```javascript
var hello = require(./build/Release/hello.node);
console.log(hello.sayHello());
```

以上代码存为`hello.js`，调用`node hello.js`命令即可。

对于以`.node`为扩展名的文件，Node 将会调用`process.dlopen()`方法去加载文件：

```javascript
Module._extensions['.node'] = process.dlopen;
```

对于调用者而言，`require()`是轻松愉快的。对于扩展模块的编写者而言，`process.dlopen()`中隐含的过程值得了解一番。

`require()`在引入`.node`文件的过程中，实际经历了 4 个层面上的调用。

加载`.node`文件实际经历了两个步骤，第一个步骤是调用`uv_dlopen()`方法去打开动态链接库，第二个步骤是调用`uv_dlsym()`方法去找到动态链接库中`NODE_MODULE`宏定义的方法地址。这两个过程都是通过 libuv 库进行封装的：在\*nix 平台下实际上调用的是`dlfcn.h`头文件中定义的`dlopen()`和`dlsym()`两个方法；在 Windows 平台则是通过`LoadLibraryExW()`和`GetProcAddress()`这两个方法实现的，它们分别加载`.so`和`.dll`（实际为.node 文件）。

这里对 libuv 函数的调用充分表现 Node 利用 libuv 实现跨平台的方式，这样的情景在很多地方还会出现。

由于编写模块时通过`NODE_MODULE`将模块定义为`node_modul_struct`结构，所以在获取函数地址之后，将它映射为`node_module_struct`结构几乎是无缝对接的。接下来的过程就是将传入的`exports`对象作为实参运行，将 C++中定义的方法挂载在 exports 对象上，然后调用者就可以轻松调用了。

C/C++扩展模块与 JavaScript 模块的区别在于加载后就不需要编译，直接执行之后就可以被外部调用了，其加载速度比 JavaScript 模块略快。

使用 C/C++扩展模块的一个好处在于可以更灵活和动态的加载它们，保持 Node 模块自身简单性的同时，给予 Node 无限的可扩展性。

关于`node-gyp`工具更多细节可以参考 [gitbub 仓库](https://github.com/TooTallNate/node-gyp) （作者为 Nathan Rajlich，Node 源码的核心贡献者之一）。

## 2.5 模块调用栈

下面我们来明确下模块之间的调用关系。

C/C++内建模块属于最底层的模块，它属于核心模块，主要提供 API 给 JavaScript 核心模块和第三方 JavaScript 文件模块调用。如果你不是非常了解调用的 C/C++内建模块，请尽量避免通过`process.binding()`方法直接调用，这时不推荐的。

JavaScript 核心模块主要扮演职责由两类：一类是作为 C/C++内建模块的封装层和桥阶层，供文件模块调用；一类是存粹的功能模块，它不需要跟底层打交道，但是又十分重要。

文件模块通常由第三方编写，包括普通 JavaScript 模块和 C/C++扩展模块，主要调用方向为普通 JavaScript 模块调用扩展模块。

## 2.6 包和 NPM

Node 组织了自身的核心模块，也使得第三方文件模块可以有序地编写和使用。但是在第三方模块中，模块与模块之间仍然是散列在各地的，相互之间不能直接引用。而在模块之外，包和 NPM 则是将模块联系起来的一种机制。

在介绍 NPM 之前，不得不提起 CommonJS 的包规范。JavaScript 不似 Java 或者其它语言那样，具有模块和包结构。Node 对模块规范的实现，一定程度上解决了变量依赖、依赖关系等代码组织性问题。包的出现，则是在模块的基础上进一步组织 JavaScript 代码。

CommonJS 的包规范的定义其实也十分简单，它由包结构和包描述文件两个部分组成，前者用于组织包中的各种文件，后者则用于描述包的相关信息，以供外部读取分析。

### 2.6.1 包结构

包实际上是一个存档文件，即一个目录直接打包为`.zip`或`.tar.gz`格式的文件，安装后解压还原目录。完全符合 CommonJS 规范的包目录应该包含如下文件：

- package.json：包描述文件。
- bin：用于存放可执行二进制文件的目录。
- lib：用于存放 JavaScript 代码的目录。
- doc：用于存放文档的目录。
- test：用于存放单元测试用力的代码。

可以看出，CommonJS 包规范从文档、测试等方面都做过考虑。当一个包完成后向外公布时，用户看到单元测试和文档的时候，会给他们一种踏实可靠的感觉。

### 2.6.2 包描述文件和 NPM

包描述文件用于表达非代码相关的信息，它是一个 JSON 格式的文件——pacakge.json，位于包的根目录下，是包的重要组成部分。而 NPM 得所有行为斗鱼包描述文件中得字段息息相关。由于 CommonJS 包规范尚处于草案阶段，NPM 在实践中做了一定得取舍，具体细节在和后面会介绍到。

CommonJS 为 package.json 文件定义了如下一些必须字段。

- **name**。包名。规范定义它需要由小写字母和数字组成，可以包含`.`、`_`和`-`，但不允许出现空格。包名必须是唯一得，以免对外公布时产生重名冲突得误解。除此之外，NPM 还建议不要在包名中附带上`node`或`js`来重复标识它是 JavaScript 或 Node 模块。
- **description**。包简介。
- **version**。版本号。一个语义化的版本号，在[http://semver.org](http://semver.org)上有详细定义，通常为 major.minor.revision 格式。该版本号十分重要，常常用于一些版本控制的场合。
- **keywords**。关键词数组，NPM 中主要用来做分类搜索。一个好的关键词组有利于用户快速找到你编写的包。
- **maintainers**。包维护者列表。每个维护者由 name、email 和 web 这三个属性组成。示例`"maintainers": [{"name": "Jackson Tian", "email": "shyvo1987@gmail.com", "web": "http://html5ify.com"}]`。NPM 通过改属性进行权限认证。
- **contributors**。贡献者列表。在开源社区中，为开源项目提供代码是经常出现的事，如果名字能出现在知名项目的`contributors`列表中，是一件比较由荣誉感的事。列表中第一个贡献应当是包作者本人。格式与维护者列表相同。
- **bugs**。一个可以反馈 bug 的网页或者邮箱地址。
- **licenses**。当钱包所使用的许可证列表，表示这个包可以在哪些许可证下使用。它的格式：`"licenses": [{"type": "GPLv2", "url": "http://www.example.com/licenses/gpl.html"}]`。
- **repositories**。托管源代码的位置列表，表明可以通过哪些方式和地址访问包的源代码。
- **dependencies**。使用当前包所需要依赖的包列表。这个属性十分重要，NPM 需要通过这个属性帮助自动加载依赖的包。

除了必要字段外，规范还定义了一部分可选字段，如：

- **homepage**。当前包的网站地址。

- **os**。操作系统支持列表。这些操作系统的取值包括`aix`、`freebsd`、`linux`、`macos`、`solaris`、`vxworks`、`windows`。如果设置了列表为空，则不对操作系统做任何假设。

- **cpu**。CPU 架构的支持列表，有效的架构名称有`arm`、`mips`、`ppc`、`sparc`、`x86`和`x86_64`。通`os`一样，如果列表为空，则不对 CPU 架构做任何假设。

- **engine**。支持的 JavaScript 引擎列表，有效的引擎取值包括`ejs`、`flusspferd`、`gpsee`、`jsc`、`spidermonkey`、`narwhal`、`node`、`v8`。

- **builtin**。标识当前包是否是内建在底层系统的标准组件。

- **directories**。包目录说明。

- **implements**。实现规范的列表。标志当前包实现了 CommonJS 的哪些规范。

- **script**。脚本说明对象。它主要被包管理器用来安装、编译、测试和卸载包。示例：

  ```json
  "scripts":{
    "install": "install.js",
    "unistall": "uninstall.js",
    "build": "build.js",
    "doc": "make-doc.js",
    "test": "test.js"
  }
  ```

包规范的定义可以帮助 Node 解决依赖包安装的问题，而 NPM 正式基于该规范进行了实现。最初，NPM 工具是由 ISaac Z. Schlueter 单独创建，提供给 Node 服务的 Node 包管理器，需要单独安装。后来，在 v0.6.3 版本集成进 Node 中作为默认的包管理器，作为软件包的一部分一起安装。

在包描述文件的规范中，NPM 实际需要的字段主要有`name`、`version`、`description`、`keywords`、`repositories`、`author`、`bin`、`main`、`scripts`、`engines`、`dependencies`、`devDependencies`。

与包规范区别在于多了`author`、`bin`、`main`和`devDependencies`4 个字段：

- **author**。包作者。
- **bin**。一些包作者希望包可以作为命令行工具使用。配置好`bin`字段后，通过`npm install package_name -g`命令可以将脚本添加到执行路径中，之后可以在命令行中直接执行。前面的`node-gyp`即是这样安装的。通过`-g`命令安装的模块包称为全局模式。
- **main**。模块引入方法`require()`在引入包时，会优先检查该字段，并将其作为包 中其余模块的入口。如果不存在这个字段，`require()`方法会查找包目录下的`index.js`、`index.node`、`index.json`文件作为默认入口。
- **devDependencies**。一些模块只在开发时需要依赖。配置这个属性，可以提示包的后续开发者安装依赖包。

下面是知名框架`express`项目的`package.json`文件，具有一定的参考意义：

```json

  "name": "express",
  "description": "Fast, unopinionated, minimalist web framework",
  "version": "4.17.1",
  "author": "TJ Holowaychuk <tj@vision-media.ca>",
  "contributors": [
    "Aaron Heckmann <aaron.heckmann+github@gmail.com>",
    "Ciaran Jessup <ciaranj@gmail.com>",
    "Douglas Christopher Wilson <doug@somethingdoug.com>",
    "Guillermo Rauch <rauchg@gmail.com>",
    "Jonathan Ong <me@jongleberry.com>",
    "Roman Shtylman <shtylman+expressjs@gmail.com>",
    "Young Jae Sim <hanul@hanul.me>"
  ],
  "license": "MIT",
  "repository": "expressjs/express",
  "homepage": "http://expressjs.com/",
  "keywords": [
    "express",
    "framework",
    "sinatra",
    "web",
    "http",
    "rest",
    "restful",
    "router",
    "app",
    "api"
  ],
  "dependencies": {
    "accepts": "~1.3.7",
    "array-flatten": "1.1.1",
    "body-parser": "1.19.0",
    "content-disposition": "0.5.3",
    "content-type": "~1.0.4",
    "cookie": "0.4.0",
    "cookie-signature": "1.0.6",
    "debug": "2.6.9",
    "depd": "~1.1.2",
    "encodeurl": "~1.0.2",
    "escape-html": "~1.0.3",
    "etag": "~1.8.1",
    "finalhandler": "~1.1.2",
    "fresh": "0.5.2",
    "merge-descriptors": "1.0.1",
    "methods": "~1.1.2",
    "on-finished": "~2.3.0",
    "parseurl": "~1.3.3",
    "path-to-regexp": "0.1.7",
    "proxy-addr": "~2.0.5",
    "qs": "6.7.0",
    "range-parser": "~1.2.1",
    "safe-buffer": "5.1.2",
    "send": "0.17.1",
    "serve-static": "1.14.1",
    "setprototypeof": "1.1.1",
    "statuses": "~1.5.0",
    "type-is": "~1.6.18",
    "utils-merge": "1.0.1",
    "vary": "~1.1.2"
  },
  "devDependencies": {
    "after": "0.8.2",
    "connect-redis": "3.4.2",
    "cookie-parser": "~1.4.4",
    "cookie-session": "1.3.3",
    "ejs": "2.7.2",
    "eslint": "2.13.1",
    "express-session": "1.17.0",
    "hbs": "4.1.0",
    "istanbul": "0.4.5",
    "marked": "0.7.0",
    "method-override": "3.0.0",
    "mocha": "7.0.1",
    "morgan": "1.9.1",
    "multiparty": "4.2.1",
    "pbkdf2-password": "1.2.1",
    "should": "13.2.3",
    "supertest": "4.0.2",
    "vhost": "~3.0.2"
  },
  "engines": {
    "node": ">= 0.10.0"
  },
  "files": [
    "LICENSE",
    "History.md",
    "Readme.md",
    "index.js",
    "lib/"
  ],
  "scripts": {
    "lint": "eslint .",
    "test": "mocha --require test/support/env --reporter spec --bail --check-leaks test/ test/acceptance/",
    "test-ci": "istanbul cover node_modules/mocha/bin/_mocha --report lcovonly -- --require test/support/env --reporter spec --check-leaks test/ test/acceptance/",
    "test-cov": "istanbul cover node_modules/mocha/bin/_mocha -- --require test/support/env --reporter dot --check-leaks test/ test/acceptance/",
    "test-tap": "mocha --require test/support/env --reporter tap --check-leaks test/ test/acceptance/"
  }
}
```

### 2.6.3 NPM 常用功能

CommonJS 包规范是理论，NPM 是其中的一种实践。NPM 之于 Node，相当于 gem 之于 Ruby，pear 之于 PHP。对于 Node 而言，NPM 帮助完成了第三方模块的发布、安装和依赖等。借助 NPM，Node 与第三方模块之间形成了很好的生态系统。

借助 NPM，可以帮助用户快速安装和管理依赖包。除此之外，NPM 还有一些巧妙地用法：

#### 1.查看帮助

- 在安装 Node 之后，执行`npm -v`命令可以查看当前 NPM 地版本。

- 在不熟悉 NPM 命令之前，可以直接执行`npm`查看帮助引导说明。
- `npm help <command>`可以查看具体的命令说明。

#### 2. 安装依赖包

安装依赖包是 NPM 最常见的用法，它的执行语句是`npm install express`。执行该命令后，NPM 会在当前目录下创建`node_modules`目录，然后在`node_modules`目录下创建`express`目录，接着将包解压到这个目录下。

安装好依赖包后，直接在代码中调用`require('express');`即可引入该包。`require()`方法在做路径分析的时候会通过模块路径查找到`express`所在的位置。模块引入和包的安装这两个步骤是相辅相成的。

- **全局模式安装**

  全局模式这个称谓其实并不精确，存在诸多误导。实际上，`-g`是将一个包安装为全局可用的可执行命令。它根据包描述文件中的`bin`字段配置，将实际脚本连接到 Node 可执行文件相同的路径下：

  ```json
  "bin": {
    "express": "./bin/express"
  }
  ```

  事实上，通过全局模式安装的所有模块包都被安装进了一个统一的目录下，这个目录可以通过如下方式推算出来：

  ```javascript
  path.resolve(process.execPath, '..', '..', 'lib', 'node_modules');
  ```

  如果 Node 可执行文件的位置是`/usr/local/bin/node`，那么模块目录就是`/usr/local/lib/node_modules`。最后，通过软连接的方式将`bin`字段配置的可执行文件链接到 Node 的可执行目录下。

- **从本地安装**

  对于一些没有发布到 NPM 上的包，或是因为网络原因导致无法直接安装的包，可以通过将包下载到本地，然后进行本地安装。本地安装只需要为 NPM 指明`package.json`文件所在的位置即可：它可以是一个包含`package.json`的存档文件，也可以是一个 URL 地址，也可以是一个目录下有`package.json`文件的目录位置。具体参数：

  ```sh
  npm install <tarball file>
  npm install <tarball url>
  npm install <folder>
  ```

- **从非官方源安装**

  如果不能通过官方源安装，可以通过镜像源安装。在执行命令时，添加`--registry=http://registry.url`即可，示例如下：

  ```sh
  npm install underscore --registry=http://registry.url
  ```

  如果使用过程中几乎都采用镜像源安装，可以执行以下命令指定默认安装源：

  ```sh
  npm config set registry http://registry.url
  ```

#### 3. NPM 钩子命令

另一个需要说明的是 C/C++模块实际上是编译后才能使用的。`package.json`中的`scripts`字段的提出就是让包在安装或卸载等过程中提供钩子机制，示例如下：

```json
"scripts": {
  "preinstall": "preinstall.js",
  "install": "install.js",
  "uninstall": "uninstall.js",
  "test": "test.js"
}
```

在以上字段中执行`npm install <package>`时，`preinstall`指向的脚本会被加载执行，然后`install`指向的脚本会被执行。在执行`npm uninstall <package>`时，`uninstall`指向的脚本也许会做一些清理工作等。

当在一个具体的包目录下执行`npm test`时，将会运行`test`指向的脚本。一个优秀的包应当包含测试用例，并在`package.json`文件中配置好运行测试的命令，方便用户运行测试用例，以便检验包是否稳定可靠。

#### 4. 发布包

为了将整个 NPM 的流程串联起来，这里演示如何编写一个包，将其发布到 NPM 仓库中，并通过 NPM 安装回本地。

- **编写模块**

  模块的内容尽量保持简单，这里还是使用 sayHello 作为例子：

  ```javascript
  exports.sayHello = function () {
    return 'Hello World!';
  };
  ```

  将代码保存为`hello.js`即可。

- **初始化包描述文件**

  `package.json`文件的内容尽管相对较多，但是实际发布一个包时，并不需要一行一行编写。NPM 提供的`npm init`命令会帮助你生成`package.json`文件

  NPM 通过提问式的交互逐个填入选项，最后生成预览的包描述文件。如果你满意，输入`yes`,此时会在目录下得到`package.json`文件。

- **注册包仓库账号**

  为了维护这个包，NPM 必须要使用仓库账号才允许将包发布到仓库中。注册账号的命令是`npm adduser`。这也是一个提问式的交互过程，按照顺序进行即可

- **上传包**

  上传包的命令是`npm publish <folder>`。在刚刚创建的`package.json`文件所在的目录下，执行`npm publish`开始上传包。

  在这个过程中，NPM 会将目录打包为一个存档文件，然后上传到官方源仓库中。

- **安装包**

  为了体验和测试自己上传的包，可以换一个目录执行`npm install hello_test_jackson`安装它

- **包权限管理**

  通常一个包只有一个人拥有权限进行发布。如果需要多人进行发布，可以使用`npm owner`命令帮助你管理所有者

  ```sh
  npm owner ls express
  ```

  使用这个命令，也可以添加包的拥有者，删除一个包的拥有者：

  ```sh
  npm owner ls <package name>
  npm owner add <user> <package name>
  npm owner rm <user> <package name>
  ```

#### 5. 分析包

在使用 NPM 的过程中，或许你不能确认当前目录下能否通过`require()`顺利引入想要的包，这时可以执行`npm ls`分析包。这个命令可以为你分析出当前路径下能够通过模块路径找到的所有包，并生成依赖树，如下：

```sh
`-- connect@3.7.0
  +-- debug@2.6.9
  | `-- ms@2.0.0
  +-- finalhandler@1.1.2
  | +-- debug@2.6.9 deduped
  | +-- encodeurl@1.0.2
  | +-- escape-html@1.0.3
  | +-- on-finished@2.3.0
  | | `-- ee-first@1.1.1
  | +-- parseurl@1.3.3 deduped
  | +-- statuses@1.5.0
  | `-- unpipe@1.0.0
  +-- parseurl@1.3.3
  `-- utils-merge@1.0.1
```

### 2.6.4 局域 NPM

在企业的内部应用中使用 NPM 与开源社区中使用有一定的差别。企业的限制在于，一方面需要享受模块开发带来的低耦合和项目组织上的好处，另一方面却要考虑模块保密性的问题。所以，通过 NPM 共享和发布存在潜在风险。

为了同时能享受 NPM 上众多的包，同时对自己的包进行保密和限制，现有的解决方案就是企业自己搭建自己的 NPM 仓库。

所幸，NPM 自身是开源的，无论是它的服务器端和客户端。通过源代码搭建自己的仓库并不是什么秘密。

局域 NPM 仓库的搭建方法与搭建镜像站（详情可参见附录 D）的方式几乎一样。

与镜像仓库不同的地方在于，企业局域 NPM 可以选择不同步官方源仓库中的包。

对于企业内部而言，私有的可重用模块可以打包到局域 NPM 仓库中，这样可以保持更新的中心化，不至于让各个小项目各自维护相同功能的模块，杜绝通过复制粘贴实现代码共享的行为。

### 2.6.5 NPM 潜在问题

作为为模块和包服务的工具，NPM 十分便捷。它实质上已经是一个包共享平台，所有人都可以贡献模块并将其打包分享到这个平台上，也可以在许可证（大多是 MIT 许可证）的允许下免费使用它们。NPM 提供的这些边界，将模块连接到一个共享平台上，缩短了贡献者与使用者之间的距离，着十分有利于模块的传播，进而也十分利于 Node 的推广。几乎没有一种语言或平台有 Node 这样出现三年多就有成千上万个第三方模块的情景。这个功劳一部分是因为 Node 选择了 JavaScript 这门语言，它拥有极大的开发人员基数，具有强大的生产力；另一方面则是因为 CommonJS 规范和 NPM，它们使得产品能够更好地组织、传播和使用。

潜在问题在于，在 NPM 平台上，每个人都可以分享包到平台上，鉴于开发人员水平不一，上面的包的质量良莠不齐。另一个问题则是，Node 代码可以运行在服务器端，需要考虑安全问题。

对于包的使用者而言，包质量和安全问题需要作为是否采纳模块的一个判断条件。在安全问题上，经过模块质量的考查之后，应该可以去掉一大半候选包。基于使用者大多都是 JavaScript 程序员，难点其实存在于第三方 C/C++扩展模块，这类模块建议在企业的安全部门检查后方可允许使用。

## 2.7 前后端共用模块

谈论了许多后端模块的具体失陷后，现在我们围绕 CommonJS 规范再次回到前端模块上。javaScript 在 Node 出现之后，比别的编程语言多了一项优势，那就是一些模块可以在前后端实现共用，这时因为很多 API 在各个宿主环境都提供。但是在实际情况中，前后端的环境是略有差别的。

### 2.7.1 模块的侧重点

前后端 JavaScript 分别搁置在 HTTP 的两端，它们扮演的角色不相同。浏览器端的 JavaScript 需要经历从一个服务器分发到多个客户端执行，而服务器端 JavaScript 则是相同的代码需要多次执行。前者的瓶颈在于带宽，后者的瓶颈则在于 CPU 和内存等资源。前者需要通过网络加载代码，后者从磁盘加载，两者的加载速度不在一个数量级上。

纵观 Node 的模块引入全程，几乎全都是同步的。尽管与 Node 强调异步的行为有些相反，但是它是合理的。如果前端模块也采用同步的方式来引入，那将会在用户体验上造成很大的问题。UI 在初始化过程中需要花费很多时间来等待脚本加载完成。

鉴于网络的原因，CommonJS 为后端 JavaScript 指定的规范并不完全适合前端的应用场景。经过一段争执之后，AMD 规范最终在前端应用场景中胜出它的全称是 Asynchronous Module Definition，即[“异步模块定义”](https://github.com/amdjs/amdjs-api/wiki/AMD)，详见。除此之外，还有玉伯定义的 CMD 规范。

### 2.7.2 AMD 规范

AMD 规范是 CommonJS 模块规范的一个延伸，它的模块定义如下：

```javascript
define(id?, dependencies?, factory);
```

它的模块 id 和依赖是可选的，与 Node 模块相似的地方在于`factory`的内容就是实际代码的内容。下面的代码定义了一个简单的模块：

```javascript
define(function () {
  var exports = {};
  exports.sayHello = function () {
    console.log('Hello from module: ' + module.id);
  };
  return exports;
});
```

不同之处在于 AMD 模块需要用`define`来明确定义一个模块，而在 Node 实现中式隐式包装的，它们的目的就进行作用域隔离，仅在需要的时候被引入，避免过去那种通过全局变量或者全局命名空间的方式，以免变量污染和不小心修改。另一个区别则是内容需要通过返回的方式实现导出。

### 2.7.3 CMD 规范

CMD 规范由国内的玉伯提出，与 AMD 规范的主要区别在于定义模块和依赖引入的部分。AMD 需要在声明模块的时候指定所有的依赖，通过形参传递依赖到模块内容中：

```javascript
define(['dep1', 'dep2'], function (dep1, dep2) {
  return function () {};
});
```

与 AMD 规范相比，CMD 更接近于 Node 对 CommonJS 规范的定义：`define(factory)`

在依赖部分，CMD 支持动态引入：

```javascript
define(function (require, exports, module) {
  // The module code goes here
});
```

`require`、`exports`和`module`通过形参传递给模块，在需要依赖模块时，随时调用`require()`引入即可。

### 2.7.4 兼容多种模块规范

为了让同一个模块可以运行在前后端，在写作过程中需要考虑兼容前端也实现了模块规范的环境。为了保持前后端的一致性，类库开发者需要将类库代码包装在一个闭包内。以下代码演示如何将`hello()`方法定义到不同的运行环境中，它能够兼容 Node、AMD、CMD 以及常见的浏览器环境中：

```javascript
(function (name, definition) {
  // 检测上下文环境是否为 AMD或 CMD
  var hasDefine = typeof define === 'function',
    // 检查上下文环境是否为Node
    hasExports = typeof module !== 'undefined' && module.exports;
  if (hasDefine) {
    // AMD 或 CMD
    define(definition);
  } else if (hasExports) {
    module.exports = definition();
  } else {
    // 将模块的执行结果挂载在Window变量中，浏览器中this指向Window对象
    this[name] = definition();
  }
})('hello', function () {
  var hello = function () {};
  return hello;
});
```

## 2.8 总结

CommonJS 提出的规范均十分简单，但是现实意义十分强大。Node 通过模块规范，组织了自身的原生模块，弥补了 JavaScript 弱结构性的问题，形成了稳定的结构，并向外提供服务。NPM 通过对包规范的支持，有效地组织了第三方模块，这使得项目开发中地依赖问题得到很好的解决，并有效提供了分享和传播的平台，借助第三方开源力量，使得 Node 第三方模块的发展速度前所未有，这对于其他后端 JavaScript 语言实现而言是从未有过的。从一定角度上来讲，CommonJS 规范帮助 Node 形成了它的骨骼。只有茁壮成长的根，才能培养出茂盛的枝叶，并成长为参天大树。正是这些底层的的规范和实践，使得 Node 有序地发展着，摆脱掉过去 JavaScript 纷乱和被误解地局面，进而进化成良性的生态系统。
