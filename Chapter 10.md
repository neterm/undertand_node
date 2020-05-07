# 测试

---

- [10.1 单元测试](#101-单元测试)
- [10.2 性能测试](#102-性能测试)
- [10.3 总结](#103-总结)

---

在使用 Node 进行实际的项目开发之前，我内心也曾十分忐忑。尽管 JavaScript 历史悠久，但相较成熟的后端语言而言，Node 尚且算是新晋同学。甚至对于前端，因为各种各样的原因，JavaScript 的测试都十分少。Node 编写的在线产品，在成千上万用户面签能否具备良好的质量保证，我是心存疑问的。

总最早写处的代码让自己睡不着觉，无法精确定位 bug 到底位于一堆程序里的哪个位置，到后来很踏实地面对自己产出地代码，对自己代码的了解如手心纹路那么清晰明了。从面对问题时的被动到主动，测试在这个演变过程中起到了至关重要的作用。

测试的意义在于，在用户消费产出的代码之前，开发者首先消费它，给予其重要的质量保证。这里值得提醒的是，JavaScript 开发者需要转变观念，正视自己的代码，对于自己产出的代码负责。为自己的代码写测试用例则是一种行之有效的方法，它能够让开发者明确掌握到代码的行为和性能等。

测试包含单元测试、性能测试、安全测试和功能测试等几个方面，本章从 Node 实践的角度来介绍单元测试和性能测试。

## 10.1 单元测试

单元测试在软件项目中扮演者举足轻重的角色，是几种软件质量保证的方法中投入产出比最高的一种。尽管在过去的 JavaScript 开发中，绝大多数人都忽视了这个缓解，但今天 Node 的盛行让我们不得不重新审视这块领域。

### 10.1.1 单元测试的意义

最初接触单元测试时，很多开发者都很疑惑，自己写的代码，自己写测试，这件事的意义何在？有的团队则配备了专门的测试工程师帮助开发者测试代码。这里第一种对自己写的代码不在意的行为时开发者对自己测试自己代码心存侥幸，认为测试是一种形式，小算盘是既然是形式，那为何要去实践。如果强迫实践，那就随意写写，蒙混过关吧，这使得开发者不正是测试代码，进而不正视自己的代码。配备专门的测试工程师则让开发者对测试人员产生依赖，完全不关心自己代码的测试。

这里需要倡导的是，**开发者应该吃自己的狗粮**。项目成员共同开发出来的代码会构成项目的产品，开发者写出来的代码是开发者自己的产品。要保证产品质量，就应该有相应的手段去验证。对于开发者而言，单元测试就是最基本的一种方式。如果开发者不自己测试代码，那必然要面对如下问题：

- 测试工程师是否可依赖？

  这里涉及的问题有两个层面。第一层面是测试工程师是否熟悉 Node 领域，不了解一个领域而只凭借过往经验来对这个项目进行测试，有可能演变为敷衍的行为，这对质量保证的目标背道而驰。另一个层面是，如果存在人事变动等原因，可能并不一定覆盖到开发者的代码，从而使测试用例的维护成本变高。

- 第三方代码是否可信赖？

  对于 Node 开源社区而言（共有 3 万多模块），作为一个不知名的开发者，其产出的模块如果连单元测试都没有提供，使用者在挑选模块时，内心会闪过多个“靠谱吗”的疑问。

- 在产品迭代过程中，如何继续保证质量？

  单元测试的意义在于每个测试用例的覆盖都是一种可能的承诺。如果 API 升级时，测试用例可以很好地检查是否向下兼容。对于各种可能的输入，一旦测试覆盖，都能明确它的输出。代码改动后，可以通过测试结果判断代码的改动是否影响已确定的结果。

对于上述问题，如果你的答案是不关心，那么恭喜你，你的项目只能供短时间玩玩，甚至只是一个演示产品。

另一个对单元测试持疑问的观点是，如果要在项目中进行单元测试，那么势必会影响开发者的项目进度。这个答案是肯定的，因为产出品质可以久经考验的产品，必然要花费较多的经理。如果只是豆腐渣工程，自然可以快速产出。区别在于后续维护的差异，因为有单元测试的质量保证，可以放心地增加和删除功能。后者则会陷入举步维艰地维护之路，拆东墙补西墙，开发者也间件变得只想做新项目，而旧项目最后变得不可维护，或者不敢维护。甚至到项目下线时，依然充斥幽灵代码和重复代码。

单元测试只是在早期会花费一定地成本，但这个成本要远远低于后期深陷维护泥潭的投入。至于选择在早期投入成本还是在后期投入，只是朝三暮四还是朝四目三的选择。

展开介绍单元测试之前，需要提及的问题是代码的可测试性，它是能够为其编写单元测试的前提条件。复杂的逻辑代码充满各种分支和判断，甚至像面条一样乱作一团，要对他们进行测试，难度相当大。一个感觉就是当无法为一段代码写出单元测试时，这段代码必然有坏味道，这会为开发者带来心理压力，这样的代码最需要重构。好代码的单元测试必然是轻量的，重构和写单元测试之间是一个相互促进的步骤，当重构代码的压力比较小的时候，也就意味着代码比较稳定，代码的可测试性越好，甚至代码越简洁。

简单而言，编写可测试代码有以下几个原则可以遵循。

- **单一职责**。如果一端代码承担的职责越多，为其编写单元测试的时候就要构造更多的输入数据，然后推测它的输出。比如，一段代码中既包含数据库的连接，也包含查询，那么为它编写测试用例就要同时关注数据库连接和数据库查询。较好的方式是将这两种职责进行解耦分离，编程两个单一职责的方法，分别测试数据库连接和数据库查询。

- **接口抽象**。通过对程序代码进行接口抽象后，我们可以针对接口进行测试，而具体代码实现的变化不影响为接口编写单元测试。

- **层次分离**。层次分离实际上时单一职责的一种实现。在 MVC 结构的应用中，就是典型的层次分离模型，如果不分离各个层次，无法想象这个代码该如何切入测试。通过分层之后，可以逐渐测试，逐层保证。

对于开发者而言，不仅要编写单元测试，还应当编写可测试代码。

### 10.1.2 单元测试介绍

单元测试主要包含断言、测试框架、测试用例、测试覆盖率、mock、持续集成等几个方面，由于 Node 的特殊性，它还会加入异步代码测试和私有方法的测试这两个部分。

#### 1. 断言

鉴于 JavaScript 入门较为容易，在开源社区中国可以看到许多不带单元测试的模块出现，甚至有的模块作者并不了解单元测试究竟是怎么回事。开发者通常仅仅在 test.js 或者 demo.js 里看到示例代码，这对想进一步使用模块的用户会存在心理负担。以下为某个开源模块的示例代码：

```js
var readOF = require('readof')
readOF.read(pic, target_path, function (error, data) {
  // do something
})
```

此类代码对质量没有任何保证，主要源于以下两点：

- 没有对输出结果进行任何的检测。
- 输入条件覆盖率并不完备。

这样的示例代码展现的是“It works”而不是“Testing”。示例代码可以正常运行并不代表代码是没有问题的。如果对输出结果进行测试，以确认方法调用是正常的，是最基本的测试点。**断言**就是单元测试中用来保证最小单元是否正常的检测方法。

如果对 Node 源码进行过研究，会发现 Node 中存在着 assert 这个模块，以及很多主要模块都调用了这个模块。何谓断言，维基百科上的解释是：

> 在程序设计中，断言（assertion）是一种放在程序中的一阶逻辑（如果一个结果为真或是假的逻辑判断式），目的是为了标示程序开发者预期的结果——当程序运行到断言的位置时，对应的断言应该为真。若断言不为真，程序会中止运行，并出现错误信息。

一言以蔽之，断言用于检查程序在运行时是否满足期望。JavaScript 的断言规范最早来自于 CommonJS 的单元测试规范，Node 实现了规范中的断言部分。

如下代码是 assert 模块的工作方式：

```js
var assert = require('assert')
assert.equal(Math.max(1, 100), 100)
```

一旦`assert.equal()`不满足期望，将会抛出 AssertionError 异常，整个程序将会停止执行。没有对输出结果做任何断言检查的代码，都不是测试代码。没有测试代码的代码，都是不可信赖的代码。

在断言规范中，我们定义了以下几种测试方法：

- ok()：判断结果是否为真。
- equal()：判断实际值与期望值是否相等。
- notEqual()：判断实际值与期望值是否不相等。
- deepEqual()：判断实际值与期望值是否深度相等（对象或数组的元素是否相等）。
- notDeepEqual()： 判断实际值与期望值是否不深度相等。
- strictEqual()：判断实际值与期望值是否严格相等（相当于===）。
- notStrictEqual()：判断实际值与期望值是否不严格相等（相当于!==）。
- throws()：判断代码块是否抛出异常。

除此之外，Node 的 assert 模块还扩充了如下两个断言方法：

- doesNotThrow()：判断代码块是否没有抛出异常。
- ifError()：判断实际值是否为一个假值（null、undefined、0、''、false)，如果实际值为真值，则会抛出异常。

目前，市面上的断言库大多都是给予 assert 模块进行封装和扩展的，这包括著名的 should.js 断言库。

#### 2. 测试框架

前面提到断言一旦检查失败，将会抛出异常停止整个应用，这对于做大规模断言检查时，并不友好。更通用的做法是，记录下抛出异常并继续执行，最后生成测试报告。这些任务的承担着就是测试框架。

测试框架用于测试服务，它本身不参与测试，主要用于管理测试用例和生成测试报告，提升测试用例的开发速度，提高测试用例的可维护性和可读性，以及一些周边行的工作。这里我们要介绍的优秀端元测试框架是 mocha，它来自于 Node 社区的明星开发者 TJ Holowaychuk。通过`npm install mocha`命令即可安装，在安装时添加`-g`命令可以将其全局安装。

- **测试风格**
  我们将测试用例的不同组织方式称为测试风格，现金流行的单元测试风格主要有 TDD（测试驱动开发）和 BDD（行为驱动开发）两种，他们的差别如下：

  - **关注点不同**。TDD 关注所有功能是否被正确实现，每一个功能都具备对应的测试用例；BDD 关注整体行为是否符合预期，适合自顶而下的设计方式。
  - **表达方式不同**。TDD 的表达方式偏向于功能说明书的风格；BDD 的表述方式更接近于自然语言的习惯。

  mocha 对于两种测试风格都有支持。下面为两种测试风格的示例，其 BDD 风格的示例如下：

  ```js
  describe('Array', function () {
    before(function () {
      // ....
    })

    describe('#indexOf()', function () {
      it('should return -1 when not present', function () {
        ;[1, 2, 3].indexOf(4).should.equal(-1)
      })
    })
  })
  ```

  BDD 对测试用例的组织主要采用 describe 和 it 进行组织。describe 可以描述多层级的结构，具体到测试用例时，用 it。另外，它还提供了 before、after、boforeEach 和 afterEach 这 4 个钩子方法，用于协助 describe 中测试用例的准备、安装、卸载和回收等工作。before 和 after 分别在进入和退出 describe 时触发，beforeEach 和 afterEach 则分别在 describe 中每一个测试用例（it）执行前和执行后触发执行。

  TDD 风格的示例如下：

  ```js
  suite('Array', function () {
    setup(function () {
      // ...
    })

    suite('#indexOf()', function () {
      test('should return -1 when not present', function () {
        assert.equal(-1, [1, 2, 3].indexOf(4))
      })
    })
  })
  ```

  TDD 对测试用例的组织主要采用 suite 和 test 完成。suite 也可以实现多层级描述，测试用例用 test。它提供的钩子函数仅包含 setup 和 teardown，对应 BDD 中的 before 和 after。

- **测试报告**

  作为测试框架，mocha 设计得十分灵活，它与断言之间并不耦合，使得具体的测试用例既可以采用 assert 原生模块，也可以采用扩展的断言库，如 should.js、expect 和 chai 等。但无论采用哪个断言库，运行测试用力后，测试报告时开发者和质量管理者都关注的东西。

  mocha 提供了相当丰富的报告格式，调用`mocha --list-reporters`即可查看所有报告格式：

  ```sh
  mocha --list-reporters

    doc         - HTML documentation
    dot         - dot matrix representation
    json        - single JSON object
    json-stream - newline delimited JSON events
    landing     - Unicode landing strip
    list        - like "spec" reporter but flat
    markdown    - GitHub Flavored Markdown
    min         - essentially just a summary
    nyan        - "nyan cat"
    progress    - a progress bar
    spec        - hierarchical & verbose [default]
    tap         - TAP-compatible output
    xunit       - XUnit-compatible XML output
  ```

  默认的报告格式为 dot，其它比较常用的格式为 spec、json、html-cov 等。执行`mocha -R <reporter>`命令即可采用这些报告。json 报告因为其格式非常通用，多用于将结果传递给其它程序进行处理。执行`mocha -help`命令可以看到更多地帮助信息来了解如何使用它们。

#### 3. 测试代码的文件组织

还记得第二章中介绍到的包规范嘛？包规范中定义了测试代码存在于 test 目录中，而模块代码存在于 lib 目录下。

除此之外，想让你的单元测试顺利运行起来，请记得在包描述文件（package.json）中添加相应模块的依赖关系。由于 mocha 只在运行测试时需要，所以添加到 devDependencies 节点即可：

```json
"devDepandencies": {
  "mocha": "*"
}
```

#### 4. 测试用例

介绍完测试框架的基本功能后，我们对测试用例也有了简单的认知了。简单来讲，一个行为或者功能需要完善的、多方面的测试用例，一个测试用例中包含知少一个断言：

```js
describe('#indexOf()', function () {
  it('should return -1 when not present', function () {
    ;[1, 2, 3].indexOf(4).should.equal(-1)
  })

  it('should return index when present', function () {
    ;[1, 2, 3].indexOf(1).should.equal(0)
    ;[1, 2, 3].indexOf(2).should.equal(1)
    ;[1, 2, 3].indexOf(3).should.equal(2)
  })
})
```

测试用例最少需要通过正向测试和反向测试来保证测试对应功能的覆盖，这是最基本的测试用例。对于 Node 而言，不仅有这样简单的方法调用，还有异步代码和超时设置需要关注。

- 异步测试

  由于 Node 环境的特殊性，异步调用非常常见，这也带来了异步代码在测试方面的挑战。在其它典型编程语言中，如 Java、Ruby、Python，代码大多是同步执行的，所以测试用例基本上只要包含一些断言检查返回值即可。但是在 Node 中，检查方法的返回值时含无意义，并不知道回调函数具体何时调用结束，浙江导致我们对异步调用进行测试时，无法调度后续测试用例执行。

  所幸，mocha 解决了这个问题，以下为 fs 模块中的 readFile 的测试用例：

  ```js
  it('fs.readFile should be ok', function (done) {
    fs.readFile('file_path', 'utf8', function (err, data) {
      should.not.exist(err)
      done()
    })
  })
  ```

  在上述代码中，测试用例方法 it()接收两个参数；用例标题（title）和回调函数（fn）。通过检查这个回调函数的形参长度（fn.length）来判断这个用例是否是异步调用，如果时异步调用，在执行测试用例时，会将一个函数 done()注入实参，测试代码需要主动调用这个函数通知测试框架当前测试用例执行完成，然后测试框架才会进行下一个测试用例执行，这与第四章中提到的尾触发十分相似。

* 超时设置

异步方法给测试带来的问题并不是断言方面有什么异同，主要在于回调函数执行的时间无从预期。通过上面的例子，我们无法知道 done()具体在什么时候执行。如果代码偶然出错，导致 done()一直没有执行，将会造成所有的测试用例处于暂停状态，这显然不是框架所期望的。

mocha 给所有涉及异步的测试用例添加了超时限制，如果一个用例的执行时间超过了预期时间，将会记录下一个超时错误，然后执行下一个测试。

下面这个测试用例因为 10s 之后才执行，导致测试框架处理为超时错误：

```js
it('async text', function (done) {
  // 模拟一个要执行很久的异步方法
  setTimeout(done, 10000)
})
```

mocha 的 more 超时时间为 2000ms，一般情况下，通过`mocha -t <ms>`设置所有用例的超时时间。若需要更细粒度地设置超时时间，可以在 cesium 用例 it 中调用`this.timeout(ms)`实现单个用例地特殊设置，示例代码：

```js
it('should take less than 500ms', function (done) {
  this.timeout(500)
  setTimeout(done, 300)
})
```

也可以在描述 describe 中调用`this.timeout(ms)`设置描述当前层级的所有用例：

```js
describe('a suite of tests', function () {
  this.timeout(500)

  it('should take less than 500ms', function (done) {
    setTimeout(done, 300)
  })

  it('should take less than 500ms as well', function (done) {
    setTimeout(done, 200)
  })
})
```

#### 5. 测试覆盖率

通过不停地给代码添加测试用例，将会不断地覆盖代码的分支和不同的情况。但是如何判断单元测试对代码的覆盖情况，我们需要直观的工具来体现。测试覆盖率是单元测试中的一个重要指标，它能够概括性地给出整体覆盖度，也能明确地给出统计到行地覆盖情况。

对于如下代码：

```js
exports.parseAsync = function (input, callback) {
  setTimeout(function () {
    var result

    try {
      result = JSON.parse(input)
    } catch (e) {
      return callback(e)
    }

    callback(null, result)
  }, 10)
}
```

我们为其添加部分测试用例：

```js
describe('parseAsync', function () {
  it('parseAsync should ok', function (done) {
    lib.parseAsync('{"name": "JacksonTian"}', function (err, data) {
      should.not.exist(err)
      data.name.should.be.equal('JacksonTian')
      done()
    })
  })
})
```

若要探知这个测试用例对源代码的覆盖率，需要一种工具来统计每一行代码是否执行，这里要介绍的相关工具是 jscover 模块。通过`npm install jscover -g`的方式来安装该模块。

假设你的这段代码遵循 CommonJS 规范并且存放在 lib 目录下，那么调用`jscover lib lib-cov`进行源代码编译。jscover 会将 lib 目录下的.js 文件编译到 lib-cov 目录下，你会得到类似下面的代码：

```js
/* ****** automatically generated by jscover - do not edit ******/
if (typeof _$jscoverage === 'undefined') {
  _$jscoverage = {}
}
/* ****** end - do not edit ******/
function BranchData() {
  this.position = -1
  this.nodeLength = -1
  this.src = null
  this.evalFalse = 0
  this.evalTrue = 0

  this.init = function (position, nodeLength, src) {
    this.position = position
    this.nodeLength = nodeLength
    this.src = src
    return this
  }

  this.ranCondition = function (result) {
    if (result) this.evalTrue++
    else this.evalFalse++
  }

  this.pathsCovered = function () {
    var paths = 0
    if (this.evalTrue > 0) paths++
    if (this.evalFalse > 0) paths++
    return paths
  }

  this.covered = function () {
    return this.evalTrue > 0 && this.evalFalse > 0
  }

  this.toJSON = function () {
    return (
      '{"position":' +
      this.position +
      ',"nodeLength":' +
      this.nodeLength +
      ',"src":' +
      jscoverage_quote(this.src) +
      ',"evalFalse":' +
      this.evalFalse +
      ',"evalTrue":' +
      this.evalTrue +
      '}'
    )
  }

  this.message = function () {
    if (this.evalTrue === 0 && this.evalFalse === 0)
      return 'Condition never evaluated         :\t' + this.src
    else if (this.evalTrue === 0) return 'Condition never evaluated to true :\t' + this.src
    else if (this.evalFalse === 0) return 'Condition never evaluated to false:\t' + this.src
    else return 'Condition covered'
  }
}

BranchData.fromJson = function (jsonString) {
  var json = eval('(' + jsonString + ')')
  var branchData = new BranchData()
  branchData.init(json.position, json.nodeLength, json.src)
  branchData.evalFalse = json.evalFalse
  branchData.evalTrue = json.evalTrue
  return branchData
}

BranchData.fromJsonObject = function (json) {
  var branchData = new BranchData()
  branchData.init(json.position, json.nodeLength, json.src)
  branchData.evalFalse = json.evalFalse
  branchData.evalTrue = json.evalTrue
  return branchData
}

function buildBranchMessage(conditions) {
  var message = 'The following was not covered:'
  for (var i = 0; i < conditions.length; i++) {
    if (conditions[i] !== undefined && conditions[i] !== null && !conditions[i].covered())
      message += '\n- ' + conditions[i].message()
  }
  return message
}

function convertBranchDataConditionArrayToJSON(branchDataConditionArray) {
  var array = []
  var length = branchDataConditionArray.length
  for (var condition = 0; condition < length; condition++) {
    var branchDataObject = branchDataConditionArray[condition]
    if (branchDataObject === undefined || branchDataObject === null) {
      value = 'null'
    } else {
      value = branchDataObject.toJSON()
    }
    array.push(value)
  }
  return '[' + array.join(',') + ']'
}

function convertBranchDataLinesToJSON(branchData) {
  if (branchData === undefined) {
    return '[]'
  }
  var array = []
  var length = branchData.length
  for (var line = 0; line < length; line++) {
    var branchDataObject = branchData[line]
    if (branchDataObject === undefined || branchDataObject === null) {
      value = 'null'
    } else {
      value = convertBranchDataConditionArrayToJSON(branchDataObject)
    }
    array.push(value)
  }
  return '[' + array.join(',') + ']'
}

function convertBranchDataLinesFromJSON(jsonObject) {
  if (jsonObject === undefined) {
    return []
  }
  var length = jsonObject.length
  for (var line = 0; line < length; line++) {
    var branchDataJSON = jsonObject[line]
    if (branchDataJSON !== null) {
      for (var conditionIndex = 0; conditionIndex < branchDataJSON.length; conditionIndex++) {
        var condition = branchDataJSON[conditionIndex]
        if (condition !== null) {
          branchDataJSON[conditionIndex] = BranchData.fromJsonObject(condition)
        }
      }
    }
  }
  return jsonObject
}
try {
  if (
    typeof top === 'object' &&
    top !== null &&
    typeof top.opener === 'object' &&
    top.opener !== null
  ) {
    // this is a browser window that was opened from another window

    if (!top.opener._$jscoverage) {
      top.opener._$jscoverage = {}
      top.opener._$jscoverage.branchData = {}
    }
  }
} catch (e) {}

try {
  if (typeof top === 'object' && top !== null) {
    // this is a browser window

    try {
      if (typeof top.opener === 'object' && top.opener !== null && top.opener._$jscoverage) {
        top._$jscoverage = top.opener._$jscoverage
      }
    } catch (e) {}

    if (!top._$jscoverage) {
      top._$jscoverage = {}
      top._$jscoverage.branchData = {}
    }
  }
} catch (e) {}

try {
  if (typeof top === 'object' && top !== null && top._$jscoverage) {
    this._$jscoverage = top._$jscoverage
  }
} catch (e) {}
if (!this._$jscoverage) {
  this._$jscoverage = {}
  this._$jscoverage.branchData = {}
}
if (!_$jscoverage['parseAsync.js']) {
  _$jscoverage['parseAsync.js'] = []
  _$jscoverage['parseAsync.js'][1] = 0
  _$jscoverage['parseAsync.js'][2] = 0
  _$jscoverage['parseAsync.js'][3] = 0
  _$jscoverage['parseAsync.js'][5] = 0
  _$jscoverage['parseAsync.js'][6] = 0
  _$jscoverage['parseAsync.js'][8] = 0
  _$jscoverage['parseAsync.js'][10] = 0
}
_$jscoverage['parseAsync.js'].source = [
  'module.exports = parseAsync = function (input, callback) {',
  '    setTimeout(function () {',
  '        var result',
  '',
  '        try {',
  '            result = JSON.parse(input)',
  '        } catch (e) {',
  '            return callback(e)',
  '        }',
  '        callback(null, result)',
  '    }, 10)',
  '}',
]
_$jscoverage['parseAsync.js'][1]++
module.exports = parseAsync = function (input, callback) {
  _$jscoverage['parseAsync.js'][2]++
  setTimeout(function () {
    _$jscoverage['parseAsync.js'][3]++
    var result
    _$jscoverage['parseAsync.js'][5]++
    try {
      _$jscoverage['parseAsync.js'][6]++
      result = JSON.parse(input)
    } catch (e) {
      _$jscoverage['parseAsync.js'][8]++
      return callback(e)
    }
    _$jscoverage['parseAsync.js'][10]++
    callback(null, result)
  }, 10)
}
```

我们看到，每一行原始代码前都有一些`_$jscoverage`的代码出现，它们将会在执行时统计每一行代码被执行了多少次，也即除了统计是否执行外，还统计次数。

在测试代码时，我们通常通过 require 引入 lib 目录下的文件进行测试。但是为了得到测试覆盖率，必须在运行测试用例时执行编译之后的代码。

为了区分这种注入代码和原始代码的区别，我们在模块的入口文件（通常是包目录下的 index.js）中需要做简单的区别，示例代码如下：

```js
module.exports = process.env.LIB_COV ? require('./lib-cov/index') : require('./lib/index')
```

在运行测试代码时，会设置一个 LIB_COV 的环境变量，以此区分测试环境还是正常环境。备妥编译好的代码之后，执行以下命令行即可得到覆盖率的输出结果：

```sh
# 设置当前命令行有效的变量
export LIB_COV=1
mocha -R html-cov > coverage.html
```

在这次测试中，我们用到了 html-cov 报告，它帮我们生成了一张 HTML 页面，具体地标出了哪一行未执行到，整体覆盖率为多少。

单元测试覆盖率方便我们定位没有测试到的代码行。通常我们往往会不经意地遗漏掉一些异常情况地覆盖。构造一个错误的输入可以覆盖错误情况，下面我们为其补足测试用例：

```js
it('parseAsync should throw err', function (done) {
  lib.parseAsync('{"name": "JacksonTian"}}', function (err, data) {
    should.exist(err)
    done()
  })
})
```

再次执行测试用例，我们将得到一个 100%覆盖率的页面。

在使用过程中，也可以使用 json-cov 报告，这样结果数据对其余系统较为友好。事实上，html-cov 报告即采用 json-cov 的数据与模板渲染而成的。

jscover 模块虽然已经够用，但是还有两个问题：

- 它的编译部分时通过 java 实现的，这样环境依赖上多出了 java。
- 它需要编译代码到额外的新目录，这个过程相对比较麻烦。

而 blanket 模块解决了这两个问题，它由纯 JavaScript 实现，编译代码的过程也是隐式的，无须配置额外的目录，对于原模块项目没有额外的侵入。

blanket 与 jscover 的原理基本一致，在实现上有所不同，其差别在于 blanket 将编译的步骤注入在 require 中，而不是去额外编译成文件，执行测试时再去引用编译后的文件，它的技巧在 require 中。

它的配置比 jscover 要简单，只需要在所有测试用例运行之后通过`--require`选项引入它即可：

```sh
mocha --require blanket -R html-cov > coverage.html
```

另一个需要注意的是，在包描述文件中配置 scripts 节点。在 scripts 节点中，pattern 属性用以匹配需要编译的文件：

```json
"scripts": {
  "blanket": {
    "pattern": "eventproxy/lib"
  }
}
```

当在测试文件中通过 require 引入一个文件模块时，它将判断这个文件的实际路径，如果符合这个匹配规则，就对他进行编译。它的编译与 jscover 不同，jscover 需要将文件编译到磁盘上的另一个目录 lib-cov 中。但是 blanket 则不同，它的原理与第二章中讲到的文件模块编译相同。我们知道，对于`.js`文件，Node 会将它的编译逻辑封装在`require.extensions['.js']`中。blanket 正式在这个环节实现了编译，将覆盖率的追踪代码插入到原始代码中，然后再由原始模块处理逻辑进行处理。

使用 blanket 之后，就无须配置环境变量了，也无须根据环境去判断引入哪些代码，所以之前关于环境判断的代码就不再需要了。

#### 6. mock

前面提到开发者常常会遗漏掉一些异常案例，其中相当大一部分原因在于异常的处理情况较难实现。大多数异常与输入数据并无绝对关系，比如数据库的异常调用，除了输入异常外，还有可能时网络异常、全掀异常等非输入数据相关的情况，这相对难以模拟。

再测试领域中，模拟异常其实是一个不小的科目，它有着特殊的名词：mock。我们铜鼓哦伪造被调用方来测试上层代码的健壮性等。

以下面的代码为例，文件系统的异常时绝对不容易呈现的，为了测试代码的健壮性而专程调节磁盘上的全掀等，成本略高：

```js
exports.getContent = function (filename) {
  try {
    return fs.readFileSync(filename, 'utf8')
  } catch (e) {
    return ''
  }
}
```

为了解决这个问题，我们通过伪造`fs.readFileSync()`方法抛出错误来触发异常。同时为了保证该测试用例不影响其它测试用例，我们需要再执行完后还原它。为此，前面提到的`before()`和`after()`钩子函数就派上用场了，相关代码：

```js
describe('getContent', function () {
  var _readFileSync
  before(function () {
    _readFileSync = fs.readFileSync
    fs.readFileSync = function (filename, encoding) {
      throw new Error('Mock readFileSync error')
    }
  })

  // it()

  after(function () {
    fs.readFileSync = _readFileSync
  })
})
```

我们在执行测试用例前将引用替换掉，执行结束后还原它。如果每个测试用例执行前后都要进行设置和还原，就是用`beforeEach()`和`afterEach()`这两个钩子函数。

由于 mock 的过程比较繁琐，这里推荐一个模块来解决此事——muk，示例代码如下：

```js
var fs = require('fs')
var muk = require('muk')

before(function () {
  muk(fs, 'readFileSync', function (path, encoding) {
    throw new Error('Mock readFileSync error')
  })
})

// it()

after(function () {
  muk.restore()
})

// 当有多个测试用例时，相关代码：
var fs = require('fs')
var muk = require('muk')

beforeEach(function () {
  muk(fs, 'readFileSync', function (path, encoding) {
    throw new Error('Mock readFileSync error')
  })
})

// it()
// it()

afterEach(function () {
  muk.restore()
})
```

模拟时无须临时缓存正确引用，用例执行结束后调用`muk.restore()`恢复即可。通过模拟底层方法出现异常的情况，现在只要检测调用方的输出值是否符合期望即可，无须关注是否是真正的异常。模拟异常可以很大程度地帮助开发者提升代码的健壮性，完善调用方代码的容错能力。

值得注意的一点，对于异步方法的模拟，需要十分小心是否将异步方法模拟为同步，下面的 mock 方式可能会引起意外的结果：

```js
fs.readFile = function (filename, encoding, callback) {
  callback(new Error('Mock readFile error'))
}
```

正确的 mock 方式是尽量让 mock 后的行为与原始行为保持一致，相关代码：

```js
fs.readFile = function (filename, encoding, callback) {
  process.nextTick(function () {
    callback(new Error('Mock readFile error'))
  })
}
```

模拟异步方法时，我们调用`process.nextTick()`使得回调方法能够异步执行即可。关于`process.nextTick()`的原理，在第三章中有所阐述，此处不做更多解释。

#### 7. 私有方法的测试

对于 Node 而言，有一个难点会出现在单元测试的过程中，那就是私有方法的测试，这在第二章中介绍过。只有挂载在 exports 或 module.exports 上的变量或方法才能被外部的 require 引入访问，其余方法只能在模块内部被调用和访问。

在 Java 一类语言里，私有方法的访问可以通过反射的方式实现。那么 Node 该如何实现呢？是否可以因为它们时私有方法就不用为它们添加单元测试？

答案是否定的，为了应用的健壮性，我们应该尽可能地给方法添加测试用例。那么除了将这些私有方法通过 exports 导出外，还有别的方法吗？答案是肯定地。rewire 模块提供了一种巧妙地方式实现对私有方法的访问。

rewire 的调用方式和 require 十分类似。对于如下的私有方法，我们获取它并为其执行测试用例非常简单：

```js
var limit = function (num) {
  return num < 0 ? 0 : num
}

// 测试用例如下：
it('limit should return success', function () {
  var lib = rewire('../lib/index.js')
  var limit = lib.__get__('limit')
  limit(10).should.be.equal(10)
})
```

rewire 的诀窍在于它引入文件时，像 require 一样对原始文件做了一定的手脚。除了添加`(function(exports, require, module, __filename, __dirname){});`的头尾包装外，还注入了部分代码：

```js
(function (exports, require, module, __filename, __dirname) {
  var method = function () {}
  exports.__set__ = function (name, value) {
    eval(name " = " value.toString())
  }

  exports.__get__ = function (name) {
    return eval(name)
  }
})

```

每一个被 rewire 引入的模块都有`__set__()`和`__get__()`方法。它巧妙地利用了闭包地诀窍，在`eval()`执行时，实现了对模块内部局部变量的访问，从而可以将局部变量导出给测试用例调用执行。

### 10.1.3 工程化与自动化

Node 以及第三方模块提供的方法都相对于偏底层，在开发项目时，还需要一定的工具来实现工程化和自动化（这里我们介绍其中一种方式——持续集成），以减少手工成本。

#### 1. 工程化

Node 在\*nix 系统下可以很好地利用一些成熟工具，其中 Makefile 比较小巧灵活，适合用来构建工程。

下面时我们常用的 Makefile 文件内容：

```makefile
TESTS =test/*.js
REPRTER = spec
TIMEOUT = 10000
MOCHA_OPTS =

  test:
    @NODE_ENV=test ./node_modules/mocha/bin/mocha \
      --reporter $(REPORTER) \
      --timeout $(TIMEOUT) \
      $(MOCHA_OPTS) \
      $(TESTS)

  test-cov:
    @$(MAKE) test MOCHA_OPTS='--require blanket' REPORTER=html-cov > coverage.html

  test-all: test test-cov

  .PHONY: test
```

开发者改动代码后，只需要通过`make test`和`make test-cov`命令即可执行复杂的单元测试和覆盖率。这里需要注意以下两点：

- Makefile 文件的缩进必须是 tab 符号，不能用空格。
- 记得在包描述文件中配置 blanket

#### 2. 持续集成

将项目工程化可以帮助我们把项目组织成比较固定的结构，以供扩展。但是对于实际的项目而言，频繁地迭代时常见的状态，如何记录版本的迭代信息，还需要一个持续集成的环境。

至于如何持续集成，各个公司都有自己特定的方案，这里介绍下社区中比较流行的方式——利用 travis-ci 实现持续集成。

travis-ci 与 GitHub 的配合可谓相得益彰。GitHub 提供代码托管和社交编程的良好环境，程序员们可以在上面很社交化地进行代码的 clone、fork、pull、request、issues 等操作，travis-ci 则补足了 GitHub 在持续集成方面的缺点。Git 版本控制系统提供了 hook 机制，用户在 push 代码后，会触发一个 hook 脚本，而 travis-ci 即是通过这种方式与 GitHub 衔接起来的。将你的代码与 travis-ci 链接起来十分容易，只需要如下几步：

1. 在`https://travis-ci.org`上通过 OAuth 授权绑定你的 GitHub 账号。
2. 在 GitHub 仓库的管理面板（admin）中打开 services hook 页，在这个页面中可以发现 GitHub 上提供了很多基于 git hook 方式的钩子服务。
3. 找打 travis 服务，点击激活即可。
4. 每次将代码 push 到 GitHub 仓库后，将会触发该钩子服务。

除此之外，一旦绑定了 GitHub 之后，也可以通过 travis-ci 的管理界面来设置哪些代码仓库开启持续集成服务。

travis-ci 除了提供简单的语言运行时环境外，还提供数据库服务、消息队列、无界面浏览器等，十分强大，值得深度利用。需要注意的一点是，travis-ci 是基于 Ruby 创建的项目，最开始是为 Ruby 项目服务的，目前提供了许多后端语言的测试持续集成服务，但是它会将项目默认当作 Ruby 项目。为了解决该问题，需要在自己项目中提供一个`.travis.yml`说明文件，告之 travis-ci 是哪种类型项目。Node 项目的说明文件如下：

```yaml
language: node_js
node_js:
  - '0.8'
  - '0.10'
```

其中主要有两个说明，language 和支持的版本号。travis-ci 在收到 GitHub 的通知后，将会 pull 最新的代码到测试机中，并根据配置文件准备对应的环境和版本。还记得第二章中提到的 scripts 描述么？前面的 blanket 的配置就在这个节点上。这里 travis-ci 将会执行`npm test`命令来启动整个测试，而前面提到的`mocha -R spec`或`make test`命令应当配置在`package.json`文件中：

```json
"scripts": {
  "test": "make test"
}
```

travis-ci 提供了一个测试状态的服务。在 GitHub 上，也经常会看到此类图标：`passing`或者红色失败图标`failing`。它就是由 travis-ci 提供的项目状态服务，由如下格式组成：

```
https://traivs-ci.org/<username>/<repo>.png?branch=<branch>
```

该图标能够实时反应出项目的测试状态。passing 状态的图标能够在使用者调研模块时增加使用当前模块的信心。

travis-ci 除了提供状态服务外，还详细记录了每次测试的详细报告和日志，通过这些信息我们可以跟踪项目的迭代健康状态。

### 10.1.4 小结

在这一节中，我们介绍了普通的单元测试的方方面面，对于一些特定场景下的单元测试方式并未做过多介绍，比如测试 Web 应用等，读者可以自行查看所用 Web 框架的测试方式，比如 Connect 或 Express 提供了`supertest`辅助库来简化单元测试的编写。

在项目中经常会因为依赖方的变化产生业务代码的跟随变动，如果没有单元测试的覆盖，依赖方逻辑发生变化后，很难定位该变动影响的范围。一旦为项目覆盖完善的单元测试，项目的状态将会因为测试报告而了然于心。完善的单元测试在一定程度上也昭示着项目的成熟的。

## 10.2 性能测试

单元测试主要用于检测代码的行为是否符合预期。在完成代码的行为检测后，还需要对已有代码的性能作为评估，检测已有功能是否能满足生产环境的性能要求，能否承担实际业务中带来的压力。换句话说，性能也是功能。

性能测试的范畴比较广泛，包括负载测试、压力测试和基准测试等。由于这部分内容并非 Node 特有，为了收敛范围，这里将只会简单介绍下基准测试。

除了基准测试，这里还将介绍如何对 Web 应用进行网络层面的性能测试和业务指标换算。

### 10.2.1 基准测试

基本上，每个开发者都具备为自己的代码写基准测试的能力。基准测试要统计的就是在多少时间内执行了多少次某个方法。为了增强可比性，一般会以次数作为参照物，然后比较时间，以此来判别性能的差距。

加入我们要测试 ECMAScript 5 提供的 Array.prototype.map 和循环提取值两种方式，它们都是迭代一个数组，根据回调函数执行的返回值得到一个新的数组，相关代码：

```js
var nativeMap = function (arr, callback) {
  return arr.map(callback)
}

var customMap = function (arr, callback) {
  var ret = []
  for (var i = 0; i < arr.length; i++) {
    ret.push(callback(arr[i], i, arr))
  }

  return ret
}
```

比较简单直接地方式就是构造相同的输入数据，然后自行相同的次数，最后比较时间。为此，我们可以写一个方法来执行这个任务，具体如下：

```js
var run = function (name, times, fn, arr, callback) {
  var start = new Date().getTime()
  for (var i = 0; i < times; i++) {
    fn(arr, callback)
  }
  var end = new Date().getTime()

  console.log('Running %s %d times const %d ms', name, times, end - start)
}

// 分别调用 1000000次：
var callback = function (item) {
  return item
}

run('nativeMap', 1000000, nativeMap, [0, 1, 2, 3, 4, 5, 6], callback)
run('customMap', 1000000, customMap, [0, 1, 2, 3, 4, 5, 6], callback)
```

得到结果如下：

```sh
Running nativeMap 1000000 times cost 873ms
Running customMap 1000000 times cost 112ms
```

在我的机器上测试结果显示 Array.prototype.map 执行相同的任务，要花费 for 循环方式 7 倍左右的时间。

上面就是进行基准测试的基本方法。为了得到更规范和更好的输出结果，这里介绍 benchmark 这个模块是如何组织基准测试的：

```js
var Benchmark = require('benchmark')
var suite = new Benchmark.Suite()

var arr = [0, 1, 2, 3, 4, 5, 6]
suite
  .add('nativeMap', function () {
    return arr.map(callback)
  })
  .add('customMap', function () {
    var ret = []
    for (var i = 0; i < arr.length; i++) {
      ret.push(callback[arr[i]])
    }
    return ret
  })
  .on('cycle', function (event) {
    console.log(String(event.target))
  })
  .on('complete', function () {
    console.log('Fastest is ' + this.filter('fastest').pluck('name'))
  })
  .run()
```

它通过 suite 来组织每组测试，在测试套件中调用`add()`来添加被测试的代码。
执行上述代码，得到结果如下：

```sh
nativeMap x 1,227,341 ops/sec ±1.99% (83 runs sampled)
customMap x 7,919,649 ops/sec ±0.57% (96 runs sampled)
Fastest is customMap
```

benchmark 模块输出的结果与我们用普通方式进行测试多出 ±1.99% (83 runs sampled) 这么一段。事实上，benchmark 模块并不是简单地统计执行多少次测试代码后对比时间，它对测试有着严密地抽样过程。执行多少次方法取决于采样到地数据能否完成统计。83 runs sampled 表示对 nativeMap 测试地过程中，由 83 个样本，然后我们根据这些样本，可以推算出标准方差，即 ±1.99%这部分数据。

### 10.2.2 压力测试

除了可以对基本地方法进行基准测试外，通常还会对网络接口进行压力测试以判断网络接口的性能，这在 6.4 节演示过。对网络接口做压力测试需要考察的几个指标有吞吐率、响应时间和并发数，这些指标反映了服务器的并发处理能力。

最常用的工具是 ab、siege、http_load 等，下面我们通过 ab 工具来构造压力测试，相关代码：

```sh
ab -c 10 -t 3 http://localhost:8001/

This is ApacheBench, Version 2.3 <$Revision: 1430300 $>
Copyright 1996 Adam Twiss, Zeus Technology Ltd, http://www.zeustech.net/
Licensed to The Apache Software Foundation, http://www.apache.org/

Benchmarking 192.168.2.188 (be patient)
Finished 3532 requests


Server Software:
Server Hostname:        192.168.2.188
Server Port:            1337

Document Path:          /
Document Length:        12 bytes

Concurrency Level:      10
Time taken for tests:   3.002 seconds
Complete requests:      3532
Failed requests:        0
Write errors:           0
Total transferred:      399342 bytes
HTML transferred:       42408 bytes
Requests per second:    1176.51 [#/sec] (mean)
Time per request:       8.500 [ms] (mean)
Time per request:       0.850 [ms] (mean, across all concurrent requests)
Transfer rate:          129.90 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0    3   2.8      3      35
Processing:     2    5   3.3      4      38
Waiting:        0    3   2.8      3      29
Total:          4    8   4.2      7      40

Percentage of the requests served within a certain time (ms)
  50%      7
  66%      7
  75%      8
  80%      8
  90%     10
  95%     19
  98%     24
  99%     26
 100%     40 (longest request)
```

上述命令表示 10 个并发用户持续 3s 向服务器端发出请求。下面简要介绍上述代码中各个参数的含义。

- Document Path：表示文档的路径，此处为`/`。
- Document Length：表示文档的长度，就是报文的大小，这里有 10kb。
- Concurrency Level：并发级别，就是我们在命令中传入的 c，此处为 10，即 10 个并发。
- Time taken for tests：表示完成所有测试所花费的时间，它与命令行中传入的 t 选项有细微出入。
- Complete requests：表示在这次测试中以供完成多少次请求。
- Failed request：表示其中产生失败的请求书，这次测试中没有失败请求。
- Write errors：表示在写入过程中出现的错误次数（连接断开导致）。
- Total Transferred：表示所有的报文大小。
- HTML Transferred：表示仅 HTTP 报文的正文大小，它比上一个值小。
- Requests per second： 这是我们重点关注的一个值，它表示服务器每秒能处理多少请求，是重点反应服务器并发能力的指标，这个值又称为 RPS 或 QPS。
- 两个 Time per request 值：第一个代表的是用户平均等待时间，第二个代表的是服务器平均请求处理时间，前者初一并发数得到后者。
- Transfer rate：表示传输率，等于传输的大小除以传输时间，这个值受网卡的带宽限制。
- Connection Times：连接时间，它包括客户端向服务器端简历连接、服务器端处理请求、等待报文响应的过程。

最后的数据是请求的响应时间分布，这个数据是 Time per request 的实际分布。可以看出，50%的请求都在 7ms 内完成，99%的请求都在 26ms 内返回。

### 10.2.3 基准测试驱动开发

Felix Geisendorfer 是 Node 早期的一个代码贡献者，同时也是优秀模块的作者，其中最著名的为他的几个 MySQL 驱动，以追求性能著称。他在“Faster than C”幻灯片中提到一种他所使用的开发模式，简称也是 BDD，全称为 Benchmark Driven Development，即基准测试驱动开发，其中主要分为如下几步流程：

1. 写基准测试。
2. 改/写代码。
3. 收集数据。
4. 找出问题。
5. 回到第 2 步。

之前测试的服务器端脚本运行在单个 CPU 上，为了验证 cluster 模块是否有效，我们可以参照他的方法进行迭代。通过上面的测试，我们已经完成了一遍上述流程。接下来我们回到第 2 步，看看性能是否有提升。

原始代码无无需任何更改，下面我们新增一个 cluster.js 文件，用于根据机器上的 CPU 数量启动多进程来进行服务：

```js
var cluster = require('cluster')

cluster.setupMaster({
  exec: 'server.js',
})

var cpus = require('os').cups()

for (var i = 0; i < cpus.length; i++) {
  cluster.fork()
}

console.log('Start ' + cpus.length + ' workers.')
```

使用相同的参数测试，根据结果判断启动多个进程是否是行之有效的方法。从测试结果看来，QPS 从原来的 3857.60 变成了 4699.53，这个结果显示性能并没有与 CPU 的数量成线性增长，这个问题我们暂不排查，但是它已验证了我们的改动确实能够提升性能。

### 10.2.4 测试数据与业务数据的转换

通常，在进行实际的功能开发之前，我们需要评估业务量，以便功能开发完成后能够胜任实际的在线业务量。如果用户量只有几个，每天的 PV 只有几十个，那么网站开发几乎不需要什么优化就能胜任。如果 PV 上 10w 甚至百万、千万，就需要运用性能测试来验证是否能满足实际业务需求，如果不满足，就要运用各种优化手段提升服务能力。

假设某个页面每天的访问量为 100w，根据实际业务情况，主要访问量大致集中在 10 个小时以内，那么换算公式就是：

```
QPS = PV / 10h
```

100w 的业务访问量换算为 QPS，约等于 27.7，即服务器需要每秒处理 27.7 个请求才能胜任业务量。

## 10.3 总结

测试时应用或者系统最重要的质量保证手段。有单元测试实践的项目，必然对代码的粒度和层次都掌握较好。单元测试能够保证项目每个局部的正确性，也能够在项目迭代过程中很好地监督和反馈迭代质量。如果没有单元测试，就如同黑夜里没有秉烛的行走。

对于性能，在编码过程中一定存在部分感性认知，与实际情况有部分偏差，而性能测试则能很好地斧正这种差异。
