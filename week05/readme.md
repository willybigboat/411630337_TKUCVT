# W05｜把容器拆開來看：Namespace / Cgroups / Union FS / OCI

## Docker 環境

- Storage Driver：overlayfs
- Cgroup Version：2
- Cgroup Driver：systemd
- Default Runtime：runc

## Namespace 觀察

### 六種 namespace 用途（用自己的話）
- PID： 讓容器以為自己是最大的，同時也看不到其他運行的process
- NET： 隔離的網路資源，IP、PORT、路由表、防火牆，自己的獨立虛擬網卡
- MNT： 自己的檔案管理系統，同時也看不到其他人以及HOST的檔案
- UTS： 每個容器都有自己的name，不影響HOST的
- IPC： process間的隔離，確保一個容器無法透過任何方式去干擾另一個容器的資料
- USER： 在容器內是ROOT，但是在HOST裡卻是一般使用者或是自訂使用者，即使在容器內已經被攻擊到被獲取ROOT權限，但是在HOST中可能啥都沒得到

### Host vs 容器 inode 對照
請見 [namespace-table.md](namespace-table.md)

### 容器內 `ps aux` 輸出
我在容器內執行 `ps aux` 只能看到少數幾個 process，是因為 Docker 啟動容器時建立了namespace。
namespace 會隔離，所以容器內只能看到同一個 PID namespace 裡的程序，看不到 host 上其他服務。
|PID|   USER  |   TIME | COMMAND |
|---|---|---|---|
|1| root |     0:00| sleep 3600|
|6| root |     0:00| sh|
|12| root |     0:00| ps aux|
 

## Cgroups 實驗

### 容器內讀到的限制
- memory.max：268435456
- cpu.max：50000 100000

### Host 端對照（用 `docker inspect -f '{{.HostConfig.CgroupParent}}'` 動態取得路徑）
- memory.max：268435456
- cpu.max：50000 100000
- memory.current（執行時某一刻）：524288

### OOM 故障三階段
| 項目 | 故障前 | 故障中（memory=32m + dd 200m）| 回復後（memory=256m）|
|---|---|---|---|
| 容器 exit code | - | 137 | 0 |
| OOMKilled | - | true | false |
| dmesg 關鍵字 | 無 OOM | `Memory cgroup out of memory / Killed process` | 無 OOM |

## Image 分層

### `docker image inspect nginx:1.27-alpine` layer 數量
 8
```
["sha256:08000c18d16dadf9553d747a58cf44023423a9ab010aab96cf263d2216b8b350"
"sha256:d71eae0084c1aa823dd8fb2ecf8604d5c0f4911226c042bb1f8297e819f4b192"
"sha256:c56f134d380585340a68d0db2f2c170641a1c0ff72ccf2438cf2f693df756a85"
"sha256:e244aa659f612a80c40dd8645812301e3def6b15ec67b9e486ed2201172b51d1"
"sha256:b8d7d1d2263425d6044e059b2810017d062d659b9b755241f3747eda77726250"
"sha256:811a4dbbf4a5309e4390cf655c12db92e1a4304fb9d9731f83e7b02e95a617c6"
"sha256:947e805a4ac71f68e6703550c0b36c2aa2e554c4fa670ca2da6a25c6d7dccb66"
"sha256:0d853d50b128aa460b47e7121849463a14b18d4fd976caf5014744aae24d28aa"]
```

### 兩個同源 image 共享 layer 的證據
都不同  
1.27
```
["sha256:08000c18d16dadf9553d747a58cf44023423a9ab010aab96cf263d2216b8b350"
"sha256:d71eae0084c1aa823dd8fb2ecf8604d5c0f4911226c042bb1f8297e819f4b192"
"sha256:c56f134d380585340a68d0db2f2c170641a1c0ff72ccf2438cf2f693df756a85"
"sha256:e244aa659f612a80c40dd8645812301e3def6b15ec67b9e486ed2201172b51d1"
"sha256:b8d7d1d2263425d6044e059b2810017d062d659b9b755241f3747eda77726250"
"sha256:811a4dbbf4a5309e4390cf655c12db92e1a4304fb9d9731f83e7b02e95a617c6"
"sha256:947e805a4ac71f68e6703550c0b36c2aa2e554c4fa670ca2da6a25c6d7dccb66"
"sha256:0d853d50b128aa460b47e7121849463a14b18d4fd976caf5014744aae24d28aa"]
```

