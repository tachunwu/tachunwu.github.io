---
title: "<Life beyond Distributed Transactions: an Apostate’s Opinion> 碩論筆記"
date: 2022-12-28T19:12:03+08:00
draft: true
tags: ["CS master"]
---
# Abstract
探討 Scale Appilication 的語言學
# 假設
## Layers of the Appilication & Scale-aware
* 上層不用考慮 Scale
* 下層需要實作 Scale-aware

## Transactional Scopes
本文聚焦在討論 Transaction 在特定集合內的 Serializability，我們只要求在特定的機器內部維持隔離性，而不是整個 Data Center 強制維持一致。

# 用詞定義
## Entities
本文定義 Entity 必須擁有一個 ID 或是 Key，而且在 Entity 之內必須擁有唯一的 Serializability。

## Atmoic Transactions
Atmoic Transaction 不可以涵蓋多數的 Entity。(這邊和 Database 的定義就不太一樣了，這裡更像是應用程式的角度)

## Messages 
Messages 必須綁定 Entity ID，也就是說上層呼叫下層的時候是經由 Entity ID 作為目標。

## Activities
Entity 除了接收訊息本身還管理 State，對於他們來說，Messages 分為兩種。
* 會更改 Entity State
* 不會更改 Entity State (Idempotence)

Activities 表示 Entity 收到訊息之後內部的變化。

# Distributed Transaction
由於我們上面定義的 Transaction 只能在單一個 Entity 內發生，當橫跨多個 Entity 時，我們就需要使用 Async Messages 去拼湊出整個 Distributed Transaction。

# Remembering Messages as State
這裡給出很精確的 Activity 的定義，當我們需要一個 Entity 記得其他一起合作的 Entity 做了什麼。

# Tentative/Cancel/Confirm
作者版本的 2PC，不過概念上偏向鬆耦合。

# 個人思考
上層的商業邏輯藉由 Messaging/Stream 等基礎設施對下層進行呼叫，溝通的方式就是藉由 Subject。
* Entity 的 CRUDL 操作
* 對外部系統的混合操作

Subject 設計
* Entity Subject: <Entity>.<PK>
    * Read 要怎麼呼叫？
    * Write 要怎麼呼叫？
* Transaction Subject: Transaction.<ID>
* Distributed Cache Subject: Cache.<PK>
* Object Subject: Object.<PK>

* 當今天設計一個分離的 Full-Text Index Service，會希望 API 形式是如何？



