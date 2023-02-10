---
title: "<OLEP：Online Event Processing 系統設計>"
date: 2023-02-10T18:26:57+08:00
draft: false
tags: ["Engineering"]
---
![](/images/olep_head.png)
{{<toc>}}
# **導言**

今天分享的這篇論文 [<**Online Event Processing: Achieving consistency where distributed transactions have failed**>]() 是由頂頂大名的 **Martin Kleppmann** 所撰寫，**也就是大家熟知 DDIA 的作者 (下圖)，此書被譽為北美軟體工程師必看的一本書**。

而這篇論文是發表在 [**ACM Queue 期刊上的一篇論文**]()，[**主要的觀點是提出一種以 Event 的觀念來設計系統，也就是 OLEP：Online Event Processing。**]()

![](/images/olep_0.png)

# **前言**

傳統上開發都是圍繞著 OLTP，也就是把資料庫當成主要的核心。但是現在的環境越來越多 [**Heterogeneous storage 整合在系統中，Martin Kleppmann 就開始以 Event 的方式重新思考系統上的設計**。]()

# [**Applcation Architecture Today：多異質性儲存**]()

OLTP (online transaction processing) 在過去的幾十年是非常棒的應用，ACID 的保證讓邏輯可以順利運行應用程式，但隨著組織的擴大，越來越多異質的儲存系統使得 Transaction 越來越難管理。舉個實際例子大家應該會比較能夠理解，像是一個比較大型的公司可能會有以下基礎設施需要管理。

## [**Full-text search**]()

像是 user 搜尋 product catalog，我們需要用 full-text search index，雖然說有些 relational
databases (例如：Postgres) 有支援，但是到某個量級之後還是需要 Elasticsearch 這種專門處理 full-text search index 的系統。

## [**Data warehousing**]()

很多企業希望從 OLTP database 讀取資料到 warehouse 然後進行商業分析的決策，通常都是 column-oriented 的儲存方式，和 OLTP 取向十分不同。

## [**Stream processing**]()

還有 application 希望可以從 stream event 中追蹤使用者的行為，像是預防 fraud 或是 abuse。 

## [**Application-level caching**]()

為了加速 read-only 請求的效能 cache 是不可避免的。

## [**Example: Heterogeneous storage**]()

![](/images/olep_1.png)

上圖只是簡化版本的情境圖，**實際上一個使用者的請求可能觸發一大堆異質性的儲存系統 (就如上面所列)，[你不會希望 OLTP 成功寫入、然後 search index 寫入失敗；反之也是，這就是作者想要解決的問題。]()**

# [**Distributed Transactions**]()

作者細緻化了 transactions 的類別，方便後續的討論

## [**Homogeneous distributed transactions**]()

這裡指的是在單一的 Database 系統之內所提供的 transactions，[**像是 Google’s Cloud Spanner 和 VoltDB 都提供了很不錯的解決方式**，不過這些不是作者本文要討論的範疇。]()

## [**Heterogeneous distributed transactions**]()

異質性的 distributed transactions 就像上面講圖例所描述的，**我們要如何確保這些異質的系統可以達到 Transaction？**

作者有研究一下先前的解決方案例如 XA 之類的 protocol，但是發現務實上有太多問題了，我就不一一羅列 **(這部分大家有興趣自己去看論文聽作者抱怨 XD)**

# [**Event Logs**]()

接著作者就提出自己的解決方案，[**也就是把 RDBMS WAL 的概念抽取出來當 Application Layer 來用！**]()

**什麼意思呢？** 

如下圖所示，**當使用者發起一個請求，不會直接寫入任何一個儲存系統，而是先寫進一個 Log (在 Disk)，下游的儲存系統再依序的去做寫入的動作。**

![](/images/olep_2.png)

作者更進一步列出 Event Logs 的一些特性，**Log 本身寫入就有 atomic 的特性，只要寫入不論下游的系統是死是活，[最終都會讀取到 Log 的內容]()。[由於 Log 是 append-only，所有下游都會看到一樣的 Event 順序，這其實是一個很強的 Serialize 保證]()，下面我條列式的整理出內容。**

## **Log 的特性**

- **[Durable]()：Log 本身寫入 Disk 不會有遺失的問題**
- **[Append-only]()：新的 events 只能 append 在 Log 後端**
- **[Sequential reads]()：所有 subscribers 會看到相同的順序**
- **[Fault-tolerant]()：Log 本身可以做到 highly available**
- **[Partitioned]()：當 Log 超出單一機器的容量時可以採取分散式的方式拆分，不過拆分的 Log 互相之間就失去了 Sequential 的特性，要特別注意。**

## [**Subscribers 使用 Log 時的重點**]()

- subscriber [**可以依照 Events 維護自己的 state (例如：database)**]()
- 如果 subscriber [可能重複執行 event 就要考慮 **idempotent 的設計**]()
- [**subscriber 需要在 Disk 記錄自己的 LSN** (log sequence number)，**因為有可能處理 Event 到一半就 crush 了，那樣復原之後就不知道自己要處理哪個 Event，最好的方式就是在處理的同時一併 transaction 寫入 LSN**]() ，如下圖：

![](/images/olep_3.png)

