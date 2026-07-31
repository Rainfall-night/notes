# Vue 项目初始化与入口

> 基于 MES 前端项目 `STwebsiteFrontEnd_V1` 的实际代码

---

## 一、项目技术栈

| 技术 | 版本 | 用途 |
|------|------|------|
| Vue | 2.6.10 | 核心框架 |
| Vue Router | 3.0.6 | 路由管理 |
| Vuex | 3.1.0 | 状态管理 |
| Element UI | 2.13.2 | UI 组件库 |
| Axios | 0.18.1 | HTTP 请求 |
| vue-cli | 4.4.4 | 构建工具 |

## 二、目录结构

```
STwebsiteFrontEnd_V1/
├── public/index.html        # HTML 入口，含 <div id="app">
├── src/
│   ├── main.js              # JS 入口，new Vue() 启动
│   ├── App.vue              # 根组件，纯 <router-view />
│   ├── permission.js        # 路由守卫 + v-permission 指令
│   ├── settings.js          # 站点配置
│   ├── router/index.js      # 全部路由定义 + resetRouter()
│   ├── store/               # Vuex 状态管理
│   ├── api/                 # 后端接口封装（每个 Controller 一个文件）
│   ├── views/               # 页面视图（约 15 个业务模块）
│   ├── components/          # 公共组件（9 个）
│   ├── layout/              # 布局组件（Sidebar/Navbar/TagsView/AppMain）
│   ├── utils/               # 工具函数（12 个）
│   └── styles/              # 全局 SCSS 样式
└── vue.config.js            # 构建配置（devServer、代理等）
```

## 三、main.js 加载顺序

```js
// 1. 基础库
import Vue from 'vue'
import ElementUI from 'element-ui'

// 2. 全局样式
import 'normalize.css/normalize.css'
import 'element-ui/lib/theme-chalk/index.css'
import '@/styles/index.scss'

// 3. 路由和状态管理
import router from './router'
import store from './store'

// 4. 工具注入
import request from '@/utils/request'
Vue.prototype.$http = request        // this.$http → axios 实例
Vue.prototype.$message = Message     // this.$message → 消息提示

// 5. 自定义指令
Vue.directive('focus', { inserted(el) { el.focus() } })

// 6. 启动！
new Vue({
  el: '#app',
  router,
  store,
  render: h => h(App)
})
```

## 四、关键理解

- **Vue 是渐进式框架**：可以只用渲染，也可以搭配 Router + Vuex + Element UI 做完整 SPA
- **Runtime-only 构建**：`.vue` 的 `<template>` 在 `npm run dev/build` 时由 `vue-loader` 编译成 `render` 函数
- **Vue = 数据驱动的 HTML 装配器**：编译时翻译成 render 指令，运行时根据数据动态产出 HTML

---

## 相关笔记

- [[02-Vue实例与渲染流程]] — new Vue()、虚拟DOM、模板编译
- [[03-单文件组件]] — .vue 文件结构
- [[10-自定义指令]] — v-permission 原理
- [[04-模板语法]] — v-if、v-for、v-model 等
