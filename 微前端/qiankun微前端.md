# qiankun 微前端学习文档

> 本文档记录了 `streamlake-console-wanqing` 项目中 qiankun 微前端的使用方式、通信机制和路由配置。

---

## 1. 项目定位

**这是一个 qiankun 子应用**，不是主应用。

- **项目名称**: `streamlake-console-wanqing`
- **角色**: 微前端子应用
- **输出格式**: UMD
- **挂载元素**: `#wanqing-root`

子应用被嵌入到主应用中运行，不能独立提供完整的用户界面（如左侧菜单栏）。

---

## 2. qiankun 是什么

qiankun 是蚂蚁金服开源的微前端框架，基于 single-spa 封装，允许多个独立的前端应用组合在一起运行。

### 核心概念

| 概念 | 说明 |
|------|------|
| **主应用 (Base App)** | 负责加载子应用、提供公共布局（菜单、导航等） |
| **子应用 (Micro App)** | 独立开发、独立部署的前端应用，被主应用动态加载 |
| **沙箱隔离** | 子应用的 JS 和 CSS 相互隔离，互不影响 |
| **全局状态** | 主应用和子应用之间通过全局状态进行通信 |

### 优势

- 技术栈无关：子应用可以使用不同的框架（Vue、React、Angular）
- 独立开发部署：各团队独立开发，独立部署
- 增量升级：逐步迁移老项目，无需一次性重构

---

## 3. 架构图解

```
┌─────────────────────────────────────────────────────────────┐
│                    主应用 (Base App)                         │
│  ┌──────────────┐  ┌────────────────────────────────────┐  │
│  │   左侧菜单    │  │           子应用渲染区域              │  │
│  │              │  │  ┌────────────────────────────────┐ │  │
│  │  - 模型广场   │  │  │                                │ │  │
│  │  - 模型推理   │  │  │   streamlake-console-wanqing   │ │  │
│  │  - 数据集     │  │  │        (当前项目)               │ │  │
│  │  - 模型微调   │  │  │                                │ │  │
│  │  - 模型部署   │  │  └────────────────────────────────┘ │  │
│  └──────────────┘  └────────────────────────────────────┘  │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              全局状态通信 (GlobalState)               │   │
│  │  { externalProjectId, projectName, projectId, ... }  │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

**为什么没有左侧菜单栏？**

因为菜单栏由主应用统一提供和管理，子应用只负责内容区域的渲染。这样可以：
- 保持整体 UI 一致性
- 统一管理导航和权限
- 减少重复代码

---

## 4. 生命周期函数

qiankun 子应用需要导出三个生命周期函数，定义在 `src/index.ts`：

### 4.1 bootstrap - 启动

```typescript
// src/index.ts:118-121
export async function bootstrap() {
    // 应用启动时调用，只会调用一次
    console.log('wanqing bootstrap');
}
```

**调用时机**: 子应用首次加载时调用一次，可用于全局初始化。

### 4.2 mount - 挂载

```typescript
// src/index.ts:123-182
export async function mount(props: {
    container?: Element;
    currentMicroAppActions: MicroAppStateActions
}) {
    // 1. 渲染应用
    render(props.container);

    // 2. 保存主应用传递的通信对象
    const projectStore = useProjectStore();
    if (props.currentMicroAppActions) {
        currentMicroAppActions = props.currentMicroAppActions;

        // 3. 通知主应用当前子应用已挂载
        setGlobalStateSafe({
            currentMicroApp: 'streamlake-console-wanqing',
        });

        // 4. 处理挂载前缓存的状态
        if (pendingGlobalStateQueue.length > 0) {
            const mergedState = pendingGlobalStateQueue.reduce(
                (acc, item) => Object.assign(acc, item), {}
            );
            props.currentMicroAppActions.setGlobalState(mergedState);
            pendingGlobalStateQueue.length = 0;
        }

        // 5. 同步路由中的项目 ID
        await router.isReady();
        const pid = router.currentRoute.value.params.pid as string;
        if (pid && pid !== projectStore.projectId) {
            setGlobalStateSafe({ externalProjectId: pid });
            projectStore.projectId = pid;
        }

        // 6. 监听主应用状态变化
        props.currentMicroAppActions.onGlobalStateChange(state => {
            if (state.externalProjectId !== projectStore.projectId) {
                projectStore.projectId = state.externalProjectId;
                projectStore.projectName = state.projectName;
                projectStore.steamLakeProjectId = state.projectId;
                // 项目切换时重定向到列表页
                debouncedRouteReplace();
            }
        }, true);
    }
}
```

**调用时机**: 子应用每次进入时调用。

### 4.3 unmount - 卸载

```typescript
// src/index.ts:184-191
export async function unmount() {
    if (app) {
        await userInfoManager.stopTimer();
        app.unmount();
        app = null;
        console.log('wanqing unmount');
    }
}
```

**调用时机**: 子应用切出时调用，用于清理资源。

---

## 5. 与主应用通信

qiankun 提供全局状态管理来实现主应用和子应用之间的通信。

### 5.1 通信方式

```
┌─────────────┐                    ┌─────────────┐
│   主应用     │  setGlobalState   │   子应用     │
│            │ ←───────────────── │            │
│            │ ──────────────────→ │            │
│            │ onGlobalStateChange │            │
└─────────────┘                    └─────────────┘
```

### 5.2 子应用 → 主应用（发送状态）

```typescript
// src/index.ts:28-35
export function setGlobalStateSafe(state: Record<string, unknown>) {
    console.log('set currentMicroAppActions: ', JSON.stringify(currentMicroAppActions));
    if (currentMicroAppActions?.setGlobalState) {
        // 已初始化，直接发送
        currentMicroAppActions.setGlobalState(state);
    } else {
        // 未初始化，缓存起来等挂载后发送
        pendingGlobalStateQueue.push(state);
    }
}
```

**使用示例**：

```typescript
// 通知主应用当前激活的子应用
setGlobalStateSafe({
    currentMicroApp: 'streamlake-console-wanqing',
});

