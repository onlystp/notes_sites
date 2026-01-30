
# Options API vs. Composition API

這是開發者最有感的地方。

- **Vue 2 (Options API):** 將程式碼依據屬性分類（如 `data`, `methods`, `computed`）。當組件變大時，邏輯會像「散佈」在不同地方，維護起來不方便。
- **Vue 3 (Composition API):** 引入了 `setup()` 和 **Hooks** 的概念。你可以將相關聯的功能邏輯封裝在一起，大大提升了程式碼的可重用性與可讀性。

**Vue 2 (Options API):**

```js
export default {
  data() {
    return { count: 0 };
  },
  methods: {
    increment() { this.count++; }
  }
};
```

**Vue 3 (Composition API):**

```vue
<script setup>
import { ref } from 'vue';

const count = ref(0);
const increment = () => count.value++;
</script>
```


---

# 響應式原理（Proxy）

這是性能提升的關鍵。

- **Vue 2 (Object.defineProperty):**
    - **限制：** 無法偵測到物件屬性的新增或刪除，也無法直接監聽陣列索引的修改（需要動用 `this.$set`）。
        
- **Vue 3 (Proxy):**
    - **優勢：** 改用 ES6 的 `Proxy`。它能直接代理整個物件，無論你怎麼增刪屬性，Vue 都能精準掌握。
    - **結果：** 效能更好，記憶體佔用更少。
        

**Vue 2:**

```js
// ❌ 這樣寫畫面不會動
this.user.age = 25; 
// ✅ 必須使用專門的 API
this.$set(this.user, 'age', 25);
```

**Vue 3:**

```js
// ✅ 直接操作即可，Proxy 會自動偵測
user.age = 25; 
```


---

# 多根節點 (Fragment)

在 Vue 2 中，每個組件的 `<template>` 下必須只能有一個根元素（通常要包一個無意義的 `<div>`）。**Vue 3 支持多根節點**，你的 DOM 結構會更乾淨。

**Vue 2:**

```vue
<template>
  <div>
	<h1>標題</h1>
	<p>內容</p>
  </div>
</template>
```

**Vue 3:**

```vue
<template>
  <h1>標題</h1>
  <p>內容</p>
</template>
```

---

# 生命週期的改變

為了配合 Composition API，命名做了一些微調：

|**Vue 2**|**Vue 3 (in setup)**|
|---|---|
|`beforeCreate` / `created`|不需要 (直接在 `setup` 執行)|
|`beforeMount`|`onBeforeMount`|
|`mounted`|`onMounted`|
|`beforeUpdate`|`onBeforeUpdate`|
|`destroyed`|**`onBeforeUnmount`**|
|`destroyed`|**`onUnmounted`**|

**Vue 2:**

```js
export default {
  mounted() {
    console.log('組件掛載完成');
  }
}
```

**Vue 3:**

```vue
<script setup>
import { onMounted } from 'vue';

onMounted(() => {
  console.log('組件掛載完成');
});
</script>
```


---

# 其它

- **Teleport (傳送門):** 可以將組件（如 Modal、彈窗）渲染到 DOM 樹中的任何位置，解決 CSS 層級（z-index）被父層限制的問題。

	```vue
	<teleport to="body">
	  <div class="modal">
	    我是彈窗，我不受父層 overflow:hidden 的影響！
	  </div>
	</teleport>
	```
    
- **更好的 TypeScript 支持:** Vue3 本身就是用 TS 重寫的，對型別推斷非常友善。
    
- **Vite 構建工具:** 雖然 Vue2 也能用，但 Vue3 + Vite 的組合開發體驗極快，告別 Webpack 漫長的編譯等待。