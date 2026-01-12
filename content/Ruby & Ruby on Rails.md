# Ruby

## 特點

- OOP
- 動態型別：不需要宣告變數的型別，且變數可以隨時改變型別
- 豐富的 library

## 型別

### Integer

整數與整數運算，結果必為**整數**（無條件捨去小數點）。

```ruby
puts 10 / 3    # => 3 (不是 3.333)
puts 10 % 3    # => 1 (取餘數)
puts 2 ** 3    # => 8 (次方的運算子是 **)
```

### Float

只要運算中有任何一個數字是 Float，結果就會轉為 **Float**

```ruby
puts 10 / 3.0  # => 3.3333333333333335
puts 1.2 + 2   # => 3.2
```

### String

- `+` 用於**接合**兩個字串。
- `*` 用於**重複**字串次數。

```ruby
puts "Ruby" + " on Rails"  # => "Ruby on Rails"
puts "Ha" * 3             # => "HaHaHa"

# 注意：字串不能直接加數字，會噴錯 (TypeError)
# puts "Age： " + 25       # 錯誤！
puts "Age： " + 25.to_s    # => "Age： 25" (需手動轉型)
```

### Array

- `+` 合併兩個陣列。
- `-` 移除存在於第二個陣列中的元素。
- `<<` (Shovel operator) 在末端增加一個元素。

```ruby
list = [1, 2, 3]
puts list + [4, 5]    # => [1, 2, 3, 4, 5]
puts [1, 2, 2, 3] - [2] # => [1, 3] (所有的 2 都會被刪掉)

list << 4             # list 變成 [1, 2, 3, 4]
```

### Hash

像是 Object 的東西。
Hash 沒有 `+` 運算子，通常使用 `merge` 方法來合併。

```ruby
h1 = { a： 1, b： 2 }
h2 = { b： 99, c： 3 }

puts h1.merge(h2)     # => {：a=>1, ：b=>99, ：c=>3} (後方的 key 會覆蓋前方)
```

## 基本語法 - 流程控制

### If else

不需要小括號，結尾必須加上 `end`。

```ruby
score = 80
if score >= 60
  puts "及格"
elsif score >= 40 # 注意是 elsif，不是 elseif
  puts "補考"
else
  puts "不及格"
end
```

- **補充**：還有 `unless` 語法（當條件為假時執行），等於 `if !condition`。

### 三元運算子

```ruby
status = (age >= 18) ? "成年" ： "未成年"
```

### Case

```ruby
fruit = "apple"
case fruit
when "apple"
  puts "這是蘋果"
when "banana"
  puts "這是香蕉"
else
  puts "未知水果"
end
```

### Loop

Ruby 較少使用傳統的 `for`，多使用 `each` 或 `times` 等迭代器。

```ruby
# each 語法
[1, 2, 3].each do |num|
  puts num
end

# while 語法
i = 0
while i < 5
  puts i
  i += 1
end
```

### Function

- 在 Ruby 中使用 `def` 定義方法，結尾必須有 `end`。
- 隱含回傳 (Implicit Return)：Ruby 會自動回傳方法中「最後一行執行後的結果」，不需要寫 `return`。

```ruby
def add(a, b)
  a + b # 這行會被自動回傳
end

puts add(5, 3) # 印出 8
```

**慣例**：如果方法會回傳布林值（true/false），通常會在名稱後面加問號，例如 `empty?`。如果是會修改物件本身的「危險」方法，通常會加驚嘆號，例如 `sort!`。

## 語法特性補充

### Type Conversion

雖然 Ruby 是動態型別（變數不用宣告型別），但它是**強型別**，代表它不會幫你「自動轉型」（例如字串加數字會報錯）。你必須手動轉型：

- `.to_i`：轉成整數 (Integer)
- `.to_f`：轉成浮點數 (Float)
- `.to_s`：轉成字串 (String)
- `.to_a`：轉成陣列 (Array)

### 物件方法呼叫

在 Ruby 中，連數字都是物件，你可以直接對數字下指令：

```
puts -10.abs    # => 10 (絕對值)
puts 1.5.round  # => 2  (四捨五入)
puts 2.0.floor  # => 2  (無條件捨去)
```

### True | False

在 Ruby 的條件判斷中（If else），規則非常簡單：

- **只有 `nil` 和 `false` 是「假」**。
- **其餘所有東西都是「真」**（包含 `0`, `""`, `[]` 在 Ruby 裡通通都是 **True**）。這點跟 JavaScript 或 Python 非常不同，請特別注意。

# Rails
## 特點

