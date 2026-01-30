
# 概念

- 單元測試：以 function 為單位做測試
- 整合測試：以 module 為單位做測試
- 端對端 E2E：模擬使用者場景做測試

# 要測什麼

重要、影響使用的功能，比方金流、登入等影響公司產品運作的功能。

# 前置作業

- 環境的搭建
因為是很尾端的測試，所以環境能否一致、資料能否一致是會面臨到的困難，這時候，使用 docker container 來管理測試環境，確保一致跟穩定就非常重要。

- 定位問題
因為涉及多個系統與模組，難以定位失敗因素。使用 OpenTelemetry 如 Metric、Traing、Logging 的監控機制，就可以提升觀測性。

- 執行時間冗長
使用 PaaS 或 SaaS 服務可以大幅縮短測試時間。但也只有大量的 E2E 測試才有這個需求。
如： Microdoft Playwright Testing | Microsoft Azure

- 維護成本高
只要 UI 改變，測試腳本得不斷更新。須有專責人員維護、管理自動化測試腳本。


# 預期困難

![[Pasted image 20251229112413.png]]


# Playwright

## 初始化專案

```shell
# 專案目錄下執行
npm init playeright@latest

# 或是
yarn create playwright
```

接著會跑一些選項，選完後就會在專案中有初始設定跟基礎檔案建立：

```
playwright.config.ts # Test configuration  
package.json  
package-lock.json # Or yarn.lock / pnpm-lock.yaml  
tests/  
	example.spec.ts # Minimal example test
```

## VSCode Extension

- Playwright Helpers
- Playwright Snippets for VS Code
- Playwright Snippets UI for VS Code
- Playwright Test for VS Code

## 設定檔

`playwright.config.ts` 就是設定檔，裡面會有基礎設定，基本上都很夠用。

如果需要更改 device 可以到 github 上面找 deviceSource 來確定有哪些可以選。

## 測試腳本

基本上沒有更改的話，就會寫在 `test/example.spec.ts` 內。

更多的測試 API，可至官網文件查詢。

但其實太多 API，不可能一個一個慢慢查慢慢寫，寫以也可以透過以下方式來速成：

- codegen：透過錄製操作來生成腳本
- DOM 選取：會自動在 console 產生測試碼

## Command Line

![[Pasted image 20251229112502.png]]

![[Pasted image 20251229112519.png]]

- 驗證碼認證就是 MFA，可以只執行一次 MFA 驗證，然後透過 `--save-storage` 存起來，接著產出測試碼，未來都使用這個 storage 來通過 MFA 測試就好。
- E2E 測試是沒辦法抓第三方 API 資源的，所以可以從 Network tab 把 fetch 到的用 `--save-har` 資源存成 HAR 檔，用 `--load-har` 為給測試腳本，就可以解決這個問題。

![[Pasted image 20251229112538.png]]

## Hooks

![[Pasted image 20251229112555.png]]

## Page Object

![[Pasted image 20251229112613.png]]

## Selector

![[Pasted image 20251229112634.png]]

- 需重視 Accecibility，無障礙屬性在 playwright 來說可以是個很準確的定位點。
- 如果無障礙屬性無法配合，專業的前端團隊會用 `test-id` 來做定位點。

## Settings

![[Pasted image 20251229112652.png]]

![[Pasted image 20251229112708.png]]

## Visual Records

![[Pasted image 20251229112723.png]]

## More

![[Pasted image 20251229112740.png]]


# 遇到的問題

## 如何復用登入資訊

在自動化測試中，如果每個 Test Case 都要重走一遍登入，會導致：

1. **測試速度慢**：登入通常涉及多次網路請求。
2. **不穩定性增加**：登入頁面出問題會導致所有測試失敗。

### 步驟一：存放 auth 資訊

Playwright 提供了一個強大的 API：`storageState`。它的運作原理如下：

新增一個 auth.setup.js 檔：
1. 執行登入驗證
2. 透過 `page.context().storageState()` 抓取瀏覽器當前所有的身份驗證資訊
3. 將 auth 資訊存入 `auth.json`

```js
import { test as setup, expect } from '@playwright/test'
	setup('authenticate', async ({ page }) => {
		// 訪問登入頁
		await page.goto('http://192.168.26.124:6001/login')
		// 檢查是否已經自動重定向到登入後的頁面（說明已登入）
		const currentUrl = page.url()
		if (!currentUrl.includes('/login')) {
		// 已經登入，直接保存狀態
			await page.context()
				.storageState({ path: 'e2e_test/auth.json' })
			return
		}
		
		// 執行登入流程
		await page.getByRole('textbox', { name: '員工編號/帳號' })
			.fill('s0003')
		await page.getByRole('textbox', { name: '密碼' }).fill('amastek')
		await page.getByRole('button', { name: '登入' }).click()
		
		// 等待登入完成 - 等待確認文字出現
		await expect(page.getByText('哈囉, Ken')).toBeVisible()
		// 保存 cookie 和 local storage 到 auth.json 檔案
		await page.context().storageState({ path: 'e2e_test/auth.json' })
})
```

