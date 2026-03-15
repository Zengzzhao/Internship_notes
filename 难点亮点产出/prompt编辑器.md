# 功能

xml高亮

使用# xxx高亮

{{}}变量（1.输入{自动补全}；2.{{}}变量捕获高亮效果；3.有的需要支持变量后跟随笔按钮，点击后弹出变量表单填写变量值，有值没值的变量高亮颜色有区分，支持AI一键生成变量）

浮层（1.输入{{}}弹出浮层展示当前智能体设置的配置、自定义变量，选中后回填到编辑器；2.点击浮层中的管理自定义变量按钮，弹出管理自定义变量弹窗可对变量的类型、变量名、是否必填、默认值、描述等信息crud，保存后同步到编辑器的自定义变量中）



# 约定

## 数据格式约定

**编辑器内容格式**：

- 传递给编辑器的内容为纯文本 (text)
- 编辑器传出的内容也是纯文本 (text)
- 编辑器内部自动将纯文本处理渲染成 HTML 到页面



# 应用场景

## 应用模块

充当零代码智能体的prompt编辑器

功能：xml高亮、# xxx高亮、浮层显示

## prompt模块

**创建prompt、展示prompt详情内容**

功能：xml高亮、# xxx高亮

**调试prompt：输入prompt与模型对话看结果是否满意**

调试prompt有两种模式：一种常规模式，一种上下文模式（基于之前轮的消息进行回答）；模式切换时常规模式内容与上下文模式的System的内容互转。

功能：xml高亮、# xxx高亮，填写变量的值（通过变量后的一个笔按钮，点击弹出变量表单，光标聚焦对应表单项供用户输入实现）

特别的，在上下文模式中，System、每个User、每个Assistant都使用了不同的编辑器，每个编辑器点击变量笔按钮打开对应编辑器的变量表单，点击全局的填写变量按钮弹出全局变量表单，但是相同变量名在整个上下文中都是同一个，即某个编辑器中、全局表单中修改了变量值，其他编辑器中、全局表单中同名的变量值都需要同步。

# 实现

## 变量系统的设计

在没有自定义变量弹窗前，只有编辑器侧可以修改变量，所以使用单一store一套状态

有了自定义变量弹窗后，变为编辑器侧、弹窗侧两个地方可以修改变量

核心问题是：之前编辑器所显示的变量就是全部的实际变量，现在变为编辑器所显示的变量不一定是实际变量、也不一定是全部的实际变量（有的实际变量通过弹窗添加却没有通过编辑器键入）

### 编辑器侧更新

**按delete实现删除**

核心：本次删去的变量是否是编辑器内该变量最后一个，如果是最后一个则删除，如果不是则说明前面还有该变量不能从实际变量中删除

代码实现：每次内容更新时，用上一次所有变量与本次所有变量对比，不在上一次变量中出现的变量就是要删除的实际变量。

```
{{a}}{{b}}{{a}}
{{a}}{{b}}
此时不触发，因为前面有变量a，我删后面的a，前一次与本次变量是相同的，不会触发更新实际变量。

{{a}}{{b}}
{{a}}
此时触发，因为我删除的变量b就是最后一个，前一次与本次变量相差的是b，触发更新实际变量。
```

**按非delete实现添加**

核心：将本次输入的变量变为真实变量

代码实现：每次内容更新时，用上一次所有变量及其出现次数的map与本次所有变量及其出现次数的map对比，次数增加的就是要添加的实际变量

```
{{a}}(a在弹窗中被删除)
{{a}}{{a}}
此时触发，因为前后比变量a数量增加了，触发更新实际变量
```

### 弹窗侧更新

窗删内变量直接覆盖之前的实际变量



# 未来

## 变量系统增强

### 变量类型扩展

**当前支持**：

- `text-input`: 普通文本变量
- `file`: 文件类型变量（`image::`、`video::`、`audio::` 前缀）

**扩展方向**：

```typescript
// 新增变量类型
enum VariableType {
  TEXT = 'text',
  FILE = 'file',
  SELECT = 'select',        // 下拉选择
  MULTI_SELECT = 'multi-select', // 多选
  DATE = 'date',            // 日期
  NUMBER = 'number',        // 数字
  JSON = 'json',            // JSON 对象
  CODE = 'code'             // 代码块
}

// 变量配置扩展
interface VariableConfig {
  key: string;
  type: VariableType;
  defaultValue?: any;
  options?: Array<{ label: string; value: any }>; // 用于 select
  validation?: {
    required?: boolean;
    min?: number;
    max?: number;
    pattern?: RegExp;
    custom?: (value: any) => boolean | string;
  };
  display?: {
    label?: string;
    placeholder?: string;
    description?: string;
  };
}
```

**实现要点**：

