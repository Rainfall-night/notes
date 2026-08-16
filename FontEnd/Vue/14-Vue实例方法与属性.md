# Vue 实例方法与属性

> Vue 2 原生提供的 `$` 开头实例方法/属性

---

## DOM 相关

### `$el`

组件根 DOM 元素。即 `<template>` 下方**唯一一个最外层标签**。`mounted` 之后可用，在此之前为 `undefined`。

```vue
<template>
  <div class="app-container">          ← this.$el（根元素）
    <div class="filter-container">     ← 后代
      <el-input ref="searchInput" />   ← 后代的后代
    </div>
  </div>
</template>
```

```js
mounted() {
    console.log(this.$el)                          // <div class="app-container">
    this.$el.querySelector('.filter-container')     // ✅ 能找到（后代）
    this.$el.querySelector('el-input')              // ✅ 也能找到（深层后代）
}

created() {
    console.log(this.$el)    // undefined — 还没挂载！
}
```

**关键点：**

- `$el` 是组件模板**最外层那个标签**，不是组件内部的某个子元素。Vue 2 要求每个组件模板有且只有一个根元素。
- `querySelector` 在 `$el` 上搜索的是**整个子树**，所有后代都能搜到，不是仅直接子元素。
- **`mounted` 之前 `$el` 是 `undefined`**，不要尝试在 `created` 里访问它。

### `$refs`

拿模板中 `ref="xxx"` 标记的**子组件实例**或**原生 DOM 元素**。是一个对象，key 是 ref 名，value 是元素/组件实例。

```vue
<template>
  <el-input ref="searchInput" />              ← ref 打在 Element UI 组件上 → 拿到组件实例
  <input ref="nativeInput" type="text" />     ← ref 打在原生标签上 → 拿到 DOM 元素
  <child-component ref="child" />             ← ref 打在自定义组件上 → 拿到组件实例
</template>

<script>
mounted() {
    // Element UI 组件实例，可以调其 methods
    this.$refs.searchInput.focus()

    // 原生 DOM，可以调所有 DOM API
    this.$refs.nativeInput.value = 'hello'

    // 自定义子组件，可以调子组件的 methods
    this.$refs.child.fetchData()
}
</script>
```

**`$refs` 什么时候可用？**

`$refs` 里的值**不是响应式的**。组件渲染完成后才会填充，在此之前为 `undefined`。

```js
// ❌ 刚改了数据就让 ref 干活 — 没用，DOM 还没更新
this.dialogVisible = true
this.$refs.input.focus()                   // TypeError: $refs.input is undefined

// ✅ 等 DOM 更新完
this.dialogVisible = true
this.$nextTick(() => {
    this.$refs.input.focus()               // 此时 ref 已存在
})
```

**`ref` 配合 `v-for`：**

```vue
<template>
  <div v-for="item in list" :key="item.id" ref="itemRefs">
    {{ item.name }}
  </div>
</template>
```
```js
this.$refs.itemRefs   // [div, div, div...] — 数组，不是单个值！
```

> 一个 ref 名打了多个元素时，`$refs.xxx` 是数组，顺序和渲染顺序一致。

### `$nextTick(cb)`

等 DOM 更新完成后再执行回调。

**为什么需要它？** Vue **异步**更新 DOM。你改完 data，Vue 不会立即渲染——它会把多次数据变化攒起来，在一次"tick"中批量更新。所以你改完数据**同一行立即读 DOM**，读到的是旧 DOM。

```js
// data: { listData: [1, 2, 3] }

this.listData = [4, 5, 6]                // 改数据 → Vue 标记需要更新（异步的）
console.log(this.$el.textContent)         // ⚠️ 还是渲染的旧数据！渲染还没执行

this.$nextTick(() => {
    console.log(this.$el.textContent)     // ✅ 新数据已渲染
})
```

**原理简述：**

```
你改 data → Vue 把更新任务推入"微任务队列"（Promise.then）
你立即读 DOM → 微任务还没执行，DOM 是旧的
$nextTick → 你的回调也推入微任务队列，排在更新任务之后
微任务执行 → 先更新 DOM，再执行你的回调
```

**带返回值的用法（Vue 2.1+）：**

```js
// $nextTick 不传回调时返回 Promise
await this.$nextTick()
this.$refs.input.focus()
```

**常见场景：**

