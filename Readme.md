[![Express Logo](https://i.cloudup.com/zfY6lL7eFa-3000x3000.png)](http://expressjs.com/)

  Fast, unopinionated, minimalist web framework for [node](http://nodejs.org).

  [![NPM Version][npm-image]][npm-url]
  [![NPM Downloads][downloads-image]][downloads-url]
  [![Linux Build][travis-image]][travis-url]
  [![Windows Build][appveyor-image]][appveyor-url]
  [![Test Coverage][coveralls-image]][coveralls-url]


---

## 模块依赖

模块依赖参考：[npmgraph](http://npm.broofa.com/?q=express)

![img](/graphviz/module_depends.png)

## 目录结构

```
├── application.js - 封装一些app.**的API
├── express.js - 真正暴露给外界的express对象
├── middleware
|  ├── init.js - 将app.request/app.response赋予req/res的中间件，基于setPrototypeOf
|  └── query.js - 对query进行解析的中间件，底层基于qs模块
├── request.js - 封装了一些req.**的API和属性
├── response.js - 封装了一些res.**的API和属性
├── router
|  ├── index.js
|  ├── layer.js
|  └── route.js
├── utils.js
└── view.js
```

## 执行流程

### express.js

入口文件，封装express对象，调用图：

![](/graphviz/express.dot.svg)

1、通过mixin给app对象植入EventEmitter和app，其中app依赖application.js，

2、然后调用app.init进行初始化。

3、并将各个内部模块对外进行暴露。

### application.js

主要封装app.** 的各种api，调用图：

![](/graphviz/application.dot.svg)

#### app.all

给所有methods模板的对应方法引入fn。

#### app.init初始化
express引入application会立即调用的入口方法，通过defaultConfiguration完成默认配置的设置。针对不同的配置有不同的底层库来对应配合。

比如说etag使用etag模块；查询字符串解析使用qs模块或原生的querystring模块；代理使用proxy-addr模块。

然后监听了mount事件，基于setPrototypeOf模块将自己的原型设置为传入的对象。

#### app.use使用中间件

将封装好的中间件传入，会遍历this._router,给每个路由引入传入的中间件的fn.handle方法。

#### app.render渲染
调用view.js对象，使用view.render来完成模板视图的渲染

#### app.listen监听

通过原生http模块的createServer创建server，再通过server.listen来监听即可

#### app.engine指定引擎

指定引擎到this.engines中，方便render方法调用时获取引擎传递给view对象

#### app.set设置或获取配置

如果只有一个参数则获取配置，如果多个则设置。终点是this.settings。基于几种特殊情况，比如说etag、queryparser、trustproxy采用的util，使用底层库组装

#### app.enable/app.disable

调用app.set，只不过第二个参数自动为true/false

#### app.methods(app.get/app.post)请求

使用methods模块，对每个不同请求类型引入fn处理函数。如果是get方法，则进行多态处理，单参数情况直接调用set，完成配置的读取。

#### route/param/handle

直接调用了底层router的对应方法。

#### app.lazyrouter

对路由配置进行设置，如果this._router已设置，则直接返回。

### middleware/init.js

用来设置header头X-Powered-By Express，以及将传入的app的request和response，通过setprototypeof模块注入到req和res上。

调用时机在application的lazyrouter方法中，针对this._router使用了该中间件。

### middleware/query.js

给req增加query属性，基于qs模块。

调用时机同样在application的lazyrouter方法中，针对this._router使用了该中间件。

### request.js

给req增加大量属性和方法。调用关系图：

![](/graphviz/request.dot.svg)

#### req对象
基于http模块的IncomingMessage对象。

#### req.acceptsLanguage/accepts/acceptsEncodings/acceptsCharsets

均基于accepts模块，用来处理Http请求头之Accept字段。

#### req.range()

基于range-parser模块，用来处理Http请求头之Range字段。

#### req.param()

已准备废弃的方法，通过优先级依次从params,body,query找对应key的值。

#### req.is()

基于type-is模块，判断http请求头之content-type字段

#### defineGetter()

通过Object.defineProperty给某些属性增加get函数，以下几个属性都基于此。使用Object.defineProperty是为了来防止篡改这些对象，因为默认情况下writable为false，所以即使外界设置了值，也无效。

#### req.fresh

通过fresh模块判断是否走协商缓存

#### req.ips/ip

基于proxy-addr模块，和代理相关.

#### req.path

基于parseurl模块做url的路径解析

#### req.subdomains

基于req.hostname做字符串解析，拿到子域名

#### req.secure

基于req.protocol，判断是否是https，是否安全

#### req.xhr

基于http请求头之x-requested-with判断是否是xhr

#### req.protocol

基于http请求头之x-forwarded-proto获取协议名

#### req.get/header

从this.header上拿到指定http头的值

### response.js

给res增加大量属性和方法。调用关系图：

![](/graphviz/response.dot.svg)

#### res对象 

res对象基于原生http模块的ServerResponse对象

#### res.get()

获取对应响应头，调用原生http.ServerResponse的getHeader方法。

#### res.status()

设置状态码，底层ServerResponse.statusCode

#### res.json()

通过封装的stringify方法，序列化json对象为字符串，再通过res.send发送

#### res.send()

发送响应，基于底层ServerResponse.end()

#### res.jsonp()

支持封装jsonp

#### res.sendStatus()

发送状态码，基于statuses模块。

#### res.append()

通过app.set()，增加响应头。

#### res.set/res.header

基于ServerResponse.setHeader()，来设置响应头。

#### res.render

基于app.render来渲染模板。

#### res.links/res.cookie/res.attachment

通过res.set()，来修改不同的http响应头信息。底层基于不同的三方模块。

### router/index.js router/layer.js router/route.js

调用关系图：

![](/graphviz/router.dot.svg)

三者关系是，router用来做Router类的入口。layer是用来处理同一path的，route是用来处理不同method的。

通过Router.route()方法来实例化Route和Layer，并传入Route的dispatch方法，最后挂载Layer到自己的栈Router.stack上。
Router对于不同methods，则调用Route的methods方法。
Router.use会实例化Layer，并挂载Layer到自己的栈Router.stack上。
Router.handle方法会处理栈，直到全部处理完成，底层基于Layer的handle_error和handle_request方法。

Route也有自己的栈Route.stack，在使用Route.methods方法时，会挂载到stack上，并在dispatch方法时去消费栈，底层也是调用Layer的handle_error和handle_request方法。

Layer只要是用来将同一path的路由安排在一起，每一个中间件都是一个Layer对象。


## 原理

### 请求的完整流程

当请求到来时，处理过程是app.handle → router.handle，事实上，app.handle调用了router.handle，而router.handle的过程，则是依次对router.stack中存放的中间件进行调用。

router.stack中存的是一个个的Layer对象，用来管理中间件。如果Layer对象表示的是一个路由中间件，则其route属性会指向一个Route对象，而route.stack中存放的也是一个个的Layer对象，用来管理路由处理函数。

因此，当一个请求到来的时候，会依次通过router.stack中的Layer对象，如果遇到路由中间件，则会依次通过route.stack中的Layer对象。

对于router.stack中的每个Layer对象，会先判断是否匹配请求路径，如果不匹配，则跳过，继续下一个。在路径匹配的情况下，如果是非路由中间件，则执行该中间件函数；如果是路由中间件，则继续判断该中间件的路由对象能够处理请求的HTTP方法，如果不能够处理，则跳过继续下一个，如果能够处理则对route.stack中的Layer对象（与请求的HTTP方法匹配的）依次执行。

## 实例说明

- auth - 通过express-session和pbkdf2-password处理用户登录鉴权，通过session保存用户登录信息

- content-negotiation - 通过res.format，来format不同类型如html/text/json的内容协商类型

- cookie-sessions - 使用cookie-session，用cookie加密做session的简单方案

- cookies - 使用cookie-parser来管理cookie

- download - 使用res.download去下载文件

- ejs - 使用ejs模板引擎，并将后缀识别变为html

- error - 自定义异常处理，并置于所有route之后

- error-pages - 错误页处理

- hello-world - 一个最基本的helloworld

- markdown - 使用markdown模板引擎

- multi-router - 多路由管理，封装controller

- multiparty - 使用multiparty处理表单`enctype="multipart/form-data"`

- mvc - 一个mvc的demo

- online - redis使用

- params - 解析各种不同的参数风格

- resource - 通过app.resource管理资源

- route-map - 自定义map，统一管理router

- route-middleware - 自定义路由中间件，配合各路由使用

- route-separation - 路由分离管理

- search - 一个查询接口的例子

- session - 使用session和使用redis配合session的例子

- static-files - 通过express.static指定静态资源的例子

- vhost - 使用vhost虚拟主机域名管理

- view-consctructor - 自定义模板引擎并使用

- view-local - 稍微复杂点的使用模板的例子，使用的是ejs

- web-service - 写一个服务提供稍微复杂点的接口



---

```js
var express = require('express')
var app = express()

app.get('/', function (req, res) {
  res.send('Hello World')
})

app.listen(3000)
```

## Installation

This is a [Node.js](https://nodejs.org/en/) module available through the
[npm registry](https://www.npmjs.com/).

Before installing, [download and install Node.js](https://nodejs.org/en/download/).
Node.js 0.10 or higher is required.

Installation is done using the
[`npm install` command](https://docs.npmjs.com/getting-started/installing-npm-packages-locally):

```bash
$ npm install express
```

Follow [our installing guide](http://expressjs.com/en/starter/installing.html)
for more information.

## Features

  * Robust routing
  * Focus on high performance
  * Super-high test coverage
  * HTTP helpers (redirection, caching, etc)
  * View system supporting 14+ template engines
  * Content negotiation
  * Executable for generating applications quickly

## Docs & Community

  * [Website and Documentation](http://expressjs.com/) - [[website repo](https://github.com/expressjs/expressjs.com)]
  * [#express](https://webchat.freenode.net/?channels=express) on freenode IRC
  * [GitHub Organization](https://github.com/expressjs) for Official Middleware & Modules
  * Visit the [Wiki](https://github.com/expressjs/express/wiki)
  * [Google Group](https://groups.google.com/group/express-js) for discussion
  * [Gitter](https://gitter.im/expressjs/express) for support and discussion

**PROTIP** Be sure to read [Migrating from 3.x to 4.x](https://github.com/expressjs/express/wiki/Migrating-from-3.x-to-4.x) as well as [New features in 4.x](https://github.com/expressjs/express/wiki/New-features-in-4.x).

### Security Issues

If you discover a security vulnerability in Express, please see [Security Policies and Procedures](Security.md).

## Quick Start

  The quickest way to get started with express is to utilize the executable [`express(1)`](https://github.com/expressjs/generator) to generate an application as shown below:

  Install the executable. The executable's major version will match Express's:

```bash
$ npm install -g express-generator@4
```

  Create the app:

```bash
$ express /tmp/foo && cd /tmp/foo
```

  Install dependencies:

```bash
$ npm install
```

  Start the server:

```bash
$ npm start
```

## Philosophy

  The Express philosophy is to provide small, robust tooling for HTTP servers, making
  it a great solution for single page applications, web sites, hybrids, or public
  HTTP APIs.

  Express does not force you to use any specific ORM or template engine. With support for over
  14 template engines via [Consolidate.js](https://github.com/tj/consolidate.js),
  you can quickly craft your perfect framework.

## Examples

  To view the examples, clone the Express repo and install the dependencies:

```bash
$ git clone git://github.com/expressjs/express.git --depth 1
$ cd express
$ npm install
```

  Then run whichever example you want:

```bash
$ node examples/content-negotiation
```

## Tests

  To run the test suite, first install the dependencies, then run `npm test`:

```bash
$ npm install
$ npm test
```

## People

The original author of Express is [TJ Holowaychuk](https://github.com/tj)

The current lead maintainer is [Douglas Christopher Wilson](https://github.com/dougwilson)

[List of all contributors](https://github.com/expressjs/express/graphs/contributors)

## License

  [MIT](LICENSE)

[npm-image]: https://img.shields.io/npm/v/express.svg
[npm-url]: https://npmjs.org/package/express
[downloads-image]: https://img.shields.io/npm/dm/express.svg
[downloads-url]: https://npmjs.org/package/express
[travis-image]: https://img.shields.io/travis/expressjs/express/master.svg?label=linux
[travis-url]: https://travis-ci.org/expressjs/express
[appveyor-image]: https://img.shields.io/appveyor/ci/dougwilson/express/master.svg?label=windows
[appveyor-url]: https://ci.appveyor.com/project/dougwilson/express
[coveralls-image]: https://img.shields.io/coveralls/expressjs/express/master.svg
[coveralls-url]: https://coveralls.io/r/expressjs/express?branch=master
[gratipay-image-visionmedia]: https://img.shields.io/gratipay/visionmedia.svg
[gratipay-url-visionmedia]: https://gratipay.com/visionmedia/
[gratipay-image-dougwilson]: https://img.shields.io/gratipay/dougwilson.svg
[gratipay-url-dougwilson]: https://gratipay.com/dougwilson/
