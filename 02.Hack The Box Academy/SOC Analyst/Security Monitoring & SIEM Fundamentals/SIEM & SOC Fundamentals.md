## SIEM 的「靈魂」—— 概念與演進

|名詞|全名|核心職責 |為什麼重要|
|----|---|--------------|---------|
|SIM|Security Information Management|長期儲存|負責把好幾個月、好幾年的日誌存好，用來做報表跟法規審計。|
|SEM|Security Event Management|即時監控|負責盯著當下的螢幕，看到有人在撞門 (暴力破解) 立刻吹哨。|
|SIEM|SIM + SEM|資安指揮中心|兩者合體。既能看現在的攻擊，也能翻舊帳找關聯，是現代企業的標配。|


## 第一線守衛：IDS 與 IPS (網路感應器)
在資料進入 SIEM 之前，通常會先經過這兩個設備。它們是 SIEM 最重要的「情報來源」。
|設備名稱|全名|動作|比喻|
|--------|---|----|----|
|IDS|Intrusion Detection System|偵測、告警。發現攻擊會記錄並通知，但不會攔截。|紅外線感應器：有人闖入會響鈴，但不會關門。|
|IPS|Intrusion Prevention System|偵測、阻斷。發現攻擊會直接丟棄惡意封包，中斷連線。|自動防護門：發現沒刷卡的人想闖入，門會直接鎖死。|


## Elastic Stack 的「骨架」—— 實體工具
Elastic Stack (原名 ELK) 是一套開源軟體組合成的「工具箱」。你可以用這組工具箱親手蓋出一個 SIEM。
核心組件：

1.Beats (基層搬運工)：裝在主機上，負責把日誌搬出來。
 - 角色：裝在每一台伺服器或電腦上的小程式。
 - 任務：只負責「看」日誌，然後第一時間「搬」走。
   
2.Logstash (加工過濾廠)：負責過濾資料、轉換格式（這就是 ECS 發揮作用的地方）。
 - 角色：中繼站。
 - 任務：收下 Beats 搬來的貨，把格式弄整齊（標準化）。比如把「IP位址」欄位統一命名為 source.ip。
   
3.Elasticsearch (超級大倉庫)：負責極速搜尋與儲存。
 - 角色：心臟。
 - 任務：它是個極速搜尋引擎，把所有日誌「索引」化。讓你在幾億條數據中找東西只要零點幾秒。
   
4.Kibana (監控室螢幕)：分析師操作的介面，使用 KQL 進行查詢。
 - 角色：介面。
 - 任務：你看到的圖表、搜尋框 (KQL) 都在這裡。這是你唯一需要操作的地方。
 - 在 Kibana 中，你會用到一種叫做 KQL (Kibana Query Language) 的語言來下指令。
 - 常用語法整理：
  1.精準比對 (field:value)：
   - `event.code: 4625` (尋找 Windows 登入失敗)。
  2.邏輯組合 (AND, OR, NOT)：
   - `event.code: 4625 AND user.name: admin*` (找所有 admin 開頭帳號的登入失敗)。
  3.範圍搜尋 (>, <, >=)：
   - `@timestamp > "2024-03-01"` (找特定時間後的事件)。
  4.萬用字元 (*)：
   - `user.name: admin*` (匹配 admin, administrator, admin123...)。
     
 - |語法類型|範例|意思|
   |-------|----|---|
   |精準搜尋|event.code: 4625|找出所有 Windows 「登入失敗」 的紀錄。|
   |邏輯組合|`"event.code: 4625 AND user.name: ""admin*"""`|找出所有**「管理員」**登入失敗的紀錄。|
   |深入調查|`winlog.event_data.SubStatus: 0xC0000072`|這是一個神祕代碼，代表**「嘗試登入已被停用的帳號」**（駭客很常亂槍打鳥弄到這種帳號）。|
  數據流向圖：
  Beats (採集) ➔ Logstash (清洗) ➔ Elasticsearch (存儲) ➔ Kibana (分析)

## 共同語言：ECS (Elastic Common Schema)
這是資安分析中最重要的概念。如果沒有它，分析師會瘋掉。
 - 為什麼需要？
   - 防火牆日誌寫：src_ip: 1.1.1.1
   - Windows 日誌寫：SourceAddress: 1.1.1.1
   - Web 日誌寫：client_addr: 1.1.1.1
 - ECS 的做法：強制把所有來源的「來源 IP」通通重新命名為 source.ip。
 - 好處：你只需要學一套欄位名稱，就能同時搜尋全公司的設備，大大提升關聯分析的效率。









