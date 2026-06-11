# W08｜容器生產實踐

## Healthcheck 故障測試
- 停 db 後幾秒被標 unhealthy：34s
- 對應的 log 訊息：`127.0.0.1 - - "GET /healthz HTTP/1.1" 503 -`

## Log 失控估算
- noisy 容器 30s log 大小：3291264960 bytes
- 預估 24h 大小：8827.9 GB
- 套 rotation 後穩定上限：4.6 MB

## 資源限制實驗
| 實驗 | 命令 | 觀察結果 | 對應 cgroup 檔 | 值 |
|---|---|---|---|---|
| OOM | stress-ng --vm 1 --vm-bytes 200m | exit 137, OOMKilled=true | memory.max | 134217728 |
| CPU throttle | stress-ng --cpu 4 | docker stats CPU% ≈ 50% | cpu.max | 50000 100000 |

## 權限四階對照
| 階梯 | id | CapEff | NoNewPrivs | curl /healthz |
|---|---|---|---|---|
| 0 | uid=0(root) | 00000000a80425fb | 0 | 200 |
| 1 | uid=1000(appuser) | 0000000000000000 | 0 | 200 |
| 2 | uid=1000(appuser) | 0000000000000000 | 0 | 200 |
| 3 | uid=1000(appuser) | 0000000000000000 | 0 | 200 |
| 4 | uid=1000(appuser) | 0000000000000000 | 1 | 200 |

## 排錯紀錄
- 症狀 : docker compose說服務起不來
- 診斷 : 發現有其他容器在跑沒關掉
- 修正 : 停止現有的容器
- 驗證 : 再跑一次可以正常運作

## 設計決策
app 設 mem_limit 256m、cpus 0.5：Flask + psycopg2 閒置約 40 MiB，實測後取約 1.5 倍以上作為緩衝，避免正常流量下就碰到上限。db 設 512m、1.0：PostgreSQL 相對更吃記憶體與 CPU，因此給較寬鬆的資源避免查詢與 healthcheck 抖動。read_only: true 之後補了 /tmp 與 /home/appuser/.cache 兩個 tmpfs：唯讀 rootfs 下這兩處若不開可寫區會噴 errno 30，但它們只是暫存，不需要落地保存，所以用 tmpfs 比 named volume 