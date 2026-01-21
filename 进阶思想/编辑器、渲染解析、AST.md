# 专业术语

MVC：Model管数据、View管显示、Controller管操作，把数据、界面、控制逻辑分离，降低耦合，便于维护和扩展

Undo：撤销。撤销上一次操作，回到操作前的状态。`command Z`

Redo：重做，重新执行被撤销的操作。`command Y`

WYSIWYG ：What You See Is What You Get，所见即所得



# 常见编辑器框架

ProseMirror

Tiptap：基于ProseMirror，基于该框架开发的产品：Team

Quill：基于该框架开发的产品：以前的KStack

CKEditor：基于该框架开发的产品：CSDN、现在的KStack

Medium Editor：基于该框架开发的产品：稀土掘金

**md编辑器**

Milkdown：基于ProseMirror、Remark

StackEditor：基于该框架开发的产品：CSDN

Bytemd：基于该框架开发的产品：稀土掘金

# ProseMirror

基于Schema（结构文档模型）+ Transaction（事务）+ State（状态）+ Plugins（插件）的富文本编辑框架。

ProseMirror不是操作HTML的dom元素，而是操作一个类似Json、AST的树形文档结构。为了在浏览器中呈现编辑器内容需要将文档结构转换为dom元素，为了保存编辑内容需要将dom元素转换为文档结构

## 基本模块

Prosemirror-model：定义编辑器的文档模型，描述编辑器内容的数据结构

Prosemirror-state：描述编辑器整个状态的数据结构，包括光标选区、从一个状态转移到另一个状态的事务系统

Prosemirror-view：将编辑器状态转换为浏览器dom元素，并处理用户与该元素的交互

Prosemirror-transform：用可记录和可重放的方式修改文档的功能，支持撤销历史记录和协作编辑

## 核心概念

Schema：定义文档可以有哪些内容，由Node和Mark组成，Node是文档中的某个内容元素，如 `doc`、`paragraph`、`text`；Mark是附着在text文本节点上的样式，如 `bold`、`italic`、`underline`。

> 文档树中节点分为两类：Node（类型结构节点，以纯文本节点作为子节点，比如'paragraph'、'heading'），TextNode（纯文本节点，没有子节点，即叶子节点，比如'hello'、'你好'等）

State：文档存储在state中，所有对编辑器的操作都通过transaction应用到state上。state包含了当前文档doc、当前光标选区selection、当前激活的标记storedMarks、插件状态pluginStates

```
state.doc:当前文档
state.selection:当前光标选区
state.storedMarks:存储的 Mark（空选区下的“将应用的样式”）
state.schema:Schema
state.plugins:当前启用的 PM 插件数组
state.tr:创建一个新的 Transaction
```

> doc上的属性与方法：
>
> ```
> doc.type：节点类型（Schema 中的 NodeType，例如 doc、paragraph）
> doc.attrs：节点属性对象（如 {level: 1}）
> doc.content：Fragment，子节点集合容器
> doc.marks：节点上的 mark 列表（文本节点常有）
> 
> doc.child(index)：获取第 index 个子节点
> doc.textBetween(from, to, blockSeparator?, leafText?)：读取文档片段文本
> doc.descendants((node,pos,parent,index)=>booleam|void)：对文档树的深度优先遍历DFS
> 回调中的参数：node为当前遍历到的节点,pos为当前节点在文档中的绝对位置,parent为当前节点的父节点,index为当前节点是父节点的第几个子节点
> 回调返回false则跳过该节点的子节点，不在继续深入
> ```
>
> selection上的属性与方法
>
> ```
> selection.from：选择起点位置（number）
> selection.to：选择终点位置（number）
> selection.anchor：锚点位置（number）
> selection.head：头部位置（number）
> selection.empty：是否为空选区（光标态）
> selection.$from / selection.$to：ResolvedPos解析位置（包含更丰富的位置信息）,内部包含pos在整篇文档里的绝对数字位置,parent为所在父节点
> selection.ranges：范围列表（一般单一范围，少用）
> 
> selection.eq(other)：是否与另一选择相等
> selection.content()：返回被选中的 Slice
> ```
>
> 

Transaction：所有对编辑器的操作都必须通过Transaction完成，提交Transaction生成一个新的state更新视图

Plugin：提供了props（提供钩子，拦截dom事件）、state（插件自己的state）、view（视图生命周期）