- MVC 架構的 Web 框架
- 慣例優於配置：提供預設的應用程式結構和命名規則，能更快進行開發，無需手動配置大量設定
- 強大的函式庫
- 自動化測試
- 資料庫抽象層：支援多種常見的資料庫，如 MySQL、PostgreSQL、SQLite，並提供簡潔的資料庫操作接口
- 安全性：內建許多安全性功能，如 XSS、CSRF 保護等

## MVC 架構

![[Pasted image 20251229104629.png]]

1. 當有使用者輸入網址，連到你的網站的時候，第一關會遇到的是路徑對照表（Route，檔案：`config/routes.rb`）。
2. 在這個路徑對照表裡，記錄著這個網站對外開放的路徑對照表。Rails 會根據使用者輸入的網址及參數，比對這個路徑對照表的資料，告訴你應該去找哪個 `Controller` 上的哪個 `Action`；或是如果在對照表裡查不到對應的資料，就會告訴你 `HTTP 404` 找不到頁面。
3. 在 `Controller` 上通常會有一個以上的 `Action`，這些 `Action` 說穿了就是一般 Ruby 裡的方法（method）而已。透過路徑對照表，找到了對應的 `Action`，這個 `Action` 會決定要做什麼事。
4. 舉例個子來說，在這個 `Action` 可能會需要查閱「目前所有的商品列表」，接著它就會去請 `Model` 幫忙查資料。
5. Model 本身並不是資料庫，但它可以幫你把你跟 `Model` 說的「人話」轉成資料庫看得懂的資料庫查詢語言（Structured Query Language，簡稱 SQL）。
6. 透過 SQL，`Model` 可以跟資料庫取得你想要的資料。
7. `Model` 把這包資料交回 `Controller/Action` 手上。
8. 雖然 `Controller/Action` 拿到資料了，但目前這包東西還沒美化、整理過，還不適合給使用者看，所以 `Controller/Action ` 還需要跟 `View` 借一下畫面，讓資料更適合使用者閱讀。
9. `Controller/Action` 把資料跟 `View` 的畫面組合，最後呈現給使用者看。

## Global Resource

在 Rails 中，所謂的「全域資源」通常是指在 **Controller** 或 **View** 中可以直接調用的物件或方法。這些資源是由 Rails 框架自動注入的，不需要實例化或另外引入。

### 通用的請求與狀態物件

這類物件讓你處理瀏覽器傳來的資訊，或是在頁面間攜帶資料。

- **`params`**: 最常用的物件，包含網頁傳來的所有參數（Query String、表單資料、路由 ID）。
- **`session`**: 用於儲存使用者的登錄狀態（加密存於 Cookie 中，跨 Request 持續存在）。
- **`cookies`**: 直接讀取或寫入瀏覽器 Cookie。
- **`flash`**: 專門用於「暫時性提示訊息」（例如：「登入成功」），資料在下一個 Request 結束後會自動消失。
- **`request`**: 包含目前的 HTTP 請求詳細資訊（如 `request.fullpath`, `request.remote_ip`, `request.user_agent`）。
- **`response`**: 伺服器回傳給瀏覽器的內容物件。

### 環境與設定物件

- **`Rails.env`**: 判斷目前環境（`Rails.env.development?`, `Rails.env.production?`）。
- **`Rails.application.credentials`**: 讀取加密的秘密金鑰（如 API Key）。
- **`Rails.logger`**: 用於輸出日誌到控制台或檔案。

### View 專用的 Helper 方法

這些方法在所有 `.html.erb` 檔案中都可以直接使用：

- **路徑類**: `root_path`, `edit_user_path(id)`, `login_url`。
- **標籤類**: `link_to`, `image_tag`, `content_tag`。
- **表單類**: `form_with`, `text_field_tag`。

## 專案結構

```tree
repo/
- .bundle/
- app/
- bin/
- config/
- db/
- lib/
- public/
- spec/
- Gemfile
- package.json
- postcss.config.js
- tailwind.config.js
- vite.config.mts
- README.md
      
```

`.bundle/`：包含了與專案的 gem 依賴有關的文件，用於確保專案在不同環境中具有一致的依賴。

`app/`：這是 Rails 專案的核心目錄，包含了 MVC 架構中的模型、視圖和控制器。

`bin/`： 存放執行指令（例如伺服器啟動和任務管理）的可執行文件。

`config/`： 存放專案的設定文件。

`db/`： 與數據庫相關的文件，包括遷移和 schema。

`lib/`： 存放自定義函數庫、擴展模組和任務。

`public/`： 靜態文件資料夾，包含不需要處理的靜態資源。這些文件可以直接被網頁伺服器提供，如 404 錯誤頁面、favicon.ico 等。

`spec/`： 測試文件資料夾，通常包含 RSpec 測試。這些測試用於確保應用程式功能正常。

