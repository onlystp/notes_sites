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

- 源碼目錄為 packs（存放前端代碼）。
- 監視額外路徑如 views（當 Rails 視圖變更時自動重新打包）。
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
- 支援多種文件擴展名，包括 `.vue`、`.js`、`.ts` 等。
- 建置時將 Vue 和 Ant Design Vue 分離成獨立 chunk，以優化載入效能。

## 前端代碼結構

- 前端代碼放在 packs 目錄下：
  - `entrypoints/`：入口點文件，如 `application.js`（載入全局腳本）。
  - `src/vueComponents/`：存放 Vue 組件，如 `IndexPage.vue`。
  - `src/javascripts/`：存放 JavaScript 邏輯和插件。

## Vue 組件的載入機制

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

（此為專案的入口目錄）

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

vite_stylesheet_tag 跟 stylesheet_link_tag 差異？

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

💡 為什麼在 Vite 專案中不能混用？

如果你在專案中使用了 Vite，卻用 `stylesheet_link_tag` 去找 Vite 管理的檔案，會發生以下狀況：

	1. **找不到檔案：** Rails 的 Asset Pipeline 找不到 Vite 資料夾裡的檔案，會噴出 `Asset not found` 錯誤。

	2. **沒有 Tailwind 效果：** 因為你的 Tailwind 插件是掛在 Vite 下面的（例如 `postcss.config.js`），Sprockets 看不懂這些設定，導致樣式失效。

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

```js=
// app/packs/entrypoints/application.js
import '@/src/javascripts/session/profile.js'
```

Dev Mode

```js=
// app/packs/entrypoints/devApplication.js
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

# Staging Mode

在網站上產品前，可以跑 staging 來讓 QC 可以測試。staging 是最接近 production 的環境，所以應該得用編譯好的靜態檔，來跑網站的測試。

## DB key

公司的 DB yaml key 是雜湊出來的，本地需要有 key 可以解雜湊的碼。目前我們的 DB 在 QC 要測試時，會有 stagin DB，但目前練習可以先借用 dev DB 就好。

可以看到公司的 `config/credentials` 內已經有 `development.yml.enc`，裡面就是雜湊出來的亂碼。這是在 github 上拉得下來的，畢竟雜湊過了嘛。

我們需要在本地建立 `development.key` 並放入 key 來解開 `development.yml.enc` 的雜湊碼：

```tree
config/
	└─ credentials/
		├─ development.key
		└─ development.yml.enc
```

因為是跑 staging mode，所以我們也要建一個 staging 的雜湊碼與相對應的 key：

```shell
EDITOR="vim" bin/rails credentials:edit -e staging
```

這個指令是在啟動「解密 -> 編輯 -> 重新加密」的自動化流程。

- **解密：** 它會去讀取你的 `staging.key`，把原本亂碼般的 `staging.yml.enc` 解密成人類看得懂的文字。但如果這兩個檔案不存在：
	- 自動生成一把新鑰匙：在 `config/credentials/` 底下建立 `staging.key`。
	- 它會自動生成一個加密檔：在 `config/credentials/` 底下建立 `staging.yml.enc`。
- 開啟 Vim 編輯：：讓你編輯這個 `tmp/` 資料夾下，全新的、內容幾乎是空白的暫存檔。
- 重新加密： 當你在 Vim 輸入 `:wq` 存檔退出後，Rails 會立刻把新內容加密，並儲存覆蓋原本的加密檔。

所以上面指令，會產生：

```tree
config/
	└─ credentials/
		├─ staging.key
		└─ staging.yml.enc
```
## Build

```shell
$ RAILS_ENV=staging bundle exec vite build
```

## Staging Server

```shell
$ RAILS_ENV=staging bundle exec rails s -p ${PORT} -b 0.0.0.0
```

## 後續流程

公司的 [QC 測試 Server](https://scada.amastek.com.tw/.ror/) 屆時需要用 Apache HTTP Server 跟 Nginx 來跑 Server，用的 IP 也跟 VM 的不一樣。但這到時候再繼續往前學習，目前先搞懂 Staging 的設定跟機制即可。

## 清除靜態檔

clobber rails 關鍵字自行搜尋

```shell
$ precompile # 比較全面，因為是使用 rails 環境來編譯
```

## 延伸問題

- staging 的靜態檔放哪去了？

> [!SUCCESS] 解答
> 建在 `public/vite` 底下

### `0.0.0.0` 到底是幹嘛的？

> [!SUCCESS] `0.0.0.0` 是開放所有網域連上你開的伺服器
> 如果設定了，其他人不管 IP 是多少，你開伺服器的 IP（比方 `124.xxx.xxx:6001`）也可以拜訪；沒有設定，就只有同網域可以拜訪。

- 解完雜湊碼之後，怎麼連上 DB 的？
- 目前是自動產 staging key，到時候真的要讓 QC 測的話，還是自己產嗎？


# 前端套件使用

- [swal](https://sweetalert2.github.io/) - 處理反饋 dialogs
- antDesignVue - UI library
- dayjs
- jquery - 主要處理額外的頁面 event
- tablesorter
- yup - schema builder，主要用來處理 form 表單欄位 validate

# 專案可優化部分

- ✅ meta.html.erb 內的 charset="uft-8" 錯誤，應改成 "utf-8"
- 開發模式下，turbo event 堆疊會造成效能不好
- ESLint 跟 Prettier 的 format 衝突，會卡在存檔機制無法改成 ESLint 的格式。
	- 目前在 vscode 設定存檔行為可以稍微解決， 但文字長度排版、Object 排版不會被執行。

# 延伸問題


> [!SUCCESS] 共用的 Vue Component、全域 CSS Style 是可以被修改的嗎？
> 可以

> [!SUCCESS] 安裝享用的套件
> 可以
