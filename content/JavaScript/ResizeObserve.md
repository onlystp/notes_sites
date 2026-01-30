
ResizeObserver API 是 JS 提供可以根據元素的大小來調整 layout 或觸發特定功能。

### 語法

傳入Callback Function，當目標元素大小改變時，瀏覽器會執行這個函數。

```js
const observer = new ResizeObserver((entries) => {
  for (let entry of entries) {
    // entry.contentRect 包含元素的尺寸資訊
    const { width, height } = entry.contentRect;
    console.log(`目前寬度：${width}px，高度：${height}px`);
  }
});
```

指定要監聽的元素

```js
const myBox = document.querySelector('#target-box');
observer.observe(myBox);
```


停止觀察

```js
// 停止觀察特定元素
observer.unobserve(myBox);

// 完全停止觀察器
observer.disconnect();
```

---

### 情境範例

當側邊欄寬度小於 300px 時，將其背景變為紅色，並調整字體大小。

```js
function setSidebarColor (searchSection) {
	const sidebar = document.querySelector('.sidebar');
	
	const ro = new ResizeObserver(entries => {
	  for (let entry of entries) {
	    const currentWidth = entry.contentRect.width;
	
	    if (currentWidth < 300) {
	      entry.target.classList.add('mobile-mode');
	      entry.target.style.backgroundColor = '#ff4d4d'; // 變為紅色
	    } else {
	      entry.target.classList.remove('mobile-mode');
	      entry.target.style.backgroundColor = '#f0f0f0'; // 回復灰白色
	    }
	  }
	});
	
	// 啟動監聽
	ro.observe(sidebar);
	
	return () => ro.disconnect()
}
```

---

### 結語：優勢總結

- **效能更佳**：相比於舊式的 `onresize`，`ResizeObserver` 是基於瀏覽器內建機制的非同步觀察，效能開銷較低。
    
- **精準度高**：即使視窗沒有縮放，只要元素本身的大小發生變化（如動態增減內容），它都能準確捕捉。
    
- **避免佈局震盪**：它在佈局繪製後觸發，能有效減少迴圈觸發佈局更新（Layout Thrashing）的風險。