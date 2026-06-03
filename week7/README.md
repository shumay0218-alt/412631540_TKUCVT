# W07｜Docker Compose 與資料持久化

## 拓樸圖
（mermaid 或 ASCII，標出 app、db、default network、db-data volume）
<img width="273" height="126" alt="mermaid014925" src="https://github.com/user-attachments/assets/e44ab844-ac28-4785-827e-a2f0d774af9d" />

## 從 docker run 到 compose.yaml
（自己的話：你最有感的一個改善是什麼？）
我最有感的改善是「環境的一鍵啟動與基礎設施即程式碼（IaC）的確定性」。
以前用 docker run 時，必須手動先建網路、建磁碟卷，然後小心翼翼地處理 --network、-v、-e 的先後順序與命名，只要手殘打錯一個字，整個系統就連不起來。現在透過 Docker Compose，只要寫好 compose.yaml，網路和磁碟卷會自動綁定與建立，服務之間的依賴關係一目了然。再加上能將敏感密碼抽離至 .env，管理效率與安全性完全不是同一個級別。
## 三種掛載對照
| 掛載類型 | 路徑（host） | 容器砍重起資料還在嗎 | 重啟容器資料狀態 | 適合情境 |
|---|---|---|---|---|
| named volume | /var/lib/docker/volumes/w07_db-data/_data | 在 | 完全保留，資料無縫接軌 | 資料庫（Postgres/MySQL）、需要持久化的高 I/O 正式資料。 |
| bind mount |./app  |  在| 保持最新狀態，Host 修改容器內立即同步 | 開發環境（Source Code 掛載），方便即時修改程式碼與熱編編譯。 |
| tmpfs |  無（直接掛載於 Host 記憶體中）| 不在 |變回初始空狀態，重啟後清空  | 暫存快取（Cache）、Session 憑證、高機密敏感且不需落盤的暫存資料。 |

## healthcheck 前後對照
| 寫法 | curl /healthz t=1s | t=3s | t=5s | t=10s |
|---|---|---|---|---|
| 只 depends_on |503  | 503 | 503 | 200 |
| service_healthy | 000 (Refused) |200  | 200 | 200 |

觀察（自己的話）：
只使用 depends_on：Docker 僅保證 db 容器的行程被啟動，就立刻拉起 app。此時 Postgres 還在內部初始化，尚未準備好接受連線，導致前幾秒打 healthz 會得到由 Flask 回傳的 503 db unreachable。

加入 service_healthy：根據我的實測截圖，t=1s 時得到 000（Connection Refused），這是因為 app 嚴格等待 db 通過健康檢查，此時 app 還沒啟動，所以連接被拒。但在 t=2s 時 db 就緒，app 瞬間秒啟，之後全部直接回傳 200。這完美解決了服務啟動時的時序競爭（Race Condition）問題。

## 排錯紀錄
- 症狀：執行安裝指令 sudo apt-get install docker-compose-plugin 時，Ubuntu 系統回報 E: Unable to locate package docker-compose-plugin 錯誤，且執行 docker compose 顯示 unknown command
- 診斷：由於本機環境使用的是 Ubuntu 官方維護的軟體庫，而非 Docker 官方來源。在 Ubuntu 官方庫中，該外掛的套件名稱被命名為 docker-compose-v2，而非 docker-compose-plugin。
- 修正：將安裝命令更改為 sudo apt-get update && sudo apt-get install -y docker-compose-v2。
- 驗證：安裝完成後執行 docker compose version，成功印出 Docker Compose version 2.40.3，功能恢復正常。

## 設計決策
（為什麼 db 用 named volume 而不是 bind mount？為什麼不能在生產用 tmpfs 存資料庫？）
權限與檔案鎖衝突：Postgres 對底層資料夾的權限（UID/GID）、檔案鎖要求極高。Bind mount 會直接將 Host 的檔案系統權限帶入容器，若 Host 與容器內的 UID 不一致，極易導致 Permission denied 或資料損壞。Named volume 由 Docker Engine 統一託管在 Linux 原生檔案系統內，效能最佳且安全。

為什麼不能在生產環境用 tmpfs 存資料庫？

資料不具持久性：tmpfs 的資料完全儲存在記憶體（RAM）中。生產環境一旦遇到伺服器重啟、容器異常崩潰、或日常的升級維護（docker compose down），所有的商業資料與使用者帳號將會徹底蒸發，無法做到任何資料持久化。

<img width="976" height="1008" alt="螢幕擷取畫面 2026-06-04 014419" src="https://github.com/user-attachments/assets/603f1602-1f6e-401f-9e5f-dccd818e5438" />
<img width="976" height="1008" alt="螢幕擷取畫面 2026-06-04 014233" src="https://github.com/user-attachments/assets/1a58ee50-a41b-40a0-a9d3-36cb4cc83e9a" />
<img width="976" height="1008" alt="螢幕擷取畫面 2026-06-04 014522" src="https://github.com/user-attachments/assets/a22c2f43-354c-4863-a62e-8ace57ae6fcd" />
