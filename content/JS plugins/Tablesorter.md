
# 快速上手

`tablesorter` 是一個 jQuery 插件，能將標準的 HTML 表格（帶有 `thead` 和 `tbody`）轉化為可排序的表格。

## ## 安裝

使用 `tablesorter` ，`jquery` 也是必要安裝的套件

```shell
npm install jquery tablesorter
# 或者
yarn add jquery tablesorter
```

引入 JS 檔

```js
import jquery from 'jquery'
import 'tablesorter'
```


---

## 綁定 HTML

要讓一個表格擁有排序功能，最基本的要求是 HTML 結構必須包含 `<thead>` 與 `<tbody>`。


```html
<table class="querysorter">
  <thead>
    <tr>
      <th>姓名</th>
      <th>年齡</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>張小明</td>
      <td>25</td>
    </tr>
  </tbody>
</table>
```



```js
const setQuerysorter = () => {
  const $tablesorter = jquery('.querysorter');
  
  if ($tablesorter.length) {
    $tablesorter.tablesorter(); // 啟動預設的前端排序
  }
}
```

# SSR 列表排序


透過 `tablesorter` 結合 URL 參數，實現 SSR 列表排序，並可以達到「頁面重新整理或導向後，排序狀態仍能保持」。

## 設定

以下是適合 SSR 的配置：

| **屬性**              | **說明**                                              | **範例**                     |
| ------------------- | --------------------------------------------------- | -------------------------- |
| `headers`           | 針對特定欄位設定排序功能（如禁用排序）                                 | `{ 0: { sorter: false } }` |
| `sortList`          | 初始排序狀態，格式為 `[[index, direction]]` (0: asc, 1: desc) | `[[0, 0]]`                 |
| `serverSideSorting` | **關鍵設定**。若為 `true`，點擊表頭僅會改變 UI，不會實際移動前端 DOM 列       | `true`                     |
| `sortReset`         | 允許循環點擊回到 Reset（不排序）                                 | `true`                     |
| `sortMultiSortKey`  | 多欄位排序，若設為 `null` 則禁用多欄位排序                           | `null`                     |


在為 DOM 啟動 `tablesorter()` 同時，在裡面寫設定：

```js
$tablesorter.tablesorter({
  headers: {                         // 定義哪些列表有 sorting 功能
	  0: { sorter: false },
	  1: { sorter: true }
  },
  sortList: [[0, 0]],                // 初始狀態同步
  serverSideSorting: true,           // 告知套件不要在前端自行排序
  sortReset: true,                   // 允許點擊多次後回到不排序狀態
  sortMultiSortKey: null             // 禁用多欄位排序
})
```


## 定義可排序欄位

tablesorter 的 `headers` 可以用來設定哪些可以排序，哪些不能排序。它是用 index 來表示欄位，而不是 key、value，所以比較好的做法，是讓開發者可以自己寫 table column，再轉換成 headers 看得懂的格式：

```js
const columns = [
	{ title: null, field: null, sort: false },
	{ title: '狀態', field: 'state', sort: true },
	{ title: '帳號', field: 'account', sort: true },
	{ title: '姓名', field: 'name', sort: true },
	{ title: '權限', field: 'permission', sort: false }
]

const headers = columns.reduce((accu, curr, index) => {
	accu[index] = { sorter: curr.sort }
	return accu
}, {})
```

## Query Sync

因為是 SSR 的做法，透過 URL Queries 來跟後端的資料互動。那過程中就會產生問題：

👎  URL 變更，表單就也會重新渲染，排序就會一直回到預設狀態

所以要讓表單渲染的同時，在初始化設定內讓 sortList 跟 URL Query 同步：

```js
const sortPrefix = 'sort['

function getQueries () {
	return new URLSearchParams(window.location.search)
}

function sortSyncer (columns) {
	const queries = getQueries()
	
	// 從 URL Queries 抓取目前正在 sorting 的欄位
	const columnIndex = columns.findIndex((column) => {
		queries.has(`${sortPrefix}${column.field}]`
	}))
	
	if (columnIndex === -1) return [] // 無欄位排序就回傳空陣列
	
	// 從 URL Queries 抓取該 sorting 欄位的 direcction (asc 或 desc)
	const currentDirection = queries
		.get(`${sortPrefix}${columns[columnIndex].field}]`)
	
	// tablesorter 用 0 跟 1 來代表 'asc' 跟 'desc'
	const columnDirection = currentDirection === 'asc' ? 0 : 1
	
	return [[columnIndex, columnDirection]] // 整理成用 sortList 認得的格式
}
```


> [!SUCCESS]  補充：`sortList` 是用雙陣列來表示：
> ```js
> const sortList = [
> 	[0, 1],  // index 0 的表單欄位排序為 'desc'
> 	[1, 0],  // index 1 的表單欄位排序為 'asc'
> 	[2, 0],  // index 2 的表單欄位排序為 'asc'
> ]
> ```
> 
> 
> 不在這個陣列內的欄位，就是處在無排序狀態

```js
$tablesorter.tablesorter({
	headers,
	sortList: sortSyncer(columns),
	sortMultiSortKey: null,
	serverSideSorting: true,
	sortReset: true
})
```


## Redirect URL

