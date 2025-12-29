## 🛡️ 滲透測試學習筆記：網頁列舉 (Web Enumeration)
### 0. 核心心法：偵查 (Reconnaissance)
在嘗試進入系統前，必須盡可能地收集目標的技術細節與隱藏路徑。網頁列舉的目標是發現那些被遺忘的檔案、沒鎖好的門或開發者的疏忽。

#### 1. 基礎指紋識別 (Fingerprinting)
了解網站是用什麼技術架構搭建的，這能幫助我們縮小攻擊範圍。

- Whatweb: 最推薦的快速識別工具。

  - 用途: 自動識別 CMS（如 WordPress）、網頁伺服器（Apache/Nginx）、後端語言（PHP/Python）及作業系統，主要適用於大方向了解這個網站「到底是甚麼」時使用的。

  - 實戰指令: `whatweb <目標IP>`。

  - 本次觀察: 發現目標使用 Apache 2.4.41 跑在 Ubuntu Linux 上。

- cURL:

  - 用途: 透過 `curl -IL <URL>` 抓取伺服器的 HTTP 標頭 (Headers)，可以更精確提出想看到的請求，內容也會更詳細，但也只能顯示伺服器丟回來的原文(Raw data)。

  - 重點: Server 標頭會暴露軟體版本；Link 標頭可能暗示這是一個 WordPress 網站。

#### 2. 目錄與子網域爆破 (Brute-forcing)
這是在尋找「肉眼看不見的路徑」。

🛠️ 工具：Gobuster
- 目錄模式 (dir):

- 原理: 拿著單字列表 (Wordlist)，也就是常見的子網域會出現的名詞，一個個去猜測網址後面的路徑，並在根據回應的狀態碼來判斷網頁是否存在。
  
- 實戰指令: `gobuster dir -u <URL> -w /usr/share/dirb/wordlists/common.txt`。

  - 狀態碼解讀:

    - 200 OK: 檔案存在，可直接讀取。

    - 301/302 Redirect: 目錄存在，伺服器會將你導向到下一個路徑，也可能還是有內容。

    - 403 Forbidden: 目錄存在但權限被鎖住（值得關注）。

#### 3. 關鍵的資訊洩漏點 (Information Disclosure)
這些地方通常能直接拿到進入系統的鑰匙。

- robots.txt:

  - 目的: 此檔案告知搜尋引擎哪些路徑不要抓取，是管理遠不希望被搜尋到的目錄，但這個檔案必須放在公開網站的根目錄，否則搜尋引擎也看不懂，而且名稱是統一的。

  - 弱點: 駭客可以直接查看此檔案（網址後加上 /robots.txt）來獲取敏感路徑，但也要小心可能是陷阱。

  
- 網頁原始碼 (Source Code):

  - 操作: 瀏覽器按下 CTRL + U。

  - 重點: 尋找開發者的 HTML 註解 (``)，或是其他程式碼中任何可能可用的資訊。

  - 可能發現: 開發者在註解中留下了測試帳號與密碼 `admin:password123`。

#### 4. 實戰脈絡回顧 (Attack Sequence)
以下是你成功拿到 Flag 的完整邏輯流程：

1.啟動與確認: 點擊 Spawn Target 取得 IP 位址。

2.初步偵查: 執行 whatweb 確認伺服器正在運行，且標題為 HTB Academy。

<img width="985" height="304" alt="image" src="https://github.com/user-attachments/assets/796251c6-052f-471c-aab8-9070bd0d952d" />


3.自動化掃描: 執行 gobuster dir，發現了 robots.txt。
- 在這階段有發現三個有回應
  - index.php (Status: 200) => 通常是進入該網站的首頁或是主程式進入點，可以檢查網頁原始碼。
  - robots.txt (Status: 200) => 一個資訊洩漏的黃金來源，可以注意是否有`Disallow`開頭的路徑，這是管理者不希望被搜尋引擎找到的路徑。
  - wordpress (Status: 301) => 目標伺服器裝上了WordPress內容管理系統，此插件常存有漏洞。可能可以檢查是否有還未設定完成的WordPress，可能允許攻擊者直接接管伺服器。
    
<img width="989" height="751" alt="螢幕擷取畫面 2025-12-29 153918" src="https://github.com/user-attachments/assets/e2a54a62-7eab-4e58-a79b-a2bff8aa0cff" />


4.獲取線索 A: 讀取 robots.txt 內容，找到隱藏的登入路徑 /admin-login-page.php，順著進到隱藏路徑，發現了登入頁面。

<img width="983" height="354" alt="image" src="https://github.com/user-attachments/assets/6a802472-38b6-44f0-9c74-7f56cd972416" />


5.獲取線索 B: 檢查網頁原始碼，在註解中意外挖出開發者未刪除的測試憑證 admin / password123。

<img width="1919" height="653" alt="螢幕擷取畫面 2025-12-29 154815" src="https://github.com/user-attachments/assets/26ccf54e-ba9a-43b8-a44d-d20605e8d64e" />


6.最終突破: 瀏覽 /admin-login-page.php，輸入帳密登入，成功在後台管理頁面取得 Flag。

<img width="1919" height="666" alt="螢幕擷取畫面 2025-12-29 155338" src="https://github.com/user-attachments/assets/30f167de-edfd-48ae-8755-af363d76badf" />

