# W04｜Linux 系統基礎：檔案系統、權限、程序與服務管理

## FHS 路徑表

| FHS 路徑 | FHS 定義 | Docker 用途 |
|---|---|---|
| /etc/docker/ | 系統設定檔 | Docker daemon 設定 |
| /var/lib/docker/ | 程式跑的持久性資料 | 資料根目錄：images、containers、volumes |
| /usr/bin/docker | user可執行檔 | Docker CLI 執行檔 |
| /run/docker.sock | 暫存 | Docker daemon socket |

## Docker 系統資訊

- Storage Driver：overlayfs
- Docker Root Dir：/var/lib/docker
- 拉取映像前 /var/lib/docker/ 大小：368K
- 拉取映像後 /var/lib/docker/ 大小：372K

## 權限結構

### Docker Socket 權限解讀）  
srw-rw---- 1 root docker 0  4月 16 15:36 /var/run/docker.sock  
socket檔案類型，擁有者、群組成員有讀寫權，其他人都沒有，擁有者是root，群組是docker


### 使用者群組
uid=1000(wdc) gid=1000(wdc) groups=1000(wdc),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),100(users),114(lpadmin)


### 安全意涵
只要是docker group，就能透過 Docker daemon的敏感檔案掛載進容器並讀取。代表可以使用Docker幾乎等於有root權限。安全示範中輸出的是各個帳號的密碼雜湊，通常可以看到這些內容只有root才能讀取

## 程序與服務管理

### systemctl status docker
（貼上 `systemctl status docker` 輸出）
```
● docker.service - Docker Application Container Engine
     Loaded: loaded (/usr/lib/systemd/system/docker.service; enabled; preset: enabled)
     Active: active (running) since Thu 2026-04-16 15:36:39 CST; 2h 1min ago
TriggeredBy: ● docker.socket
       Docs: https://docs.docker.com
   Main PID: 1788 (dockerd)
      Tasks: 15
     Memory: 136.9M (peak: 153.2M)
        CPU: 2.889s
     CGroup: /system.slice/docker.service
             └─1788 /usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock
```

### journalctl 日誌分析
（貼上 `journalctl -u docker --since "1 hour ago"` 的重點摘錄，說明看到什麼事件）
```
 4月 16 17:17:19 UbuntuCVT dockerd[1788]: time="2026-04-16T17:17:19.003039678+08:00" level=info msg="sbJoin: gwep4 ''->'350d949afe14', gwep6 ''->''" eid=350d949afe14 ep=romantic_morse net=bridge nid=61bae7df7c35
 4月 16 17:17:19 UbuntuCVT dockerd[1788]: time="2026-04-16T17:17:19.073644423+08:00" level=info msg="received task-delete event from containerd" container=d466da1b28460c2d42a352c07452ebddaa29101887b5a0db5eeb176a09aa26ca module=libcontainerd namespace=moby topic=/tasks/delete type="*events.TaskDelete"
```
Docker 在建立/啟動容器時，將容器的 network sandbox 加入預設的 bridge 網路
### CLI vs Daemon 差異
CLI只是一個命令列工具，只會在當下的那一條指令執行完就沒了；Deamon則是常駐服務，用root權限運作，而docker --version 正常只代表CLI可以正常使用，不用連到Deamon

## 環境變數

- $PATH： /usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin

- which docker：/usr/bin/docker
- 容器內外環境變數差異觀察：容器是獨立的環境，環境變數由映像決定，不會使用host的設定

## 故障場景一：停止 Docker Daemon

| 項目 | 故障前 | 故障中 | 回復後 |
|---|---|---|---|
| systemctl status docker | active | 沒在動 | active |
| docker --version | 正常 | 正常 | 正常 |
| docker ps | 正常 | Cannot connect | 正常 |
| ps aux grep dockerd | 有 process | empty | 有process |

## 故障場景二：破壞 Socket 權限

| 項目 | 故障前 | 故障中 | 回復後 |
|---|---|---|---|
| ls -la docker.sock 權限 | srw-rw---- | srw- | srw-rw---- |
| docker ps（不加 sudo） | 正常 | permission denied | 正常 |
| sudo docker ps | 正常 | 正常 | 正常 |
| systemctl status docker | active | active | active |

## 錯誤訊息比較

| 錯誤訊息 | 根因 | 診斷方向 |
|---|---|---|
| Cannot connect to the Docker daemon | Deamon沒在跑，CLI連不上 | systemctl status docker看有沒有active |
| permission denied…docker.sock | Demon有在跑，但是權限不夠不能指揮到那邊 | 看權限有沒有在group裡面，用sudo |

（用自己的話說明兩種錯誤的差異，各自指向什麼排錯方向）

## 排錯紀錄
- 症狀：已將user加入group，但是再執行後還是要用sudo才能執行docker ps
- 診斷：id
- 修正：用su -- 重新登入
- 驗證：id看有沒有在docker group裡面

## 設計決策
（說明本週至少 1 個技術選擇與取捨，例如：為什麼教學環境用 `usermod` 加 group 而不是每次 sudo？這個選擇的風險是什麼？）
這樣不需要每次執行docker都要加sudo，風險是在安全性上會因為等於直接有root權限，因此在多人共用上要多加注意