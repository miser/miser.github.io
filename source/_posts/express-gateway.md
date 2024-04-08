---
title: Express Gateway
category: JavaScript
keywords: Express,Gateway,Node.js 网关
description: 为了更好的做BFF层，最近看了一些网关资料，Node.js的网关类库相对薄弱很多。Express Gateway，背靠强大的Express社区，很多现成的中间件可以运用其中，省去了不少开发成本和风险。
tags:
  - Node.js
  - JavaScript
date: 2020-01-22T15:32:03.586Z
---

为了更好的做BFF层，最近看了一些网关资料，Node.js的网关类库相对薄弱很多，主要有2个

- [Express Gateway](https://www.express-gateway.io/)
- [Moleculer](https://moleculer.services/)

和Lua [Kong](https://konghq.com/)相比缺少很多刚需，比如金丝雀发布、灰度发布等等；和Java Zuul等相比，又少了很多中文文档。但是不管如何这2个是Javascript技术栈的，对于一个Node.js程序工作者来说怎能不香呢？

今天我们主要介绍**Express Gateway**，背靠强大的Express社区，很多现成的中间件可以运用其中，省去了不少开发成本和风险。一些不是很大的项目或者没有时间慢慢构建底层的团队来说，我觉得可以试试。

<!-- more -->

<br/>

## 基础概念

### **Endpoints**

URL的集合，分为2类：

- API endpoints
- Service endpoints

API endpoints是暴露给外网访问的，通过它将请求转发到具体的内网Service endpoints服务上。



### **Policies**

策略（policy）以Express Middleware的方式相互组织，作用在API请求和网关的响应上，包含触发条件、具体行为和参数。



### **Pipelines**

一个管道（pipeline）是API endpoints上一组策略（policies）的连接关系，管道里的策略被定义和执行。通过管道配置各种策略，一个API请求由API endpoint接受。在管道里的最后一个策略通常是代理，它将请求路由到一个service endpoint。



<br/>



## 安装

官方的[Installation](https://www.express-gateway.io/getting-started/)，可以通过CLI命令快速启动一个项目

```javascript
npm install -g express-gateway

eg gateway create

cd `Dir Path` && npm start
```

**2个重要的配置文件**

- system.config.yml：主要是数据库、加密方式、session等等
- gateway.config.yml：主要就是路由相关的策略

在浏览器里输入 http://localhost:8080/ip ，就会被路由到 https://httpbin.org/ip 这个网站，获得一个JSON数据

```javascripton
{
  "origin": "180.167.xxx.xxx"
}
```

当然这是一个超级简单的例，用的是自带的proxy策略，如果有多个微服务节点默认使用的是`round-robin`做负载均衡，除此之外是`static`方式，只能指定死IP或URL了，显然无法满足真实的场景，很多时候需要我们自己根据实际的情况做代理开发，比如从注册中心获取微服务IP和端口做路由转发。



## 例子

具体的代码可以在[express-gateway-example](https://github.com/miser/express-gateway-example)查看。

- 使用JWT做用户登录验证
- 2个微服务，一个是`account`，另一个是`banner`
  - account: 用户登录和需要验证有效身份后显示用户ID
  - banner: 无需登录就能访问2个banner图片

在新的文件夹下（express-gateway-example），通过eg(express-gateway)和express命令工具分别生成网关项目gateway和2个微服务项目account、banner，另外新建一个存放JWT秘钥的目录`secret-files`，在该目录下通过下面命令创建新的秘钥

```c
openssl genrsa -out private.pem 512
openssl rsa -in private.pem -outform PEM -pubout -out public.pem

```

**修改网关配置**

```yaml
// gateway.config.yml
http:
  port: 8080
# admin:
#   port: 9876
#   host: localhost
apiEndpoints:
  login:
    host: localhost
    paths: '/account/login'
  account:
    host: localhost
    paths: '/account/*'
  banner:
    host: localhost
    paths: '/banner/*'
serviceEndpoints:
  accountSrv:
    url: 'http://localhost:3001'
  bannerSrv:
    url: 'http://localhost:3002'
policies:
  - jwt
  - proxy
pipelines:
  login:
    apiEndpoints:
      - login
    policies:
      - proxy:
          - action:
              serviceEndpoint: accountSrv 
  banner:
    apiEndpoints:
      - banner
    policies:
      - proxy:
          - action:
              serviceEndpoint: bannerSrv
  account:
    apiEndpoints:
      - account
    policies:
      - jwt:
          - action:
              jwtExtractor: 'query'
              jwtExtractorField: 'token'
              checkCredentialExistence: false
              secretOrPublicKeyFile: '../secret-files/public.pem'
      - proxy:
          - action:
              serviceEndpoint: accountSrv

```

**3个apiEndpoints**

- login：访问path _/account/login_
- account：访问path _/account/*_
- banner：访问path _/banner/*_

**2个serviceEndpoints**

- accountSrv：对应`account`服务，本地端口3001
- bannerSrv：对应`banner`服务，本地端口3002

**配置pipelines**

- login和banner这2个apiEndpoints被分别简单的路由到各自的微服务上
- account这个apiEndpoints除了路由外还有JWT策略，action告诉系统它将以URL query的token字段传递加密的认证信息

**启动项目和访问它们**

在各个目录下通过`npm start`分别启动它们，通过Postman GET请求http://localhost:8080/account/profile 发现返回401，可见没有通过token验证就会返回401，符合pipelines里account的配置；GET 请求http://localhost:8080/banner/ 会正常返回数据，它没有被JWT策略约束

```
{"code":200,"message":"success","data":["/images/1.png","/images/2.png"]}
```

**获取token**

POST http://localhost:8080/account/login ,它在配置中也未加JWT策略，返回如下：

```javascripton
{
    "code": 200,
    "message": "success",
    "token": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6MjAyMCwibmFtZSI6Im1pc2VyIiwiaWF0IjoxNTgwMTQxODIwLCJleHAiOjE1ODAxNDU0MjB9.mixQa9rJqQAT2makAqWfpOCxTC-r0XussuoSrYYTb0aXcs0gMItSI5Aj6ShneX2H1BW1grXwtkrSqY8_FfhIjA"
}
```

将token以URL query形式传参，重新访问profile接口，就正常返回用户ID了

```javascripton
{
    "code": 200,
    "message": "success",
    "data": {
        "id": "2020"
    }
}
```

上述就是一个简单的Gateway简单的例子，除开内置的一些策略（中间件）外，我们还可以自己开发一些中间件来满足具体需求——**`插件`**。

<br/>

## 插件

插件分两种：

- [Conditions](https://www.express-gateway.io/docs/policies/customization/conditions/)

- [Policies](https://www.express-gateway.io/docs/policies/)

无论是上述哪一个，写好后都要注册到网关中，在gateway项目中新建`plugins`目录，及其子目录`policies`、`conditions`， 和`manifest.js`、`policies/index.js`、`conditions/index.js`文件

```javascript
// manifest.js
module.exports = {
  version: '0.0.1',
  init: function (pluginContext) {
    const policy = require('./policies/index.js');
    pluginContext.registerPolicy(policy);

    const condition = require('./conditions/index.js');
    pluginContext.registerCondition(condition);
  }
}
```

通过registerPolicy和registerCondition将策略和条件注册到网关系统中，另外在`system.config.yml`配置文件中添加插件的配置路径

```yaml
// system.config.yml
plugins:
  example:
    package: './plugins/manifest.js'
```



**Conditions 条件**

它通过定义一个方法来判断是否执行或跳过一个策略

> ##### `function (req, conditionConfig) => true/false` Handler
>
> - Executes on each request in current pipeline. If not matched will prevent policy from being fired

在上面的例子中，我们在apiEndpoints和pipelines都定义了一个login，其实它是accountSrv的一个特殊的存在，除了login其它的url地址都是受JWT策略约束的，按照之前的写法显得格外的冗余，我们可以通过一个Condition来做改进，重新改写gateway.config.yml

```yaml
// gateway.config.yml
apiEndpoints:
  # login:
  #   host: localhost
  #   paths: '/account/login'
  
# ...

pipelines:
  # login:
  #   apiEndpoints:
  #     - login
  #   policies:
  #     - proxy:
  #         - action:
  #             serviceEndpoint: accountSrv 
  account:
    apiEndpoints:
      - account
    policies:
      - jwt:
          - 
            condition:
              name: 'white-list'
              list: ['/account/login']
            action:
              jwtExtractor: 'query'
              jwtExtractorField: 'token'
              checkCredentialExistence: false
              secretOrPublicKeyFile: '../secret-files/public.pem'
      - proxy:
          - action:
              serviceEndpoint: accountSrv
```

我们看到在jwt下面多了一个condition，然后在`conditions/index.js`里实现一个简单的URL Path 过滤

```javascript
module.exports = {
  name: 'white-list',
  schema: {
    $id: 'white-list',
  },
  handler: conditionConfig => req => {
    return conditionConfig.list.indexOf(req.url) < 0;
  }
};
```

这个Condition就完成了白名单功能了。



**Policies 策略**

它通过一个中间件方法对所有流入网关请求的预处理

结合上面的banner接口，我们新开发了一个v2版本的banner列表接口`/banner/v2/list`，为了老版本的客户端依旧能通过`/banner`接口访问到新的v2版本，我们需要做一个URL替换

```yaml
pipelines:
  banner:
    apiEndpoints:
      - banner
    policies:
      - rewrite:
          - action:
              search: '/banner'
              replace: '/banner/v1/list'
      - proxy:
          - action:
              serviceEndpoint: bannerSrv
```

在`policies/index.js`添加替换方法

```javascript
module.exports = {
  name: 'rewrite',
  schema: {
    $id: 'rewrite',
  },
  policy: (actionParams) => {
    return (req, res, next) => {
      req.url = req.url.replace(actionParams.search, actionParams.replace);
      next()
    };
  }
};
```

当我们再GET http://localhost:8080/banner/ 时候，将返回新的v2版本的数据

```javascripton
{
    "code": 200,
    "message": "success",
    "data": [
        "/images/v2_1.png"
    ]
}
```

至此，简单的URL地址替换就完成了。



<br/>

<br/>



## 总结

上诉只是Express-Gateway的一角，还有很多有趣灵活的功能值得慢慢探索。BFF层最为整个大系统的前沿征地，而网关更是前沿的前沿，配合GraphQL我相信能不断释放出JavaScript快速开发和迭代的能力，为客户端提供更好的服务和需求响应。