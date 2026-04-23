# 期中實作 — <412631540> <許沛恩>

## 1. 架構與 IP 表
<Mermaid 圖 + 表格>
VM 角色,介面 (Interface),網卡模式,IP 位址,作用
bastion,ens33,NAT,192.168.106.128 (預估*),對外接收 Host 的 SSH 連線
bastion,ens37,Host-only,192.168.150.128,內網對 app 的通訊門戶
app,ens33,Host-only,192.168.150.131,執行容器服務 (僅內網可達)
<img width="2504" height="2755" alt="Bastion VM Network-2026-04-23-082510" src="https://github.com/user-attachments/assets/55151d47-9e8c-4249-8fe5-7cdacb79f36c" />


## 2. Part A：VM 與網路
<命令 + 關鍵輸出>
bastion (跳板機)：同時擁有 NAT (對外) 與 Host-only (對內) 兩個 IP。
shuamy@bastion:~$ ip -4 addr
# 關鍵輸出截取
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> ...
    inet 192.168.106.128/24 brd 192.168.106.255 scope global dynamic ens33
3: ens37: <BROADCAST,MULTICAST,UP,LOWER_UP> ...
    inet 192.168.150.128/24 brd 192.168.150.255 scope global dynamic ens37
app (應用機)：僅有一張 Host-only 網卡，確保外部無法直接存取。
shuamy@app:~$ ip -4 addr show ens33
# 關鍵輸出截取
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> ...
    inet 192.168.150.131/24 brd 192.168.150.255 scope global dynamic noprefixroute ens33
2. 內網連通性測試 (Ping Test)
從 bastion 測試連向 app，確保兩機透過 192.168.150.x 網段可正常溝通。這也是後續 ProxyJump 能成功的基礎。
shuamy@bastion:~$ ping -c 4 192.168.150.131
PING 192.168.150.131 (192.168.150.131) 56(84) bytes of data.
64 bytes from 192.168.150.131: icmp_seq=1 ttl=64 time=1.50 ms
64 bytes from 192.168.150.131: icmp_seq=2 ttl=64 time=2.33 ms
64 bytes from 192.168.150.131: icmp_seq=3 ttl=64 time=1.55 ms
64 bytes from 192.168.150.131: icmp_seq=4 ttl=64 time=1.26 ms

--- 192.168.150.131 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3005ms
## 3. Part B：金鑰、ufw、ProxyJump
<防火牆規則表 + ssh app 成功證據>
項目,設定內容,說明
防火牆狀態,active (已啟動),確保過濾規則正在運作。
主要放行規則,22/tcp ALLOW 192.168.150.130,核心安全證據：僅允許來自 Bastion 跳板機的內部 IP 進行 SSH 連線。
輔助規則,22/tcp ALLOW 192.168.150.0/24,放行整個內部虛擬網段的 SSH 通訊。
<img width="965" height="1008" alt="螢幕擷取畫面 2026-04-23 181241" src="https://github.com/user-attachments/assets/8d58bffc-4181-4ebf-8a07-8bb344327c0c" />
<img width="864" height="927" alt="螢幕擷取畫面 2026-04-23 181205" src="https://github.com/user-attachments/assets/c54e2768-9cdb-481a-9bbf-165bf0b8d37a" />
<img width="965" height="1008" alt="螢幕擷取畫面 2026-04-23 181104" src="https://github.com/user-attachments/assets/d75a2b89-7452-4793-b95a-c5eb796f2b81" />

## 4. Part C：Docker 服務
<systemctl status docker + curl 輸出>
A. Docker 服務狀態 (systemctl status docker)在 App 伺服器 (shuamy@app) 上，Docker 引擎已成功安裝並啟動。指令：sudo systemctl status docker狀態證明：畫面上顯示 active (running)。這代表 Docker Daemon 運作正常，隨時可以處理容器部署請求。容器運行證據：透過 sudo docker ps 可以看到名為 web 的 Nginx 容器正在運行，且其埠號映射為 0.0.0.0:8080->80/tcp。B. 內網服務存取驗證 (curl 輸出)在 Bastion 跳板機 (shuamy@bastion) 上，我們透過 Host-only 內網網段測試 App 上的網頁服務。測試指令：curl -I http://192.168.150.131:8080驗證證據 (200 OK)：第一行明確回傳 HTTP/1.1 200 OK。Server 欄位顯示 nginx/1.29.8。安全意義：這證明了儘管 App 機器沒有外網，但位於同一私有網段的 Bastion 機器可以繞過防火牆（因已設定 ufw allow 8080）成功存取其內部服務。C. 最終防火牆規則表 (防禦總結)為了達成上述連線，App 伺服器的防火牆規則最終配置如下：埠號 (Port)動作 (Action)來源 (From)用途22/tcpALLOW192.168.150.130僅允許 Bastion 進行 SSH 跳板登入8080ALLOW192.168.150.130僅允許 Bastion 存取 Nginx 網頁服務
<img width="964" height="1008" alt="螢幕擷取畫面 2026-04-23 183551" src="https://github.com/user-attachments/assets/72f7cc28-1742-4f63-92f4-129fc4df5523" />

<img width="974" height="1006" alt="螢幕擷取畫面 2026-04-23 184848" src="https://github.com/user-attachments/assets/f15710a5-0c81-4a9f-b221-658227ec753a" />