| 场景 | 为什么需要 `$nextTick` |
|------|------|
| 打开弹窗后聚焦输入框 | `v-if` 刚变 true，DOM 还没渲染 |
| 获取列表渲染后的高度 | 数据变更后 DOM 尺寸还没更新 |
| 操作第三方库初始化 | 需要 DOM 先存在才能挂载（如图表、地图） |

---

## 数据相关

### `$data`

组件当前的 data 对象。等同于 `data()` 返回的对象（经 Vue 响应式包裹后）。**极少直接使用**——通常 `this.xxx` 就够了。

```js
this.$data.listData           // 等价于 this.listData
console.log(this.$data)        // 整个 data 对象快照，调试时用
```

**调试技巧：** 在浏览器控制台中 `$vm0.$data` 可以看到组件的完整 data 对象（需要在 Vue Devtools 中选中组件）。

> 注意：`this.$data` 和 `data()` 原始函数返回的对象**不是同一个引用**——Vue 内部做了响应式包装。

### `$props`

组件当前接收到的 props 对象。和 `this.xxxProp` 等价。

```js
// 子组件 checkin.vue
props: ['operateId', 'params', 'formType']

mounted() {
    console.log(this.$props)
    // { operateId: '123', params: {...}, formType: 'checkin' }
    console.log(this.operateId)     // '123' — 和 this.$props.operateId 一样
}
```

**`$props` vs 直接 `this.xxx`：**

| 方式 | 区别 |
|------|------|
| `this.operateId` | 日常使用，简洁 |
| `this.$props.operateId` | 和上面完全一样 |
| `this.$props` | 一次性拿到**所有** props 的快照，调试/动态场景用 |

> 实际开发中几乎不会直接写 `this.$props.xxx`，`this.xxx` 更短更直观。`$props` 主要用于需要遍历所有 props 或调试时。

### `$watch(expr, cb, [options])`

**动态**创建监听器，返回一个**取消函数**。和 `watch` 选项功能相同，但可在运行时动态添加和移除——`watch` 选项是声明式的（写死的），`$watch` 是命令式的（代码控制的）。

```js
mounted() {
    // 动态开始监听
    const unwatch = this.$watch('queryParams.searchText', (newVal, oldVal) => {
        console.log('搜索词变了：', oldVal, '→', newVal)
    })

    // 5 秒后停止监听
    setTimeout(() => {
        unwatch()                      // ← 调用返回的函数即可取消
        console.log('已停止监听搜索词')
    }, 5000)
}
```

**第二个参数的两种写法：**

```js
// 写法一：字符串路径（只能监听 data 属性）
this.$watch('queryParams.searchText', callback)

// 写法二：函数（可以监听计算值）
this.$watch(
    () => this.listData.length + this.totalCount,
    (newVal) => { console.log('合计变了：', newVal) }
)
```

**第三个参数 `options`：**

```js
this.$watch('listData', callback, {
    deep: true,      // 深度监听，对象内部变了也触发
    immediate: true  // 立即执行一次 callback（不等第一次变化）
})
```

| 选项 | 默认 | 效果 |
|------|------|------|
| `deep` | false | `true` 时递归监听对象内部所有嵌套属性 |
| `immediate` | false | `true` 时立即以当前值执行一次 callback |

**和 `watch` 选项的对比：**

| | `watch` 选项 | `$watch` |
|------|------|------|
| 声明方式 | 组件选项里写死 | JS 代码里动态调用 |
| 什么时候用 | 组件一创建就要监听 | 运行时按需监听/取消 |
| 自动清理 | 组件销毁时自动解绑 | 需要手动调 `unwatch()`，否则可能内存泄漏 |

> `$watch` 返回值是**函数本身**（不是对象），类比 `setInterval` 返回 timer ID、`clearInterval(timerID)` 取消。如果组件销毁前没调 `unwatch()`，回调会继续执行（导致内存泄漏），所以一般要在 `beforeDestroy` 里清理。

### `$set(obj, key, val)`

给**已存在的**响应式对象**添加新属性**并触发视图更新。

**为什么需要它？** Vue 2 用 `Object.defineProperty` 实现响应式——它只能拦截**已存在**属性的读写。新增属性时 Vue 2 感知不到（ES5 的 `Object.defineProperty` 没法拦截新属性的添加），必须用 `$set` 手动告诉 Vue "这个对象多了个新属性"。