// 通知主应用项目切换
setGlobalStateSafe({
    externalProjectId: pid,
});

// 通知主应用权限不足（触发主应用显示提示）
setGlobalStateSafe({ noPermissionTip: true });
```

### 5.3 主应用 → 子应用（监听状态）

```typescript
// src/index.ts:167-179
props.currentMicroAppActions.onGlobalStateChange(state => {
    // 当主应用切换项目时
    if (state.externalProjectId !== projectStore.projectId) {
        console.log('global state change', state.externalProjectId, projectStore.projectId);

        // 更新本地状态
        projectStore.projectId = state.externalProjectId;
        projectStore.projectName = state.projectName;
        projectStore.steamLakeProjectId = state.projectId;

        // 路由重定向到列表页
        debouncedRouteReplace();
    } else if (state.projectId !== projectStore.steamLakeProjectId) {
        projectStore.steamLakeProjectId = state.projectId;
        projectStore.projectName = state.projectName;
    }
}, true); // true 表示立即执行一次
```

### 5.4 全局状态数据结构

```typescript
interface GlobalState {
    currentMicroApp: string;       // 当前激活的子应用标识
    externalProjectId: string;     // 外部项目 ID（万倾项目 ID）
    projectName: string;           // 项目名称
    projectId: string;             // StreamLake 项目 ID
    noPermissionTip: boolean;      // 权限不足提示
}
```

### 5.5 权限错误通信示例

```typescript
// src/common/request/axios.ts:72-85
const handleErrorInterceptor = {
    fail: (err: AxiosError) => {
        if (err.response) {
            // 检测权限错误
            if (err.response.data?.response?.ResponseMeta?.ErrorMessage?.startsWith(
                'current subject not allow to access Action',
            )) {
                // 通知主应用显示权限不足提示
                setGlobalStateSafe({ noPermissionTip: true });
            }
        }
        return Promise.reject(err);
    },
};
```

---

## 6. 路由系统

### 6.1 路由配置

路由定义在 `src/router.ts`：

```typescript
// src/router.ts:30-33
const router = createRouter({
    // 根据是否被 qiankun 加载来设置 base 路径
    history: createWebHistory(window.__POWERED_BY_QIANKUN__ ? '/wanqing' : '/'),
    routes,
});
```

- **qiankun 模式**: 路由 base 为 `/wanqing`
- **独立运行模式**: 路由 base 为 `/`

### 6.2 动态路由加载

所有模块路由通过 webpack 的 `require.context` 动态加载：

```typescript
// src/router.ts:10-16
const context = require.context('./modules', true, /([\w\d-]+)\/routes\.ts/);

