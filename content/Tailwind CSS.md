
目前公司專案都是用 v.3，未來可能會升版至 v.4。

# Installation

## V.3

在專案夾下安裝 v.3 Tailwind CSS

```shell
npm install -D tailwindcss@3
```

生成設定檔

```shell
npx tailwindcss init
```

接著就可以在設定檔內寫入全域設定

```js title:"tailwind.config.js"
/* 自行發揮 */
export default {
  content: ["./src//*.{html,js}"],
  theme: {
    extend: {},
  },
  plugins: [],
}
```

接著引入 tailwind 的三大核心 styles，並打包成 tailwind.css

```css title:"app/packs/src/stylesheets/tailwindcss.css"
/* 因權重的考量，一定要照順序放 */
@import 'tailwindcss/base';        /* normalize */
@import 'tailwindcss/components';  /* component style */
@import 'tailwindcss/utilities';   /* 各種細部的 class */
```

然後引入 application.scss

```css title:"app/packs/entrypoints/application.scss"
@use '@/src/stylesheets/tailwindcss.css';

/* 還有其他 CSS，此略... */
```


## V.4

安裝套件

```shell
npm install tailwindcss @tailwindcss/vite
```

將 Tailwind 注入 Vite 環境

```js title:"vite.config.js" hl:2,6
import { defineConfig } from 'vite'
import tailwindcss from '@tailwindcss/vite'

export default defineConfig({
  plugins: [
    tailwindcss(),
  ],
})
```

引入 Tailwind CSS 的 style

```css title:"application.css
@import "tailwindcss";
```



> [!NOTE] 升版細節
> 請見 [V.3 to V.4 Upgrade Guide](https://tailwindcss.com/docs/upgrade-guide)


# Configuration

公司使用到的設定：

```js title:"tailwind.config.js"
module.exports = {
	mode: 'jit',  // just in time 模式，存檔即時編譯
	content: [    // 告訴 Tailwind 哪些檔案會使用到 CSS
		'./app//*.html*',
		'./app//*.vue',
		'./app/packs//*.scss',
		'./app/packs//*.css'
	],
	important: true,  // Tailwind 的 style 加上 !important
	theme: {
		extend: {
			screens: {  // 螢幕適應斷點
				xxl: '1920px',
				xl: {
					max: '1439px'
				},
				lg: {
					max: '1279px'
				},
				md: {
					max: '1023px'
				},
				sm: {
					max: '767px'
				}
			},
			colors: {                // 客製化全域的 color theme
				primary: '#4479b6',  // class ex: bg-primary 
				secondary: '#8bb03e',
				tertiary: '#1d4481',
				system: {            // class ex: text-system-textgray-1
					textgray: {
						1: '#FAFAFA'
					},
					gray: {
						1: '#cbc7cb'
					},
					indigo: {
						1: '#dedee8'
					},
					// ...略
				}
			},
			// position 的定位客製化
			inset: {                  // class ex: top--05
				0: 0,
				'1/2': '50%',
				'-05': '-1rem'
			},
			// 字體隨視窗寬度適應大小
			fontSize: {               // class ex: text-1vw
				'1vw': '1vw',
				'1.5vw': '1.5vw',
				'2vw': '2vw',
				'2.5vw': '2.5vw'
			},
			// 扣掉 header 跟 footer 的畫面高度
			height: {                 // class ex: h-screenContent
				screenContent: 'calc(100vh - 6.5vw)'
			},
			// 扣掉 header 跟 footer 的畫面高度
			minHeight: {              // class ex: min-h-screenContent
				screenContent: 'calc(100vh - 6.5vw)'
			},
			// 客製化更多的 background opacity
			backgroundOpacity: {      // class ex: bg-opacity-10
				10: '0.1',
				20: '0.2',
				95: '0.95'
			}
		}
	}
}
```


> 更多設定請見 [Tailwind CSS Configuration](https://v3.tailwindcss.com/docs/configuration)


# Plugins

公司使用到的 Tailwind CSS Plugins：

1. `@tailwindcss/aspect-ratio`
	- 用途： 強制元素保持特定的寬高比（如 16:9 或 1:1）。
	- 範例： `aspect-w-16 aspect-h-9`，保持 16:9 寬高比。
	- 版本適應： 雖然 Tailwind v3 內建 `aspect-video`，但不支援 IE 瀏覽器，所以套件保留可以相容舊版瀏覽器。
	- [官方文件](https://github.com/tailwindlabs/tailwindcss-aspect-ratio)

---

2. `@tailwindcss/forms`
	- 用途： 移除所有表單相關的 html tag （checkbox、input、radio ... 等）預設樣式，並統一不同瀏覽器（Chrome, Safari, Firefox）的表單樣式。
	- 範例：  `class="form-checkbox text-primary rounded"`。
	- [官方文件](https://tailwindcss-forms.vercel.app/)

---

3. `@tailwindcss/line-clamp`
	- 用途：限制文字顯示的行數，超過的部分顯示省略號`...`。
	- 範例：`line-clamp-2`：文字只顯示兩行，超過就變 `...`。
	- 版本適應： Tailwind v3.3 之後已將此功能內建。
	- [官方文件](https://www.npmjs.com/package/@tailwindcss/line-clamp)

---

4. `@tailwindcss/typography`
	- 用途： 當內容是從後端吐出來的 Markdown 或 HTML 時，沒辦法手動幫每個 `<h1>`、`<p>` 加上 Tailwind Class。它提供一個 `prose` 類別，可以自動幫內部的標題、清單、連結加上樣式。
	- 範例：`<article class="prose lg:prose-xl"> {{ content }} </article>`
	- [官方文件](https://github.com/tailwindlabs/tailwindcss-typography)

# 使用

以上都設定好後，就可以開始使用了

```html
<div class="relative h-fit basis-[250px] rounded-sm border-2 md:w-full">
```

以上包含了 Tailwind CSS utility class，也有一些是 plugin 提供的 class。這邊需要時翻翻文件就好。


# 樣式打包與復用

如果不想要寫落落長的 class：

## Custom Utilities

寫在 tailwind config 內，但最好是很多地方要用到，不然沒必要寫全域設定。

```js title:"tailwind.config.js"
// tailwind.config.js
module.exports = {
  theme: {
    extend: {
      boxShadow: {
        'company-soft': '0 2px 15px rgba(0, 0, 0, 0.05)',
      }
    }
  }
}

// 在 HTML 使用 class：shadow-company-soft
```

## Custom Classes

使用 `@layer` 與 `@apply`

- Global Use：寫在 tailwindcss.css 內
- Local Use：寫在 vue component style 內

```scss
@layer components {
  .my-custom-card {
    /* 使用 @apply 將多個 tailwind class 組合在一起 */
    @apply bg-white shadow-lg rounded-xl p-6 border border-system-gray-1;
    
    /* 混用原生 CSS */
    transition: transform 0.3s ease;
  }

  .my-custom-card:hover {
    @apply shadow-2xl;
    transform: translateY(-5px);
  }
}
```


> [!NOTE] 一次性的特殊樣式
> 
> 如果只有一處需要特殊的樣式，又不想要特別另外寫，可以用 [] 客製化：
> 
> ```html
> <div class="w-[150px] bg-[#b8860b] rotate-[3deg]">
>   一次性客製化
> </div>
> ```

