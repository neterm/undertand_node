# 内存控制

---

- [5.1 V8 的垃圾回收机制与内存限制](#51-v8-的垃圾回收机制与内存限制)
- [5.2 高效使用内存](#52-高效使用内存)
- [5.3 内存指标](#53-内存指标)
- [5.4 内存泄漏](#54-内存泄漏)
- [5.5 内存泄漏排查](#55-内存泄漏排查)
- [5.6 大内存应用](#56-大内存应用)
- [5.7 总结](#57-总结)

---

也许读者会好奇为何会有这样一章存在于本书中，因为在过去很长一段时间内，JavaScript 开发者很少在开发过程中遇到需要对内存精确控制的场景，也缺乏控制的手段。说到内存泄漏，大家首先想起的也是在早期版本的 IE 中 JavaScript 与 DOM 交互时发生的问题。如果页面里的内存占用过多，基本等不到进行代码回收，用户已经不耐烦地刷新了当前页面。

随着 Node 地发展，JavaScript 已经实现了 CommonJS 的生态圈大一统的梦想，JavaScript 的应用场景早已不再局限于浏览器中。本章将暂时抛开那些短时间执行场景，比如网页应用、命令行工具等，这类场景由于运行事件短，且运行在机器上，即使内存使用过多或内存泄漏，也只会影响到终端用户。由于运行时间短，随着进程退出，内存释放，几乎没有内存管理的必要。但是随着 Node 在服务器端广泛应用，其它语言里存在着的问题在 JavaScript 中也暴露出来了。

基于无阻塞、事件驱动建立的 Node 服务，具有内存消耗低的优点，非常适合处理海量的网络请求。在海量请求的前提下，开发者就需要考虑一些平常不会形成影响的问题。本书写道这里算是正式迈进服务器端编程的领域了，内存控制正式在海量请求和长时间运行的前提下进行探讨的。在服务器端，资源向来是寸土寸金，腰围海量用户服务，旧的使一切资源都要高效循环利用。在第三章，差不多已经介绍完 Node 是如何利用 CPU 和 I/O 这两个服务器资源，而本章将介绍在 Node 中如何高效地使用内存。

## 5.1 V8 的垃圾回收机制与内存限制

我们在学习 JavaScript 编程时听说过，它与 Java 一样，由垃圾回收机制来进行自动内存管理，这使得开发者不需要像 C/C++程序员那样在编写代码的过程中时刻关注内存的分配和释放问题。但在浏览器中进行开发时，几乎很少有人能遇到垃圾回收对应用程序构成性能影响的情况。Node 极大地拓宽了 JavaScript 的应用场景，当主流应用场景从客户端延伸到服务器端后，我们就能发现，对于性能敏感的服务器端程序，内存管理的好坏、垃圾回收状况是否优良，都会对服务构成影响。而 Node 中，这一切都与 Node 的 JavaScript 执行引擎 V8 息息相关。

### 5.1.1 Node 与 V8

回溯历史可以发现，Node 在发展历程中离不开 V8，所以在官方的主页介绍中就提到 Node 是一个构建在 Chrome 的 JavaScript 运行时上的平台。2009 年，Node 的创始人 Ryan Dahl 选择 V8 来作为 Node 的 JavaScript 脚本引擎，这离不开当时硝烟四起的第三次浏览器大战。那次大战之中，来自 Google 的 Chrome 浏览器以其优异的性能称为焦点。Chrome 成功的背后离不开 JavaScript 引擎 V8。V8 出现后，JavaScript 性能优势使得用 JavaScript 写高性能后台服务程序称为可能。在这样的契机下，Ryan Dahl 选择了 JavaScript，选择了 V8，在事件驱动、非阻塞 I/O 模型的设计下实现了 Node。

关于 V8，它的来历与背景亦是大有来头。作为虚拟机，V8 的性能表现优异，这与它的领导者有莫大的渊源，Chrome 的成功也离不开它背后的天才——Lars Bak。在 Lars 的工作履历里，绝大部分都是与虚拟机相关的工作。在开发 V8 之前，他曾经在 Sun 公司工作，担任 HotSpot 团队的技术领导，主要致力于开发高性能的 Java 虚拟机。在这之前，他也曾为 Self、Smalltalk 语言开发过高性能虚拟机。这些无与伦比的经历让 V8 一出世就超越了当时所有的 JavaScript 虚拟机。

Node 在 JavaScript 的执行上直接受益于 V8，可以随着 V8 的升级就能享受到更好的性能或新的语言特性（如 ES5 和 ES6）等，同时也收到 V8 的一些限制，尤其是本章重点讨论的内存限制。

### 5.1.2 V8 的内存限制

在一般的后端开发语言中，在基本的内存使用上没什么限制，然后再 Node 中使用 JavaScript 使用内存时，就会发现只能使用部分内存（64 位系统下约为 1.4GB，32 位系统下约为 0.7GB）。在这样的限制下，将会导致 Node 无法直接操作大内存对象，比如无法将一个 2GB 的文件读入内存中进行字符串分析处理，即使物理内存有 32GB。这样在单个 Node 进程的情况下，计算机的内存资源无法得到充足的使用。

造成这个问题的主要原因在于 Node 基于 V8 构建，所以在 Node 中使用的 JavaScript 对象基本上都是通过 V8 自己的方式来进行分配和管理的。V8 的这套内存管理机制在浏览器的应用场景下使用起来绰绰有余，足以胜任前端页面中的所有需求。但在 Node 中，这却限制了开发者随心所欲使用大内存的想法。

尽管在服务器端操作大内存也不是常见的需求场景，但是有了这个限制之后，我们的行为就如同带着镣铐跳舞，如果在实际应用中不小心触碰到了这个界限，会造成进程退出。要知晓 V8 为何限制了内存的用量，则需要回归到 V8 在内存使用上的策略。知晓其原理后，才能避免问题并更好地进行内存管理。

### 5.1.3 V8 的对象分配

在 V8 中，所有的 JavaScript 对象都是通过堆来进行分配的。Node 提供了 V8 中内存使用量的查看方式，执行下面的代码，将得到输出的内存信息：

```sh
$node
process.memoryUsage();
{
  rss: 20889600,
  heapTotal: 4911104,
  heapUsed: 2442864,
  external: 1417572
}
```

在上述代码中，在`memoryUsage()`方法返回的 3 个属性中，`heapTotal`和`heapUsed`是 V8 的堆内存使用情况，前者是已申请到的堆内存，后者是当前使用量。至于`rss`为何，我们在后续的内容中会介绍到。

当我们在代码中声明变量并赋值时，所使用对象的内存就是分配在堆中。如果已申请的堆空闲内存不够分配新的对象，将继续申请堆内存，直到堆的大小超过 V8 的限制位置。

至于 V8 为何要限制堆的大小，表层原因为 V8 最初为浏览器而涉及，不太可能遇到用大量内存的场景。对于网页来说，V8 的限制值已经绰绰有余。深层原因是 V8 的垃圾回收机制的限制。按官方的说法，以 1.5GB 的啦经济回收堆内存为例，V8 做依次效地垃圾回收需要 50ms 以上，做一次非增量式的垃圾回收甚至需要 1s 以上。这是垃圾回收中引起 JavaScript 线程暂停执行的事件，在这样的时间花销下，应用的性能和响应能力都会直线下讲。这样的情况不仅仅后端服务无法接收。因此，在当时的考虑下直接限制堆内存是一个好的选择。

当然，这个限制也不是不能打开，V8 依然提供了选项让我们使用更多的内存。Node 在启动时传递`--max-old-space-size`或`--max-new-space-size`来调整内存限制的大小：

```sh
node --max-old-space-size=1700 test.js # 单位为MB
node --max-new-space-size=1500 test.js # 单位为KB
```

上述参数在 V8 初始化时生效，一旦生效就不能再动态改变。如果遇到 Node 无法分配足够内存给 JavaScript 对象的情况，可以用这个办法来放宽 V8 默认的内存限制，避免在执行过程中稍微多用了一些内存就轻易崩溃。

接下来，让我们更深入地了解 V8 的垃圾回收方面的策略。在限制的前提下，带着镣铐跳出的舞蹈并不一定就难看。

### 5.1.4 V8 的垃圾回收机制

在展开介绍 V8 的垃圾回收机制前，有必要简略介绍下 V8 用到的各种垃圾回收算法。

#### 1. V8 主要的垃圾回收算法

V8 的垃圾回收策略主要基于分代式垃圾回收机制。在自动垃圾回收的演变过程中，人们发现没有一种垃圾回收算法能够胜任所有的场景。因为在实际的应用中，对象的生存周期长短不一，不同的算法只能针对特定情况具有最好的效果。为此，统计学在垃圾回收算法的发展中产生了较大的作用，现代的垃圾回收算法中按对象的存活时间，将内存的垃圾回收进行不同的分代，然后分别堆不同分代的内存施以更高效的算法。

- V8 的内存分代

  在 V8 中，主要将内存分为新生代和老生代两代。新生代中的对象为存活时间较短的对象，老生代中的对象为存活时间较长或常驻内存的对象。

  V8 堆的整体大小就是新生代所用空间加上老生代的内存空间。前面我们提及的`--max-old-space-size`命令行参数可以用于设置老生代空间的最大值，`--max-new-space-size`命令行参数则用于设置新生代内存空间的大小。比较遗憾的是，这两个最大值需要在启动时就指定。这意味着 V8 使用的内存没有办法根据使用情况自动扩充，当内存分配过程中超过极限值时，就会引起进程出错。

  前面提到过，在默认设置下，如果一直分配内存，在 64 位系统和 32 位系统下，会分别只能使用约 1.4GB 和 0.7GB 的大小。这个限制可以从 V8 的源码中找到。在下面的代码中，`Page::kPageSize`的值位 1MB。可以看出老生代的设置在 64 位系统下为 1400MB，在 32 位系统下约为 700MB：

  ```c++
  // semispace_size_ should be a power of 2 and old_generation_size_ should be
  // a multiple of Page::kPageSize
  #if defined(V8_TARGET_ARCH_X64)
  #define LUMP_OF_MEMORY (2 * MB)
    code_range_size_(512*MB),
  #else
  #define LUMP_OF_MEMORY MB
    code_range_size_(0),
  #endif

  #if defined(ANDROID)
    reserved_semispace_size_(4 * Max(LUMP_OF_MEMORY, Page::kPageSize)),
    max_semispace_size_(4 * Max(LUMP_OF_MEMORY, Page::kPageSize)),
    initial_semispace_size_(Page::kPageSize),
    max_executable_size_(max_old_generation_size),
  #else
    reserved_semispace_size_(8 * Max(LUMP_OF_MEMORY, Page::kPageSize)),
    max_semispace_size_(8 * Max(LUMP_OF_MEMORY, Page::kPageSize)),
    initial_semispace_size_(Page::kPageSize),
    max_executable_size_(700ul * LUMP_OF_MEMORY),
    max_executable_size_(256l * LUMP_OF_MEMORY),
  #endif
  ```

  对于新生代内存，它由两个`reserved_semispace_size_`所构成，后面将描述其原因。按机器位数不同，`reserved_semispace_size_`在 64 位系统和 32 位系统分别是 16MB 和 8MB。所以新生代内存的最大值在 64 位系统和 32 位系统分别是 32MB 和 16MB。

  V8 堆内存的最大保留空间可以从下面代码中看出来，其公式为`4* reserved_semispace_size_ + max_old_generation_size`：

  ```c++
  intptr_t MaxReserved() {
    return 4 * reserved_semispace_size_ + max_old_generation_size;
  }
  ```

  因此默认情况下，V8 堆内存的最大值在 64 位系统上位 1464MB，32 位系统上则为 732MB。这个数值可以解释为何 64 位系统只能使用约 1.4GB 和 32 位系统只能使用约 0.7GB 内存。

- Scavenge 算法

  在分代的基础上，新生代中的对象主要通过 Scavenge 算法进行垃圾回收。在 Scavenge 的具体视线中，主要采用了 Cheney 算法，该算法由 C.J.Cheney 于 1970 年首次发表在 ACM 论文上。

  Cheney 算法是一种采用复制的方式实现的垃圾回收算法。它将堆内存一分为二，每一部分空间称为 semispace。在这两个 semiseapce 空间中，只有一个处于使用中，另一个处于闲置状态。处于使用状态的 semispace 空间称为 From 空间，处于闲置状态的空间称为 To 空间。但我们分配对象时，先是在 From 空间分进行分配。当开始进行垃圾回收时，会检查 From 空间中的存活对象，这些存活对象被复制到 To 空间中，而非存活对象占用的空间将会被释放。完成复制后，From 空间和 To 空间的角色发生对换。简而言之，在垃圾回收的过程中，就是通过将存活对象在两个 semispace 空间之间进行复制。

  Scavenge 的缺点是只能使用堆内存中的一半，这是由划分空间和复制机制所决定的。但 Scavenge 由于只复制存活的对象，并且对于生命周期短的场景存活对象只占少部分，所以它在时间效率上有优异的表现。

  由于 Scavenge 是典型的牺牲空间换取时间的算法，所以无法大规模应用到所有垃圾回收中，Scavenge 非常适合应用在新生代中，因为新生代对象的生命周期较短，恰恰适合这个算法。

  实际使用的堆内存是新生代的两个 semispace 空间大小和老生代所用内存大小之和。

  当一个对象经过多次复制依然存活时，它将会被认为是生命周期较长对象。这种较长生命周期的对象将会被移动到老生代中，采用新的算法进行管理。对象从新生代中移动到老生代中的过程称为晋升。

  在单纯的 Scavenge 过程中，From 空间中的存活对象会被复制到 To 空间中去，然后对 From 空间和 To 空间进行角色对换（又称翻转）。但在分代式垃圾回收的前提下，From 空间中的存活对象在复制到 To 空间之前需要进行检查。在一定条件下，需要将存货周期长的对象移动到老生代中，也就是完成对象晋升。

  对象晋升的条件主要有两个，一个是对象是否经历过 Scavenge 回收，一个是 To 空间的内存占用比超过闲置。

  在默认情况下，V8 的对象分配主要集中在 From 空间中。对象从 From 空间中复制到 To 空间时，会检查它的内存地址来判断这个对象是否已经经历过依次 Scavenge 回收。如果已经经历过了，会将该对象从 From 空间复制到老生代空间中，如果没有则复制到 To 空间中。

  另一个判断条件是 To 空间的内存占用比。当要从 From 空间复制一个对象到 To 空间时，如果 To 空间已经使用了超过 25%，则这个对象直接晋升到老生代空间中，这个晋升到老生代空间中。

  设置 25%这个限制值得原因时当这次 Scavenge 回收完成后，这个 To 空间将变成 From 空间，接下来得内存分配将在这个空间中进行。如果占用比过高，会影响后续的内存分配。

  对象晋升后，将会在老生代空间中作为存活周期较长的对象来对待，接收新的回收算法处理。

- Mark-Sweep & Mark-Compact

  对于老生代中的对象，由于存活对象占比较大比重，再采用 Scavenge 的方式会有两个问题：一是存活对象较多，复制存活对象的效率将会降低；另一个问题依然是浪费一半空间的问题。这两个问题导致应对生命周期较长的对象时，Scavenge 会显得捉襟见肘。为此，V8 在老生代中主要采用了 Mark-Sweep 和 Mark-Compact 相结合的方式进行垃圾回收。

  Mark-Sweep 是标记清除的意思，它分为标记和清除两个阶段。与 Scavenge 相比，Mark-Sweep 并不将内存空间划分为两半，所以不存在浪费一半空间的行为。与 Scavenge 复制活着的对象不同，Mark-Sweep 在标记阶段遍历堆中的所有对象，并标记活着的对象，在随后的清理阶段中，只清除没有被标记的对象。可以看出 Scavenge 中只复制活着的对象，而 Mark-Sweep 只清理死亡的对象。活对象在新生代中只占用较小的部分，死对象在老生代中只占用较小的部分，这是两种回收方式能高效处理的原因。

  Mark-Sweep 最大的问题是在进行依次标记清除回收后，内存会出现不连续的状态。这种内存碎片会对后续的内存分配造成问题，因为有可能出现需要分配一个大对象的情况，这时所有的碎片空间都无法完成此次分配，就会提前触发垃圾回收，而这次回收是不必要的。

  为了解决 Mark-Sweep 的内存碎片问题，Mark-Compact 被提出来。Mark-Compact 是标记整理的意思，实在 Mark-Sweep 的基础上演变而来的。它们的差别在于对象在标记位死亡后，在整理的过程中，将活着的对象往一端移动，移动完成后，直接清理掉边界外的内存。

  完成移动后，就可以直接清除最右边的存活对象后面的内存区域完成回收。

  这里将 Mark-Sweep 和 Mark-Compact 结合着介绍不仅仅是因为这两种策略是递进关系，在 V8 的回收策略中，两者是结合使用的。

  在 Mark-Sweep 和 Mark-Compact 之间，由于 Mark-Compact 需要移动对象，所以它的执行速度并不快，所以在取舍上，V8 主要使用 Mark-Sweep，在空间不足以堆从新生代晋升过来的对象进行分配时才使用 mark-Compact。

- Incremental Marking

  为了避免出现 JavaScript 应用逻辑与垃圾回收器看到的不一致的情况，垃圾回收的三种基本算法都需要将应用逻辑暂停下来，待执行完垃圾回收后再恢复执行应用逻辑，这种行为称为“全停顿”。在 V8 的分代式来及回收中，一次小垃圾回收只收集新生代，由于新生代默认配置得较小，且其中存活对象通常较少，所以即便它时全停顿也影响不大。但 V8 得老生代通常配置较大，且存活对象较多，全堆垃圾回收（full 垃圾回收）的标记、清理、整理等动作造成的停顿就会比较可怕，需要设法改善。

  为了降低全堆垃圾回收带来的停顿时间，V8 先从标记阶段入手，将原本要一口气停顿完成的动作改为增量标记，也就是拆分为许多小“步进”，每昨晚一个“步进”，就让 JavaScript 逻辑执行一小会儿，垃圾回收与应用逻辑交替执行直到标记阶段完成。

  V8 在经过增量标记改进后，垃圾回收的最大停顿时间可以减少到原来的 1/6 左右。

  V8 后续还引入了延迟清理与增量式整理，让清理和整理动作也变成增量式的。同时还计划引入并行标记与并行清理，进一步利用多核性能降低每次停顿时间。鉴于篇幅有限，此处不再深入讲解。

#### 2. 小结

从 V8 的自动垃圾回收机制的设计角度可以看到，V8 对内存使用进行限制的缘由。新生代设计为一个较小的内存空间是合理的，而老生代空间过大对于垃圾回收并无特殊意义。V8 对内存限制的设置对于 Chrome 浏览器这种每个选项卡使用一个 V8 实例而言，内存的使用是绰绰有余了。对于 Node 编写的服务器端来说，内存限制也并不影响正常场景下的使用。但是对于 V8 的垃圾回收特点和 JavaScript 在单线程上的执行情况，垃圾回收是性能正常场景下的使用。但是对于 V8 的垃圾回收特点和 JavaScript 在单线程上的执行情况，垃圾回收也是影响性能的因素之一。想要高性能的执行效率，需要注意让垃圾回收尽量少进行，尤其是全堆垃圾回收。

以 Web 服务器中的会话实现为例，一般通过内存来存储，但在访问量大的时候会导致老生代中的存活对象骤增，不仅造成清理/整理过程费事，还会造成内存紧张，甚至溢出。

### 5.1.5 查看垃圾回收日志

查看垃圾回收日志的方式主要是在启动时添加--trace_gc 参数。在进行垃圾回收时，将会从标准输出中打印垃圾回收的日志信息。下面是一段示例，执行结束后，将会在 gc.log 文件中得到所有垃圾回收信息：

```sh
node --trace_gc -e "var a = [];for (var i = 0; i< 1000000; i++) a.push(new Arrya(100));" > gc.log
```

通过分析垃圾日志，可以了解垃圾回收的运行状况，找出垃圾回收的哪些阶段比较耗时，触发的原因是什么。

通过在 Node 启动时使用`--prof`参数，可以得到 V8 执行时的性能分析数据，其中包含了垃圾回收执行占用的时间。下面的代码不断创建对象并将其分配给局部变量 a，这里将以下代码存为 test01.js 文件：

```js
for (var i = 0; i < 1000000; i++) {
  var a = {};
}
```

执行命令加上`--prof`参数，将会在目录下得到一个 v8.log 日志文件。该日志文件基本不具备可读性。

所幸，V8 提供了`linux-tick-processor`工具用于统计日志信息。该工具可以从 Node 源码的`deps/v8/tools`目录下找到，Windows 下对应的命令文件为`windows-tick-processor.bat`。将改目录添加到环境变量 path 中，即可直接调用：

```sh
$ linux-tick-processor v8.log
```

统计内容较多，其中垃圾回收部分：

```sh
[GC]:
  ticks   total   nonlib    name
    2      5.4%
```

由于不断分配对象，垃圾回收所占的时间为 5.4%。按此比例，这意味着事件循环执行 1000ms 的过程中要给出 54ms 的时间用于垃圾回收。

## 5.2 高效使用内存

在 V8 面签，开发者所要具备的责任是如何让垃圾回收机制更高效地工作。

### 5.2.1 作用域

提到如何触发垃圾回收，第一个介绍的是作用域（scope）。在 JavaScript 中能形成作用域的有函数调用、with 以及全局作用域。

以如下代码为例：

```js
var foo = function () {
  var local = {};
};
```

foo()函数在每次被调用时会创建对应的作用域，函数执行结束后，该作用域将会销毁。同时作用域中声明的局部变量分配在该作用域上，随作用域的销毁而销毁。只被局部变量引用的对象存货周期较短。在这个示例中，由于对象非常小，将会分配在新生代的 From 空间中。在作用域释放后，局部变量 local 失效，其引用的对象将会在下次垃圾回收时被释放。

以上就是最基本的内存回收过程。

#### 1.标识符查找

与作用域相关的即是标识符查找。所谓标识符，可以理解为变量名。在下面代码中，执行 bar()函数时，将会遇到 local 变量：

```js
var bar = function () {
  console.log(local);
};
```

JavaScript 在执行时回去查找该变量定义在哪里。它最先查找的是当前作用域，如果在当前作用域中无法找到该变量的声明，将会向上级作用域查找，直到查到位置。

#### 2.作用域链

在下面代码中：

```js
var foo = function () {
  var local = 'local var';
  var bar = function () {
    var local = 'another var';
    var baz = function () {
      console.log(local);
    };
    baz();
  };
  bar();
};
foo();
```

local 变量在 baz()函数形成的作用域里查找不到，继而将在 bar()的作用域里寻找。如果去掉上述代码 bar()中的 local 声明，将会继续向上查找，一直到全局作用域。这样的查找方式使得作用域像一条链条。由于标识符的查找方向是向上，所以变量只能向外访问，而不能向内访问。

当我们在 baz()函数中访问 local 变量时，由于作用域中的变量列表没有 local，所以会向上一个作用域中查找，接着会在 bar()函数执行得到的变量列表中找到一个 local 变量的定义，于是使用它。尽管在再上一层的作用域中也存在 local 的定义，但是不会继续查找了。如果查找一个不存在的变量，将会一直沿着作用域链查找到全局作用域，最后抛出未定义错误。

了解了作用域，有助于我们了解变量的分配和释放。

#### 3. 变量的主动释放

如果变量是全局作用域（不通过 var 声明或定义在 global 变量上），由于全局作用域需要直到进程退出才释放，此时将导致引用的对象常驻内存（常驻在老生代）。如果需要释放常驻内存的对象，可以通过 delete 操作来删除引用关系。或者将变量重新赋值，让旧的对象脱离引用关系。在接下来的老生代内存清理和整理过程中，会被回收释放。下面为示例代码：

```js
global.foo = 'I am global object';
console.log(global.foo); // => I am global object
delete global.foo;

// 或者重新赋值
global.foo = undefined; // or null
console.log(global.foo); // => undefined
```

同样，如果在非全局作用域中，想主动释放变量引用的对象，也可以通过这样的方式。虽然 delete 操作和重新赋值具有相同效果，但是在 V8 中通过 delete 删除对象的属性有可能干扰 V8 的优化，所以通过赋值解除引用更好。

### 5.2.2 闭包

我们直到作用域链上的对象访问只能向上，这样外部无法向内部访问。如下代码可以正常打印：

```js
var foo = function () {
  var local = '局部变量';
  (function () {
    console.log(local);
  })();
};
```

但在下面代码中，却会得到 local 未定义的异常：

```js
var foo = function () {
  (function () {
    var local = '局部变量';
  })();
  console.log(local);
};
```

在 JavaScript 中，实现外部作用域访问内部作用域中变量的方法叫做闭包（closure）。这得益于高阶函数的特性：函数可以作为参数或者返回值。示例代码如下：

```js
var foo = function () {
  var bar = function () {
    var local = '局部变量';

    return function () {
      return local;
    };
  };
  var baz = bar();
  console.log(baz());
};
```

一般而言，在 bar()函数执行完成后，局部变量 local 将会随着作用域的销毁而被回收。但是注意这里的特点在于返回值是一个匿名函数，且这个函数中具备了访问 local 的条件。虽然在后续执行中，在外部作用域还是无法直接访问 local，但是若要反问他，只要通过这个中间函数稍作周转即可。

闭包是 JavaScript 的高级特性，利用它可以产生很巧妙的效果。它的问题在于，一旦变量引用了这个中间函数，这个中间函数将不会释放，同时也会使原来的作用域不会得到释放，作用域中产生的内存占用也不会得到释放。除非不再有引用，才会逐步释放。

### 5.2.3 小结

在正常的 JavaScript 执行中，无法立即回收的内存有闭包和全局变量引用这两种情况。由于 V8 的内存限制，要十分小心此类变量是否无限地增加，因为它会导致老生代中的对象增多。

## 5.3 内存指标

一般而言，应用中存在一些全局性的对象是正常的，而且在正常的使用中，变量都会自动释放回收。但是也会存在一些我们认为会回收但是却没有被回收的对象，这会导致内存占用无限增长。一旦增长达到 V8 的内存限制，将会得到内存溢出错误，进而导致进程退出。

### 5.3.1 查看内存使用情况

前面我们提到了`process.memoryUsage()`可以查看内存使用情况。除此之外，os 模块中的`totalmem()`和`freemem()`方法也可以查看内存使用情况。

#### 1.查看进程的内存占用

调用`process.memoryUsage()`可以查看 Node 进程的内存占用情况：

```sh
$node
> process.memoryUsage()
{
  rss: 20959232,
  heapTotal: 4911104,
  heapUsed: 2378248,
  external: 1417561
}
```

rss 是 resident set size 的缩写，即进程的常驻内存部分。进程的内存总共分为几部分，一部分是 rss，其余部分在交换区（swap）或者文件系统（filesystem）中。

除了 rss 外，heapTotal 和 heapUsed 对应的是 V8 的堆内存信息。heapTotal 是堆中总共申请的内存量，heapUsed 表示目前堆中使用中的内存量。这 3 个值得单位都是字节。

为了更好地查看效果，我们格式化以下输出结果：

```js
var showMen = function () {
  var mem = process.memoryUsage();
  var format = function (bytes) {
    return (bytes / 1024 / 1024).toFixed(2) + 'MB';
  };
  console.log(
    'Process: heapTotal ' +
      format(mem.heapTotal) +
      ' heapUsed ' +
      format(mem.heapUsed) +
      ' rss ' +
      format(mem.rss)
  );
  console.log('----------------------------------------------');
};
```

同时写一个方法不停地分配内存但不是放内存，相关代码如下：

```js
var useMem = function () {
  var size = 20 * 1024 * 1024;
  var arr = new Array(size);
  for (var i = 0; i < size; i++) {
    arr[i] = 0;
  }
  return arr;
};

var total = [];

for (var j = 0; j < 15; j++) {
  showMen();
  total.push(useMem());
}
showMen();
```

将以上代码存为`outofmemory.js`并执行它。

可以看到，每次调用`useMem`都导致 3 个值的增长。在接近 1500MB 的时候，无法继续分配内存，然后进程内存溢出了，连循环体都无法执行完成，仅执行了 7 次。

#### 2.查看系统的内存占用

与`process.memoryUsage()`不同的是，os 模块中的`totalmem()`和`freemem()`这两个方法用于查看操作系统的内存使用情况，它们分别返回系统的总内存和闲置内存，以字节为单位。示例代码如下：

```sh
$ node
> os.totalmem();
> 12865875968
> os.freemem();
> 7039471616
```

从输出信息可以看到我的电脑总内存为 12GB，当前闲置内存大致是 7GB。

### 5.3.2 堆外内存

通过`process.memoryUsage()`的结果可以看出，堆中的内存用量总是小于进程的常驻内存用量，这意味着 Node 中的内存使用并非都是通过 V8 进行分配的。我们将不是通过 V8 分配的内存称为**堆外内存**。

这里我们将前面`useMem()`方法稍微改造下，将 Array 变为 Buffer，将 size 变大，每一次构造 200MB 的对象，相关代码如下：

```js
var useMem = function () {
  var size = 200 * 1024 * 1024;
  var buffer = new Buffer(size);
  for (var i = 0; i < size; i++) {
    buffer[i] = 0;
  }
  return buffer;
};
```

重新执行该代码。我们看到 15 次循环都完整执行，并且三个内存占用之与前一个示例完全不同。在改造后的输出结果中，heapTotal 与 heapUsed 的变化极小，唯一变化的是 rss 的值，并且该值远远超过 V8 的限制值。这其中的原因是 Buffer 对象不同于其它对象，它不经过 V8 的内存分配机制，所以也不会有堆内存的大小限制。

这意味着利用堆外内存可以突破内存限制问题。

为何 Buffer 对象并非通过 V8 分配？这在于 Node 不同于浏览器的应用场景。在浏览器中，JavaScript 直接处理字符串即可满足绝大多数的业务场景需求，而 Node 则需要处理网络流和文件 I/O 流，操作字符串远远不能满足传输的性能需求。

### 5.3.3 小结

从上面介绍可以得知，Node 的内存构成主要由通过 V8 进行分配的部分和 Node 自行分配的部分。受 V8 的垃圾回收限制的主要是 V8 的堆内存。

## 5.4 内存泄漏

Node 对内存泄漏非常敏感，一旦线上应用有成千上万的流量，哪怕是一个字节的内存泄漏也会造成堆积，垃圾回收过程中将会耗费更多时间进行对象扫描，应用响应缓慢，直到进程内存溢出，应用奔溃。

在 V8 的垃圾回收机制下，在通常的代码编写中，很少会出现内存泄漏的情况。但是内存泄漏通常产生于无意间，较难排查。尽管内存泄漏的情况不尽相同，但其实质只有一个，那就是应当回收对象出现意外而没有被回收，变成了常驻老生代中的对象。

通常，造成内存泄漏的原因有如下几个：

- 缓存。
- 队列消费不及时。
- 作用域未释放。

### 5.4.1 慎将内存当作缓存

缓存在应用中的作用举足轻重，可以十分有效地节省资源。因为它的访问效率要比 I/O 的效率高，一旦命中缓存，就可以节省一次 I/O 的时间。

但是在 Node 中，缓存并非物美价廉。一旦一个对象被当作缓存来使用，那么就意味着它将会常驻老生代中。缓存中存储的键越多，长期存活的对象也就越多，这将导致垃圾回收在进行扫描和整理时，对这些对象做无用功。

另一个问题在于，JavaScript 开发者通常喜欢用对象的键值对来缓存东西，但这与严格意义上的缓存又有着区别，严格意义的缓存有着完善的过期策略，而普通对象的键值对并没有。

如下代码虽然利用 JavaScript 对象十分容易创建一个缓存对象，但是受垃圾回收机制的影响，只能小量使用：

```js
var cache = {};
var get = function (key) {
  if(cacke[key])
    return cache[key];
  else
    // get from otherwise
};

var set = function (key, value) {
  cache[key] = value;
};
```

上述示例代码在解释原理后，十分容易理解，如果需要，只要限定缓存对象的大小，加上完善的过期策略以防止内存无限制增长，还是可以一用的。

这里给出一个可能无意识造成内存泄漏的场景： memoize。下面时著名类库 underscore 对 memoize 的实现：

```js
_.memoize = function (func, hasher) {
  var memo = {};
  hasher || (hasher = _.identity);
  return _.has(memo, key) ? memo[key] : (memo[key] = func.apply(this, arguments));
};
```

它的原理时以参数作为键进行缓存，以对象空间换 CPU 执行时间。这里潜藏的陷阱即是每个被执行的结果都会按参数缓存在 memo 对象上，不会被清除。这在前端网页这种短时间应用场景中不存在大问题，但是执行量大和参数多样性的情况下，就会造成内存占用不是放。

所以在 Node 中，任何视图拿内存当缓存的行为都应当被限制。当然，这种限制并不是不允许使用的意思，而是要小心用之。

#### 1.缓存限制策略

为了缓解缓存中的对象永远无法释放的问题，需要加入一种策略来限制缓存的无限增长。为此我曾写过一个模块 limitablemap，它可以实现对简直数量的限制。下面是其实现：

```js
var LimitableMap = function (limit) {
  this.limit = limit || 10;
  this.map = {};
  this.keys = {};
};

var hasOwnProperty = Object.prototype.hasOwnProperty;

LimitableMap.prototype.set = function (key, value) {
  var map = this.map;
  var keys = this.keys;
  if (!hasOwnProperty.call(map, key)) {
    if (keys.length === this.limit) {
      delete map[firstKey];
    }
    keys.push(key);
  }
  map[keys] = value;
};

LimitableMap.prototype.get = function (key) {
  return this.map[key];
};

module.exports = LimitableMap;
```

可以看到，实现过程还是非常简单的。记录在数组中，一旦超过变量，就先进先出的方式进行淘汰。

当然，这种淘汰策略并不十分高效，只能应付小型应用场景。如果需要更高效的缓存，可以参见 Isaac Z. Schlueter 采用[LRU 算法](https://github.com/issacs/node-lru-cache 'LRU算法')的缓存。结合有限制的缓存，memoize 还是可用的。

另一个案例在于模块机制。为了加速模块的引入，所有模块都会通过编译执行，然后被缓存起来。由于通过 exports 导出的函数，可以访问文件模块中的私有变量，这样每个文件模块在编译后形成的作用域因为模块缓存的原因，不会被释放。

```js
(function (exports, require, module, __filename, __dirname) {
  var local = '局部变量';

  exports.get = function () {
    return local;
  };
});
```

由于模块的缓存机制，模块是常驻老生代的。在设计模块时，要十分小心内存泄漏的出现。在下面代码中，每次调用`leak()`方法时，都导致局部变量`leakArray`不停增加内存的占用，且不被释放：

```js
var leakArray = [];
exports.leak = function () {
  leakArray.push('leak', Math.random());
};
```

如果模块不可避免地需要这么设计，请添加清空队列地响应接口，以供调用者释放内存。

#### 2.缓存地解决方案

直接将内存作为缓存地方案要十分慎用。除了限制缓存大小外，另外要考虑地事情是，进程之间无法共享内存。如果在进程内使用缓存，这些缓存不可避免地有重复，对物理内存的使用也是一种浪费。

如何使用大量缓存，目前比较好的解决方案是采用进程外的缓存，进程自身不存储状态。外部的缓存软件有着良好的缓存过期淘汰策略以及自身的内存管理，不影响 Node 进程的性能。它的好处多多，在 Node 中主要可以解决以下两个问题：

- 将缓存转移到外部，减少常驻内存的对象数量，让垃圾回收更高效。
- 进程之间可以共享缓存。

目前，市面上较好的缓存有 Redis 和 Memcached。Node 模块的生态系统十分完善，这两个产品客户端都有：

- [Redis](https://github.com/mranney/node_redis 'redis')。
- [Memcached](https://github.com/3rd-Eden/node-memcached 'Memcached')。

### 5.4.2 关注队列状态

为了解决缓存带来的内存泄漏问题后，另一个不经意产生内存泄漏则是队列。在 JavaScript 中可以通过队列（数组对象）来完成许多特殊需求，比如 Bagpipe。队列在消费者-生产者模型中经常充当中间产物。这时一个容易忽略的情况，因为在大多数应用场景下，消费的速度远大于生产的速度，内存泄漏不易发生。但是一旦消费速度低于生产速度，将会形成堆积。

举个实际例子，有的应用会收集日志。如果欠缺考虑，也许会采用数据库来记录日志。日志通常会是海量的，数据库构建在文件系统之上，写入效率远远低于文件直接写入，于是会形成数据库写入操作的堆积，而 JavaScript 中相关的作用域也不会得到释放，内存占用不会回落，从而出现内存泄漏。

遇到这种场景，表层的解决方案是换用消费速度更高的技术。在日志收集的案例，换用文件写入日志的方式会更高效。需要注意的是，如果生产速度因为某些原因突然激增，或者消费速度因为突然的系统故障降低，内存泄漏还是可能出现的。

深度的解决方案应该是监控队列的长度，一旦堆积，应当通过监控系统产生报警并通知相关人员。另一个解决方案是任意异步调用都应该包含超时机制，一旦在限定的时间内完成响应，同故宫回调函数传递超时异常，使得任意异步调用的回调都具备可控响应时间，给消费速度一个下限值。

对于 Bagpipe 而言，它提供了超时模式和拒绝模式。启用超时模式时，调用加入到队列中就开始计时，超时就直接响应一个超时错误。启用拒绝模式时，当队列拥塞时，新到来的调用就会直接响应拥塞错误。这两种模式都能有效防止队列拥塞导致的内存泄漏问题。

## 5.5 内存泄漏排查

前面提及了集中导致内存泄漏的常见类型。在 Node 中，由于 V8 的堆内存大小的限制，它对内存泄漏非常敏感。当在线服务的请求量变大时，哪怕是一个字节的泄漏都会导致内存占用过高。这里介绍一下遇到内存泄漏时的排查方案。

现在已经有许多工具用于定位 Node 应用的内存泄漏，下面时一些常见的工具。

- v8-profiler。由 Danny Coates 提供，它可以用于对 V8 堆内存抓取快照和对 CPU 进行分析，但该项目已经有 3 年没有维护了。

- node-heapdump。这时 Node 核心贡献者之一 Ben Noordhuis 编写的模块，它允许对 V8 堆内存钻取快照，用于事后分析。

- node-memwatch。来自 Mozilla 的 Lloyd Hilaiel 贡献的模块，采用 WTFPL 许可发布。

由于各种条件限制，这里将只着重介绍通过 node-heapdump 和 node-memwatch 两种方式进行内存泄漏的排查。

### 5.5.1 node-heapdump

想要了解 node-heapdump 对内存泄漏排查的方式，我们需要先构造如下一份包含内存泄漏的代码示例，并将其存为 server.js：

```js
var leakArray = [];
var leak = function () {
  leakArray.push('leak', Math.random());
};

http
  .createServer(function (req, res) {
    leak();
    res.writeHead(200, {
      'Content-Type': 'text/plain',
    });
    res.end('Hello World\n');
  })
  .listen(1337, function () {
    console.log('Server running at http://localhost:1337');
  });
```

在上面这段代码中，每次访问服务器进程将引起 leadArray 数组的元素增加，而且得不到回收。我们可以用 curl 工具输入`http://127.0.0.1:1337`命令来模拟用户访问。

- 安装 node-heapdump

  ```sh
  $ npm install heapdump
  ```

  安装 node-heapdump 后，就可以启动服务进程，并接受客户端的请求。访问多次之后 leakArray 中就会具备大量的元素。这个时候我们通过向服务进程发送 SIGUSR2 信号，让 node-heapdump 抓拍一份堆内存的快照。

  ```sh
  $ kill -USR2 <pid>
  ```

  这份抓取的快照将会在文件目录下以`heapdump-<sec>.<usec>.heapsnapshot`的格式存放。这是一份较大的 JSON 文件，需要通过 Chrome 的开发者工具打开查看。

  在 Chrome 的开发者工具中选中 Profiles 的面板，右击该文件后，从弹出的快捷菜单中选择 Load...选项，打开刚才的快照文件，就可以查看堆内存中的详细信息。

  可以看到大量的 leak 字符串存在，这些字符串就是一直未能得到回收的数据。通过在开发者工具的面板中查看内存分布，我们找到了泄漏的数据，然后根据这个信息找到造成泄漏的代码。

### 5.5.2 node-memwatch

node-memwatch 的用法和 node-heapdump 一样，我们需要准备一份内存泄漏代码。这里就不再赘述 node-memwatch 的安装过程。整个示例代码如下：

```js
var memwatch = require('memwatch');
memwatch.on('leak', function (info) {
  console.log('leak:');
  console.log(info);
});

memwatch.on('stats', function (stats) {
  console.log('stats:');
  console.log(stats);
});

var http = require('http');

var leakArray = [];
var leak = function () {
  leakArray.push('leak' + Math.random());
};

http
  .creatServer(function (req, res) {
    leak();

    res.writeHead(200, { 'Content-Type': 'text/plain' });
    res.end('Hello World\n');
  })
  .listen(1337, function () {
    console.log('Server running at http://127.0.0.1:1337/');
  });
```

#### 1. stats 事件

在进程中使用 node-memwatch 之后，每次进行全堆垃圾回收时，就会触发一次 stats 事件，这个事件将会传递内存的统计信息。在对上述代码创建的服务进程进行访问时，某次 stats 事件打印数据如下：

```js
stats: {
  'num_full_gc': 4, // 第几次全堆垃圾回收
  'num_inc_gc': 23, // 第几次增量垃圾回收
  'heap_compactions': 4, // 第几次对老生代进行整理
  'usage_trend': 0, // 使用趋势
  'estimated_base': 7152944, // 预估基数
  'current_base': 7152944, // 当前基数
  'min': 6720776, // 最小
  'max': 7152944 // 最大
}
```

在这些数据中，`num_full_gc`和`num_inc_gc`比较直观地反应了垃圾回收的情况。

#### 2.leak 事件

如果经过连续 5 次垃圾回收后，内存仍然没有被释放，这意味着有内存泄漏的产生，node-memwatch 会触发一个 leak 事件。某次 leak 事件得到的数据如下所示：

```js
leak: {
  start: 'Mon Oct 07 2013 13:46:27 GMT+0800 (CST)',
  end: 'Mon Oct 07 2013 13:54:40 GMT+0800 (CST)',
  growth: 6222576,
  reason: 'heap growth over 5 consecutive GCs (8m 13s) - 43.33 mb/hr'
}
```

这个数据能现实 5 次垃圾回收的过程中内存增长了多少。

#### 3.堆内存比较

最终得到的 leak 事件的信息只能告知我们应用中存在内存泄漏，具体问题产生在何处还需要从 V8 的堆内存定位。node-memwatch 提供了抓取快照和比较快照的功能，它能够比较堆上对象的名称和分配数量，从而找到导致内存泄漏的元凶。

下面为一段导致内存泄漏的代码，这是通过 node-memwatch 获取堆内存差异结果的示例：

```js
var memwatch = require('memwatch');
var leakArray = [];

var leak = function () {
  leakArray.push('leak' + Math.random());
};

// Take first snapshot
var hd = new memwatch.HeapDiff();

for (var i = 0; i < 100000; i++) {
  leak();
}

// Take the second snapshot and compute the diff
var diff = hd.end();
console.log(JSON.stringify(diff, null, 2));
```

```json
// 执行上述代码，得到结果如下：
{
  "before": {
    "nodes": 11719,
    "time": "2013-10-07T06:32:07.000Z",
    "size_bytes": 1493304,
    "size": "1.42 mb"
  },
  "after": {
    "nodes": 31618,
    "time": "2013-10-07T06:32:07.000Z",
    "size_bytes": 2684864,
    "size": "2.56 mb"
  },
  "change": {
    "size_bytes": 1191560,
    "size": "1.14 mb",
    "freed_nodes": 129,
    "allocated_nodes": 20028,
    "details": [
      {
        "what": "Array",
        "size_bytes": 323720,
        "size": "316.13 kb",
        "+": 15,
        "-": 65
      },
      {
        "what": "Code",
        "size_bytes": -10944,
        "size": "-10.69 kb",
        "+": 8,
        "-": 28
      },
      {
        "what": "String",
        "size_bytes": 879424,
        "size": "858.81 kb",
        "+": 20001,
        "-": 1
      }
    ]
  }
}
```

在上面的输出结果中，主要关注 change 节点下的 freed_nodes 和 allocated_nodes，它们记录了释放的节点数量和分配的节点数量。这里由于有内存泄漏，分配的节点数量远远多于释放的节点数量。在 details 下可以看到具体每种类型的分配和释放数量，主要问题展现在下面这段输出：

```json
{
  "what": "String",
  "size_bytes": 879424,
  "size": "858.81 kb",
  "+": 20001,
  "-": 1
}
```

在上述代码中，加号和减号分别表示分配和释放的字符串数量。可以通过上面的输出结果猜测，有大量的字符串没有被回收。

### 5.5.3 小结

从本节的内容我们可以得知，排查内存泄漏的原因主要通过对堆内存进行分析而找到。node-heapdump 和 node-memwatch 各有所长，读者可以结合它们的优势进行内存泄漏排查。

## 5.6 大内存应用

在 Node 中，不可避免地还是会操作大文件的场景。由于 Node 的内存限制，操作大文件也需要小心，好在 Node 提供了 stream 模块用于处理大文件。

stream 模块是 Node 的原生模块，直接引用即可。stream 继承自 EventEmitter,具备基本的自定义事件功能，同时抽象出标准的事件和方法。它分可读和可写两种。Node 中的大多数模块都有 stream 的应用，比如 fs 的`createReadStream()`和`createWriteStream()`方法可以分别用于创建文件的可读流和可写流，process 模块中的 stdin 和 stdout 则分别是可写流和可读流的示例。

由于 V8 的内存限制，我们无法通过`fs.readFile()`和`fs.writeFile()`直接进行大文件的操作，而还用`fs.createReadStream()`和`fs.createWriteStream()`方法通过流的方式实现对大文件的操作。下面的代码展示了如何读取一个文件，然后将数据写入到另一个文件的过程：

```js
var reader = fs.createReadStream('in.txt');
var writer = fs.createWriteStream('out.txt');
reader.on('data', function (chunk) {
  write.write(chunk);
});

reader.on('end', function () {
  write.end();
});

// 由于读写模式固定，上述方法有更简洁的方式：
var reader = fs.createReadStream('in.txt');
var writer = fs.createWriteStream('out.txt');

reader.pipe(writer);
```

可读流提供了管道方法`pipe()`，封装了 data 事件和写入操作。通过流的方式，上述代码不会收到 V8 内存限制的影响，有效地提高了程序地健壮性。

如果不需要进行字符串层面地操作，则不需要借助 V8 来处理，可以尝试进行纯粹地 Buffer 操作，这不会收到 V8 内存地限制。但是这种大片使用内存地情况依然要小心，即使 V8 不限制堆内存大小，物理内存依然有限制。

## 5.7 总结

Node 将 JavaScript 的主要应用场景扩展到了服务器端，相应要考虑的细节也从浏览器端不同，需要更严谨地为每一份资源做出安排。总的来说，内存在 Node 中不能随心所欲地使用，但也不是完全不擅长。本行介绍了内存地各种限制，希望读者可以在使用中规避禁忌，与生态系统中地各种软件搭配，发挥 Node 的长处。
