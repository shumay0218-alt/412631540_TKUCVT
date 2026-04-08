# W02｜VMware 網路模式與雙 VM 排錯

## 網路配置

| VM | 網卡 | 模式 | IP | 用途 |
|---|---|---|---|---|
| dev-a | NIC 1 | NAT | （192.168.79.128） | 上網 |
| dev-a | NIC 2 | Host-only | （192.168.150.129） | 內網互連 |
| server-b | NIC 1 | Host-only | （192.168.150.128） | 內網互連 |

## 連線驗證紀錄

- [ ] dev-a NAT 可上網：`ping google.com` 輸出
- [ ] 雙向互 ping 成功：貼上雙方 `ping` 輸出
- [ ] SSH 連線成功：`ssh <user>@<ip> "hostname"` 輸出
- [ ] SCP 傳檔成功：`cat /tmp/test-from-dev.txt` 在 server-b 上的輸出
- [ ] server-b 不能上網：`ping 8.8.8.8` 失敗輸出
![ping 驗證] <img width="880" height="1008" alt="螢幕擷取畫面 2026-04-09 014838" src="https://github.com/user-attachments/assets/a84f7806-e0d9-41e2-9aea-efd292c9a465" />
![ping 驗證]<img width="870" height="1008" alt="螢幕擷取畫面 2026-04-09 014852" src="https://github.com/user-attachments/assets/ba3192c0-6a91-4929-9c20-8a0429ec6dc7" />
![ping 驗證]<img width="870" height="1008" alt="螢幕擷取畫面 2026-04-09 020244" src="https://github.com/user-attachments/assets/c614bb84-aa84-4dcc-88f7-9e0614455301" />
![ping 驗證]<img width="870" height="1008" alt="螢幕擷取畫面 2026-04-09 020252" src="https://github.com/user-attachments/assets/5a1db91d-f114-495b-bade-5c52bdba75ee" />
![ping 驗證]<img width="1021" height="1008" alt="螢幕擷取畫面 2026-04-09 021156" src="https://github.com/user-attachments/assets/c32b8f0d-ad90-48c5-9f52-f154bdc8a5a5" />
![ping 驗證]<img width="1017" height="1008" alt="螢幕擷取畫面 2026-04-09 021605" src="https://github.com/user-attachments/assets/3f7ab4a0-9876-47fc-8781-884c1c0e7987" />
![ping 驗證]<img width="995" height="1012" alt="螢幕擷取畫面 2026-04-09 022429" src="https://github.com/user-attachments/assets/d0989334-b60a-4d82-942e-0a95299b0fee" />
![ping 驗證]<img width="995" height="1012" alt="螢幕擷取畫面 2026-04-09 023004" src="https://github.com/user-attachments/assets/cbbb9df4-5d85-4f84-979e-d941c9512f2c" />


  

## 故障演練一：介面停用

| 項目 | 故障前 | 故障中 | 回復後 |
|---|---|---|---|
| server-b 介面狀態 | UP | DOWN | （UP） |
| dev-a ping server-b | 成功 | 失敗 | （成功） |
| dev-a SSH server-b | 成功 | 失敗 | （成功） |

## 故障演練二：SSH 服務停止

| 項目 | 故障前 | 故障中 | 回復後 |
|---|---|---|---|
| ss -tlnp grep :22 | 有監聽 | 無監聽 | （有監聽） |
| dev-a ping server-b | 成功 | 成功 | （成功） |
| dev-a SSH server-b | 成功 | Connection refused | (成功） |

## 排錯順序
（寫出你的 L2 → L3 → L4 排錯步驟與每層使用的命令）
當連線發生問題時，我遵循由下而上（Bottom-up）的順序：
L2 (Data Link)：使用 ip link 檢查網卡狀態是否為 UP。
L3 (Network)：使用 ping <IP> 檢查兩端網路路徑是否通達、ip route 檢查路由。
L4 (Transport/Application)：使用 ss -tlnp 檢查服務是否在 Port 22 監聽，並確認防火牆（ufw）狀態

## 網路拓樸圖
（嵌入或連結 network-diagram.png）


## 排錯紀錄
- 症狀：從 dev-a 執行 SSH 連線至 server-b 時出現 Connection refused
- 診斷：
首先執行 ping 192.168.150.128，發現網路是通的 (L3 OK)。
切換至 server-b 執行 ss -tlnp | grep :22，發現沒有任何服務在監聽。
- 修正：執行 sudo systemctl start ssh 重新啟動 SSH 服務
- 驗證：回到 dev-a 重新執行 ssh 指令，成功登入

## 設計決策
安全性（Security）：模擬真實企業環境中的「後端資料庫」或「內部主機」，這些機器不應直接曝露在互聯網，避免被外部攻擊。
管理規範：所有對 server-b 的操作必須透過 dev-a（跳板機/管理機）進行，方便進行流量控管與操作審計。
環境隔離：確保測試環境不會意外連向外網下載未經審核的內容
