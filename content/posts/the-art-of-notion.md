---
title: "<The Art of Notion>"
date: 2022-12-25T20:22:36+08:00
draft: false
tags: ["Engineering"]
---
{{<toc>}}

# 簡介

這篇文章會統整 Notion 兩篇非常重要的工程部落格，分別是 <The data model behind Notion flexibility> 和 <Herding elephants: Lessons learned from sharding Postgres at Notion>，其簡單扼要地講述了 Notion 背後的工程設計，我一向喜歡研究自己覺得很好用的工具怎麼設計的，就讓我們來看看吧。

# Notion 是啥？

對於不知道 Notion 的人我想還是有的 (如果你是這種人，我要特別謝謝你ㄟ哈哈，因為你應該是看到是我寫的才點進來 XD)。它可以說是一款功能非常強大的線上筆記軟體(甚至可以說是資料庫)，我會用來整理幾乎所有我目前在研究的東西。

甚至我寫這篇部落格的草稿也是在 Notion 上而不是 VScode。(雖然我沒收 Notion 錢，但是 Notion 真的是個好軟體)
![](/images/the-art-of-notion_0.png)

# Blocks

這邊建議可以先去創個 Notion 帳號，然後邊玩玩看邊搭配我的文章服用。應該不難發現 Notion 使用起來很直覺，你可以把資訊塊拉來拉去、重新組合。這些資訊小碎片，在 Notion 裡面稱作 Block，每一個小碎片都有以下五個屬性。

- ID：唯一的編號，就像每個 Block 的名字，用 UUIDv4 亂數生成的。
- Properties：一些屬性，像是 title 之類的，如果是比較特殊的 Block，Properties 就會長的比較不一樣。
- Type：這個 Block 是個什麼樣的 Block (如下圖，Type 就是 Text, Page, To-do list …etc)。
- Content：主要是一個 Array 紀錄這個 Block 裡面有什麼 Blocks。
- Parent：這個 Block 的 Parent，主要用來管理權限。
![](/images/the-art-of-notion_1.png)
## Dynamic Change

你可以發現 Notion 可以自由的變換顯示的方式，比如說原來的 H1 你可以換成 H2 或純內文，主要就是 Content 本身設計上和 Type 互相獨立，Properties 本身不會因為你更改 Type 而變，只是 render 的時候不會顯示出來。(有就意味著你把 Checkbox 改成 Header 然後改回來，打勾的屬性還是會記錄著)。

## Render Tree

Notion 屌的地方在於，除了多樣化自由變動的 Block，還可以無窮的巢狀去組織它。Notion 稱作個設計叫做 Render Tree，實作方式也蠻好理解的(對於 CS 背景的來說…)，用剛剛上面講的 ID 當作指標，指過去就好了。(其中原文有補充一點小細節，比較和單純 Tree Render 的差異，不過我覺得不是特別重要)

![](/images/the-art-of-notion_2.png)
## Indentation

至於 Indentation 則是直接操作  Render Tree 的結構，所以並不是單純的加上甚麼 Tab 字符，而是直接操作整個 Block。
![](/images/the-art-of-notion_3.png)
## Permissions

Notion 還有一個特點，就是可以分享 Blocks，最終他們使用參考 Parent 的方式，舉例來說，如果你要讀某行文字，你必須取得這行文字 Page 的權限。

一開始原本打算用多個 content arrays 去管理，但是這種方法導致 Blocks 互相參考會有模糊空間，所以最終還是採用 Parent 這種方式。細節上，只記錄 Parent 用來給 Permission Service 驗證，Parent 則指紀錄自己有哪些 Sub-Content。
![](/images/the-art-of-notion_4.png)
# Life of a block

## Creating and updating

一個 Block 的生命週期出始於 User 建立，後續的每一個 Block 操作都會被 Batch 起來，變成 Transaction，再看看是要 commit 還是 reject。舉例來說，如果你建造一個新的 Block 除了你增加的內容本身，還要改寫這個 Block 的 Parent，這些都要變成一個 Transaction。

對 Client 來說，當他們修改內容的時候，都會被 Cache 起來，實作上 Notion 有說用 SQLite 或 IndexedDB 做 Cache (取名叫做 RecordCache)，至於遠端的操作都會寫入 TransactionQueue 最終同步到遠端的資料庫，至於 TransactionQueue 則是實作在 SQLite 或 IndexedDB (看你是哪個平台)。

正常來說，TransactionQueue 都是空的，只要 Client 有任何操作，Transactions 會打到 /saveTransactions 這個 Endpoint。後續操作也沒有超出大家的想像，API server 就把這些操作做完回個 HTTP 200，然後執行下一個 TransactionQueue 的工作。

