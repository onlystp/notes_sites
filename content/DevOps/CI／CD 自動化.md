
# 什麼是 CI/CD

![[Pasted image 20260130164220.png]]

- CI（Continuous Integration）持續整合：開發-> 編譯 -> 跑測試
- CD（Continuous Delivery）持續交付：發佈 -> 部署至測試環境、產品

以上流程少不了重複的動作，而 CI/CD 自動化，就是寫 pipeline，透過工具（比方 Github Actions、Gitlab CI/CD）自動執行這些流程。

## 建立 workflow

### 創建 CI workflow

在專案資料夾下的 `.github/` 檔案夾建 `workflow/`，新建一個 workflow 叫 `test.yml`

```tree
repo/
  - .github/
    - workflow/
	      - test.yml
```

定義這個處理測試環節的 workflow 要處理什麼：

```yml title:"test.yml"
name: "Test" #命名

on: # 定義何時觸發
  push:
    branches: # 定義哪個分支才觸發
      - main
  pull_request:
    branches:
      - main
    types: [ opened ] # PR 有分階段，所以清楚定義是開 PR 的時候觸發

jobs: # 定義觸發後要跑什麼工作
  test:  # 指定要跑測試
    runs-on: ubuntu-latest  # 指定在 ubantu 環境下跑
    # 設定在什麼容器下跑，這樣就不用在虛擬機上安裝
    container: astral/uv:python3.12-bookworm-slim
    steps: # 可以想像在本地要怎麼跑起來
      - name: Checkout # 幫步驟取名（可以不寫）
        uses: actions/checkout@v6 # 使用封裝好的 workflow，用來獲取程式碼（來自 github market）
      - name: Install Dependency
        run: uv sync  # 安裝依賴，隨項目使用（這是 python 的）
      - name: Test
        run: uv run pytest tests/  # 跑測試的指令
```

- `on` 觸發事件可以定義多個事件，只要滿足其中一個，就會觸發。可見更多 [trigger events](https://docs.github.com/en/actions/reference/workflows-and-actions/events-that-trigger-workflows)
- `jobs` 可以定義多個行為，是同時跑起來的，而不是按順序

### 創建 CD workflow

新建一個 workflow 叫 `deploy.yml

```tree
repo/
  - .github/
    - workflow/
	      - deploy.yml
```

定義這個處理部署環節的 workflow 要處理什麼：

```yml title:"deploy.yml"
name: "Deploy" #命名

on: # 定義何時觸發
  push:
    tag: # 定義有壓 tag 的才觸發
      - v*  # 只觸發 tag 是 v（有版本號）的才觸發

jobs: # 定義觸發後要跑什麼工作
  Deploy:  # 指定要跑測試
    runs-on: ubuntu-latest  # 指定在 ubantu 環境下跑
    steps:
      - name: Checkout
        uses: actions/checkout@v6 # 獲取程式碼
      - name: Upload
        run: appleboy/scp-action@v1 # 來自 Github market
        with:
            # 以下為敏感訊息
	        host: ${{ secrets.SERVER_HOST }}
	        username: ${{ secrets.SERVER_USERNAME }}
	        key: ${{ secrets.SERVER_KEY }}
	        source: "./"  # 從現在的位置
	        target: "/home/user/project"  # 搬到服務器上的位置
      - name: Test
        run: uv run pytest tests/  # 跑測試的指令
      - name: Start service
        uses: appleboy/ssh-action@v1
        with:
	        host: ${{ secrets.SERVER_HOST }}
	        username: ${{ secrets.SERVER_USERNAME }}
	        key: ${{ secrets.SERVER_KEY }}
	        script:
		        cd /home/user/project
		        python main.py
```

#### Github Secret 環境變數

部署環節需要用到的敏感訊息，可以透過 Github Secret 去設定環境變數，這樣才不會直接把需要保密的訊息暴露在 yml 檔內：

![[Pasted image 20260130175042.png]]

## 推上 Github Action

接著，就把剛剛建的 workflow 推到 Github Repo 上同步，到 Github 的 Actions 頁面查看，就可以看到在運行了。

![[Pasted image 20260130174040.png]]