### 步驟二：指定共用 auth

在 `playwright.config.js` 中：

1. 設定共用的 storage 路徑 `storageState: 'e2e_test/auth.json'` 
2. 幫 Test Project 設定依賴項 `setup`。在啟動後續的測試（如 `chromium`, `firefox`）時，會自動將這些 Cookie 和 Local Storage 注入到全新的瀏覽器 Context 中。
3. 當測試腳本訪問頁面時，伺服器看到瀏覽器帶有正確的 Cookie，就會認定該用戶已登入，從而直接跳過登入頁面。

```js title:"playwright.config.js" hl:4-5,10-13,17,22,27
export default defineConfig({
testDir: './e2e_test',
use: {
	// 設定 storage 使用路徑
	storageState: 'e2e_test/auth.json',
	trace: 'on-first-retry'
},
/* Configure projects for major browsers */
projects: [
	{  // 將 setup 也放進要執行的測試範圍內
		name: 'setup',
		testMatch: /auth\.setup\.js/
	},
	{
		name: 'chromium',
		use: { ...devices['Desktop Chrome'] },
		dependencies: ['setup'] // 要求該測試執行前，先執行 setup
	},
	{
		name: 'firefox',
		use: { ...devices['Desktop Firefox'] },
		dependencies: ['setup'] // 要求該測試執行前，先執行 setup
	},
	{
		name: 'webkit',
		use: { ...devices['Desktop Safari'] },
		dependencies: ['setup'] // 要求該測試執行前，先執行 setup
	}
})
```


## 測試資料如何穩定

在測新增使用者、新增權限時，會遇到資料重複而無法完成測試的問題，但又不希望一直手動更改測試資料。

- 可以資料後面加上「時間戳記」或「隨機字串」。這樣每次執行的資料都是唯一的，永遠不會重複。
- 使用 JavaScript Plugins，例如 [Faker.js](https://fakerjs.dev/)。

## 使用 Faker.js 

安裝套件

```shell
yarn add @faker-js/faker --dev
```

產 faker data：

```js
import { faker } from '@faker-js/faker';
// 或是使用繁體中文語系
import { fakerZH_TW as faker } from '@faker-js/faker';

// 產生符合預期格式的帳號
const randomId = faker.string.numeric(4, { allowLeadingZeros: true })
const uniqueAccount = `s${randomId}`

 // 產生隨機中文姓名與權限名稱
 const uniqueName = faker.person.fullName()
 const uniqueRole = `權限組_${faker.commerce.productAdjective()}${randomId}`
```

應用在 Playwright 內：

```js hl:3,11-16,23,29,31-32,20-21
// @ts-check
import { test, expect } from '@playwright/test'
import { fakerZH_TW as faker } from '@faker-js/faker';

test.describe('使用者管理功能', () => {
	// 每個測試執行前，都會自動跑這段
	test.beforeEach(async ({ page }) => {
		await page.goto('http://192.168.26.124:6001/setting/users')
	})
	test('新增使用者', async ({ page }) => {
		// 產生符合預期格式的帳號
		const randomId = faker.string.numeric(4
			, { allowLeadingZeros: true })
		const uniqueAccount = `s${randomId}`
		// 產生隨機中文姓名
		const uniqueName = faker.person.fullName()
		
		// 新增使用者
		await page.getByRole('link', { name: '新增使用者' }).click()
		await page.getByRole('textbox', { name: '帳號 *' })
			.fill(uniqueAccount)
		await page.getByRole('textbox', { name: '密碼 *' }).fill('amastek')
		await page.getByRole('textbox', { name: '姓名 *' }).fill(uniqueName)
		await page.getByRole('combobox', { name: '權限 * 請選擇權限' })
			.click()
		await page.getByText('系統管理').click()
		await page.getByRole('button', { name: '送出' }).click()
		// 搜尋是否有剛剛新增的使用者
		await page.getByRole('textbox', { name: '姓名' }).fill(uniqueName)
		await page.getByRole('button', { name: '查詢' }).click()
		await expect(page.getByRole('gridcell', { name: uniqueAccount }))
			.toBeVisible()
	})
})
```

### 文件支援

- [Localization](https://fakerjs.dev/guide/localization.html)
- [API](https://fakerjs.dev/api/)




