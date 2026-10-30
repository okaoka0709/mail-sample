# Roundmail · Vue 3 郵件 Demo

純前端郵件介面，提供附件雛型所示的「經典」與「清新」兩套風格。一鍵切換保留目前的信件、資料夾與搜尋條件。

## 本機啟動

需要 Node.js 18+（建議 22）與 npm。

```sh
npm install
npm run dev
```

開啟終端顯示的本機網址。請透過 HTTP 網址使用，直接雙擊 `index.html` 無法載入 Vue 與 JSON。

## 功能

- 三欄桌面配置；手機使用資料夾選單與清單／閱讀切換。
- 資料夾、全文搜尋、星號、已讀切換、勾選與批次操作。
- 移至垃圾桶、還原至原資料夾、上一封／下一封。
- 撰寫、回覆、轉寄、儲存／編輯草稿與模擬寄送。
- 兩份可下載的文字附件。
- 風格與信件狀態存於 `localStorage`；「重設 Demo」恢復原始信件並保留風格。

寄送僅模擬加入寄件備份，不會寄出真實郵件。關閉撰寫視窗會放棄尚未儲存的編輯。

## 修改示範資料

`public/data/mail.json` 包含 12 封虛構信件，由瀏覽器透過 `fetch` 讀取。欄位：

| 欄位 | 用途 |
| --- | --- |
| `id` | 唯一字串 |
| `folder` | `inbox`、`drafts`、`sent`、`spam`、`trash` |
| `from` | 寄件者 `{ "name": "姓名", "email": "name@example.com" }` |
| `to` | 單一收件者 Email |
| `subject`、`body` | 主旨與純文字內文；內文以 `\n` 換行 |
| `date` | 包含時區的 ISO 日期字串 |
| `read`、`starred` | 已讀與星號布林值 |
| `attachments` | `{ "name", "url", "size" }` 陣列；無附件使用 `[]` |
| `banner` | 選填，電子報標題 |
| `previousFolder` | 選填，移至垃圾桶前的資料夾 |

摘要由內文截取。內文以純文字呈現，不執行 HTML。附件放在 `public/attachments/`，URL 使用 `./attachments/檔名`。修改 JSON 後按「重設 Demo」，才會覆蓋此瀏覽器已儲存的信件。所有圖片式裝飾與圖示均為本地 CSS／SVG，不依賴外部圖片或字型。

## 建置與靜態部署

```sh
npm run build
npm run preview
```

將 `dist/` 內容上傳至任意靜態網站空間。已使用相對基底路徑，支援放在子目錄。無後端、登入或資料庫。

## 程式結構

- `src/App.vue`：畫面、信件狀態與必要操作。
- `src/components/Icon.vue`：共用 SVG 圖示。
- `src/style.css`：共用排版、雙風格與響應式樣式。
- `public/data/mail.json`：原始示範資料。
