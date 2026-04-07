# 📝 實作練習：Splunk實作練習
## 1. 抓「偷密碼」的現行犯(Q1 & Q2)
- 場景：懷疑有人在記憶體裡偷密碼。
- 關鍵指令：`EventCode=10 TargetImage="*lsass.exe"`
- 進階過濾：
  - 搜 `CallTrace="*UNKNOWN*"` $\rightarrow$ 找「惡意注入」（如`notepad.exe`）。
  - 搜 `CallTrace!="*UNKNOWN*"` $\rightarrow$ 找「被濫用的系統工具」（如`comsvcs.dll`）。
- 重點警覺：看到 `rundll32.exe` 調用了 `comsvcs.dll`，這就是典型的 LOLBin 攻擊（利用合法工具幹壞事）。

## 2. 抓「披著羊皮的狼」(Q3)
- 場景：有些程式看起來很正常，但行為很怪（例如載入了不該載入的 DLL）。
- 關鍵邏輯：
  - 先用 `EventCode=7` 找哪些「非 .NET 程式」載入了 .NET 的核心 clr.dll。
  - 再用 `EventCode=1` 回頭查這些程式的 「爸爸（ParentImage）」 是誰。
- 重點警覺：如果一個合法程式（如`rundll32.exe`）是由一個拼錯字（如`PSEXECSCVCS.exe`）或亂取名（如 randomfile.exe）的程式啟動的，那它就是被當成了「犧牲品（`Sacrificial Process`）」。

## 3. 追蹤「駭客的老巢與擴張」(Q4 & Q5)
- 場景：已經抓到壞人程式了，想知道他在網路連去哪、還害了誰。
- 關鍵邏輯：
  - 由內向外：查壞人程式的`EventCode=3` $\rightarrow$ 找出 C2 IP（駭客基地）。
  - 由外向內：查這個 C2 IP 作為`SourceIp`又連去了內網哪裡？
- 重點警覺：看到連向 Port 3389 $\rightarrow$ 駭客在玩 **遠端桌面（RDP）** 控制。這代表駭客正在進行橫向移動(Lateral Movement)。