```js
// data: { listData: [{ name: 'A' }, { name: 'B' }] }

// ❌ 直接给对象加新属性 — Vue 2 拦截不到，视图不更新
this.listData[0].newField = 'value'

// ✅ $set — 视图会更新
this.$set(this.listData[0], 'newField', 'value')
```

**数组也可以用 `$set`：**

```js
// ❌ 通过索引直接赋值 — 视图不更新
this.listData[0] = newItem

// ✅ $set — 视图更新
this.$set(this.listData, 0, newItem)
```

> Vue 3 用 `Proxy` 重构后不需要 `$set` 了，`Proxy` 能拦截所有操作包括新增属性。所以如果你看到 `$set`，基本可以确认这是 Vue 2 项目。

**常见触发场景：**

| 场景 | 为什么触发不了更新 |
|------|------|
| `obj.newKey = val` | `defineProperty` 不知道有这个 key |
| `arr[0] = val` | 按索引赋值也不会触发 |
| `arr.length = 0` | 修改 length 也不会触发 |

### `$delete(obj, key)`

删除响应式对象的属性并触发视图更新。和 `$set` 同理——直接 `delete` 操作 Vue 2 感知不到。

```js
// ❌ 直接 delete — 视图不更新
delete this.userInfo.phone

// ✅ $delete — 视图更新
this.$delete(this.userInfo, 'phone')
```

### `$forceUpdate()`

**强制**组件重新渲染，跳过响应式系统，直接走 render → diff → patch 流程。

```js
this.$forceUpdate()   // 强制刷新当前组件
```

**什么时候用？** 几乎不应该用。只有当数据变了但 Vue 没检测到变化时（比如你用了 `Object.freeze` 或修改了非响应式数据），才需要它兜底。

> 属于"最后手段"。如果代码需要靠 `$forceUpdate` 才能刷新，说明数据流设计有问题——应该用 `$set` 或重新赋值触发响应式系统。

**正确做法 vs 错误做法：**

```js
// ❌ 绕过响应式系统，靠 forceUpdate 硬刷新
this.nonReactiveData = newData
this.$forceUpdate()

// ✅ 正确：通过响应式数据驱动
this.reactiveData = newData   // 自动触发更新，不需要 forceUpdate
```

---

## 组件树相关

### `$parent`

当前组件的**直接父组件实例**。可以在子组件中访问父组件的 data、methods 等一切东西。

```js
this.$parent.fetchData()         // 调父组件的方法
this.$parent.listData            // 读父组件的 data
this.$parent.$parent              // 可以一直往上找——不推荐
```

**为什么不推荐？**

```
子组件用 $parent → 子组件依赖了"父组件是谁" → 这个子离开这个父就不能用了
```

组件应该通过 **props 接收数据、`$emit` 发事件** 来和父通信，而不是偷偷去翻父的口袋。一旦你写了 `this.$parent.xxx`，这个组件就被焊死在这个父组件下面了。

```js
// ❌ 紧耦合 — 换个父组件就崩
this.$parent.fetchData()

// ✅ 松耦合 — 发事件让父自己决定怎么处理
this.$emit('refresh')
```

> ⚠️ 慎用。直接依赖父组件结构会导致组件不可复用。唯一比较安全的用法是 `this.$parent.$emit(...)` 但不常见。

### `$root`

根 Vue 实例（`new Vue()` 那个），所有组件往上找最终都是它。在 `main.js` 中创建的那个实例。

```js
this.$root.$router    // 等价于 this.$router
this.$root.$store     // 等价于 this.$store
```

**为什么可以这样写？** 因为 Vue Router 和 Vuex 在 `main.js` 中挂载到了根实例上：

```js
// main.js
new Vue({
  router,     // ← 挂在这
  store,      // ← 挂在这
  render: h => h(App)
}).$mount('#app')
```

> 几乎不需要用 `$root` 访问自定义数据。`$router` 和 `$store` 本身已经被 Vue 注入到每个组件了，直接 `this.$router` 就行。

### `$children`

当前组件的**直接子组件实例数组**。顺序和模板里写的顺序一致。

```js
// layout/index.vue 内部
this.$children    // [Sidebar组件实例, AppMain组件实例, ...]
this.$children[0].someMethod()   // 能调，但不推荐
```

**为什么不推荐？**

1. **顺序不稳定**：`v-if`/`v-for` 会影响子组件数量和顺序
2. **紧耦合**：和 `$parent` 一样，依赖具体子组件的结构

**应该用什么代替？**

