---
title: "<Building Microservices Design Fine-Grained Systems> 筆記"
date: 2022-10-11T15:56:19+08:00
draft: false
tags: ["coding"]
---
# Introduction
這篇是 <Building Microservices Design Fine-Grained Systems> 的筆記。分成三個部分 Foundation, Implementation, People。是我認為的 Microservices 我看過寫過最好的書。

# Foundation
> Microservices 和 SOA 的差異？
Microservices 本身是 SOA 的子集合，SOA 只說明了是由 Service 組成(也就是 OS 的 Process)，但 Microservices 卻包含更多自己的設計原則。

> Microservices 主要觀念？
* 能夠獨立部署
* 圍繞在 Business Domain。像是傳統都是分成 3-Tier 但是 Microservice 一個團隊就是以商業邏輯為導向組織團隊。
* 自己管理 State，算是前面一點的技術實作概念(擁有自己的 Database)
* Size 能夠被放進自己的大腦裡面，意味著邏輯並不會過度抽象與複雜。

> Microservices 導入的技術？
1. Log Aggregation & Distributed Tracing: 當服務拆分的時候必須要能夠在分散式的環境統一遙測
2. Container & k8s: 加速整體的生命週期和管理方面的問題
3. Streaming: 分散式的非同步溝通

> Microservice 的 Boundary 要考慮什麼？
* Information Hiding
    * 獨立部署
    * 服務模組本身可以單獨理解其邏輯
    * 對內可以改變實作的方式
* Coupling 的方式    

> Microservices 彼此如何 Coupling?
(以下由鬆到緊)
1. Domain Coupling
需要看 Interact 來彼此溝通。比如說 Order Processor 需要告訴 Warehouse 保留庫存，告訴 Payment 收錢。這種不可避免的，不然就不叫 Microservice 了。
2. Pass-Through Coupling
Microservice 需要其他微服務的資料，經由另一個服務 Pass-Through 資料。糟糕之處就在於當下游的服務需要更動 Interface，比如說今天 Shipping 要採用新的計算格式好了，Order 和 Warehouse 都要進行更動。解法有兩個，第一就是直接由 Order 直接溝通兩個下游服務，但是這樣的邏輯會變得非常複雜，要先保留商品，然後寄送，兩件事情成功之後還要去 Warehouse 扣除。

另一種解法就是，一樣是靠 Warehouse pass-through 但是以 blob 的方式傳送，完全不去更動內部的資料。
3. Common Coupling
這個情況發生在兩個服務需要用到相同的資料，比如說像 Order 和 Warehouse 都需要對訂單的狀態進行修改。最經典的解法就是引入 FSM 把不合法的狀態拒絕，但是這種狀態代表 code 的 cohesion 不夠，所以要進行重新設計。

4. Content Coupling
請直接避免這種問題！服務繞過商業邏輯，直接修改內部狀態，基本上會造成系統嚴重內部錯誤。

> 如何跨服務 Reference?

> Microservice 溝通模式？
* Synchronous blocking
定義：
優點：
缺點：
使用情境：
* Asynchronous nonblocking
定義：
優點：
缺點：
使用情境：
* Request-response
定義：
優點：
缺點：
使用情境：
* Event-driven
定義：
優點：
缺點：
使用情境：
* Common data
定義：
優點：
缺點：
使用情境：


# Implementation



