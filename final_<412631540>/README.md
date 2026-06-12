# 期末實作 — <412631540> <許沛恩>

## 1. 架構總覽
<Mermaid 圖 + 一段話說明>
本實作建構了一個具備生產級安全加固與高可用性維運思維的多服務 Web 架構。前端採用基於 Python 輕量級網頁框架的 final-web 容器，後端則相依於 final-db (PostgreSQL 15) 資料庫容器。兩者透過獨立且隔離的 backend-tier 橋接網路進行內部通訊，並透過具名磁碟卷 postgres_data 確保資料持久化
<img width="212" height="259" alt="螢幕擷取畫面 2026-06-12 225853" src="https://github.com/user-attachments/assets/fac4efff-b1b0-4c66-84c8-ee5fe007ff22" />


## 2. Part A：底座與基準點
<ssh 證據 + 版本 + snapshot>
宿主機工具版本驗證：
Docker Version: 29.1.3
Docker Compose Version: v2.40.3

SSH 運維通道取證：已成功實施 sudo ufw allow 22/tcp 金鑰加固，並完成虛擬機快照（Snapshot）備份，建立健康的運維基準點。

## 3. Part B：Dockerfile 與快取
<Dockerfile + 兩次 build 對照>
本專案採用分層儲存（Layered Storage）與優化快取排序設計。透過將變動頻率極低的環境依賴檔 requirements.txt 獨立前置進行 COPY 與 RUN，再將頻繁修改的應用程式碼 app.py 置於後端。當進行二次建構時，Docker 檢查到相依套件層的雜湊值（Hash）未變，將直接使用快取（Cache Layer），極大縮短 CI/CD 部署時間。

### 為什麼聽 8080 不聽 80？
在 Linux 核心的安全機制中，1024 以下的連接埠（Well-known Ports，如 HTTP 80）屬於特權連接埠，只有具備 root (UID 0) 權限的進程才能依法監聽綁定。為了落實「最小特權原則（Least Privilege）」，我們在 Dockerfile 中使用了非特權使用者 shuamy (UID 1000) 來運行程式。因此，網頁程式必須選擇監聽 1024 以上的非特權連接埠（如 8080），再透過 Compose 在 L4 層級將外網的 8081 對應進來。

## 4. Part C：Compose 與資料持久化
<compose.yaml 重點 + 三段對照>
docker compose down vs docker compose down -v 的本質區別
docker compose down：
僅移除容器實體、虛擬橋接網路與暫時性的容器快取。掛載在宿主機物理路徑上的具名磁碟卷（Named Volume）擁有獨立的生命週期，此時不會被刪除。當再次執行 up 起飛時，資料庫會自動載入磁碟卷，包含學號 412631540 的資料表仍完好存在。

docker compose down -v：
帶上 -v 參數屬於毀滅性命令，Docker 會強制將宣告的具名磁碟卷（postgres_data）連同宿主機上的物理數據目錄一併永久銷毀。再次啟動時，資料庫只能重新初始化空白目錄，導致撈取資料時直接噴出 ERROR: relation "exam" does not exist 的中斷慘劇。
### down vs down -v

## 5. Part D：生產化加固
<權限驗證輸出 + cgroup 讀值對照表>
 docker-compose.yml 生產級安全配置，成功在 Linux 系統內核層級實現了進程權限階梯式隔離與核心硬體資源限流，有效防止潛在的容器逃逸與惡意阻斷服務攻擊（DoS）。
1. 安全加固與 cgroup 核心物理指標對照表
以下為透過 docker compose exec web cat /proc/self/status 與 /sys/fs/cgroup/ 虛擬檔案系統所撈取出的真實取證數據對照：
<img width="611" height="75" alt="螢幕擷取畫面 2026-06-12 230738" src="https://github.com/user-attachments/assets/c8a890c5-1a16-4f73-b3a9-f83a3ffe2a28" />


### yaml 的值怎麼對回 cgroup 檔案？
yaml 的值怎麼對回 cgroup 檔案？Docker 內部掛載的虛擬檔案系統 /sys/fs/cgroup/ 是 Linux 核心調度硬體資源的直通窗口。當我們在 YAML 宣告 cpus: '0.5' 時，內核會將其轉換為時間配額寫入 cpu.max。檔案內的兩個數字分別代表 [quota] [period]。在內核中，預設的週期週期（period）為 $100000 \mu s$（即 100 毫秒），依照比例乘以 $0.5$ 得到配額（quota）為 $50000 \mu s$，因此印出的物理值必定精準呈現 50000 100000。

