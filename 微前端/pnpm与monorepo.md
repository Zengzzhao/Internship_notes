# pnpm

**软链接/符号链接：**后面的文件只是创建一个指向前面文件的快捷方式

**硬链接：**前后两份文件指向同一个磁盘地址，两者内容一样

## 与npm的区别

**依赖存储方式：**

npm每个项目都有一份node_modules依赖重复存储，占用大量磁盘空间

pnpm所有依赖统一存储在全局仓库中

```bash
# 查看pnpm全局仓库位置
pnpm store path
# mac:~/Library/pnpm/store
# win:~/.pnpm-store
```

每个项目中的node_modules下有一个.pnpm的虚拟仓库，该虚拟仓库.pnpm下的依赖是通过硬链接指向全局仓库的，而每个项目中的node_modules下的依赖包通过符号链接指向当前项目的虚拟仓库.pnpm中的包

```bash
node_modules/
├─ .pnpm/
│  ├─ react@18.2.0/
│  │  └─ node_modules/
│  │     └─ react/   ← 真实包文件（hard link）
│  ├─ lodash@4.17.21/
│  └─ ...
├─ react        -> .pnpm/react@18.2.0/node_modules/react
├─ lodash       -> .pnpm/lodash@4.17.21/node_modules/lodash

```

对于monorepo项目，根目录与每个子包都同理，该包下的node_modules通过该包的虚拟仓库最终指向了同一个底层文件

**安装速度：**

首次需要远程下载，后续依赖复用

**优势：**

节省磁盘空间并提升安装速度

## 开发实际使用

vscode中的软链接文件，其右侧有一个箭头，这个表示该文件是软链接

![image-20251002143943486](pnpm与monorepo.assets/image-20251002143943486.png)

使用命令pnpm init后，会生成pageage.json

之后再使用pnpm i vue安装相应的文件时，会生成一个pnpm-lock.yaml锁定版本号的文件以及node_modules

![image-20251002144328279](pnpm与monorepo.assets/image-20251002144328279.png)

在node_modules中会显示安装的vue，我们可以看到安装的vue是一个软链接，指向的是在.pnpm中的vue，.pnpm中的vue通过硬链接的方式指向了全局仓库中的包

![image-20251002144630810](pnpm与monorepo.assets/image-20251002144630810.png)

## 命令

执行工程级（项目的根目录）命令，在任何一个目录下都行，只要加了工作区参数

```shell
pnpm --workspace-root [xxx命令]
# 或
pnpm -w [xxx命令]
```

执行子包命令

```shell
进入子目录中执行命令
# 或
# 在任何目录下运行都行，只要指定了子包路径参数
pnpm -C 子包路径 [xxx命令]
```

当前执行目录下的所有子包中执行命令

```bash
pnpm --recursive [xxx命令]
# 或
pnpm -r [xxx命令]
```

过滤精确选择在某些包下执行命令

```bash
pnpm --filter 子包路径/子包名 [xxx命令]
```



# monorepo工程管理

## mutirepo vs monorepo