Decoration：不属于schema，仅用于UI显示

> 装饰器允许你控制视图绘制文档的方式
>
> 分为三类：
>
> Node decorations节点装饰器：为单个节点的 DOM 表示添加样式或其他 DOM 属性。
>
> Widget decorations小部件装饰：会在指定位置*插入*一个 DOM 节点，该节点不属于实际文档的一部分。
>
> Inline decorations内联装饰器：可以像节点装饰器一样添加样式或属性，但它是添加到给定范围内的所有内联节点。
>
> ```js
> Decoration.widget(pos, domOrFactory, options?)
> ```
>
> 

InputRules：输入规则

EditorView：渲染文档结构为真实dom



# Tiptap

基于ProseMirror的无头（不提供样式）富文本编辑器框架。tiptap的所有功能都通过extension组成，tiptap将每个extension解析成prosemirror构件，简化了prosemirror

## Command

底层创建一个transaction，改变编辑器状态（内容、选区等）

每个command命令只会作用于当前selection选区，仅返回true/false

### 使用command的两种方式

**直接调用**

```ts
editor.commands.xxx()
// 每次只能执行一个command，使用后立即执行
```

**链式调用**

```ts
editor.chain().xxx1().xxx2().xxx3().run()
```

`.chain()`创建一个command队列，将xxx1()、xxx2()、xxx3()依次加入到队列中，`.run()`时才会依次执行队列中所有command



## 三大件

### Node



### Mark



### Extension

tiptap所有功能通过extension实现

### 常用钩子

#### 属性

name：声明扩展名称

priority：优先级，决定扩展的注册顺序，优先级高的扩展优先运行

inclusive：标记是否节点的起始点、结束点，为true时表示包含左右边界点，此时光标在右边界继续输入会继承此标记

#### addOptions()

设置扩展的配置，用户可以在注册扩展时通过configure()在置扩展的配置项

#### addAttributes()

在Node/Mark中声明可以存在于schema中的属性/数据以及其默认值、html解析规则、html渲染规则

相当于给Node/Mark增加字段，被存进prosemirror的schema中，能写入dom元素中并从dom元素中解析出来，可以通过commands修改

```ts
// 基本结构如下
addAttributes(){
  return{
    '属性名':{
      default:默认值,
      parseHTML:element=>返回属性值
      renderHTML:attributes=>返回HTML属性
    }
  }
}
```



#### parseHTML()

HTML → Doc  解析外部内容（setContent、paste）

#### renderHTML()

Doc → HTML  导出 HTML 时使用

#### addCommands()

注册命令  可通过 `editor.commands.xxx()` 调用

#### addInputRules()

**功能：**定义输入触发规则，在用户通过编辑器真实输入且没有被阻止的情况下才会触发输入规则进行匹配（如果在特定按键输入的拦截中return true阻止了默认字符输入则不会触发输入规则；通过命令`insertContent()`编程式插入也不会触发输入规则），用于自动转换输入（如 `**bold**`）

**多个扩展的输入规则的执行先后顺序：**priority高的优先执行，priority相同时按照扩展注册的顺序执行，先匹配到的规则将后续规则短路

**与键盘监听的执行先后顺序：**晚于addKeyboardShortcuts键盘监听执行

#### addPasteRules()

定义粘贴触发规则，用于粘贴内容自动格式化

#### addKeyboardShortcuts()

**功能：**定义快捷键（如 `Cmd+B` 加粗）、拦截特定的按键输入进行特殊处理

**返回值：**返回true表示当前键盘输入已经被扩展处理，阻止默认的字符输入，并阻断后续扩展的同键处理；返回false表示不会阻止默认字符输入，同时交给下一个扩展继续处理

**多个扩展对相同按键拦截的执行先后顺序：**priority高的优先执行，priority相同时按照扩展注册的逆序执行，后注册的扩展先执行

**与输入规则的执行先后顺序：**先于addInputRules输入规则执行

#### addProseMirrorPlugins()

注入原生 PM 插件  高级行为定制

#### 事件

### 核心方法

getMarkRange(ResolvedPos,markType)：传入一个ResolvedPos和某个mark类型，返回该mark的整体范围{from:number,to:number}，用于快速获取整个mark边界进行整体删除、替换等操作



## 保存与回写

