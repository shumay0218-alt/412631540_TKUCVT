# W01｜虛擬化概論、環境建置與 Snapshot 機制

## 環境資訊
- Host OS：Windows 11 
- VM 名稱：vct-w01-412631540
- Ubuntu 版本：Ubuntu 24.04.4 LTS
- Docker 版本：Docker version 25.0.3
- Docker Compose 版本：Docker Compose version v2.24.5
<img width="993" height="1008" alt="04339dbf-5c0c-4360-976c-aa5fe0c2299b" src="https://github.com/user-attachments/assets/2fecfcd8-e3ba-46f5-811b-fe361957c2a1" />

## VM 資源配置驗證

| 項目 | VMware 設定值 | VM 內命令 | VM 內輸出 |
|---|---|---|---|
| CPU | 2 vCPU | `lscpu \| grep "^CPU(s)"` | CPU(s): 2 |
| 記憶體 | 4 GB | `free -h \| grep Mem` | Mem: 3.8Gi (約 4 GB) |
| 磁碟 | 40 GB | `df -h /` | /dev/sda3 39G (約 40 GB) |
| Hypervisor | VMware | `lscpu \| grep Hypervisor` | Hypervisor vendor: VMware |
<img width="993" height="1008" alt="螢幕擷取畫面 2026-03-24 180303" src="https://github.com/user-attachments/assets/27352f19-9279-4c83-9003-2870b77f5983" />
<img width="993" height="1008" alt="螢幕擷取畫面 2026-03-24 180352" src="https://github.com/user-attachments/assets/9addc669-4786-4f03-887c-1048549e08f6" />

## 四層驗收證據
- [x ] ① Repository：`cat /etc/apt/sources.list.d/docker.list` 輸出
- [x ] ② Engine：`dpkg -l | grep docker-ce` 輸出
- [ x] ③ Daemon：`sudo systemctl status docker` 顯示 active
<img width="993" height="1008" alt="螢幕擷取畫面 2026-03-24 180434" src="https://github.com/user-attachments/assets/34cbe50c-2c75-446b-9ad6-4b2c2f8b919b" />
- [ x] ④ 端到端：`sudo docker run hello-world` 成功輸出
- <img width="993" height="1008" alt="螢幕擷取畫面 2026-03-24 180451" src="https://github.com/user-attachments/assets/0d68e793-17c7-466a-b9b4-8737511a6b01" />
- [ x] Compose：`docker compose version` 可執行
- <img width="993" height="1008" alt="螢幕擷取畫面 2026-03-24 180510" src="https://github.com/user-attachments/assets/5cb2327c-cd78-41f2-b68f-d6636fa2cca4" />

<img width="987" height="746" alt="螢幕擷取畫面 2026-03-24 180554" src="https://github.com/user-attachments/assets/ac1c7ff6-2f53-4c7e-9936-08a9f0977683" />
<img width="993" height="1008" alt="螢幕擷取畫面 2026-03-24 181035" src="https://github.com/user-attachments/assets/ada87e05-351b-43e0-bfd3-b54579e62a13" />

## 容器操作紀錄
- [x ] nginx：`sudo docker run -d -p 8080:80 nginx` + `curl localhost:8080` 輸出
- [ x] alpine：`sudo docker run -it --rm alpine /bin/sh` 內部命令與輸出
- [ x] 映像列表：`sudo docker images` 輸出

## Snapshot 清單

| 名稱 | 建立時機 | 用途說明 | 建立前驗證 |
|---|---|---|---|
| clean-baseline | （時間點） | （此節點代表的狀態） | （列出建點前做了哪些驗證） |
| docker-ready | （時間點） | （此節點代表的狀態） | （列出建點前做了哪些驗證） |

## 故障演練三階段對照

| 項目 | 故障前（基線） | 故障中（注入後） | 回復後 |
|---|---|---|---|
| docker.list 存在 | 是 | 否 | （是） |
| apt-cache policy 有候選版本 | 是 | 否 | （是） |
| docker 重裝可行 | 是 | 否 | （是） |
| hello-world 成功 | 是 | N/A | （是） |
| nginx curl 成功 | 是 | N/A | （是） |

## 手動修復 vs Snapshot 回復

| 面向 | 手動修復 | Snapshot 回復 |
|---|---|---|
| 所需時間 | （約 1-2 分鐘 (需手動找回檔案名稱並更名)） | （約 10 秒 (點擊 Revert 即可)） |
| 適用情境 | （已知確切損壞原因與修復指令時） | （不知原因的系統崩潰或大規模設定錯誤時） |
| 風險 | （高 (可能打錯路徑或權限設定錯誤)） | （極低 (系統狀態完美回滾)） |

## Snapshot 保留策略
- 新增條件：完成一階段重大軟體安裝或完成複雜的專案部署前
- 保留上限：最多保留 3 個關鍵節點，避免佔用過多主機磁碟空間
- 刪除條件：當新的快照節點經過完整測試確認穩定後，刪除最舊的過期節點

## 最小可重現命令鏈
（列出讓他人能重現故障注入與回復驗證的命令序列）
# 1. 注入故障：破壞 Docker 來源清單與指令連結
sudo mv /etc/apt/sources.list.d/docker.list /etc/apt/sources.list.d/docker.list.broken

# 2. 驗證故障
sudo apt update  # 發現缺少 Docker 軟體源
sudo docker run hello-world # 提示 command not found

# 3. 回復驗證
# 使用 VMware 工具列：VM -> Snapshot -> Revert to Snapshot: docker-ready

## 排錯紀錄
- 症狀：使用 VMware Easy Install 安裝 Ubuntu 24.04 時，畫面長時間卡在「Copying files」且重啟後不斷循環安裝
- 診斷：（你首先查了什麼？）VMware 的自動安裝腳本與 Ubuntu 24.04 核心引導過程不相容。
- 修正：（做了什麼改動？）在建立虛擬機時選擇「I will install the operating system later」，手動掛載 ISO 映像檔
- 驗證：（如何確認修正有效？）成功進入手動安裝畫面，並順利完成 vct-w01-412631540 主機建置。

## 設計決策
（說明本週至少 1 個技術選擇與取捨）
選擇在安裝 Docker 前先建立一個 clean-baseline 快照。 考量到 Ubuntu 24.04 是較新的版本，安裝 Docker 軟體源（GPG Key）的過程較容易因網路或版本差異出錯。建立此快照可以讓我即使在安裝過程中失敗，也能在 10 秒內回到純淨狀態重新嘗試，而不需浪費時間重裝整個作業系統。