```vue
<!-- 用 ref 精确拿到想要的子组件 -->
<child-component ref="child" />

<script>
this.$refs.child.fetchData()   // ✅ 精确、明确
</script>
```

> ⚠️ 不推荐。子组件顺序不保证稳定（`v-if` 会变），应优先用 `$refs`。

---

## 插槽相关

### `$slots`

当前组件接收到的**普通插槽内容**（虚拟节点数组）。不传数据，只传 DOM。

```vue
<!-- 父组件 — 在子标签之间塞 HTML -->
<my-dialog>
  <p>确定要删除吗？</p>           ← 父直接写好的内容
</my-dialog>

<!-- 子组件 my-dialog.vue — 定义 <slot /> 坑位 -->
<div class="dialog">
  <slot />                       ← 父的 <p> 渲染在这里
</div>

<!-- my-dialog 内部 JS -->
this.$slots.default    // [VNode] — 父给的那段 <p> 的虚拟节点
```

> 数据流向：父把 DOM 片段直接塞进子的 `<slot />` 坑位。不经过 props，不经过 data，硬塞。

### `$scopedSlots`

当前组件接收到的**作用域插槽函数**。每个插槽是一个**函数**，由子组件提供数据，父组件决定渲染成什么。即"子给数据，父定外观"。

```vue
<!-- ===== 父组件 outputindex.vue ===== -->
<el-table :data="listData">
  <el-table-column label="PN">            ← el-table-column 是子组件
    <template slot-scope="scope">         ← 这一段代码在父组件里！是你写的！
      {{ scope.row.PartNumber }}          ← 也是你的代码，决定显示哪个字段
    </template>
  </el-table-column>
</el-table>
```

```js
// el-table-column 内部（子组件）
listData.forEach((row, index) => {
    const content = this.$scopedSlots.default({
        row: row,                          // ← 子提供数据
        $index: index
    })
    // content = 父用子给的数据渲染出来的结果
})
```

> **分清父子和职责：** 你的页面（`outputindex.vue`）是 `el-table-column` 的父组件——你在模板里用了 `<el-table-column>` 标签。`el-table-column` 是子组件——它提供 `scope.row` 数据给父。父的 `<template slot-scope>` 接收数据后决定显示哪个字段。

**普通 vs 作用域插槽对比：**

| | `$slots`（普通） | `$scopedSlots`（作用域） |
|------|------|------|
| 谁提供内容 | 父直接写好 DOM | 父写模板，子提供数据 |
| 子拿到什么 | 静态 VNode 数组 | 函数，传入数据返回 VNode |
| 数据方向 | 不传数据 | 子→父（通过函数参数） |

### `$attrs`

**父组件传来、但子组件未用 `props` 声明**的所有属性（不含 class 和 style）。Vue 自动收集，但**不会自动生效**——需要你手动 `v-bind="$attrs"` 透传。

```vue
<!-- ===== 父页面 ===== -->
<my-input placeholder="请输入" :value="text" size="large" class="my-class" />

<!-- ===== 子组件 my-input.vue ===== -->
<template>
  <div class="wrapper">
    <!-- v-bind="$attrs" 把父传来的属性全部"倒"给原生元素 -->
    <input v-bind="$attrs" />
    <!-- 等效于 <input placeholder="请输入" :value="text" size="large" /> -->
  </div>
</template>

<script>
export default {
  // 故意不声明 props，让它们进 $attrs
  mounted() {
    console.log(this.$attrs)
    // { placeholder: '请输入', value: 'hello', size: 'large' }
    // class 和 style 不在里面——它们自动继承到根元素
  }
}
</script>
```

**完整流程：**

```
父在 <my-input placeholder="请输入" :value="text" />
  → my-input 没声明 props 接收
  → Vue 自动收集 → this.$attrs = { placeholder: '请输入', value: 'hello' }
  → 子模板里 <input v-bind="$attrs" />
  → 实际渲染成 <input placeholder="请输入" value="hello" />
```

> 如果没有 `v-bind="$attrs"`，父传来的属性就丢了，不会自动出现在任何 DOM 上。封装高阶组件（对 el-input 二次包装）时的核心工具。

**`inheritAttrs: false` 配合使用：**

默认情况下，`$attrs` 里的属性会**自动继承到子组件的根 DOM 元素**。设置 `inheritAttrs: false` 可以关掉这个行为，让 `v-bind="$attrs"` 完全由你控制绑到哪里：