`Gemfile`： 定義專案所需的 gem 及其版本。Bundler 使用這個文件來管理專案依賴。

`package.json`： 定義專案的 Node.js 套件依賴、腳本和專案元數據。使用 npm 或 yarn 來管理這些依賴。

`postcss.config.js`： PostCSS 的配置文件，用於轉換 CSS 與插件，像是 Auto-prefixer。

`tailwind.config.js`： Tailwind CSS 的配置文件，用於自定義 Tailwind 的設置，如配色方案、斷點等。

`vite.config.ts`： Vite 的配置文件，用於配置 Vite 構建工具，包括路徑別名、插件和其他構建選項。

`README.md`： 專案的 README 文件，通常包含專案簡介、安裝指南、使用說明和貢獻指南。


```tree
- app/
	- actors/
	- assets/
	- blueprints/
	- channels/
	- controllers/
	- helpers/
	- jobs/
	- mailers/
	- models/
	- packs/
	- services/
	- views/
```

`app/` 目錄中的子目錄：

* `actors/`： 存放專門處理特定功能或業務邏輯。
* `assets/`：存放靜態資源（如 JavaScript、CSS 和圖片）
* `blueprints`： 存放視圖模板或數據 JSON 輸出格式的定義。
* `channels/`：存放 Action Cable 的 WebSocket 通道。
* `controllers/`： 控制器，負責處理請求和回應。
* `helpers/`：存放視圖幫助模組。
* `jobs/`：存放 Active Job ，用於後台作業。
* `mailers/`： 負責發送電子郵件的。
* `models/`： 資料模型和業務邏輯。
* `packs/`： 存放前端資源的地方，這些資源會被編譯並打包成適合瀏覽器使用的文件。
* `services/`： 通常用來封裝複雜的邏輯或處理流程，例如：用戶認證、數據導入導出、通知發送等。這有助於保持控制器和模型的簡潔，讓它們只關心各自的責任。
* `views/`： 存放視圖模板。


```tree
- config/
	- credentials/
	- envirimemts/
	- initializers/
	- locales/
	- application.rb
	- database.yaml
	- route.rb
```

`config`： 存放專案的設定文件。

* `environments/`：根據不同環境（如 development、test、production）進行配置的文件。
* `initializers/`：初始化設置文件，在應用程序啟動時運行。
* `locales/`：存放翻譯文件。
* `application.rb`：主要的應用配置文件。
* `database.yml`：數據庫配置文件。
* `routes.rb`：路由配置文件。

```tree
- db/
	- migrate/
	- schema.rb
	- seeds.rb
```

`db/`： 包括遷移文件（migrations）、種子數據（seeds）、架構文件（schema）等。

* `migrate/`：存放數據庫遷移文件，用於變更數據庫結構。
* `seeds.rb`：存放種子數據，用於初始化數據庫。
* `schema.rb`：存放數據庫結構的定義。

## ERB Views

Rails 有三種建立 template 的方式：

- `.html.erb`：會用內建功能產出 HTML response
- `.json.jbuilder`：會用 Jbuilder gem 來建立 JSON response
- `.xml.builder`： 會使用 Builder::XmlMarkup 函式庫來建立 XML response

目前僅需要知道 `.erb` 就好。

### Basics

Rails 的特殊 tags 為：

- `<% %>`：用來跑 Ruby 程式碼，但不產出 template
- `<%= %>`：產出 template

```html
<!-- 用迴圈產出 person.name 的 template -->
<h1>Names</h1>
<% @people.each do |person| %>
  Name: <%= person.name %><br>
<% end %>
```

### Partials

可拆出 component files 並以 `_component.html.erb` 命名後，用 `render` 屬性來動態產生 template：

```html
<%= render "application/product" %>
```

### Passing Data

父層使用 `local` 來傳遞資料：

```html
<h1>Products</h1>

<p>Here are a few of our fine products:</p>
<% @products.each do |product| %>
  <%= render partial: "product", locals: { product: product } %>
<% end %>
```

子層使用 `local_assigns` 執行條件渲染：

```html
<!-- 只有在外部有傳 redirect: true 時，才顯示這個欄位 -->
<% if local_assigns[:redirect] %>
  <%= form.hidden_field :redirect, value: true %>
<% end %>
```

精簡寫法：

```html
<!-- 如果只有傳一個 key 進去 locals -->
<%= render partial: "product", locals: { product: @product } %>

<!-- 精簡成 -->
<%= render "product", product: @product %>

<!-- 更精簡 -->
<%= render @product %>
```

```html
<!-- 如果 controller 內變數命名跟 partial 名衝突 -->

<%= render partial: "product", locals: { product: @item } %>

<!-- 可用 object 精簡 -->
<%= render partial: "product", object: @item %>
```

