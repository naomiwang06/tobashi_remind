# Firebase 安全性檢查報告 — tobashi_remind

專案：Firebase `care-tobashi`，使用 **Authentication（Google 登入）+ Realtime Database**。
- `index.html`：照護提醒 App，共用資料在 `care_tobashi/shared`（患者與照顧者各自用 Google 帳號登入共用同一份）。
- `game/index.html`：土撥鼠任務大冒險，雲端存檔在 `dig_quest/saves/<uid>`。
- 沒有使用 Firestore、Cloud Storage（`storageBucket` 只出現在設定裡）。

**repo 內原本沒有任何 rules 檔**，我無法登入 Console，所以線上規則是從程式行為反推的（見風險 1）。

## 一、風險清單（嚴重度由高到低）

| # | 嚴重度 | 問題 | 狀態 |
|---|---|---|---|
| 1 | 🔴 嚴重（未能確認） | 線上 rules 不在版控內、未知。App 的「登入」只是前端畫面：常見設定 `auth != null`（**任何** Google 帳號登入後都能讀寫 `care_tobashi/shared` 與別人的 `dig_quest/saves/*`），或測試模式 `true` / 到期日。`shared` 內含**健康提醒、待辦、行事曆、接案收入（vault）**等個人資料。遊戲裡「規則擋住了，請奈奈開通 dig_quest 存檔規則」的提示也顯示規則曾被手動改過。 | ✅ 提供 `database.rules.json`（email 白名單 + 驗證）；**需你手動填 Email 並部署** |
| 2 | 🔴 高 | 任何擁有 Google 帳號的人都能完成登入流程；前端沒有（也無法有）授權檢查。 | ✅ 由 rules 以 `email_verified` + Email 白名單強制 |
| 3 | 🟠 高 | 遊戲存檔路徑 `dig_quest/saves/<uid>` 若規則只檢查 `auth != null`，使用者可讀寫他人存檔、塞入任意大小資料。 | ✅ rules：只允許本人 uid + 白名單；欄位白名單；`json` < 2MB |
| 4 | 🟠 中 | **XSS**：`index.html` 把雲端資料（標題、名稱、截止日、圖示…）直接塞進 `innerHTML` 未跳脫，id 也直接放進 inline `onclick`。能寫入共用資料的人（或被盜的帳號）可在對方手機執行腳本。 | ✅ 35 處輸出加 `esc()`；`sanitizeState` 會把 id 限制為 `[A-Za-z0-9_-]` |
| 5 | 🟡 中 | 規則收緊後，未授權帳號登入時 `get()` 丟出 permission denied，畫面卡在「資料載入中」。 | ✅ 改為顯示「此帳號沒有權限」+ 登出按鈕 |
| 6 | 🟡 低 | Web API key 在前端（正常，非機密），但若沒有限制來源，可被他人拿去濫用 Identity Toolkit 配額。 | ⚠️ 需到 Google Cloud Console 手動限制 |
| 7 | 🟡 低 | Firestore / Storage 未使用，但若曾以測試模式啟用仍可能公開（`storageBucket` 有設定）。 | ✅ 提供全拒絕的 `firestore.rules`、`storage.rules`；請到 Console 確認 |
| 8 | ℹ️ 資訊 | `game/index.html` 以 `textContent`/DOM API 建立畫面，未發現 HTML 注入問題。Firebase SDK 從 gstatic 載入且鎖版本 10.12.2。無 CSP。 | 未改（CSP 可能擋到 Google 登入彈窗，無法在此驗證） |

## 二、我改了什麼

### 新增檔案
- `database.rules.json`
  - 預設全部拒絕。
  - `care_tobashi/shared`：僅 Email 在白名單且 `email_verified` 的已登入者可讀寫；寫入內容必須是物件。
  - `dig_quest/saves/$uid`：白名單 + `auth.uid === $uid`；只允許 `json`（字串 < 2,000,000 字）、`savedAt`（數字）、`v`（數字）。
- `firestore.rules`、`storage.rules`：全拒絕。
- `firebase.json`：部署設定。

### `index.html`
- `esc()` 輸出跳脫（35 處）、`sanitizeIds()`（見風險 4）。
- 登入後讀取失敗（無權限/網路）的友善畫面與登出按鈕（風險 5）。
- 原本的登入、即時同步、儲存流程**沒有改動**。

### `game/index.html`
未修改——現有程式本來就只讀寫 `dig_quest/saves/<自己的 uid>`，與新規則完全相容。

### 驗證
兩支 inline script 通過 `node --check` 語法檢查。**沒有實際連 Firebase 測試**，也沒有用 Emulator 驗證 rules（本環境沒有 firebase CLI / 連不到 Firebase）。部署前請用 Console「規則遊樂場」或 Emulator 測試。

## 三、需要你手動處理

1. **先備份 + 看現況**：Console → Realtime Database → 匯出 JSON；記下目前的 Rules（有沒有 `true`、到期日、`auth != null`）。
2. **編輯 `database.rules.json` 的白名單**：裡面有 3 個 Email——`naomiwang0619@gmail.com`（我用你的帳號當管理者）以及兩個 `REPLACE_ME_*@example.invalid` 佔位符。**請換成患者與照顧者實際登入用的 Google Email**（遊戲玩家的 Email 也要在內）。
   ⚠️ 沒換之前，那兩位會被擋在外面（顯示「沒有存取權限」）。這是刻意的「失敗即關閉」。若白名單的人不只 3 位，複製 `auth.token.email === '...'` 那段並用 `||` 串起來（4 處：shared 的 read/write、saves 的 read/write）。
3. **部署規則**：`firebase use care-tobashi` → `firebase deploy --only database,firestore,storage`，或貼進 Console 的 Rules 頁籤。部署後用每個人的帳號各登入一次確認；再用一個**非白名單**的 Google 帳號確認會看到「沒有存取權限」。
4. **限制 API key**：Google Cloud Console → API 和服務 → 憑證 → 找到 key `AIzaSyCk…`（設定在 `index.html` 與 `game/index.html`）→ 應用程式限制選「HTTP 參照網址」，填你的網站網域；API 限制只留 Identity Toolkit API、Token Service API、Firebase Realtime Database 相關。
5. **Authentication 設定**：Sign-in method 只保留 Google；Settings → Authorized domains 移除用不到的網域（含不需要的 `localhost`）。
6. **Console 檢查**：Firestore Database / Storage 若從未使用，確認「未建立」或已套用全拒絕規則。
7. （建議）啟用 App Check；設定預算/用量警示。
8. 若曾有不明帳號登入過，請到 Authentication → Users 檢視，並檢查 `care_tobashi/shared` 是否被改動。

## 四、遇到不確定時採取的「最安全」決定
- 線上規則看不到 → 假設是寬鬆的，直接改成預設拒絕 + Email 白名單 + `email_verified`。
- 不知道患者/照顧者的 Email → 佔位符用不可能存在的 `.invalid` 網域（失敗即關閉），不使用「任何已登入者」的寬鬆規則。
- 遊戲存檔也套用白名單（比單純 uid 隔離更嚴格）；如果玩家不在名單內會顯示既有的「規則擋住了」提示，本地存檔不受影響。
- 因為規則需要你填 Email 才能部署，**合併此 PR 本身不會改變線上行為**；真正生效在你部署 rules 的那一刻。
