# npm命令

npm config list：列出所有的 npm 配置信息

<img src=".\npm.assets\image-20250306201632781.png" alt="image-20250306201632781" style="zoom: 67%;" />

npm init xxx与npm create xxx是等价，两者只是别名，可以互换使用

**npm init/create vite@latest背后做的事情：**（vite脚手架工具包名称为create-vite）

1. 添加create-前缀：如果提供的名字没有以create-开头，npm会自动加上该前缀，这里将vite添加create-前缀形成名称create-vite

2. 查找官方包并运行，这里是查找名为create-vite的包并运行



# package.json

```json
{
  "name": "唯一标识的包名,普通报名xxx默认公开的,作用域包@xxx/xxx默认私有,需要会员才能发布,也可以配置publishConfig声明为公开包",
  "publishConfig":{
    "access":"public"
  },
  "version": "1.0.0", // 版本号,主版本号.次版本号.修订号
  "private":"是否是私有包",
  "author": "作者",
  "description": "描述信息",
  "keywords": ["关键词，提示SEO"],
  "repository": {
    // 代码托管位置
    "type": "git",
    "url": "https://github.com/xxx/shang-utils"
  },
  "license": "许可证",
  "homepage": "包的主页或者文档首页",
  "bugs": "用户问题反馈地址",
  "files":["指定推送到npm服务器上的文件是哪些"],
  “typings”:"指定typescript的入口文件",
  “types”:"指定typescript的入口文件,与typings配置一个即可",
  "main": "该包被CMJ方式引入时的入口文件",
  "module":"该包被ESM方式引入时的入口文件",
  "exports":{
    // 声明被引入时各个方式下的入口文件，优先级大于上面的mian、module
    ".":{
      "import":"该包被ESM方式引入时的入口文件",
      "require":"该包被CMJ方式引入时的入口文件",
      "types":"指定typescript的入口文件"
    }
  },
  "scripts": {
    // 存放可执行脚本
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "dependencies": {
    // 运行依赖
  },
  "devDependencies": {
    // 开发依赖
  }
}

```



# 下载使用包

## 全局包

可以直接通过命令行运行的包命令，通过npm i xxx -g来安装

当前设备全局包存放路径

<img src="./npm.assets/image-20250306201849722.png" alt="image-20250306201849722" style="zoom: 50%;" />

为什么全局包可以直接在命令行通过命令运行呢：因为系统环境变量中有全局包的位置，所以命令行可以找到该命令，故可以直接通过命令行运行命令

<img src="./npm.assets/image-20250306202053079.png" alt="image-20250306202053079" style="zoom: 50%;" />

## npx

npx是一个命令行工具，它是npm 5.2.0版本中新增的功能。它允许用户在不安装全局包的情况下，运行已安装在本地项目中的包或者远程仓库中的包。

解决问题：节省磁盘空间、使用最新版本的包

命令：npx 要运行的命令

查找规则如下

- 先从当前项目的node_modules/.bin去查找可执行命令xxx
- 如果没找到就去全局的node_modules 去找可执行命令xxx
- 如果还没找到就去npm官网上下载，之后用完之后删除 

### 使用npx的例子——以nodemon为例子

使用命令npm ls -g查看当前安装的全部包可知，当前并没有全局安装nodemon包

**方案1**

npm i nodemon -g全局安装nodemon后，使用nodemon xxx

**方案2**

npx nodemon xxx运行

## package-lock.json

作用：

- 锁定版本记录依赖树详细信息
- 做缓存：通过`name + version + integrity` 信息生成一个唯一的key，这个key能在npm_cache 下的_cache下的index-v5 下的找到对应的缓存记录，如果发现有缓存记录，就会找到tar包的hash值，然后将对应的二进制文件解压到node_modeules

<img src=".\npm.assets\image-20250306202402037.png" alt="image-20250306202402037" style="zoom: 50%;" />



## npm包格式

发展时间线：IIFE-->AMD-->CJS-->UMD-->ESM

作为npm包作者，只要处理ESM、CJS格式就行了，对于ts版本则再加一个.d.ts的类型声明文件

### ESM

ESModule格式，官方标准，支持tree-shaking

在现代浏览器、node大于14的环境都可以使用

### CJS

CommonJs格式，不支持tree-shaking

在node环境中可以使用

### AMD

浏览器的第一代模块系统，使用Requirejs

### IIFE

立即执行函数

通过`<script>`标签直接使用

### UMD

Universal Module Definition

对上述CJS、AMD、IIFE进行统一处理，通过判断当前所处环境，用对应的方式暴露导出



# npm运行package.json中的脚本

## npm run xxx脚本发生了什么

读取package json 的scripts 对应的脚本命令的内容，之后按照如下规则去查找该命令

- 先从当前项目的node_modules/.bin去查找可执行命令xxx
- 如果没找到就去全局的node_modules 去找可执行命令xxx
- 如果还没找到就去环境变量查找
- 再找不到就进行报错

由于nodejs 是跨平台的所以可执行命令兼容各个平台，有三类对应的文件：.sh，.cmd，.ps1

- .sh文件是给Linux unix Macos 使用
- .cmd 给windows的cmd使用
- .ps1 给windows的powerShell 使用

<img src=".\npm.assets\image-20250306202733048.png" alt="image-20250306202733048" style="zoom: 50%;" />

## npm的生命周期

```json
"predev": "node prev.js",
"dev": "node index.js",
"postdev": "node post.js"
```

执行 npm run dev 命令的时候 predev 会自动执行，它的生命周期是在dev之前执行，然后执行dev命令，再然后执行postdev

运用场景：npm run build 可以在打包之后删除dist目录，post例如你编写完一个工具发布npm，那就可以在之后写一个脚本顺便帮你推送到git

**内置预设钩子**

prepare：安装依赖前、npm publish前，执行npm run prepare运行该脚本