1. 扩展 `VariableMark` 解析变量类型语法：

   ```
   {{userName}}           // 默认 text
   {{age:number}}         // 数字类型
   {{gender:select}}      // 下拉选择
   {{tags:multi-select}}  // 多选
   ```

2. `PromptParameterEditor` 根据类型动态渲染输入组件：

   ```vue
   <component
     :is="getInputComponent(variable.type)"
     v-model="variables[key]"
     v-bind="getInputProps(variable)"
   />
   ```

### 变量依赖与联动

**场景**：变量 B 的可选项依赖于变量 A 的值。

```typescript
interface VariableDependency {
  target: string;  // 目标变量
  source: string;  // 依赖的源变量
  transform: (sourceValue: any) => any; // 转换函数
}

// 示例：城市依赖于省份
const dependencies: VariableDependency[] = [
  {
    target: 'city',
    source: 'province',
    transform: (province) => getCitiesByProvince(province)
  }
];
```

### 变量模板库

**功能**：保存常用变量组合为模板。

```typescript
interface VariableTemplate {
  id: string;
  name: string;
  description: string;
  variables: Record<string, VariableConfig>;
  tags: string[];
}

// 使用场景
const emailTemplate: VariableTemplate = {
  id: 'email-001',
  name: '邮件发送模板',
  variables: {
    'recipient': { type: 'text', validation: { required: true, pattern: EMAIL_REGEX } },
    'subject': { type: 'text', validation: { required: true } },
    'body': { type: 'text', validation: { required: true } }
  }
};
```

## 编辑器功能增强

### 条件渲染语法

**需求**：支持根据变量值动态显示不同内容。

```
{{#if isPremiumUser}}
  尊敬的高级会员，您好！
{{else}}
  尊敬的用户，您好！
{{/if}}
```

**实现思路**：

1. 创建 `ConditionalBlock` Node Extension
2. 解析 `{{#if}}` `{{else}}` `{{/if}}` 语法
3. 在最终渲染时根据变量值选择分支

```typescript
export const ConditionalBlock = Node.create({
  name: 'conditionalBlock',
  group: 'block',
  content: 'block+',
  addAttributes() {
    return {
      condition: { default: '' },
      branch: { default: 'if' } // 'if' | 'else'
    };
  },
  parseHTML() {
    return [{ tag: 'div[data-conditional-block]' }];
  },
  renderHTML({ node, HTMLAttributes }) {
    return ['div', { ...HTMLAttributes, 'data-conditional-block': '' }, 0];
  }
});
```

### 循环渲染语法

**需求**：支持遍历数组变量。

```
{{#each products}}
  - {{this.name}}: {{this.price}}
{{/each}}
```

**实现思路**：

1. 创建 `LoopBlock` Node Extension
2. 解析 `{{#each}}` `{{/each}}` 语法
3. 在渲染时重复内容块

## 协同编辑

### 实时协作

**技术方案**：

- **Yjs + y-prosemirror**：分布式 CRDT 算法
- **WebSocket**：实时通信
- **Hocuspocus**：Yjs 后端服务

```typescript
import { ySyncPlugin, yCursorPlugin, yUndoPlugin } from 'y-prosemirror';
import * as Y from 'yjs';
import { WebsocketProvider } from 'y-websocket';

const ydoc = new Y.Doc();
const provider = new WebsocketProvider('ws://localhost:1234', 'prompt-editor', ydoc);

const editor = useEditor({
  extensions: [
    // ... 其他扩展
    ySyncPlugin(ydoc.getXmlFragment('prosemirror')),
    yCursorPlugin(provider.awareness),
    yUndoPlugin()
  ]
});
```

### 版本历史

**功能**：记录编辑历史，支持回溯。

```typescript
interface EditorVersion {
  id: string;
  timestamp: number;
  content: string;
  variables: Record<string, string>;
  author: { id: string; name: string };
  comment?: string;
}

// 自动保存版本
let autoSaveTimer: any;
watch(modelContent, () => {
  clearTimeout(autoSaveTimer);
  autoSaveTimer = setTimeout(() => {
    saveVersion({
      content: modelContent.value,
      variables: variables.value
    });
  }, 5000); // 5秒后自动保存
});
```

## 性能优化

### 延迟加载装饰器

**优化**：只为可见区域的变量渲染编辑按钮。

```typescript
addProseMirrorPlugins() {
  return [
    new Plugin({
      props: {
        decorations: state => {
          const { from, to } = getVisibleRange(state); // 获取可见区域
          const editBtns: Decoration[] = [];
          state.doc.nodesBetween(from, to, (node, pos) => {
            // 只处理可见区域内的节点
            if (hasVariableMark(node)) {
              editBtns.push(createEditButton(node, pos));
            }
          });
          return DecorationSet.create(state.doc, editBtns);
        }
      }
    })
  ];
}
```



