---
title: "<Discord 如何處理一天數億的訊息>"
date: 2023-01-21T16:06:39+08:00
draft: false
tags: ["Engineering"]
---
![](/images/discord-cassandra_head.png)
{{<toc>}}

# 前言

這篇部落格會翻譯和參考 <[How Discord Stores Billions of Messages](https://discord.com/blog/how-discord-stores-billions-of-messages)> 這篇部落格的內容做整理和翻譯。

我個人一向喜歡研究自己喜歡用的軟體，身為每天都會上 DC 和朋友社交的我來說，研究一下這麼厲害的服務，也是我的興趣之所在。

# 目標

這篇文章的目標也很明確，Discord **每天會需要處理一億的訊息 (100 million/day)**，這篇文章會解釋他們怎麼做到的。

# 故事開始

羅馬不是一天造成的，Discord 在 2015 年只花了兩個月就建立好了第一版。當時他們只用了一台 MongoDB 當作主要 Database，作者 (Discord CTO) 也直接講明了，**當時選用 MongoDB 純粹只是為了快速疊代產品，以最快的速度去打造市場想要的東西。從一開始就不打算用 Sharding 的 MongoDB，不僅複雜難用而且還不穩定。所以一開始使設計上就給出很大的移植特性。**(MongoDB QQ)

在此之上，作者提出 Discord 的核心文化…

> **Build quickly to prove out a product feature, but always with a path to a more robust solution**

# 遇到瓶頸

最初的設計是用 **(channel_id, created_at)** 對 Messages 組成 compound index，還不到一年的時間，已經保存了 100 million 的訊息量，開始會有很**明顯資料放不進去 RAM 的狀況**，這個時候就差不多要做 migrate database 的動作了。

# 選新的候選人

由於產品的特徵已經在先前的迭代過程被市場接受了，所以核心模型本身不會有太多的修正，這個時候就要條列式的把要解決的問題列出來。

## 問題

- Read/Write Ratio: 50/50
- 在語音服務的部分雖然不會有太多的訊息(甚至可以說是沒有)，**但是只要有 User 去 random seek，就會造成 Disk Cache Eviction**
- 私人訊息則是最主要的流量(每年大約從十萬到一百萬的數量級)，不過通常**只會讀去最近發送的訊息，而且數量很少讀取的量也相對低，所以不太容易在 Disk Cache**
- 超大型 DC 伺服器則是有幾千到數萬的成員，**隨便都是幾千則訊息，一年的量都是百萬等級在跳，所以大部分都會在 Disk Cache 裡面**
- 而且我們知道接下來的幾年，我們會開發一些**全文搜尋的功能，讓使用者可以搜尋關鍵字，然後查找大約 30 天以內的訊息，也就代表我們會有更多的 Random Read**

## 需求

接著我們可以基於以上的議題，更進一步地確認需求。

- **Linear scalability**：我們不想之後還要手動 sharding 資料
- **Automatic failover**：最好要有 Self-Heal 沒有人喜歡半夜被 on-Call
- **Low maintenance**：維護需要相對簡單，只要增加 Node 數量就可以應付未來增加的 User
- **Proven to work**：我們不想當第一個踩雷的公司…..
- **Predictable performance**：我們希望 API 的 95th 要在 80ms，還有我們不想用 Redis Memcached
- **Not a blob store**：顯然面對大量的讀取，我們不想把資料一天到晚 deserialize
- **Open source**：我們不想依賴第三方的資料庫公司。

綜合以上，我們選擇 Cassandra 作為我們新一代的基礎建設。

# Data Modeling

對於不熟悉 Cassandra 的人來說，可以想像是一個 **KKV Store**，**前面的 K 是 Partition Key 用來決定要放在哪一個實體的 Node**，**後面的 K 則是 Clustering key，可以想像成 Relational Database 的 Primary Key 會用來做 Order** (這樣 Scan 的操作會很有效率)。

還記得我們上面提到的 Message 是用 **(channel_id, created_at)** 做 index。**channel_id 可以作為 Partition key**，所以單一的頻道可以在特定單點上處理完畢，**但是 created_at 就不太適合作為 clustering key，因為兩個訊息很可能有同樣的建立時間。**

所以我們就做了修正，用 message_id 作為 Clustering key，我們使用 **Snowflake 來做 (不清楚的可以看一下我的註記)**。**(channel_id, message_id)** 變成了目前的結果，可以 scale 也可以支援 range scan。

> *註記：Snowflake 總共 64bit 前 41 位是時間戳，表示了自選定的時期以來的毫秒數，接下來的10位代表計算機ID，防止衝突，其餘12位代表每台機器上生成ID的序列號，所以 Snowflake 自帶時間排序的功能。*

```sql
CREATE TABLE messages (
  channel_id bigint,
  message_id bigint,
  author_id bigint,
  content text,
  PRIMARY KEY (channel_id, message_id)
) WITH CLUSTERING ORDER BY (message_id DESC);
```

# Bucket

雖然上面的設計的確解決了主要的問題，但是 Discord 在把資料搬移過去的時候，發現一個問題，那就是 partition 太大了！這會導致 Cassandra 壓縮和 GC 的壓力很大，所以還需要再做一次拆分，**作者使用一種 Bucket 的技巧，讓 10 天左右的訊息量放在一個 Bucket 當中，這樣就算 channel 無止盡的增長，還是可以把 bucket 限制在 100MB 左右。**

```python
DISCORD_EPOCH = 1420070400000
BUCKET_SIZE = 1000 * 60 * 60 * 24 * 10

def make_bucket(snowflake):
   if snowflake is None:
       timestamp = int(time.time() * 1000) - DISCORD_EPOCH
   else:
       # When a Snowflake is created it contains the number of
       # seconds since the DISCORD_EPOCH.
       timestamp = snowflake_id >> 22
   return int(timestamp / BUCKET_SIZE)
  
  
def make_buckets(start_id, end_id=None):
   return range(make_bucket(start_id), make_bucket(end_id) + 1)
```

所以最終的模型變成 **((channel_id, bucket), message_id)**

```sql
CREATE TABLE messages (
   channel_id bigint,
   bucket int,
   message_id bigint,
   author_id bigint,
   content text,
   PRIMARY KEY ((channel_id, bucket), message_id)
) WITH CLUSTERING ORDER BY (message_id DESC);
```

# Dark Launch

對於這種大公司，當然不可能一次性的把所有的東西都換掉，出了問題會很慘，所以採用 double read/write 到 MongoDB 和 Cassandra，這個過程可以順便檢查 Bug。(雖然說原文並沒有提到實作的部分，我猜應該是搭配 Messaging Queue 之類的技術)

# Trade-off

Cassandra 本身是 AP 系統，所以是採取 Eventual Consistency 的策略，更精確地說是一種 LWW (last write wins) 併發控制手法。當兩個 User 同時修改一個 Row，最後一個寫入的人會是最終的數值。

作者有提一個例子的細節，在 Cassandra 內部假如有一個 User 刪除了一則 Message，但差不多的時間又有 User 去修改，為了統一這種 edge case，就已清空該 row 作為基準。

# Performance

至於效能，一定是超級進步，所以才會有這篇部落格可以給我研究嘛~ 

- Write 可以在 sub-microsecond 完成
- Read 則是 5ms 之內完成

![](/images/discord-cassandra_0.png)

# 出大事拉

當徹底從 MongoDB 轉移到 Cassandra 的時候事情都非常的順利，**第一個麻煩是發生在六個月後，有一天 Cassandra 就掛掉了。**

我們注意到 Cassandra 有 10 sec 的 GC，完全停止了服務，像是剛剛上面說的刪除，**其實 Cassandra 會先標記成 tombstone 在讀的時候直接跳過，大約 10 天左右 (這是預設) 會 GC 掉，真的把資料刪除掉。**

結果發現罪魁禍首是 <龍族拼圖討論版> XD，由於這個討論區伺服器是公開的，我們就自己進去看看發生甚麼事情，結果發現這麼大的板，只有一則訊息(？？) 顯然是有人把東西全刪了。

但是由於剛剛講的 tombstone，**Cassandra 不會把資料真的刪除，只是先標記成刪除，所以導致讀取的時候會掃過一大堆 tombstone (就算只讀取一則訊息！)**

我們採取了以下的措施

- tombstones GC 的時間從十天縮短成兩天，還有每天晚上跑 Cassandra repairs
- 還有我們追蹤空的 bucket，如果 User 去查詢它，我們很有可能給 User 最近的訊息

# The Future

現在我們維護了 12 個 Cassandra Node，每份資料會複製三次，如果未來需要更多的 User，我們再增加 Node 就好。

## Near Term

短期基本上就是升級到 Cassandra 3，據說有對 Storage 做優化。

## Long Term

我們開始研究 ScyllaDB，用 C++ 寫成的 Cassandra，除了效能有很大的優化，還有在修復期間有比較低的 CPU 使用率，這是值得我們研究的。

# 結語

其實不用多說，光是我們能夠如此順利的處理這些數量級，就已經很有里程碑的意義了，而且我們只有 4 個backend engineers (好難想像….)

# 心得

我自己在 Database/Data-Intensive 領域研究這麼多，多多少少都會有本位主義，比如說我研究的是 Streaming Dynamic 和 Transaction，就會覺得系統一定要以這種 Feature 為核心。但是**現實世界是 Database 只是用來輔助我們想要推行產品的工具而已**。如何**快速 tune 產品，然後把 Scale 推上去是比較符合工程師的想法。**

除了技術細節，我想這篇文章給我最多的啟發應該是**對產品和基礎設施之間特性的充分理解是很重要的！**

