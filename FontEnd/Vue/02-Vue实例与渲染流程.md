# Vue 实例与渲染流程

> 基于 MES 前端项目 `STwebsiteFrontEnd_V1` 的实际代码

---

## 一、new Vue() 根实例

```js
new Vue({
  el: '#app',               // 挂载点
  router,                   // 注入路由 → this.$router / this.$route
  store,                    // 注入状态管理 → this.$store
  render: h => h(App)       // 渲染根组件（h = createElement）
})
```

整个项目只有**一个** `new Vue()`，它是所有组件的根。

## 二、Vue.prototype 原型扩展

```js
Vue.prototype.$http = request       // this.$http → axios 实例
Vue.prototype.$message = Message    // this.$message → Element UI 消息提示
```

挂在原型上的属性/方法，所有组件都能通过 `this.$xxx` 直接访问，无需 import。

## 三、Vue.use() 插件机制

```js
Vue.use(ElementUI, { locale })
```

调用插件的 `install` 方法，ElementUI 做了两件事：
1. 全局注册所有 `el-xxx` 组件
2. 在 `Vue.prototype` 上挂载 `$message`、`$notify` 等

## 四、Vue.directive() 全局指令

```js
Vue.directive('focus', {
  inserted(el) { el.focus() }
})
```

详见 [[10-自定义指令]]。

## 五、Vue.config 全局配置

```js
Vue.config.productionTip = false   // 关闭开发版本提示
```

## 六、虚拟 DOM 与 diff 算法

### 什么是虚拟 DOM

用 JS 对象模拟真实 DOM，避免频繁操作真实 DOM 导致的性能问题。

```js
// 虚拟 DOM（VNode）
{ tag: 'div', data: { class: 'box' }, children: [{ text: 'Hello' }] }
```

### 为什么需要虚拟 DOM

```
直接操作 DOM：3 次修改 → 3 次重排重绘
虚拟 DOM：   3 次修改 → 1 次 diff → 1 次批量更新
```

### diff 算法核心

- **同层比较**，复杂度 O(n)
- 类型相同 → 深入对比属性和子节点
- 类型不同 → 删旧建新
- `v-for` 必须有 `:key`，否则 diff 效率极低

```
没 key：逐个对比，可能全部重建
有 key：按 key 匹配，只移动/新增真正变化的节点
```

## 七、模板编译（runtime-only）

```
<template>  →  vue-template-compiler  →  render(h) { return h('div', ...) }
   (开发时)       (构建时编译)              (打包产物中)
```

Vue CLI 项目默认 **runtime-only**（不含编译器），`.vue` 的 template 在构建时编译，体积更小、性能更好。

## 八、总结

> **Vue = 用数据驱动的 HTML 装配器**
>
> 编译时：`<template>` → `render` 装配指令
> 运行时：数据变了 → render 重新执行 → 新 VNode → diff → 最小化 DOM 更新

---

## 相关笔记

- [[01-Vue项目初始化与入口]] — 项目结构和启动流程
- [[03-单文件组件]] — .vue 文件结构
- [[10-自定义指令]] — v-permission 原理
- [[04-模板语法]] — 数据如何渲染到 DOM
