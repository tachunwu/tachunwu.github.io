---
title: "<Microservice Research: Dendrite by Matrix"
date: 2022-12-27T14:35:50+08:00
draft: true
---
{{<toc>}}
# 簡介

這篇文章會研究 Microservice 的 **實際案例**，我們會研究 Dendrite 這個專案，這個是一個由 Matrix 組織所發起的訊息軟體，而這個是他們的第二代軟體，主要由 Go 寫成，我認為作為研究對象十分適合。

# Services
在官方文件中提供了 Microservice 和 Monolith 的部署方式，不過我們只會著重在前者的分析。主要由以下 8 個服務所組成。
## Client API

## Sync API
這個服務主要負責 ```/sync``` 這個路徑。


## Media API
這個服務主要負責 ```/media``` 這個路徑。
### ```types```
放置 Request/MediaMetadata/Result 等資料結構

### ```storage```
* interface: 放置 MediaRepository

## Federation API

## Roomserver API

## Appservice API

## User API

## Keyserver API


