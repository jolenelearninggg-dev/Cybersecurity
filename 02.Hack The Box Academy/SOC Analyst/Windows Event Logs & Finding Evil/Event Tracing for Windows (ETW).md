# ETW 核心筆記：Windows 的「行車紀錄器」
ETW (Event Tracing for Windows) 是作業系統層級的深度監控機制，它能捕捉那些連傳統日誌（Event Logs）都看不到的微小細節。

## 1.三大核心角色 (The Trinity)

|角色|專業術語|比喻與功能|
|---|---|---|
|發佈者|Providers|感應器：散落在系統各處（如：網路、檔案、處理器），負責產生數據。|
|控制者|Controllers|遙控器：負責開啟、關閉錄影會話。最常用的工具是 logman.exe。|
|消費者|Consumers|播放器：讀取錄製好的數據，將其轉化為人類看得懂的文字日誌。|

## 2.運作邏輯：錄影四部曲
ETW 的運作是「非即時性」的，必須先有指令才有紀錄：

  1.啟動會話 (Start Session)：管理員透過 logman.exe 下令：「啟動錄影，目標是監控某個感應器」。
  
  2.產生事件 (Generate Events)：感應器（Providers）被喚醒，開始產生原始數據。
  
  3.儲存證據 (Storage)：系統將數據存成 .ETL 格式 的原始錄影檔。
  
  4.解碼分析 (Analysis)：分析人員使用工具（消費者）打開 .ETL 檔，查看「發生了什麼事」。
  常見播放器：
  - Event Viewer： 它可以讀入 .ETL 並把它變成人類看得懂的文字日誌。
  - PowerShell (Get-WinEvent)： 用來大規模搜尋日誌內容。
  - 專業工具： 例如 EtwExplorer 或 Message Analyzer。
  
## 3. 資安研究員為什麼要學這個？
「沒錄影，就沒真相。」
- Sysmon 的爸爸：Sysmon 本質上就是一個「全天候自動化」的 ETW 消費者。它幫你簡化了開啟錄影和分析的過程。
- 深度調查：當駭客使用高階手法躲過 Sysmon 時，研究員可以手動開啟特定的 ETW 感應器（例如 `.NET` 運行時），從最底層抓到駭客的尾巴。
- 效能平衡：ETW 平時不會全部開啟（因為會讓電腦變慢），它是鑑識人員在需要「搜證」時才會啟動的重裝武力。

## 實戰常用指令 (Logman)
- `logman query -ets`：查看目前電腦有哪些「感應器」正在錄影。
- `logman query providers`：列出這台電腦上所有可以被叫醒的感應器名稱。