因為是 SSR 的排序方式，所以原本 tablesorter 的排序行為已經關掉，不是前端 DOM 的排列了。當使用者按下排序鍵，這邊需要把收到的「要排序欄位 column」跟「方向 direction」，轉換成 URL Queries：

```js
const redirectUrl = (field, direction) => {
	const queries = new URLSearchParams(window.location.pathname)
	// 加入新的排序參數
	if (field && direction) queries.set(`${sortPrefix}${field}]`, direction)
	// 將組好的 URL 導向
	const newUrl = `${window.location.pathname}?${queries.toString()}`
	window.Turbo.visit(newUrl) // 使用 turbo 導向，避免頁面重整
}
```


---

# Event

可以使用 jQuery 的 `.on()` 來監聽：

### Event Listener

| **事件名稱**              | **觸發時機**            | **常用情境**                                                  |
| --------------------- | ------------------- | --------------------------------------------------------- |
| **`sortBegin`**       | 當使用者點擊列表標題，排序即將開始時。 | 顯示「載入中」的遮罩或動畫。                                            |
| **`sortEnd`**         | 排序邏輯執行完畢且 UI 已更新時。  | 這時候才能取得取得最新的 `sortList` ，所以我們的需求適合在這個階段執行 URL 跳轉或 API 請求。 |
| **`refreshComplete`** | 當表格重新整理。            | 處理與表格外掛相關的 UI 更新。                                         |
| **`filterEnd`**       | 如果有使用篩選外掛，在篩選結束時觸發。 | 取得篩選後的結果總筆數。                                              |

## Event Trigger

當變動了表格資料或狀態，需要主動告訴 `tablesorter` 的指令，使用 `.trigger()` 發送。通常是適合前後端分離，透過 API 拿到資料的做法。

| **指令名稱**          | **用途**                                             | **範例**                                       |     |
| ----------------- | -------------------------------------------------- | -------------------------------------------- | --- |
| **`update`**      | **最重要**。當你動態新增/刪除 `<tbody>` 裡的資料列時，必須呼叫此指令讓套件重新解析。 | `$('table').trigger('update');`              |     |
| **`sorton`**      | 從外部透過程式碼強制指定排序條件。                                  | `$('table').trigger('sorton', [ [[0,0]] ]);` |     |
| **`appendCache`** | 僅更新快取而不重新解析整個 DOM（效能優化用）。                          | `$('table').trigger('appendCache');`         |     |
| **`destroy`**     | 徹底移除表格的 tablesorter 功能，恢復成一般 HTML 表格。              | `$('table').trigger('destroy');`             |     |

# 完整程式碼範例


```js
import jquery from 'jquery'
import 'tablesorter'

const sortPrefix = 'sort['
const sortRegex = /^sort\[.*\]$/

/**
 * 取得當前 URL 的搜尋參數
 */
const getQueries = () => new URLSearchParams(window.location.search)

/**
 * 核心功能：將 URL 參數同步至 Tablesorter 內部狀態
 */
const sortSyncer = (columns) => {
  const queries = getQueries()
  // 找出目前 URL 中哪一個欄位正在排序
  const columnIndex = columns.findIndex((column) => queries.has(`${sortPrefix}${column.field}]`))

  if (columnIndex === -1) return [] 

  const currentDirection = queries.get(`${sortPrefix}${columns[columnIndex].field}]`)
  const columnDirection = currentDirection === 'asc' ? 0 : 1 // 0: asc, 1: desc

  return [[columnIndex, columnDirection]]
}

/**
 * 觸發頁面跳轉並更新 URL
 */
const redirectUrl = (field, direction) => {
  const queries = getQueries()

  // 1. 清理：移除舊排序與頁碼 (回到第一頁)
  queries.forEach((value, key) => {
    if (sortRegex.test(key) || key === 'page') queries.delete(key)
  })

  // 2. 加入新排序參數
  if (field && direction) queries.set(`${sortPrefix}${field}]`, direction)

  // 3. 導向新網址
  const newUrl = `${window.location.pathname}?${queries.toString()}`
  window.Turbo.visit(newUrl) 
}

/**
 * 主入口：初始化 Table 排序設定
 */
const setQuerysorter = (columns) => {
  const $tablesorter = jquery('.querysorter')
  if (!$tablesorter.length) return

  // 依照欄位定義組裝 headers 設定 (是否開啟排序功能)
  const headers = columns.reduce((accu, curr, index) => {
    accu[index] = { sorter: curr.sort }
    return accu
  }, {})

  $tablesorter.tablesorter({
    headers,                       // 將轉換過的標題排序餵給設定
    sortList: sortSyncer(columns), // 同步 URL 狀態
    sortMultiSortKey: null,        // 禁用多欄位同時排序 (視需求設定)
    serverSideSorting: true,       // 強制關閉前端排序邏輯
    sortReset: true
  })

  // 監聽排序結束事件
  $tablesorter.on('sortEnd', (event) => {
    const config = event.target.config
    const sortList = config.sortList

    if (sortList.length > 0) {
      const columnIndex = sortList[0][0]
      const direction = sortList[0][1] === 0 ? 'asc' : 'desc'
      const field = columns[columnIndex].field
      redirectUrl(field, direction)
    } else {
      // 若取消排序 (sortReset)，則移除參數
      redirectUrl(null, null)
    }
    return false // 阻止事件冒泡
  })
}

export default setQuerysorter
```