# 指南

---

- [新手指南](#新手指南)
  - [Egg.js 是什么？](#eggjs-是什么)
  - [Egg.js 与 Koa](#eggjs-与-koa)
  - [快速入门](#快速入门)
  - [渐进式开发](#渐进式开发)
- [基础功能](#基础功能)
  - [目录结构](#目录结构)
  - [内置对象](#内置对象)
  - [运行环境](#运行环境)
  - [配置](#配置)
  - [中间件](#中间件)
  - [路由（Router）](#路由router)
  - [控制器（Controller）](#控制器Controller)
  - [服务（Service）](#服务Service)
  - [插件](#插件-1)
  - [定时任务](#定时任务)
  - [框架扩展](#框架扩展)
  - [启动自定义](#启动自定义)
- [核心功能](#核心功能)
  - [本地开发](#本地开发)
  - [单元测试](#单元测试-1)
  - [应用部署](#应用部署)
  - [日志](#日志)
  - [HttpClient](#httpclient)
  - [Cookie and Session](#cookie-and-session)
  - [多进程模型与进程间通讯](#多进程模型与进程间通讯)
  - [模板渲染](#模板渲染-1)
  - [异常处理](#异常处理-1)
  - [安全](#安全)
  - [国际化](#国际化)
- [教程](#教程)
  - [MySQL](#mysql)
  - [Sequelize](#sequelize)
  - [RESTful API](#restful-api)
  - [Passport 鉴权](#passport-鉴权)
  - [Soket.IO](#soketio)
  - [静态资源](#静态资源-1)
  - [TypeScript](#typescript)
  - [前置代理模式](#前置代理模式)
- [进阶](#进阶)
  - [加载器（Loader）](#加载器Loader)
  - [插件开发](#插件开发)
  - [框架开发](#框架开发)
  - [多进程研发模式增强](#多进程研发模式增强)
  - [模板插件开发规范](#模板插件开发规范)
  - [代码风格指南](#代码风格指南)

---

## 新手指南

### Egg.js 是什么？

**Egg.js 为企业级框架和应用而生**，我们希望由 Egg.js 孕育出更多上层框架，帮助开发团队和开发人员降低开发和维护的成本。

> 注： Egg.js 缩写为 Egg

#### 设计原则

我们甚至企业级应用在追求规范和共建的同时，还需要考虑如何平衡不同团队之间的差异，求同存异。所以我们没有选择社区常见框架的大集市模式（集成如数据库、模板引擎、前端框架等功能），而是专注于提供 Web 开发的核心功能和一套灵活可扩展的插件机制。我们不会做出技术选型，因为固定的技术选型会使框架的扩展性变差，无法满足各种定制需求。通过 Egg，团队的架构师和技术负责人可以非常容易地基于自身技术架构在 Egg 地基础上扩展出适合自身业务场景地框架。

Egg 奉行【**约定优于配置**】，按照*一套统一的约定*进行应用开发，团队内部采用这种方式可以减少开发人员的学习成本，开发人员不再是【钉子】，可以流动起来。没有约定的团队，沟通成本也是非常高的，比如有人会按目录分栈而其他人按目录分功能，开发者认知不一致很容易犯错。但约定不等于扩展性差，相反 Egg 有很高的扩展性，可以按照团队的约定定制框架。而*loader*可以让框架根据不同的环境定义默认配置，还可以覆盖 Egg 的默认约定。

#### 与社区框架的差异

Express 是 Node 社区广泛应用的框架，简单且扩展性强，非常适合个人项目。但是框架本身缺少约定，标准的 MVC 模型会有各种千奇百怪的写法。Egg 按照约定进行开发，奉行【约定优于配置】，团队协作成本低。

Sails 是和 Egg 一样奉行【约定优于配置】的框架，扩展性也非常好。但是相比 Egg，Sails 支持 Blueprint REST API、WaterLine 这样可扩展的 ORM、前端集成、WebSocket 等，这些功能都是由 Sails 提供的。而 Egg 不提供功能，只是集成各种功能插件，比如实现 egg-blueprint，egg-waterline 等这样的插件，再使用 sails-egg 框架整合这些插件就可以替代 Sails 了。

#### 特性

- 提供基于 Egg*定制上层框架*的能力；
- 高度可扩展的*插件机制*；
- 内置*多进程管理*；
- 基于*Koa*开发，性能优异；
- 框架稳定，测试覆盖率高；
- _渐进式开发_；

### Egg.js 与 Koa

#### 异步编程模型

Node 是一个异步的世界，官方 API 支持的都是 callback 形式的异步模型，这带来了许多问题：

- callback hell：最臭名昭著的 callback 嵌套问题。
- release zalgo：异步函数中可能同步调用 callback 返回数据，带来不一致性。

因此社区提供了各种异步的解决方案，最终胜出的是 Promise，它也内置到了 ECMAScript 2015 中。而在 Promise 的基础上，结合 Generator 提供的切换上下文能力，出现了`co`等第三方类库来让我们用同步写法编写异步代码。同时`async function`这个官方解决方案也于 ECMAScript 2017 中发布，并在 Node 8 中实现。

##### async function

`async function`是语言层面提供的语法糖，在`async function`中，我们可以通过`await`关键字来等待一个 Promise 被 resolve(或 reject，此时会抛出异常)，Node 现在的 LTS 版本（8.x）已原生支持。

```js
const fn = async function () {
  const user = await getUser();
  const posts = await fetchPosts(user.id);
  return { user, posts };
};

fn()
  .then(res => console.log(res))
  .catch(err => console.error(err.stack));
```

#### Koa

> Koa 是一个新的 Web 框架，由 Express 幕后的原班人马打造，致力于称为 Web 应用和 API 开发领域中的一个更小、更富有表现力、更健壮的基石。

Koa 和 Express 的设计风格非常类似，底层也都是共用的同一套 HTTP 基础库，但是有几个显著的区别，出了上面提到的默认异步解决方案之外，主要的特点还有以下几个。

##### Middleware

Koa 的中间件和 Express 不同，Koa 选择了洋葱圈模型。

- 中间件洋葱
- 中间件执行顺序

所有的请求经过一个中间件的时候都会执行两次，对比 Express 形式的中间件，Koa 的模型可以非常方便的实现后置处理逻辑，对比 Koa 和 Express 的 Compress 中间件就可以明显的高手到 Koa 中间件模型的优势。

##### Context

和 Express 只有 Request 和 Response 两个对象不同，Koa 增加了一个 Content 对象，作为这次请求的上下文对象（在 Koa 1.x 中为中间件的`this`，在 Koa 2.x 中作为中间件的第一个参数传入）。我们可以将依次请求相关的上下文都挂载到这个对象上。类似于 `traceld`这种需要贯穿整个请求（在后续任何一个地方其它调用都会徐娅用到）的属性就可以挂载上去。相比于 request 和 respinse 而言更加符合语义。

同时 Context 上也挂载了 Request 和 Response 两个对象。和 Express 类似，这两个对象都提供了大量的便捷方法辅助开发：

- `get request.query`
- `get request.hostname`
- `set response.body`
- `set response.status`

##### 异常处理

通过同步方式编写异步代码带来的另一个非常大的好处就是异常处理非常自然，使用`try catch`就可以将按规范编写的代码中所有的错误捕获到。这样我们很便捷的编写一个自定义的错误处理中间件。

```js
async function onerror(ctx, next) {
  try {
    await next();
  } catch (err) {
    ctx.app.emit('error', err);
    ctx.body = 'Server Error';
    ctx.status = err.status || 500;
  }
}
```

只需要将这个中间件放在其它中间件之前，就可以捕获他们所有的同步或异步代码中抛出的异常了。

#### Egg 继承于 Koa

如上述，Koa 是一个非常优秀的框架，然而对于企业级应用来说，它还是比较基础的。

而 Egg 选择了 Koala 作为其基础框架，在它的模型基础上，进一步对它进行了一些增强。

##### 扩展

在基于 Egg 的框架或应用中，我们通过`app/extend/{application,context,request,response}.js`来扩展 Koa 中对应的四个对象的原型，通过这个功能，我们可以快速的增加更多的辅助方法，例如我们在`app/extend/context.js`中写下：

```js
// app/extend/context.js

module.exports = {
  get isIOS() {
    const iosReg = /iphone|iPad|ipod/i;
    return iosReg.test(this.get('user-agent'));
  },
};
```

在 Controller 中，我们就可以使用刚才定义的这个便捷属性了：

```js
// app/controller/home.js
exports.handler = ctx => {
  ctx.body = ctx.isIOS ? 'Your operating system is iOS.' : 'Your operating system is not iOS.';
};
```

更多关于扩展的内容，请查看扩展章节。

##### 插件

众所周知，在 Express 和 Koa 中，经常会引入许多中间件来提供各种各样的功能，例如引入`koa-session`提供`Session`的支持，引入`koa-bodyparser`来解析请求 body。而 Egg 提供了一个更强大的插件机制，让这些独立领域的功能模块更加容易编写。

一个插件可以包含：

- extend：扩展基础对象的上下文，提供各类工具类、属性。
- middleware：增加一个或多个中间件，提供请求的前置、后置处理逻辑。
- config：配置各个环境下插件自身的默认配置项。

一个独立领域下的插件实现，可以在代码维护性非常高的情况下实现非常完善的功能，而插件也支持配置各个环境下默认（最佳）配置，让我们使用插件的时候几乎可以不需要修改配置项。

`egg-security`插件就是一个典型的例子。

更多关于插件的内容，请查看插件的章节。

##### Egg 与 Koa 的版本关系

- Egg 1.x

Egg 1.x 发布时，Node 的 LTS 尚不支持 async function，所以 Egg 1.x 仍然基于 Koa 1.x 开发，但是在此基础上，Egg 全面增加了 async function 的支持，再加上 Egg 对 Koa 2.x 的中间件也完全兼容，应用层代码可以完全基于`async function`来开发。

- 底层基于 Koa 1.x，异步解决方案基于`co`封装的 generator function。
- 官方插件以及 Egg 核心使用 generatro function 编写，保持对 Node LTS 版本的支持，在必要处通过 co 包装以兼容在 async function 中的使用。
- 应用开发者可以选择 async function(Node 8.x+) 或者 generator function(Node 6.x+)进行编写。

- Egg 2.x

Node 8 正式进入 LTS 后，async function 可以在 Nodejs 中使用并且没有任何性能问题了，Egg 2.x 基于 Koa 2.x，框架底层以及所有内置插件都使用 async function 编写，并保持了对 Egg 1.x 以及 generator function 的完全兼容，应用层只需要升级到 Node 8 即可从 Egg 1.x 迁移到 Egg 2.x。

- 底层基于 Koa 2.x，异步解决方案基于 async function。
- 官方插件以及 Egg 核心使用 async function 编写。
- 建议业务层迁移到 async function 方案
- 只支持 Node 8 及以上版本。

### 快速入门

本文从实例的角度，一步步地搭建一个 Egg 应用，让你能快速入门 Egg

#### 环境准备

#### 快速初始化

我们推荐直接用脚手架，只需要几条简单指令，即可快速生成项目（npm >= 6.1.0）：

```sh
mkdir egg-example && cd egg-example
npm init egg --type=simple
npm i
```

启动项目：

```sh
npm run dev

# open http://localhost:7001/
```

#### 逐步搭建

通常可以通过上一届地方式，使用`npm init egg`夸苏选择适合对应业务模型地脚手架，快速启动 Egg 项目地开发。

但为了让大家更好的了解 Egg，接下来我们跳过脚手架，手动一步步的搭建出一个 Hack News。

**注意：实际项目中，我们推荐使用上一届的脚手架直接初始化**。

##### 初始化项目

先来初始化目录结构：

```sh
mkdir egg-example
cd egg-example
npm init
npm i egg --save
npm i egg-bin --save-dev
```

添加`npm scripts`到`package.json`：

```json
{
  "name": "egg-example",
  "scripts": {
    "dev": "egg-bin dev"
  }
}
```

##### 编写 Controller

如果你熟悉 Web 开发或者 MVC，肯定菜刀我们第一步需要编写的是 Controller 和 Router。

```js
// app/controller/home.js
const Controller = require('egg').Controller;

class HomeController extends Controller {
  async index() {
    this.ctx.body = 'Hello World';
  }
}

module.exports = HomeController;
```

配置路由映射：

```js
// app/router.js
module.exports = app => {
  const { router, controller } = app;
  router.get('/', controller.home.index);
};
```

加一个配置文件：

```js
// config/config.default.js
exports.keys = <此处改为自己的Cookie安全字符串>;
```

此时目录结构如下：

```sh
egg-example
├── app
│   ├── controller
│   │   └── home.js
│   └── router.js
├── config
│   └── config.default.js
└── package.json
```

完整的目录结构规范参见目录结构。

好，现在我们启动应用来体验下：

```sh
npm run dev
# open http://localhost:70001
```

> 注意：
>
> - Controller 有 `class` 和 `exports`两种编写方式，本文示范的是前者，你可能需要参考 Controller 文档。
> - Config 也有 `module.exports` 和 `exports`的写法，具体参考 Node modules 文档。

##### 静态资源

Egg 内置了 static 插件，线上环境建议部署到 CDN，无须该插件。

static 插件默认映射`/public/* -> app/public/*`目录。

此处，我们把静态资源都放到`app/public`目录即可：

```sh
app/public
├── css
│   └── news.css
└── js
    ├── lib.js
    └── news.js
```

##### 模板渲染

绝大多数情况，我们都需要读取数据后渲染模板，然后呈现给用户。故我们需要引入对应的模板引擎。

框架并不强制你使用某种模板引擎，只是约定了 View 插件开发规范，开发者可以引入不同的插件来实现差异化定制。

更多用法参见 View。
在本例中，我们使用`Nunjucks`来渲染，先安装对应的插件`egg-view-nunjucks`：

```sh
npm i egg-view-nunjucks --save
```

开启插件：

```js
// config/plugin.js
exports.nunjucks = {
  enable: true,
  package: 'egg-view-nunjucks',
};

// config/config.default.js
// ...

// 添加view配置
exports.view = {
  defaultViewEngine: 'nunjucks',
  mapping: {
    '.tpl': 'nunjucks',
  },
};
```

**注意：是`config`目录，不是`app/config`目录！**

为列表页编写模板文件，一般放在`app/view`目录下：

```html
<!-- app/view/news/list.tpl -->

<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <title>Hack News</title>
    <link rel="stylesheet" href="/public/css/news.css" />
  </head>
  <body>
    <ul class="news-view view">
      {% for item in list %}
      <li class="item"><a href="{{ item.url }}">{{ item.title }}</a></li>
      {% endfor %}
    </ul>
  </body>
</html>
```

添加 Controller 和 Router

```js
// app/controller/news.js
const Controller = require('egg').Controller;

class NewsController extends Controller {
  async list() {
    const dataList = {
      list: [
        { id: 1, title: 'this is news 1', url: '/news/1' },
        { id: 2, title: 'this is news 2', url: '/news/2' },
      ],
    };

    await this.ctx.render('news/list.tpl', dataList);
  }
}
module.export = NewsController;

// app/router.js
module.exports = app => {
  const { router, controller } = app;
  router.get('/', controller.home.index);
  router.get('/news', controller.news.list);
};
```

启动浏览器，访问`http://localhost:7001/news`即可看到渲染后的页面。

**提示：开发期默认开启了`development`插件，修改后端代码后，会自动重启 Worker 进程。**

##### 编写 service

在实际开发中，Controller 一般不会自己产生数据，也不会包含复杂的逻辑，复杂的过程应抽象为业务逻辑层 Service。

我们来添加一个 Service 抓取 Hack News 的数据：

```js
// app/service/news.js
const Service = require('egg').Service;

class NewsServer extends Service {
  async list(page = 1) {
    // read config
    const { serverUrl, pageSize } = this.config.news;

    // use build-in http client to GET hacker-news api
    const { data: idList } = await this.ctx.curl(`${serverUrl}/topstories.json`, {
      data: {
        orderBy: '"$key"',
        startAt: `"${pageSize * (page - 1)}"`,
        endAt: `"${pageSize * page - 1}"`,
      },
      dataType: 'json',
    });

    // parallel GET detail
    const newsList = await Promise.all(
      Object.eys(idList).map(key => {
        const url = `${serverUrl}item/${idList[key]}.json`;
        return this.ctx.curl(url, { dataType: 'json' });
      })
    );

    return newsList.map(res => res.data);
  }
}

module.exports = NewsService;
```

> 框架提供了内置的 HttpClient 来方便开发者使用 HTTP 请求。

稍微改动下之前的 Controller：

```js
// app/controller/news.js
async list() {
  const ctx = this.ctx;
  const page = ctx.query.page || 1;
  const newList = await ctx.service.news.list(page);
  await ctx.render('news/list.tpl', {list: newsList });
}
```

还需要增加`app/service/news.js`中读取到的配置：

```js
// config/config.default.js

// 添加news的配置项
exports.news = {
  pageSize: 5,
  serverUrl: 'https://hacker-news.firebaseio.com/v0',
};
```

##### 编写扩展

遇到一个小问题，我们的咨询时间的数据是 UnixTime 格式的，我们希望显示为便于阅读的格式。

框架提供了一种快速扩展的方式，只需要在`app/extend`目录下提供扩展脚本即可，具体参见扩展。

在这里我们使用 View 插件支持的 Helper 来实现：

```sh
npm i moment --save
```

```js
// app/extend/helper.js
const moment = require('moment');
exports.relativeTime = time => moment(new Date(time * 1000)).fromNow();
```

在模板里使用：

```html
<!-- app/view/news/list.tpl -->
{{ helper.relativeTime(item.time) }}
```

##### 编写 Middleware

假设有个需求：我们的新闻站点，禁止百度爬虫范文。

聪明的同学一定很快能想到可以通过 Middleware 判断 User-Agent，如下：

```js
// app/middleware/robot.js
module.exports = (options, app) => {
  return async robotMiddleware(ctx, next) {
    const source = ctx.get('user-agent') || '';
    const match = options.ua.some(ua => ua.test(source));

    if(match) {
      ctx.status = 403;
      ctx.message = 'Go away, robot.';
    } else {
      await next();
    }
  }
};

// config/config.default.js
// add middleware robot
exports.middleware = [
  'robot'
];

//robot's configurations
exports.robot = {
  ua: [
    /Baiduspider/i,
  ]
};
```

现在我们可以使用 `curl http://localhost:7001/news -A "Baiduspider"`看看效果。

更多参见中间件文档。

##### 配置文件

写业务的时候，不可避免地需要有配置文件，框架提供了强大的配置合并管理功能：

- 支持按环境变量加载不同的配置文件，如`config.local.js`，`config.prod.js` 等等。
- 应用/插件/框架都可以配置自己的配置文件，框架将按顺序合并加载。
- 具体合并逻辑可参见配置文件。

```js
// config/config.default.js
exports.robot = {
  ua: [/curl/i, /Baiduspider/i],
};

// config/config.local.js
// only read at development mode, will override default
exports.robot = {
  ua: [/Baiduspider/i],
};

// app/service/some.js
const Service = require('egg').Service;

class SomeService extends Service {
  async list() {
    const rule = this.config.robot.ua;
  }
}

module.exports = SomeService;
```

##### 单元测试

单元测试非常重要，框架也提供了`egg-bin`来帮助开发者无痛的编写测试。

测试文件应该放在项目根目录下的 test 目录下，并以`test.js`为后缀命名，即`{app_root}/test/**/*.text.js`。

```js
// test/app/middleware/robot.test.js
const { app, mock, assert } = require('egg-mock/bootstrap');

describe('test/app/middleware/robot.test.js', () => {
  it('should block robt', () => {
    return app.httpRequest().get('/').set('User-Agent', 'Baiduspider').expect(403);
  });
});
```

然后配置依赖和`npm scripts`：

```json
{
  "scripts": {
    "test": "egg-bin test",
    "cov": "egg-bin cov"
  }
}
```

```sh
npm i egg-mock --save-dev
```

执行测试：

```sh
npm test
```

就这么简单，更多请参见 单元测试。

#### 后记

短短几张内容只能讲 Egg 的冰山一角，我们建议开发者继续阅读其它章节：

- 关于骨架类型，参见骨架说明
- 提供了强大的扩展机制，参加插件。
- 一个大规模的团队需要遵循一定的约束和约定，在 Egg 里我们建议封装适合自己团队的上层框架，参见框架开发。
- 这是一个渐进式的框架，代码的共建，复用和下沉，竟然可以那么无痛，建议阅读渐进式开发。
- 写单元测试其实很简单的事，Egg 提供了非常多的配套辅助，我们强烈建议大家测试驱动开发，具体参见单元测试。

### 渐进式开发

在 Egg 中，有插件，也有框架，前者包括了`path`和`package`两种加载模式，那我们应该如何选择呢？

本文讲以实例的方式，一步步给大家演示下，如何渐进式地进行代码演进。

全部地示例代码可以参见[eggjs/examples/orogressive](https://github.com/eggjs/examples/tree/master/progressive)。

#### 最初始的状态

假设我们有一段分析 UA 地代码，实现以下功能：

- `ctx.isAndroid`
- `ctx.isIOS`

通过之前的教程，大家一定可以很快地写出来。

目录结构：

```sh
example-app
├── app
│   ├── extend
│   │   └── context.js
│   └── router.js
├── test
│   └── index.test.js
└── package.json
```

核心代码：

```js
// app/extend/context.js
module.exports = {
  get isIOS() {
    const iosReg = /iphone|ipad|ipod/i;
    return iosReg.test(this.get('user-agent'));
  },
};
```

#### 插件的雏形

我们很明显的感知到，这段逻辑是具备通用性的，可以写成插件。

但是一开始的时候，功能还没有完善，直接独立插件，维护起来比较麻烦。

此时，我们可以把代码协程插件的形式，但并不独立出去。

新的目录结构：

```sh
example-app
├── app
│   └── router.js
├── config
│   └── plugin.js
├── lib
│   └── plugin
│       └── egg-ua
│           ├── app
│           │   └── extend
│           │       └── context.js
│           └── package.json
├── test
│   └── index.test.js
└── package.json
```

核心代码：

- `app/extend/context.js`移动到`lib/plugin/egg-ua/app/extend/context.js`。
- `lib/plugin/egg-ua/package.json`声明插件

```json
{
  "eggPlugin": {
    "name": "ua"
  }
}
```

- `config/plugin.js`中通过`path`来挂载插件。

```js
// config/plugin.js
const path = require('path');
exports.ua = {
  enable: true,
  path: path.join(__dirname, '../lib/plugin/egg-ua'),
};
```

#### 抽成独立插件

经过一段时间开发后，该模块的功能成熟，此时可以考虑抽出来成为独立的插件。

首先我们抽出一个 egg-ua 插件，看过插件文档的同学应该都比较熟悉，这里我们简单过一下：

目录结构：

```sh
egg-ua
├── app
│   └── extend
│       └── context.js
├── test
│   ├── fixtures
│   │   └── test-app
│   │       ├── app
│   │       │   └── router.js
│   │       └── package.json
│   └── ua.test.js
└── package.json
```

对应代码参见[step3/egg-ua](https://github.com/eggjs/examples/tree/master/progressive/step3/egg-ua)。

然后改造原有应用：

- 移除`lib/plugin/egg-ua`目录。
- `package.json`中声明对`egg-ua`的依赖。
- `config/plugin.js`中修改依赖声明为`package`方式。

```js
//config/plugin.js
exports.ua = {
  enable: true,
  package: 'egg-ua',
};
```

**注意：插件还没发布前，可以通过`npm link`的方式本地测试，具体参见[npm-link](https://docs.npmjs.com/cli/link)**。

```sh
cd example-app
npm link ../egg-ua
npm i
npm test
```

#### 沉淀到框架

重复上述的过程，很快我们会积累好几个插件和配置，并且我们会发现在团队中的大部分项目中，都会用到这写插件。

此时，可以考虑抽象出一个适合团队业务场景的框架。

首先，抽象出 example-framework 框架，如上看过框架文档的同学，应该都会比较熟悉：

目录结构：

```sh
example-framework
├── config
│   ├── config.default.js
│   └── plugin.js
├── lib
│   ├── agent.js
│   └── application.js
├── test
│   ├── fixtures
│   │   └── test-app
│   └── framework.test.js
├── README.md
├── index.js
└── package.json
```

- 对应代码参见[example-framework](https://github.com/eggjs/examples/tree/master/progressive/step4/example-framework)

- 把原来的`egg-ua`等插件的依赖，从 example-app 中移除，配置到该框架的`package.json`和`config/plugin.js`中。

然后改造原有的应用：

- 移除`config/plugin.js`中对`egg-ua`的依赖。
- `package.json`中移除对`egg-ua`的依赖。
- `package.json`中声明对`example-framework`的依赖，并配置`egg.framework`。

```json
{
  "name": "progressive",
  "version": "1.0.0",
  "private": true,
  "egg": {
    "framework": "example-framework"
  },
  "dependencies": {
    "example-framework": "*"
  }
}
```

```sh
cd example-app
npm link ../egg-framework
npm i
npm test
```

#### 写在最后

综上所诉，大家可以看到我们是如何一步步渐进地去进行框架演进，这得益于 Egg 强大的插件机制、代码的共建，以及复用和下沉，这些步骤竟然可以那么地无痛来得以完成！

- 一般来说，当应用中有可能会服用到地代码时，字节放到`lib/plugin`目录去，如例子中的`egg-ua`。
- 当该插件功能稳定后，即可以独立出来作为一个`node module`。
- 如此以往，应用中相对复用性较强的代码都会逐渐独立为单独的插件。
- 当你的逐渐进化到针对某类业务场景的解决方案时，将其抽象为独立的 framework 进行发布。
- 当新项目中抽象出来的插件，下沉集成到框架后，其它项目只需要简单的重新`npm install`下就可以使用上，这对于整个团队的效率有极大提升。

**不管是应用/插件/框架，都必须编写单元测试，并尽量实现 100%覆盖率**。

## 基础功能

### 目录结构

在快速入门中，大家对框架应该有了初步的印象，接下来我们简单了解下目录约定规范。

```sh
egg-project
├── package.json
├── app.js (可选)
├── agent.js (可选)
├── app
|   ├── router.js
│   ├── controller
│   |   └── home.js
│   ├── service (可选)
│   |   └── user.js
│   ├── middleware (可选)
│   |   └── response_time.js
│   ├── schedule (可选)
│   |   └── my_task.js
│   ├── public (可选)
│   |   └── reset.css
│   ├── view (可选)
│   |   └── home.tpl
│   └── extend (可选)
│       ├── helper.js (可选)
│       ├── request.js (可选)
│       ├── response.js (可选)
│       ├── context.js (可选)
│       ├── application.js (可选)
│       └── agent.js (可选)
├── config
|   ├── plugin.js
|   ├── config.default.js
│   ├── config.prod.js
|   ├── config.test.js (可选)
|   ├── config.local.js (可选)
|   └── config.unittest.js (可选)
└── test
    ├── middleware
    |   └── response_time.test.js
    └── controller
        └── home.test.js
```

如上，由框架约定的目录：

- `app/router.js` 用于配置 URL 路由规则，具体参见 Router。
- `app/controller/**` 用于解析用户的输入，处理后返回相应的结果，具体参见 Controller。
- `app/service/**` 用于编写业务逻辑层，可选，建议使用，具体参见 Service。
- `app/middleware/**` 用于编写中间件，可选，具体参见 Middleware。
- `app/public/**` 用于防止静态资源，可选，具体参见内置插件 egg-static。
- `app/extend/**` 用于框架的扩展，可选，绝体参见框架扩展。
- `config/config.{env}.js` 用于编写配置文件，具体参见配置。
- `config/plugin.js` 用于配置需要加载的插件，具体参见插件。
- `test/**` 用于单元测试，具体参见单元测试。
- `app.js`和`agent.js` 用于自定义启动时的初始化工作，可选，具体参见启动自定义。关于`agent.js`的作用参见 Agent 机制。

由内置插件约定的目录：

- `app/public/**` 用于存放静态资源，可选，具体参见内置插件 egg-static。
- `app/schedule/**` 用于定时任务，可选，具体参见定时任务。

若需要定义自己的目录规范，参见 Loader API：

- `app/view/**` 用于放置模板文件，可选，由模板插件约定，具体参见模板渲染。
- `app/model/**` 用于放置领域模型，可选，由领域类相关插件约定，如 egg-sequelize。

### 内置对象

在本章，我们会初步介绍以下框架中内置的一些基础对象，包括从 Koa 集成而来的 4 个对象（Application，Context，Request，Response）以及框架扩展的一些对象（Controller、Service、Helper、Config、Logger），在后续的文档阅读中我们经常会遇到它们。

#### Application

Application 是全局应用对象，在一个应用中，只会实例化一个，它继承自 Koa.Application，在它上面我们可以挂载一些全局方法和对象。我们可以轻松的在插件或者应用中扩展 Application 对象。

##### 事件

在框架运行时，会在 Application 实例上触发一些事件，应用开发者或者插件开发者可以监听这些事件做一些操作。作为应用开发者，我们一般会在启动自定义脚本中进行监听。

- server：该事件一个 worker 进程只会触发依次，在 HTTP 服务完成启动后，会将 HTTP server 通过这个事件暴露出来给开发者。

- error： 运行时有任何异常被 onerror 插件补货后，都会触发 error 事件，讲错误对象和关联的上下文（如果有）暴露给开发者，可以进行自定义的日志记录上报等处理。

- request 和 response：应用收到请求和响应请求时，分别触发 request 和 response 事件，并将当前请求上下文暴露出来，开发者可以监听这两个世家进行日志记录。

```js
// app.js
module.exports = app => {
  app.once('server', server => {
    // websocket
  });

  app.on('error', (err, ctx) => {
    // report error
  });

  app.on('request', ctx => {
    // log receive request
  });

  app.on('response', ctx => {
    // ctx.starttime is set by framework
    const used = Date.now() - ctx.starttime;
    // log total cost
  });
};
```

##### 获取方式

Application 对象几乎可以在编写应用时任何一个地方都获取到，下面介绍几个经常用到的获取方式：

技术所有被框架 Loader 加载的文件（Controller、Service、Schedule 等），都可以 export 一个函数，这个函数会被 Loader 调用，并使用 app 作为参数：

- 启动自定义脚本

```js
// app.js
module.export = app => {
  app.cache = new Cache();
};
```

- Controller 文件

```js
// app/controller/user.js
class UserController extends Controller {
  async fetch() {
    this.ctx.body = this.app.cache.get(this.ctx.query.id);
  }
}
```

和 Koa 一样，在 Context 对象上，可以通过`ctx.app`访问到 Application 对象。

在继承于 Controller，Service 基类的实例中，可以通过`this.app`访问到 Application 对象。

#### Context

Context 是一个请求级别的对象，继承自 Koa.Context。在每一次收到用户请求是，框架会实例化一个 Context 对象，这个对象封装了本次用户请求的信息，并提供了许多边界的方法来获取请求参数或者设置相应信息。框架会将所有的 Service 挂载到 Context 实例上，一些插件也会将一些其它的方法和对象挂载到它上面（egg-sequelize 会将所有的 model 挂载在 Context 上）。

##### 获取方式

最常见的 Context 实例获取方式实在 Middleware，Controller 以及 Service 中。Controller 中的获取方式在上面的例子中已经展示过了，在 Service 中获取和 Controller 中获取方式一样，在 Middleware 中获取 Context 实例则和 Koa 框架在中间件中获取 Context 对象的方式是一致。

框架的 Middleware 同时支持 Koa 1.x 和 Koa 2.x 两种不同的中间件写法，根据不同的写法，获取 Context 实例的方式也稍有不同：

```js
// Koa 1.x
function* middleware(next) {
  // this is instance of Context
  console.log(this.query);
  yield next;
}

// Koa 2.x
async function middleware(ctx, next) {
  // ctx is instance of Context
  console.log(ctx.query);
}
```

出了在请求时可以获取 Context 实例外，在有些非法用户请求的场景下我们需要访问 service/model 等 Context 实例上的对象，我们可以通过`Application.createAnonymousContext()`方法创建一个匿名 Context 实例：

```js
// app.js
module.exports = app => {
  app.beforeStart(async () => {
    const ctx = app.createAnonymousContext();
    // preload before app start
    await ctx.service.posts.load();
  });
};
```

在定时任务中的每一个 task 都接受一个 Context 实例作为参数，以便我们更方便的执行一些定时的业务逻辑：

```js
// app/schedule/refresh.js
exports.task = async ctx => {
  await ctx.service.posts.refresh();
};
```

#### Request & Response

Request 是一个请求级别的对象，继承自 Koa.Request。封装了 Node 原生的 HTTP Request 对象，提供了一系列辅助方法获取 HTTP 请求常用参数。

Response 是一个请求级别的对象，继承自 Koa.Response。封装了 Node 原生的 HTTP Response 对象，提供了一系列辅助方法设置 HTTP 响应。

##### 获取方式

可以在 Context 的实例上获取到当前请求的 Request(ctx.request)和 Response(ctx.response)实例。

```js
// app/controller/user.js
class UserController extends Controller {
  async fetch() {
    const { app, ctx } = this;
    const id = ctx.request.query.id;
    ctx.response.body = app.cache.get(id);
  }
}
```

- Koa 会在 Context 上代理一部分 Request 和 Response 上的方法和属性，参见 Koa.Context。
- 上面例子中`ctx.request.query.id`和`ctx.query.id`是等价的，`ctx.response.body=`和`ctx.body=`是等价的。
- 需要注意的是，获取 POST 的 body 应该使用`ctx.request.body`，而不是`ctx.body`。

#### Controller

框架提供了一个 Controller 积累，并推荐所有的 Controller 都继承于该基类实现。这个 Controller 基类有下列属性：

- ctx - 当前请求的 Context 实例
- app - 应用的 Application 实例
- config - 应用的配置
- service - 应用所有的 service
- logger - 为当前 contrller 封装的 logger 对象。

在 Controller 文件中，可以通过两种方式来引用 Controller 基类：

```js
// app/controller/user.js

// 从egg上获取（推荐）
const Controller = require('egg').Controller;
class UserController extends Controller {
  // implement
}
module.exports = UserController;

// 从app实例上获取：

module.exports = app => {
  return class UserController extends app.Controller {
    // implement
  };
};
```

#### Service

框架提供了一个 Service 基类，并推荐所有的 Service 都继承于该基类实现。

Service 基类的属性和 Controller 基类属性一致，访问方式也类似：

```js
// app/service/user.js

// 从egg上获取（推荐）
const Service = require('egg').Service;
class UserService extends Service {
  // implement
}
module.exports = UserService;

// 从app上获取
module.exports = app => {
  return class UserService extends app.Service {
    // implement
  };
};
```

#### Helper

Helper 用来提供一些使用的 utility 函数。它的作用在于我们可以将一些常用的动作抽离在 helper.js 里面称为一个独立的函数，这样可以用 JavaScript 来写复杂的逻辑，避免逻辑分散各处，同时可以更好的编写测试用例。

Helper 自身是一个类，有和 Controller 基类一样的属性，它也会在每次请求时进行实例化，因此 Helper 上的所有函数也能获得当前请求相关的上下文信息。

##### 获取方式

可以在 Context 的实例上获取到当前请求的 Helper（ctx.helper)实例。

```js
// app/controller/user.js
class UserController extend Controller {
  async fetch() {
    const { app, ctx } = this;
    const id = ctx.query.id;
    const user = app.cache.get(id);
    ctx.body = ctx.helper.formatUser(user);
  }
}
```

除此之外，Helper 的实例还可以在模板中获取到，例如可以在模板中获取到 security 插件提供的 shtml 方法。

```html
<!-- app/view/home.nj -->
{{ helper.shtml(value) }}
```

##### 自定义 helper 方法

应用开发中，我们可能经常要自定义一些 helper 方法，例如上面例子中的`formatUser`，我们可以通过框架扩展的形式来定义 helper 方法。

```js
// app/extend/helper.js
module.exports = {
  formatUser(user) {
    return only(user, ['name', 'phone']);
  },
};
```

#### Config

我们推荐应用开发遵循配置和代码分离的原则，将一些需要硬编码的业务配置都放到配置文件中，同时配置文件支持各个不同的运行环境使用不同的配置，使用起来也非常方便，所有框架、插件和应用级别的配置都可以通过 Config 对象获取到，关于框架的配置，可以详细阅读 Config 配置章节。

##### 获取方式

我们可以通过`app.config`从 Application 实例上获取到 config 对象，也可以在 Controller，Service，Helper 的实例上通过`this.config`获取到 config 对象。

#### Logger

框架内置了强大的日志功能，可以非常方便的打印各种级别的日志到对应的日志文件中，每个 logger 对象都提供了 4 个级别的方法：

- `logger.debug()`
- `logger.info()`
- `logger.warn()`
- `logger.error()`

在框架中提供了多个 Logger 对象，下面我们简单介绍下各个 Logger 对象的获取方式和使用场景。

##### App Logger

我们可以通过`app.coreLogger`来获取它，一般我们在开发应用时都不应该通过 CoreLogger 打印日志，而框架和插件则需要通过它来打印应用级别的日志，这样可以更轻清晰的区分应用和框架打印的日志，通过 CoreLogger 打印的日志会放到和 Logger 不同的文件中。

##### Context Logger

我们可以通过`ctx.logger`从 Context 实例上获取到它，从访问方式上我们可以看出，Contter Logger 一定是与请求相关的，它打印的日志都会在前面带上一些当前请求相关的信息（如：`[$userId/$ip/$traceId/${cost}ms $method $url]`），通过这些信息，我们可以从日志快速定位请求，并串联一次请求中的所有的日志。

##### Context CoreLogger

我们可以通过`ctx.coreLogger`获取到它，和 Context Logger 的区别是一般只有插件和框架会通过它来记录日志。

##### Controller Logger & Service Logger

我们可以在 Controller 和 Service 实例上通过 `this.logger` 获取它们，它们本质上就是一个 Context Logger，不过在打印日志的时候还会额外的加上文件路径，方便定位日志的打印位置。

#### Subscription

订阅模式是一种比较常见的开发模式，譬如消息中间件的消费者或调度任务。因此我们提供了 Subscription 基类来规范化这个模式。

可以通过以下方式来引用 Subscription 基类：

```js
const Subscription = require('egg').Subscription;

class Schedule extends Subscription {
  async subscribe() {
    // 需要实现此方法
    // subscribe 可以为 async function 或 generator function
  }
}
```

插件开发者根据自己的需求基于它定制订阅规范，如定时任务就是使用这种规范实现的。

### 运行环境

一个 Web 应用本身应该是无状态的，并拥有根据运行环境设置自身的能力。

##### 指定运行环境

框架有两种方式指定运行环境：

1. 通过`config/env`文件指定，该文件的内容就是运行环境，如`prod`。一般通过构建工具来生成这个文件。

```js
// config/env
prod;
```

2. 通过`EGG_SERVER_ENV`环境变量指定运行环境更加方便，比如在生产环境中启动应用：

```sh
EGG_SERVER_ENV=prod npm start
```

##### 应用内获取运行环境

框架提供了变量`app.config.env`来表示应用当前的运行环境。

##### 运行环境相关配置

不同的运行环境对应不同的配置，具体请阅读 Config 配置。

##### 与 NODE_ENV 的区别

很多 Node 应用会使用`NODE_ENV`来区分运行环境，但`EGG_SERVER_ENV`区分得更加精细。一般的项目开发流程包括本地开发环境、测试环境、生产环境等，除了本地开发环境和测试环境外，其它环境可统称为**服务器环境**，服务器环境的`NODE_ENV`应该为`production`。而且 npm 也会使用这个变量，在应用部署的时候一般不会安装`devDependencies`，所以这个值也应该是`production`。

框架默认支持的运行环境及映射关系（如果未指定`EGG_SERVER_ENV`会根据`NODE_ENV`来匹配）。

| NODE_ENV   | EGG_SERVER_ENV | 说明         |
| ---------- | -------------- | ------------ |
|            | local          | 本地开发环境 |
| test       | unittest       | 单元测试     |
| production | prod           | 生产环境     |

例如，当`NODE_ENV`为`production`而`EGG_SERVER_ENV`未指定时，框架会将`EGG_SERVER_ENV`设置成`prod`。

##### 自定义环境

常规开发流程可能不仅仅只有以上几种环境，EGG 支持自定义环境来适应自己的开发流程。

比如，要为开发流程增加集成测试环境 SIT。将`EGG_SERVER_ENV`设置成`sit`（并建议设置`NODE_ENV`=`production`），启动会加载`config/config.sit.js`，运行环境变量`app.config.env`会被设置成`sit`。

##### 与 Koa 的区别

在 Koa 中我们通过`app.env`来进行环境判断，`app.env`默认的值是`process.env.NODE_ENV`。但是在 Egg(和基于 Egg 的框架)中，配置统一都放在`app.config`上，所以我们需要通过`app.config.env`来区分环境，`app.env`不再使用。

### 配置

框架提供了强大且可扩展的配置功能，可以自动合并应用、插件、框架的配置，按顺序覆盖，且可以根据环境维护不同的配置。合并后的配置可直接从`app.config`获取。

配置的管理方案有多种方案，以下列出一些常见的方案：

1. 使用平台管理配置，应用构建是当前环境和配置放入包内，启动时指定该配置。但应用就无法一次构建多次部署，而且本地开发环境想使用配置会变得很麻烦。

2. 使用平台管理部署，在启动时将当前环境的配置通过环境变量传入，这是比较优雅的方式，但框架对运维的要求会比较高，需要部署平台支持，同时开发环境也有相同的痛点。

3. 使用代码管理配置，在代码中添加多个环境的配置，在启动时传入当前环境的的参数即可。但无法全局配置，必须修改代码。

我们选择了最后一种配置方案，**配置即代码**，配置的变更也应该经历过 review 后才能发布。应用包本身是可以部署在多个环境的，只需要指定运行环境即可。

##### 多环境配置

框架支持根据环境来加载配置，定义多个环境的配置文件，具体环境请查看运行环境配置：

```sh
config
|- config.default.js
|- config.prod.js
|- config.unittest.js
|- config.local.js
```

`config.default.js`为默认的配置文件，所有环境都会加载这个配置文件，一般也会作为开发环境的默认配置文件。

当指定 env 时会同时加载对应的配置文件，并覆盖默认配置文件的同名配置。如`prod`环境会加载`config.prod.js`和`config.default.js`文件，`config.prod.js`会覆盖`config.default.js`的同名配置。

##### 配置写法

配置文件返回的是一个 object 对象，可以覆盖框架的一些配置，应用也可以将自己业务的配置放在这里方便管理。

```js
// 配置logger文件的目录，logger默认配置由框架提供
module.exports = {
  logger: {
    dir: '/home/admin/logs/demoapp',
  },
};
```

配置文件也可以简化写成 `exports.key = value`形式：

```js
exports.keys = 'my-cookie-secret-key';
exports.logger = {
  level: 'DEBUG',
};
```

配置文件也可以返回一个 function，可以接收 appInfo 参数：

```js
// 将logger目录放到代码目录下
const path = require('path');
module.exports = appInfo => {
  return {
    logger: {
      dir: path.join(appInfo.baseDir, 'logs'),
    },
  };
};
```

内置的 appInfo 有：

| appInfo | 说明                                                                 |
| ------- | -------------------------------------------------------------------- |
| pak     | package.json                                                         |
| name    | 应用名，同 pak.name                                                  |
| baseDir | 应用代码的目录                                                       |
| HOME    | 用户目录，如 admin 账户为/home/admin                                 |
| root    | 应用根目录，只有在 local 和 unittest 环境下为 baseDir，其它都为 HOME |

`appInfo.root`是一个优雅的适配，比如在服务器环境我们使用`/home/admin/logs`作为日志目录，而本地开发时又不想污染用户目录，这样的适配就很好解决这个问题。

请根据具体场合选择合适的写法，但请确保没有写处以下代码：

```js
exports.someKeys = 'abc';
module.exports = appInfo => {
  const config = {};
  config.keys = '123456';
  return config;
};
```

##### 配置加载顺序

应用、插件、框架都可以定义这些配置，而且目录结构都一致的，但存在优先级（应用 > 框架 > 插件），相对于此于宁环境的优先级会更高。

比如在 prod 环境加载一个配置的加载顺序如下，后加载的会覆盖前面的同名配置：

```sh
-> 插件 config.default.js
-> 框架 config.default.js
-> 应用 config.default.js
-> 插件 config.prod.js
-> 框架 config.prod.js
-> 应用 config.prod.js
```

**注意：插件之间也会有加载顺序，大致顺序类似，具体逻辑可查看加载器**。

##### 合并规则

配置的合并使用[extend2](https://github.com/eggjs/extend2)模块进行深度拷贝，extend2 fork 自 extend，处理数组时会存在差异。

```js
const a = {
  arr: [1, 2],
};
const b = {
  arr: [3],
};
extend(true, a, b);
// => { arr: [3] }
```

##### 配置结果

框架在启动时会把合并后的最终配置 dump 到`run/application_config.json`（worker 进程）和`run/agent_config.json`（agent 进程）中，可以用来分析问题。

配置文件中会隐藏一些字段，主要包括两类：

- 如密码、密钥等安全字段，这里可以通过`config.dump.ignore`配置，必须时 Set 类型，查看默认配置。

- 如函数、Buffer 等类型，`JSON.stringify`后内容特别大

还会生成`run/application_config_meta.json`（worker 进程）和`run/agent_config_meta.json`（agent 进程）文件，用来排查属性的来源，如：

```js
{
  "logger": {
    "dir": "/path/to/config/config.default.js"
  }
}
```

### 中间件

在前面的章节中，我们介绍了 Egg 是基于 Koa 实现的，所以 Egg 的中间件形式和 Koa 的中间件形式是一样的，都是基于洋葱圈模型。每次我们编写一个中间件，就相当于在洋葱外面包了一层。

#### 编写中间件

##### 写法

我们先来通过编写一个简单的 gzip 中间件，来看看中间件的写法：

```js
// app/middleware/gzip.js
const isJSON = require('koa-is-json');
const zlib = require('zlib');

async function gzip(ctx, next) {
  await next();

  // 后续中间件执行完后将响应体转换成gzip
  let body = ctx.body;
  if (!body) return;
  if (isJSON(body)) body = JSON.stringify(body);

  // 设置gzip body，修正响应头
  const stream = zlib.createGzip();
  stream.end(body);
  ctx.body = stream;
  ctx.set('Content-Encoding', 'gzip');
}
```

可以看到，框架的中间件和 Koa 的中间件写法是一模一样的，所以任何 Koa 的中间件都可以直接被框架使用。

##### 配置

一般来说中间件也会有自己的配置。在框架中，一个完整的中间件是包含了配置处理的。我们约定一个中间件是一个放置在`app/middleware`目录下的单独文件，它需要 exports 一个普通的 function，接收两个参数：

- options： 中间件的配置项，框架会将`app.config[${middlewareName}]`传递过来。
- app: 当前应用 Application 的实例。

我们将上面的 gzip 中间件做一个简单的优化，让他支持指定只有当 body 大于配置项的 threshold 时才进行 gzip 压缩，我们要在`app/middleware`目录下新建一个文件`gzip.js`：

```js
// app/middleware/gzip.js
const isJSON = require('koa-is-json');
const zlib = require('zlib');

module.exports = options => {
  return async function gzip(ctx, next) {
    await next();

    // 后续中间件执行完成后将响应体转换为gzip
    let body = ctx.body;
    if (!body) return;

    // 支持 options.threshold
    if (options.threshold && ctx.length < options.threshold) return;
    if (isJSON(body)) body = JSON.stringify(body);

    // 设置gzip body，修正响应头
    const stream = zlib.createGzip();
    stream.end(body);
    ctx.body = stream;
    ctx.set('Content-Encoding', 'gzip');
  };
};
```

#### 使用中间件

中间件编写完成后，我们还需要手动挂载，支持以下方式：

##### 在应用中使用中间件

在应用中，我们可以完全通过配置来加载自定义中间件，并决定他们的顺序。

如果我们需要加载上面的 gzip 中间件，在`config.default.js`中加入下面的配置就完成了中间件的开启和配置：

```js
module.exports = {
  // 配置需要的中间件，数组顺序即为中间件加载顺序
  middleware: ['gzip'],

  // 配置gzip中间件的配置
  gzip: {
    threshold: 1024, // 小于1k的响应体不压缩
  },
};
```

该配置最终将在启动时合并到`app.config.appMiddleware`。

##### 在框架和插件中使用中间件

框架和插件不支持在`config.default.js`中匹配`middleware`，需要通过以下方式：

```js
// app.js
module.exports = app => {
  // 在中间件最前面统计请求事件
  app.config.coreMiddleware.unshift('report');
};

// app/middleware/report.js
module.exports = () => {
  return async function (ctx, next) {
    const startTime = Date.now();
    await next();

    // 上报请求时间
    reportTime(Date.now() - startTime);
  };
};
```

应用层定义的中间件（`app.config.appMiddleware`)和框架默认中间件（`app.config.coreMiddleware`）都会被加载器加载，并挂载到`app.middleware`上。

##### router 中使用中间件

以上两种方式配置的中间件是全局的，会处理每一次请求，如果你只想针对单个路由生效，可以直接在`app/router.js`中实例化和挂载：

```js
module.exports = app => {
  const gzip = app.middleware.gzip({ threshold: 1024 });
  app.router.get('/needgzip', gzip, app.controller.handler);
};
```

#### 框架默认中间件

除了应用层加载中间件外，框架自身和其它的插件和其它的插件也会加载许多中间件。所有的这些自呆的中间件的配置项都通过配置中修改中间件同名配置项进行修改，例如框架自带的中间件中有一个 bodyParser 中间件（框架的加载器会将文件名中的各个分隔符都修改成驼峰形式的变量名），我们想要修改 bodyParser 的配置，只需要在`config/config.default.js`中编写：

```js
module.exports = {
  bodyParser: {
    jsonLimit: '10mb',
  },
};
```

**注意：框架和插件加载的中间件会在应用层配置的中间件之前，框架默认中间件不能被应用中间件覆盖，如果应用层有自定义同名中间件，在启动时会报错**。

#### 使用 Koa 的中间件

在框架里面可以非常容易地引入 Koa 中间件生态。

以 koa-compress 为例，在 Koa 中使用时：

```js
const Koa = require('koa');
const compress = require('koa-compress');

const app = new Koa();

const options = { threshold: 2048 };
app.use(compress(options));
```

我们按照框架地规范来应用中加载这个 Koa 地中间件：

```js
// app/middleware/compress.js
// koa-compress 暴露的接口(`(options) => middleware`)和框架对中间件要求一致
module.exports = require('koa-compress');

// config/config.default.js
module.exports = {
  middleware: ['compress'],
  compress: {
    threshold: 2048,
  },
};
```

如果使用的 Koa 中间件不符合入参规则，则可以自行处理：

```js
// config/config.default.js
module.exports = {
  webpack: {
    compiler: {},
    ohters: {},
  },
};

// app/middleware/webpack.js
const webpackMiddleware = require('some-koa-middleware');

module.exports = (options, app) => {
  return webpackMiddleware(options.compiler, options.other);
};
```

#### 通用配置

无论是应用层加载的中间件还是框架自带中间件，都支持几个通用的配置项：

- enable: 控制中间件是否开启。
- match：设置只有符合某些规则的请求才会经过这个中间件。
- ignore：设置符合某些规则的请求不经过这个中间件。

##### enable

如果我们的应用并不需要默认的 bodyParser 中间件来进行请求体的解析，此时我们可以通过配置 enable 为 false 来关闭它：

```js
module.exports = {
  bodyParser: {
    enable: false,
  },
};
```

##### match 和 ignore

match 和 ignore 支持的参数都一样，只是作用完全相反，match 和 ignore 不允许同时配置。

如果我们只想让 gzip 只针对`/static`前缀开头的 url 请求开启，我们可以配置 match 选项：

```js
module.exports = {
  gzip: {
    match: '/static',
  },
};
```

match 和 ignore 支持多种类型的配置方式：

1. 字符串：当参数为字符串类型是，配置的是一个 url 的路径前缀，所有以配置的字符串作为前缀的 url 都会匹配上。当然你也可以直接使用字符串数组。

2. 正则：当参数为正则时，直接匹配满足正则验证的 url 的路径。

3. 函数：当参数为一个函数时，会将请求上下文传递给这个函数，最终取函数返回的结果（true/false）来判断是否匹配。

```js
module.exports = {
  gzip: {
    match(ctx) {
      // 只有ios设备才开启
      const reg = /iphone|ipad|ipod/i;
      return reg.test(ctx.get('user-agent'));
    },
  },
};
```

有关更多的 match 和 ignore 配置情况，详见 egg-path-matching。

### 路由（Router）

Router 主要用来描述请求 URL 和具体承担执行动作的 Controller 的对应关系，框架约定了`app/router.js`文件用于统一所有路由规则。

通过统一的配置，我们可以避免路由规则逻辑散落在多个地方，从而出现未知的冲突,集中在一起我们可以更方便的来查看全局的路由规则。

#### 如何定义路由

- `app/router.js`里面定义 URL 路由规则

```js
// app/router.js

module.exports = app => {
  const { router, controller } = app;
  router.get('/user/:id', controller.user.info);
};
```

- `app/controller`目录下实现 Controller：

```js
// app/controller/user.js
class UserController extends Controller {
  async info() {
    const { ctx } = this;
    ctx.body = {
      name: `hello ${ctx.params.id}`,
    };
  }
}
```

这样就完成了一个最简单的 Router 定义，当用户执行`GET /user/123`，`user.js`这个里边的 info 方法就会执行。

#### Router 详细定义说明

下面时路由的完整定义，参数可以根据场景的不同，自由选择：

```js
router.verb('path-match', app.controller.action);
router.verb('router-name', 'path-match', app.controller.action);
router.verb('path-match', middleware1, ..., middlewareN, app.controller.action);
router.verb('router-name', 'path-match', middleware1, ..., middlewareN, app.controller.action);
```

酷游完整定义主要包括 5 个主要部分：

- verb - 用户触发动作，支持 get，post 等所有 HTTP 方法，后面会通过示例详细说明。
  - router.head - HEAD
  - router.get - GET
  - router.put - PUT
  - router.post - POST
  - router.patch - PATCH
  - router.delete - DELETE
  - router.del - DELETE，由于 delete 是一个保留字，所以提供了一个 delete 方法的别名。
  - router.redirect - 可以对 URL 进行重定向处理，比如我们最经常使用的可以把用户访问的根目录路由到某个主页。
- router-name 给路由设定一个别名，可以通过 Helper 提供的辅助函数`pathFor`和`urlFor`来生成 URL（可选）。
- path-match - 路由 URL 路径。
- middleware1 - 在 Router 里面可以配置多个 Middleware（可选）。
- controller - 指定路由映射到的具体的 controller 上，controller 可以有两种写法：
  - `app.controller.user.fetch` - 直接指定一个具体的 controller
  - `user.fetch` - 可以简写为字符串形式

##### 注意实现

- 在 Router 定义中，可以支持多个 Middleware 串联执行。
- Controller 必须定义在`app/controller`目录中。
- 一个文件里面也可以包含多个 Controller 定义，在定义路由的时候，可以通过`${filename}.${functionName}的方式指定对应的Controller`。
- Controller 支持子目录，在定义路由时，可以通过`${directoryName}.${fileName}.${functionName}`的方式指定对应的 Controller。

下面是一些路由定义的方式：

```js
// app/router.js

module.exports = app => {
  const { router, controller } = app;
  router.get('/home', controller.home);
  router.get('/user/:id', controller.user.page);
  router.post('/admin', isAdmin, controller.admin);
  router.post('/user', isLoginUser, hasAdminPermission, controller.user.create);
  router.post('/api/v1/comments', controller.v1.comments.create); // app/controller/v1/comments.js
};
```

##### RESTful 风格的 URL 定义

如果想通过 RESTful 的方式来定义路由，我们提供了`app.router.resources('routerName', 'pathMatch', controller)`快速在一个路径上生成 CRUD 路由结构。

```js
// app/router.js
module.exports = app => {
  const { router, controller } = app;
  router.resources('posts', '/api/posts', controller.posts);
  router.resources('users', '/api/v1/users', controller.v1.users); // app/controller/v1/users.js
};
```

上面代码就在`/posts`路径上部署了一组 CRUD 路径结构，对应的 Controller 为`app/controller/posts.js`。接下来，你只需要在`posts.js`里边实现对应的函数即可：

| Method | Path       | Route Name | Controller.Action             |
| ------ | ---------- | ---------- | ----------------------------- |
| GET    | /posts     | posts      | app.controllers.posts.index   |
| GET    | /posts/new | new_post   | app.controllers.posts.new     |
| GET    | /posts/:id | edit_post  | app.controllers.edit          |
| POST   | /posts     | post       | app.controllers.posts.create  |
| PUT    | /posts/:id | post       | app.controller.posts.update   |
| DELETE | /posts/:id | post       | app.controllers.posts.destroy |

```js
// app/controller/posts.js

exports.index = async () => {};
exports.new = async () => {};
exports.create = async () => {};
exports.show = async () => {};
exports.edit = async () => {};
exports.update = async () => {};
exports.destroy = async () => {};
```

如果我们不需要其中的某几个方法，可以不用再`posts.js`里面实现，这样对对应的 URL 路径也不会注册到 Router。

#### router 实战

下面通过更多实际例子，来说明 router 的用法。

##### 参数获取

###### Query String 方式

```js
// app/router.js
module.exports = app => {
  app.router.get('/search', app.controller.search.index);
};

// app/controller/search.js
exports.index = async ctx => {
  ctx.body = `search: ${ctx.query.name}`;
};

// curl http://127.0.0.1/search?name=egg
```

###### 参数命名方式

```js
// app.router.js
module.exports = app => {
  app.router.get('/user/:id/:name', app.controller.user.info);
};

// app/controller/user.js
exports.info = async ctx => {
  ctx.body = `user: ${ctx.params.id}, ${ctx.params.name}`;
};

// curl http://127.0.0.1:7001/user/123/xiaoming
```

###### 复杂参数的获取

路由里面也支持定义正则，可以更加灵活的获取参数：

```js
// app/router.js
module.exports = app => {
  app.router.get(/^\/package\/([\w.]+\/[\w-.]+)$/, app.controller.package.detail);
};

// app/controller/package.js
exports.detail = async ctx => {
  // 如果请求URL 被正则表达式匹配，可以按照捕获分组的顺序，从ctx.params 中获取。
  // 按照下面的用户请求，`ctx.params[0]`的内容就是`egg/1.0.0`
  ctx.body = `package: ${ctx.params[0]}`;
};

// curl http://127.0.0.1:7001/package/egg/1.0.0
```

##### 表单内容的获取

```js
// app/router.js
module.exports = app => {
  app.router.post('/form', app.controller.form.post);
};

// app/controller/form.js
exports.post = async ctx => {
  ctx.body = `body: ${JSON.stringify(ctx.request.body)}`;
};

// 模拟发起post请求：
// curl -X POST http://127.0.0.1:7001/form -data '{"name": "controller"}' --header 'Content-Type: application/json'
```

> 附：
> 这里直接发起 POST 请求会**报错**： 'secret is missing'。错误信息来自：koa-csrf/index.js#L69。
> **原因**： 框架内部针对表单 POST 请求均会验证 CSRF 的值，因此我们在表单提交时，带上 CSRF key 进行提交，可参考安全威胁 csrf 的防范。
> **注意**： 上面的校验是因为框架内置了安全插件 egg-security，提供了一些默认的安全实践，并且框架的安全插件时默认开启的，如果需要关闭其中一些安全防范，直接设置该项的 enable 属性为 false 即可。
> 除非清除的确认后果，否则不建议擅自关闭安全插件提供的功能。
> 这里在写例子的话可以临时在 `config/config.default.js`中设置：

```js
exports.security = {
  csrf: false,
};
```

##### 表单校验

```js
// app/router.js
module.exports = app => {
  app.router.post('/user', app.controller.user);
};

// app/controller/user.js
const createRule = {
  username: {
    type: 'email',
  },
  password: {
    type: 'password',
    compare: 're-password',
  },
};

exports.create = async ctx => {
  // 如果校验报错，会抛出异常
  ctx.validate(createRule);
  ctx.body = ctx.request.body;
};

// curl -X POST http://127.0.0.1:7001/user --data 'username=abc@abc.com&passowrd=111111&re-password=111111'
```

##### 重定向

###### 内部重定向

```js
// app/router.js
module.exports = app => {
  app.router.get('index', '/home/index', app.controller.home.index);
  app.router.redirect('', '/home/index', 302);
};

// app/controller/home.js
exports.index = async ctx => {
  ctx.body = 'hello controller';
};

// curl -L http://localhost:7001
```

###### 外部重定向

```js
// app/router.js
module.exports = app => {
  app.router.get('/search', app.controller.search.index);
};

// app/controller/search.js
exports.index = async ctx => {
  const type = ctx.query.type;
  const q = ctx.query.q || 'node.js';

  if (type === 'bing') {
    ctx.redirect(`http://cn.bing.com/search?q=${q}`);
  } else {
    ctx.redirect(`https://www.google.co.kr/search?q=${q}`);
  }
};

// curl http://localhost:7001/search?type=bing&q=node.js
// curl http://localhost:7001/search?q=node.js
```

##### 中间件的使用

我们想要把用户某一类请求的参数都大写，可以通过中间件来实现。这里我们只是简单说明下如何使用中间件，更多请查看中间件。

```js
// app/controller/search.js
exports.index = async ctx => {
  ctx.body = `search: ${ctx.query.name}`;
};

// app/middleware/uppercase.js
module.exports = () => {
  return async function uppercase(ctx, next) {
    ctx.query.name = ctx.query.name && ctx.query.name.toUppserCase();
    await next();
  };
};

// app/router.js
module.exports = app => {
  app.router.get('s', '/search', app.middleware.uppercase(), app.controller.search);
};

// curl http://localhost:7001/search?name=egg
```

##### 太多路由映射

如上所述，我们并不建议把路由规则逻辑散落在多个地方，会给排查问题带来困扰。

若确实有需求，可以如下拆分：

```js
// app/router.js
module.exports = app => {
  require('./router/news')(app);
  require('./router/admin')(app);
};

// app/router/news.js
module.exports = app => {
  app.router.get('/news/list', app.controller.news.list);
  app.router.get('news/detail', app.controller.news.detail);
};

// app/router/admin.js
module.exports = app => {
  app.router.get('/admin/user', app.controller.admin.user);
  app.router.get('/admin/log', app.controller.admin.log);
};
```

也可以使用 egg-router-plus。

### 控制器（Controller）

#### 什么是 Controller

我们通过 Router 将用户的请求基于 method 和 URL 分发到了对应的 Controller 上，那 Controller 负责做什么？

简单的说，Controller 负责**解析用户的输入，处理后返回相应的结果**，例如：

- 在 RESTful 接口中，Controller 接收用户的参数，从数据库中查找内容返回给用户或者将用户的请求更新到数据库中。

- 在 HTML 页面请求中，Controller 根据用户访问不同的 URL，渲染不同的模板得到 HTML 返回给用户。
- 在代理服务器中，Controller 将用户的请求转发给其它服务器，并将其它服务器的处理结果返回给用户。

框架推荐 Controller 层主要对用户的请求参数进行处理（校验、转换），然后调用对应的 service 方法处理业务，得到业务结果后封装并返回：

1. 获取用户通过 HTTP 传递过来的请求参数。
2. 校验、组装参数。
3. 调用 Service 进行业务处理，必要时处理转换 Service 的返回结果，让它适应用户的需求。
4. 通过 HTTP 将结果响应给用户。

#### 如何编写 Controller

所有的 Controller 文件都必须放在`app/controller`目录下，可以支持多级目录，访问的时候可以通过目录名级联访问。Controller 支持多种形式编写，可以根据不同的场景和开发习惯来选择。

##### Controller 类

我们可以通过定义 Controller 类的方式来编写代码：

```js
// app/controller/post.js
const Controller = require('egg').Controller;

class PostController extends Controller {
  async create() {
    const { ctx, service } = this;
    const createRule = {
      title: { type: 'string' },
      content: { type: 'string' },
    };

    // 校验参数
    const author = ctx.session.userId;
    const req = Object.assign(ctx.request.body, { author });
    // 设置Service进行业务处理
    const res = await service.post.create(req);

    // 设置相应内容和响应状态码：
    ctx.body = { id: res.id };
    ctx.status = 201;
  }
}
module.exports = PostController;
```

我们通过上面的代码定义了一个`PostController`的类，类里面的每一个方法都可以作为一个 Controller 在 Router 中引用到，我们可以从`app.controller`根据文件名的方法名定位找到他。

```js
// app/router.js
module.exports = app => {
  const { router, controller } = app;
  router.post('createPost', '/api/posts', controller.post.create);
};
```

Controller 支持多级目录，例如如果我们讲上面的代码放到`app/controller/sub/post.js`中，则可以在 router 中这样使用：

```js
// app/router.js
module.exports = app => {
  app.router.post('createPost', '/api/posts', app.controller.sub.post.create);
};
```

定义的 Controller 类，会在每一个请求访问到 server 时实例化一个全新对象，而项目中的 Controller 类继承于`egg.Controller`，会有下面几个属性挂载在`this`上。

- `this.ctx`：当前请求的上下文 Context 对象的实例，通过它我们可以拿到框架封装好的处理当前请求的各种便捷属性和方法。
- `this.app`：当前应用 Application 对象的实例，通过它我们可以拿到框架提供的全局对象和方法。
- `this.service`：应用定义的 Service，通过它我们可以访问到抽象出的业务层，等价于`this.ctx.service`。
- `this.config`：应用运行时的配置项。
- `this.logger`：logger 对象，上面有四个方法（`debug`、`info`、`warn`、`error`），分别代表打印四个不同等级的日志，使用方法和效果与 context logger 中介绍的一样，但是通过这个 logger 对对象打印的日志，会在日志前面加上打印该日志的文件路径，方便快速定位日志打印位置。

###### 自定义 Controller 基类

按照类的方式编写 Controller，不仅可以让我们更好的对 Controller 层代码进行抽象（例如讲一些统一的处理抽象成一些私有方法），还可以通过自定义 Controller 基类的方式封装应用中常用的方法。

```js
// app/core/base_controller.js
const { Controller } = require('egg');

class BaseController extends Controller {
  get user() {
    return this.ctx.session.user;
  }

  success(data) {
    this.ctx.body = {
      success: true,
      data,
    };
  }

  nodeFound(msg) {
    msg = msg || 'not Found';
    this.ctx.throw(404, msg);
  }
}

module.exports = BaseController;
```

此时在编写应用时，可以继承 BaseController,直接使用基类上的方法：

```js
// app/controller/post.js
const Controller = require('../core/base_controller.js');
class PostController extends Controller {
  async list() {
    const posts = await this.service.listByUser(this.user);
    this.success(posts);
  }
}
```

##### Controller 方法

该方案不推荐使用，只是为了兼容

每一个 Controller 都是一个 async function，它的入参为请求的上下文 Context 对象的实例，通过它我们可以拿到框架封装好的各种便捷的属性和方法。

例如我们写一个应用到`POST /api/posts`接口的 Controller，我们会在`app/controller`目录下创建一个`post.js`文件：

```js
// app/controller/post.js
exports.create = async ctx => {
  const createRule = {
    title: { type: 'string' },
    content: { type: 'string' },
  };

  // 校验参数
  ctx.validate(createRule);
  // 组装参数
  const author = ctx.session.userId;
  const req = Object.assign(ctx.request.body, { author });
  // 调用 service 进行业务处理
  const res = awati ctx.service.post.create(req);
  // 设置响应内容和响应状态码
  ctx.body = { id: res.id };
  ctx.status = 201;
};

```

在上面的例子中我们引入了许多新概念，但是还是比较直观，容易理解，我们会在下面对它们进行更详细的介绍。

#### HTTP 基础

由于 Controller 基本上时业务开发中唯一和 HTTP 协议打交道的地方，在继续往下了解之前，我们首先简单的看一下 HTTP 协议时怎样的。

如果我们发起一个 HTTP 请求来访问前面例子中提到的 Controller：

```sh
curl -X POST http://localhost:3000/api/posts --data '{"title": "controller", "content": "what is controller"}' --header 'Content-Type: application/json; charset=UTF-8'
```

通过 curl 发起的 HTTP 请求的内容就会是下面这样的：

```sh
POST /api/posts HTTP/1.1
Host: localhost:3000
Content-Type: application/json; charset=UTF-8

{"title": "controller", "content": "what is controller"}
```

请求的第一行包含了三个信息，我们比较常用的是前两个：

- method：这个请求中的 method 的值是`POST`。
- path：值为`/api/posts`，如果用户的请求中包含 query，也会在这里出现。

从第二行开始知道遇到一个空行的位置，都是请求的 Headers 部分，这一部分带有许多常用属性，包括这里能看到的 Host，Content-Type，还有 Cookie，User-Agent 等等。在这个请求头中有两个头：

- Host：我们在浏览器发起请求的时候，域名会用来通过 DNS 解析找到服务的 IP 地址，但是浏览器也会将域名和端口号放在 Host 头中以并发给服务端。
- Content-Type：当我们的请求有 body 的时候，都会有 Content-Type 来标明我们的请求体是什么格式的。

之后的内容全部都是请求的 body，当请求是 POST，PUT，DELETE 等方法的时候，可以带上请求体，服务端会根据 Content-Type 来解析请求体。

在服务端处理完这些请求后，会发送一个 HTTP 响应给客户端。

```sh
HTTP/1.1 201 Created
Content-Type: application/json; charset=utf-8
Content-Length: 8
Date: Mon, 09 Jan 2017 08:40:28 GMT
Connection: keep-alive

{"id": 1}
```

第一行中也包含了三段，我们最常用的主要是响应状态码，这个例子中它的值是 201，它的含义是在服务端成功创建了一条资源。

和请求一样，从第二行开始到下一个空行之间就是响应头，这里的 Content-Type，Content-Length 表示这个响应的格式是 JSON，长度为 8 个字节。

最后剩下的部分就是这次响应的真正内容。

#### 获取 HTTP 请求参数

从上面的 HTTP 请求示例中可以看到，有很多地方可以放用户请求数据，框架通过在 Controller 上绑定的 Context 示例，提供了许多便捷方法和属性获取用户通过 HTTP 请求发送过来的参数。

##### query

在 URL 中的`?`后面的部分就是一个 Query String，这一部分经常用于 GET 类型的请求中传递参数。例如`GET /posts?category=egg&language=node`中，`category=egg&language=node`就是用户传递过来的参数。我们可以通过`ctx.query`拿到解析过后的参数体：

```js
class PostController extends Controller {
  async listPosts() {
    const query = this.ctx.query;
    /**
     * {
     *   category: 'egg',
     *   language: 'node',
     * }
     */
  }
}
```

当 Query String 中的 key 重复时，`ctx.query`只取 key 第一次出现时的值，后面再出现的时候都会被忽略。`GET /posts?category=egg&category=koa`通过`ctx.query`拿到的值是`{category: 'egg'}`。

这样处理的原因是为了保持统一性，由于通常情况下我们不会设计让用户传递 key 相同的 Query String，所以我们经常会写类似下面的代码：

```js
const key = ctx.querykey || '';
if (key.startsWith('egg')) {
  // do something
}
```

而如果有人故意发起请求在 Query String 中带上重复的 key 来请求时，就会引发系统异常。因此框架保证了从`ctx.query`上获取的参数一旦存在，一定是字符串类型。

###### queries

又是我们系统会设计成让用户传递相同的 key，例如`GET /posts?category=egg&id=1&id=2&id=3`。针对此类情况，框架提供了`ctx.queries`对象，这个对象也解析了 Query String，但是它不会丢弃任何一个重复的数据，而是将他们都放到一个数组中：

```js
// GET posts?category=egg&id=1&id=2&id=3
class PostController extends Controller {
  async listPosts() {
    console.log(this.ctx.queries);
    /**
     * {
     *   category: ['egg'],
     *   id: ['1', '2', '3'],
     * }
     */
  }
}
```

`ctx.queries`上所有的 key 如果有值，也一定会是数组类型。

##### Router params

在 Router 中，我们介绍了 Router 上也可以申明参数，这些参数都可以通过`ctx.params`获取到。

```js
// app.get('/projects/:projectId/app/:appId', 'app.listApp');
// GET /projects/1/app/2
class AppController extends Controller {
  async listApp() {
    assert.equal(this.ctx.params.projectId, '1');
    assert.equal(this.ctx.params.appId, '2');
  }
}
```

##### body

虽然我们可以通过 URL 传递参数，但是还是有诸多限制：

- 浏览器中会对 URL 的长度有所限制，如果需要传递的参数过多就会无法传递。
- 服务端经常会将访问的完整 URL 记录到日志文件中，有一些敏感数据通过 URL 传递会不安全。

在前面的 HTTP 请求报文实例中，我们看到在 header 之后还会有一个 body 部分，我们通常会在这个部分传递 POST、PUT 和 DELETE 等方法的参数。一般请求中有 body 的时候，客户端（浏览器）会同时发送`Content-Type`告诉服务端这次请求的 body 是什么格式的。Web 开发中数据传递最常用的两类格式分别是 JSON 和 Form。

框架内置了 bodyParser 中间件来对这两类格式的请求 body 解析成 object 挂载到`ctx.request.body`上。HTTP 协议中并不建议在通过 GET、HEAD 方法访问时传递 body，所以我们无法在 GET、HEAD 方法中按照此方法获取到内容。

```js
// POST /api/posts HTTP/1.1
// Host: localhost:3000
// Content-Type: application/json; charset=UTF-8
//
// {"title": "controller", "content": "what is controller"}

class PostController extends Controller {
  async listPosts() {
    assert.equal(this.ctx.request.body.title, 'controller');
    assert.equal(this.ctx.request.body.content, 'what is controller');
  }
}
```

框架对 bodyParser 设置了一些默认参数，配置好之后拥有以下特性：

- 当请求的 Content-Type 为`application/json`，`application/json-patch+json`，`application/vnd.api+json`和`application/csp-report`时，会按照 json 格式对请求 body 进行解析，并限制 body 最大长度为`100kb`。

- 当请求的 Content-Type 为`application/x-www-form-urlencoded`时，会按照 form 格式对请求 body 进行解析，并限制 body 最大长度为`100kb`。

- 如果解析成功，body 一定会是一个 Object（可能是一个数组）。

一般来说我们最经常调整的配置项就是变更解析式允许的最大长度，可以在`config/config.default.js`中覆盖框架的默认值：

```js
module.exports = {
  bodyParser: {
    jsonLimit: '1mb',
    formLimit: '1mb',
  },
};
```

如果用户的请求 body 超过了我们配置的解析最大长度，会抛出一个状态码`413`的异常，如果用户请求的 body 解析失败（错误的 JSON），会抛出一个状态码为`400`的异常。

**注意：在调整 bodyParser 支持的 body 长度时，如果我们应用当前还有一层反向代理（Nginx），可能也需要调整它的配置，确保反向代理也支持同样长度的 body。**

一个常见的错误是把`ctx.request.body`和`ctx.body`混淆，后者其实是`ctx.response.body`的简写。

##### 获取上传的文件

请求 body 除了可以带参数之外，还可以发送文件，一般来说，浏览器上都是通过`Multipart/form-data`格式发送文件，框架通过内置 Multipart 插件来支持获取用户上传的文件，我们为你提供了两种方式：

- **File 模式**：

  如果你完全不知道 Node 中的 Stream 用法，那么 File 模式非常适合你：

  1）在 config 文件中启用`file`模式：

  ```js
  // config/config.default.js
  exports.multipart = {
    mode: 'file',
  };
  ```

  2. 上传/接收文件：

  1) 上传/接收单个文件：

  你的前端静态页代码应该看上去如下样子：

  ```html
  <form method="post" action="/upload?_csrf={{ ctx.csrf | safe }}" enctype="multipar/form-data">
    title: <input type="text" name="title" /> file: <input type="file" name="file" />
    <button type="submit">Upload</button>
  </form>
  ```

  对应的后端代码如下：

  ```js
  // app/controller/upload.js
  const Controller = require('egg').Controller;
  const fs = requuire('mz/fs');

  module.exports = class extends Controller {
    async upload() {
      conft { ctx } = this;
      const file = ctx.request.files[0];
      const name = 'egg-multipart-test/' + path.basename(file.filename);

      let result;

      try {
        // 处理文件，比如上传到云端
        result = await ctx.oss.put(name, file.filepath);
      } finally {
        // 需要删除临时文件
        await fs.unlin(file.filepath);
      }

      ctx.body = {
        url: result.url,
        // 获取所有的字段值
        requestBody: ctx.request.body,
      };
    }
  };
  ```

  2. 上传/接收多个文件：

  对于多个文件，我们借助`ctx.request.files`属性进行遍历，然后分别进行处理：

  前端静态页代码：

  ```html
  <form method="POST" action="/upload?_csrf={{ ctx.csrf | safe }}" enctype="multipart/form-data">
    title: <input name="title" /> file1: <input name="file1" type="file" /> file2:
    <input name="file2" type="file" />
    <button type="submit">Upload</button>
  </form>
  ```

  对应的后端代码：

  ```js
  // app/controller/upload.js
  const Controller = require('egg').Controller;
  const fs = require('mz/fs');

  module.exports = class extends Controller {
    async upload() {
      const { ctx } = this;
      console.log(ctx.request.body);
      console.log('got %d files', ctx.request.files.length);

      for (const file of ctx.request.files) {
        console.log('field: ' + file.filename);
        console.log('filename: ' + file.filename);
        console.log('encoding: ' + file.encoding);
        console.log('mime: ' + file.mime);
        console.log('tmp filepath: ' + file.filepath);

        let result;
        try {
          // 处理文件，上传云端
          result = await ctx.oss.put('egg-multipart-test/' + file.filename, file.filepath);
        } finally {
          // 需要删除临时文件
          await fs.unlink(file.filepath);
        }

        console.log(result);
      }
    }
  };
  ```

- Stream 模式：

  如果你对于 Node 的 Stream 农事非常熟悉，那么你可以选择此模式。在 Controller 中，我们可以通过`ctx.getFileStream()`接口能获取到上传的文件流。

  1. 上传/接收单个文件：

  ```js
  const path = require('path');
  const sendToWormhole = requier('stream-wormhole');
  const Controller = require('egg').Controller;

  class UploaderController extends Controller {
    aysnc upload() {
      const ctx = this.ctx;
      const stream = await ctx.getFileStream();
      const name = 'egg-multipart-test/' + path.basename(stream.filename);

      // 文件处理，上传到云存储等等
      let result;
      try {
        result = await ctx.oss.put(name, stream);
      } catch (err) {
        // 必须将上传的文件流消费掉，要不然浏览器会卡死
        await sendToWormhole(stream);
        throw err;
      }

      ctx.body = {
        url: result.url,
        // 所有表单字段都能通过`stream.fields`获取到
        fields: stream.fields,
      };
    }
  }

  module.exports = UploaderController;
  ```

  要通过`ctx.getFileStream`便捷的获取到用户上传的文件，需要满足两个条件：

  - 只支持上传一个文件。
  - 上传文件必须在所有其它的 fields 后面，否则在拿到文件流时可能还获取不到 fields。

2. 上传/接收毒功而文件：

如果要获取同时上传的多个文件，不能通过`ctx.getFileStream()`来获取，只能通过下面这种方式：

```js
const sendToWormhole = require('stream-wormhole');
const Controller = require('egg').Controller;

class UploaderController extends Controller {
  async upload() {
    const ctx = this.ctx;
    const parts = ctx.multipart();
    let part;
    // parts() 返回promise对象
    while ((part = await parts()) != null) {
      if (part.length) {
        // 这是 busboy的字段
        console.log('field: ' + part[0]);
        console.log('value: ' + part[1]);
        console.log('valueTruncated: ' + part[2]);
        console.log('fieldnameTruncated: ' + part[3]);
      } else {
        if (!part.filename) {
          // 这时是用户没有选择文件酒店及上传（part 是file stream ，但是 part.filename为空）
          // 需要做出处理，例如给出错误提示消息
          return;
        }

        // part 是上传的文件流
        console.log('field: ' + part.filedname);
        console.log('filename: ' + part.filename);
        console.log('encoding: ' + part.encoding);
        console.log('mime: ' + part.mime);

        // 文件处理，上传云端
        let result;
        try {
          result = await ctx.oss.put('egg-multipart-test/' + part.filename, part);
        } catch (err) {
          // 必须将上传的文件流消费掉，要不然浏览器响应会卡死
          await sendToWormhole(part);
          throw err;
        }

        console.log(result);
      }
    }
    console.log('and we are done parsing the form!');
  }
}

module.exports = UploaderController;
```

为了保证上传的安全，框架限制了支持的文件格式，框架默认支持白名单如下：

```js
//images
'.jpg', '.jpeg', // image/jpeg
'.png', // image/png, image/x-png
'.gif', // image/gif
'.bmp', // image/bmp
'.wbmp', // image/vnd.wap.wbmp
'.webp',
'.tif',
'.psd',

// text
'.svg',
'.js', '.jsx',
'.json',
'.css', '.less',
'.html', '.htm',
'.xml',

// tar
'.zip',
'.gz', '.tgz', '.gzip',

// video
'.mp3',
'.mp4',
'.avi',
```

用户可以通过在`config/config.default.js`中配置来新增支持的文件名，或者重写整个白名单。

- 新增支持的文件扩展名

```js
module.exports = {
  multipart: {
    fileExtensions: ['.apk'], // 增加对apk扩展名的支持
  },
};
```

- 覆盖整个白名单

```js
module.export = {
  multipart: {
    whitelist: ['.png'], // 覆盖整个白名单，只允许上传'.png'格式
  },
};
```

**注意：当重写了 whitelist 时，fileExtensions 不生效。**

欲了解更多相关此技术的细节和详情，请参阅 Egg-Multipart。

##### header

除了从 URL 和请求 body 上获取参数之外，还有许多参数时通过请求 header 传递的。框架提供了一些辅助属性和方法来获取。

- `ctx.header`，`ctx.header`，`ctx.request.headers`，`ctx.request.header`：这几个方法是等价的，都是获取整个 header 对象。

- `ctx.get(name)`，`ctx.request.get(name)`：获取请求 header 的一个字段值，如果这个字段不存在，则返回空字符串。

- 我们建议用`ctx.get(name)`而不是`ctx.headers['name']`，因为前者会自动处理大小写。

由于 header 比较特殊，有一些 HTTP 协议规定了具体的含义（例如`Content-Type`，`Accept`），有些是反向代理设置的，已经约定俗成（X-Forwarded-For），框架也会对他们增加一些 getter，详细的 getter 可以查看 API 文档。

特别是如果我们通过`config.proxy = true`设置了应用部署在反向代理（Nginx）之后，有一些 Getter 的内部处理会发生改变。

`ctx.host`，会优先读取`config.hostHeaders`中配置的 header 的值，读不到时在尝试获取 host 这个 header 的值，如果都获取不到，则返回空字符串。`config.hostHeaders`默认配置为`x-forwarded-host`。

`ctx.protocol`，通过这个 Getter 获取 protocol 时，首先会判断当前连接是否是加密连接，如果是加密连接，则返回 https。

如果处于非加密连接时，优先读通过`config.protocolHeaders`中配置的 header 的值来判断是 HTTP 还是 HTTPS，如果读取不到，我们可以在配置中通过`config.protocol`类设置兜底值，默认为 HTTP。`config.protocolHeaders`默认配置为`x-forwarded-proto`。

`ctx.ips`，通过`ctx.ips`获取请求通过所有的中间设备 IP 地址列表，只有在`config.proxy = true`时，才会通过读取`config.ipHeaders`中配置的 header 的值来获取，获取不到时为空数组。`conifg.ipHeaders`默认配置为`x-forwarded-for`。

`ctx.ip`，通过`ctx.ip`获取请求发起方的 IP 地址，优先从`ctx.ips`中获取，`ctx.ips`为空时使用连接上发起方的 IP 地址。

**注意：ip 和 ips 不同，ip 当`config.proxy = false`时会返回当前连接发起者的 IP 地址，ips 此时会为空数组。**

##### Cookie

HTTP 请求都是无状态的，但是我们的 Web 应用通常都需要知道发起请求的人是谁。为了解决这个问题，HTTP 协议设计了一个特殊的请求头：Cookie。服务端可以通过响应头（set-cookie）将少量数据响应给客户端，浏览器会遵循协议将数据保存，并在下一次请求同一个服务器的时候带上（浏览器也会遵循协议，只在访问符合 Cookie 指定规则的网站时带上对应的 Cookie 来保证安全性）。

通过`ctx.cookies`，我们可以在 Controller 中便捷、安全的设置和读取 Cookie。

```js
class CookieController extends Controller {
  async add() {
    const ctx = this.ctx;
    let count = ctx.cookies.get('count');
    count = count ? Number(count) : 0;
    ctx.cookies.set('count', ++count);
    ctx.body = count;
  }

  async remove() {
    const ctx = this.ctx;
    const count = ctx.cookies.set('count', null);
    ctx.status = 204;
  }
}
module.exports = CookieController;
```

Cookie 虽然在 HTTP 中只有一个头，但是通过`foo=bar;foo1=bar1;`的格式可以设置多个键值对。

Cookie 在 Web 应用中经常承担了传递客户端身份信息的作用，因此有许多安全相关的配置，不可忽视，Cookie 文档中详细介绍了 Cookie 的用法和安全相关的配置项，可以深入阅读了解。

###### 配置

对于 Cookie 来说，主要有以下几个属性可以在`config.default.js`中进行配置：

```js
module.exports = {
  cookies: {
    // httpOnly: true | false,
    // sameSite: 'none | lax | strict'
  },
};
```

举例：配置应用界别的 Cookie SameSite 属性等于 Lax。

```js
module.exports = {
  cookies: {
    sameSite: 'lax',
  },
};
```

##### Session

通过 Cookie，我们可以给每个用户设置一个 Session，用来存储用户身份相关的信息，这份信息会加密后存储在 Cookie 中，实现跨请求的用户身份保持。

框架内置了 Session 插件，给我们提供了`ctx.session`来访问或者修改当前用户 Session。

```js
class PostController extends Controller {
  async fetchPosts() {
    const ctx = this.ctx;
    // 获取Session上的内容
    const userId = ctx.session.userId;
    const posts = await ctx.service.post.fetch(userId);
    // 修改Session的值
    ctx.session.visited = ctx.session.visited ? ++ctx.session.visited : 1;
    ctx.body = {
      success: true,
      posts,
    };
  }
}

module.exports = PostController;
```

Session 的使用方法非常之高管，直接读取它或者修改它就可以了，如果删除它，直接将它赋值为 null：

```js
class SessionController extends Controller {
  async deleteSession() {
    this.ctx.session = null;
  }
}
module.exports = SessionController;
```

和 Cookie 一样，Session 也有许多安全等选项和功能，在使用之前最好阅读 Session 文档深入了解。

###### 配置

对于 Session 来说，主要有下面几个属性可以在`config.default.js`中进行配置：

```js
module.exports = {
  key: 'EGG_SESS', // 继承Session的Cookie键值对名字
  maxAge: 86400000， // Session的最大有时间
}
```

#### 参数校验

在获取到用户请求的参数后，不可避免地要对参数进行一些校验。

借助 Validate 插件提供便捷地参数校验机制，帮助我们完成各种复杂的参数校验。

```js
// config/plugin.js
export.validate = {
  enable: true,
  package: 'egg-validate',
};
```

通过`ctx.validate(rule[, body])`直接对参数进行校验：

```js
class PostController extends Controller {
  async create() {
    // 校验参数
    // 如果不传第二个参数会自动校验`ctx.request.body`
    this.ctx.validate({
      title: { type: 'string' },
      content: { type: 'string' },
    });
  }
}
```

当校验异常时，会直接抛出一个异常，异常状态码为 422,errors 字段包含了详细的验证不通过信息。如果想要自己处理异常，可以通过`try catch`来自行捕获。

```js
class PostController extends Controller {
  async create() {
    const ctx = this.ctx;
    try {
      ctx.validate(createRule);
    } catch (err) {
      ctx.logger.warn(err.errors);
      ctx.body = { success: false };
      return;
    }
  }
}
```

##### 校验规则

参数校验通过 Parameter 完成，支持的校验规则可以在该模块的文档中查阅到。

###### 自定义校验规则

除了上一节介绍的内置校验类型外，有时候我们希望自定义一些校验规则，让开发时更便捷，此时可以通过`app.validator.addRule(type, check)`的方式新增自定义规则。

```js
// app.js
app.validator.addRule('json', (rule, value) => {
  try {
    JSON.parse(value);
  } catch (err) {
    return 'must be json string';
  }
});
```

添加完自定义规则后，就可以在 Controller 中直接使用这条规则来进行参数校验了。

```js
class PostController extends Controller {
  async handler() {
    const ctx = this.ctx;
    // query.test 字段必须时json字符串
    const rule = { test: 'json' };
    ctx.validate(rule, ctx.query);
  }
}
```

#### 调用 Service

我们并不像在 Controller 中实现太多业务逻辑，所以提供了一个 Service 层进行业务逻辑的封装，这不仅能提高代码的复用性，同时还可以让我么业务逻辑更好测试。

在 Controller 中可以调用任何一个 Service 上的任何方法，同时 Service 是懒加载的，只有当访问它的时候框架才会实例化它。

```js
class PostController extends Controller {
  async create() {
    const ctx = this.ctx;
    const author = ctx.session.userId;
    const req = Object.assign(ctx.request.body, { author });
    // 调用service进行业务处理
    const res = await ctx.service.post.create(req);
    ctx.body = { id: res.id };
    ctx.status = 201;
  }
}
```

Service 的具体写法，请查看 Service 章节。

#### 发送 HTTP 响应

当业务逻辑完成之后，Controller 的最后一个职责就是将业务逻辑的处理通过 HTTP 响应发给用户。

##### 设置 status

HTTP 设计了非常多的状态码，每一个状态码代表了一个特定的含义，通过设置正确的状态码，可以让响应更符合语义。

框架提供了一个便捷的 Setter 来进行状态码的设置：

```js
class PostController extends Controller {
  async create() {
    // 设置状态码为201
    this.ctx.status = 201;
  }
}
```

具体什么场景设置什么样的状态码，可以参考各个状态码的含义。

##### 设置 body

绝大多数的数据都是通过 body 发送给请求访的，和请求中的 body 一样，在响应中发送的 body，也需要有配套的 Content-Type 告知客户端如何对数据进行解析。

- 作为一个 RESTful 的 API 接口 controller，我们通常会返回 Content-Type 为`application/json`格式的 body，内容是一个 JSON 字符串。

- 作为一个 html 页面的 controller，我们通常会返回 Content-Type 为`text/html`格式的 body，内容是 html 代码段。

**注意：`ctx.body`是`ctx.response.body`的简写，不要和`ctx.request.body`混淆了。**

```js
class ViewController extends Controller {
  async show() {
    this.ctx.body = {
      name: 'egg',
      category: 'framework',
      language: 'Node.js',
    };
  }

  async page() {
    this.ctx.body = '<html><h1>Hello</h1></html>';
  }
}
```

由于 Node 的流式特性，我们还有很多场景需要通过 Stream 返回响应，例如一个大文件，代理服务器直接返回上游的内容，框架也支持直接将 body 设置成一个 Stream，并会同时处理好这个 Stream 上的错误时间。

```js
class ProxyController extends Controller {
  async proxy() {
    const ctx = this.ctx;
    const result = await ctx.curl(url, {
      streaming: true,
    });
    ctx.set(result.header);
    // reslut.res 是一个stream
    ctx.body = result.res;
  }
}
```

###### 渲染模板

通常来说，我们不会手写 HTML 页面，而是会通过模板引擎进行生成。框架自身没有集成任何一个模板引擎，但是约定了 View 插件的规范，通过接入的模板引擎，可以直接使用`ctx.render(template)`来渲染模板生成 html。

```js
class HomeController extends Controller {
  async index() {
    const ctx = this.ctx;
    await ctx.render('home.tpl', { name: 'egg' });
  }
}
```

具体示例可以查看模板渲染。

###### JSONP

有时我们需要给非本域的页面提供接口服务，又由于一些历史原因我无法通过 CORS 实现，可以通过 JSONP 进行响应。

由于 JSONP 如果使用不当会导致非常多的安全问题，所以框架中提供了便捷的响应 JSONP 格式数据的方法，封装了 JSONP XSS 选相关的安全规范，并支持进行 CSRF 校验和 referrer 校验。

- 通过`app.jsonp()`提供的中间件来让一个 controller 支持响应 JSONP 格式的数据。在路由中，我们给需要支持 jsonp 的路由加上这个中间件：

```js
// app/router.js
module.exports = app => {
  const jsonp = app.jsonp();
  app.router.get('/api/posts/:id', jsonp, app.controller.posts.show);
  app.router.get('/api/posts', jsonp, app.controller.posts.list);
};
```

- 在 Controller 中，只需要正常编写即可：

```js
// app/controller/posts.js
class PostController extends Controller {
  async show() {
    this.ctx.body = {
      name: 'egg',
      category: 'frameworkd',
      language: 'Node.js',
    };
  }
}
```

用户请求对应的 URL 访问到这个 controller 的时候，如果 query 中又`_callback=fn`参数，将会返回 JSONP 格式的数据，否则返回 JSON 格式的数据。

- **JSONP 配置**

  框架默认通过 query 中的`_callback`参数作为识别是否返回 JSONP 格式数据的一句，并且`_callback`中设置的方法名长度最多只允许 50 个字符。应用可以在`config/config.default.js`全局覆盖默认配置：

  ```js
  // config/config.default.js
  exports.jsonp = {
    callback: 'callback', // 识别query中`callback`参数
    limit: 100, // 函数名最长为100个字符
  };
  ```

  通过上面的方式配置之后，如果用户请求`/api/posts/1?callback=fn`，响应为 JSONP 格式，如果用户请求`/api/posts/1`，响应格式为 JSON。

  我们同样可以在`app.jsonp()`创建中间件是覆盖默认的配置，以达到不同路由使用不同配置的目的：

  ```js
  // app/router.js

  module.exports = app => {
    const { router, controller, jsonp } = app;
    router.get('/api/posts/:id', jsonp({ callback: 'callback' }), controller.posts.show);
    router.get('/api/posts', jsonp({ callback: 'cb' }), controller.posts.list);
  };
  ```

- **跨站防御配置**

  默认配置下，响应 JSONP 时不会进行任何跨站攻击的防范，在某些情况下，这是很危险的。我们粗略将 JSONP 接口分为三种类型：

  1. 查询非敏感数据，例如获取一个论坛的公开文章列表。
  2. 查询敏感数据，例如获取一个用户的交易记录。
  3. 提交数据并修改数据库，例如给某一个用户创建一笔订单。

  如果我们的 JSONP 接口提供下面两类服务，在不做任何跨站防御的情况下，可能泄漏用户敏感数据甚至导致用户被钓鱼。因此框架给 JSONP 默认提供了 CSRF 校验支持和 referrer 校验支持。

- **CSRF**

  在 JSONP 中，我们只需要打开`csrf: true`，即可对 JSONP 接口开启 CSRF 校验。

  ```js
  // config/config.default.js
  module.export = {
    jsonp: {
      csrf: true,
    },
  };
  ```

  **注意：CSRF 校验依赖于 security 插件提供的基于 Cookie 的 CSRF 校验。**

  在开启 CSRF 校验时，客户端在发起 JSONP 请求时，也要带上 CSRF token，如果发起 JSONP 的请求方所在的页面和我们服务器在同一个主域名之下的话，可以读取到 Cookie 中的 CSRF token（在 CSRF token 缺失时也可以自行设置 CSRF token 到 Cookie 中），并在请求时带上该 token。

- **referrer 校验**

  如果在同一个主域下，可以通过开启 CSRF 的方式来校验 JSONP 请求的来源，而如果想对其它域名的网页提供 JSONP 服务，我们可以通过配置 referrer 白名单的方式来限制 JSONP 的请求方在可控范围内。

  ```js
  // config/config.default.js
  exports.jsonp = {
    whiteList: /^https?:\/\/test.com\//,
    // whiteList: '.test.com',
    // whiteList: 'sub.test.com',
    // whiteList: ['sub.test.com', 'sub2.test.com' ],
  };
  ```

  `whiteList`可以配置为正则表达式、字符串或者数组：

  - 正则表达式：此时只有请求的 Referrer 匹配该正则时才允许访问 JSONP 接口。在设置正则表达式的时候，注意开头的`^`以及结尾的`\/`，保证匹配到完整的域名。

  ```js
  exports.jsonp = {
    whiteList: /^https?:\/\/test.com\//,
  };
  // matches referrer:
  // https://test.com/hello
  // http://test.com/
  ```

  - 字符串：设置字符串形式的白名单时，分为两种情况，当字符串以`.`开头，例如`.test.com`时，代表 referrer 白名单为`test.com`的所有子域名，包括`test.com`自身。当字符串不以`.`开头，例如`sub.test.com`，代表 referrer 白名单为`sub.test.com`这一个域名（同时支持 HTTP 和 HTTPS）。

  ```js
  exports.jsonp = {
    whiteList: '.test.com',
  };

  // matches domain test.com
  // https://test.com/hello
  // http://test.com/
  // http://sub.test.com/

  exports.jsonp = {
    whiteList: 'sub.test.com',
  };

  // only matches domain sub.ttest.com
  // https://sub.test.com/hello
  // http://sub.test.com/
  ```

  - 数组：当设置的白名单为数组时，代表只要满足数组中任意一个元素的条件即可通过 referrer 校验。

  ```js
  exports.jsonp = {
    whiteList: ['sub.test.com', 'sub2.test.com'],
  };

  // matches domain sub.test.com and sub2.test.com
  // http://sub.test.com/hello
  // https://sub2.test.com/asdf
  ```

  当 CSRF 和 referrer 校验同时开启，请求发起方只需要满足任意一个条件即可通过 JSONP 的安全校验。

##### 设置 header

我们通过状态码标识请求是否成功、状态如何，在 body 中设置响应的内容。而通过响应的 Header，还可以设置一些扩展信息。

通过`ctx.set(key, value)`方法可以设置一个响应头，`ctx.set(headers)`设置多个 Header。

```js
// app/controller/api.js
class ProxyController extends Controller {
  async show() {
    const ctx = this.ctx;
    const start = Date.now();
    ctx.body = await ctx.service.post.get();
    const used = Date.now() - start;
    // 设置一个响应头
    ctx.set('show-response-time', used.toString());
  }
}
```

##### 重定向

框架通过 security 插件覆盖了 Koa 原生的`ctx.redirect`实现，可以提供更加安全的重定向。

- `ctx.redirect(url)` 如果不再配置的白名单域名内，则禁止跳转。
- `ctx.unsafeRedirect(url)` 不判断域名，直接跳转，一般不建议使用，明确了解可能带来的风险后使用。

如果用户使用`ctx.redirect`方法，需要在应用文件配置如下：

```js
// config/config.default.js

exports.security = {
  domainWhiteList: ['.domain.com'], // 安全白名单，以`.`开头。
};
```

若用户没有配置`domainWhiteList`或者`domainWhiteList`数组为空，则默认会对所有跳转请求放行，即等同于`ctx.unsafeRedirect(url)`。

### 服务（Service）

简单来说，Service 就是在复杂业务场景下用于业务逻辑封装的一个抽象层，提供这个抽象层有以下几个好处：

- 保持 Controller 中的逻辑更加简洁。
- 保持业务逻辑的独立性，抽象出来的 Service 可以被多个 Controller 重复调用。
- 将逻辑和展现分离，更容易编写测试用例。

#### 使用场景

- 复杂数据的处理，比如要展现的信息需要从数据库获取，还要经过一定的规则计算，才能返回用户显示。或者计算完成后，更新到数据库。

- 第三方服务的调用，比如 GitHub 信息获取等。

#### 定义 Service

```js
// app/service/user.js
const Service = require('egg').Service;

class UserService extends Service {
  async find(uid) {
    const user = await this.ctx.db.query('select * from user where uid = ?', uid);
    return user;
  }
}

module.exports = UserService;
```

##### 属性

每一次用户请求，框架都会实例化对应的 Service 实例，由于它继承了`egg.Service`，故拥有下列属性方便我们进行开发：

- `this.ctx`：当前请求的上下文 Context 对象的实例，通过它我们可以看到框架封装好的处理当前请求的各种便携属性和方法。

- `this.app`：当前应用 Application 对象的实例，通过它我们可以拿到框架提供的全局对象和方法。

- `this.service`：应用定义的 Service，通过它可以访问其它业务层，等价于`this.ctx.service`。
- `this.config`：应用运行时的配置项。
- `this.logger`：logger 对象，上面有四个方法（`debug`，`info`，`warn`，`error`），分别代表打印四个不同登记的日志，使用方法和效果与 context logger 中介绍一样，但是通过这个 logger 对象记录的日志，在日之前会加上打印该日志的文件路径，以便快速定位日志打印位置。

##### Service ctx 详解

为了可以获取用户请求的链路，我们在 Service 初始化中，注入了请求上下文，用户在方法中可以直接通过`this.ctx`来获取上下文相关信息。关于上下文的具体详解可以查看 Context，有了 ctx 我们可以拿到框架给我们封装的各种便捷属性和方法。比如我们可以使用：

- `this.ctx.curl`发起网络调用。
- `this.ctx.service.otherService`调用其它 Service。
- `this.ctx.db`发起数据库调用等，db 可能是其它插件提前挂载到 app 上的模块。

##### 注意事项

- Service 文件必须放在`app/service`目录，可以支持多级目录，访问的时候可以通过目录名级联访问。

```js
app/service/biz/user.js => ctx.service.biz.user
app/service/sync_user.js => ctx.service.syncUser
app/service/HakerNews.js => ctx.service.hackerNews
```

- 一个 Service 文件只能包含一个类，这个类需要通过`module.exports`的方式返回。
- Service 需要通过 class 的方式定义，父类必须是`egg.Service`。
- Service 不是单例，是请求级别的对象，在框架每次请求中首次访问`ctx.service.xx`时延迟实例化，所以 Service 中可以通过`this.ctx`获取当前请求的上下文。

#### 使用 Service

下面就通过一个完整的例子，看看怎么使用 Service。

```js
// app/router.js
module.exports = app => {
  app.router.get('/user/:id', app.controller.user.info);
};

// app/controller/user.js
const Controller = require('egg').Controller;
class UserController extends Controller {
  async info() {
    const { ctx } = this;
    const userId = ctx.params.id;
    const userInfo = await ctx.service.user.find(userId);
    ctx.body = userInfo;
  }
}
module.exports = UserController;

// app/service/user.js
const Service = require('egg').Service;
class UserService extends Service {
  // 默认不需要提供构造函数
  /**
   * constructor(ctx) {
   *   super(ctx); // 如果需要在构造函数中做一些处理，一定要有这句话，才能保证`this.ctx`的使用
   *   // 这样就可以直接通过this.ctx 获取 ctx了。
   *   // 还可以通过this.app 获取 app
   * }
   */

  async find(uid) {
    // 假如我们拿到用户id从数据库获取用户详细信息
    const user = await this.ctx.db.query('select * from user where uid = ?', uid);
    // 假定这里还要一些复杂的计算，然后返回需要的信息。
    const picture = await this.getPictrue(uid);

    return {
      name: user.user_name,
      age: user.age,
      picture,
    };
  }

  async getPicture(uid) {
    const result = await this.ctx.curl(`http://photoserver/uid=${uid}`, { dataType: 'json' });
    return result.data;
  }
}

module.exports = UserService;

// curl http://127.0.0.1:7001/user/1234
```

### 插件

插件机制时我们框架的一大特色。它不但可以保证框架核心的足够精简、稳定、高效，还可以促进业务逻辑复用，生态圈的形成。有人可能会问了：

- Koa 已经有了中间件机制，为什么还要插件呢？
- 中间件、插件、应用他们之间时什么关系，有什么区别？
- 我该怎么使用一个插件？
- 如何编写一个插件？
- ...

接下来我们就逐一讨论。

#### 为什么要插件

我们在使用 Koa 中间件过程中发现了下面一些问题：

1. 中间件加载其实是有先后顺序的，但是中间件自身却无法管理这种顺序，只能交给使用者。这样其实非常不友好，一旦顺序不对，可能会有天壤之别。
2. 中间件的定位时拦截用户请求，并在前后做一些事情，例如：鉴权、安全检查、访问日志等等。实际情况是，有些功能是和请求无关的，例如：定时任务、消息订阅、后台逻辑等等。
3. 有些功能包含非常复杂的初始化逻辑，需要在应用启动的时候完成。这显然也不适合放到中间件中去实现。

综上所述，我们需要一套更加强大的机制，来管理、编排哪些相对独立的业务逻辑。

##### 中间件、插件、应用的关系

一个插件其实就是一个【迷你应用】，和应用（app）几乎是一样：

- 它包含了 Service、中间件、配置、框架扩展等等。
- 它没有独立的 Router 和 Controller。
- 它没有`plugin.js`，只能声明跟其它插件的依赖，而不能决定其它插件的开启与否。

他们的关系是：

- 应用可以直接引入 Koa 的中间件。
- 当遇到上一节提到的场景时，则应用需要引入插件。
- 插件本身可以包含中间件。
- 多个插件可以包装为一个上层框架。

#### 使用插件

插件一般通过 npm 模块的方式进行复用：

```sh
npm i egg-mysql --save
```

然后需要再应用或框架的`config/plugin.js`中声明：

```js
// config/plugin.js
// 使用mysql 插件
exports.mysql = {
  enable: true,
  package: 'egg-mysql',
};
```

就可以直接使用插件提供的功能：

```js
app.mysql.query(sql, values);
```

##### 参数介绍

`plugin.js`中的每个配置项支持：

- `{Boolean} enable` - 是否开启插件，默认为 true
- `{String} package` - `npm`模块名称，通过`npm`模块形式引入插件
- `{String} path` - 插件绝对路径，跟 package 配置互斥
- `{Array} env` - 只有再指定运行环境才能开启，会覆盖插件自身`package.json`中的配置

##### 开启和关闭

再上层框架内部内置的插件，应用再使用时就不用配置 package 或者 path，只需要指定 enable 与否：

```js
// 对于内置插件，可以用下面的间接方式开启或关闭
exports.onerror = false;
```

##### 根据环境配置

同时，我们还支持`plugin.{env}.js`这种模式，会根据运行环境加载插件配置。

比如定义了一个开发环境使用的插件`egg-dev`，只希望再本地环境加载，可以安装到`devDependencies`。

```json
// npm i egg-dev --save-dev
// package.json

{
  "devDependencies": {
    "egg-dev": "*"
  }
}
```

然后在`plugin.local.js`中声明：

```js
// config/plugin.local.js
exports.dev = {
  enable: true,
  package: 'egg-deg',
};
```

这样在生产环境中可以`npm i --production`不需要下载`egg-dev`的包了。

> 注意：
>
> - 不存在`plugin.default.js`。
> - 只能在应用层使用，在框架层请勿使用。

##### package 和 path

- `package`是`npm`方式引入，也是最常见的引入方式。
- `path`是绝对路径引入，如应用内部抽了一个插件，但是还没有达到开源发布独立`npm`的阶段，或者是应用自己覆盖了框架的一些插件。
- 关于这两种方式的使用场景，可以参考渐进式开发。

```js
// config/plugin.js
const path = require('path');
exports.mysql = {
  enable: true,
  path: path.join(__dirname, '../lib/plugin/egg-mysql'),
};
```

#### 插件配置

插件一般会包含自己的默认配置，应用开发者可以在`config.default.js`覆盖对应的配置：

```js
// config/config.default.js

exports.mysql = {
  client: {
    host: 'mysql.com',
    port: '3306',
    user: 'test_user',
    password: 'test_password',
    database: 'test',
  },
};
```

具体合并规则可以参见配置。

#### 插件列表

- 框架默认内置了企业级应用常见的插件：
  - onerror 统一异常处理
  - Session Session 实现
  - i18n 多语言
  - watcher 文件和文件夹监控
  - multipart 文件流式上传
  - security 安全
  - development 开发环境配置
  - logrotator 日志切分
  - schedule 定时任务
  - static 静态服务器
  - jsonp jsonp 支持
  - view 模板引擎
- 更多社区插件可以在 GitHub 搜索 egg-plugin

#### 如何开发一个插件

参考文档：插件开发

### 定时任务

虽然我们通过框架开发的 HTTP Server 是请求响应模型的，但是仍然还有许多场景需要执行一些定时任务，例如：

1. 定时上报应用状态。
2. 定时从远端接口更新本地缓存。
3. 定时进行文件切割、临时文件删除。

框架提供了一套机制来让定时任务的编写和维护更加优雅。

#### 编写定时任务

所有的定时任务都统一存放在`app/schedule`目录下，每一个文件都是一个独立的定时任务，可以配置定时任务的属性和要执行的方法。

一个简单的例子，我们定义一个更新远程数据到内存缓存的定时任务，就可以在`app/schedule`目录下创建一个`update_cache.js`文件：

```js
// app/schedule/update_cache.js
const Subscription = require('egg').Subscription;

class UpdateCache extends Subscription {
  // 通过Schedule 属性来设置定时任务的执行间隔等配置

  static get schedule() {
    return {
      interval: '1m', // 1分钟间隔
      type: 'all', // 指定所有的worker都需要执行
    };
  }

  // subscribe 是真正定时任务执行时被运行的函数

  async subscrebe() {
    const res = await this.ctx.curl('http://www.api.com/cache', {
      dataType: 'json',
    });
    this.ctx.app.cache = res.data;
  }
}

module.exports = UpdateCache;
```

还可以简写为：

```js
module.exports = {
  schedule: {
    interval: '1m', // 1分钟间隔
    type: 'all', // 指定所有worker都需要执行
  },

  async task(ctx) {
    const res = await ctx.curl('http://www.api.com/cache', {
      dataType: 'json',
    });

    ctx.app.cache = res.data;
  },
};
```

这个定时任务会在每一个 Worker 进程上每一分钟执行一次，将远程数据请求回来挂载到`app.cache`上。

##### 任务

- `task`或`subscribe`同时支持`generator function`和`async function`。
- `task`的入参为`ctx`，匿名的 Context 实例，可以通过它调用`service`等。

##### 定时方式

定时任务可以指定 interval 或者 cron 两种不同的定时方式

###### interval

通过`schedule.interveal`参数来配置定时任务的执行时机，定时任务将会每间隔指定的时间执行一次。interval 可以配置成：

- 数字类型，单位为毫秒数，例如`5000`。
- 字符类型，会通过 ms 转换成 ms 数，例如`5s`。

```js
module.exports = {
  schedule: {
    // 每10秒执行一次
    interval: '10s',
  },
};
```

###### cron

通过`schedule.cron`参数来配置定时任务的执行时机，定时任务将会按照 cron 表达式在指定的事件点执行。cron 表达式通过`con-parser`进行解析。

**注意：cron-parser 支持可选的秒（linux crontab 不支持）。**

```sh
*    *    *    *    *    *
┬    ┬    ┬    ┬    ┬    ┬
│    │    │    │    │    |
│    │    │    │    │    └ day of week (0 - 7) (0 or 7 is Sun)
│    │    │    │    └───── month (1 - 12)
│    │    │    └────────── day of month (1 - 31)
│    │    └─────────────── hour (0 - 23)
│    └──────────────────── minute (0 - 59)
└───────────────────────── second (0 - 59, optional)
```

```js
module.exports = {
  schedule: {
    // 每三个小时准点执行一次
    cron: '0 0 */3 * * *',
  },
};
```

##### 类型

框架提供的定时任务默认支持两种类型，worker 和 all。worker 和 all 都支持上面的两种定时方式，只是当到执行时机时，会执行定时任务的 worker 不同：

- worker 类型：每台机器上只有一个 worker 会执行这个定时任务，每次执行定时任务的 worker 的选择是随机的。

- all 类型：每台机器上每个 worker 都会执行这个定时任务。

##### 其它参数

除了刚才介绍的几个参数外，定时任务还支持这些参数：

- `cronOptions`：配置 cron 的时区等，参见 cron-parser 文档。
- `immediate`：配置了该参数为 true 时，这个定时任务会在应用启动并 ready 后立刻执行一次这个定时任务。
- `disable`：配置该参数为 true 时，这个定时任务不会被启动。
- `env`：数组，仅在指定的环境下才启动该定时任务。

##### 执行日志

执行日志会输出到`${appInfo.root}/logs/{app_name}/egg-schedule.log`，默认不会输出到控制台，可以通过`config.customLogger.scheduleLogger`来定义。

```js
// config/config.default.js
config.customLogger = {
  scheduleLogger: {
    // consleLevel: 'NONE',
    // file: path.join(appInfo.root, 'logs', appInfo.name, 'egg-schedule.log'),
  },
};
```

##### 动态配置定时任务

有时候我们需要配置定时任务的参数。定时任务还支持另一种写法：

```js
module.exports = app => {
  return {
    schedule: {
      interval: app.config.cacheTick,
      type: 'all',
    },
    async task(ctx) {
      const res = await ctx.curl('http://www.api.com/cache', {
        contentType: 'json',
      });
      ctx.app.cache = res.data;
    },
  };
};
```

#### 手动执行定时任务

我们可以通过`app.runSchedule(schedulePath)`来运行一个定时任务。`app.runSchedule`接收一个定时任务文件路径（`app/schedule`目录下的相对路径或者完整的绝对路径），执行对应的定时任务，返回一个 Promise。

有一些场景我们可能需要手动的执行定时任务，例如：

- 通过手动执行定时任务可以更优雅的编写对定时任务的单元测试。

```js
const mm = require('egg-mock');
const assert = require('assert');

it('should schedule work fine', async () => {
  const app = mm.app();
  await app.ready();
  await app.runSchedule('update_cache');
  assert(app.cache);
});
```

- 应用启动时，手动执行定时任务进行系统初始化，等初始化完毕后再启动应用。参见应用启动自定义章节，我们可以在`app.js`中编写初始化逻辑。

```js
module.exports = app => {
  app.beforeStart(async () => {
    // 保证应用启动监听端口前数据已经准备好了
    // 后续数据的更新由定时任务自动触发
    await app.runSchedule('update_cache');
  });
};
```

#### 扩展定时任务类型

默认框架提供的定时任务只支持每台机器的单个进程执行和全部进程执行，有些情况下，我们的服务并不是单机部署的，这时候可能有一个集群的某一个进程执行一个定时任务的需求。

框架并没有直接提供此功能，但开发者可以在上层框架自行扩展新的定时任务类型。

在`agent.js`中继承`agent.ScheduleStrategy`，然后通过`agent.schedule.use()`注册即可：

```js
module.exports = agent => {
  class ClusterStrategy extends agent.ScheduleStrategy {
    start() {
      // 订阅其它的分布式调度服务发送的消息，收到消息后让一个进程执行定时任务
      // 用户在定时任务的 schedule 配置中来泪痣分布式调度的场景（scene）
      agent.mq.subscribe(schedule.scene, () => this.sendOne());
    }
  }
  agent.schedule.use('cluster', ClusterStrategy);
};
```

`ScheduleStrategy`基类提供了：

- `schedule` - 定时任务的属性，`disable`是默认统一支持的，其它配置可以自行解析。
- `this.sendOne(...args)` - 随机通知一个 worker 执行 task，`args`会传递给`subscribe(...args)`或`task(ctx, ...args)`。
- `this.sendAll(...args)` - 通知所有的 worker 执行 task。

### 框架扩展

框架提供了多种扩展点扩展自身的功能：

- Application
- Context
- Request
- Response
- Helper

在开发中，我们既可以使用已有的扩展 API 来方便开发，也可以让以上对象进行自定义扩展，进一步加强框架的功能。

#### Application

`app`对象指的是 Koa 的全局应用对象，全局只有一个，在应用启动时被创建。

##### 访问方式

- `ctx.app`
- Controller，Middleware，Helper，Service 中可以通过`this.app`访问到 Application 对象，例如`this.app.config`访问配置对象。
- 在`app.js`中`app`对象会作为第一个参数注入到入口函数中

```js
// app.js
module.exports = app => {
  // 使用 app对象
};
```

##### 扩展方式

框架会把`app/extend/application.js`中定义的对象与 Koa Application 的 prototype 对象进行合并，在应用启动时会基于扩展后的 prototype 生成`app`对象。

###### 方法扩展

例如，我们要增加一个`app.foo()`方法：

```js
// app/extend/application.js
module.exports = {
  foo(param) {
    // this就是app对象，在其中可以调用app上的其它方法，或访问属性
  },
};
```

###### 属性扩展

一般来说属性的计算只需要进行一次，那么一定要实现缓存，否则在多次访问属性时会计算多次，这样会降低应用性能。

推荐方式是使用 Symbol + Getter 的模式。

例如，增加一个`app.bar`属性 Getter：

```js
// app/extend/application.js
const BAR = Symbol('Application#bar');

module.exports = {
  get bar() {
    if (!this[BAR]) {
      // 实际情况肯定会更复杂
      this[BAR] = this.config.xx + this.config.yy;
    }
    return this[BAR];
  },
};
```

#### Context

Context 指的是 Koa 的请求上下文，这是请求级别的对象，每次请求生成一个 Context 实例，通常我们也简写成`ctx`。在所有的文档中，Context 和 ctx 都是指 Koa 的上下文对象。

##### 访问方式

- middleware 中`this`就是 ctx，例如`this.cookies.get('foo')`。
- controller 有两种写法，类的写法通过`this.ctx`，方法的写法直接通过`ctx`入参。
- helper，service 中的 this 指向 helper，service 对象本身，使用`this.ctx`访问 context 对象，例如`this.ctx.cookies.get('foo')`。

##### 扩展方式

框架会把`app/extend/context.js`中定义的对象与 Koa Context 的 prototype 对象进行合并，在处理请求时会给予扩展后的 prototype 生成 ctx 对象。

###### 方法扩展

例如，我们要增加一个`ctx.foo()`方法：

```js
// app/extend/context.js
module.exports = {
  foo(param) {
    // this就是ctx对象，在其中可以调用ctx上的其它方法或访问属性
  },
};
```

###### 属性扩展

一般来说属性的计算只需要进行一次，那么一定要实现缓存，否则在同义词请求中多次访问属性时会计算多次，这样会降低应用性能。

推荐方式是使用 Symbol + Getter 的模式。

例如，增加一个`ctx.bar`属性 Getter：

```js
// app/extend/context.js
const BAR = Symbol('Context#bar');

module.exports = {
  get bar() {
    if (!this[BAR]) {
      this[BAR] = this.get('x-bar');
    }

    return this[BAR];
  },
};
```

#### Request

Request 对象和 Koa 的 Request 对象相同，是请求级别的对象，它提供了大量请求相关的属性和方法使用。

#### 访问方式

```js
ctx.request;
```

`ctx`上很多属性和方法都被代理到`request`对象上，对于这个属性和方法使用`ctx`和使用`request`去访问他们是等价的，例如`ctx.url === ctx.request.url`。

Koa 内置的代理`request`的属性和方法列表：[Koa Request aliases](http://koajs.com/#request-aliases)。

#### 扩展方式

框架会把`app/extend/request.js`中定义的对象与内置`request`的 prototype 对象进行合并，在处理请求时会基于扩展后的 prototype 生成`request`对象。

例如，增加一个`request.foo`属性 Getter：

```js
// app/extend/request.js
module.exports = {
  get foo() {
    return this.get('x-request-foo');
  },
};
```

#### Response

Response 对象和 Koa 的 Response 对象相同，是请求级别的对象，它提供了大量响应相关的属性和方法供使用。

##### 访问方式

```js
ctx.response;
```

ctx 上的很多属性和方法都被代理到`response`对象上，对于这些属性和方法使用`ctx`和使用`response`去访问他们是等价的，例如`ctx.status = 404`和`ctx.response.status = 404`是等价的。

Koa 内置的代理`response`的属性和方法列表：[Koa Response aliases](http://koajs.com/#response-aliases)。

##### 扩展方式

框架会把`app/extend/response.js`中定义的对象与内置`response`的 prototype 对象进行合并，在处理请求时会基于扩展后的 prototype 生成`response`对象。

例如，增加一个`response.foo`属性 Setter：

```js
// app/extend/response.js
module.exports = {
  set foo(value) {
    this.set('x-response-foo', value);
  },
};
```

#### Helper

Helper 函数用来提供一些实用的 utility 函数。

它的作用在于我们可以将一些常用的动作抽离在 helper.js 里面成立一个独立的函数，这样可以用 JavaScript 来写简单的逻辑，避免逻辑分散各处。另外还有一个好处是 Helper 这样一个简单的函数，可以让我们更容易编写测试用例。

框架内置了一个常用的 Helper 函数。我们也可以编写自定义的 Helper 函数。

##### 访问方式

通过`ctx.helper`访问 helper 对象，例如：

```js
// 假设在app/router.js中定义了 home router
app.get('home', '', 'home.index');

// 使用helper计算指定 url path
ctx.helper.pathFor('home', { by: 'recent', limit: 20 });
// => /?by=recent&limit=20
```

##### 扩展方式

框架会把`app/extend/helper.js`中定义的对象与内置`helper`的 prototype 对象进行合并，在处理请求时会基于扩展后的 prototype 生成`helper`对象。

例如，增加一个`helper.foo()`方法：

```js
// app/extend/helper.js
module.exports = {
  foo(param) {
    // this 是 helper对象，在其中可以调用其它helper方法
    // this.ctx => context 对象
    // this.app => application 对象
  },
};
```

#### 按照环境进行扩展

另外，还可以根据环境进行有选择地扩展，例如，只在 unittest 环境中提供`mockXX()`方法以便进行 mock 方便测试。

```js
// app/extend/application.unittest.js
module.exports = {
  mockXX(k, v) {},
};
```

这个文件只会在 unittest 环境加载。

同理，对于 Application、Context、Request、Response、Helper 都可以使用这种方式针对某个环境进行扩展，更多请参见运行环境。

### 启动自定义

我们常常需要在应用启动期间进行一些初始化工作，等初始化完成后应用才可以启动成功，并开始堆外提供服务。

框架提供了统一地入口文件（`app.js`）进行启动过程自定义，这个文件返回一个 Boot 类，我们可以通过定义 Boot 类中的生命周期方法来执行启动应用过程中的初始化工作。

框架提供了这些生命周期函数供开发人员处理：

- 配置文件即将加载，这是最后动态修改配置的时机（`configWillLoad`）
- 配置文件加载完成（`configDidLoad`）
- 文件加载完成（`didLoad`）
- 插件启动完毕（`willReady`）
- worker 准备就绪（`didReady`）
- 应用启动完成（`serverDidReady`）
- 应用即将关闭（`beforeClose`）

我们可以在`app.js`中定义这个 Boot 类，下面我们抽取几个在应用开发中常用的生命周期函数来举例：

```js
// app.js
class AppBootHook {
  constructor(app) {
    this.app = app;
  }

  configWillLoad() {
    // 此时 config文件已经读取并合并，但是并未生效
    // 这是应用层修改配置的最后时机
    // 注意：此函数只支持同步调用

    // 例如：参数中的密码是加密的，在此处进行解密
    this.app.config.mysql.password = decrypt(this.app.config.passowrd);
    // 例如：插入一个中间件到框架的coreMiddleware之间
    const statusIdx = this.app.config.coreMiddleware.indexOf('status');
    this.app.config.coreMiddleware.splice(statusIdx + 1, 0, 'limit');
  }

  async didLoad() {
    // 所有的配置已经加载完毕
    // 可以用来加载应用来自定义文件，启动自定义的服务

    // 例如：创建自定义应用的示例
    this.app.queue = new Queue(this.app.config.queue);
    await this.app.queue.init();

    // 例如：加载自定义的目录
    this.app.loader.loadToContext(path.join(__dirname, 'app/tasks'), 'tasks', {
      fieldClass: 'taskClasses',
    });
  }

  async willReady() {
    // 所有的插件都已经启动完毕，但是应用整体还未ready
    // 可以做一些数据初始化等操作，这些操作成功才会启动应用

    // 例如：从数据库加载数据到内存缓存
    this.app.cacheData = await this.app.model.query(QUERY_CACHE_SQL);
  }

  async didReady() {
    // 应用已经启动完毕

    const ctx = await this.app.createAnonymousContext();
    await ctx.service.Biz.request();
  }

  async serverDidReady() {
    // http / https server 已启动，开始接收外部请求
    // 此时可以从app.server 拿到 server的实例

    this.app.server.on('timeout', socket => {
      // handle socket timeout
    });
  }
}

module.exports = AppBootHook;
```

**注意：生命周期函数中不建议做太耗时的操作，框架会有启动的超时检测。**

如果你的 Egg 框架生命周期函数是旧版本的，建议你升级到类方法模式，详情查看升级你的生命周期函数。

## 核心功能

### 本地开发

### 单元测试

### 应用部署

### 日志

### HttpClient

### Cookie and Session

### 多进程模型与进程间通讯

### 模板渲染

### 异常处理

### 安全

### 国际化

## 教程

### MySQL

### Sequelize

### RESTful API

### Passport 鉴权

### Soket.IO

### 静态资源

### TypeScript

### 前置代理模式

## 进阶

### 加载器（Loader）

### 插件开发

### 框架开发

### 多进程研发模式增强

### 模板插件开发规范

### 代码风格指南
