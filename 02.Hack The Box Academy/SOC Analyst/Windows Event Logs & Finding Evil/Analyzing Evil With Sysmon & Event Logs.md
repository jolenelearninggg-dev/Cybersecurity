## Analyzing Evil With Sysmon & Event Logs

### 偵測惡意活動的核心在於從「正常」的雜訊中揪出「異常」。

#### 1. Windows 內建核心日誌 (Security Logs)
在還沒安裝 Sysmon 前，我們主要依靠這兩個關鍵 ID：
- Event ID 4624：新的登入事件（監控誰在什麼時間、用什麼方式進入系統）。
- Event ID 4688：新進程建立（監控有哪些程式被啟動）。

#### 2. 什麼是 Sysmon (System Monitor)？
Sysmon 是一個強大的「外掛」監控服務，它會常駐在系統中（重啟也不會消失），提供比內建日誌更細緻的紀錄。<br>
**Sysmon 的三大組成部分：**
- Windows 服務：負責持續監控系統活動。
- 設備驅動 (Driver)：負責從系統底層攔截並捕捉數據。
- 事件日誌 (Event Log)：將捕捉到的數據轉化為可閱讀的紀錄。

#### 3. Sysmon 的核心優勢與關鍵 ID
Sysmon 最強大的地方在於它能記錄 Security Log 漏掉的細節：
- Event ID 1：進程建立 (Process Creation)。
- Event ID 3：網路連線 (Network Connection)。
- Event ID 7：
- Event ID 10：
- 其他：包含檔案建立時間變動、驅動加載等深度資訊。

#### 4. 設定與安裝 (Configuration & Setup)
為了精確抓到我們要的行為（避免日誌量太大爆炸），Sysmon 使用 XML 設定檔 來決定要包含 (Include) 或排除 (Exclude) 哪些內容。
- 推薦設定檔：SwiftOnSecurity/sysmon-config（這是業界標準，也是本章節使用的版本）。
**安裝指令：**
```
DOS
:: 安裝並開啟 MD5/SHA256 指紋計算、載入模組監控與網路監控
sysmon.exe -i -accepteula -h md5,sha256,imphash -l -n
```
**更新自定義設定：**
```
DOS
:: 將自訂的 XML 設定檔套用到現有的 Sysmon
sysmon.exe -c filename.xml
```


## 📂 實戰案例
### 案例1：DLL 劫持偵測 (Detecting DLL Hijacking)
#### 1.核心概念：什麼是 DLL 劫持？
Windows 程式在執行時，如果需要用到某個工具箱 (DLL)，它會按照一定的搜尋順序去找`(先從同資料夾下目錄去翻，沒找到再去System32去找)`。 <br>
駭客的策略是：
- 偷天換日：將一個惡意的 DLL 重新命名為程式正在尋找的名稱（例如 WININET.dll）。
- 近水樓台：把這個惡意 DLL 放在跟程式（如 calc.exe）同一個資料夾。
- 結果：程式會優先加載同資料夾下的惡意 DLL，而不會去 System32 找真正的官方 DLL。

#### 2.實驗操作指令彙整
|步驟|指令 / 動作|目的|
|---|---|---|
|環境準備|`copy C:\Windows\System32\calc.exe ...\Desktop\`|將目標程式移至可寫入的桌面資料夾。|
|武器偽裝|`copy "...\reflective_dll.x64.dll" ...\Desktop\WININET.dll`|將惡意代碼重新命名，符合 calc.exe 尋找的名稱。|
|重啟監控|`sysmon.exe -c sysmonconfig-export.xml`|載入修改過的 XML，確保不漏掉任何模組載入事件。|
|執行攻擊|執行桌面上的`calc.exe`|觀察是否跳出 MessageBox 而非計算機。|

#### 3.關鍵偵測指標 (IOCs)
當你在 Sysmon Event ID 7 (Module Load) 裡看到紀錄時，要檢查這三點：
- 1.路徑異常 (Binary Path)：calc.exe 理論上只應該出現在 C:\Windows\System32。如果它出現在 Desktop 或 Downloads，這就是紅燈警訊。
- 2.加載來源 (Module Path)：WININET.dll 這種核心組件應該從 System32 加載。如果它跟著 calc.exe 從桌面一起被加載，這就是劫持。
- 3.數位簽章 (Signature Status)：正版的 DLL 會有 Signed: true 且來源是 Microsoft。惡意 DLL 通常是 Unsigned (false)。
  
#### 4.Sysmon 設定的邏輯：Include vs. Exclude <br>
在修改 sysmonconfig-export.xml 時：
- 原本 (Include)：可能只記錄特定危險的 DLL。
- 修改為 (Exclude)：將篩選規則改為排除模式，但在實作中不填寫排除內容，這等於是**「開啟全紀錄模式」**。
- 原因：為了確保我們能捕捉到 calc.exe 載入 WININET.dll 的瞬間，不被原本的減噪規則過濾掉。























從原理到偵測的對照表


|案例|攻擊手法 (白話文)|核心指令/動作|關鍵 Sysmon ID|偵測邏輯 (分析師的直覺)|
|---|---|---|---|---|
|1. DLL 劫持|「換工具箱」。在真程式旁邊放一個「假名字、真惡意」的工具箱。|複製 calc.exe + 把惡意 DLL 改名為 WININET.dll。|ID 7 (Module Load)|為什麼 calc.exe 用的工具箱沒有數位簽章？且路徑不在 System32？|
|2. 非託管注入|「奪舍附身」。讓一個老實的程式載入它原本不需要的翻譯官 (CLR)。|Invoke-PSInject 把 PowerShell 引擎塞進 spoolsv.exe。|ID 7 (Module Load)|為什麼一個「列印程式」會載入 .NET 翻譯官 (clrjit.dll)？它又不寫 C#！|
|3. 憑證傾印|「偷翻金庫」。利用特權去讀取密碼服務的記憶體，把 Hash 倒出來。|privilege::debug (拿鑰匙) ➔ sekurlsa::logonpasswords (搬金庫)。|ID 10 (Process Access)|為什麼一個從 Downloads 執行的程式會去存取 lsass.exe？這絕對是在偷密碼！|













https://github.com/SwiftOnSecurity/sysmon-config <br>
https://github.com/olafhartong/sysmon-modular
