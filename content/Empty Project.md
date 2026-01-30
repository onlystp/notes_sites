# Vue in Rails

公司 empty_project 透過 `vite_rails` gem 將 Vite（作為前端打包工具）與 Vue.js（作為前端框架）整合起來。以下是詳細的整合方式說明：

## Vite_rails Gem

在 Gemfile 中加入 `gem "vite_rails"`，這是官方的 Rails-Vite 整合 gem。

```json title="config/vite.json"
{
  "all"： {
    "sourceCodeDir"： "app/packs",
    "watchAdditionalPaths"： [
      "app/views"
    ]
  },
  "development"： {
    "autoBuild"： true,
    "host"： "0.0.0.0",
    "port"： 3036
  },
  "test"： {
    "autoBuild"： true,
    "publicOutputDir"： "vite-test",
    "port"： 3037
  }
}
```

- 源碼目錄為 `packs/`（存放前端代碼）。
- 監視額外路徑如 `views/`（當 Rails 視圖變更時自動重新打包）。
- 開發模式下自動建置並運行在 `http：//0.0.0.0：3036`。

## Vite 配置

```ts title="vite.config.mts"
import { defineConfig } from 'vite'
import RubyPlugin from 'vite-plugin-ruby'
import vue from '@vitejs/plugin-vue';
import inject from '@rollup/plugin-inject'

export default defineConfig({
  plugins： [
    RubyPlugin(),
    vue()
  ],
  build： {
    commonjsOptions： {
      transformMixedEsModules： true
    },
    rollupOptions： {
      output： {
        manualChunks： {
          vue： ['vue'],
          antd： ['ant-design-vue']
        }
      }
    },
  },
  resolve： {
    alias： {
      'vue'： 'vue/dist/vue.esm-bundler.js'
    },
    extensions： [".mjs", ".js", ".ts", ".jsx", ".tsx", ".json", ".vue", ".sass", ".scss", ".css", ".png", ".svg"],
  }
})
```

- 使用 `vite-plugin-ruby` 插件，讓 Vite 與 Rails 環境無縫整合。
- 加入 `@vitejs/plugin-vue` 插件來處理 `.vue` 文件。
- 配置 Vue 別名：`'vue'： 'vue/dist/vue.esm-bundler.js'`，確保使用完整的 Vue 版本。
- 支援多種文件，包括 `.vue`、`.js`、`.ts` 等。
- 建置時將 Vue 和 Ant Design Vue 分離成獨立 chunk，以優化載入效能。

## 前端代碼結構

- 前端代碼放在 packs 目錄下：
  - `entrypoints/`：入口點文件，如 `application.js`（載入全局腳本）。
  - `src/vueComponents/`：存放 Vue 組件，如 `IndexPage.vue`。
  - `src/javascripts/`：存放 JavaScript 邏輯和插件。

## Turbo

Turbo (Hotwire) 的事件體系非常完整，理解它們就像理解網頁的「生命週期」。當你從頁面 A 跳到頁面 B 時，Turbo 會經歷一系列過程，你可以根據需求在不同階段切入。

主要可以分為以下三大類：

### Event

下列為最常用的，當使用者點擊連結或瀏覽器前進/後退時觸發。

|**事件名稱**|**觸發時機**|**常用情境**|
|---|---|---|
|**`turbo:click`**|點擊標記為 Turbo 的連結時。|顯示自定義載入動畫。|
|**`turbo:before-visit`**|導航開始前（最快的一環）。|**取消導航**（例如表單未儲存時彈出確認）。|
|**`turbo:visit`**|向伺服器發送請求時。|紀錄追蹤代碼 (Analytics)。|
|**`turbo:submit-start`**|表單開始送出時。|禁用送出按鈕防止重複點擊。|
|**`turbo:before-render`**|得到回應後，**替換 HTML 前**。|修改回傳的 HTML 或加入切換動畫。|
|**`turbo:render`**|**替換 HTML 後**，渲染完成。|重新初始化頁面上的小工具。|
|**`turbo:load`**|頁面載入完成（初次或導航後）。|**最推薦的 JS 初始化位置**（類似 `DOMContentLoaded`）。|

#### 快取與清理事件 (Caching)

- **`turbo:before-cache`**：
    
    - **觸發時機**：在 Turbo 把當前頁面存入「快取快照」之前。
    - **關鍵用途**：**清理工作**。關閉 Observer、重設 Vue 實例、關閉打開的 Modal。如果不在此清理，回上一頁時會看到「殘留」的 UI 或重複的監聽器。

