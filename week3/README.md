# W03｜多 VM 架構：分層管理與最小暴露設計

## 網路配置

| VM | 角色 | 網卡 | 模式 | IP | 開放埠與來源 |
|---|---|---|---|---|---|
| bastion | 跳板機 | NIC 1 | NAT | （192.168.79.128） | SSH from any |
| bastion | 跳板機 | NIC 2 | Host-only | （192.168.150.130） | — |
| app | 應用層 | NIC 1 | Host-only | （192.168.150.131） | SSH from 192.168.56.0/24 |
| db | 資料層 | NIC 1 | Host-only | （192.168.150.128） | SSH from app + bastion |

## SSH 金鑰認證

- 金鑰類型：（例：ed25519）
- 公鑰部署到：（例：app 和 db 的 ~/.ssh/authorized_keys）
- 免密碼登入驗證：
  - bastion → app：（shuamy@bastion:~$ ssh 192.168.150.131 "hostname && echo '金鑰認證成功'"
app）
  - bastion → db：（shuamy@bastion:~$ ssh 192.168.150.128 "hostname && echo '金鑰認證成功'"
db
金鑰認證成功）

## 防火牆規則

### app 的 ufw status
（貼上 `sudo ufw status verbose` 輸出）
Status: active
Default: deny (incoming), allow (outgoing)
To                         Action      From
--                         ------      ----
22/tcp                     ALLOW       192.168.150.0/24

### db 的 ufw status
（貼上 `sudo ufw status verbose` 輸出）
Status: active，允許來自 150.131 與 150.130 的 Port 22 連入

### 防火牆確實在擋的證據
（貼上步驟 13 的 curl 8080 失敗輸出）
shuamy@bastion:~$ curl -m 5 http://192.168.150.131:8080
curl: (28) Connection timed out after 5002 milliseconds

## ProxyJump 跳板連線
- 指令：（貼上你使用的 ssh -J 或 ssh config 設定）
- ssh -J shuamy@192.168.150.131 shuamy@192.168.150.128
- 驗證輸出：（貼上連線成功的 hostname 輸出）
- shuamy@db:~$ hostname
db
- SCP 傳檔驗證：（貼上結果）
scp -J shuamy@192.168.150.131 test.txt shuamy@192.168.150.128:/home/shuamy/
## 故障場景一：防火牆全封鎖

| 項目 | 故障前 | 故障中 | 回復後 |
|---|---|---|---|
| app ufw status | active + rules | deny all | （active + allow 22） |
| bastion ping app | 成功 | （Request timeout） | （成功） |
| bastion SSH app | 成功 | **timed out** | （成功） |

## 故障場景二：SSH 服務停止

| 項目 | 故障前 | 故障中 | 回復後 |
|---|---|---|---|
| ss -tlnp grep :22 | 有監聽 | 無監聽 | （有監聽） |
| bastion ping app | 成功 | 成功 | （成功） |
| bastion SSH app | 成功 | **refused** | （成功） |

## timeout vs refused 差異
（用自己的話說明兩種錯誤的差異、各自指向什麼排錯方向）
Connection timed out：發生在封包被防火牆「靜默丟棄 (Drop)」時。客戶端發送請求後，伺服器完全不回應，導致客戶端一直等待直到超過時間限制。這通常指向 網路層或防火牆規則 的問題。

Connection refused：發生在目標主機存在，但指定的 服務未啟動 或該埠口沒有在監聽 (Listen) 時。主機會主動回傳一個拒絕連線的訊息給客戶端。這通常指向 應用服務 (如 SSHD) 崩潰或未安裝 的問題。
## 網路拓樸圖
（嵌入或連結 network-diagram）
<img width="1028" height="1024" alt="Gemini_Generated_Image_6f8w2q6f8w2q6f8w" src="https://github.com/user-attachments/assets/057a0f4a-1a89-4160-85d4-42831977644b" />


## 排錯紀錄
- 症狀：嘗試透過 bastion 進行跳板連線時，出現 ssh: connect to host 192.168.150.130 port 22: Connection refused
- 診斷：（你首先查了什麼？）嘗試使用 sudo systemctl restart ssh 檢查服務，發現報錯 Unit ssh.service not found
- 修正：（做了什麼改動？）確認服務名稱或直接改用已驗證正常的 app (150.131) 作為跳板機
- 驗證：（如何確認修正有效？）成功透過 ssh -J 穿透 app 登入 db

## 設計決策
（說明本週至少 1 個技術選擇與取捨，例如：為什麼 db 允許 bastion 直連而不是只允許從 app 跳？）在本次設計中，我選擇讓 db 允許 bastion 直連。

理由：雖然從安全角度「只允許 app 連 db」最嚴謹，但實務上系統管理員（管理跳板機的人）需要獨立的維護通道。如果 app 層因為更新失敗或配置錯誤導致無法登入，透過 bastion 直接連向 db 進行緊急維護或備份，能提供更好的管理彈性
