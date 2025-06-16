+++
author = "Hugo Authors"
title = "關於XSS的ㄋㄚˋ些事"
date = "2024-05-28T15:04:05Z"
description = "A brief description of Hugo Shortcodes"
tags = [
    "XSS", "資訊安全"
]
+++
# 深入理解 Cross-Site Scripting (XSS) 攻擊與防禦

Cross-Site Scripting (XSS)，即跨網站指令碼，是前端開發中常見且危險的資安漏洞。它的核心概念是讓攻擊者能夠在其他使用者（客戶端）的瀏覽器上執行惡意的 JavaScript 或其他指令碼，進而竊取敏感資訊、劫持使用者會話，甚至操控網頁行為。

XSS 攻擊之所以發生，通常是因為網站未能妥善處理來自使用者的輸入，導致輸入的內容被瀏覽器誤認為是可執行的程式碼，而非單純的資料。常見的輸入點包括表單欄位（如搜尋框、留言區、使用者名稱）、URL 參數等。攻擊者可能會輸入類似 `<script>alert('XSS')</script>` 或 `<img src onerror="alert('XSS')">` 這樣的代碼來測試或發動攻擊。

根據惡意代碼 (payload) 的來源和其在伺服器上的存儲方式，XSS 可以分為幾種類型：

## XSS 的分類 (依據 Payload 的來源/儲存)

1.  **Reflected XSS (反射型 XSS)：**
    這種攻擊的 payload 不會被永久儲存到伺服器的資料庫中。攻擊者通常會將惡意代碼嵌入到 URL 參數中，然後誘騙受害者點擊這個包含 payload 的惡意連結。當受害者的瀏覽器向伺服器發出請求時，伺服器未經處理地將包含 payload 的內容反射回受害者的瀏覽器，導致 payload 在受害者的瀏覽器中執行。它的生命週期短暫，只在使用者訪問帶有惡意參數的 URL 時觸發一次。

2.  **Stored XSS (儲存型 XSS)：**
    這種攻擊是最具破壞力的類型之一。攻擊者成功地將惡意 payload 永久地儲存到目標網站的資料庫中（例如在論壇發帖、留言或修改使用者資料時注入）。當其他使用者瀏覽包含此 payload 的網頁時，伺服器會將儲存的惡意代碼連同正常內容一起發送給客戶端瀏覽器，導致 payload 在每個訪問該頁面的使用者瀏覽器中執行。危害範圍廣，且不需要額外的使用者互動（如點擊連結）。

3.  **DOM-based XSS (基於 DOM 的 XSS)：**
    與前兩種主要依賴伺服器端處理不同，DOM-based XSS 漏洞存在於前端的 JavaScript 代碼中。惡意 payload 可能來自 URL 的片段識別符號（# 後面的部分）或其他客戶端可控的來源。前端 JavaScript 使用這些受控的資料不安全地操作文件物件模型 (DOM)，例如將資料寫入 `innerHTML` 或使用 `eval()` 函數，導致 payload 被瀏覽器作為程式碼執行。這種攻擊完全在客戶端進行，伺服器端可能完全不受影響。

## 常見的 XSS Payload 目標

攻擊者注入惡意 JavaScript 的目的多種多樣，常見的 payload 功能包括：

* `Workspace()` 或 `XMLHttpRequest`: 將使用者的敏感資料（如 Cookies）發送到攻擊者控制的伺服器。
* `document.cookie`: 竊取使用者的 Session Cookies，可能導致攻擊者劫持使用者會話。
* `html2canvas` (或其他截圖庫): 對使用者正在瀏覽的頁面進行截圖，可能捕獲敏感視覺資訊。
* Keylogger: 記錄使用者在頁面上的鍵盤輸入，竊取密碼、信用卡號等。
* `alert()`: 最簡單的 PoC (Proof of Concept)，用於驗證 XSS 漏洞是否存在。
* 重新導向使用者到惡意網站、修改網頁內容進行釣魚等。

## XSS 的進階概念

* **XSS Worm (XSS 蠕蟲)：**
    一種更進階的 XSS 攻擊形式。惡意 JavaScript 不僅攻擊當前使用者，還會利用當前使用者的權限（如社交網路發帖權限）將 XSS payload 注入到他們能影響到的內容中，進而感染他們的聯繫人或粉絲。這導致 payload 像病毒一樣在使用者之間互相傳播，造成大規模感染。

* **Blind XSS (盲打型 XSS)：**
    攻擊者注入 payload 後，不會立即看到效果，也不知道 payload 何時何地會被觸發。payload 被儲存在網站的某個位置，可能是在後台管理系統、日誌查看器、客戶服務面板等只有網站內部人員會訪問的地方。當內部人員訪問這些頁面時，payload 被執行，並通常會“打回來”通知攻擊者（例如向攻擊者發送一個包含執行環境資訊的請求）。攻擊者是「盲打」，等待 payload 在一個未知的時機和地點被「看到」。

