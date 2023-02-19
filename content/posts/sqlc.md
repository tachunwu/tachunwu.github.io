---
title: "<How We Went All In on ：sqlc/pgx for Postgres + Go>"
date: 2023-02-19T16:58:55+08:00
draft: false
tags: ["Engineering"]
---
![](/images/sqlc_head.png)
*註記：作者本人有攝影的興趣，我拿他的一張相片當封面 XD*

{{<toc>}}
# 導言

這篇文章是專為 Go 開發者寫的，主要翻譯 <[How We Went All In on sqlc/pgx for Postgres + Go](https://brandur.org/sqlc)>，**這篇文章的作者本身在 Heroku 工作過也在 stripe 工作過，同時也是提出 Idempotent Key API 的人**。

現在他在 [Crunchy Data](https://www.crunchydata.com/)  開發 platform API，對於 Postgres 和 Go 都算是非常有經驗的專家，這篇文章會分享一些他對當前生態環境 Go 如何操作 Postgres 的一些看法。

# 前言

幾個月的實驗和研究後，**我們跑了一些 DB-dependent Go app，我們得到一個結論就是 [sqlc](https://github.com/kyleconroy/sqlc) 可以說是最好的拿來使用的 Postgres** (可以能其他 databases 也是)。而且在 Go code 也很容易使用，以下就讓來介紹一下我們的研究吧！

# Tour

當然我們會研究很多在 Go ecosystem 很有名的 Projects：

## database/sql

Go built-in 的 database package，我很很多人會同意我，**最好避免使用它。對 database 來說是難以預測的，而且不支援一些 Postgres-specific 的功能。**

## [lib/pq](https://github.com/lib/pq)

一個早期在 Go System 的 Postgres 專案，雖然有它的輝煌時期，**但是現在的維護量已經很少了。**

## [pgx](https://github.com/jackc/pgx)

**這個 package 寫得非常好**，對 Postgres 提供非常徹底的 full-featured, performant connections。**然而這個專案武斷的不提供 ORM 相關的功能，除了基本的 query interface 根本沒有提供什麼功能。**

就像  `database/sql` 將 database 和 stuct 集合和是非常痛苦的事情，你不只必需要手動標記 target fields 令人作嘔的字串，還需要手動 `Scan` 他們進去 struct 內部，。

### [scany](https://github.com/georgysavva/scany)

Scany 在 pgx 上面加了一些讓你使用 pgx 生活更快樂的功能，但是你還是需要羅列 field names 在 `SELECT …` 之中，**所以只算提供了半個樣板。**

## [go-pg](https://github.com/go-pg/pg)

**我個人以前有用過這個 Project，這其實是個蠻好用的 Postgres-specific ORM**。下面我們會花篇幅說明 ORMs 對 Go 的缺點，**還有 go-pg 的另一個缺點就是它時做了自己的 driver，不能相容 pgx。**

### [Bun](https://bun.uptrace.dev/guide/pg-migration.html#new-features)

go-pg 還有被收錄進去 Bun 來 maintain，重寫了 go-pg rewrite 但是去使用 non-Postgres databases，自然不在本篇的討論範圍內。

## [gorm](https://gorm.io/)

一樣和 go-pg 差不多特性，但是不只是針對 Postgres，還包還了其他的 database。你可以使用 pgx 當 driver，但是失去了很多 Postgres features。

# Research

## Queries as strings

對於原生 `database/sql` 和 `pgx` 最大的缺點就是 SQL queries 是 strings：

```go
var name string
var weight int64
err := conn.QueryRow(ctx, "SELECT name, weight FROM widgets WHERE id = $1", 42).
	Scan(&name, &weight)
if err != nil {
	...
}
fmt.Println(name, weight)
```

當然這些 queries 很簡單，**但是實際的查詢其實沒有什麼信心它們可以 work。compiler 只會看到一個 string 然後你還需要些額外得測試去驗證他們。**

更糟剛的事情是當你在寫一個大型的 Application，需要去混合一些 models 來減少 code 的重複率。你可能開始將這些 string 黏在一起，例如：

```go
err := conn.QueryRow(ctx, `SELECT ` + scanTeamFields + ` ...)
```

## ORMs

ORMs 像是 go-pg 混合了一些 type，某種程度上避免了錯誤，例如：

```go
story := new(Story)
err = db.Model(story).
    Relation("Author").
    Where("story.id = ?", story1.Id).
    Select()
if err != nil {
    panic(err)
}
```

然而沒有 generics (現在版本已經有了 XD)，Go 的 type system 能提供的也只有這樣，務實上 compiler 無法檢查到我們連接更多字串的情境。

在上面的 code，`Model()`回傳了一個 `*Query` 物件。`Relation()` 一樣也回傳 `*Query`  物件、`Where()` 也一樣。pg-go 的確有做一些優化，**但是這些錯誤只能能在 runtime 被發現。**

**ORMs 還有一個問題，與大多數習慣 SQL 得人還需要時間去適應 ORM，意味著你可能花整天的時間在看文件，只是為了翻譯 SQL ORM 之中。簡單的 queries 雖然不用花太多時間，但是試著想想看如果要處理 upsert 或是 CTE 之類的功能。**

# [sqlc](https://www.notion.so/How-We-Went-All-In-on-pgx-for-Postgres-Go-1b1b1b242db14317bf8de9bda27c0fe8)

在以上的研究後，我們發現了 sqlc。你可以簡單的寫 `*.sql` files，其中可以包涵 table 和 query，只需要加簡單的註解，例如：

```sql
CREATE TABLE authors (
  id   BIGSERIAL PRIMARY KEY,
  name text      NOT NULL,
  bio  text
);

-- name: CreateAuthor :one
INSERT INTO authors (
  name, bio
) VALUES (
  $1, $2
)
RETURNING *; 

```

之後你可以用 `sqlc generate` 自動產生 Go code，你會得到像是下面的結果：

```go
author, err = dbsqlc.New(tx).CreateAuthor(ctx, dbsqlc.CreateAuthor{
    Name: "Haruki Murakami",
    Bio:  "Author of _Killing Commendatore_. Running and jazz enthusiast.",
    ...
})

if err != nil {
    return nil, xerrors.Errorf("error creating author: %w", err)
}

fmt.Printf("Author name: %s\n", author.Name)
```

**sqlc 不算是 ORM 但它實作了一樣最有用的功能，就是 mapping query 回到 struct 而不用任何的 boilerplate**。

如果你由一個 query 包含了 `SELECT *` 或是 `RETURNING *`，**它會知道要回傳什麼東西並且綁定回去 strcut，所也的特定 table 的 queries 都有相同的 output 結構。**

**sqlc 本身用 [PGAnalyze](https://github.com/pganalyze/pg_query_go)，這和 Postgres 的 query parser 基本上是同一個**。到目前為止都沒有給我帶過什麼麻煩，而且非常複雜的查詢也能順利執行。

**這個 query parsing 還有 pre-runtime code verification，可以預先確認你的 code 是不是有 bug，如果你的 SQL 寫錯，編譯到 Go 就直接不會過。比起 SQL-in-Go-strings 實在是好太多了。如果你還是想要寫測試也是可以，不過你不需要去考慮所有詳盡的 corner case。**

## Codegen

其實我個人得哲學對 codegen 有點 ”過敏”，讓我不太願意花時間深入研究。

不過最終 sqlc 獲得了我的青睞。sqlc 就像其他 pkg 安裝一樣簡單只需要一個指令就可以了 (`go install github.com/kyleconroy/sqlc/cmd sqlc@latest`)，**而且我們的而且我們的 project 包含超過 100 個 queries，實際上編譯完成的時間不到一秒鐘！**

```bash
$ time sqlc generate

real    0.07s
user    0.08s
sys     0.01s
```

我想就算我我們把 queries 的數量增加 x100 變成 10000，我在開發生時也不會造成太大的障礙。

## pgx support appears

在這之前，**我們最大原因沒有選擇 sqlc 的原因是因為不支援 pgx**，不過最近的 PR 已經解決了這個問題，而且還提供很多的 driver 供我們使用。

**sqlc 的作者們用非常 loosely coupled 的方式去寫。像我們很多專案已經大量依賴 pgx，但是我依然能夠在一個小時內把 sqlc 的 code 遷移到 pgx 上面，這讓我們 migrate code 變得非常輕鬆！**

## Caveats and workarounds

比起傳統 ORM sqlc 還是有一些比較不方便的地方，但是有方法可以很好的解決。例如：sqlc 不能任意填寫參數，所以 multi-row insert 可能不會像你預期的一樣執行，但是你可以採取傳輸 batch 當作 array 的方式去執行它，例如：

```sql
-- Upsert many marketplaces, inserting or replacing data as necessary.
INSERT INTO marketplace (
    name,
    display_name
)
SELECT unnest(@names::text[]) AS name,
    unnest(@display_names::text[]) AS display_names
ON CONFLICT (name)
    DO UPDATE SET display_name = EXCLUDED.display_name
RETURNING *;
```

另外一個例子則是 `UPDATE`，正常來說 ORM 就是把目標欄位的的數值填上去就好 (例如：`UPDATE foo SET a = 1, b = 2, c = 3, …`)。這種方式在 sqlc 是行不通的，所有的 Queries 都必須事先結構化，所以你可以在更新時帶入 bool 的方式確認，例如：

```sql
-- Update a team.
-- name: TeamUpdate :one
UPDATE team
SET
    customer_id = CASE WHEN @customer_id_do_update::boolean
        THEN @customer_id::VARCHAR(200) ELSE customer_id END,

    has_payment_method = CASE WHEN @has_payment_method_do_update::boolean
        THEN @has_payment_method::bool ELSE has_payment_method END,

    name = CASE WHEN @name_do_update::boolean
        THEN @name::text ELSE name END
WHERE
    id = @id
RETURNING *;
```

Go 生成的 code 就會像是這樣

```sql
team, err = queries.TeamUpdate(ctx, dbsqlc.TeamUpdateParams{
    NameDoUpdate: true,
    Name:         req.Name,
})
```

# Summary and future

以上我已經介紹了大部分的 sqlc 的好處，感覺就像是用 Go 一樣我可以快速又正確的完成我的工作，不用整天和計算機本身對著幹。

不過我不會說它是整個 ecosystems 最好的解決方案，Rust’s SQL drivers 也有可能做出像魔法一樣的東西，不過至少在 Go 領域沒有疑問 sqlc 是最好的方案。

未來 Go 會引入 generics，可能會徹底改變這個專案內部建構的模式，或是啟發新的 ORMs，但至少這一兩年的經驗我們對 sqlc 非常滿意！

