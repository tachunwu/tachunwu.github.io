---
title: "<SaaS Series: Clean Architecture>"
date: 2022-12-26T11:52:39+08:00
draft: false
---
這個系列是研究 SaaS 實作的技術系列部落格，我們會從設計一路到實作走一遍。

{{<toc>}}

# Clean Architecture 
說到 SaaS 業界目前的首選絕對是 Uncle Bob 的 Clean Architecture，我主要會參考這個概念當作出發點。

簡單描述一下 Clean Architecture 有什麼樣的特性。
* Independent of Frameworks: 獨立於框架之外，你可以使用框架，但是不能夠被框架給綁住。
* Testable: 可以測試，這邊指的是能夠完全獨立測試商業邏輯，就算拔除 UI/Database...等外部元件。
* Independent of UI/Database/External agency: 簡單來說希望所有抽象的核心邏輯完全脫離外部的元件。

為了達到以上的目標，我們用下圖進行說明。
![](/images/saas-clean-architecture_1.png)

為了達到獨立性，外圍的程式碼只能依賴內圈的程式碼，當我們修改外部程式碼的時候，內部邏輯不能夠被影響。接著我們介紹一下這些同心圓代表的意義...

## Entities
最核心的業務邏輯部分，可以是 object/method/data structures/functions，這是整個系統最穩穩定的部分，我們不會希望業務以外的更動去影響到這層。

## Use Cases 
這層是介於 Entities 和外圍元件的中間層，這層會依賴 Entities，但是又不會希望更動 Database/UI 會影響到這層。

## Interface Adapters
這層是轉換層，我們希望把所有外部的資料結構全部轉換成內部 Use Cases/Entities 看得懂的資料結構。舉例來說，這裡進去你不應該看到任何 SQL 之類的語句，這些全部都會轉換成 Use Cases/Entities。

## Frameworks and Drivers
最外圍的部分，基本上我們不會自己寫什麼 code，此處多半是處理細節，避免影響到內部的部分就好。

