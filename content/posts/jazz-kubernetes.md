---
title: "<Kubernetes: 一種爵士即興演出！>"
date: 2023-02-02T23:27:11+08:00
draft: false
tags: ["Engineering"]
---
![](/images/jazz-kubernetes_head.png)
{{<toc>}}

# **導言**

這篇文章會翻譯介紹 <[Core Kubernetes: Jazz Improv over Orchestration](https://blog.heptio.com/core-kubernetes-jazz-improv-over-orchestration-a7903ea92ca)> 這篇文章，主要講解 **kubernetes 的運作原理** (不是怎麼使用喔 XD)。[文章作者是 kubernetes 三位創始人之一的 **Joe Beda**]()，他用很精簡的語言和例子為我們分析 kubernetes 這樣的系統。

# **前言**

這篇文章會仔細講解 kubernetes 內部如何運作，[作者直接在第二句話就說，如果你是需要知道如何 Operate kubernetes 的人 (比如說安裝或是用 kubectl)，是不需要懂這些的 XD。這篇文章的受眾，是給那些想要深入了解 kubernetes 內部的人的。]()

# **預先知識**

由於這篇文章會採用大量篇幅的 kubernetes 專有名詞，所以至少希望你知道一些 [**kubernetes 的名詞的定義**]()，例如

- **Pod**
- **Node**
- **Kubelet**
- **…etc**

# **其實不是 Container Orchestration？**

後續的篇幅我們會介紹 kubernetes 是如何運作的，不過作者卻先做了一點名詞上的澄清。[如果你去 Google 查 kubernetes，多半會告訴你 kubernetes 是一種 **Container Orchestration**]()，[Orchestration 這個詞**像是有一個交響樂團的中心指揮家，指揮著樂團的演奏。**]()

雖然某種層面上來說 kubernetes 是 Container Orchestration 沒有錯，[**但是作者覺得比喻成 jazz improv 會更為恰當！**]()

# **kubernetes 系統和元件解釋**

![](/images/jazz-kubernetes_0.png)

以上這張圖就是 kubernetes 的概覽。

## **Datastore: etcd**

etcd 可以說是 kubernetes 的 state 心臟，負責儲存所有 kubernetes 最重要的資料。

對於不熟悉 etcd 的人，我簡單總結一下它的設計，[**etcd 是一種 distributed key-value database**，把 consistency 優先於 partition tolerance。]()

[*註記*]()：為了讀者閱讀的順暢性，大家可以想像 etcd 是資料庫就好，我會把作者有點離題的部分放在此文章末的附錄說明。

## **Policy Layer: API Server**

這個元件是 kubernetes 的心臟，**也是唯一會和 etcd 溝通的元件**。事實上作者認為，etcd 已經算是實作的細節了，[**理論上我們甚至可以換掉 etcd 變成其他的 storage system。**]()

API Server 本身是 policy component，會過濾請求的內容，禁止你直接接觸 etcd。API Server 主要圍繞著 resource，對外就是一個簡單的 REST API，[**簡單來說 API Server 只有做 create, read, write, update, watch 之類的任務**]()。

接著我們聊聊 API Server 有什麼特性的細節。

1. [**Authentication and authorization**]()：

Kubernetes 有一個 pluggable 的驗證系統，除了內部的驗證方式，還有可以外部化的託管方式。

2. [**Admission controllers**]()：

其次， API Server 不會照單全收請求，當 client 發出請求之後，會先經過內部的政策計算，**當然我們也可以藉由自己設計 controllers 來擴充 kubernetes 本身的功能。**

3. [**API versioning**]()：

由於 kubernetes 本身的版本不斷推進，API Server 本身也要支援不同版本的生命週期，至於資料則是靠 etcd 來保存。

[**在 API Server 中最重要的功能就是 watch！watch 的意思是當其中一項資源被更動了，正在 watch 這項資源的元件可以得知訊息，然後做出相對應的行為。**]()

# ****Business Logic: Controller Manager & Scheduler****

至於最後一片拼圖就能讓 kubernetes 整個活起來！[**我們有兩種 Servers 圍繞使用 API Server**]()，她們分別是 [**Controller Manager**]() 和 [**Scheduler**]()。作者提到，的確可以把這些東西全部寫進去一個超大的 binary 檔案，不過分開來設計也就為了未來的擴展性做出鋪路的動作。(雖然作者說這樣設計是一個歷史性的意外 XD)

## **Scheduler**

主要負責這些工作：

1. **看 etcd 中那些 Pods 還沒有被安排到 Node 上**
2. **檢查 cluster 的狀態** 
3. **選一個 node，符合請求的限制**
4. **部屬 Pods 到 Node 上面**

## **Controller**

同樣的 Controller 也會做和 Scheduler 差不多的事情。

1. **透過 watch API Server 得知目前 cluster 狀態**
2. **偵測到異常，然後重新佈署 Pods** 

舉例來說：當今天使用者部屬了 ReplicaSet (一種確保特定數量的 Pods 資源)，假設今天是設定為 3 好了，因為意外 Pod 死亡了變成 2，Controller 會 watch 這個資源，然後重新佈署 Pod，讓 Pod 數量穩定。

## ****Node Agent: Kubelet****

在工作的 Node 都會有代理，叫做 Kubelet 負責監控和調度這個 Node 的狀態，除了一些基本的工作，[最主要就是 watch 自己 Pods 的變化，並且回報 API Server，並且視情況管理 Pods。]()

# **Workflow**

接著我們看看實際上的運作流程吧！

![](/images/jazz-kubernetes_1.png)

這是一個 user 部屬一個 Pod 的流程例子。

1. **首先 user 先把請求發到 API Server，驗證過後就會把這個資源的狀態寫到 etcd。**
2. **scheduler 一直 watch API Server，會發現一個 “unbound” 的 Pod，經由 scheduler 演算法決定好部署計畫後，透過 API Server 把計畫寫回 etcd。**
3. **Kubelet 同時也一直 watch 有什麼新的 Pods 被分配到自己身上，一旦發現就透過 Docker 把容器給啟動。**
4. **Kubelet 的 monitors 會不斷回報給 API Server 更新裡面的 cluster states。**

# **結論**

kubernetes 的 API Server 就是所有的中心點，雖然說 wiki 和一些文件都會說是 orchestration，但是[**作者會認為用 jazz improv 來形容會更加貼切。**]()

[**我想是因為爵士樂的演奏並不像是大家看著同一張譜演奏自己的部分，而是根據別人的演奏來判斷自己該如即興演出！在上面的例子，我們可以發現其實這個模式和爵士樂的演奏真的比較相似，超有趣的！**]()

# **心得**

這篇短短的部落格，相比於 kubernetes 官網的文件可以說是短的可以，但是卻能啟發我的很多思考，其實這樣看來整個 kubernetes 本質上是一個 Pub/Sub 系統，元件彼此都在做自己的事情，[**與其說是 API Server，我更覺得像是 Message Queue 的 Topic/Subject。**]()

其中還隱藏了兩個重要的觀念作者沒有提到，就是 [**Command 和 Event**]()。

[**Command 本身就像是打 API 一樣，可能成功可能失敗；但是 Event 則是記錄下已經發生的事情，被保存在 Disk 上了，所以後續的元件可以依照 Event 繼續執行整個系統的任務。**]()

舉例來說，上面的故事 user 發完請求之後，東西被寫入 etcd，API Server 就掛點了，但是重啟之後其他人還是能夠從 API Server → etcd 繼續執行 user 想要的部屬行為。我想分散式系統的魅力就在此吧！透過一些紀錄和配合的 protocol 可以完成一些看似非常困難的任務！

# **附錄：chubby Families**

這邊就來聊聊 etcd，就像上面說的 [etcd 這種分散式資料庫把一致性的優先性拉的很高]()，這類的系統家族有 ZooKeeper 和一部分的 Consul，它們的祖師爺是 Google 的 [chubby](https://research.google.com/archive/chubby.html)，[這個家族通稱 “lock servers”，主要的目的是協調分散式系統的 coordinate 和 locking 問題。]()

其實仔細看，他們的 API 跟 File system 的設計很像，根本像是操作 File，對於 highly consistent 和 strict ordering 的極度要求讓使用者可以安心地用 atomic 的操作狀態機。

作者認為，kubernetes 很大一部份要感謝 etcd，raft 或是 paxos 都是很難的演算法，當 etcd 成功實作之後，kubernetes 可以基於此之上開發很多有趣的東西。