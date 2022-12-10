---
title: "系統設計面試準備"
date: 2022-12-07T16:34:30+08:00
draft: true
---
# 前言
這個筆記是我用來紀錄 System Design 面試的過成。由於之前的公司要馬只看你有沒有基本能力、或是問很專門的領域，所以都不太擔心。

而這次參加 Cooby 的面試基本上我完全沒有先關的經驗，所以借此機會順便練一下。

*註記和揭露:我是在還沒面試前就寫這篇文章*

# 參考

主要參考 Alex Xu(CMU 畢業、任職 Twitter, Apple, Zynga, Oracle) <System Design Interview> 兩冊，他是北美軟體圈蠻有名的作者，大家可以去逛逛他的 LinkedIn，有很多漂亮的圖和知識推文。

還有我上 YT 頻道打上 System Design Interview 加減看一下訓練臨場感，我蠻推薦 Exponent 這個頻道，內容做的蠻不錯的，可以參考一下回答的方式，反正這種就是照一個套路打完，不要漏掉一些關鍵的細節，剩下就老天安排吧～

# 硬知識部分
這邊開始就直接分類我覺得重要的硬知識。

## 認識名詞

這部分就是認識名詞，然後可以解釋基本的運作原理，這個部分不可能全部都知道，比較像是經驗活。
* Load balancer
* Database replication (講用途和架構)
* Cache
* CDN
* Message queue
* DAU
* TPS/QPS

## Size & Latency
知道一些數字的大小，雖然可以查，但是還是當做反射常識會比較好～
* KB << MB << GB << TB << PB


## 回答流程
* 確認需求(多問，確保沒有亂做東西)
    * Functional
    * Non-Functional
* High-Level Design
    * Box Diagram
* Low-Level Design
    * Workflow
    * Schema
    * Infra
* Wrap
    * Error Handling
    * Scale
