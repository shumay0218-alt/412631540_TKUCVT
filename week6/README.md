# W06｜Docker Image 與 Dockerfile

## 映像組成
- Layers 是什麼：Layers（唯讀層）是 Docker 映像檔的基石。每一條在 Dockerfile 中會更動檔案系統的指令（如 `RUN`, `COPY`, `ADD`）都會產生一個唯讀的層。Docker 利用 Union File System（如 overlay2）將這些層疊加，並透過寫入時複製（CoW）技術讓容器共享底層結構，達到節省空間與快速啟動的目的。
- Config 是什麼：Config 是一份 JSON 格式的元數據（Metadata），用來記錄映像檔的設定資訊。它不包含實際的檔案系統內容，而是定義了容器啟動時的預設行為，例如：環境變數（Env）、預設執行命令（Cmd/Entrypoint）、工作目錄（WorkingDir）以及對外暴露的埠號（Expose）。
- Manifest 是什麼：Manifest 是一個索引清單（JSON 檔），負責將 Layers 與 Config 組合在一起。它指向了該映像檔所包含的 Config 檔案名稱，以及所有 Layers 的 SHA256 內容定址（Content Addressable）雜湊值。在 Multi-architecture（多架構）映像檔中，Manifest 亦負責引導不同 CPU 架構（如 amd64, arm64）下載對應的映像檔。