TipTap编辑器的内容可以存储为Json格式/Html字符串，且两者都可以传入编辑器进行内容回写。使用`getJSON()`即可获得编辑器中富文本内容的Json格式，使用`getHTML()`即可获得编辑器中富文本内容的Html字符串



# Quill

## Delta

Quill的内容数据格式，本质上是一个json，只有一个属性ops，ops是一个对象数组，每一项代表对编辑器的一个操作

```json
{
  "ops": [
    { "insert": "Hello " },
    { "insert": "World", "attributes": { "bold": true } },
    { "insert": "\n" }
  ]
}
```

一个操作op由操作和attributes格式两个属性组成

> 操作有三种可能：insert插入、delete删除、retain保留
>
> arributes分两种：
>
> 行内样式作用在普通字符上
>
> ```json
> {
>   insert:"World",
>   attributes:{bold:true}
> }
> ```
>
> 行级样式作用在换行符\n上，因为Quill文档就是一个字符流，没有段落节点，用\n作为行结束的标志
>
> ```json
> {
>   insert:"\n",
>   attributes:{header:"1"}
> }
> ```
>

**常见的Detla**

```json
// 插入Hello World
{
  ops:[
    {insert:"Hello World"},
    {insert:"\n"}
  ]
}
// 将Hello World的World改成红色
{
  ops:[
    {retain:6}, // 保留前面6个字符
    {retain:5,attributes:{color:"#ff0000"}} // 保留接下来后面5个字符，格式颜色使用#ff0000
  ]
}
// 插入图片
{
  ops:[
    {insert:{ "image": "https://quilljs.com/assets/images/logo.svg" }},
    {insert:'\n'}
  ]
}
// 插入公式
{
  ops:[
    {insert:{ "formula": "e=mc^2" }},
    {insert:'\n'}
  ]
}
```



## Parchment与Blot

Parchment是Quill的文档模型，是Quill世界里面的Html+Css；Bolt是组成Parchment的节点。它们决定了文档中哪些结构合法、有什么样的格式



## Modules

是Quill的功能插件系统，Quill 一共有6个内置模块：Toolbar 工具栏、Keyboard 快捷键、History 操作历史（每隔1s记录一次操作历史，并放入历史操作栈中（最大100），便于撤销与回退）、Clipboard 粘贴版（处理复制/粘贴事件、html元素节点匹配、html到delta的转换）、Syntax 语法高亮、Uploader 文件上传

自定义Modules

```js
// Counter.js
class Counter {
  constructor(quill, options) {
    // 依赖注入，控制反转，拿到quil实例可以做一些自定义事情了
  }
}

// index.vue
import Quill from 'quill';
import Counter from './counter';
Quill.register('modules/counter', Counter);
const editor = new Quill("#editor", {
    theme: "snow",
    modules: {
      counter: true,
    },
  });
```



> 三者的关系：
>
> 自定义内容靠Bolt，保存回显靠Delta，交互靠Modules



## API

### Content

`getContents`：获取当前文档状态的Delta，并不会以操作历史的操作流的形式展示，而是将当前文档按行展示

`setContents`：给编辑器设置Delta内容，用当前Delta内容完整替换当前文档，不会导致样式丢失，用来md结果回填

`setText`：给编辑器设置纯文本内容（本质是清空编辑器，插入纯文本，自动补一个\n），将当前纯文本内容作为一次op全部插入，会导致样式丢失，不用来md结果回填

### Event

```js
editor.on('text-change',(delta,oldDelta,source)=>{
  // delta:本次变更的delta,以操作历史的操作流的形式展示
  // oldDelta:变更前的delta,以操作历史的操作流的形式展示
})
```



# Babel

@babel/parser——将代码转为AST

@babel/traverse——对AST进行操作

@babel/generator——将AST转为代码文件



# Markdown解析渲染

Marked、showdown：将Markdown转为Html的解析器

Highlight.js：基于正则的代码高亮库

Shiki：基于 VS Code TextMate 语法的高亮引擎

## marked

marked本质上是一个正则驱动的匹配解析引擎

**marked工作原理：**

词法分析Lexer（lexical analyzer词法分析器的简写），顺序遍历md文本字符串，用正则匹配语法片段，将字符串拆解成一个个有意义的单元token，最终得到一个线性token序列

语法分析Parser，遍历词法分析得到的线性token序列，生成一个结构化token（不是AST，AST是可逆的树形结构，可以根据AST还原原来的）