使用範例： `ResizeObserver` 洩漏問題的關鍵。

```js title:"user.js"
// 在 Turbo 把當前頁面存入快取前，把 Observer 停掉，才不會一直疊加新的監測
const cleanup = setIndexTableHeight(indexSearchApp)
document.addEventListener(
	'turbo:before-cache',
	() => cleanup?.(),
	{ once: true }
)
```

```js title:"setIndexTableHeight.js"
function setIndexTableHeight (searchSection) {
	const tableContainer = document.querySelector('.form-container')
	if (searchSection && tableContainer) {
		const resizeObserver = new ResizeObserver(() => {
			// 略
		})
		resizeObserver.observe(searchSection)
		// 離開頁面停止監測
		return () => resizeObserver.disconnect()
	}
}
```

#### Frame 與 Stream 事件 (局部更新)

當你使用 `turbo-frame` 或 `turbo-stream` 時會觸發。
- **`turbo:frame-load`**：當一個 `turbo-frame` 內容加載完成時。
- **`turbo:before-stream-render`**：當 Stream 準備插入新的 HTML 碎片前。

---

#### 💡 範例

情境 A：想在每一頁都執行某個 JS（例如初始化按鈕）

不要用原生 `window.onload`，要用 `turbo:load`：

```js
document.addEventListener('turbo:load', () => {
  console.log('頁面已就緒，無論是初次載入還是跳轉回來的')
})
```

---

情境 B：攔截導航（未儲存警告）

使用 `turbo:before-visit`：

```js
document.addEventListener('turbo:before-visit', (event) => {
  if (hasUnsavedChanges) {
    if (!confirm("變更尚未儲存，確定離開？")) {
      event.preventDefault() // 攔截並停止跳轉
    }
  }
})
```

---

情境 C：清理動作

使用 `turbo:before-cache`：

```js
document.addEventListener('turbo:before-cache', () => {
  // 這裡執行 disconnect() 或 destroy()
}, { once: true })
```

---

💡 為什麼要有這麼多事件？

因為 Turbo 不是真的「換頁」，它只是用 AJAX 抓回新頁面的 `<body>` 然後蓋掉舊的。如果你在 A 頁面監聽了 `scroll`，跳到 B 頁面時那個監聽器**依然活著**。必須養成在 `before-cache` 清理、在 `load` 或 `render` 重新掛載的好習慣。

**如果有遇到「按回上一頁，畫面長得怪怪的」或者「功能失效」**，這通常就是 `turbo:render` 與 `turbo:before-cache` 沒配對好的徵兆。

---

### Vue 組件的載入機制

**按頁面動態載入**：不是一次性載入所有 Vue 組件，而是根據 Rails 控制器和動作動態掛載。

```js title="app/packs/src/javascript/plugins/pageWatcher"
import '@hotwired/turbo'

const pageWatcher = ({ controller, action, handler }) => {
  if (!controller || !action || !handler) {
    console.warn('[PageWatcher] - Missing controller, action, or handler：', {
      controller,
      action,
      handler
    })
    return
  }

  const handleTurboEvent = (event) => {
    const body = document.body
    // 當頁面載入時檢查 data-controller 和 data-action 屬性    
    const windowAction = body.getAttribute('data-action')
    const windowController = body.getAttribute('data-controller')

    const actionMatches =
      (typeof action === 'string' && windowAction === action) ||
      (Array.isArray(action) && action.includes(windowAction))
    
    // 如果匹配指定條件，就執行對應的 handler 函數來掛載 Vue 應用
    if (windowController === controller && actionMatches && typeof handler === 'function') {
      handler(event)
    }
  }

  // 監聽 Turbo（Hotwire）事件（如 turbo：load）
  $(document).on('turbo：load turbo：render', handleTurboEvent)

  if (import.meta.env.DEV) {
    /* 只有在開發模式下需要直接觸發 */
    handleTurboEvent()
  }
}

export default pageWatcher
```