## 5. Part D：故障演練
### 故障 1：<F1/F2/F3 擇一>
- 注入方式：在 app 機器執行 sudo ip link set ens33 down
- 故障前：執行 ip a 看到 ens33 狀態為 UP，且從 Host 端執行 ssh app 可正常登入
- 故障中：如截圖所示，執行 ssh app 出現 connect failed: No route to host。這是因為 Layer 2 (ARP) 無法解析目標 MAC 地址，導致 Layer 3 無法建立路由。
- 回復後：在 app 本體畫面執行 sudo ip link set ens33 up，連線恢復正常。
- 診斷推論：當出現 No route to host 時，我判斷這不是單純的應用服務掛掉。透過 ping 測試完全無回應，代表基礎網路鏈結已斷開

### 故障 2：<另一個>
（同上）
F3 - Docker 服務停止 (Service 故障)
注入方式：在 app 機器執行 sudo systemctl stop docker
故障前：從 bastion 執行 curl -I http://192.168.150.131:8080 回傳 HTTP/1.1 200 OK。
故障中：curl 回傳 Connection refused。這代表 TCP 三向交握封包到達了目的地，但該埠號（8080）沒有對應的進程在監聽。
回復後：執行 sudo systemctl start docker 並重啟容器，連線恢復。
診斷推論：此故障中 SSH 依然能通，證明網路層與防火牆規則（Port 22）均正常。唯獨 8080 被拒絕，因此判斷故障點在於特定的軟體服務（Docker）而非網路環境。

### 症狀辨識（若選 F1+F2 必答）
兩個都 timeout，我怎麼分？
使用 ping 指令 (最快速的區分點)
若為 F1 (網卡掛了)：ping 192.168.150.131 會完全失敗，出現 Destination host unreachable 或 Request timed out。因為介面 ens33 已經 down 掉，封包根本進不去目標主機。

若為 F2 (防火牆擋了)：ping 通常會通（除非 UFW 預設連 ICMP 封包都捨棄）。如果 Ping 會通但 SSH timeout，那 100% 是防火牆 Port 22 被阻斷。

2. 觀察 SSH 報錯細節
F1 症狀：如你先前截圖所示，會直接報 No route to host。這代表在網路路由表或 ARP 解析上就已經找不到路徑了。

F2 症狀：會顯示 Connection timed out。這代表「路是通的」，封包有發出去，但被對方的防火牆無聲無息地丟掉 (Drop)，導致發起端在那邊空等直到超時。

3. 檢查 ARP 表格 (arp -a)
F1 狀態：在 Host 端執行 arp -a，你可能會發現目標 IP 192.168.150.131 對應的 MAC 地址是空的或顯示 <incomplete>。介面 down 掉時，Layer 2 的廣播無法回應。

F2 狀態：MAC 地址通常還在。因為防火牆 UFW 主要運作在 Layer 3/4，不會影響 Layer 2 的 ARP 運作。
## 6. 反思（200 字）
這次做完，對「分層隔離」或「timeout 不等於壞了」的理解有什麼改變？
透過本次實驗，我對網路架構的「分層隔離」有了從理論到實務的跨越式理解。過去認為連線失敗（Timeout）即代表設備損壞，但在實作中發現，Timeout 其實是網路安全防禦的具體表現。

在「分層隔離」的架構下，我們將 app 伺服器置於 Host-only 的私有網段，這層物理上的隔離讓它免於直接暴露在外部威脅中。然而，這也帶來了維護上的挑戰，例如在下載 Docker 映像檔時，必須暫時打破隔離切換至 NAT 模式才能抓取資源。這種「刻意的不便」正是為了確保服務在運行時具備最高層級的安全控管。

最深刻的體會是對於 「故障不等於損壞」 的認知。在 Part D 的故障演練中，我學會了透過 ping、arp 與 SSH 報錯訊息來進行邏輯推理：

F1 (網卡停用) 導致的是 No route to host，代表底層路徑的徹底消失。

F2 (防火牆攔截) 導致的是 Connection timed out，這證明了網路路徑依然存在，只是被安全政策拒於門外。

這種診斷能力的提升，讓我理解到一名合格的資管人員不應只會下指令，更需具備剖析 OSI 模型各層級狀態的推理鏈，才能在複雜的雲端架構中精準定位問題。
## 7. Bonus（選做）
Dockerfile 各層解析：

FROM nginx:alpine：作為基礎層，提供最小化的 Nginx 執行環境。

COPY index.html ...：將含學號的靜態網頁放入容器。

EXPOSE 80：聲明容器內部監聽的埠號。

為什麼 COPY index.html 要放在 FROM 後面？

快取優化（Build Cache）：Docker 在建構時會由上而下檢查每一層。基礎環境 (FROM) 通常不會變動，而我們的 index.html 可能會頻繁修改。

將變動頻繁的指令放在越後面，可以確保在修改網頁內容後重新 build 時，Docker 能直接使用前面基礎層的快取，只需更新 COPY 那一層，大幅加快建構速度。<img width="995" height="1012" alt="螢幕擷取畫面 2026-04-09 022429" src="https://github.com/user-attachments/assets/603fe6b9-4eaa-42d4-8c58-ef5adbc48a01" />
<img width="974" height="1006" alt="螢幕擷取畫面 2026-04-23 195735" src="https://github.com/user-attachments/assets/e5158c13-aa42-46e8-b8fe-e4ef5631e66c" />

