# 介紹

`git cherry-pick` 的用途是：<span style="color:rgb(255, 0, 0)">從其他分支挑選特定的 Commit，並應用到當前分支</span>。這在修復緊急 Bug 或從開發分支提取特定功能時非常有用。

# 指令

| **動作**           | **指令**                             |
| ---------------- | ---------------------------------- |
| 挑選單一 commit      | `git cherry-pick <commit-hash>`    |
| 挑選多個連續 commits   | `git cherry-pick <A>..<B>` (不包含 A) |
| 挑選包含 A 到 B       | `git cherry-pick <A>^..<B>` (包含 A) |
| 挑選不連續的多個 commits | `git cherry-pick <hash1> <hash2>`  |

# 常用參數

- **`-n`, `--no-commit`**
    
    - 用途：只將該 commit 的變更放到 Staging，不自動產生 Commit。
    - 情境：當你想把多個 Commit 放到目標分支上，再自己一起將它們合併成一個新的 Commit 時。
        
- **`-e`, `--edit`**
    
    - 用途：在 Commit 之前跳出編輯器，編輯 commit message。
        
- **`-x`**
    
    - 用途：在新的 commit message 結尾加上 `(cherry picked from commit ...)`。
    - 情境：方便追蹤程式碼的來源。
        
- **`-s`, `--signoff`**
    
    - 用途：加上簽署資訊。
        

# Conflicts

Cherry-pick 是合併程式碼，因此很可能會遇到衝突。就跟一般解衝突一樣：

1. **手動解決衝突**：打開衝突檔案，修正內容。
2. **加入暫存區**：執行 `git add <file-name>`。
3. **繼續流程**：執行 `git cherry-pick --continue`。
    

> 如果想放棄：
> 
> 執行 `git cherry-pick --abort`，分支會回到執行指令前的狀態。

