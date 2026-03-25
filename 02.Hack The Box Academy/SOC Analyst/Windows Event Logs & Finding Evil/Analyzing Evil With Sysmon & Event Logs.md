# 💻Analyzing Evil With Sysmon & Event Logs

## 偵測惡意活動的核心在於從「正常」的雜訊中揪出「異常」。

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


# 📂 實戰案例

## 案例1：DLL 劫持偵測 (Detecting DLL Hijacking)
#### 1.核心概念：什麼是 DLL 劫持？
Windows 程式在執行時，如果需要用到某個工具箱 (DLL)，它會按照一定的搜尋順序去找`(先從同資料夾下目錄去翻，沒找到再去System32去找)`。 <br>
駭客的策略是：
- 偷天換日：將一個惡意的 DLL 重新命名為程式正在尋找的名稱（例如 WININET.dll）。
- 近水樓台：把這個惡意 DLL 放在跟程式（如 calc.exe）同一個資料夾。
- 結果：程式會優先加載同資料夾下的惡意 DLL，而不會去 System32 找真正的官方 DLL。

#### 2.實驗操作指令彙整
|步驟|指令/動作|目的|
|---|---|---|
|環境準備|`copy C:\Windows\System32\calc.exe ...\Desktop\`|將目標程式移至可寫入的桌面資料夾。|
|武器偽裝|`copy "...\reflective_dll.x64.dll" ...\Desktop\WININET.dll`|將惡意代碼重新命名，符合 calc.exe 尋找的名稱。|
|重啟監控|`sysmon.exe -c sysmonconfig-export.xml`|載入修改過的 XML，確保不漏掉任何模組載入事件。|
|執行攻擊|執行桌面上的`calc.exe`|觀察是否跳出 MessageBox 而非計算機。|

#### 3.關鍵偵測指標 (IOCs)
當你在 Sysmon Event ID 7 (Module Load) 裡看到紀錄時，要檢查這三點：
- 1.路徑異常 (Binary Path)：`calc.exe` 理論上只應該出現在 `C:\Windows\System32`。如果它出現在 Desktop 或 Downloads，這就是紅燈警訊。
- 2.加載來源 (Module Path)：`WININET.dll` 這種核心組件應該從 System32 加載。如果它跟著 calc.exe 從桌面一起被加載，這就是劫持。
- 3.數位簽章 (Signature Status)：正版的 DLL 會有 Signed: true 且來源是 Microsoft。惡意 DLL 通常是 Unsigned (false)。
  
#### 4.Sysmon 設定的邏輯：Include vs. Exclude <br>
在修改 `sysmonconfig-export.xml` 時：
- 原本 (Include)：可能只記錄特定危險的 DLL。
- 修改為 (Exclude)：將篩選規則改為排除模式，但在實作中不填寫排除內容，這等於是**「開啟全紀錄模式」**。
- 原因：為了確保我們能捕捉到 calc.exe 載入 WININET.dll 的瞬間，不被原本的減噪規則過濾掉。


## 案例2：非託管 PowerShell 注入偵測
#### 1. 核心概念：託管 (Managed) vs. 非託管 (Native)
- 託管碼 (Managed Code)：如 C# 或 PowerShell。它們需要一個「翻譯官」才能執行，這個環境叫 CLR (Common Language Runtime)。
- 非託管碼 (Native Code)：如 C++ 寫的 `spoolsv.exe` (列印服務) 或 `notepad.exe`。它們直接跟硬體溝通，不需要翻譯官。
- 關鍵 DLL (翻譯官的身分證)：
  - `clr.dll`：.NET 運行時的核心。
  - `clrjit.dll`：即時編譯器 (JIT)，負責把代碼轉成電腦指令。

#### 2. 攻擊原理：借屍還魂 (Possession)
駭客為了躲避監控，不會直接開啟 `powershell.exe`（因為這會觸發警報），而是將 PowerShell 的「靈魂」（核心 DLL 和代碼）注入到一個原本不需要 .NET 的原生程式（如 `spoolsv.exe`）裡面執行。 <br>
結果： spoolsv.exe 雖然外表沒變，但它的記憶體裡突然多出了 clr.dll，這就代表它被「附身」了。

#### 3. 實驗操作指令彙整
|步驟|指令/動作|目的|
|---|---|---|
|準備環境|`Set-ExecutionPolicy Unrestricted -Scope CurrentUser`|解開系統對腳本執行的限制（開鎖）。|
|載入工具|`Import-Module .\Invoke-PSInject.ps1`|將注入工具載入到目前的 PowerShell 環境（載入武器）。|
|發動注入|`Invoke-PSInject -ProcId <PID> -PoshCode "..."`|將代碼注入指定的 PID 程式。PoshCode 是 Base64 編碼過的指令。|
|觀察顏色|在Process Hacker查看`spoolsv.exe`|觀察其是否由原本的顏色轉變為 綠色（代表變為 Managed 狀態）。|

#### 4. 關鍵偵測指標 (IOCs) - Sysmon Event ID 7
身為分析師，我們要尋找的是**「身分與行為的矛盾」**：

1.異常模組載入：
  - 當 `spoolsv.exe`（列印服務）或 notepad.exe（記事本）這類原生程式載入了` clr.dll` 或 `clrjit.dll` 時，這 99% 是注入攻擊。
    
2.狀態轉變：
  - 在 Process Hacker 中，非 .NET 程式突然變成了 綠色 (Managed)。
    
3.指紋驗證 (Hash)：
  - 即使程式被注入，載入的 `clrjit.dll` 通常還是系統原本的那一個。紀錄它的 SHA256 是為了確保環境的一致性，並確認是由哪一個版本的 .NET 引擎發動的。



## 案例3：憑證傾印偵測 (Detecting Credential Dumping)
#### 1. 核心概念：LSASS 與 MimikatzLSASS (lsass.exe)：
- Windows 的憑證管理中心，就像系統的「密碼金庫」。它在記憶體中存放著使用者登入後的密碼雜湊值（Hash）。
- Mimikatz：資安界最著名的憑證提取工具。它能讀取 LSASS 的記憶體，並把裡面的 NTLM Hash 或明文密碼倒出來。
- 攻擊邏輯：駭客進入系統後，首要目標就是 LSASS。拿到 NTLM Hash 後，即使不知道密碼，也能進行「傳遞雜湊攻擊 (Pass-the-Hash)」來控制整個網域。

#### 2. 實驗操作指令彙整
|步驟|指令/動作|目的|
|---|---|---|
|修改監控邏輯|在XML中將ID 10的`onMatch`從`include` 改為`exclude`|撤掉白名單濾網，記錄「所有」對行程的存取行為。|
|更新Sysmon|`Sysmon.exe -c sysmonconfig-export.xml`套用新的監控規則。|
|拿取權限|`privilege::debug`|請求`SeDebugPrivilege`(偵錯權限)，這是讀取系統記憶體的「通行證」。|
|倒出憑證|`sekurlsa::logonpasswords`|執行Mimikatz的核心功能，傾印所有登入者的Hash。|

#### 3. 關鍵偵測指標 (IOCs) - Sysmon Event ID 10
這一案偵測的重點是 ProcessAccess (行程存取)，而非載入模組。
1.目標行程 (TargetImage)：
  - 監控任何對`C:\Windows\System32\lsass.exe`的存取行為。
    
2.來源異常 (SourceImage)：
  - 如果是一個奇怪路徑下的程式（如 `Downloads\AgentEXE.exe`）在存取 LSASS，極度可疑。

3.權限越級 (SourceUser vs TargetUser)：
  - 來源使用者是普通人（如 waldo），目標卻是 SYSTEM。

4.特殊權限請求：
  - 日誌中出現請求 SeDebugPrivilege 的動作。
  
### 三大案例大總結 (Final Summary)
這三個練習讓你從三個不同層面掌握了系統監控的精髓：
|案例|監控重點|攻擊手法|核心指令/動作|關鍵 Sysmon ID|偵測邏輯 (分析師的直覺)|
|---|---|---|---|---|---|
|1. DLL 劫持|檔案路徑與簽章|「換工具箱」。在真程式旁邊放一個「假名字、真惡意」的工具箱。|複製 calc.exe + 把惡意 DLL 改名為 WININET.dll。|ID 7 (Module Load)|為什麼 calc.exe 用的工具箱沒有數位簽章？且路徑不在 System32？|
|2. 非託管注入(powershell)|程式語言運行環境|「奪舍附身」。讓一個老實的程式載入它原本不需要的翻譯官 (CLR)。|Invoke-PSInject 把 PowerShell 引擎塞進 spoolsv.exe。|ID 7 (Module Load)|為什麼一個「列印程式」會載入 .NET 翻譯官 (clrjit.dll)？它又不寫 C#！|
|3. 憑證傾印|行程間的非法觸碰|「偷翻金庫」。利用特權去讀取密碼服務的記憶體，把 Hash 倒出來。|privilege::debug (拿鑰匙) ➔ sekurlsa::logonpasswords (搬金庫)。|ID 10 (Process Access)|為什麼一個從 Downloads 執行的程式會去存取 lsass.exe？這絕對是在偷密碼！|

###### 參考資料
https://github.com/SwiftOnSecurity/sysmon-config <br>
https://github.com/olafhartong/sysmon-modular