### Data From Collections

```ruby title:"app/controllers/users_controller.rb"
def show
  # 定義實例變數 @user
  @user = User.find(params[:id]) 
  
  # 沒帶 @ 的變數，View 讀不到
  local_info = "這段話傳不過去" 
end
```

```html title="app/views/users/show.html.erb"
<h1>使用者檔案</h1>
<p>姓名：<%= @user.name %></p>

<p><%= local_info %></p>
```

## 常見的 Rails 指令

### 建立 Rails Repo

`rails new myapp`

### 大懶包：Scaffold

快速建立一整套功能（包含 Model、Migration、Controller、Views 以及 RESTful Routes），最快的方式是：

```shell
bin/rails generate scaffold Product name:string price:integer
```

- **Routes**: 會在 `config/routes.rb` 自動加上 `resources :products`。
- **Views**: 會在 `app/views/products/` 下產生 `index`, `edit`, `show`, `new`, `_form` 等檔案。

### Controller + Views

如果你已經有 Model 了，或者這只是一個單純的展示頁面，可以使用 `controller` 產生器：


```shell
bin/rails generate controller Pages home about contact
```

- **Routes**: 會自動加上 `get 'pages/home'`, `get 'pages/about'` 等路徑。
- **Views**: 會在 `app/views/pages/` 產生 `home.html.erb`, `about.html.erb` 與 `contact.html.erb`。

### 生成 Controller

```shell
rails generate controller ControllerName action1 action2
```

### 生成 Model

```shell
rails generate model ModelName field1：type field2：type
```

### 運行 Migration

```shell
rails db：migrate
```

回復 Migration
```shell
rails db：rollback
```

### Resource

如果只需要 Routes、Controller 和 Model，但想**手動**刻 View，可以使用：

```shell
bin/rails generate resource Post title:string body:text
```

這會設定好 `resources :posts` 的路由，並建立空的 Controller，但**不會**幫你寫好 View 的內容（只會建立資料夾）。

### 查看 Routes

在指令列輸入以下指令，可以查看目前專案中定義的所有路由路徑、對應的 Controller 與 Action：

```shell
bin/rails routes
```

> **小技巧**：如果路由太多，可以用 `-g` 過濾，例如 `bin/rails routes -g products`。

### 刪除產生的檔案

如果不小心打錯字或想重來，只要把 `generate` 改成 `destroy`（簡為 `d`）即可：

```shell
bin/rails destroy controller Pages home
```

這會自動幫你刪掉對應的 Controller 檔案、View 檔案以及移除 `routes.rb` 裡的相關設定。

### 總結對照表

| **指令**       | **產生 Routes** | **產生 Views** | **產生 Model** | **適用情境**           |
| ------------ | ------------- | ------------ | ------------ | ------------------ |
| `scaffold`   | ✅ (RESTful)   | ✅ (完整)       | ✅            | 快速建立 CRUD 功能       |
| `controller` | ✅ (特定)        | ✅ (空白)       | ❌            | 靜態頁面或特定邏輯頁面        |
| `resource`   | ✅ (RESTful)   | ❌            | ✅            | 需要 API 或自定義 View 時 |

- 如何在 Rails 中設定路由？

    打開 `config/routes.rb` 文件，添加你的路由。

- 如何在控制器中使用參數？

    可以使用 params 來訪問 URL 中的參數。
    ```ruby
    class UsersController < ApplicationController
      def show
        @user = User.find(params[：id])
      end
    end
    ```

- 如何在視圖中顯示變數？
    在控制器中設置實例變數，然後在視圖中使用 ERB 語法來顯示。

    例如，在控制器中：
    ```ruby
    class WelcomeController < ApplicationController
      def index
        @message = "Hello, Rails!"
      end
    end
    ```

    在視圖中：

    ```html
    <%= @message %>
    ```

- 如何在 Rails 中使用部分視圖（partial）？
    使用 render 方法來渲染部分視圖。

    例如，在 `app/views/shared/_header.html.erb` 中定義部分視圖：

    ```html title="app/views/shared/_header.html.erb"
    <header>
      <h1>My Application</h1>
    </header>
    ```

    然後在其他視圖中渲染這個部分視圖：

    ```html
    <%= render 'shared/header' %>
    ```

- 如何在 Rails 中使用表單？
    ```html
    <%= form_tag('/welcome/submit_form') do %>
        <div>
          <%= label_tag(：name, "Name：") %>
          <%= text_field_tag(：name) %>
        </div>

        <div>
          <%= label_tag(：email, "Email：") %>
          <%= text_field_tag(：email) %>
        </div>

        <div>
          <%= submit_tag("Submit") %>
        </div>
      <% end %>
    ```
    
