# Node 编码规范

---

- [C.1 根源](#c1-根源)
- [C.2 编码规范](#c2-编码规范)
- [C.3 最佳实践](#c3-最佳实践)
- [C.4 总结](#c4-总结)

---

## C.1 根源

JavaScript 作为一门编程语言，在语法上可谓是最为灵活的语言了。有人喜欢它的灵活，也有人讨厌它的混乱。无论它的灵活也好，混乱也罢，都离不开其诞生的历史。Brendan Eich 在 1995 年里花了 10 天设计出了这门语言，其后微软在 1996 年也发布了支持 JavaScript 浏览器 IE 3.0。网景公司为了保护自己，在 1996 年 11 越将 JavaScript 提交给 ECMA 标准化组织，次年 6 月份第一版标准发布，命名为 ECMAScript，编号 262。

早年的 JavaScript 编写十分混乱，它的灵活性和容忍度非常高，使得开发者可以毫无顾忌地编码，最终导致它在一定程度上臭名昭著。在编码规范上，一个重要的任务是 Douglas Crockford，他是 JavaScript 开发 sequence 最知名的权威，是 JSON、JSlint、JSMin 和 ADSafe 之父，其中 JSLint 现在仍然是最重要的 JavaScript 质量检测工具。他出版的 JavaScript：The Good Parts 一书对于 JavaScript 社区影响深远。

通常，一门语言的发展要经历十多年的锤炼才能为大众所接受。由于历史原因，JavaScript 在短时间内就被标准化定性，这样它的优点和缺点都暴露在大众之下。Douglas Crockford 的 JSLint 和 JavaScript：The Good Parts 对于 JavaScript 的贡献在于，他让我们能够甄别语言中的精华和糟粕，写处更好的代码。

与其他语言（比如 Python 和 Ruby）的程序员相比，JavaScript 程序员要更多的自律才能写处易读、易维护的代码。为了避免这个问题，部分开发者选择 TypeScript 或 CoffeeScript 来编写应用。但我认为了解一门语言为何是当下这种情况有必要。编码规范的目的是在一定程度上约束程序员，使之能够在团队中以维护并且避免低级错误。

尽管 JavaScript 规范已经相当成熟，利用 JSLint 能够解决大部分问题，但是随着 Node 的流行，带来了一些新的变化，这些需要引起我们注意。本附录是总结了 JavaScript 的编码规范的基础上，根据 Node 的特殊环境和社区的习惯进行改进而成。

## C.2 编码规范

### C.2.1 空格与格式

#### 1. 缩进

采用 2 个空格缩进，而不是 tab 缩进。空格在编辑器中与字符是等宽的，而 tab 可能因编辑器的设置而不同。2 个空格会让代码看起来更紧凑、明快。

#### 2. 变量声明

永远用 var 声明变量，不加 var 时会将其变成全局变量，这样可能会意外污染上下文，或是被意外污染。在 ECMAScript 5 的 strict 模式下，未声明的变量将会直接抛出 ReferenceError 异常。

需要说明的时，每行声明应该带上 var，而不是只有一个 var，示例：

```js
var assert = require('assert')
var fork = require('child_process').fork
var net = require('net')
var EventEmitter = require('events').EventEmitter
```

错误示例：

```js
var assert = require('assert'),
  fork = require('child_process').fork,
  net = require('net'),
  EventEmitter = require('events').EventEmitter
```

#### 3. 空格

在操作符前后需加空格，比如`+`、`-`、`*`、`%`、`=`等操作符前后都应该存在一个空格，示例：

```js
var foo = 'bar' + baz
```

此外，在小括号前后应该存在空格，如：

```js
if (true) {
  // some code
}
```

#### 4. 单引号的使用

由于双引号在别的场景下使用较多，在 Node 中使用字符串时尽量使用单引号，这样无需转义，如：

```js
var html = '<a href="http://cnodejs.org">CNode</a>'
```

而在 JSON 中，严格的规范时要求字符串用双引号，内容中出现双引号是，需要转义。

#### 5. 大括号的位置

一般情况下，大括号无需另起一行，如：

```js
if (true) {
  // some code
}
```

#### 6. 逗号

逗号用于变量声明的分隔或是元素的分割。如果逗号不在行结尾，前面需要一个空格。此外，逗号不允许出现在行首，比如：`var foo = 'hello', bar = 'world';`或者`var hello = { foo: 'hello', bar: 'world' };`或者`var world = ['hello', 'world'];`。

#### 7. 分号

给表达式结尾添加分号。尽管 JavaScript 编译器会自动给行尾添加分号，但是还是会带来一些误解：

```js
function add() {
  var a = 1,
    b = 2

  return
  a + b
}
```

将会得到 undefined 的返回值，因为 return 后会自动加上分号，后续的 a + b 将不会执行。

### C.2.2 命名规范

在编码过程中，命名是重头戏。好的命名可以令代码赏心悦目，带来愉悦的阅读享受，令代码具有良好的可维护性。命名的主要范畴有变量、常量、方法、类、文件、包等。

#### 1. 变量命名

变量命名都采用小驼峰式命名，即除了第一个单词的首字母不大写外，每个单词的首字母都大写，词与词之间没有任何符号：

```js
// 正确示例
var adminUser = {}

// 错误示例
var admin_user = {}
```

#### 2. 方法命令

方法命名与变量命名一样，采用小驼峰式命名法。与变量不同的是，方法名尽量采用动词或判断性词汇，如：

```js
var getUser = function () {}
var isAdmin = function () {}
User.prototype.getInfo = function () {}
```

#### 3. 类命名

类名采用大驼峰式命名，即所有单词的首字母都大写，如：

```js
function User {
}
```

#### 4. 常量命名

作为常量时，单词的所有字母都大写，并用下划线分割，如：

```js
var PINK_COLOR = 'pink'
```

#### 5. 文件命名

命名文件时，请尽量采用下划线分割单词，比如 child_process.js 和 string_decode.js。如果你不想将文件暴露给其他用户，可以约定以下划线开头，如\_linklist.js。

#### 6. 包名

也许你有贡献模块并将其打包发布到 NPM 上。在包名中，尽量不要包含 js 或 node 的字样，它是重复的。包名应当适当短且有意义的：

```js
var express = require('express')
```

### C.2.3 比较操作

在比较操作中，如果无容忍的场景，请尽量使用`===`代替`==`，否则你会遇到下面这样不符合逻辑的结果：

```js
'0' == 0 // true
'' == 0 // true
'0' === '' // false
```

此外，当判断容忍假值时，可以无需使用`===`或`==`。在下面代码中，当 foo 是 0、undefined、null、false、''时，都会进入分支：

```js
if (!foo) {
  // some code
}
```

### C.2.4 字面量

请尽量使用`{}`、`[]`代替`new Object()`、`new Array()`，不要使用`string`、`bool`、`number`对象类型，即不要调用`new String()`、`new Boolean()`和`new Number()`。

### C.2.5 作用域

在 JavaScript 中，需要注意一个关键字和一个发给发，它们是`with`和`eval()`，容易引起作用域混乱。

#### 1. 慎用 with

示例代码如下：

```js
with (obj) {
  foo = bar
}
```

它的结果有可能是如下四种之一：`obj.foo = obj.bar`、`obj.foo = bar`、`foo = bar`、`foo = obj.bar`，这些结果取决于它的作用域。如果作用域链上没有导致冲突的变量存在，使用它则是安全的。但是多人合作的项目中，这并不能保证，所以慎用`with`。

#### 2. 慎用 eval()

慎用 eval()的原因与 with 相同。如果不影响作用域上已存在的变量，用它则是安全的。另外，利用 eval()的这个特性，也可以玩出一些好玩的特性来，比如`wind.js`利用它实现了流程控制，详见第四章。在大多数情况下，基本上轮不到 eval()来完成特殊使命：

```js
var obj = {
  foo: 'hello',
  bar: 'world',
}

var key = Math.round(Math.random() * 100) % 2 === 0 ? 'foo' : 'bar'
var value = eval('(obj.' + key + ')')
```

上述代码多出现在新手中，实际上只要加上一行代码即可：`var value = obj[key];`。

### C.2.6 数组与对象

在 JavaScript 中，数组其实也是对象，但是两者在使用时有些细节需要注意。

#### 1. 字面量格式

创建对象或者数组时，注意在结尾用光逗号分隔。如果分行，一行只能一个元素，示例代码：

```js
var foo = ['hello', 'world']
var bar = {
  hello: 'world',
  pretty: 'code',
}
```

#### 2. for in 循环

使用`for in`循环实，请对对象使用，不要对数组使用，示例：

```js
var foo = []
foo[100] = 100
for (var i in foo) {
  console.log(i)
}

for (var i = 0; i < foo.length; i++) {
  console.log(i)
}
// 第一个循环只打印以此，而第二个循环则打印0~100
```

#### 3. 不要把数组当成对象使用

尽管在 JavaScript 内部实现中可以把数组当作对象来使用，如下：

```js
var foo = [1, 2, 3]
foo['hello'] = 'world'

// 使用for in 迭代，会得到所有值
for (var i in foo) {
  console.log(foo[i])
}
```

### C.2.7 异步

在 Node 中，异步使用非常广泛并且在实践过程中形成了一些约定，这是以往不曾在意的点。

#### 1. 异步回调函数的第一个参数应该是错误提示

该部分内容在第四章中有所提及。并不是所有回调函数都需要将第一个参数设计为错误对象。但是一旦涉及异步，将会导致`try/catch`无法捕获到异步回调期的异常。将第一个参数设计为错误对象，告知调用方是一个不错的约定：

```js
function (err, data) {
}
```

这个约定被很多流程控制库所采用。遵循这个约定，可以享受社区流程控制库带来的业务编写便利。

#### 2. 执行传入的回调函数

在异步方法中一旦有回调函数传入，就一定要执行它，且不能多次执行。如果不执行，可能造成调用一直等待不结束，多次执行也可能会造成未期望的结果。

### C.2.8 类与模块

关于如何在 JavaScript 中实现集成，有各种各样的方式，但在 Node 中我们只推荐一种，那就是类集成的方式。另外，在 Node 中，如果要将一个类作为一个模块，就需要在意它的导出方式。

#### 1. 类继承

一般情况下，我们采用 Node 推荐的类继承方式，示例：

```js
function Socket(options) {
  // ...
  stream.Stream.call(this)
  // ...
}

util.inherits(Socket, stream.Stream)
```

#### 2. 导出

所有供外部调用的方法或变量均需挂载在 exports 变量上。当需要将文件当作一个类导出时，需要通过如下方式挂载：

```js
module.exports = Class

// 而不是通过
exports = Class
```

私有方法无需因为测试等原因导出给外部，所以无须挂载。

### C.2.9 注解规范

一般情况下，我们会对每个方法编写注释，这里采用 dox 的推荐注释，示例：

```js
/**
 * Queries some records
 * Examples:
 *
 * query('SELECT * FROM table', function (err, data) {
 *   // some code
 * });
 *
 * @param {String} sql Queries
 * @param {Function} callback Callback
 */
exports.query = function (sql, callback) {
  // ...
}
```

dox 的注释规范源自于 JSDoc。可以通过注释生成对应的 API 文档。

## C.3 最佳实践

细致的编码规范有很多，有争议的也不少，但这并不阻碍我们找到共同点。

### C.3.1 冲突的解决原则

如果你要贡献部分代码给某个开源项目，而它的编码规范与你并不相同，这种情况下需要采用入乡随俗的原则，尽量遵循开源项目本身的编码规范而不是自己的编码规范。

### C.3.2 给编辑器设置检测工具

实际上，现在的编辑器基本上都可以通过安装插件的方式将 JSLint 或者 JSHint 这样的代码质量扫描工具集成进开发环境中，这样编码完成后就可以及时得到提示。

如果采用 Sublime Text 编辑器，在安装好插件后，可以在项目中配置`.jshintrc`文件，每次保存都会在编辑器中提醒不规范的信息。

### C.3.3 版本控制中的 hook

另一种最佳实践实在版本控制工具中完成的。无论 SVN 还是 Git，都有 precommit 这样的钩子脚本，通过在提交时实现代码质量的检查。如果质量不达标，将停止提交。

### C.3.4 持续集成

持续集成包含两个方面：一方面仍然是代码质量的扫描，可以选择定时扫描，或是触发式扫描；另一方面可以通过集中的平台统计代码质量的好坏变化趋势。根据统计结果可以判定团队中的个人对编码规范的执行情况，决定用宽松的质量管理方式还是严格的方式。

## C.4 总结

代码质量关乎产品的质量，最容易改进的地方即是编码规范，收效也是最高的，它远比单元测试要容易付诸实践。一旦团队制定了编码规范，就应该严格执行，严格杜绝团队中编码规范拖后腿的现象。

也许可以采用 CoffeeScript 的方式来避免规范编码的问题，但是我相信在使用 CoffeeScript 之前，了解这些规范会更好地帮助你理解 CoffeeScript。

如果你还采用非编译式地 JavaScript 来编写你的应用，请记住这些编码规范。尽管因为历史原因无法一步到位改进这些缺点，但是既然知晓何为优秀，何为糟粕，就应该将优秀当作一种习惯。