## 6. Part E：故障演練
### 故障 1：<F1–F4 擇一>
- 注入方式：於終端機執行 docker stop final-db
- 故障前：執行 curl -i http://localhost:8081/ 能完美回傳 HTTP/1.1 200 OK，並印出學號與資料庫物理時間
- 故障中：執行 curl -i http://localhost:8081/ 回傳 HTTP/1.1 500 INTERNAL SERVER ERROR
- 回復後：執行 docker start final-db，再次敲門立刻秒彈回傳 HTTP/1.1 200 OK
- 診斷推論：此現象屬於典型的 L7 應用程式內部邏輯斷層。此時前端網頁容器（Web Container）與 TCP Port 傾聽皆健全存活，但網頁程式在執行 Python 業務邏輯、嘗試透過驅動程式連接後端資料庫時，因連線失敗拋出未捕獲的例外（Exception），最終由 Werkzeug/Flask 框架攔截並向客戶端拋出 500 內部伺服器錯誤

### 故障 2：<另一個>
（同上）

### 三症狀分層表（必答）
| 症狀 | 最可能的層 | 第一條驗證命令 |
| ---- | ---------- | -------------- |
| timeout | 網路路由與傳輸控制層 |ping <IP> 或 nc -zv <IP> <Port> |
| connection refused | 作業系統與監聽程序層 |ss -tlnp或 docker ps  |
| HTTP 503 |應用程式內部業務邏輯層  | docker compose logs |

## 7. 反思（200 字）
這學期從 VM 做到 production-ready 容器，「隔離」這個概念在 VM、namespace、
cgroup、權限階梯四個地方各出現一次——它們在防的東西一樣嗎？
這學期從虛擬機跨入 Production-Ready 容器架構，「隔離」這個核心概念貫穿了所有層級，但它們防範的威脅本質截然不同。

VM（虛擬機）是在硬體物理層實施強隔離，透過 Hypervisor 劃分獨立的核心與虛擬硬體，防範的是不同租戶間的 noisy neighbor 效應與虛擬化底座層級的系統崩潰；Namespace（命名空間）則是在 Linux 核心實施檢視邊界隔離，它像障眼法一樣讓容器以為自己獨佔了 PID 和網路棧，防範的是容器間的進程互相干擾與窺探；cgroup（控制組）是在 核心層級實施資源用量隔離，透過 CFS 與記憶體配額設定剛性邊界，防止單一容器因 Fork Bomb 或記憶體洩漏引發雪崩，進而拖垮整個宿主機；權限階梯加固則是在 進程執行期實施特權隔離，透過 USER 拋棄、NoNewPrivs 與 cap_drop 限制進程能調用的系統 API 能力，防範的是應用程式遭入侵後的容器逃逸與提權攻擊。

四者相輔相成，從硬體、環境、資源到權限構築成了現代雲端原生架構中不可或缺的「縱深防禦」體系
<img width="696" height="871" alt="螢幕擷取畫面 2026-06-12 143750" src="https://github.com/user-attachments/assets/d2fc3a54-80b5-43b5-810c-4e6e890d36d0" />
<img width="696" height="871" alt="螢幕擷取畫面 2026-06-12 135202" src="https://github.com/user-attachments/assets/b310e22d-cf5e-4c89-9136-1575e04f39ab" />
<img width="480" height="504" alt="螢幕擷取畫面 2026-06-12 225549" src="https://github.com/user-attachments/assets/839fc0f3-7b34-41fb-a4f0-034fad029769" />
<img width="480" height="504" alt="螢幕擷取畫面 2026-06-12 225450" src="https://github.com/user-attachments/assets/8a1aeb81-7768-49fb-808b-5381c4f5403f" />
<img width="480" height="504" alt="螢幕擷取畫面 2026-06-12 225344" src="https://github.com/user-attachments/assets/3ef96eab-afc3-476b-8080-85904d4479f8" />
<img width="960" height="1008" alt="螢幕擷取畫面 2026-06-12 144701" src="https://github.com/user-attachments/assets/2d0f5711-595a-4c76-a97f-185ff07456ae" />
<img width="696" height="871" alt="螢幕擷取畫面 2026-06-12 134628" src="https://github.com/user-attachments/assets/1c772de9-ec01-4b2e-b856-cdefb2437dda" />
<img width="696" height="871" alt="螢幕擷取畫面 2026-06-12 135009" src="https://github.com/user-attachments/assets/fc65ee2a-2237-41d0-bd30-8a2497db3227" />


