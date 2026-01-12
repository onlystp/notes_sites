
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
$ npm init playeright@latest
```

接著會跑一些選項，選完後就會在專案中有初始設定跟基礎檔案建立

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