```js title="app/packs/src/javascript/session/index.js"
import { createApp } from 'vue'
import pageWatcher from '@/src/javascripts/plugins/pageWatcher'
import setFlashAlert from '@/src/javascripts/plugins/setFlashAlert.js'
import IndexPage from '@/src/vueComponents/session/IndexPage.vue'

pageWatcher({
  controller： 'session',
  action： ['index'],
  handler： () => {
    setFlashAlert()

    // 掛載到 DOM
    // createApp(IndexPage, { moduleList }).mount(indexApp)
    const indexApp = document.getElementById('index-app')
    if (indexApp && !indexApp.__vue_app__) {
      const moduleList = JSON.parse(indexApp.dataset.modules)
      createApp(IndexPage, { moduleList }).mount(indexApp)
    }
  }
})
```

## Rails View & Vue data-set

```html title="app/views/session/index.html.erb"
<% modules = [
  {
    link： setting_users_path,
    name： "基本設定",
    icon： "icon-setting-hollow",
    perm： true
  },
  {
    link： setting_users_path,
    name： "使用者設定",
    icon： "icon-setting-hollow",
    perm： true
  }
]
%>
<%= render "/notice" %>

<!-- 渲染一個 <div id="index-app"> 元素 -->
<!-- 使用 data 屬性傳遞數據，讓 JavaScript 可以讀取 -->
<%= content_tag "div", "", id： "index-app", data： { modules： modules } %>
```

Vue 組件透過 `props` 接收這些數據，並渲染動態內容。

（ 此為專案的入口 view ）

```js= title="app/packs/src/vueComponent/session/IndexPage.vue"
<script setup>
import { toRefs } from 'vue'

const props = defineProps({
  moduleList： {
    type： Array,
    default： () => {
      return []
    }
  }
})
const { moduleList } = toRefs(props)
const enableModuleList = moduleList.value.filter((moduleItem) => moduleItem.perm)
const welcomeText = `歡迎使用${I18n.t('system_name')}`
</script>

<template>
  <div class="container mx-auto flex justify-center pt-8">
    <div class="max-w-[904px] px-4 md：max-w-[768px]">
      <div class="item-center flex flex-wrap gap-3 md：justify-center">
        <template v-for="moduleItem in enableModuleList">
          <a
            v-if="moduleItem.perm"
            class="session-button"
            ：href="moduleItem.link"
            ：key="moduleItem.name"
          >
            <i ：class="[moduleItem.icon, 'text-[4rem]']"></i>
            <p class="mt-1 whitespace-nowrap text-xl">{{ moduleItem.name }}</p>
          </a>
        </template>
        <h2 v-if="enableModuleList.length === 0" class="mt-[2em] font-bold text-primary">
          {{ welcomeText }}
        </h2>
      </div>
    </div>
  </div>
</template>
```

## 開發與建置流程

- 開發模式：運行 `bin/vite dev`，Vite 會監視文件變更並自動重新打包。
- 建置：`bin/vite build` 產生生產環境的靜態文件。
- 將以上的 command 寫入 `bin/dev` 並接受動態輸入的 `port`，每個工程師可以定義自己的 PORT 阜。

# Views

## Main View

```html title:"views/layout/application.html.erb"
<!DOCTYPE html>
<html lang="zh-Hant-TW">
	<head>
		<title><%= t("system_name") %></title>
		<%= csrf_meta_tags %> <!-- 防止 CSRF 攻擊 -->
		<%= csp_meta_tag %>   <!-- 防止 XSS 攻擊 -->
		<%= render "/layouts/meta" %> <!-- 引入另外寫的 meta data -->
		<%= vite_client_tag %> <!-- Vite Hot Module Replacement -->
		<!-- 使用 Vite 掛載 SCSS -->
		<%= vite_stylesheet_tag "application.scss" %>
		<!-- 判斷環境並賦予相對應 JS 檔，接著以變數掛載 JS 檔 -->
		<% application_js = Rails.env.development?
			? "devApplication"
			: "application"	%>
		<%= vite_javascript_tag application_js %>
	</head>

  
	<body
		class="h-full w-full"
		data-controller="<%= controller.controller_path.to_s %>"
		data-action="<%= controller.action_name.to_s %>">
	
		<%= render "/layouts/header" %>
		<%= render "/layouts/i18n_js" %>
		<div id="app" class="pb-4">
			<%= yield %>
		</div>
		<%= render "/layouts/footer" %>
	</body>
</html>

```

### Vite vs Rails CSS#


❓ vite_stylesheet_tag 跟 stylesheet_link_tag 差異？

