# W06｜Docker Image 與 Dockerfile

## 映像組成
- Layers 是什麼：
是一層一層疊起來的唯讀檔案系統，紀錄每次操作對檔案的不同部分，可以被多個image共用，不會重複得占用到硬碟空間
- Config 是什麼：是一個JSON檔，內容是有關環境變數目錄及user等等部分
- Manifest 是什麼：綁定上述兩個以及記錄每層layer對應的資料，讓docker知道怎麼設定image

## python:3.12-slim inspect 摘錄
- Config.Cmd：python3
- Config.Env：
```
"PATH=/usr/local/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
                "LANG=C.UTF-8",
                "GPG_KEY=7169605F62C751356D054A26A821E680E5FA6305",
                "PYTHON_VERSION=3.12.13",
                "PYTHON_SHA256=c08bc65a81971c1dd5783182826503369466c7e67374d1646519adf05207b684"
```
- Config.WorkingDir：沒看到?
- RootFS.Layers 數量：4

## Layer 快取實驗
| 情境 | build 時間 |
|---|---|
| v1 首次 build | 5.718s |
| v1 改 app.py 後 rebuild | 5.290s |
| v2 首次 build | 5.118s |
| v2 改 app.py 後 rebuild | 0.347s |

觀察（用自己的話寫）：為什麼 v2 的 rebuild 這麼快？  
因為docker是用上一個結果+指令+copy檔案來算cache，V2單獨copy出來，即使改程式碼也不去影響到pip，所以才可以比較快。

## CMD vs ENTRYPOINT 實驗
| 寫法 | `docker run <img>` 輸出 | `docker run <img> extra1 extra2` 輸出 |
|---|---|---|
| CMD shell form | argv = ['show_args.py', 'default1', 'default2']<br>PID = 7 | exec: "extra1": executable file not found in $PATH<br>(exit code=127) |
| CMD exec form | argv = ['show_args.py', 'default1', 'default2']<br>PID = 1 | exec: "extra1": executable file not found in $PATH<br>(exit code=127) |
| ENTRYPOINT + CMD | argv = ['show_args.py', 'default1', 'default2']<br>PID = 1 | argv = ['show_args.py', 'extra1', 'extra2']<br>PID = 1 |

結論（用自己的話寫）：CMD在docker run的時候會去找extra1來當作要執行的程式，但是找不到所以會報錯

## Multi-stage 大小對照
| Image | SIZE |
|---|---|
| python:3.12（builder base） | 428M |
| python:3.12-slim（runtime base） | 45.4M |
| myapp:v2（單階段） | 48.1M |
| myapp:multi（多階段） | 44.8M |

解釋（用自己的話寫）：builder stage 的 layer 去哪了？  
他只把需要的東西拿過來，所以看起來沒有，但是它實際上可能還在機器裡的build cache裡

## .dockerignore 故障注入
| 項目 | 故障前 | 故障中 | 回復後 |
|---|---|---|---|
| du -sh . | 44K | 151M | 44K |
| build context 傳輸大小 | 129B | 129B | 129B |
| build 時間 | 0.363s | 0.386s | 0.363s |

## 排錯紀錄
- 症狀：有製造大量檔案，但build沒有明顯增加
- 診斷：用ls看有沒有dockerignore已經在目錄下
- 修正：最後發現是因為新版本的BuildKit已經擋下
- 驗證：再執行一次發現還是同樣情況，證明BuildKit有在作用

## 設計決策
（說明本週至少 1 個技術選擇與取捨，例如：為什麼 runtime 選 `python:3.12-slim` 而不是 `alpine`？）  
因為選擇完整版的檔案大小太大，另外如果使用alpine有可能會讓套件沒辦法安裝，要在build時重新跑一遍，這樣會無法觀察本次要做的實驗部分，所以選擇silm版本對於本次實驗來說是最好的折衷方案。