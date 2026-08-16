# Axios 封装

> 基于 MES 前端项目 `STwebsiteFrontEnd_V1` 的实际代码

---

## 一、全局注入

```js
// main.js
import request from '@/utils/request'
Vue.prototype.$http = request          // axios包装方法挂载到Vue原型上，所有组件都可以通过 this.$http 发请求
```

## 二、API 文件统一写法

每个后端 Controller 对应一个 API 文件，所有接口导出为具名函数：

```js
// api/workorder.js
import Vue from 'vue'
export function getList(params) {
    return Vue.prototype.$http({
        url: '/WorkOrder/GetList',
        method: 'post',
        data: params
    })
}
```

## 三、request.js 核心结构

```js
// utils/request.js
const service = axios.create({
    baseURL: '/api',           // 统一前缀
    timeout: 5000,             // 超时 5 秒
    withCredentials: true,     // 跨域带 cookie
})
```

### ① 请求拦截器 — 自动带 token

```js
service.interceptors.request.use(config => {
    const token = getToken()
    if (token) {
        config.headers.common['Authorization'] = 'Bearer ' + token
    }
    return config  // 放行
})
```

每个请求发出前自动走这里，不需要每个 API 手动加 token。

### ② 响应拦截器 — 统一处理

```js
service.interceptors.response.use(
    response => {
        const res = response.data
        if (res.Code !== 20000) {
            Message.error(res.Message)           // 统一弹错误消息
            if (res.Code === 50014) { /* token过期 → 跳登录 */ }
            return Promise.reject(new Error(res.Message))
        }
        return res                               // 成功 → 正常返回
    },
    error => {
        if (error.response?.status === 401) { /* 未登录 */ }
        if (error.response?.status === 403) { /* 无权限 */ }
        return Promise.reject(error)
    }
)
```

---

## 四、完整请求流程

```
组件调用 getList(params)
    │
    ├── 请求拦截器 → 自动加 Authorization header
    │
    ├── axios 发 HTTP 请求到 /api/WorkOrder/GetList
    │
    ├── 服务器返回
    │
    └── 响应拦截器 → 判断 Code
            ├── 20000  → 返回 data
            └── ≠20000 → 弹错误消息 + reject
```

## 五、后端统一响应格式

```js
{
    Code: 20000,                    // 20000 = 成功
    Success: true,
    Message: "操作成功",
    Data: {
        Items: [...],               // 列表数据
        TotalCount: 200,            // 总条数
        ExtendData: {                // 扩展数据
            CurrentCountSum: 5000
        }
    }
}
```

## 六、页面中调用

```js
// TEST_LIV/index.vue
getList(this.queryParams).then(response => {
    this.listData = response.Data.Items          // 列表
    this.totalCount = response.Data.TotalCount   // 总数
    this.listLoading = false
})
```

---

## 相关笔记

- [[02-Vue实例与渲染流程]] — Vue.prototype.$http 的挂载机制
- [[09-Vuex]] — actions 中调 API 的模式
- [[06-响应式原理]] — API 返回后赋值给 data，触发响应式更新