## python:3.12-slim inspect 摘錄
- Config.Cmd：`["python3"]`
- Config.Env：```text
  [
    "PATH=/usr/local/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
    "LANG=C.UTF-8",
    "GPG_KEY=716960DF42B4DD6502953290DE21128A31229AB7",
    "PYTHON_VERSION=3.12.13",
    "PYTHON_SHA256=c08bc65a81971c1dd5783182826503369466c7e67374d164519adf05207b684"
  ]
- Config.WorkingDir：""
- RootFS.Layers 數量：4 層

## Layer 快取實驗
| 情境 | build 時間 |
|---|---|
| v1 首次 build | （45.112s） |
| v1 改 app.py 後 rebuild | （56.655s） |
| v2 首次 build | （14.086s） |
| v2 改 app.py 後 rebuild | （0.359s） |

觀察（用自己的話寫）：為什麼 v2 的 rebuild 這麼快？
在 v1 的設計中，我們將經常變動的程式碼與不安定的安裝指令綁在同一個脈絡。當使用 sed 改動 app.py 後，由於 COPY app/ . 這一層的檔案雜湊值改變，導致 Docker 快取直接失效（Cache Miss）。根據 Docker 的連鎖失效特性，該步驟之後的所有指令（包含耗時的 RUN pip install）都必須強制作廢重跑，因而 rebuild 依舊耗時將近一分鐘。
而在 v2 中，我們利用了優化排序，將幾乎不變動的 requirements.txt 獨立提早複製並先行執行 pip install。當再次改動 app.py 時，安裝套件的層成功命中 Using cache，只有最後一哩路的程式碼複製被更新，成功讓建置時間瞬間從 56 秒暴跌至 0.359 秒，實現秒殺。

## CMD vs ENTRYPOINT 實驗
| 寫法 | `docker run <img>` 輸出 | `docker run <img> extra1 extra2` 輸出 |
|---|---|---|
| CMD shell form | （argv = ['show_args.py', 'default1', 'default2']


PID = 7） | （argv = ['show_args.py', 'default1', 'default2']


PID = 7） |
| CMD exec form | （argv = ['show_args.py', 'default1', 'default2']）PID = 1 | （docker: Error response from daemon: ... exec: "extra1": executable file not found in $PATH.） |
| ENTRYPOINT + CMD | （argv = ['show_args.py', 'default1', 'default2']


PID = 1） | （argv = ['show_args.py', 'extra1', 'extra2']


PID = 1） |

結論（用自己的話寫）：Shell Form (PID != 1)：底層其實是呼叫 /bin/sh -c 來執行命令，導致我們的 Python 程式被包在 shell 的子程序中。這不僅會盲目忽略外部帶入的任何參數（如 extra1 extra2 直接被無視），更嚴重的缺省是它無法接收主機傳來的 Linux 訊號（如 SIGTERM），會造成容器在關閉時無法優雅停機。

Exec Form CMD 的致命傷：一旦後面尾隨了外部參數，整個預設的 CMD 都會被強行「全部覆蓋」。當輸入 extra1 extra2 時，Docker 會誤以為 extra1 是可執行指令而引發找不到檔案的崩潰。

最佳實踐 (ENTRYPOINT + CMD)：將不變的執行主體（python）定死在 Exec Form 的 ENTRYPOINT，動態參數留給 CMD。這樣一來，外部傳入的引數會體面地覆蓋 CMD 並追加於主體之後，且維持最安全的 PID = 1 核心主程序。

## Multi-stage 大小對照
| Image | SIZE |
|---|---|
| python:3.12（builder base） | （1.02GB） |
| python:3.12-slim（runtime base） | （146MB） |
| myapp:v2（單階段） | （157MB） |
| myapp:multi（多階段） | （151MB） |

解釋（用自己的話寫）：builder stage 的 layer 去哪了？
多階段建置完成後，在第一階段（builder stage）所產生的編譯器相依套件與快取雜物，在最終的 Runtime 映像檔中完全不會被上傳或保留。在本機端，那些中間產物會被 BuildKit 的 cache 結構保留在本機硬碟中，以便下一次能加速建置（透過 docker images -a 可能會看到標籤為 <none> 的懸空映像檔）。然而，當我們將 Image 進行部署、推送（Push）到儲存庫時，只會傳輸輕量的 Runtime 階段，徹底在生產環境做到了體積瘦身。
## .dockerignore 故障注入
| 項目 | 故障前 | 故障中 | 回復後 |
|---|---|---|---|
| du -sh . | （44K） | （151M） | （151M） |
| build context 傳輸大小 | （9.72kB） | （157.3MB） | （12.29kB） |
| build 時間 | （0.42s (cached)） | （2m53.786s） | （0.38s (cached)） |

## 排錯紀錄
- 症狀：多階段容器（Dockerfile.multi）啟動後狀態顯示為 Up，但是當從 Host 執行 curl http://localhost:8080/ 或是連線指定埠口時，遭遇 Connection refused 拒絕連線。
- 診斷：利用 docker logs 追查，發現 Flask 預設監聽在 127.0.0.1:5000。在隔離的容器網路中，127.0.0.1 只對容器內部自己有效，外面 Host 的連接埠映射（Port Forwarding）流量根本無法敲門。
- 修正：在程式碼 app.run() 中，加入並指定外部綁定參數 host="0.0.0.0"，且將監聽埠改為 port=80，確保其面向容器內的所有虛擬介面。
- 驗證：重新進行 multi 建置後，以 docker run -d -p 8080:80 myapp:clean 開機，成功透過 curl 噴出正確的 Hello from [Container ID] | version=final 網頁訊息。

## 設計決策
（說明本週至少 1 個技術選擇與取捨，例如：為什麼 runtime 選 `python:3.12-slim` 而不是 `alpine`？）C Extensions 相依地獄與建置時間：Alpine Linux 為了追求極致輕量，底層採用的是 musl libc，而絕大多數 Python 的主流套件（包含未來的資料科學套件如 numpy、pandas 等）在官方預編譯的二進位 Wheels 檔中，都是基於 glibc（Debian/Ubuntu 體系）打包的。一旦在 Alpine 上安裝，將會因為不相容而被迫在建置當下「現場從原始碼編譯」，這需要大舉安裝 gcc 等編譯工具，反而會使 build time 暴增、最終 Image 體積不減反增。

生產環境（Production）穩定性考慮：musl libc 在處理多執行緒（Threading）的高併發網路解析與 DNS 邊緣案例時，偶爾會有與標準 Linux 行為相左的隱藏 Bug。為了長遠應用的擴充性與在企業部署時的環境高穩定度，選用基於 Debian 裁剪的 slim 映像檔，是兼顧「體積」與「健壯性」的最佳工程取捨。
<img width="949" height="1008" alt="螢幕擷取畫面 2026-06-03 005316" src="https://github.com/user-attachments/assets/96ee7db6-180d-4e2e-b9cc-d2d8543826b8" />
<img width="949" height="1008" alt="螢幕擷取畫面 2026-06-03 005251" src="https://github.com/user-attachments/assets/ec43baaa-108b-4a06-9abb-8d65a4ca5543" />
<img width="949" height="1008" alt="螢幕擷取畫面 2026-06-03 005209" src="https://github.com/user-attachments/assets/3a0a6a0c-9afd-4a86-b7a1-4017a111fe1a" />
<img width="949" height="1008" alt="螢幕擷取畫面 2026-06-03 005115" src="https://github.com/user-attachments/assets/aed65214-3661-49d7-8ae3-c1fb2e960efe" />
<img width="949" height="1008" alt="螢幕擷取畫面 2026-06-03 005045" src="https://github.com/user-attachments/assets/995e70ef-d844-488d-8145-407cba3973a8" />
<img width="949" height="1008" alt="螢幕擷取畫面 2026-06-03 004534" src="https://github.com/user-attachments/assets/33837ee8-494f-4f0d-b077-ae7b8eecacc2" />
<img width="949" height="1008" alt="螢幕擷取畫面 2026-06-03 004519" src="https://github.com/user-attachments/assets/adef5278-9087-4a13-a6ab-20def07d6e7a" />
<img width="949" height="1008" alt="螢幕擷取畫面 2026-06-03 004342" src="https://github.com/user-attachments/assets/7ee6973c-5bb5-4091-89fd-7432a552220d" />
<img width="949" height="1008" alt="螢幕擷取畫面 2026-06-03 004258" src="https://github.com/user-attachments/assets/d7f2f02b-a17e-48d5-a362-c5735e5c3eee" />
<img width="949" height="1008" alt="螢幕擷取畫面 2026-06-03 003950" src="https://github.com/user-attachments/assets/ac16e549-6420-4627-8bde-f31bac4a2e9c" />
<img width="949" height="1008" alt="螢幕擷取畫面 2026-06-03 003759" src="https://github.com/user-attachments/assets/598cf10b-a9b2-4ed5-8409-2019659e263a" />
<img width="949" height="1008" alt="螢幕擷取畫面 2026-06-03 003707" src="https://github.com/user-attachments/assets/4e6e6028-0be9-4155-a6da-5ec94b9ba29f" />
<img width="949" height="1008" alt="螢幕擷取畫面 2026-06-03 003406" src="https://github.com/user-attachments/assets/ec4de458-aa95-4e5d-a9c9-62f9a11ce40f" />
<img width="949" height="1008" alt="螢幕擷取畫面 2026-06-03 003310" src="https://github.com/user-attachments/assets/9ed1a3d9-62ea-43cc-b4bf-2e53a73be5dd" />
