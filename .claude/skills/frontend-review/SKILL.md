---
name: frontend-review
description: Review 前端程式碼改動時使用（React、Vue、Svelte、TypeScript、CSS、HTML 等）。找出會讓使用者看到錯誤畫面、資料出錯、被攻擊或用不了的問題，並排好優先順序。
---

讀者：寫這段程式的工程師。他懂前端，只想知道哪裡會壞、為什麼、怎麼修。

回覆用繁體中文寫。檔名、指令、程式碼裡的名稱和技術名詞（state、effect、hydration、re-render 等）保持英文。

## 步驟

1. 決定 review 範圍：
   - 使用者有給 PR 編號、branch 或路徑，就用那個。
   - 沒給的話，看 `git diff` 加上 `git diff --staged`；兩個都是空的，就跟 main branch 比（`git diff <base>...HEAD`）。
2. 看 `package.json` 和設定檔（`tsconfig.json`、`vite.config.*`、`next.config.*`、`.eslintrc*` 等），確認用的框架、版本和 lint 規則。之後的判斷都要符合這個專案的寫法，不要套別的框架的習慣。
3. 讀每個改動的檔案的完整內容，不是只讀 diff。改到的 component 如果被別的地方用到，或是用到改過的 hook / store / util，也要去讀。
4. 照下面的清單找問題。
5. 每個問題都要回頭確認：真的會發生嗎？在什麼情況下？找不到具體的觸發情境就不要報。
6. 專案有 `typecheck`、`lint`、`test` 這類 script 的話就跑，把失敗的結果一起放進報告。沒有 `node_modules` 就不要自己 `npm install`（會下載套件，要先問使用者）。跑不了就說跑不了，不要假裝跑過。

## 要找什麼

照優先順序：

**會壞掉的**
- 非同步 race condition：請求回來的順序不對、component 已經 unmount 還在 setState、沒取消舊的 request。
- Stale closure：effect、callback、timer 裡用到舊的 state 或 props。
- Effect / watch 的 dependency 漏寫或多寫，造成沒更新或無限迴圈。
- 直接改 state 或 props（mutation），畫面不會更新。
- List 的 `key` 用 index 或不穩定的值，在排序、刪除時會錯位。
- SSR / hydration：在 render 階段用到 `window`、`document`、`localStorage`、`Date.now()`、亂數，造成 server 和 client 結果不同。
- 沒處理 loading、error、空資料的狀態；`undefined` / `null` 沒檢查就取值。
- 表單：重複送出、送出後沒重設、驗證只做在前端。

**安全**
- XSS：`dangerouslySetInnerHTML`、`v-html`、`innerHTML`、`{@html}` 放了使用者能控制的內容；`href` / `src` 沒擋 `javascript:`。
- Token 或敏感資料存在 `localStorage`、寫進 log、放進 URL。
- 把 secret 放進會打包到前端的環境變數（`NEXT_PUBLIC_`、`VITE_`、`REACT_APP_` 開頭的都會公開）。
- `target="_blank"` 連到外部網站沒加 `rel="noopener noreferrer"`（舊瀏覽器才有問題，優先度低）。

**無障礙（a11y）**
- 可以點的東西用 `div` / `span` 做，沒有 `button` 或沒有鍵盤操作。
- 圖片沒 `alt`、icon button 沒有 `aria-label`、表單欄位沒有 `label`。
- Modal / dropdown 沒處理 focus（打開時沒移進去、關掉沒移回來、Esc 關不掉）。
- 只用顏色表達狀態，或文字對比太低。

**效能**（只報會被使用者感覺到的）
- 每打一個字就送一次 request，沒有 debounce；輸入是空的也照送。
- 大 list 沒有 virtualization、每次 render 都重算很重的東西。
- 傳給 memo 過的 component 的 props 每次都是新物件 / 新函式，讓 memo 失效。
- 整包 import 很大的 library（例如 `import _ from 'lodash'`），或該 lazy load 的頁面沒 lazy load。
- 圖片沒設尺寸造成畫面跳動（layout shift）、沒壓縮、沒 lazy load。
- Event listener、interval、subscription 沒在 cleanup 移除，造成 memory leak。

**樣式和版面**
- 寫死寬高，在手機或長文字（翻譯後變長）時會爆版。
- `z-index` 隨便給大數字、`!important`、會影響到別的 component 的全域 CSS。
- 沒照專案現有的 design token / theme 變數，直接寫死顏色或間距。

**維護性**（最多報 3 個，挑最重要的）
- 重複了專案裡已經有的 component、hook 或 util。
- Component 太大、做太多事，很難測。
- TypeScript 用 `any`、`as` 硬轉型、`!` 跳過檢查，把真正的錯誤藏起來。

## 不要報的

- Lint 或 formatter 會自動處理的東西（空白、分號、import 順序）。
- 個人偏好，例如「我會用另一種寫法」，但現在的寫法沒有問題。
- 沒改到的舊程式碼的問題，除非這次的改動讓它變得更嚴重。

## 輸出格式

先用一句話講結論：可以合併／修完幾個問題再合併／需要大改。

然後依嚴重程度列出問題，每個問題這樣寫：

```
### [嚴重度] 一句話講問題
`path/to/file.tsx:42`

會發生什麼：使用者或系統會遇到什麼，在什麼情況下觸發。
怎麼修：具體的改法，需要時附一小段程式碼。
```

同一種問題出現在好幾個地方（例如三個 `div` 都沒有鍵盤操作），合成一條，把每個位置都列出來。

嚴重度分三級：
- **必修**：會造成錯誤、資料出錯、安全問題，或某些使用者完全用不了。
- **建議修**：在某些情況下會出問題，或明顯讓體驗變差。
- **可以考慮**：不修也不會壞，但修了比較好。最多列 3 個。

最後如果有跑 typecheck / lint / test，附上結果（一行一個指令，寫通過或失敗，失敗的話貼錯誤重點）。

沒找到問題就直接說沒找到，不要為了湊數硬找。