![monorepo vs multirepo](https://resource.duyiedu.com/yuanjin/202509190953043.svg)

常见monorepo管理工具：

- **pnpm**
- npm
- Yarn
- Lerna
- Nx
- Turborepo
- Rush
- ...

## workspace协议

workspace将monorepo中的包正确连接成模块的机制

包管理器会将workspace中配置的子包链接到根目录下的 `node_modules` 目录。这样，构建工具就可以像处理使用install安装的包一样处理这些子包。避免了路径别名的开销。

**子包间互相依赖**

```shell
pnpm -F 子包1 add 子包2 --workspace
```

这样子包1的package.json中对应子包2的依赖版本为`workspace:*`，而不是具体版本，这样可以确保子包2更新发包后，子包1不需要再次显式下载子包2的最新版本，使用`workspace:*`用的就是最新版本。当我们用 `pnpm publish` 发包的时候，pnpm会将 `workspace:` 替换为实际的版本。



## 搭建过程

### 初始工程搭建

建立各个子包文件夹，如`apps/backend--后端代码`、`apps/fronted--前端代码`、`packages/components--组件库代码`、`packages/utils--工具函数代码`

建立`pnpm-workspace.yaml`文件，声明工作区包含的包的位置

```yaml
# 这是一个 pnpm 工作区配置文件，定义了工作区中包含的包的位置。
packages:
  - "packages/*"
  - "apps/*"

```

在根目录下初始化工程，创建`package.json`文件

```bash
pnpm -w init
```

进入各个子包目录初始化工程，创建`package.json`文件

```bash
pnpm init
```

同时将每个工程的`package.json`的`name`改为`@根项目名/子包名`的形式

```json
{
  "name":"@monorepo-demo/backend"
}
```



### 环境版本锁定

将所有项目的环境版本锁定，在根目录下的`package.json`中进行统一配置管理

```json
// package.json
"engines": {
  "node": ">=22.14.0",
  "npm": ">=10.9.2",
  "pnpm": ">=10.15.1"
}
```

以上配置是非严格的，当不满足环境版本时只会警告，并不会报错。如果想要将其变为严格的，当环境版本不匹配时报错则在根目录下的`.npmrc`中配置严格模式	

```yaml
# .npmrc
engine-strict=true
```



### TypeScript配置

根目录下安装`ts`

```shell
pnpm -Dw add typescript @types/node
```

根目录下建立`tsconfig.json`文件，配置公共`ts`配置

```json
// tsconfig.json
{
  "compilerOptions": {
    "baseUrl": ".",
    "module": "esnext",
    "target": "esnext",
    "types": [],
    "lib": ["esnext"],
    "sourceMap": true,
    "declaration": true,
    "declarationMap": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
    "strict": true,
    "verbatimModuleSyntax": false,
    "moduleResolution": "bundler",
    "isolatedModules": true,
    "noUncheckedSideEffectImports": true,
    "moduleDetection": "force",
    "skipLibCheck": true
  },
  "exclude": ["node_modules", "dist"]
}
```

在各个子包的目录中建立`tsconfig.json`，配置各个子包独有特性

```json
// apps/backend/tsconfig.json
{
  "extends": "../../tsconfig.json",
  "compilerOptions": {
    "types": ["node"],
    "lib": ["esnext"]
  },
  "include": ["src"]
}
```

```json
// apps/frontend/tsconfig.json
{
  "extends": "../../tsconfig.json",
  "compilerOptions": {
    "types": ["node"],
    "lib": ["esnext", "DOM"]
  },
  "include": ["src"]
}
```



### 代码风格与质量检查

#### editorconfig

确保开发者在不同的编辑器IDE中开发代码风格一致

建立`.editorconfig`文件进行配置

```
root = true

[*]
indent_style = space 缩进使用空格
indent_size = 2	一个缩进对应两个空格，按下tab时对应一个缩进，此时空两个空格
end_of_line = lf 换行符使用LF格式
charset = utf-8 文件编码统一用utf-8
trim_trailing_whitespace = true 删除每行行尾的空格
insert_final_newline = true	在文件末尾自动插入一个空行

[*.md]
trim_trailing_whitespace = false

```

`vscode`编辑器上要搭配`editorconfig`必须下载插件`EditorConfig for VS Code`，这样在我们保存代码时会自动按照配置统一风格

#### prettier

`vscode`安装`Prettier - Code formatter`插件后，该插件就自带了默认配置，当我们没有`prettier`配置文件时会使用插件默认配置格式化。插件应用配置优先使用项目目录下的`prettier.config.js`配置文件，若没有则使用插件默认配置。

`prettier`配置文件供给`vscode`插件、`prettier`依赖库使用。通过`vscode`格式化快捷键来格式化代码是通过插件（插件读取`prettier`配置文件）实现的，而通过命令行运行格式化命令`prettier --write`是通过`prettier`依赖库（插件读取`prettier`配置文件）实现的。如果只是想在开发时格式化代码安装插件即可，如果想要实现CICD提交代码时自动格式化代码则需要安装`prettier`依赖库来运行命令行命令

`prettier`安装

```shell
pnpm -Dw add prettier
```

建立`prettier.config.js`文件进行配置

```js
// prettier.config.js
/**
 * @type {import('prettier').Config}
 * @see https://www.prettier.cn/docs/options.html
 */
export default {
  // 指定最大换行长度
  printWidth: 120,
  // 缩进制表符宽度 | 空格数
  tabWidth: 2,
  // 使用制表符而不是空格缩进行 (true：制表符，false：空格)
  useTabs: false,
  // 结尾不用分号 (true：有，false：没有)
  semi: true,
  // 使用单引号 (true：单引号，false：双引号)
  singleQuote: false,
  // 在对象字面量中决定是否将属性名用引号括起来 可选值 "<as-needed|consistent|preserve>"
  quoteProps: "as-needed",
  // 在JSX中使用单引号而不是双引号 (true：单引号，false：双引号)
  jsxSingleQuote: false,
  // 多行时尽可能打印尾随逗号 可选值"<none|es5|all>"
  trailingComma: "none",
  // 在对象，数组括号与文字之间加空格 "{ foo: bar }" (true：有，false：没有)
  bracketSpacing: true,
  // 将 > 多行元素放在最后一行的末尾，而不是单独放在下一行 (true：放末尾，false：单独一行)
  bracketSameLine: false,
  // (x) => {} 箭头函数参数只有一个时是否要有小括号 (avoid：省略括号，always：不省略括号)
  arrowParens: "avoid",
  // 指定要使用的解析器，不需要写文件开头的 @prettier
  requirePragma: false,
  // 可以在文件顶部插入一个特殊标记，指定该文件已使用 Prettier 格式化
  insertPragma: false,
  // 用于控制文本是否应该被换行以及如何进行换行
  proseWrap: "preserve",
  // 在html中空格是否是敏感的 "css" - 遵守 CSS 显示属性的默认值， "strict" - 空格被认为是敏感的 ，"ignore" - 空格被认为是不敏感的
  htmlWhitespaceSensitivity: "css",
  // 控制在 Vue 单文件组件中 <script> 和 <style> 标签内的代码缩进方式
  vueIndentScriptAndStyle: false,
  // 换行符使用 lf 结尾是 可选值 "<auto|lf|crlf|cr>"
  endOfLine: "auto",
  // 这两个选项可用于格式化以给定字符偏移量（分别包括和不包括）开始和结束的代码 (rangeStart：开始，rangeEnd：结束)
  rangeStart: 0,
  rangeEnd: Infinity
};
```

建立`.prettierignore`文件配置忽略项

```yaml
# .prettierignore
dist
public
.local
node_modules
pnpm-lock.yaml
```

在`package.json`中配置`prettier`脚本命令

```json
"scripts":{
    //......其他省略
    "lint:prettier": "prettier --write \"**/*.{js,ts,mjs,cjs,json,tsx,css,less,scss,vue,html,md}\"",
}
```

执行命令

```shell
pnpm run lint:prettier
pnpm lint:prettier
```

#### ESLint

```shell
pnpm -Dw add eslint@latest @eslint/js globals typescript-eslint eslint-plugin-prettier eslint-config-prettier eslint-plugin-vue
```

各个包的作用：

| 类别                 | 库名                                                         |
| -------------------- | ------------------------------------------------------------ |
| **核心引擎**         | `eslint`                                                     |
| **官方规则集**       | `@eslint/js`                                                 |
| **全局变量支持**     | `globals`                                                    |
| **TypeScript 支持**  | `typescript-eslint`                                          |
| **类型定义（辅助）** | `@types/node`                                                |
| **Prettier 集成**    | `eslint-plugin-prettier`：`eslint`插件，将`prettier`的错误报到`eslint`中； `eslint-config-prettier`：解决`prettier`和`eslint`的冲突，当两者规则冲突时，优先按照`prettie`的规则 |
| **Vue.js 支持**      | `eslint-plugin-vue`                                          |

建立`eslint.config.js`配置文件 

```js
import { defineConfig } from "eslint/config";
import eslint from "@eslint/js";
import tseslint from "typescript-eslint";
import eslintPluginPrettier from "eslint-plugin-prettier";
import eslintPluginVue from "eslint-plugin-vue";
import globals from "globals";
import eslintConfigPrettier from "eslint-config-prettier/flat";

const ignores = ["**/dist/**", "**/node_modules/**", ".*", "scripts/**", "**/*.d.ts"];

export default defineConfig(
  // 通用配置
  {
    ignores, // 忽略项
    extends: [eslint.configs.recommended, ...tseslint.configs.recommended, eslintConfigPrettier], // 继承规则
    plugins: {
      prettier: eslintPluginPrettier
    },
    languageOptions: {
      ecmaVersion: "latest", // ecma语法支持版本
      sourceType: "module", // 模块化类型
      parser: tseslint.parser // 解析器
    },
    rules: {
      // 自定义
    }
  },
  // 前端配置
  {
    ignores,
    files: ["apps/frontend/**/*.{ts,js,tsx,jsx,vue}", "packages/components/**/*.{ts,js,tsx,jsx,vue}"],
    extends: [...eslintPluginVue.configs["flat/recommended"], eslintConfigPrettier],
    languageOptions: {
      globals: {
        ...globals.browser
      }
    }
  },
  // 后端配置
  {
    ignores,
    files: ["apps/backend/**/*.{ts,js}"],
    languageOptions: {
      globals: {
        ...globals.node
      }
    }
  }
);
```

在`package.json`中配置`eslint`脚本命令

```json
"scripts":{
    //......其他省略
    "lint:eslint": "eslint",
}
```



#### 拼写检查

vscode插件： Code Spell Checker

安装

```shell
pnpm -Dw add cspell @cspell/dict-lorem-ipsum
```

建立`cspell.json`配置文件

```json
{
  "import": ["@cspell/dict-lorem-ipsum/cspell-ext.json"],
  "caseSensitive": false,
  "dictionaries": ["custom-dictionary"],
  "dictionaryDefinitions": [
    {
      "name": "custom-dictionary",
      "path": "./.cspell/custom-dictionary.txt",
      "addWords": true
    }
  ],
  "ignorePaths": [
    "**/node_modules/**",
    "**/dist/**",
    "**/build/**",
    "**/lib/**",
    "**/docs/**",
    "**/vendor/**",
    "**/public/**",
    "**/static/**",
    "**/out/**",
    "**/tmp/**",
    "**/*.d.ts",
    "**/package.json",
    "**/*.md",
    "**/stats.html",
    "eslint.config.mjs",
    ".gitignore",
    ".prettierignore",
    "cspell.json",
    "commitlint.config.js",
    ".cspell"
  ]
}
```

建立自定义字典

```shell
mkdir -p ./.cspell && touch ./.cspell/custom-dictionary.txt
```

在`package.json`中配置检查脚本

```json
"lint:spellcheck": "cspell lint \"(packages|apps)/**/*.{js,ts,mjs,cjs,json,css,less,scss,vue,html,md}\""
```

### git提交规范

git仓库创建

```shell
touch .gitignore
```

```yaml
# .gitignore
# Node
node_modules/
dist/
build/
.env
.env.*
*.log
npm-debug.log*
yarn-debug.log*
yarn-error.log*
pnpm-debug.log*

# IDE
.vscode/
.idea/
*.suo
*.ntvs*
*.njsproj
*.sln
*.sw?

# OS
.DS_Store
Thumbs.db

# TypeScript
*.tsbuildinfo

# Misc
coverage/
*.local
*.cache
*.tmp

# Git
.git/
```

```shell
git init
```

#### commitizen

Commitizen 是一个用于规范化 Git 提交信息的工具，通过交互式命令行工具让开发者按照约定格式编写提交信息，从而生成清晰、一致的提交历史。安装后使用`git cz`或`cz`替代`git commit`提交就会触发项目维护者配置的规则，在命令行中完善信息即可提交

安装

```shell
pnpm -Dw add commitizen @commitlint/cli @commitlint/config-conventional  cz-git
```

- `commitizen` 提供了一个交互式撰写commit信息的插件
- `@commitlint/cli 是 commitlint` 工具的核心，提供cli命令行界面。
- `@commitlint/config-conventional` 是基于 conventional commits 规范的配置文件。
- [cz-git](https://cz-git.qbb.sh/zh/guide/)国人开发，工程性更强，自定义更高，交互性更好，且命令行是中文界面。

配置脚本命令、适配器

```json
// package.json
"scripts": {
  // 其他省略
	"commit": "git-cz"
},
"config": {
  "commitizen": {
    "path": "node_modules/cz-git"
  }
}
```

创建`commitlint.config.js`配置文件配置`cz-git`

```js
/** @type {import('cz-git').UserConfig} */
export default {
  extends: ["@commitlint/config-conventional"],
  rules: {
    // @see: https://commitlint.js.org/#/reference-rules
    "body-leading-blank": [2, "always"],
    "footer-leading-blank": [1, "always"],
    "header-max-length": [2, "always", 108],
    "subject-empty": [2, "never"],
    "type-empty": [2, "never"],
    "subject-case": [0],
    "type-enum": [
      2,
      "always",
      [
        "feat",
        "fix",
        "docs",
        "style",
        "refactor",
        "perf",
        "test",
        "build",
        "ci",
        "chore",
        "revert",
        "wip",
        "workflow",
        "types",
        "release"
      ]
    ]
  },
  prompt: {
    types: [
      { value: "feat", name: "✨ 新功能: 新增功能" },
      { value: "fix", name: "🐛 修复: 修复缺陷" },
      { value: "docs", name: "📚 文档: 更新文档" },
      { value: "refactor", name: "📦 重构: 代码重构（不新增功能也不修复 bug）" },
      { value: "perf", name: "🚀 性能: 提升性能" },
      { value: "test", name: "🧪 测试: 添加测试" },
      { value: "chore", name: "🔧 工具: 更改构建流程或辅助工具" },
      { value: "revert", name: "⏪ 回滚: 代码回滚" },
      { value: "style", name: "🎨 样式: 格式调整（不影响代码运行）" }
    ],
    scopes: ["root", "backend", "frontend", "components", "utils"],
    allowCustomScopes: true,
    skipQuestions: ["body", "footerPrefix", "footer", "breaking"], // 跳过“详细描述”和“底部信息”
    messages: {
      type: "📌 请选择提交类型:",
      scope: "🎯 请选择影响范围 (可选):",
      subject: "📝 请简要描述更改:",
      body: "🔍 详细描述 (可选):",
      footer: "🔗 关联的 ISSUE 或 BREAKING CHANGE (可选):",
      confirmCommit: "✅ 确认提交?"
    }
  }
};

```

#### husky

安装`husky`

```shell
pnpm -Dw add husky
```

初始化，运行如下命令，会在根目录下生成一个`.husky`文件夹

```cmd
pnpx husky init
```

在`.husky`下的`pre-commit`钩子中配置如下命令

```cmd
#!/usr/bin/env sh
pnpm lint:prettier && pnpm lint:eslint && pnpm lint:spellcheck
```

#### lint-staged

只会检查暂存区的文件

安装

```shell
pnpm -Dw add lint-staged
```

配置命令

```json
// package.json
"scripts": {
  // 其他省略
	"precommit": "lint-staged"
},
```

创建`.lintstagedrc.js`配置文件

```js
export default {
  "*.{js,ts,mjs,cjs,json,tsx,css,less,scss,vue,html,md}": ["cspell lint"],
  "*.{js,ts,vue,md}": ["prettier --write", "eslint"]
};

```



### 公共库打包

安装`rollup`

```shell
pnpm -Dw add rollup @rollup/plugin-node-resolve @rollup/plugin-commonjs rollup-plugin-typescript2 @rollup/plugin-terser @vitejs/plugin-vue rollup-plugin-postcss
```

- `@rollup/plugin-node-resolve`: 解析 node_modules 中的依赖
- `@rollup/plugin-commonjs`: 将 CommonJS 模块转为 ESM
- `rollup-plugin-typescript2`: 让 Rollup 支持 TS 编译
- `@rollup/plugin-terser`： 压缩和混淆
- `@vitejs/plugin-vue`： 支持SFC编译
- `rollup-plugin-postcss`： 处理css代码

#### 具体操作

##### 打包配置与打包脚本

根目录下建立`scripts`文件夹，放置打包脚本

`buildBase.js`：打包基本配置

`build.js`：打包脚本

`dev.js`：启用观察模式的打包脚本，用于开发环境打包

在`package.json`中配置脚本命令

```json
// package.json
"scripts": {
  // 其他省略
	"build": "node ./scripts/build.js",
  "dev": "node ./scripts/dev.js"
},
```

##### 组件库components子包开发

在组件库`components`子包文件夹下的src下新建一个组件文件夹，该文件夹下vue文件为组件文件，其下的index.ts中将组件导入后导出，在子包的src的根目录下的index.ts中统一将所有组件导出

例如如下结构

```ts
# /components/src/组件1/index.ts
export {default as DuyiArea} from './DuyiArea.vue'

# /components/src/index.ts
export * from './DuyiArea.vue'
```

src下建立shims-vue.d.ts声明文件，确保ts正确处理vue文件的导入

配置package.json



##### 工具函数库utils子包开发

在工具函数库`utils`子包文件夹下的src下写每一类工具方法，在index.ts中统一将所有方法导出

例如如下结构

```ts
# /utils/src/index.ts
export * from './xxx'
```

配置package.json

```json
{
  "main": "./dist/index.cjs.js",
  "module": "./dist/index.esm.js",
  "types": "./dist/index.d.ts",
  "exports": {
    ".": {
      "import": "./dist/index.esm.js",
      "require": "./dist/index.cjs.js",
      "types": "./dist/index.d.ts"
    }
  },
  "files": [
    "dist"
  ],
}

```



### 单元测试

安装

```shell
pnpm -Dw add vitest @vitest/browser vitest-browser-vue vue
```

`package.json`中添加脚本命令

```json
"test": "vitest"
```

建立`vitest.config.ts`配置文件

```ts
import { defineConfig } from "vitest/config";
import vue from "@vitejs/plugin-vue";
export default defineConfig({
  test: {
    projects: [
      {
        test: {
          globals: true,
          name: "utils",
          include: ["packages/utils/__test__/**/*.{test,spec}.{ts,js,tsx,jsx}"],
          environment: "node"
        }
      },
      {
        plugins: [vue()],
        test: {
          globals: true,
          name: "ui",
          include: ["packages/components/__test__/**/*.{test,spec}.{ts,js,tsx,jsx}"],
          browser: {
            enabled: true,
            instances: [{ browser: "chromium" }]
          }
        }
      }
    ]
  }
});

```

更改项目根目录下的`tsconfig.json`

```json
"types": ["vitest/globals", "@vitest/browser/matchers"],
"lib": ["esnext", "DOM"],
```

配置好后运行`pnpm test`命令就会去`utils`和`components`子包中找`__test__`测试文件夹下的`xx.test.ts/xx.spec.ts`测试脚本进行测试



### 发布