渲染Render，把结构化token转成html

> Lexer是一个基于DFA（Deterministic finite automaton）确定有限状态机的词法分析器



## unified

unified：一个文本处理的生态系统，其生态中有诸多插件，unified作为一个统一的执行接口将各个插件串联起来，将AST在各阶段流转。有三类接口插件Parser（文本->AST）、Transformer（AST->AST）、Compiler（AST->文本）。Markdown用mdast，HTML用hast

### 插件生态

remark：Markdown相关插件集合

rehype：HTML相关插件集合

retext：自然语言处理相关插件集合



## rehype插件开发

基本形式

```ts
import type { Plugin } from 'unified';
import { visit } from 'unist-util-visit'; // 遍历AST的工具

export const myRehypePlugin: Plugin<[Options?]> = (options) => {
  // 第1层函数：接收插件配置 options
  // 返回的第2层函数：接收AST，做Transformer变形
  return (tree, file) => {
    // 在 tree 上做遍历/替换/增删改节点
    visit(tree, 'element', (node: any, index: number | undefined, parent: any) => {}
  };
};
```



## 场景

控制表格，给table元素外部包裹一个div、给th、td元素外部包裹一个span，已实现对表格的css精细控制

给table元素外部包裹一个div

`rehypeWrapTables` 插件本质是在rehype的HAST(AST)树上做一次遍历：找到所有 `<table>` 节点，然后把它替换成 `<div class="xxx"><table>...</table></div>`，从而给 table 增加一个“可滚动/可控布局”的外层容器。

```ts
// 给table元素包裹一个div以实现横向滚动
import type { Plugin } from 'unified';
import { visit } from 'unist-util-visit';

type Element = {
    type: 'element';
    tagName: string;
    properties?: Record<string, any>;
    children: any[];
};

function isElement(node: any): node is Element {
    return node && node.type === 'element' && typeof node.tagName === 'string' && Array.isArray(node.children);
}

export const rehypeWrapTables: Plugin<[{ className?: string }?]> = options => {
    const className = options?.className ?? 'mc-table-scroll'; // 获取配置参数中的className属性
    return (tree: any) => {
        visit(tree, 'element', (node: any, index: number | undefined | null, parent: any) => {
            if (!parent || index == null) return;
            if (!isElement(node)) return;
            if (node.tagName !== 'table') return;
            // 只处理table元素
            // 避免重复包裹：如果table的父元素已经被我们标注的带有指定class的div包裹，则跳过
            if (isElement(parent) && parent.tagName === 'div') {
                const cls = parent.properties?.className;
                const has = Array.isArray(cls) ? cls.includes(className) : cls === className;
                if (has) return;
            }
            const wrapper: Element = {
                type: 'element',
                tagName: 'div',
                properties: { className: [className] },
                children: [node],
            };
            parent.children[index] = wrapper;
        });
    };
};

```



给th、td元素外部包裹一个span

把每个表格单元格 `<td>`/`<th>` 里面的内容统一包一层 `<span class="xxx">...</span>`

```ts
import type { Plugin } from 'unified';
import { visit } from 'unist-util-visit';

type Element = {
    type: 'element';
    tagName: string;
    properties?: Record<string, any>;
    children: any[];
};

function isElement(node: any): node is Element {
    return node && node.type === 'element' && typeof node.tagName === 'string' && Array.isArray(node.children);
}

export const rehypeWrapTableCell: Plugin<[{ className?: string }?]> = options => {
    const className = options?.className ?? 'mc-thtd-content';

    return (tree: any) => {
        visit(tree, 'element', (node: any) => {
            if (!isElement(node)) return;
            if (node.tagName !== 'td' && node.tagName !== 'th') return;
            if (!node.children || node.children.length === 0) return;

            // 只处理th、td元素
            // 避免重复包裹：如果th、td的第一个子元素span已经是我们标注的带有指定class的span，则跳过
            if (node.children.length === 1 && isElement(node.children[0]) && node.children[0].tagName === 'span') {
                const first = node.children[0] as Element;
                const cls = first.properties?.className;
                const has = Array.isArray(cls) ? cls.includes(className) : cls === className;
                if (has) return;
            }

            const wrapped: Element = {
                type: 'element',
                tagName: 'span',
                properties: { className: [className] },
                children: node.children,
            };

            node.children = [wrapped];
        });
    };
};

```