- `stylesheet_link_tag` 是 Rails 傳統的 **Sprockets (Asset Pipeline)** 在使用的
- `vite_stylesheet_tag` 則是交給現代化的 **Vite** 來處理。

| **特性**          | **stylesheet_link_tag**                 | **vite_stylesheet_tag**            |
| --------------- | --------------------------------------- | ---------------------------------- |
| 處理機制            | Rails 內建的老牌機制                           | 基於 **ES Modules** 的現代開發工具          |
| 開發模式            | 將檔案原封不動地丟給瀏覽器。如果你用 SCSS，Rails 會在後台慢慢編譯。 | 透過 Vite 的開發伺服器（Vite Dev Server）供檔。 |
| 速度              | 當 CSS 檔案很大時，重新整理網頁會感覺到明顯的延遲。            | 極快！它利用瀏覽器的原生能力，幾乎是瞬間載入。            |
| **熱更新 (HMR)**   | 無（需手動重新整理網頁）                            | 有（存檔後網頁 CSS 立即變更）                  |
| **Tailwind 整合** | 較慢，需透過 `tailwindcss-rails`              | **完美整合**，速度極快                      |
| **檔案存放位置**      | `app/assets/stylesheets`                | `app/frontend/...` (通常)            |
| **編譯工具**        | Ruby (Sprockets)                        | Go/JavaScript (esbuild/Rollup)     |

---

💡 為什麼在 Vite 專案中不能混用？

如果你在專案中使用了 Vite，卻用 `stylesheet_link_tag` 去找 Vite 管理的檔案，會發生以下狀況：

1. 找不到檔案：Rails 的 Asset Pipeline 找不到 Vite 資料夾裡的檔案，會噴出 `Asset not found` 錯誤。
2. 沒有 Tailwind 效果：因為 Tailwind 插件是掛在 Vite 下面的（例如 `postcss.config.js`），Sprockets 看不懂這些設定，導致樣式失效。

---

💡 該選哪一個？

- 如果你正在使用 **Tailwind CSS** 並且追求開發速度：**務必使用 `vite_stylesheet_tag`**。
- 只有當你有一些「極度傳統」的 CSS（例如放在 `vendor/assets` 裡的舊套件），且不想搬移到 Vite 結構時，才會用到 `stylesheet_link_tag`。

## Meta Data

```html title:"views/layouts/_meta.html.erb"
<!-- 告訴瀏覽器如何調整網頁的寬度與縮放 -->
<meta name="viewport" content="width=device-width,initial-scale=1.0">

<!-- 指定文件的類型與字元編碼 -->
<meta http-equiv="Content-Type" content="text/html"; charset="utf-8">

<!-- 強制 IE 使用最新引擎 -->
<meta http-equiv="X-UA-Compatible" content="IE=edge">

<!-- 聲明這個網頁的主要語言 -->
<meta http-equiv="Content-Language" content="zh-TW">

<!-- 定義瀏覽器分頁上顯示的小圖示 -->
<!-- 瀏覽器會自動根據目前的顯示需求，抓取最合適的檔案，確保圖示看起來不模糊 -->
<link rel="icon" type="image/png" sizes="32x32" href="/favicon-32x32.png">
<link rel="icon" type="image/png" sizes="16x16" href="/favicon-16x16.png">

<!-- Progressive Web App (PWA) 的設定檔 -->
<link rel="manifest" href="/site.webmanifest">

<!-- 專門給 macOS 的 Safari 瀏覽器使用 -->
<!--  當使用者在 Safari 中將你的網頁「釘選（Pin）」時，瀏覽器會顯示這個 SVG 圖示 -->
<link rel="mask-icon" href="/safari-pinned-tab.svg" color="#421d0b">

<!-- 專門給 Windows (8/10/11) 的開始功能表使用 -->
<meta name="msapplication-TileColor" content="#421d0b">

<!-- 設定手機瀏覽器（如 Chrome for Android）上方網址列的背景顏色 -->
<meta name="theme-color" content="#421d0b">-->

<!-- **Hotwire / Turbo Drive** 專用的設定 -->
<!-- **no-cache：加上這行後，Turbo 在你跳回此頁時會**強制重新抓取最新的內容 -->
<meta name="turbo-cache-control" content="no-cache">
```

# Routes

## 步驟一：設定 Route

首先，我們需要在 `config/routes.rb` 中設定路由，告訴 Rails 我們需要處理一個特定的 URL。