## 8. Bonus（選做）
把 Part B 的 Dockerfile 改成 multi-stage（W06 步驟 22 的做法）：builder stage 裝依賴，runtime stage 用 python:3.12-slim 只 COPY --from=builder 搬產物。

docker build -t final-app:multi .
docker images | grep final-app
交付：multi-stage 版 Dockerfile、單階段 vs 多階段的 SIZE 對照、一段話解釋 builder 層去了哪裡（提示：docker images -a 還找得到它們）。服務要能用新 image 正常起來。
① multi-stage 版 Dockerfile
遵循 W06 步驟 22 的生產級優化規範，將建置流程拆分為負責編譯的 builder 階段與專注於最小化運行的 runtime 階段，使編譯工具不殘留於最終映像檔中：

Dockerfile
# ==========================================
# Stage 1: Builder 階段 (負責編譯與安裝依賴檔)
# ==========================================
FROM python:3.12-slim AS builder

WORKDIR /build

# 安裝編譯所需的系統基本套件
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    && rm -rf /var/lib/apt/lists/*

COPY requirements.txt .

# 將 Python 第三方套件獨立安裝至用戶層根目錄下 (--user 會裝在 /root/.local)
RUN pip install --no-cache-dir --user -r requirements.txt

# ==========================================
# Stage 2: Runtime 階段 (生產環境最小化運行環境)
# ==========================================
FROM python:3.12-slim AS runtime

# 建立非特權系統使用者
RUN useradd -u 1000 -m shuamy

WORKDIR /app

# 核心精髓：僅從 builder 階段搬運安裝好的乾淨 Python 套件成品與二進位檔
COPY --from=builder /root/.local /home/shuamy/.local
COPY app.py .

# 調整專案目錄擁有者為非特權使用者
RUN chown -R shuamy:shuamy /app /home/shuamy

# 切換執行非特權安全用戶
USER 1000

# 確保 Python 運行時能正確讀取到從 builder 搬過來的第三方套件路徑
ENV PATH=/home/shuamy/.local/bin:$PATH
ENV PYTHONPATH=/home/shuamy/.local/lib/python3.12/site-packages:$PYTHONPATH

EXPOSE 8080

CMD ["python", "app.py"]


單階段 vs 多階段的 SIZE 對照表
根據宿主機終端機現場執行 docker images | grep final-app 的物理取證數據，新舊版本的體積對照如下：
<img width="633" height="101" alt="螢幕擷取畫面 2026-06-12 232047" src="https://github.com/user-attachments/assets/0228b913-843e-4998-8a8f-6f8bd1391f2c" />


③ 學理思考：Builder 層去了哪裡？
💡 一段話深度解釋其去向：
在執行多階段建置時，由第一階段 FROM ... AS builder 所建立的中間層（Intermediate Layers）與編譯相依檔案，在最終產生 final-app:multi 映像檔時，並不會被打包進最終的產物中。這些遺留下來的建置層會轉化為無標籤的中間快取層（Dangling/Intermediate Layers），它們並未從硬碟消失，只要在宿主機敲入 docker images -a 命令，就能抓到這些顯示為 <none>:<none> 的中間快照層。它們留在宿主機底層的運維目的，是為了在下次執行 docker build 時，能直接為完全相同的 pip install 步驟提供極速的快取命中，在完美守護生產環境「最小化容量、最小化攻擊面」的同時，兼顧了開發階段的編譯高效率。

④ 服務正常運行驗證
將 docker-compose.yml 檔案中 web: 服務的映像檔更改為新編譯出來的 final-app:multi 標籤，並執行重啟：<img width="480" height="504" alt="螢幕擷取畫面 2026-06-12 231755" src="https://github.com/user-attachments/assets/8bce79de-2352-4955-9cef-26ccd777b080" />

