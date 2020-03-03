# EC2: Elastic Load Balancer with Auto Scaling

## 前置作業: 打包image
> 因為Auto scaling 會一直開關機器，彈性調整數量以配合需求，為了不要每次多一台機器又要ssh進去部署，所以我們先將映像檔準備好，這樣auto scaling開啟的機器直接就符合我們需要的環境。

1. 開一台EC2，將環境設置好
2. 在web console對該EC2右鍵->Create Image
![](https://i.imgur.com/8IxBxJo.png)
3. 可以在web console: EC2->Images->AMIs看到剛創好的Image
![](https://i.imgur.com/tzyCpM8.png)

## 創建 Elastic Load Balancer

0. 在Web Console: EC2->Load Balancing->Target Groups->Create target group，我這裡的Target type選Instance，其他沒變
![](https://i.imgur.com/BxrCIqJ.png)
2. 在Web Console: EC2->Load Balancing->Load Balancers->Create Load Balancer
3. 按照需求選擇Load Balancer種類（這裡我選HTTP/HTTPS）
![](https://i.imgur.com/eGF1Ah7.png)
4. Target groups選第0步創建的（或要新建也行）
5. 完成
![](https://i.imgur.com/SGmvQQt.png)

## 創建Auto-Scaling

1. 在Web Console: EC2->Auto Scaling->Launch configurations->Create launch configuration
2. 選擇一開始我們自己製作的AMI
![](https://i.imgur.com/05YHnlS.png)
3. 到*Configure details*時，點開Advanced Details，在User data打開機需要的script
![](https://i.imgur.com/XCraxJW.png)
4. 其他設定隨意，完成之後順便就直接Create an Auto Scaling group using this launch configuration
![](https://i.imgur.com/ROkiupZ.png)
5. 比較重要的是第二步: Configure scaling policies，這裡就照自己想要的policy和數量調整。Metric type有個選項是Application Load Balancer Request Count Per Target，但這裡還沒指定Load Balancer，所以不會有作用。也可以以CPU使用率或網路流量作為彈性調整的依據
![](https://i.imgur.com/tzxDjEG.png)
6. 完成Auto Scaling
![](https://i.imgur.com/WBDqWuo.png)
7. 搭配Load Balancer: 在該Auto scaling group右鍵->Edit，找到Target Groups，選擇Load balancer。
> Classic load balancer是當初創建Load Balancer時可以選的，但如果是選http/https或TCP/UDP都不用管這欄。
![](https://i.imgur.com/m0TS0aM.png)
8. 完成，這時可以更改Policy成Application Load Balancer Request Count Per Target。
![](https://i.imgur.com/9iFRCbs.png)


## Demo:
我寫了一個簡單的go http server，會print 該機器的IP（我以為它會抓公網的IP...結果只輸出內網的，但還是有分辨的效果）
造訪Load balancer的DNS，會輸出不一樣的IP，代表造訪的是不同的機器。
![](https://i.imgur.com/WOQ2z3y.png)
附上程式碼當備忘錄
```go
package main
import "net/http"
import "net"
import "log"
import "fmt"

func GetOutboundIP() net.IP {
    conn, err := net.Dial("udp", "8.8.8.8:80")
    if err != nil {
        log.Fatal(err)
    }
    defer conn.Close()

    localAddr := conn.LocalAddr().(*net.UDPAddr)

    return localAddr.IP
}
func getHost(w http.ResponseWriter, req *http.Request) {
        fmt.Fprintf(w, "%s", GetOutboundIP())
}
func main(){
        http.Handle("/", http.HandlerFunc(getHost))
        http.ListenAndServe(":80", nil)
}
```