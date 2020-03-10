---
tags: Cathay, intern
---
# [AWS] DMS：資料庫遷移（DB2 to Aurora）

## 前置作業

- DB2 server
- Aurora server

## Data Migration Service (DMS)

![](https://i.imgur.com/pPJNlMY.png)

### 創建 Replication instance

如圖所示，Replication instance負責將source的資料庫搬到target的資料庫。

![](https://i.imgur.com/Ft017yU.png)

就照著欄位填吧~

### 創建 Endpoints

Source 和 Target 都要~

#### Source (DB2)

![](https://i.imgur.com/sf1xrzM.png)

Server name填IP或domain都可以，其他也是照欄位填就好。
都填完之後，可以拉到最底下的**Test endpoint connection (optional)** 測試看看能不能用剛剛建立的Replication instance連線。
意外的，答案是不能，會報 ```the connection parameter CurrentLSN=LSN (where LSN is the current LSN) or CurrentLSN=<scan> must be set.``` 的錯，如圖
![](https://i.imgur.com/HDxVW0C.png)

照著錯誤訊息的說明，我們使用db2的工具來找出Current LSN。
根據[IBM的文件](https://www.ibm.com/support/pages/checking-lsn-size-db2-database)，在db2的user，輸入```db2pd -logs -db $database```，就可以看到Current LSN：
![](https://i.imgur.com/Yc7cZJe.png)

於是我們回到DMS，在*Endpoint-specific settings*一欄，輸入```CurrentLSN=0x00000000036F6CC0```（剛剛那行指令找到的LSN）
![](https://i.imgur.com/V39t6jU.png)

> 這裡可以下多個參數，用分號區隔

再重新測試，就可以看到successful的字樣。
![](https://i.imgur.com/h1KdS9l.png)
測試成功，就可以create endpoint了。

#### Target (aurora)

比起DB2，aurora就沒有什麼坑要避免，只要engine選對、帳密沒輸入錯，測試應該不會失敗，記得endpoint type要選target。
測試成功就一樣create endpoint，這樣就完成了兩個endpoint

### 創建 database migration task

前面都順利完成後，到這裡就只要填好前面創的那些東西就好。
![](https://i.imgur.com/Bf74KAw.png)

完成之後，它就會自動開始migration，這時候就耐心等候吧
在migration的途中，有可能會有某些table沒辦法成功遷移，這時候會顯示*Running with errors*，但DMS還是會繼續嘗試遷移其他table

![](https://i.imgur.com/VkP1SCt.png)

直到progress 達到100%才是真的完成
![](https://i.imgur.com/WWhOFMd.png)