```js
export default {
  inheritAttrs: false,   // 禁止自动继承到根元素
  // 现在 $attrs 只在你写 v-bind="$attrs" 的地方生效
}
```

### `$listeners`

父组件绑在子组件上的**事件监听器**（不含 `.native` 修饰符）。和 `$attrs` 一样，Vue 自动收集，但**不会自动生效**——需要你手动 `v-on="$listeners"` 透传。

```vue
<!-- ===== 父页面 ===== -->
<my-input @focus="handleFocus" @input="handleInput" @click.native="handleClick" />

<!-- ===== 子组件 my-input.vue ===== -->
<template>
  <div class="wrapper">
    <!-- v-on="$listeners" 把父的所有事件"倒"给原生元素 -->
    <input v-on="$listeners" />
    <!-- 等效于 <input @focus="handleFocus" @input="handleInput" /> -->
  </div>
</template>

<script>
export default {
  mounted() {
    console.log(this.$listeners)
    // { focus: handleFocus, input: handleInput }
    // click.native 不在里面——它跳过子组件直接绑在了根元素 <div> 上
  }
}
</script>
```

**完整流程：**

```
父在 <my-input @focus="handleFocus" @input="handleInput" />
  → Vue 自动收集 → 子 this.$listeners = { focus: handleFocus, input: handleInput }
  → 子模板里 <input v-on="$listeners" />
  → 用户在 <input> 上触发 focus → 父的 handleFocus 执行
```

**如果没有 `v-on="$listeners` 会怎样？** 父的事件就丢了，永远不会触发。你在 `<my-input>` 上写的 `@focus` 不会有任何反应。

**`.native` 的区别：**

| | `@focus`（进 $listeners） | `@click.native`（不进） |
|---|---|---|
| 绑在哪里 | 由子组件 `v-on="$listeners"` 决定 | 直接绑在子组件的**根 DOM 元素**上 |
| 子组件能拦截吗 | 能——不写 `v-on="$listeners"` 事件就丢了 | 不能——Vue 自动绑，跳过子组件 |
| 什么时候用 | 子组件内部有对应的原生元素来承接 | 你想监听子组件根元素的原生事件 |

**`$attrs` + `$listeners` 配合使用 — 完全透传：**

```vue
<!-- my-input.vue — 对原生 input 的二次包装 -->
<template>
  <div class="my-input-wrapper">
    <input v-bind="$attrs" v-on="$listeners" />
  </div>
</template>
```

父组件可以像用原生 `<input>` 一样用 `<my-input>`，属性、事件全部自动透传到底层 `<input>`。这是封装高阶组件最常见的模式。

---

## 事件相关

### `$emit(event, ...args)`

向父组件**发送事件**。这是子→父通信的标准方式。

```js
// 子组件
this.$emit('form-handled', 'done')       // 发事件 + 带参数

// 父组件模板
<child-component @form-handled="formHandled" />

// 父组件 methods
formHandled(val) {                        // val = 'done'
    console.log('子组件告诉我：', val)
}
```

**完整数据流：**

```
子 this.$emit('form-handled', 'done')
  → Vue 找到父组件模板里的 @form-handled="formHandled"
  → 执行父的 formHandled('done')
  → 父收到消息，做后续处理
```

**`$emit` 可以传多个参数：**

```js
this.$emit('update', id, name, status)
// 父收到的回调：handleUpdate(id, name, status)
```

**配合 `.sync` 修饰符（双向绑定语法糖）：**

```vue
<!-- 父 -->
<child :visible.sync="dialogVisible" />

<!-- 等价于 -->
<child :visible="dialogVisible" @update:visible="val => dialogVisible = val" />

<!-- 子 -->
this.$emit('update:visible', false)   // 父的 dialogVisible 自动变为 false
```

> `$emit` 是子→父通信的**唯一标准方式**。不要用 `this.$parent.xxx()` 绕过它。

### `$on(event, cb)`

**监听**事件。通常用在 eventBus（事件总线）场景——非父子关系的组件之间通信。

```js
// 组件 A — 创建 eventBus 并监听
const bus = new Vue()                    // 新建一个空 Vue 实例当"公交"

bus.$on('data-refresh', (payload) => {   // 等"车"来
    this.fetchData(payload)
})

// 组件 B — 发事件
bus.$emit('data-refresh', { id: 123 })  // 发一辆"车"
```

**eventBus 在你的项目中的实际用法：**

