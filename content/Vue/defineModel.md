`defineModel` 是 Vue 3.4 引入的 Compiler Macro。 它的出現是為了解決 Vue 長久以來「雙向綁定 (v-model) 寫法太繁瑣」的問題。

# 語法

如果你要建立一個自定義輸入框：

 舊寫法：需要定義 `props` 接收值，並定義 `emits` 來手動觸發更新。
    
```js
const props = defineProps(['modelValue']);
const emit = defineEmits(['update:modelValue']);

const onInput = (e) => {
  emit('update:modelValue', e.target.value);
};
```
    
新寫法：直接宣告一個變數，它既是數據源也是更新觸發器。

```js
const model = defineModel(); // template 可以直接使用 v-model="model"
```


---

# Template 用法

## 基本雙向綁定

在子組件中，`model` 的行為就像一個普通的 `ref`：

```vue
<script setup>
const model = defineModel();
</script>

<template>
  <input v-model="model" />
</template>
```

## 具名綁定 (多個 v-model)

如果你的組件需要綁定多個值（例如：標題、內容）：

```vue
// 子組件
<script setup>
const title = defineModel('title');
const count = defineModel('count', { default: 0 });
</script>
```

```vue
// 父組件使用
<MyComponent v-model:title="pageTitle" v-model:count="itemCount" />
```

---
# Modifiers

這可以處理像 `.trim` 或 `.uppercase` 自定義的修飾符：

```vue title:"MyInput.vue"
<script setup>
const [model, modifiers] = defineModel({
  // 當 modelValue 被修改時（例如透過 v-model 綁定到 input）
  // set 函數會被觸發，value 是準備存入的新值
  set(value) {
    if (modifiers.capitalize) {
      return value.charAt(0).toUpperCase() + value.slice(1);
    }
    return value;
  }
});
</script>

<template>
  <input v-model="model" type="text" />
</template>
```

```vue title:"App.vue"
<script setup>
import { ref } from 'vue';
import MyInput from './MyInput.vue';
const userName = ref('');
</script>

<template>
  <MyInput v-model.capitalize="userName" />
  <p>目前的用戶名：{{ userName }}</p>
</template>
```
