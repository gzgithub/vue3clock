# Vue3 Clock

以 Vue 3 + Vite 製作的時鐘網頁，顯示現在時間，針對手機畫面最佳化。

## 功能

- 大字數位時間（時:分 + 秒），每秒對齊系統時間更新
- 類比錶盤（SVG），可隱藏只看數字
- 24 / 12 小時制切換（12 小時制顯示「上午／下午」）
- 中文日期與星期、目前時區
- 螢幕常亮（Screen Wake Lock API，支援的瀏覽器才會顯示按鈕）
- 全螢幕模式
- 自動深色／淺色主題（跟隨系統設定）
- 手機直向／橫向版面、瀏海與安全區域（safe-area）適配
- 設定記憶於 localStorage

## 開發

```bash
npm install
npm run dev      # 啟動開發伺服器（--host，可用手機透過區網連線測試）
npm run build    # 輸出到 dist/
npm run preview  # 預覽打包結果
```

`vite.config.js` 設定了 `base: './'`，`dist/` 可直接部署到 GitHub Pages 或任何靜態主機的子路徑。

## 部署（GitHub Pages）

推送到 `main` 時，`.github/workflows/deploy.yml` 會自動建置並發佈到 GitHub Pages：

1. 到 repo 的 **Settings → Pages**，將 **Source** 設為 **GitHub Actions**（只需設定一次）。
2. 合併到 `main` 後，網址為 <https://gzgithub.github.io/vue3clock/>。
