# W07｜Docker Compose 與資料持久化

## 拓樸圖
（mermaid 或 ASCII，標出 app、db、default network、db-data volume）

## 從 docker run 到 compose.yaml
（自己的話：你最有感的一個改善是什麼？）  
指令可以比較好的看，不用再回去一個一個翻要輸入哪個，也可以當作DNS來用，並且compose不只是把指令寫成YAML，同時也把容器的依賴一起管理

## 三種掛載對照
| 掛載類型 | 路徑（host） | 容器砍重起資料還在嗎 | 重啟容器資料狀態 | 適合情境 |
|---|---|---|---|---|
| named volume | /var/lib/docker/volumes/w07_db-data/_data | down會在，但用down -v不會 | 還在 | 需要可穩定的環境 |
| bind mount | ./app | 還在 | 檔案還在但程式有沒有在動要看app | 開發環境，可即時看到變化 |
| tmpfs | 無 | 不在 | 不在 | 需要有暫存資料快取的環境 |

## healthcheck 前後對照
| 寫法 | curl /healthz t=1s | t=3s | t=5s | t=10s |
|---|---|---|---|---|
| 只 depends_on | 0 | 503 | 503 | 200 |
| service_healthy | 0 | 200 | 200 | 200 |

觀察（自己的話）：app先動，但是db還沒準備好所以app連不到，才會先503再200

## 排錯紀錄
- 症狀：本週挺順利
- 診斷：本週挺順利
- 修正：本週挺順利
- 驗證：本週挺順利

## 設計決策
（為什麼 db 用 named volume 而不是 bind mount？為什麼不能在生產用 tmpfs 存資料庫？）  
1.因為bind mount會把host目錄直接映射進容器，會有權限、檔案差異、路徑差異等問題；換機器或環境時也不好保證相同，並且資料庫的目錄通常不會直接暴露給host，交給named volume比較安全。  
2.RAM，容器停止資料就會不見，不適合放在需要穩定的環境裡使用。