1.26
```
["sha256:994456c4fd7b2b87346a81961efb4ce945a39592d32e0762b38768bca7c7d085"
"sha256:aad7be8b43d91f43cdc23af3440b13eea7c2957feec9c46c977cb256e92481f6"
"sha256:49c50d3fe9320c2fc37d1aee38488bad246a680333a20746a5ef63f21d074c67"
"sha256:ed2f467e1cfcfea2cff2f48b21b86e763979ee599591f3632b44899f26ce583b"
"sha256:6f197061abd698a3eaf862a101d043b50b9162024cdf830e7cfb75131a9f3725"
"sha256:51b6aefac2f5df9fa2c24d782ef818b0b96238af2511eb60f79a58d1c839513a"
"sha256:6dba76576010ad0450285be4d174f5084b0bf597a68f31f8ad597fab0f032f3d"
"sha256:a0636672c7fc32af4d1022152a8e32256abd648fb01f48f33023839e65c6d1cb"]

```


### `docker diff` 輸出範例與解讀
（貼上 A/C/D 實例並說明）
```
D /etc/nginx/conf.d/default.conf
A /etc/nginx/conf.d/custom.conf
C /etc/nginx/conf.d
```

解讀：
- `A` (Added)：新增檔案，在容器內 `echo hello > /tmp/hello.txt`，所以看到 `A /tmp/hello.txt`。
- `D` (Deleted)：刪除檔案，ex.我刪掉預設設定檔，所以看到 `D /etc/nginx/conf.d/default.conf`。
- `C` (Changed)：代表該路徑內容有變動（常見是目錄底下新增/刪除檔案，目錄本身就會被標成 Changed），例如我新增/刪除 `conf.d` 底下檔案，所以 `C /etc/nginx/conf.d` 會出現。

## OCI 呼叫鏈
 
- Docker 的主要 daemon，提供API，負責較高層的邏輯network/volume/build...，再把啟動的任務給 containerd做
- containerd主要負責容器的生命週期和管理image儲存，並產出一個符合OCI的json檔，最後去呼叫shim來啟動容器
- shim每個容器都有一個，確保他跟containerd脫離，保存stdio/exit code，即使containers重啟，容器也能正常運作不受影響
- OCI則負責最後的容器實作，會讀json檔，容器跑起來後就會退出

namespace : pid、network、ipc、uts、mount、user  
cgroup : resources、resources.unified

## 排錯紀錄
- 症狀：已經將記憶體放寬到265但是dd還是顯示error
- 診斷：發現是輸入的指令少了shm-size=256m
- 修正：將復原指令加上shm-size=256m
- 驗證：dd變為DONE，code為0

## 想一想（回答 3 題）
1. 容器裡的 PID 1 跟 host PID 1 是同一支 process 嗎？  
不是，host的是整台機器的，容器裡面的就是容器裏面的第一個，跟HOST是完全分開的  
`kill -9 1`（在容器內）會發生什麼？  
容器會直接停止

2. 兩個容器都基於 `ubuntu:24.04`，磁碟空間是吃兩份還是共用？ 
image會共用，因為每個容器都有一個可寫層，是額外的，所以在容器內操作時是寫在可寫層裡面的，有寫東西才會讓容量增加
怎麼驗證？
3. 如果 host 的 kernel 爆漏洞，容器還能稱為「隔離」嗎？
這個限制跟 VM 差在哪？  
容器主要透過namespace等機制來隔離，所以如果有漏洞可以鑽到HOST，隔離就失效了；而VM主要是硬隔離，就算有一台VM被攻擊破了，通常還要再攻擊hypervisor才會影響到另一個VM