在背景還會做一些額外的工作像是 version history snapshots、Quick Find 的 Index、還有紀錄你做的任何改變到 MessageStore Service。

## Real-time updates

那和你分享 Block 那端的朋友是如何看到更新的呢？對於每個 Client 都會維護一個 long-lived WebSocket 到上面講的 MessageStore Service，當 Transaction 已經完成的時候，就會通知 MessageStore Service 上面所有訂閱的 Client 就會拿更新的資訊去更新自己 local cache。

## Reading blocks

這部分我覺得相對簡單，設計上不會一次全部都 load 進來，而是用 loadPageChunk API 把後續一些 Blocks 加載進來，然後用 React render。

# Sharding

上面就是概略的 Notion 模型，接下來我們進入一些更技術的環節，Sharding！會進到這個步驟其實是一個重要的里程碑，因為代表公司的使用人數已經到了一個數量級。工程上的轉折點則是當使用 Postgres VACUUM 開始出現問題的時候，Infra Team 會常常需要處理就可以開始準備 Sharding。

Notion 做終決定使用 application-level sharding，也就是在 Application Logic之內做 Sharding，雖然一開始有考慮 Citus 或是 Vitess 之類的產品，確實簡化了 Sharding 的過程，但是對於 Clustering 本身的邏輯卻變得不透明，所以還是選擇上述的方案。

當決定好 application-level sharding，接下來我們探討三個主要的決策。

## 要 Sharding 什麼 Data？

對於 Notion 來說，block Table 是最主要的核心，我們只需要 Sharding 特定的 Table，並且保存其餘資料的 Locality。

## 要怎麼 Partition？

必須要選擇一個特定的 partition key 去拆分資料，這個 key 要能夠均勻的分散在各台機器之上。因為 Distributed Transaction 和 Join 都是很用意被限制在 single host。

## 實際上要建立多少個 Shard，然後我們要怎麼管理？

這個問題其實蠻明確的，比較工程的觀點就是要怎麼 mapping logical shards 和 physical hosts。

接下來我們就一一討論解決方案

## Decision 1：以 block 為中心 Sharding
## Decision 2：用 workspace ID 來 partition
## Decision 3：能力綜合評估

大致有以下的評估重點

- Instance type：選 AWS 機器的重點是 IOPS 經由估算大致需要 60K 的 IOPS 。
- Physical 和 Logical shards 的數量：經由評估 RDS 的數量，為了確保 replication 的有效性，每個 table 最多 500G，每台機器則是不超過 10TB。
- 機器的數量：越多的機器固然要花越多成本去維護，但是系統也會變得更強大。
- Cost：要求 scale linearly，可以方便估算大小。

最終選擇了 480 個 logical shards，32 台實體機器，實際上的結構如下：

- Physical database (32 台)
    - Logical shards (15 per database，總共 480 個)
        - block table (總共 480 個)
        - collection table (總共 480 個)
        - space table (總共 480 個)
        - etc (不一一羅列)

接下來的解釋我覺得是本文中的精華，有點直覺的人可能會問 “480？”，我以為在計算機科學裡面都是用 2^n 之類的ㄟ！主要的因素是 480 有很多的因數！

- 2, 3, 4, 5, 6, 8
- 10, 12, 15, 16, 20, 24, 30, 32, 40, 48, 60, 80, 96, 120, 160, 240

舉個例子來說，如果選 512，當我需要調整實體機器數量的時候，要從 32 → 64，跳太多了，但是用 480，跳的間距就不是那麼大，所以選因數多的就是個不錯的選擇！
![](/images/the-art-of-notion_5.png)

所以在 database 上可以看到的 table 會是像 schema001.block, schema002.block。

# Migrating

當設計好 sharding，接下來就是實際的 Migrating，通常都是以下的步驟：

1. Double-write：同時寫入新的和舊的 database
2. Backfill：當 Double-write 開始執行後，把舊的資料搬到新的 database
3. Verification：確認沒有錯誤。
4. Switch-over：真正切換到新的 database

對於務實上 Double-write 有以下選項可以參考

1. 直接同時寫兩個 Database，但是非常容易造成資料不一致
2. Logical replication，用 Postgres 內建的功能去操作，但是限制上蠻大的，所以也沒有採用
3. 用 Audit log 紀錄寫入的操作，然後在新的資料庫用 catch-up script 去寫入

至於還有一些細節，像是 Verification 和 Backfilling 可以參考原文。