* **Self-XSS (自體 XSS)：**
    這是一種需要使用者自己手動執行惡意代碼的 XSS。通常攻擊者會欺騙使用者在瀏覽器的開發者控制台中貼上並執行一段 JavaScript 代碼。這種攻擊通常不能強加給使用者，主要依賴於社會工程學來誘導使用者自己攻擊自己，對其他使用者不構成直接威脅，但仍可能導致帳戶被盜用。

## XSS 的防禦策略

防禦 XSS 是一個需要多層次方法的課題：

1.  **輸入驗證 (Input Validation) 與輸出編碼 (Output Encoding)：**
    這是最基本也是最重要的防禦。
    * **輸入驗證：** 檢查使用者輸入的內容是否符合預期的格式和類型。例如，如果只允許數字，則拒絕包含其他字元的輸入。這有助於過濾掉明顯不符合規範的輸入。
    * **輸出編碼/轉義 (Encoding/Escaping)：** 當需要在 HTML 頁面中顯示使用者輸入的內容時，必須對所有輸出進行嚴格的編碼或轉義。將具有特殊意義的 HTML 字元（如 `<`, `>`, `"`, `'`, `&`, `/` 等）轉換為它們的 HTML 實體（如 `&lt;`, `&gt;` 等）。這樣瀏覽器就會把這些內容當作普通文本顯示，而不會解析為 HTML 標籤或 JavaScript 代碼。這是防禦 Reflected 和 Stored XSS 的首要措施。

2.  **HTML 過濾 (HTML Sanitization)：**
    當網站允許使用者輸入包含 HTML 標籤的富文本內容時（例如評論區的粗體、斜體格式），不能簡單地全部編碼。這時需要使用 HTML 過濾器。過濾器的作用是移除或修改輸入 HTML 中潛在的危險元素（如 `<script>`, `<iframe>`, `<svg>` 標籤）和危險屬性（如 `onerror`, `onload`, `href` 中的 `javascript:` 偽協議等），只保留安全的 HTML 標籤和屬性。這是一個複雜的任務，很難手動完美處理所有情況。
    * **DOMPurify：** 是一個被廣泛使用的、強健的 HTML 過濾庫，它在瀏覽器端運行，能有效淨化 HTML 片段。但即使是 DOMPurify，不正確的使用方式也可能引入漏洞，需要仔細閱讀其文檔。

3.  **內容安全政策 (Content Security Policy, CSP)：**
    CSP 是一種由瀏覽器強制執行的安全層。通過在 HTTP 頭部或 HTML 的 `<meta>` 標籤中定義一系列策略，網站可以告訴瀏覽器允許從哪些來源載入哪些類型的資源（如腳本、樣式表、圖片等）。這本質上是一個白名單機制。
    CSP 並不能完全阻止 XSS 注入，但它可以極大地限制被注入腳本的執行能力，例如禁止內聯腳本 (`<script>... </script>`)、禁止從未知來源加載外部腳本、限制表單提交的目標等。它是作為**縱深防禦**的重要一環，即使其他防禦措施失效，CSP 也能降低攻擊的影響。
    常見的 CSP 指令 (Directive) 包括：
    * `default-src`: 所有資源的預設策略。
    * `script-src`: 規定腳本的允許來源。
    * `img-src`: 規定圖片的允許來源。
    * `connect-src`: 規定 XMLHttpRequest, WebSockets 等連線的允許目標。
    * `style-src`: 規定樣式表的允許來源。
    * `Maps-to`: 限制頁面導航和表單提交的目標 URL。

## CSP 的繞過 (CSP Bypassing)

儘管 CSP 提供了強大的保護，但配置不當的策略仍然可能被繞過。例如，如果 CSP 的 `script-src` 策略允許從一個包含 JSONP 端點的域名加載腳本，攻擊者可能可以利用 JSONP 的回調機制來執行任意 JavaScript，即使網站本身沒有直接的 XSS 漏洞。這提醒我們 CSP 配置需要非常謹慎，且不能僅依賴 CSP。

## JavaScript 偽協議 (javascript: pseudo-protocol)

`javascript:` 偽協議允許在 URL 或某些 HTML 屬性（如 `<a href="...">`, `<img src="...">`）中使用 JavaScript 代碼。例如 `<a href="javascript:alert(1)">點我</a>`。如果不對使用者輸入的 URL 或屬性值進行驗證和過濾，攻擊者可以注入這樣的連結來觸發 JavaScript 執行，這也是 XSS 的一種表現形式。

## 總結

Cross-Site Scripting (XSS) 是一個嚴重的 Web 安全問題，它可以讓攻擊者在其他使用者的瀏覽器上執行惡意程式碼。理解 Reflected, Stored, 和 DOM-based XSS 的區別對於識別和防禦漏洞至關重要。防禦 XSS 需要一個多層次的方法：優先對所有使用者輸出進行**嚴格的編碼/轉義**；在允許富文本的場景下使用**安全的 HTML 過濾庫 (如 DOMPurify)**；並配置**強大的 Content Security Policy (CSP)** 作為最後一道防線。始終記住，對使用者輸入保持警惕，並在輸出時假設它是不可信的，是防止 XSS 的關鍵原則。