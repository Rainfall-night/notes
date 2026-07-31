# keep-alive 与组件缓存

> 基于 MES 前端项目 `STwebsiteFrontEnd_V1` 的实际代码

---

## 一、是什么

`<keep-alive>` 是 Vue 的内置组件，包裹 `<router-view>` 后，切换页面时被切走的组件**不会被销毁**，而是缓存起来，切回来直接复用。

```vue
<!-- layout/components/AppMain.vue -->
<keep-alive :max="10" :exclude="cachedViews">
  <router-view />
</keep-alive>
```

## 二、三个属性

| 属性 | 作用 | 切走时 | 切回时 | 本项目值 |
|------|------|:--:|:--:|------|
| `include` | **只缓存**匹配的组件（白名单） | 保留（休眠） | `activated` 唤醒 | 未设置 |
| `exclude` | **不缓存**匹配的组件（黑名单） | **销毁** | 重建走 `mounted` | `cachedViews` |
| `max` | 最多缓存几个（超出用 LRU 淘汰最早那个） | - | - | `10` |

> `include` 里的组件 = 切走休眠，`exclude` 里的组件 = 切走销毁。

## 三、`exclude` 的数据来源

```js
// AppMain.vue
computed: {
    cachedViews() {
        return this.$store.state.tagsView.cachedViews
    }
}
```

```js
// store/modules/tagsView.js
const state = {
    visitedViews: [],    // 用户打开的标签页
    cachedViews: []      // 传给 keep-alive exclude 的页面名列表
}

// 路由 meta.noCache: true 的页面会被加入 cachedViews → 不被缓存
ADD_CACHED_VIEW: (state, view) => {
    if (state.cachedViews.includes(view.name)) return
    if (!view.meta.noCache) {
        state.cachedViews.push(view.name)
    }
}
```

## 四、完整生命周期对比

```
无 keep-alive：                     有 keep-alive：
    created                              created
    mounted                              mounted
    ...使用中...                         ...使用中...
    切走 → beforeDestroy                 切走 → deactivated（休眠）
    切回 → created + mounted（重建）      切回 → activated（唤醒）
    再切走 → beforeDestroy               再切走 → deactivated
    关闭标签 → destroyed                 关闭标签 → beforeDestroy → destroyed
```

## 五、`activated` 钩子 — 唤醒时触发

**每次从缓存中切回来都执行**，适合刷新数据、恢复状态。

### 本项目实际

```js
// processingindex.vue — 每次切回来初始化列筛选和下拉选项
activated() {
    this.initCheckList()
    this.getOptions()
    // this.fetchData()  // 故意注释：不重新查数据，保留旧数据
}
```

### 典型用法

```js
activated() {
    // ① 如果数据可能过期了，重新拉取
    if (Date.now() - this.lastFetchTime > 60000) {   // 超过 1 分钟
        this.fetchData()
    }

    // ② 恢复之前保存的滚动位置
    this.$nextTick(() => {
        this.$refs.tb.$el.scrollTop = this.savedScrollTop
    })

    // ③ 重新开启被 deactivated 暂停的定时器
    this.startPolling()
}
```

### 与 `mounted` 的区别

| | `mounted` | `activated` |
|------|------|------|
| 触发次数 | 组件一生**只一次** | 每次从缓存切回来**都触发** |
| 适用 | 初始化（首次进页面） | 刷新数据（再次切回来） |
| 没有 keep-alive 时 | 正常 | **永远不会触发** |

> `mounted` + `activated` 常同时使用：`mounted` 做一次性初始化，`activated` 做每次刷新。

## 六、`deactivated` 钩子 — 休眠时触发

**每次被缓存切走时执行**，适合保存状态、暂停任务。

```js
deactivated() {
    // ① 暂停轮询，省资源
    clearInterval(this.pollingTimer)
    this.pollingTimer = null

    // ② 保存当前滚动位置
    this.savedScrollTop = this.$refs.tb.$el.scrollTop

    // ③ 取消未完成的请求，避免切回来后数据错乱
    if (this.cancelToken) {
        this.cancelToken.cancel('页面切换，取消请求')
    }
}
```

### `activated` + `deactivated` 配对使用

```js
data() {
    return {
        pollingTimer: null,
        savedScrollTop: 0,
    }
},
activated() {
    this.startPolling()                           // 恢复轮询
    this.$refs.tb.$el.scrollTop = this.savedScrollTop  // 恢复滚动位置
},
deactivated() {
    clearInterval(this.pollingTimer)              // 暂停轮询
    this.savedScrollTop = this.$refs.tb.$el.scrollTop  // 保存滚动位置
}
```

## 七、`max` 超限 — LRU 策略

```vue
<keep-alive :max="10">
```

缓存超过 10 个时，**最早缓存的那个被销毁**（走 `destroyed`）。本项目最多同时保持 10 个标签页活跃。

## 八、tagsView 全量管理

```js
// store/modules/tagsView.js
const state = {
    visitedViews: [],     // 所有打开的标签页
    cachedViews: []       // 不缓存的页面（传给 keep-alive exclude）
}

const mutations = {
    ADD_VISITED_VIEW(state, view) { /* 打开新标签 */ },
    ADD_CACHED_VIEW(state, view)  { /* 加入不缓存名单 */ },
    DEL_VISITED_VIEW(state, view) { /* 关闭一个标签 */ },
    DEL_OTHERS_VISITED_VIEWS(state, view) { /* 关闭其他标签 */ },
    DEL_ALL_VISITED_VIEWS(state)   { /* 关闭全部 */ },
    // cachedViews 对应的 DEL 操作同理
}
```

标签页的增删通过 Vuex actions 统一管理，侧边栏右键"关闭/关闭其他/全部关闭"都走这套逻辑。

---

## 相关笔记

- [[05-组件生命周期]] — activated/deactivated 的详细触发时序
- [[09-Vuex]] — tagsView 模块的状态管理
- [[08-Vue Router]] — keep-alive 包裹 router-view 的嵌套结构
