# W04｜Linux 系統基礎：檔案系統、權限、程序與服務管理

## FHS 路徑表

| FHS 路徑 | FHS 定義 | Docker 用途 |
|---|---|---|
| /etc/docker/ | （系統主機特定設定檔） | （存放 Docker Daemon 的配置（如 daemon.json） |
| /var/lib/docker/ | （變動性狀態資料路徑） | （存放 Images、Containers 等所有實際資料） |
| /usr/bin/docker | （使用者執行檔） | （Docker CLI 客戶端程式，讓使用者輸入命令） |
| /run/docker.sock | （運行中程序的 Socket 檔案） | （CLI 與 Daemon 溝通的窗口，受權限管控） |

## Docker 系統資訊

- Storage Driver：（貼上 `docker info` 中的值）overlayfs
- Docker Root Dir：（貼上 `docker info` 中的值）/var/lib/docker
- 拉取映像前 /var/lib/docker/ 大小：（184K）
- 拉取映像後 /var/lib/docker/ 大小：（240MB (來自 docker system df 觀測結果)）

## 權限結構

### Docker Socket 權限解讀
（貼上 `ls -la /var/run/docker.sock` 輸出，逐欄說明 owner/group/others 的權限）
執行 ls -la /var/run/docker.sock 輸出：
srw-rw---- 1 root docker 0 ... /var/run/docker.sock

s: 代表這是 Unix Domain Socket。

Owner (root): 擁有讀寫權限。

Group (docker): 擁有讀寫權限，加入此群組的使用者即可操作 Docker。

Others: 完全無權限。
### 使用者群組
（貼上 `id` 輸出，說明是否包含 docker 群組）
執行 id 輸出：
uid=1000(shuamy) ... groups=1000(shuamy),124(docker)
（說明：已成功透過 newgrp docker 獲得 docker 群組權限。）

### 安全意涵
（用自己的話說明為什麼 docker group ≈ root，安全示範的觀察結果）
加入 docker 群組等同於擁有 root 權限。我在實驗中成功利用 docker run -v /etc/shadow:/host-shadow 指令，在容器內直接讀取了主機的 /etc/shadow 檔案（包含使用者密碼雜湊），這證明了 Docker 權限如果不當開放，會導致嚴重的安全性漏洞
## 程序與服務管理

### systemctl status docker
（貼上 `systemctl status docker` 輸出）
systemctl status docker
（顯示為 active (running)）
### journalctl 日誌分析
（貼上 `journalctl -u docker --since "1 hour ago"` 的重點摘錄，說明看到什麼事件）
關鍵事件: 看到 API listen on /run/docker.sock。

分析總結: 日誌證實了 Docker Daemon 啟動後會建立 Unix Socket 監聽 API 請求。當我們修改該檔案權限為 600 時，雖然 Daemon 仍在運行，但客戶端會因權限不足而無法連線
### CLI vs Daemon 差異
（用自己的話說明兩者的差異，為什麼 `docker --version` 正常不代表 Docker 能用）
CLI 是我們輸入指令的工具，Daemon 是背景執行的服務。當我停止 docker.socket 後，雖然 docker --version 還能執行，但 docker ps 會噴出錯誤，因為 CLI 找不到 Daemon 來處理請求。
## 環境變數

- $PATH：（貼上內容）/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin
- which docker：（填入路徑）/usr/bin/docker
- 容器內外環境變數差異觀察：（簡述）在主機（bastion）執行 env 會看到大量的系統變數（如 USER、LS_COLORS、SSH_AUTH_SOCK 等），代表宿主機的運行環境；而在容器內（如執行 docker run alpine env）變數極少，通常僅有 PATH、HOSTNAME 與 HOME，這體現了容器的隔離性，它只攜帶運行程式所需的最小變數。

## 故障場景一：停止 Docker Daemon

| 項目 | 故障前 | 故障中 | 回復後 |
|---|---|---|---|
| systemctl status docker | active | （inactive (dead)） | （active） |
| docker --version | 正常 | （Cannot connect...） | （正常） |
| docker ps | 正常 | Cannot connect | （正常） |
| ps aux grep dockerd | 有 process | （無相關 process (僅剩 grep 自身)） | （有 process） |

## 故障場景二：破壞 Socket 權限

| 項目 | 故障前 | 故障中 | 回復後 |
|---|---|---|---|
| ls -la docker.sock 權限 | srw-rw---- | （srw------- (600)） | （srw-rw----） |
| docker ps（不加 sudo） | 正常 | permission denied | （正常） |
| sudo docker ps | 正常 | （正常） | （正常） |
| systemctl status docker | active | （active (running)） | （active (running)） |

## 錯誤訊息比較

| 錯誤訊息 | 根因 | 診斷方向 |
|---|---|---|
| Cannot connect to the Docker daemon | （Docker 服務或 Socket 未啟動） | （檢查 systemctl status docker） |
| permission denied…docker.sock | （使用者不在 docker 群組或 Socket 權限被改動） | （檢查 id 或 ls -la /run/docker.sock） |

（用自己的話說明兩種錯誤的差異，各自指向什麼排錯方向）
「Cannot connect」代表連線目標不存在，通常是服務倒了；「permission denied」代表目標在但你沒權限進入
## 排錯紀錄
- 症狀：下達 docker ps 出現 permission denied
- 診斷：（你首先查了什麼？）執行 ls -la /var/run/docker.sock 發現群組權限不正確
- 修正：（做了什麼改動？）執行 sudo chmod 660 /var/run/docker.sock 恢復權限
- 驗證：（如何確認修正有效？）再次執行 docker ps 恢復正常輸出

## 設計決策
（說明本週至少 1 個技術選擇與取捨，例如：為什麼教學環境用 `usermod` 加 group 而不是每次 sudo？這個選擇的風險是什麼？）
技術選擇：將使用者加入 docker 群組而非每次使用 sudo
在本次實驗中，我們選擇執行 sudo usermod -aG docker $USER 來賦予使用者操作權限，而不是要求每次執行 Docker 指令時都加上 sudo。

選擇原因（取捨點）：
操作自動化與便利性：在容器化開發流程中，Docker 指令的使用頻率極高。若每次都需輸入密碼並加上 sudo，會大幅降低開發效率。此外，若未來需要撰寫自動化腳本或整合 CI/CD 工具，免密碼操作是技術上的必要條件。

減少 Root 誤操作風險：使用 sudo 會以完整的 root 身份執行指令，若在輸入指令時發生筆誤，可能會對主機造成不可挽回的損害。透過群組權限，我們將操作限制在 Docker 守護行程（Daemon）能處理的範圍內。

技術風險與副作用：
等同於 Root 的特權（核心風險）：這是本次實驗 Checkpoint B 證明的核心問題。由於 Docker Daemon 以 root 身份執行，且具備掛載主機目錄的權限，因此加入 docker 群組的使用者可以輕易透過 -v 參數讀取或修改主機上的敏感檔案（如本次實驗成功讀取的 /etc/shadow）。

安全性邊界消失：一旦使用者的帳號遭竊取，攻擊者不需要知道 root 密碼，就能透過 Docker 容器完全接管整台主機，達成權限提升（Privilege Escalation）的目的。

替代方案評估：
若要追求更高安全性，技術上可選擇 Rootless Docker 模式。該模式不依賴 root 權限啟動 Daemon，能有效避免容器逃逸（Container Escape）後威脅宿主機安全，但在網路設定與效能上會存在較多限制。在本教學環境中，為了學習標準的 FHS 結構與 Socket 通訊原理，我們選擇了傳統的群組模式
