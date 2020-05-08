# 搭建局域网 NPM 仓库

---

- [D.1 NPM 仓库的安装](#d1-npm-仓库的安装)
- [D.2 高阶应用](#d2-高阶应用)
- [D.3 总结](#d3-总结)

---

第二章提到了 NPM，它由现今 Node 的掌门人 Isaac Z. Schelueter 创建。最初，NPM 与 Node 各自发展，在 Node v0.6.3 时，它成为 Node 的一部分。NPM 的出现完善了 Node 模块的整个生态链，让第三方模块更为易用，让依赖管理成为很轻松的事情，促进整个生态圈良性发展。如今，在 GitHub 上托管源代码，在 NPM 上发布模块，在代码中使用第三方方模块包，这三者形成 Node 应用的闭环。这在开源社区中是极度流行的模式。

但是在开源社区中极度适合的应用模式并不一定适合一些企业内部。目前，在官方 NPM 上还存在一些问题，比如体现在如下几个方面：

- 模块质量良莠不齐。
- 私有模块保密、共享、安装和更新的问题。
- 版本控制存在风险。
- 模块安装速度无法保障。

对于企业应用而言，它们更看重稳定和质量。社区中模块数量非常多，不乏很多优秀的模块，但是大部分模块的质量仍然不合格，企业在使用时需要考量其安全性。

对于企业而言，企业自行编写的模块处于保密等考量，无法将模块发布到公共的 NPM 平台上，这对私有模块的共享、安装和更新都造成了应用层面上的困扰。

NPM 允许通过添加`--force`进行强制发布，尽管它会发出警告，但是对于控制权不在自己手中的模块，覆盖性发布可能造成无法预料的风险。模块可能在两次安装之间版本号相同，但是内容其实已经不同了，这带来的风险是相当不可控的。

另外，公共的 NPM 仓库是托管在 Iris Couch 的云平台上，服务并没有对中国的网络环境进行优化，曾经一度受到一些网络环境带来的影响，无法保证稳定性。

上述这些原因都促使企业应当有自己的局域 NPM 仓库。为此，Node v0.10.0 发布时，Isaac Z. Schlueter 提到 Iris Couch 基于运营 NPM 公共仓库的经验，他们团队为此推出了 irisnpm 服务来运行私有 NPM 仓库。通过在 irisnpm 站点上注册可以申请该服务。除了使用 irisnpm 的服务外，我们还可以自行搭建 NPM 仓库。自行搭建 NPM 仓库，可以实现企业内部仓库和社区公共仓库之间的隔离，一方面可以杜绝上述问题的发生，一方面可以享受 NPM 工具带来的生态链的完整性和便捷性。

在 package.json 中编写依赖，通过 NPM 工具从私有仓库中安装模块，自动完成依赖模块的安装，这与使用开源社区的官方仓库一样便利。如果没有私有 NPM 仓库，共享模块的过程甚至演变未复制粘贴的手工活，代码维护成本略高。

## D.1 NPM 仓库的安装

NPM 仓库的源代码托管在 GitHub 上，地址为`https://github.com/isaacs/npmjs.org`。相对于命令行中执行 NPM 命令，NPM 仓库是存放模块的服务器。NPM 仓库的涉及基于 CouchDB 实现。CouchDB 是一款 NoSQL 数据库，基于文档设计，它的文档带有版本性质，同时暴露的 HTTP RESTful 接口十分好用，这与 Node 的模块具有较为相似的特性。Isaac Z. Schlueter 正是在这个基础考虑用它实现模块的托管。有趣的是，作为常拿来与 Node 在网络并发方面进行比较的 Erlang 语言，看似竞争者的关系，其实在此处是有交集的。因为 CouchDB 基于 Erlang 写成，而 NPM 仓库用它来托管模块。

NPM 仓库主要由两部分组成，体现在源代码中分别是`www`和`registry`。`www`是 NPM 站点的界面，`registry`则是利用 CouchDB 存储模块包文件和提供 JSON API，面向 NPM 站点和 NPM 命令行工具服务。

由于在 CouchDB 中构建 Web 应用较为复杂，后来 Isaac Z. Schlueter 重新构建了一个新的 NPM 的 Web 应用，用来替代 CouchDB 提供的 Web 应用服务，让 CouchDB 做纯粹的数据托管并提供 HTTP RESTful 服务。这个新的 NPM Web 应用就是`new www`应用，其源代码在`https://github.com/isaacs/npm-www`中。

### D.1.1 安装 Erlang 和 CouchDB

安装 NPM 仓库所以来的环境比较复杂，对于 Windows 平台而言，可以找到编译好的 Erlang 和 CouchDB 的二进制版本。对于 Linux 或 Mac 用户，这里需要说明一下。

#### 1. 安装 Erlang

安装 Erlang 的命令如下：

```sh
wget http://www.erlang.org/download/otp_src_R15B01.tar.gz
tar zxvf otp_src_R15B01.tar.gz
cd otp_src_R15B01
./configure
make & sudo make install
```

#### 2. 安装 CouchDB

在有 Erlang 环境的情况下，CouchDB 才能被安装。安装步骤跟 Erlang 差别不大，相关命令如下：

```sh
wget http://mirror.bit.edu.cn/apache/couchdb/releases/1.2.0/apache-couchdb-1.2.0.tar.gz
tar zxvf apache-couchdb-1.2.0.tar.gz
cd apache-couchdb-1.2.0
./configure --prefix=/home/admin/couchdb # 考虑磁盘空间的因素，选择适合的目录
make & sudo make install
```

上述需要考虑的是如果仓库中存在大量模块，将会占用较多的磁盘空间，所以谨慎选择要存放的目录，在执行`./configure`时设置或者安装完成后设置配置文件。

CouchDB 的安装还要依赖 Mozilla 的 SpiderMonkey 来执行一些 JavaScript 代码，它的安装命令如下：

```sh
wget http://ftp.mozilla.org/pub/mozilla.org/js/js185-1.0.0.tar.gz
tar zxvf js185-1.0.0.tar.gz
cd js-1.8.5/js
autoconf-2.13
./configure
make & make install
```

#### 3. 启动 CouchDB 服务

启动 CouchDB 服务的命令如下：

```sh
sudu couchdb &
curl -i http://127.0.0.1:5984/ # 查看服务是否启动正确
```

### D.1.2 搭建 NPM 仓库

在前述工作就绪之后，我们就可以搭建 NPM 仓库了，这一步需要 CouchDB 一直启动作为服务。搭建 NPM 仓库主要包含如下：

1. 创建 NPM 数据库。首先我们需要调用 CouchDB 的接口为仓库创建一个数据库，之后所有的模块包文件经作为附件保存在数据库中。

```sh
curl -X PUT http://127.0.0.1:5984/registry
{"ok": true}
```

除此之外，还需要获取 NPM 仓库服务器的源代码。

2. 获取 NP 仓库源代码：

```sh
git clone https://github.com/isaacs/npmjs.org.git
cd npmjs.org
```

3. 获取安装工具：

```sh
sudo npm install couchapp -g
npm install couchapp
npm install semver
```

4. 装在 NPM 仓库到 CouchDB 中：

```sh
couchapp push registry/app.js http://127.0.0.1:5984/registry
couchapp push www/app.js http://127.0.0.1:5984/registry
```

上述步骤分别将 registry 代码和 www 下的代码放进 CouchDB 的 rigistry 库中。一个本地的 NPM 仓库就此搭建完成了。访问`http://127.0.0.1:5984/registry/_design/ui/_rewrite`，可以看到 NPM 仓库的 WebUI 界面。访问`http://127.0.0.1:5984/registry/_design/scratch/_rewrite`，则对应的时 JSON API，面向 NPM 站点和 NPM 命令行工具服务。

这两个 URL 地址相对而言比较难己住，可以在 CouchDB 前面假设反向代理，使得 URL 变得优雅，比如`http://search.npm.your_domain.com/`和`http://registry.npm.your_domain.com/`，这样可以隐藏路径和端口，有一个容易记住的二级域名即可。除此之外，更改 CouchDB 的配置，也可以达到这个效果。

> 注意：
>
> - 默认安装 CouchDB 后，将会监听 127.0.0.1 这个地址，这会导致只有当前机器可以访问 CouchDB 服务，改为 0.0.0.0 则可以被外部机器访问到。
> - 访问`http://127.0.0.1:5984/registry/_design/scratch/_rewrite`将可能得到`insecure_rewrite_rule too many ../.. segments`这样的错误，修改 CouchDB 配置中的`secure_rewrites`为 false 可以解决问题

5. 配合 NPM 客户端。任意需要从本地 NPM 仓库进行操作的命令，只要加入`--registry=http://127.0.0.1:5984/registry/_design/scratch/_rewrite`即可。比如：

```sh
npm install plusplus --registry=http://127.0.0.1:5984/registry/_design/scratch/_rewrite
```

为了解决命令行过长不容易牢记的问题，可以使用如下方法：

```sh
npm config set registry http://127.0.0.1:5984/registry/_design/scratch/_rewrite
```

这个方法的一个问题在于，如果经常需要在官方仓库和本地仓库切换，那就比较麻烦。为此，我们可以利用 bash 中的 alias 功能来解决这个问题。在`~/.bashrc`或`~/.profile`文件的结尾处添加如下：

```sh
alias lnpm='npm --registry=http://127.0.0.1:5984/registry/_design/scratch/_rewrite'
```

重新启动命令行，npm 操作的时官方仓库，lnpm 操作的则是本地仓库。其余参数和命令均相同。

## D.2 高阶应用

在上述过程中，我们完成了一个 NPM 仓库的搭建。我们可以将这个本地仓库用作镜像仓库，也可以用作自己全新的仓库。

### D.2.1 镜像仓库

镜像仓库，完全是官方仓库的一个镜像地址，我们可以通过同步的方式将官方公共仓库中的模块包完全同步到经想仓库中来。经想仓库可以解决安装过程中速度问题，稳定性可以得到保障。但是一个新的问题是要跟官方公共仓库保持同步，否则仓库中会出现落后于官方模块的情况。

由于 NPM 仓库实质上就是一个 CouchDB 数据库，同步官方仓库到经想仓库其实就是对官方数据库的复制。这个复制过程可以采用 CouchDB 自己的复制功能完成，它的实质是增量同步的功能。我尝试过很多次，由于网络问题，整体的复制性能十分低效。Node 社区的 Mikeal Rogers（request 模块的作者、NodeConf 大会组织者）写了一个 replicate 模块用来同步工作。该模块的安装命令如下：

```sh
sudo npm install -g replicate
```

下面的命令可以实现从目标 CouchDB 库同步文档到另一个 CouchDB 库中。对于公共仓库而言，它的地址是：`http://isaacs.iriscouch.com/registry/`。它的原理是调用 CouchDB 的`/_changes`接口，获取仓库源的变动细节，将其提交给目标库的`/_missing_revs`接口，得到目标库确实那些文档（也就是模块包），然后逐个同步缺失的文档。

```sh
replicate http://admin:pass@somecouch/sourcedb http://admin:pass@somecouch/destinationdb
```

如果想持续性地同步模块到镜像仓库中，可以通过 crontab 定时任务来实现。

上述的问题依然是网络问题，可能会导致终端，而截至目前官方模块有 3w 多个，更新次数达 55w 次，完全同步是一个不小的工程。

### D.2.2 私有模块应用

实现镜像仓库后，如果将这个镜像仓库用于生产，它能解决前面提到的 4 个问题中的私有模块和网络稳定性影响安装速度这两个问题。我们可以通过 NPM 工具设置 registry 的方式来使用镜像仓库，甚至发布企业自己的私有模块到私有仓库中，完美解决企业担心的隐私问题，但还不能解决的问题是模块质量和版本控制中存在的风险。

我曾经尝试过两种方案，一种是上述的将所有模块同步到自有仓库中，然后混合公司私有模块的方式进行使用。

在这个案例中，我们通过一个镜像仓库来进行隐私隔离，将私有模块发布到镜像仓库中。对于业务逻辑不想管的模块，我们可以发布到共有 NPM 仓库中，回馈到开源社区。我们相信绝大多数企业也是通过这种方式来进行 Node 开发的。这个模式中，我们可以看到 NPM 平台上为何能有越来越多的高质量模块。企业在享受开源的过程中也不断地回馈开源社区。相比单兵作战，企业产出的模块的质量可能更高，因为这个模块多数已经被企业自己使用和实践过。

### D.2.3 纯私有仓库

镜像仓库加私有模块的模式已经能够让企业最担心的稳定性和隐私性问题得以解决，但是版本发布可覆盖造成的风险和模块质量的问题还不能得到解决，我们一股脑地将所有模块都拖入到我们地企业生产环境中，对于我们解决质量问题丝毫没有帮助。相反，拖进来地模块没有得到挑选和审核。再者，NPM 平台上众多的模块，真正能够用到的不足十分之一。另外，由于是在企业内部使用这些模块，并不需要对公众开放。因此，我们可以尝试进行应用上的更改，彻底解决担心的所有问题。

由于我们并不需要同步所有的模块，所以我们尝试在全量同步这里进行改造。在这个环节，我们加入审核机制，从全量同步改为按需同步。

在这个改造过程中，也需要对工具链进行改造。按需同步只要同步指定的模块即可，对于依赖的模块，我们可以设置模式以选择是否同步依赖的模块。

#### 1. 按需同步

为了完成按需同步的需求，我在 replicate 工具的 基础上进行了改造，编写了`sync_package`模块。它的使用方式：

```sh
npm install sync_package -g
npm config set remote_registry http://isaacs.iriscouch.com/registry/
# 因为本地仓库的写入权限，所以记得加上口令
npm config set local_registry http://username:password@ip/registry/
sync_package express # 同步express模块
```

这个工具之同步指定的模块，远比 replicate 快，能够迅速完成所需模块的同步。默认情况下，这会同时同步依赖的所有模块。加`-D`可以取消同步依赖模块：

```sh
sync_package express -D
```

sync_package 模块的原理是对比源库中的文档信息和目标库中的文档信息，如果不同，则将源库中的模块同步到目标库。实现这个过程的接口是`/module_name?revs_info=true`，它将去除文档的详细信息用于对比。

其中源库和目标库的设置在前面代码中，通过 NPM 工具可以设置。

#### 2. 审查机制

实现了按需同步之后，还需要对这个同步过程加入审核机制。审核的目的在于确认是否应该同步该模块，这个模块的质量和安全性是否得到认可。这个过程就是对模块的挑选过程，通过审核，可以很好地杜绝低质量的模块进入我们生产环境。

要完成审核机制，关键在于控制同步模块的全掀。我们将隐藏私有仓库的写入密码，通过一个 Web 系统来进行管理，除了管理员外，其余开发人员没有必要知道该密码。也就是说，我们将按需同步的功能作为一个触发性功能，审核成功后自动按需同步。

同步模块包的过程对于请求同步的人来说处于黑盒环境，审核通过即可进行同步，同步过程所需要的密码只需在开始时由管理员配置好即可。

#### 3. 二方模块

通过审核机制可以很好地处理第三方模块包的同步问题，接下来，要处理的是企业自己的私有模块。在企业环境中，模块应当属于那个团队而非个人，因为个人可能存在转岗、跳槽等行文，不能像公共社区模块那样自行通过`npm adduser`注册账号来完成模块的发布。为此，可以在 Web 系统中实现这个管理，统一为团队设置一个账号，由管理员进行`npm adduser`操作。同样发布的过程也不是通过开发者进行的，而是由 Web 系统通过团队账号进行`npm publish`操作。同样发布的过程也不是通过开发者进行的，而是由 Web 系统通过团队账号进行
对于二方模块，大多数开发团队都有自己的代码审核流程。在有版本需要发布的时候，通过 Web 系统来申请发布即可。在发布的过程中，可以通过源代码版本控制系统参与。

在二方模块中，严格禁止`--force`模式的发布，通过这个 Web 系统来完成这个操作，禁止覆盖发布以避免潜在风险。

#### 4. 企业模块管理系统

通过对私有仓库加入运维机制、进行备份容灾等产品化操作后，上述模式在笔者的团队（阿里巴巴数据平台）已经有超过一年的执行经验。该仓库支撑了多个团队数个产品的日常开发和线上部署。上面提及的 Web 系统即是我们的企业模块管理系统，由于开发过程中与企业有一些耦合，之后会将这部分耦合去掉，然后开源到社区中。

## D.3 总结

NPM 在 Node 的发展历程中，有着功不可没的作用。没有 NPM，Node 就没有如此众多的模块可以使用。没有 NPM 平台，CommonJS 组织将 JavaScript 应用到任何地方的想法将不可能那么快实现。然而官方 NPM 对于企业应用支持的缺失，导致很多企业在应用 Node 的过程中要经历很多弯路。本附录带来的解决方案希望企业在应用 Node 时能够在保护企业的同时享受到开源社区的好处，让 NPM 工具不应当因为环境的不同而不能使用。