```js
// main.js 中通常已经挂好了
Vue.prototype.$bus = new Vue()

// 任意组件 A — 监听
mounted() {
    this.$bus.$on('refresh-table', this.fetchData)
}

// 任意组件 B — 触发
this.$bus.$emit('refresh-table')
```

**重要：** `$on` 注册的监听不会自动清理，组件销毁时回调还活着——造成**内存泄漏**。必须在 `beforeDestroy` 中手动 `$off`：

```js
mounted() {
    this.$bus.$on('refresh', this.fetchData)
},
beforeDestroy() {
    this.$bus.$off('refresh', this.fetchData)   // 必须！否则泄漏
}
```

### `$once(event, cb)`

监听事件，但**只触发一次**，触发后自动取消监听。不需要手动 `$off`。

```js
this.$bus.$once('app-ready', () => {
    console.log('应用初始化完成')       // 只执行一次，自动解绑
})
```

> 适合"等待某个一次性事件发生后执行"的场景，比如等待另一个组件初始化完毕。

### `$off(event, [cb])`

取消事件监听。

```js
// 取消该事件的所有监听器
this.$bus.$off('refresh')

// 取消该事件的指定回调
this.$bus.$off('refresh', this.fetchData)

// 不带参数 — 取消所有事件的所有监听器（慎用）
this.$bus.$off()
```

| 调用方式 | 效果 |
|------|------|
| `$off('event')` | 取消该事件的所有监听器 |
| `$off('event', cb)` | 只取消指定回调 |
| `$off()` | 取消当前实例的**所有**事件 |

---

## 生命周期相关

### `$mount([el])`

**手动将 Vue 组件挂载到 DOM**。如果调用时没传 `el` 参数，组件会先生成 DOM 但不插入页面，后续可手动插入。

```js
// 方式一：传选择器，自动挂载
import MyComponent from './MyComponent.vue'
const Comp = Vue.extend(MyComponent)
const instance = new Comp({ data: { msg: 'hello' } })
instance.$mount('#app')            // 挂载到 <div id="app"> 里

// 方式二：不传参数，手动插入
const instance = new Comp({ data: { msg: 'hello' } })
instance.$mount()                  // 生成 DOM 但不插到页面
document.body.appendChild(instance.$el)   // 手动插入
```

**什么场景会用到？**

| 场景 | 说明 |
|------|------|
| 动态弹窗 | 代码创建组件实例手动挂载，不用写在模板里 |
| 插件开发 | 比如 Loading/Toast 等命令式调用的组件 |
| 单元测试 | 手动创建和挂载组件进行测试 |

> 正常开发几乎不用——模板里写的组件 Vue 会自动帮你 `$mount`。只在需要**用 JS 命令式创建组件**时才用到。

### `$destroy()`

**手动销毁**组件实例。清理所有数据绑定、事件监听、子组件。

```js
this.$destroy()    // 组件凉了
```

**发生了什么事情？**

1. 触发 `beforeDestroy` 和 `destroyed` 生命周期
2. 移除所有 watcher（$watch 创建的）
3. 移除所有事件监听（$on 创建的）
4. 递归销毁所有子组件

> 和 `$mount` 一样，正常开发几乎不会手动调用。`v-if` 切换为 `false` 时 Vue 会自动调。只在**动态创建并手动管理的组件**中才需要手动销毁。

### `$options`

组件当前的**原始配置对象**——即 `export default { ... }` 那个对象。包含 `name`, `methods`, `computed`, `data`, `props` 等一切你写在组件选项里的东西。

```js
// 组件定义
export default {
    name: 'TEST_LIV',
    methods: { fetchData() {...}, handleSearch() {...} },
    customOption: '自定义的任意值'
}

// 运行时访问
this.$options.name           // 'TEST_LIV'
this.$options.methods        // { fetchData, handleSearch, ... }
this.$options.customOption   // '自定义的任意值'
```

**有什么用？**

1. **读取自定义选项**：Vue 允许你写自定义的选项，通过 `$options` 可以读取它们
2. **调试**：在控制台中查看组件的原始配置
3. **插件/混入开发**：通过 `$options` 读取特定配置

```js
// 比如你给某些组件标记了权限
export default {
    name: 'AdminPanel',
    permission: 'admin'       // 自定义字段
}

// 路由守卫中
if (to.matched.some(record => record.components.default.options.permission === 'admin')) {
    // 需要管理员权限
}
```