const moduleRoutes: RouteRecordRaw[] = context
    .keys()
    .flatMap((id: string) => es6Unwrap(context(id)))
    .filter(Boolean);
```

### 6.3 项目 ID 参数注入

所有子路由都会自动添加 `:pid?` 可选参数：

```typescript
// src/router.ts:18-28
const routes: RouteRecordRaw[] = [
    {
        path: '/',
        component: () => import('./App.vue'),
        redirect: { name: isSGP ? 'ModelInferenceList' : 'ModelList' },
        children: moduleRoutes.map(route => ({
            ...route,
            path: `:pid?${route.path}`, // 在每个子路由前添加可选参数 pid
        })),
    },
];
```

### 6.4 路由守卫

自动注入项目 ID 到路由参数：

```typescript
// src/router.ts:35-54
router.beforeEach((to, from, next) => {
    const needPid = to.matched.some(r => r.path.includes(':pid'));
    if (!needPid || to.params.pid) {
        next();
        return;
    }

    const projectStore = useProjectStore();
    const pid = projectStore.projectId;

    if (pid) {
        next({
            ...to,
            params: { ...to.params, pid },
        } as RouteLocationRaw);
        return;
    }

    next();
});
```

### 6.5 模块路由配置

各模块的路由定义在 `src/modules/*/routes.ts`：

#### 模型仓库 (model-store)

```typescript
// src/modules/model-store/routes.ts
const routes: RouteRecordRaw[] = [
    {
        path: '/model-customization/model-store',
        name: 'ModelStore',
        meta: {
            title: '模型仓库',
            projectRedirect: 'ModelStoreList', // 项目切换时重定向
        },
        component: () => import('./index.vue'),
        redirect: to => ({ name: 'ModelStoreList' }),
        children: [
            {
                path: 'list',
                name: 'ModelStoreList',
                component: () => import('./pages/list/index.vue'),
            },
            {
                path: ':id/detail/:detailTab?',
                name: 'ModelStoreDetail',
                component: () => import('./pages/detail/index.vue'),
            },
        ],
    },
];
```

#### 模型推理 (model-inference)

```typescript
// src/modules/model-inference/routes.ts
const routes: RouteRecordRaw[] = [
    {
        path: '/inference',
        name: 'Inference',
        meta: {
            title: '模型推理',
            projectRedirect: 'ModelInferenceList',
        },
        redirect: { name: 'ModelInferenceList' },
        children: [
            { path: 'list', name: 'ModelInferenceList', ... },
            { path: 'create', name: 'ModelInferenceCreate', ... },
            { path: ':eid/detail/:tab?', name: 'ModelInferenceDetail', ... },
        ],
    },
    {
        path: '/batch-inference',
        name: 'BatchInference',
        meta: { projectRedirect: 'ModelBatchInferenceList' },
        children: [ ... ],
    },
];
```

#### 数据集 (datasets)

```typescript
// src/modules/datasets/routes.ts
const routes: RouteRecordRaw[] = [
    {
        path: '/model-customization/datasets',
        name: 'datasets',
        meta: {
            title: '数据集',
            requiresAuth: true,
            projectRedirect: 'datasets-list',
        },
        children: [
            { path: 'create/', name: 'datasets-create', ... },
            { path: 'list/', name: 'datasets-list', ... },
            { path: 'detail/:id', name: 'datasets-detail', ... },
        ],
    },
];
```

### 6.6 完整路由表

| 模块 | 路由名称 | 路径 | 说明 |
|------|----------|------|------|
| 模型广场 | `ModelList` | `/ai-square/list` | 模型列表 |
| 模型仓库 | `ModelStoreList` | `/model-customization/model-store/list` | 模型仓库列表 |
| 模型仓库 | `ModelStoreDetail` | `/model-customization/model-store/:id/detail/:detailTab?` | 模型详情 |
| 模型推理 | `ModelInferenceList` | `/inference/list` | 推理服务列表 |
| 模型推理 | `ModelInferenceCreate` | `/inference/create` | 创建推理服务 |
| 模型推理 | `ModelInferenceDetail` | `/inference/:eid/detail/:tab?` | 推理服务详情 |
| 批量推理 | `ModelBatchInferenceList` | `/batch-inference/list` | 批量推理列表 |
| 数据集 | `datasets-list` | `/model-customization/datasets/list/` | 数据集列表 |
| 数据集 | `datasets-create` | `/model-customization/datasets/create/` | 创建数据集 |
| 数据集 | `datasets-detail` | `/model-customization/datasets/detail/:id` | 数据集详情 |
| 模型微调 | `ModelFinetuneJobList` | `/model-finetune/job/list` | 微调任务列表 |
| 模型部署 | `ModelDeployList` | `/model-deploy/list` | 部署列表 |
| 模型评测 | `ModelEvaluatingList` | `/model-evaluating/list` | 评测列表 |
| 调用统计 | `InvocationStatistic` | `/invocation-statistic` | 调用统计 |

### 6.7 项目切换时的路由重定向

当主应用切换项目时，子应用通过 `meta.projectRedirect` 配置进行路由重定向：

```typescript
// src/index.ts:153-165
const debouncedRouteReplace = debounce(() => {
    const redirectRouteName = router.currentRoute.value.meta.projectRedirect;
    const routeParams = {
        ...router.currentRoute.value.params,
        pid: projectStore.projectId,
    };

    const to = redirectRouteName
        ? ({ name: redirectRouteName as string, params: routeParams } as RouteLocationRaw)
        : ({ params: routeParams } as RouteLocationRaw);

    router.replace(to);
}, 200);
```

---

## 7. 状态管理

使用 Pinia 进行状态管理。

### 7.1 项目状态 Store

```typescript
// src/stores/projectStore.ts
export interface ProjectInfo {
    projectId: string;
    projectName: string;
    externalProjectId: string;
    description: string;
    createTime: number;
    type: string;
    status: string;
}

export const useProjectStore = defineStore('project', {
    state: () => ({
        projectId: '',              // 外部项目 ID（万倾项目）
        projectName: '',            // 项目名称
        steamLakeProjectId: '',     // StreamLake 项目 ID
        allProject: [] as ProjectInfo[],
    }),
    getters: {
        getProjectId(): string { return this.projectId; },
        getProjectName(): string { return this.projectName; },
        getSteamLakeProjectId(): string { return this.steamLakeProjectId; },
        getAllProject(): ProjectInfo[] { return this.allProject; },
        // 根据 StreamLake 项目 ID 获取万倾项目信息
        getWQProjectBySLId(): (projectId: string) => ProjectInfo {
            return (projectId: string) => {
                return this.allProject.find(item => item.projectId === projectId) || {} as ProjectInfo;
            };
        },
    },
    actions: {
        setProject(projectId: string) { this.projectId = projectId; },
        setProjectName(projectName: string) { this.projectName = projectName; },
        setSteamLakeProjectId(steamLakeProjectId: string) { this.steamLakeProjectId = steamLakeProjectId; },
        setAllProject(allProject: ProjectInfo[]) { this.allProject = allProject; },
    },
});
```

### 7.2 在组件中使用

```vue
<script setup lang="ts">
import { useProjectStore } from '@/stores/projectStore';

const projectStore = useProjectStore();

// 获取当前项目 ID
const currentProjectId = computed(() => projectStore.projectId);

// 获取当前项目名称
const currentProjectName = computed(() => projectStore.projectName);
</script>
```

---

## 8. 构建配置

### 8.1 UMD 输出配置

```typescript
// rsbuild.config.ts:95-103
tools: {
    rspack: {
        output: {
            publicPath: process.env.NODE_ENV === 'development'
                ? '/'
                : '//p2-sl.streamlake.com/kos/nlav11935/wanqing/fe/',
            library: {
                name: 'streamlake-console-wanqing',  // UMD 库名称
                type: 'umd',                          // 输出为 UMD 格式
            },
        },
    },
},
```

### 8.2 HTML 挂载点

```typescript
// rsbuild.config.ts:59-67
html: {
    mountId: 'wanqing-root',  // Vue 应用挂载元素 ID
    tags: [
        {
            tag: 'script',
            children: 'window.cdn_public_path = "https://p2-sl.streamlake.com";',
        },
    ],
},
```

### 8.3 开发服务器 CORS 配置

```typescript
// rsbuild.config.ts:69-81
server: {
    headers: {
        'Access-Control-Allow-Origin': '*',  // 允许跨域，主应用可以加载
    },
    proxy: {
        '/api': {
            target: process.env.PUBLIC_PROXY_URL,
            headers: {
                Cookie: process.env.PUBLIC_PROXY_COOKIE || '',
            },
        },
    },
},
```

---

## 9. 本地开发

### 9.1 独立运行模式

当不被 qiankun 加载时，项目可以独立运行：

```typescript
// src/index.ts:108-116
if (!window.__POWERED_BY_QIANKUN__) {
    render();
    if (import.meta.env.DEV) {
        const projectStore = useProjectStore();
        // 从环境变量读取项目 ID
        projectStore.projectId = import.meta.env.PUBLIC_PROJECT_ID;
        projectStore.projectName = import.meta.env.PUBLIC_PROJECT_NAME;
        projectStore.steamLakeProjectId = import.meta.env?.ST_PUBLIC_PROJECT_ID;
    }
}
```

### 9.2 环境变量配置

创建 `.env.local` 文件：

```env
PUBLIC_PROJECT_ID=your-project-id
PUBLIC_PROJECT_NAME=测试项目
ST_PUBLIC_PROJECT_ID=streamlake-project-id
PUBLIC_PROXY_URL=https://api.example.com
PUBLIC_PROXY_COOKIE=your-cookie
```

### 9.3 启动开发服务器

```bash
npm run dev
```

### 9.4 直接访问路由

独立运行时，可以直接在浏览器访问：

```
http://localhost:3000/inference/list          # 模型推理列表
http://localhost:3000/model-customization/model-store/list   # 模型仓库列表
http://localhost:3000/model-customization/datasets/list/     # 数据集列表
```

### 9.5 编程式导航

```typescript
import { useRouter } from 'vue-router';

const router = useRouter();

// 跳转到模型推理列表
router.push({ name: 'ModelInferenceList' });

// 跳转到模型详情
router.push({
    name: 'ModelStoreDetail',
    params: { id: 'model-123', detailTab: 'overview' }
});

// 带项目 ID 跳转
router.push({
    name: 'datasets-list',
    params: { pid: 'project-456' }
});
```

---

## 10. 常见问题

### Q1: 为什么没有左侧菜单栏？

**A**: 因为这是一个 qiankun 子应用，菜单栏由主应用提供。子应用只负责渲染内容区域。

### Q2: 如何切换菜单/路由？

**A**:
- **在主应用中**: 点击主应用的菜单，会自动加载对应的子应用和路由
- **在子应用中**: 使用 `router.push()` 进行编程式导航
- **本地开发**: 直接修改浏览器 URL

### Q3: 如何与主应用通信？

**A**: 使用 `setGlobalStateSafe()` 发送状态，使用 `onGlobalStateChange()` 监听状态变化。

### Q4: 项目切换时会发生什么？

**A**:

1. 主应用修改全局状态中的 `externalProjectId`
2. 子应用监听到变化，更新本地 `projectStore`
3. 子应用根据 `meta.projectRedirect` 配置重定向到对应列表页

### Q5: 如何判断是否在 qiankun 环境中运行？

**A**: 检查 `window.__POWERED_BY_QIANKUN__` 变量：

```typescript
if (window.__POWERED_BY_QIANKUN__) {
    console.log('在 qiankun 环境中运行');
} else {
    console.log('独立运行');
}
```

### Q6: 为什么路由要加 `:pid?` 参数？

**A**: 为了支持多项目切换。项目 ID 会体现在 URL 中，方便：
- 分享链接时保留项目上下文
- 刷新页面时恢复项目状态
- 主应用和子应用同步项目信息



