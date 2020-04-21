# 理解 Buffer

---

- [6.1 Buffer 结构](#61-buffer-结构)
- [6.2 Buffer 的转换](#62-buffer-的转换)
- [6.3 Buffer 的拼接](#63-buffer-的拼接)
- [6.4 Buffer 与性能](#64-buffer-与性能)
- [6.5 总结](#65-总结)

---

JavaScript 对于字符串（String)的操作十分友好，无论是宽字节字符串还是单字节字符串，都被认为是一个字符串。示例代码如下所示：

```js
console.log('0123456789'.length); //10
console.log('零一二三四五六七八九'.length); // 10
console.log('\u00bd'.length); // 1
```

对比 PHP 的字符串统计，我们需要动用额外的函数来获取字符串的长度：

```php
<?php
echo strlen('0123456789'); // 10
echo '\n';

echo strlen('零一二三四五六七八九'); // 30
echo '\n';

echo mb_strlen('零一二三四五六七八九','utf8'); // 10
?>
```

文件和网络 I/O 对于前端开发者而言都不是不曾有的应用场景，因为前端只需要做一些简单的字符串操作或对 DOM 操作基本就能满足业务要求，在 ECMAScript 规范中，也没有对这些地方做任何的定义，只有 CommonJS 中有部分二进制的定义。由于应用场景不同，在 Node 中，应用需要处理网络协议、操作数据库、处理图片、接受上传文件等，在网络流和文件的操作中，还需要处理大量二进制数据，JavaScript 自有的字符串远远不能满足这些需求，于是 Buffer 对象应运而生。

## 6.1 Buffer 结构

Buffer 是一个像 Array 的对象，但它主要用于操作字节。下面我们从模块结构和对象结构的层面上认识它。

### 6.1.1 模块结构

Buffer 是一个典型的 JavaScript 与 C++结合的模块，它将性能相关部分用 C++实现，将非性能相关部分用 JavaScript 实现。

第五章揭示 Buffer 所占用的内存不是通过 V8 分配的，属于堆外内存。由于 V8 垃圾回收性能的影响，将常用的操作对象用更高效和专有的内存分配回收策略来管理是个不错的思路。

由于 Buffer 太过常见，Node 在进程启动时就加载了它，并将其放在全局对象（global)上，所以在使用 Buffer 时，无需通过`require()`即可使用。

### 6.1.2 Buffer 对象

Buffer 对象类似于数组，它的元素为 16 进制的两位数，即 0 到 255 的数之。示例代码如下：

```js
var str = '深入浅出node.js';
var buf = new Buffer.from(str, 'utf8');
console.log(buf);

// => <Buffer e6 b7 b1 e5 85 a5 e6 b5 85 e5 87 ba 6e 6f 64 65 2e 6a 73>
```

不同编码的字符串占用的元素个数各不相同，上面代码中的中文字在 utf-8 编码下占用 3 个元素，字母和标点符号占用 1 个元素。

Buffer 受 Array 类型的影响很大，可以访问 length 属性得到长度，也可以通过下标访问元素，在构造对象时也十分相似：

```js
var buf = new Buffer(100);
console.log(buf.length); // => 100
```

上述代码分配了一个长 100 字节的 Buffer 对象。可以通过下标访问刚初始化的 Buffer 的元素：

```js
console.log(buf[10]);
```

我们可以通过下表对每个元素进行赋值。值得注意的是，如果如果不是 0 到 255 的整数，会不一样。

给元素的赋值如果小于 0，则将该值逐次加到 256，直到得到一个 0 到 255 之间的整数。如果得到的值岛屿 255，则逐次减 256，直到得到 0-255 区间的数值。如果是小数则舍弃小数部分，只保留整数。

### 6.1.3 Buffer 内存分配

Buffer 对象的内存分配不是在 V8 的堆内存，而是在 Node 的 C++层面实现的内存的申请。因为处理大量的字节数据不能采用需要一点内存就能向操作系统申请一点内存的方式，这可能造成大量的内存申请的系统调用，对操作系统有一定压力。为此，Node 在内存的使用上应用的是在 C++层面申请内存、在 JavaScript 中分配内存的策略。

为了高效地使用申请来地内存，Node 采用了 slab 分配机制。slab 是一种动态内存管理机制，最早诞生在于 SunOS 操作系统中，目前一些\*nix 操作系统中广泛应用。

简单而言，slab 就是一块申请好地固定大小的内存区域。slab 具有如下三种状态：

- full：完全分配状态。
- partial：部分分配状态。
- empty：没有被分配状态。

我们需要一个 Buffer 对象，可以通过以下分配方式指定大小的 Buffer 对象：

```js
new Buffer(size);
```

Node 以 8kb 为界限来区分 Buffer 是大对象还是小对象：

```js
Buffer.poolSize = 8 * 1024;
```

这个 8kb 的值也就是每个 slab 的大小值，在 JavaScript 层面，它作为单位单元进行内存的分配。

#### 1.分配小 Buffer 对象

如果指定 Buffer 的大小小于 8kb，Node 会按照小对象的方式进行分配。Buffer 的分配过程中主要使用一个局部变量 pool 作为中间处理对象，处于分配状态的 slab 单元将指向它。以下是分配一个全新的 slab 单元的操作，它会将申请的 slowBuffer 对象指向它：

```js
var pool;

function allocPool() {
  pool = new SlowBuffer(Buffer.spoolSize);
  pool.used = 0;
}
```

构造小 Buffer 对象时的代码如下：

```js
new Buffer(1024);

// 这次构造回去检查pool对象，如果pool没有被创建，将会创建一个新的slab单元指向它：
if (!pool || pool.length - pool.used < this.length) allocPool();

// 同时当前Buffer对象的parent属性指向该slab，并记录下从这个slab的哪个位置（offset)开始使用，
// slab对象自身也记录被使用了多少字节：
this.parent = pool;
this.offset = pool.used;
pool.used += this.length;
if (pool.used & 7) pool.used = (pool.used + 8) & ~7;

// 当再次创建一个Buffer对象时，构造过程中将会判断这个slab的剩余空间是否足够。如果足够，使用剩余空间，并更新slab的分配状态。

new Buffer(3000);
```

如果 slab 剩余的空间不够，将会构造新的 slab，原 slab 中剩余的空间会造成浪费。例如，第一次构造 1 字节的 Buffer 对象，第二次构造 8192 字节的 Buffer 对象，由于第二次分配时 slab 中的空间不够，所以创建并使用新的 slab，第一个 slab 的 8kb 将会被第一个 1 字节的 Buffer 对象独占。

这里要注意的事项时，由于同一个 slab 可能分配给多个 Buffer 对象使用，只有这些小 Buffer 对象在作用域释放并都可以回收时，slab 的 8kb 空间才会被回收。尽管创建了 1 个字节的 Buffer 对象，但是如果不释放它，实际可能时 8kb 的内存没有释放。

#### 2. 分配大 Buffer 对象

如果需要超过 8kb 的 Buffer 对象，将会直接分配一个 SlowBuffer 对象作为 slab 单元，这个 slab 单元将会被这个 Buffer 对象独占。

```js
// Big buffer, just alloc one
this.parent = new SlowBuffer(this.length);
this.offset = 0;
```

这里的 SlowBuffer 类时 C++中定义的，虽然引用 buffer 模块可以访问到它，但是不推荐直接操作它，而是用 Buffer 替代。

上面提到的 Buffer 对象都是 JavaScript 层面的，能够被 V8 的回收标记回收。但是其内部的 parent 属性指向的 SlowBuffer 对象确来自于 Node 自身 C++中的定义，如果是 C++层面的 Buffer 对象，所用内存不在 V8 的堆中。

#### 3. 小结

简单而言，真正的内存是在 Node 的 C++层面提供的，JavaScript 层面只是使用它。当进行小而频繁的 Buffer 操作时，采用 slab 的机制进行预先申请和事后分配，使得 JavaScript 到操作系统之间不必有过多的内存申请方面的系统调用。对于大块的 Buffer 而言，则直接使用 C++层面提供的内存，而无需细腻的分配操作。

## 6.2 Buffer 的转换

Buffer 对象可以与字符串之间相互转换。目前支持的字符串编码类型有如下这几种：

- ASCII
- UTF-8
- UTF-16LE/UCS-2
- Base64
- Binary
- Hex

### 6.2.1 字符串转 Buffer

字符串转 Buffer 对象主要是通过构造函数来完成的：

```js
new Buffer(str[, encoding]);
```

通过构造函数转换的 Buffer 对象，存储的只能是一种编码类型。encoding 参数不传递时，默认按 UTF-8 编码进行转换和存储。

一个 Buffer 对象可以存储不同编码类型的字符串转码的值，调用`write()`方法可以实现该目的，代码如下：

```js
buf.write(string[, offset, length, encoding]);
```

由于可以不断写入内容到 Buffer 对象中，并且每次写入可以指定编码，所以 Buffer 对象中的 toString()可以将 Buffer 对象转换为字符串，代码如下：

```js
buf.toString([encoding, start, end]);
```

比较精巧的是，可以设置 encoding（默认为 UTF-8）、start、end 这三个参数实现整体或局部的转换。如果 Buffer 对象由多种编码写入，就需要在局部指定不同的编码，才能转换正常的编码。

### 6.2.3 Buffer 不支持的编码类型

目前比较遗憾的是，Node 的 Buffer 对象支持的编码类型有限，只有少数的几种编码类型可以在字符串和 Buffer 之间转换。为此，Buffer 提供了一个`isEncoding()`函数来判断编码是否支持转换。

将编码类型作为参数传入上面的函数，如果支持转换返回`true`，否则返回`false`。很遗憾的是，中国常用的 GBK、GB2312 和 BIG-5 编码都不在支持的行列中。

对于不支持的比编码类型，包括 Window 124 系列、ISO-8859 系列、IBM/DOS 代码页系列、Macintosh 系列、KOI8 系列，以及 Latin1、US-ASCII,也不支持宽字节编码 GBK 和 GB2312。

icov-lite 采用纯 JavaScript 实现，icov 则通过 C++调用 libiconv 库完成。前者比后者更清凉，无需编译和处理环境依赖直接使用。在性能方面，由于转码都是耗用 CPU，在 V8 的高性能下，少了 C++到 JavaScript 层次的转换，纯 JavaScript 的性能比 C++实现得更好。

```js
var iconv = require('iconv-lite');

// Buffer 转字符串
var str = iconv.decode(buf, 'win1251');
// 字符串转Buffer
var buf = iconv.encode('Sample input string', 'win1251');
```

另外，iconv 和 iconv-lite 对无法转换的内容进行降级处理时方案不尽相同。iconv-lite 无法转换的内容如果是多字节，两个字节的?；如果是单字节，则输出?。iconv 则有三级降级策略，会尝试翻译无法转换内容，或者忽略这些内容。如果不设置忽略，iconv 对于无法转换的内容将会得到 EILSEQ 异常。如下是 iconv 的示例代码兼选项设置：

```js
var iconv = new Iconv('UTF-8', 'ASCII');
iconv.convert('ca va'); // throws EILSEQ

var iconv = new Iconv('UTF-8', 'ASCII//IGNORE');
iconv.convert('ca va'); // returns 'a va';

var iconv = new Iconv('UTF-8', 'ASCII//TRANSLIT');
iconv.convert('ca va'); // 'ca va'

var iconv = new Iconv('UTF-8', 'ASCII//IGNORE//TRANSLIT');
```

## 6.3 Buffer 的拼接

Buffer 在使用场景中，通常以一段一段的方式传输。以下是常见的从输入流中读取内容的示例代码：

```js
var fs = require('fs');

var rs = fs.createReadStream('test.md');
var data = '';
rs.on('data', function (trunk) {
  data += trunk;
});
rs.on('end', function () {
  console.log(data);
});
```

上面这段代码常见于国外，用于流读取的示范，data 事件中获取的 chunk 对象即是 Buffer 对象。对于初学者而言，容易将 Buffer 当作字符串来理解，所以在接受上面的示例时不会觉得有任何异常。

一旦输入流中有宽字节编码时，问题就会暴露出来。如果在通过 Node 开发的网站上看到乱码符号，那么该问题起源多半来自于这里。

```js
// 这里潜藏的问题在于：
data += trunk;

// 这句代码英藏了toString()操作，它等价于：
data = data.toString() + trunk.toString();
```

值得注意的是，外国人的语境通常是指英文环境，在该的场景下，这个 toString()不会造成任何问题。但是对于宽字节的中文，就会形成问题。为了重现这个问题，下面我们模拟近似场景，将文件可读流每次读取的 Buffer 长度限制为 11：

```js
var fs = fs.createReadStream('test.md', { highWaterMark: 11 });
```

搭配该代码的测试数据为李白的《静夜思》。所以将得到乱码数据。

### 6.3.1

上面诗歌中，产生这个输出结果的原因在于文件可读流在读取时会逐个读取 Buffer。由于我们限定了 Buffer 对象长度为 11，因此只读流读取 7 次才完成完整的读取，结果是以下几个 Buffer 对象依次输出：

上文提到的`buf.toString()`方法默认以 UTF-8 为编码，中文字在 UTF-8 下占用 3 个字节。所以一个 Buffer 对象在输出时，只能显示 3 个字符，Buffer 中剩下的 2 个字节将会以乱码的形式显示。于是形成了一些文字无法正常显示的问题。在这个示例中，我们构造了 11 这个限制，但是对于任意长度的 Buffer 而言，宽字节字符串都有可能被截断的情况，只不过 Buffer 的长度越大出现的概率越低而已，但该问题依然不可忽视。

### 6.3.2 setEncoding()与 string_decoder()

在看过上述的示例后，我们忘记了可读流还有一个设置编码的方法`setEncoding()`，示例：

```js
readable.setEncoding(encoding);
```

该方法的作用是让 data 事件中传递的不再是一个 Buffer 对象，而是编码后的字符串。为此，我们继续改进前面诗歌程序，添加`setEncoding()`的步骤如下：

```js
var rs = fs.createReadStream('test.md', { highWaterMark: 11 });
rs.setEncoding('utf8');
```

重新执行程序，发现输出不再受 Buffer 大小的影响了。那 Node 是如何实现的呢？要知道，无论如何设置编码，触发 data 事件的次数依旧相同，这意味着设置编码并未改变按段读取的基本方式。

事实上，在调用`setEncoding()`时，可读流对象在内部设置了一个`decoder`对象。每次 data 事件都通过该 decoder 对象进行 Buffer 到字符串的解码，然后传递给调用者。是故设置编码后，data 不再收到原始的 Buffer 对象。但是这依旧无法解释为何设置编码后乱码问题就解决掉了，因为前述分析中，无论如何转码，总是存在宽字节字符串被截断的问题。

最终乱码问题得以解决，还是在于`decoder`的神奇之处。decoder 对象来自于 string_decoder 模块 StringDecoder 的实例对象。它神奇的原理，我们以代码来说明：

```js
var StringDecoder = require('string_decoder').String.Decoder;
var decoder = new StringDecoder('utf8');

var buf1 = new Buffer([0xe5, 0xba, 0x8a, 0xe5, 0x89, 0x8d, 0xe6, 0x98, 0x8e, 0xe6, 0x9c]);
console.log(decoder.write(buf1));
// => 床前明

var buf2 = new Buffer([0x88, 0xe5, 0x85, 0x89, 0xef, 0xbc, 0x8c, 0xe7, 0x96, 0x91, 0xe6]);
console.log(decoder.write(buf2));
// => 月光，疑
```

我们将前文提到的两个 Buffer 对象写入 decoder 中。奇怪的地方在于“月”的转码并没有如平常一样在两个部分分开输出。StringDecoder 在得到编码后，知道宽字节字符串在 UTF-8 编码下是以 3 个字节的方式存储的，所以第一次`write()`时，只输出 9 个字节转码形成的字符，“月”字的前两个字节被保留在 StringDecoder 实例内部。第二次`write()`时，会将这两个剩余字节和后续 11 个字节组合在一起，再次用 3 的整数倍字节进行转码。于是乱码问题通过这种中间形式被解决了。

虽然 string_decoder 模块很奇妙，但是它也并非万能药，它目前只能处理 UTF-8、Base64 和 UCS-2/UTF-16LE 这三种编码。所以通过`setEncoding()`的方式不可否认能解决大部分乱码问题，但并不能从根本上解决该问题。

### 6.3.3 正确拼接 Buffer

淘汰掉`setEncoding()`方法后，剩下的解决方案只有将多个小 Buffer 对象拼接为一个 Buffer 对象，然后通过 iconv-lite 一类的模块来转码这种方式。`+=`的方式显然不行，那么正确的 Buffer 拼接方法应该是：

```js
var chunks = [];
var size = 0;
res.on('data', function (chunk) {
  chunks.push(chunk);
  size += chunk.length;
});

res.on('end', function () {
  var buf = Buffer.concat(chunks, size);
  var str = iconv.decode(buf, 'utf8');
  console.log(str);
});
```

正确的拼接方式时用一个数组来存储接收到的所有 Buffer 片段并记录下所有片段的总长度，然后调用`Buffer.concat()`方法生成一个合并的 Buffer 对象。`Buffer.concat()`方法封装了从小 Buffer 对象向大 Buffer 对象的赋值过程，实现十分细腻，值得围观学习：

```js
Buffer.concat = function (list, length) {
  if (!Array.isArray(list)) {
    throw new Error('Usage: Buffer.concat(list, [length])');
  }

  if (list.length === 0) {
    return new Buffer(0);
  } else if (list.length === 1) {
    return list[0];
  }

  if (typeof length !== 'number') {
    length = 0;
    for (var i = 0; i < list.length; i++) {
      var buf = list[i];
      length += buf.length;
    }
  }

  var buffer = new Buffer(length);
  var pos = 0;
  for (var i = 0; i < list.length; i++) {
    var buf = list[i];
    buf.copy(buffer.pos);
    pos += buf.length;
  }
  return buffer;
};
```

## 6.4 Buffer 与性能

Buffer 在文件 I/O 和网络 I/O 中运用广泛，尤其在网络传输中，它的性能举足轻重。在应用中，我们通常会操作字符串，但一旦在网络中传输，都需要转换为 Buffer,以二进制数据传输。在 Web 应用中，字符串转换到 Buffer 是时时刻刻发生的，提高字符串到 Buffer 的转换效率，可以很大程度地提高网络吞吐率。

在展开 Buffer 与网络传输的关系之前，我们可以先来进行一次性能测试。下面的例子中构造了一个 10kb 大小的字符串。我们通过纯字符串的方式向客户端发送：

```js
var http = require('http');
var helloworld = '';

for (var i = 0; i < 1024 * 10; i++) {
  helloworld += 'a';
}

// helloworld = new Buffer(helloworld);

http
  .createServer(function (req, res) {
    res.writeHead(200);
    res.end(helloworld);
  })
  .listen(8001);
```

我们通过 ab 进行一次性能测试，发起 200 个并发客户端：

```shell
$ ab -c 200 -t 100 http://127.0.0.1:8001/
```

得到的测试结果：

```shell
HTML transferred:         512000000 bytes
Requests per second:      2527.64 [#/sec] (mean)
Time per request:         79.125 [ms] (mean)
Time per request:         0.396 [ms] (mean, across all concurrent requests)
Transfer rate:            25370.16 [Kbytes/sec] received
```

测试的 QPS（每秒查询次数）是 2527.64，传输率为每秒 25370.16kb。
接下来我们打开`helloworld = new Buffer(helloworld);`的注释，使向客户端输出的是一个 Buffer 对象，无需再每次响应时进行转换。再次进行性能测试的结果如下：

```shell
Total transferred:        513900000 bytes
HTML transferred:         512000000 bytes
Requests per second:      4843.28 [#/sec] (mean)
Time per request:         41.294 [ms] (mean)
Time per request:         0.205 [ms] (mean, across all concurrent requests)
Transfer rate:            48612.56 [Kbytes/sec] received
```

QPS 提升到 4843.28，传输率为每秒 48612.56kb，性能接近提高一倍。

通过预先转换静态内容为 Buffer 对象，可以有效地减少 CPU 的重复使用,节省服务器资源。在 Node 构建的 Web 应用中,可以选择页面中的动态内容和静态内容分离，静态内容部分可以通过预先转换为 Buffer 的方式，使性能得到提升。由于文件自身时二进制数据，所以在不需要改变内容的场景下，尽量只读取 Buffer，然后直接传输，不做额外的转换，避免损耗。

- 文件读取

  Buffer 的使用除了与字符串的转换有性能损耗外，在文件的读取时，有一个 highWatermark 设置对性能的影响至关重要。在`fs.createReadStream(path, options)`时，我们可以传入一些参数，代码如下：

  ```js
  {
    flags: 'r',
    encoding: null,
    fd: null,
    mode: 0666,
    highWaterMark: 64 * 1024
  }

  // 我们还可以传递start 和 end来指定读取文件的位置范围
  {start: 90, end: 99}
  ```

  `fs.createReadStream`的工作方式是在内存中准备一段 Buffer,然后在`fs.read()`读取时逐步从磁盘将字节复制到 Buffer 中。完成依次读取时，则从这个 Buffer 中通过 slice()方法取出部分数据作为小 Buffer 对象，在通过 data 事件传递给调用方。如果 buffer 用完，则重新分配一个；如果还有剩余，则继续使用。

  ```js
  var pool;

  function allocNewPool(poolSize) {
    pool = new Buffer(poolSize);
    pool.used = 0;
  }
  ```

````

在理想状态下，每次读取的长度就是用户指定的`highWaterMark`。但是有可能读到了文件结尾，或者文件本身就没有指定的`highWaterMark`这么大，这个预先指定的Buffer对象将会有部分甚于，不过好在这里的内存可以分配给下次读取时使用。pool时常驻内存的，只有当pool单元剩余数量小于128字节时，才会重新分配一个新的Buffer对象。Node源代码中分配新的Buffer对象的判断条件如下所示：

```js
if(!pool || pool.length - pool.used < kMinPoolSpace) {
  // discard the old pool
  pool = null;
  allocNewPool(this._readabelState.highWaterMark);
}
````

这里与 Buffer 的内存分配比较类似，`highWaterMark`的大小对性能有两个影响的点：

- highWaterMark 设置对 Buffer 内存的分配和使用有一定的影响。
- highWaterMark 设置国小，可能导致系统调用次数过多。

文件流读取基于 Buffer 分配，Buffer 则给予 SlowBuffer 分配，这可以理解为两个维度的分配策略。如果文件较小（小于 8kb），有可能造成 slab 未能完全使用。

由于`fs.createReadStream()`内部采用`fs.read()`实现，将会引起对磁盘的系统调用，对于大文件而言，`highWaterMark`的大小决定会触发系统调用和 data 事件的次数。

以下为 Node 自带的基准测试，在`benchmark/fs/read-stream-throughput.js`中可以找到：

```js
function runTest() {
  assert(fs.statSync(filename).size === filesize);

  var rs = fs.createReadStream(filename, {
    highWaterMark: size,
    encoding: encoding,
  });

  rs.on('open', function () {
    bench.start();
  });
  var bytes = 0;

  rs.on('data', function (chunk) {
    bytes += chunk.length;
  });

  rs.on('end', function () {
    try {
      fs.unlinkSync(filename);
    } catch (e) {
      // TODO
    }

    // MB/sec
    bench.end(bytes / (1024 * 1024));
  });
}
```

下面为某次执行的结果：

```shell
fs/read-stream-throughput.js type=buf size=1024: 46.284
fs/read-stream-throughput.js type=buf size=4096: 139.62
fs/read-stream-throughput.js type=buf size=65535: 681.88
fs/read-stream-throughput.js type=buf size=1048576: 857.98
```

从上面的执行结果我们可以看到，读取一个相同的大文件时，`highWaterMark`值得大小与读取速度的关系： 该值越大，读取速度越快。

## 6.5 总结

体验过 JavaScript 友好的字符串操作后，有些开发发者可能会形成思维定势，将 Buffer 当作字符串来理解。但字符串与 Buffer 之间有实质上的差异，即 Buffer 是二进制数据，字符串与 Buffer 之间存在编码关系。因此，理解 Buffer 的诸多细节十分必要，对于如何高效处理二进制数据十分有用。
