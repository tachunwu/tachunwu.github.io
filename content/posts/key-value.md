---
title: "Key-Value Model"
date: 2022-10-31T08:25:51+08:00
draft: true
---
# Key-Value
Key-Value model 是最簡單的一種抽象化模型。

# Conscious
* 能夠區分內外(inter/outer): 提供外部 Publish，同時提供內部 Subscribe
* 當外界發生變化時(監聽外部狀態)，對內改變狀態
* 內部改變狀態時，改變外部狀態(Push-based event)或內部狀態(State Machine Operation)

# Cluster
可以 Scale N倍，每個 Node 有獨立的 addr，例如...
* sts-0
* sts-1
* sts-2
* sts-3
* sts-4
* sts-5

## Shard & Replica
* Shard-0: sts-0, sts-1
* Shard-1: sts-2, sts-3
* Shard-2: sts-4, sts-5

如果是 Cache 不用考慮 Pool 內部的 Consist 問題，key 本身會被序列化。

## Middleware
由 Instance 統一去 Database on-demand fill，需要寫監聽 key-level topic 的 handle function。

## Single-Key operation
### SET
用 Request/Reply 同步呼叫 Shard pool。
Instance 訂閱自己所屬的 shard topic  
* 先清除所有 Pool 的 Key
* Database update
### GET
用 Request/Reply 同步呼叫 Shard pool，
### DEL
用 Request/Reply 同步呼叫 Shard pool，

## (Future) Multi-Key operation (SLOG)
* Account-A 扣款
* Account-B 增款
* Order 新增

# [YCSB](https://courses.cs.duke.edu/fall13/compsci590.4/838-CloudPapers/ycsb.pdf)
<img width="1381" alt="key-value-ycsb" src="https://user-images.githubusercontent.com/95557928/198909938-e11b2297-442f-4708-ae8a-22f89116fb52.png">

## Workload A (Update heavy)
* Read: 50%
* Update: 50%
* Example: Session store recording

## Workload B
* Read: 95%
* Update: 5%
* Example: 增加 Tag 到相片，大多數是讀

## Workload C
* Read: 100%
* Example: 讀取使用者 Profile

## Workload D (Read latest)
* Read: 95%
* Insert: 5%
* Example: IG限時動態、FB Timeline


## Workload E (Short range)
* Scan: 95%
* Insert: 5%
* Example: 像是 Line/Messager 聊天。

# TPC-C
* New-Order: 新的訂單 (insert)
* Payment: 更新帳戶餘額/付款 (更新收款人和付款人的餘額)
* Delivery: 發貨(模擬批處理交易)
* Order-Status: 查詢客戶最近交易的狀態 (index on time)
* Stock-Level: 查詢倉庫庫存狀況，以便能夠及時補貨 (Scan 大範圍)