![](/images/olep_4.png)

*[註記：]()這不是作者的論文，是我之前在別本書上看到的，正好符合這個主題，我拿來說明*   



# [**Example：Financial Payments System**]()

作者運用以上的概念給出了一個實際的系統案例，如下圖：

![](/images/olep_5.png)


## [**Stage-1. 轉帳請求**]()

當使用者想要轉帳給別的使用者，**她先發起一個 Event 到 payment request Log，這裡僅僅是表現出 intention (意圖)，不代表交易會成功！**

而這個 **Event 會拿到一個 ID 當作識別。**

## [**Stage-2. Single-threaded 處理**]()

**Single-threaded Payment Executor，這時會根據 Log 檢查相對應的邏輯**，比如說這個使用者的餘額還夠不夠阿，她要轉的帳戶存不存在之類的，這個過程是 Deterministically (確定性的)，很像是執行 stored procedure 的概念。

## [**Stage-3. 決策 & 處理**]()

Executor 如果決定通過這筆交易，就會寫入 local database，然後發出許多 events，至少會發出 outgoing 和 incoming 的 event。當然最前面的 Event ID 會一直跟著這些 event，系統就能追蹤發生甚麼事情。

## [**Stage-4. 系統回饋**]()

由於 Executor 訂閱了 source-account log，outgoing payment event 會重新回到 executor，可以再次檢查是否成功執行。

## [**Stage-5. 接受端**]()

如圖所示，destination account 基本上也可以做同樣的邏輯，根據 Event ID 就可以追蹤。

## [**Stage-6. 回報結果**]()

一開始發起請求的使用者會訂閱 source-account log，如果這一系列的處理都沒有問題的話，她就可以得到最終的處理結果，也完成了這一連串複雜的運作。

## [**Some notes: Scale**]()

在這個 payment 例子裡面，每一個帳戶都有獨立的 Log，我們不用把所有 Log 都放在同一個 node，這就意味著[**我們能夠 scale linearly，有多少使用者帳戶，我們單純幫他們開 node 加上 log 就可以了。**]()

以上就是 OLEP 的一個例子，接下來我們聽聽作者分析 OLEP 的優點和缺點。

# [**OLEP 的優點**]()

## [**Log 獨立性和擴充性**]()

由於 Log 指是紀錄一些發生的事實，獨立於 subscribers，這有甚麼好處呢？舉例來說，如果想要加一個功能推送通知簡訊，我可以直接加上一個新的獨立的 subscriber，如果想要加上 search index 或是 view，同樣也可以各自增加 subscriber，系統可以有很好的擴充性。

## [**可再維護性 & Debug**]()

如果 application 有 bug 發送有問題的 events，subscribers 自己可以把重寫邏輯把這些 event filter 掉，但是如果是直接用傳統的方式隨意 insertions, updates, deletes database，出錯了根本沒有方式去復原，因為只剩下了 state，可能只能採取 restored backup 這條路。

同樣的思路，有 Log 可以參考對於 debug 也有不錯的參考性。更務實一點來說，replay log 很快就能知道哪裡出錯，並且修正 bug。

## [**Data Modeling**]()

對於 Data Modeling，Log 會比隨意的更改 database 的狀態還要好。這在 Domain-Driven
Design 的社群中又稱做 event sourcing。他們的理由是比起 insert/update/delete 這些在 table 上的操作，用 Event 更有語意的精確表達。

[**舉例來說**]()：“學生退選一門課” 的 Event 明顯會比 ”enrollments table 刪除一個 row，然後增加一個 row 到 student feedback table” 來的更能讓人理解。

## [**Data Analysis**]()

從 event log 的角度來說，Data Analysis 的特性一定比 database 的 state 還來的有價值。舉例來說：e-commerce 的購物車，如果能夠得知使用者把購物車增加或清空的事件，更能去修正應用程式，更符合公司利益的目的。

## [**Distributed Transaction**]()

在分散式的事物當中，如果任何參與的 nodes 壞掉，所有的 transaction 只能 abort，但是 log 的方式則是有很多的 subscribers，其中一個 subscriber 壞掉並不會影響其他的 subscriber。

# [**OLEP 的缺點**]()

## [**Latency**]()

你可以想像上面的例子，雖然我們的設計保證了一定可以做到 Event 的任務，但是時間的上限根本沒辦法預測，這也是複雜系統的痛點。

## [**一致性問題**]()

假設一個使用者 reads 兩個不同的 data stores，而這兩個 data stores 又是由不同的 consumers 管理更新，很有可能就會讀取到不一致的資料。

這也是 OLEP 目前最大的瓶頸之一，如果使用者直接去 data stores 讀取資料 (而不是透過設計好的 Queue)，根本沒有辦法保證任何的 isolation level，作者也希望未來有更好的方法可以解決這種問題。

# [**心得**]()

這篇論文可以說是以更高層次的方式去觀看大型系統，比如說有好多個部門的公司，彼此有各自的基礎設施和應用，作者算是給我們一個方法論和繪景，讓我們知道擴充系統時的一些思路和資料管理的方式，不過這種方式不是萬能的不能套用在所有東西上，只是多了一種可以觀看世界的視角，這樣也挺不錯的！

