## SOC 分析師每日儀表板監控清單 (Kibana 實戰版)
#### 1. 暴力破解監控 (Failed Logon Attempts)
 - 監控目標：識別是否有帳號正遭受大規模的自動化密碼猜測。
 - 關鍵過濾 (KQL)：`event.code: 4625`
 - 分析重點：
   - 數量激增：短時間內單一帳號有數十次失敗。
   - 排除雜訊：使用 `NOT user.name: *$` 排除電腦系統帳號的自動化嘗試。
   - 觀察 Logon Type：Type 3 (網路) 代表遠端攻擊；Type 2 (互動) 代表現場操作。
#### 2. 高風險帳號偵測 (Disabled Users)
 - 監控目標：找出「手握正確密碼」但嘗試登入已停用帳號的威脅。
 - 關鍵過濾 (KQL)：`winlog.event_data.SubStatus: 0xC0000072`
 - 分析重點：
   - 嚴重性：比一般失敗更危險。這代表駭客可能已經掌握了該帳號的有效憑證。
   - 溯源：檢查發起請求的 `host.hostname`，確認是否為公司內部的失陷主機。
#### 3. 權限濫用偵測 (Service Account RDP)
 - 監控目標：攔截駭客利用高權限服務帳號（Service Accounts）進行遠端登入。
 - 關鍵過濾 (KQL)：`user.name: svc-* AND event.code: 4624 AND winlog.logon.type: 10 (或 RemoteInteractive)`
 - 分析重點：
   - 零容忍原則：服務帳號理論上「絕不」應該有桌面登入行為。
   - 阻斷來源：利用 related.ip 鎖定攻擊者的 IP，這通常是橫向移動（Lateral Movement）的關鍵證據。
#### 4. 權限提升監控 (Privileged Group Changes)
 - 監控目標：追蹤誰把自己或他人加進了「管理員群組」。
 - 關鍵過濾 (KQL)：`event.code: (4732 OR 4733) AND group.name: "Administrators"`
 - 分析重點：
   - 動作比對：確認 `event.action` 是 Added 還是 Removed。
   - 時間與地點：利用 Absolute Time (絕對時間) 鎖定精確的變更日期（例如 2023-03-05）。
   - 身分驗證：比對 MemberSid，確認被加入的人員是否有經過正式的人事申請流程。

### 快速查表：SOC 必備代碼小手冊
|事件類型|Event ID|關鍵子狀態/類型|威脅層次|
|---|---|---|---|
|登入失敗|4625|0xC0000072 (帳號停用)|高 (High)|
|登入成功|4624|Type 10 (RDP 遠端)|視對象而定|
|新增群組成貝|4732|Administrators (管理員)|極高 (Critical)|
|網路連線 (Zeek)|N/A|外連至可疑C2 IP|極高 (Critical)|
