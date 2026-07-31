 # Vue Router

> 基于 MES 前端项目 `STwebsiteFrontEnd_V1` 的实际代码

---

## 一、路由配置结构

```js
// router/index.js
export const constantRoutes = [
  {
    path: '/procedure',              // URL 路径
    component: Layout,               // 匹配到的组件（侧边栏框架）
    name: '工作流程',
    meta: { title: '工作流程', icon: 'procedure' },
    children: [                      // 嵌套子路由
      {
        path: 'TEST_LIV',
        name: 'LIV测试',
        component: () => import('@/views/procedure/TEST_LIV/index'),  // 懒加载
        meta: {
          title: 'LIV测试',
          unreadkey: 'Procedure_TEST_LIV',   // 侧边栏角标匹配key
          roles: ['Procedure_TEST_LIV']       // 权限控制
        }
      }
    ]
  }
]
```

| 字段 | 作用 |
|------|------|
| `path` | URL 匹配规则 |
| `component` | 匹配后渲染哪个组件 |
| `children` | 嵌套路由，子路由渲染到父组件的 `<router-view>` |
| `meta` | 附加信息（标题、图标、权限、角标key） |
| `name` | 路由命名，用于 `this.$router.push({ name: 'LIV测试' })` |

## 二、嵌套路由与两个 `<router-view>`

```
App.vue
└── <router-view />          ← 渲染 Layout
        │
        └── Layout（侧边栏 + 顶栏框架）
                │
                └── AppMain.vue
                        └── <router-view />  ← 渲染具体页面
```

一级路由匹配 Layout，Layout 的 children 匹配具体页面。详见 [[05-组件生命周期#keep-alive|keep-alive 缓存机制]]。

## 三、导航守卫 — 路由跳转的关卡

```js
// permission.js
router.beforeEach(async (to, from, next) => {
    const hasToken = getToken()

    if (hasToken) {
        if (hasGetUserInfo) {
            next()                   // 有token有用户信息 → 放行
        } else {
            await store.dispatch('user/getInfo')
            store.dispatch('user/getPermission').then(() => {
                resetRouter()        // 过滤无权限菜单
                next()
            })
        }
    } else {
        next('/login')               // 没token → 跳登录
    }
})
```

| 钩子 | 触发时机 | 能阻止跳转？ |
|------|----------|:--:|
| `beforeEach` | 跳转**前** | ✅ `next(false)` / `next('/login')` |
| `afterEach` | 跳转**后** | ❌ 只能收尾（关进度条） |

### 固定参数

| 参数 | 含义 |
|------|------|
| `to` | 目标路由对象（要去哪） |
| `from` | 来源路由对象（从哪来） |
| `next` | 必须调用才能放行/跳转 |

> 每次路由跳转都经过 `beforeEach`，但 `getPermission` + `resetRouter` 只在**首次**执行一次。

## 四、resetRouter — 权限过滤菜单

```js
// router/index.js
export function resetRouter() {
    // 遍历 constantRoutes
    // 删除用户没有 meta.roles 的菜单项
    // 创建新 Router 实例替换原 router
}
```

用户登录后按角色过滤菜单，侧边栏只显示有权限的页面。详见 [[10-自定义指令|v-permission]]。

## 五、编程式导航

```js
this.$router.push('/procedure/TEST_LIV')    // 跳转
this.$router.push({ name: 'LIV测试' })       // 命名跳转
this.$route.path          // '/procedure/TEST_LIV'  当前路径
this.$route.meta.title    // 'LIV测试'               当前标题
```

| | `$router` | `$route` |
|------|------|------|
| 是什么 | 路由器实例（全局唯一） | 当前路由对象 |
| 做什么 | `push()`、`replace()` | 读 `path`、`query`、`meta` |

## 六、路由懒加载

```js
// 不懒加载：所有页面打包在一个文件
component: require('@/views/procedure/TEST_LIV/index')

// 懒加载：每个页面单独文件，访问时才下载
component: () => import('@/views/procedure/TEST_LIV/index')
```

本项目 80+ 页面全部懒加载，首页无需下载所有代码。

## 七、本项目路由全景

| 一级路由 | 子页面数 | 说明 |
|------|:--:|------|
| `/dashboard` | 1 | 主页 |
| `/workorder` | 12 | 下单、在制、历史、拆分... |
| `/procedure` | 80+ | 所有工序页面 |
| `/report` | 6 | 产出、良率、直通率... |
| `/dictionary` | 17 | 封装形式、Recipe、Spec... |
| `/user` | 1 | 人员管理 |
| `/system` | 2 | 系统设置、更新日志 |

## 八、侧边栏跳转实现

侧边栏点击菜单跳转页面，不是手写 `@click` 事件，而是通过 Vue Router 的 `<router-link>` 声明式导航。

### 完整链路

```
Sidebar/index.vue
    │  routes() { return this.$router.options.routes }
    │  <sidebar-item v-for="route in routes" :item="route" />
    │
    └── SidebarItem.vue
            │  <app-link :to="resolvePath(onlyOneChild.path)">
            │
            └── Link.vue  ← 关键组件
                    │
                    └── <component :is="type" v-bind="linkProps(to)">
```

### Link.vue 核心逻辑

```vue
<template>
  <component :is="type" v-bind="linkProps(to)">
    <slot />
  </component>
</template>

<script>
computed: {
    type() {
        return isExternal(this.to) ? 'a' : 'router-link'
        //    外部链接 → <a> 标签         内部路由 → <router-link>
    }
},
methods: {
    linkProps(to) {
        if (this.isExternal) return { href: to, target: '_blank' }
        return { to: to }   // ← 传给 <router-link to="/procedure/TEST_LIV">
    }
}
</script>
```

### 跳转流程

```
用户点击 "LIV测试"
    ↓
<router-link to="/procedure/TEST_LIV">
    ↓
Vue Router 接管 → permission.js beforeEach
    ↓
URL 变化 → AppMain 的 <router-view> 渲染 TEST_LIV/index.vue
```

### Sidebar 路由来源

```js
// Sidebar/index.vue
routes() {
    return this.$router.options.routes   // 就是 constantRoutes（已经被 resetRouter 过滤过）
}
```

> 侧边栏显示的是当前 Router 实例配置中的所有路由。

---

## 相关笔记

- [[05-组件生命周期]] — keep-alive 与路由缓存
- [[10-自定义指令]] — v-permission 按钮级权限
- [[07-组件通信]] — Vuex 全局状态（store.getters.roles）
- [[01-Vue项目初始化与入口]] — main.js 中注入 router
- [[01-Vue项目初始化与入口]] — main.js 中注入 router