打開 `config/routes.rb ` 文件。

添加以下路由設定：

```ruby
# config/routes.rb
Rails.application.routes.draw do
  get 'profile', to： 'session#profile'
end
```

## 步驟二：創建 Controller

接下來，我們需要創建 session controller 和 profile action 來處理這個請求。

1. 在 `app/controllers` 資料夾中創建一個名為 `session_controller.rb` 的文件。
2. 添加以下代碼到 `session_controller.rb`：

```ruby
# app/controllers/session_controller.rb
class SessionController < ApplicationController
  def profile
      # profile 頁面設定
  end
end
```

## 步驟三：創建 View

最後，我們需要創建一個對應的 `View` 來顯示訊息給使用者。

1. 在 `app/views` 資料夾中創建一個名為 `session` 的子資料夾。
2. 在 `app/views/session ` 資料夾中創建一個名為 `profile.html.erb` 的文件。
3. 添加以下代碼到 `profile.html.erb`：

```html
<!-- app/views/greetings/hello.html.erb -->
<!DOCTYPE html>
<html>
<head>
  <title>Hello</title>
</head>
<body>
  <h1>哈囉！</h1>
</body>
</html>
```

或是可以用 Rails template 加入想加的變數，例如這邊是放入 `current user` 的資料，但只放不敏感的資料：

```html
<%# 個人設定頁面 %>
<%= render "/notice" %>
<%= content_tag "div",
	"",
	id： "profile-app",
	data： {
		user： current_user.as_json(
			only： [：id, ：account, ：name, ：email]
		)
	}
%>
```

## 步驟四：建立 `profile.js`

如果該頁面不是單純的 erb 檔，要掛載 Vue Component 的話，需要透過 ``app/packs/src/javascript/session/頁面.js`` 來掛載：

```js title="app/packs/src/javascript/session/profile.js"
import { createApp } from 'vue'
import pageWatcher from '@/src/javascripts/plugins/pageWatcher'
import setFlashAlert from '@/src/javascripts/plugins/setFlashAlert.js'
import ProfilePage from '@/src/vueComponents/session/ProfilePage.vue'
// 使用 pageWatcher 監視頁面載入
// 因為需要用 turbo 做到 SPA 效能
pageWatcher({
  controller： 'session',
  action： 'profile',
  handler： () => {
    setFlashAlert()
    const profileApp = document.getElementById('profile-app')
    /*
	如果畫面上找得到 profile-app
	而且它現在還沒被掛載過 Vue，我才要初始化它。
    */
    if (profileApp && !profileApp.__vue_app__) {
      // 拿到 dataset 的 current user 資料
      const user = JSON.parse(profileApp.dataset.user)
      // 在掛載的時候，傳入 Vue Component 作為 props 使用
      createApp(ProfilePage, { user }).mount(profileApp)
    }
  }
})
```

### pageWatcher

```js title:"app/packs/src/javascripts/plugins/pageWatcher.js"
import '@hotwired/turbo'
const pageWatcher = ({ controller, action, handler }) => {

	// 如果 <頁面>.js 沒有傳入該有的資訊就警告
	if (!controller || !action || !handler) {
		console.warn(
			'[PageWatcher] - Missing controller, action, or handler:',
			{
				controller,
				action,
				handler
			}
		)
		return
	}
	
	const handleTurboEvent = (event) => {
		/* 從 body 讀取動態的 action、controller 資訊
		   這邊的 controller 跟 action 是看 controller.rb 檔的命名
		   比方是 session_controller.rb 內的 profile action
		   這樣抓到的就是 session 跟 profile */
		const body = document.body
		const windowAction = body.getAttribute('data-action')
		const windowController = body.getAttribute('data-controller')
	
		/* 如果有符合條件就執行傳進來的 handler
		    這裡拿來比較的 action 就是 profile.js 丟進來的 'profile' */
		const actionMatches =
			(typeof action === 'string' && windowAction === action)
			|| (Array.isArray(action) && action.includes(windowAction))
		
		if (windowController === controller
			&& actionMatches
			&& typeof handler === 'function') {
				handler(event)
		}
	}
	
	// turbo 偵測到 load 跟 render 事件就啟動 event handler
	$(document).on('turbo:load turbo:render', handleTurboEvent)
	
	if (import.meta.env.DEV) {
		/* 只有在開發模式下需要直接觸發 */
		handleTurboEvent()
	}
}

export default pageWatcher
```

