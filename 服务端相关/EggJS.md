# 运作原理

框架负责装载（Loader）+ 生命周期（BootHook）+ 约定目录结构，应用只需要把代码放在约定的位置，框架启动时自动把它们加载进来并按阶段调用。

使用yarn dev运行egg-bin dev创建Egg应用启动时

Egg启动一个loader加载config文件夹下的配置，先读取config.default.ts，再按环境读取config.local.ts、config.prod.ts等配置文件，合并成最终的app.config

按照目录约定加载业务代码：

读取app/middleware/*下的文件注册中间件

读取app/service/*下的文件挂到ctx.service

读取app/controller/*下的文件挂到app.controller下

读取app/router.ts注册路由

配置加载完后，加载app.ts，自动new AppBootHook(app)创建应用，调用生命周期中的回调

底层使用的KOA，请求进来后安装洋葱模型

- 请求进入 → 依次经过 middleware
- 命中路由 → 调用对应 controller 方法
- controller 里一般通过 `ctx.service.xxx` 调用 service
- 产出响应 → 再从中间件链“返回”出去（统一格式化、错误兜底等）



# 项目目录结构

```
/run
----/application_config.json最终合并得到的配置信息
----/application_config_meta.json最终合并得到的配置信息的文件来源

```



# 洋葱模型下中间件逻辑生效顺序

中间件1---->中间件2---->路由控制器命中，调用service---->中间件2---->中间件1

完整的一次请求有进入、出来两段轨迹。

执行某个中间件过程：

先从上至下执行该中间件到next()之前的代码（进入轨迹）

执行到next时结束进入轨迹，将控制权交给后面一层处理，当设置了ctx.body、ctx.status时结束

执行中间件next()之后的代码（出来轨迹）



# ctx

每次收到用户请求时，框架会实例化一个ctx对象，封装了本次请求的信息，并提供了便捷方法获取请求参数、设置响应信息

- 请求相关：`ctx.request`、`ctx.method`、`ctx.path`、`ctx.query`、`ctx.headers`
- 响应相关：`ctx.status`、`ctx.body`、`ctx.type`、`ctx.set(...)`
- Egg 扩展：`ctx.logger`、`ctx.service`、`ctx.model`



