---
layout: post
title:  "Fake Leancloud Auth：一个基于 node.js 的 API 服务例程"
categories: tutorial
tags:
- Node.js
- Express
- API
- Leancloud
---

最近因为需要在我们的 [CI 服务器](https://circleci.com)里面使用 [Leancloud](https://leancloud.cn/) 的 [authentication API](https://leancloud.cn/docs/rest_api.html#用户-1)，但从里面直连 Leancloud 环境并不是一个理想的解决方案，我决定自己写一个[简单的 API 服务器](https://github.com/Jimexist/fake-leancloud-auth)，尽量模仿他们的 API 服务器行为，方便 CI 环境测试使用。这个项目本身使用了一些不错的 node.js 框架，因而总结成为这篇文章，顺便作为内部 node.js 的培训材料。

## 项目功能

目前项目还在进一步开发中，我的目标是满足以下的功能：

1. 用户登录 `logIn`
2. 用户注册 `signUp`
3. 获取某个用户基本信息
4. 获取已登录用户自己的信息

目前的目标是在功能上覆盖最基本的 Leancloud API 功能，仅为了满足测试需要。

## 项目结构

因为项目本身还在进一步开发中，以下代码示例都以 [0.2.1](https://github.com/Jimexist/fake-leancloud-auth/tree/0.2.1) 为例：

```
.
├── Dockerfile  // for docker build
├── LICENSE
├── README.md
├── app.js  // app entrance
├── middlewares
│   └── ensureHeaders.js // filtering based on headers
├── models
│   └── user.js // mongoose model for `user'
├── npm-shrinkwrap.json
├── package.json // npm metadata
├── routes.js // route information, used by app.js
├── server.js  // the project entrance
└── test // tests
    ├── index.js
    └── mocha.opts

3 directories, 12 files
```

关于具体文件的作用都已经在上面 inline 的注释出来了，下面以文件为中心重点讲解一下项目内容。


## package.json

[script](https://github.com/Jimexist/fake-leancloud-auth/blob/0.2.1/package.json#L6) 部分包含了比较简单的几个命令，其中 `test` 主要是用 [mocha](https://mochajs.org/) 进行单元测试，而 `pretest` 部分用了 [snazzy](https://www.npmjs.com/package/snazzy)，这是一个对 [standard.js](https://www.npmjs.com/package/standard) 的简单包装，是一个零配置的代码检查器（linter）。写在 `pretest` 里面就可以保证每次跑测试（`npm test`）之前都自动运行。

在 node 项目中使用 linter 是一个非常好的习惯，它不仅保证了团队协作中的代码风格统一，更重要的是，以我自己的经验来讲，它还可以帮助你找到许多意想不到的bug——这对缺乏强类型支持的 JavaScript 来讲是一个不可多得的工具。

需要多说两句的是 standard.js 是一个有争议的工具——它不像 [eslint](https://github.com/eslint/eslint) 那样，允许你调整各种参数，而是直接规定了一套，并且专门强调了不允许你去调整（除极个别的例外情况）。当然我们自己也使用过另外一套 [airbnb 的标准](https://github.com/airbnb/javascript)。我自己关于 「linter 标准」的态度和 standard.js 的作者的态度基本一致：just pick one thing and stick to it。（感兴趣的人可以去了解这个包的作者 [feross](http://feross.org/resume/)，是一个斯坦福的毕业生，经历比较有趣）。

## server.js

这是整个项目的入口。`server.js` 这个名字比较特殊，即如果没有在 `package.json` 中定义 `start` 脚本，默认在 `npm start` 的时候就会在项目目录里面寻找 `server.js` 然后运行。

这里面的代码相对比较简单，主要是调用 `app.js` 中的定义，然后绑定到 3000 端口运行。项目里面把服务器拆分成 `app.js` 和 `server.js` 主要是在测试里面可以重用 `app.js` 的代码，但又不需要固定在 3000 端口来运行（比如 [supertest](https://www.npmjs.com/package/supertest) 项目里面就可以使用 ephemeral 端口进行测试，尽量保证不和运行环境端口冲突）。


## app.js

API 服务器的主要内容都在这个文件里面，以下分别讲解。

**imports**

[这一部分](https://github.com/Jimexist/fake-leancloud-auth/blob/0.2.1/app.js#L1)引用了不少各种库:

1. [passport](https://www.npmjs.com/package/passport) 是一个专门用来做 authentication 的库，支持各种第三方平台。它的每一个authentication 的方法都叫做 strategy，是为了方便其他人一起来参与开发，所以现在 passport 支持好几百种strategies。不过，在这个项目里面我们只使用到了 local 这一个strategy，因为用户的信息是保存在 MongoDB 里面的
2. [body-parser](https://www.npmjs.com/package/body-parser) 和 [express-session](https://www.npmjs.com/package/express-session) 曾经都是 express 的一部分，但是在 4.x 版本中 express 被做的更加模块化，不必要的功能都分拆出去了，比如你的项目需要用到会话管理才引入 express-session；如果用不到处理带有 body 的请求（比如POST）可以省略掉 body-parser。得益于 connect 这一中间件的约定（比如中间件插件的参数形式都是 `function (req, res, next)` ），这样的精简才变得可能——同时也使得用户可以自己加入自己独特的插件，而每个插件的开发变得独立和更方便
3. [mongoose](https://www.npmjs.com/package/mongoose) 是 node 里面优秀的 ODM 工具，可以让我们不需要直接使用 MongoDB 的 API，还可以定义一些虚拟 attribute 等。另外，这个项目使用到的 authentication strategy 就是基于 mongoose 的
4. [morgan](https://www.npmjs.com/package/morgan) 是一个蛮不错的 HTTP request logger，支持至少 `dev` 和 `combined` 格式，前一种适合开发使用，而后一种和 Apache 的格式兼容，所以一般用于 production，可以进一步的复用各种日志处理工具

最后值得一提的是 mongoose 内置的 `Promise` 实现并不理想，作者建议用别的代替。你可以使用 [bluebird](https://www.npmjs.com/package/bluebird)，而这个项目因为使用了 node 6，就直接用原生实现代替了。

**App 设置**

[接下来](https://github.com/Jimexist/fake-leancloud-auth/blob/0.2.1/app.js#L14)就是配置 app 本身了，主要是有三个部分。

最开始是配置一些日志和会话的配置。`express-session` 内置了一个 in-memory 的 session store，但是如果服务器重启就会丢失。更好的方法是用 Redis 或者 MongoDB 来存储。这里用了 `connect-mongo` 来连接 MongoDB 进行会话存储，也是因为我们已经有了 MongoDB 这个依赖，可以直接复用。MongoDB 也自带 ttl（time-to-live document）功能，因而对于过期会话的管理可以由数据库自动实现。

passport 的一些配置随后也加入到了 APP 中。其中值得注意的是初始化的顺序，比如 `passport.session` 必须在 `express-ession` 之后，因为前者依赖 `req.session` 的存在，才可以进一步的向其中注入 `passport` 属性。因为中间件系统是流式的，其中是 handler 的顺序是有意义的，这个无论是在使用第三方库还是定义自己的路由的时候都需要注意。

最后就是一些自己定义的路由。我们这里使用了单独的 `express.Router` 来写我们的代码逻辑，并且拆分到一个单独的文件里面。而 `ensureAppHeaders` 是一个检查 headers 的中间件，也是利用了 connect 中间件的写法，放在了路由的 mount point（挂载点）前面。

## routes.js

接下来是代码逻辑定义的地方，主要的功能和路由都定义到了一个 `express.Router` 里面，这样更加模块化，方便测试和复用。

其中值得一提的是[用户注册的逻辑](https://github.com/Jimexist/fake-leancloud-auth/blob/0.2.1/routes.js#L37)，这里没有直接使用 `new User` 的 mongoose 构造函数，而是调用了 `passport-local-mongoose` 插件定义的函数 `register` ，原因也很简单：用户的密码不能直接明文保存在 MongoDB 里面，而是需要进行哈希（cryptographic hashing），并且附上盐（salt），防止彩虹表攻击。补充一点是，MD5 和 SHA1 在这里都不是安全的哈希函数，如果你不知道这一点的话，也就不推荐你自己写加密的逻辑。一个比较好的选择是 [bcrypt](https://www.npmjs.com/package/bcrypt)，但是需要重申的是，用户信息安全是一个重要的话题，而这个项目本身只是为了测试使用，这里因为篇幅问题就不展开了。

另外 express 背后使用的库支持比较复杂的路由匹配，比如[这里](https://github.com/Jimexist/fake-leancloud-auth/blob/0.2.1/routes.js#L85)就使用到了正则表达式规则来保证传入的用户 ID 是符合格式的。

## 部署

这个项目的部署使用了 docker，因为我们的测试环境 CircleCI 对 docker 的支持较好。如果你对 docker 还不是很了解，可以查看docker 的文档，这里不再赘述。

[Dockerfile](https://github.com/Jimexist/fake-leancloud-auth/blob/0.2.1/Dockerfile#L5) 的书写相对比较简单，这里唯一需要注意的是为了充分利用 docker 的 layer 缓存，先把 `package.json` 和 `npm-shrinkwrap.json` 复制到目标文件夹，这样在没有修改这两个文件的时候，docker 不会重新调用 `npm install` ，极大的节省了打包时间。`npm-shrinkwrap.json` 是一个把目前安装的库的准确版本进行固定的工具，可以用 `npm shrinkwrap` 命令生成（类似于 `pip freeeze` 在 Python 中的用法），这样可以进一步减少因为依赖突然改变带来的问题。

打包好的 docker 镜像使用起来也比较简单，只需要和 MongoDB 的容器相连，并且提供相应的环境变量就可以了。如果你不想使用 docker，可以考虑 [pm2](https://www.npmjs.com/package/pm2)。


## 总结

这篇文章里面介绍了一个基于 express 的 API 服务器的简单架构，并且介绍了一些常用的库。因为篇幅问题，还有许多不完善的地方。作为一篇介绍，希望可以帮助对 node 和 express 比较陌生的人有所帮助。