## 步驟五：引入 JS 文件

Prod Mode：

```js title:"app/packs/entrypoints/application.js"
import '@/src/javascripts/session/profile.js'
```

Dev Mode

```js title:"app/packs/entrypoints/devApplication.js"
import '@/src/javascripts/session/profile.js'
```

## 完整流程

1. 當使用者在瀏覽器中訪問 `http：//localhost：3000/profile` 時，請求會被路由設定 `config/routes.rb` 捕獲，並導向 `session#profile` 動作。
2. SessionController 中的 `profile` 動作會被呼叫。
3. Rails 會自動尋找並渲染對應的 `View (app/views/session/profile.html.erb)`。

# SSR from Controller

在 Rails 專案中，這種「非 API 式」的資料傳遞行為，我們通常稱為 **Server-side Rendering (SSR)**。必須掌握一個核心概念：**實體變數 (Instance Variables)**。

## @ Variables

在 Controller 的 Method（Action）中，凡是以 **`@`** 開頭的變數，就是會傳遞給 HTML 範本（View）的資料。

```ruby title:"app/controllers/application_controller.rb"
def index
  @products = Product.all  # 這就是你要用的資料
  @page_title = "所有商品清單"
end
```

- 只要在對應的 `index.html.erb` 中，你也可以直接使用 `@products`。

## 結構對應關係

> [!SUCCESS] 確認 Action 與 View 的對應關係
> Rails 遵循「慣例優於配置 (CoC)」。
>
> - 如果你的 URL 是 `/products`，通常對應 `ProductsController` 的 `index` 方法。
>     
> - 它會去尋找 `app/views/products/index.html.erb`。
>     
> - **技巧：** 如果你不知道資料哪來的，先看資料夾路徑，再去 Controller 找同名的 function。

### 空專案結構

```tree
views/
	├─ layouts/
	   ├─ application.html.erb
	   ├─ mailer.html.erb
	   └─ mailer.text.erb
	├─ session/
	   ├─ index.html.erb
	   ├─ login.html.erb
	   └─ profile.html.erb
	├─ setting/
	   ├─ import/
	      └─ index.html.erb
	   ├─ permissions/
	      ├─ _form.html.erb
	      ├─ edit.html.erb
	      ├─ index.html.erb
	      └─ new.html.erb
	   ├─ users/
	      ├─ _form.html.erb
	      ├─ edit.html.erb
	      ├─ index.html.erb
	      └─ new.html.erb
```

```tree
controllers/
├─ setting/
│  ├─ import_controller.rb
│  ├─ permissions_controller.rb
│  └─ users_controller.rb
├─ application_controller.rb
└─ session_controller.rb

```

---

💡 Rails 會把 `controllers/` 下的子資料夾路徑，對應到 `views/` 下的子資料夾。

| View                                     | 對應的 Controller                             | **對應的 Class 名稱**                 |
| ---------------------------------------- | ------------------------------------------ | -------------------------------- |
| `views/session/login.html.erb`           | `controllers/session_controller.rb`        | `SessionController`              |
| `views/setting/users/index.html.erb`     | `controllers/setting/users_controller.rb`  | `Setting::UsersController`       |
| `views/setting/permissions/new.html.erb` | `controllers/setting/permissions_controll` | `Setting::PermissionsController` |

在 `user_controller.erb` 內，會定義給不同的 `users/<view>.html.erb` 使用的 def，裡頭就會有可取變數。不可以跨著拿。

|**View 檔案**|**Controller 方法**|**關鍵變數 (範例)**|**資料狀態**|
|---|---|---|---|
|`index.html.erb`|`def index`|`@users`|複數筆資料 (Array)|
|`new.html.erb`|`def new`|`@user`|單筆、空的 (New Object)|
|`edit.html.erb`|`def edit`|`@user`|單筆、有資料的 (Existing Object)

---

💡 `_` 開頭的 Partials (局部樣板)。它們沒有自己對應的 Controller Action，它們是被別人「載入」進去的。

必須回到「呼叫它的主頁面」去看。 例如 `edit.html.erb` 裡面通常會有一行：

```html
<%= render "form", user: @user %>
```

這代表 `_form.html.erb` 裡面的資料是從 `edit` Action 傳進來的 `@user`。

---

💡 全局應用：`application_controller.rb`

