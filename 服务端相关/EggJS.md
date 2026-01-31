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

