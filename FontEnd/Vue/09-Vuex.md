# Vuex

> 基于 MES 前端项目 `STwebsiteFrontEnd_V1` 的实际代码

---

## 一、核心概念

Vuex 是 Vue 的全局状态管理，所有组件共享一份响应式数据。一句话定版：

> **Vuex = 响应式的全局数据中心，state 是数据，mutation 是唯一的同步写入口（带操作日志），action 在 mutation 外面包了一层异步操作。**

---

## 二、五个核心角色

```
组件 dispatch('action')
    ↓
action 做异步操作（调API），拿到结果后 commit('mutation')
    ↓
mutation 同步修改 state（唯一能改 state 的地方）
    ↓
state 是响应式的 → 所有依赖的组件自动重渲染
```

| 概念 | 是什么 | 能异步吗 | 本项目例子 |
|------|------|:--:|------|
| **state** | 数据 | - | `state.sidebar.opened` |
| **getters** | state 的派生值 | ❌ | `store.getters.name` |
| **mutations** | **唯一**改 state 的方法（带操作日志） | ❌ 必须同步 | `SET_TOKEN(state, token)` |
| **actions** | 调 API + 处理逻辑 + commit mutation | ✅ | `dispatch('user/login')` |
| **modules** | 拆分成多个文件的组织方式 | - | `app`、`user`、`settings`、`tagsView` |

---

## 三、为什么 mutation 不能异步

```js
// ❌ 错误示范
mutations: {
    SET_TOKEN(state) {
        setTimeout(() => {
            state.token = 'xxx'    // DevTools 录不到这个变化！
        }, 1000)
    }
}
```

Vue DevTools 的时间旅行调试会失效——回放 mutation 时，`setTimeout` 里的代码已经不在回放范围内。

### 正确做法

```js
actions: {
    login({ commit }, { username, password }) {
        // ① action 中做异步
        return login({ username, password }).then(res => {
            // ② 异步完成后，同步 commit
            commit('SET_TOKEN', res.data.token)
        })
    }
}
```

| | 做什么 | 异步？ |
|------|------|:--:|
| **action** | 调 API、处理数据、判断逻辑 | ✅ |
| **mutation** | 把最终结果写入 state | ❌ 必须同步 |

> action 负责"怎么拿到数据"，mutation 负责"把数据写进去"。

---

## 四、`.then()` 就是异步

```js
// 这两段等价：

// Promise（本项目写法）
login({ username, password }).then(res => {
    commit('SET_TOKEN', res.data.token)
})

// async/await（等价写法）
const res = await login({ username, password })
commit('SET_TOKEN', res.data.token)
```

`login()` 返回的是 Promise（axios 请求），`.then()` 里的回调不会立即执行，等服务器返回后才触发。

---

## 五、代码风格不唯一

Vuex 只规定了顶层属性名 `state`/`mutations`/`actions`，但内部组织方式自由：

```js
// app.js — 分开声明
const state = { sidebar: {...}, device: 'desktop' }
const mutations = { TOGGLE_SIDEBAR: state => {...} }
export default { namespaced: true, state, mutations, actions }

// user.js — 内联声明
const mutations = {
    RESET_STATE: (state) => {...},
    SET_TOKEN: (state, token) => {...},
}
```

`namespaced: true`：模块隔离，访问时需带模块前缀 `$store.state.app.sidebar`。

---

## 六、本项目模块结构

```js
// store/index.js
new Vuex.Store({
  modules: {
    app,        // 侧边栏状态
    settings,   // 系统配置
    user,       // 用户信息 + 角色
    tagsView    // 顶部标签页
  }
})
```

### user 模块完整示例

```js
const state = {
    token: '',                  // 登录令牌
    name: '',                   // 用户名
    roles: [],                  // 角色列表
}

const mutations = {
    SET_TOKEN(state, token) { state.token = token },
    ADD_ROLES(state, roles) { roles.forEach(r => state.roles.push(r)) }
}

const actions = {
    login({ commit }, { username, password }) {
        return login({ username, password }).then(res => {
            commit('SET_TOKEN', res.data.token)
        })
    },
    getPermission({ commit }) {
        return getPermission().then(res => {
            commit('ADD_ROLES', res.Data)
        })
    }
}
```

---

## 七、组件中怎么用

```js
// 读 state
this.$store.state.user.name          // '张三'
this.$store.state.app.sidebar.opened  // true

// 读 getters（更推荐）
store.getters.name                    // '张三'

// 调 action
this.$store.dispatch('user/login', { username, password })

// 辅助函数
import { mapGetters } from 'vuex'
computed: { ...mapGetters(['sidebar']) }
```

---

## 八、数据流全景（以权限为例）

```
permission.js
    ↓ dispatch('user/getPermission')
user/actions: getPermission()
    ↓ API: GET /Authorization/GetPermission
    ↓ commit('ADD_ROLES', roles)
state.roles = ['Procedure_TEST_LIV', ...]
    ↓ 响应式通知
所有用到 roles 的组件自动重渲染
    ├── resetRouter() 过滤菜单
    ├── v-permission 隐藏按钮
    └── Sidebar 显示有权限的菜单
```

---

## 相关笔记

- [[06-响应式原理]] — state 也是响应式的，原理和 data 一样
- [[07-组件通信]] — Vuex 是全局通信方案
- [[08-Vue Router]] — resetRouter 依赖 store.getters.roles
- [[10-自定义指令]] — v-permission 读取 store.getters.roles