因為所有的 Controller 都繼承自它，那邊定義的變數，全專案的 View 都能用。

---

## 常見的資料抓取語法

後端寫 Ruby 抓資料時，通常有幾種模式：

| **Ruby 語法**                        | **前端理解 (概念)**         |
| ---------------------------------- | --------------------- |
| `Product.all`                      | 拿到該資料表的所有資料           |
| `Product.find(params[:id])`        | 根據 URL 的 ID 拿到「單一筆」資料 |
| `Product.where(status: "active")`  | 篩選條件為 active 的資料      |
| `Product.order(created_at: :desc)` | 排序（由新到舊）              |
| `Product.limit(10)`                | 只拿前 10 筆              |

## Peek Data

### inspect

使用 `console.log` 的 Rails 版本：`inspe。在你的 `.html.erb` 檔案最上方直接寫入這行，儲存後看網頁：

```html
<pre><%= @products.inspect %></pre>
```

這會把整串資料以類似 JSON 的格式噴在網頁畫面上，讓你一眼看清楚有哪些欄位。

### debug

```html
<%= debug @products %>
```

這會產生一個比較漂亮的 YAML 格式框，顯示所有屬性。

## jbuilder

有時候，你會發現後端並不是直接把 `@` 變數丟到 HTML，而是透過 JavaScript 抓取比較複雜的資料。

- 檢查專案中是否有 `app/views/xxx/*.json.jbuilder` 檔案。
- 這是在 Rails 內部定義 JSON 結構的地方。如果你的 JS 代碼裡有 `fetch` 指向目前的 URL 且後綴是 `.json`，資料就是從這裡定義的。

## FE to BE：`params`

當你需要從前端點擊按鈕或帶參數給後端時，你會在 Controller 看到 `params`。

- **URL:** `/users?type=admin`
- **Controller:** `params[:type]` 就會是 `"admin"`。

# 前端套件使用

- [swal](https://sweetalert2.github.io/) - 處理反饋 dialogs
- antDesignVue - UI library
- dayjs
- jquery - 主要處理額外的頁面 event
- tablesorter
- yup - schema builder，主要用來處理 form 表單欄位 validate

# 專案可優化部分

## 已解決

- ✅ meta.html.erb 內的 charset="uft-8" 錯誤，應改成 "utf-8"

### ESLint and Prettier conflicts

ESLint 跟 Prettier 雖各有各的職責，但還是有重疊 format 的部分。這邊的解決方式，是以：<span style="color:rgb(0, 176, 80)">不修改團隊原本習慣的 format 為原則，讓 ESLint 去執行 Prettier 的規則。並安裝套件解決兩者衝突。</span>

1‍⃣ 步驟一

修改 VSCode user setting：

```json
{
	// 關掉 VSCode 預設的存檔 format 功能（預設是吃 Prettier）
	"editor.formatOnSave": false,
	// 打開 ESLint 的 auto format on save 功能
	"editor.codeActionsOnSave": {
		"source.fixAll.eslint": "explicit"
},
```

2‍⃣ 步驟二：

安裝 `eslint-plugin-prettier` 套件，這個套件的目的是讓 ESLint 去把 Prettier 的規則容納進來執行。

```shell
npm i -D eslint-config-prettier
```

然後在 ESLint 設定檔將剛剛安裝的套件引入，並註冊在 ESLint 的 `plugins`：

```js title:"eslint.config.mjs"
import pluginPrettier from 'eslint-plugin-prettier'

export default [
	plugins: {
		// 前略...
		prettier: pluginPrettier
	}
]
```

3‍⃣ 步驟三：

安裝 `eslint-config-prettier` 套件，讓這個套件自動解決兩者的衝突。

```shell
npm i -D eslint-config-prettier
```

然後在 ESLint 設定檔註冊成 config（要放在最後，（要放在最後，規則最後才會吃它的）：

```js title:"eslint.config.mjs
import configPrettier from 'eslint-config-prettier'

export default [
	// 前略...
	configPrettier
]
```

都完成之後建議重開 VSCode 或 Refresh window，確認問題是否都解決了。

## 待解決

- 開發模式下，turbo event 堆疊會造成效能不好


# 其它延伸問題


> [!SUCCESS] 共用的 Vue Component、全域 CSS Style 是可以被修改的嗎？
> 可以

> [!SUCCESS] 可以安裝想用的套件嗎？
> 可以
