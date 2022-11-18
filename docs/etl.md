
---

<!-- .slide: data-background="https://i.imgur.com/M2lCepI.jpg" data-background-color="#0E0047" data-background-opacity="0.5"-->

type: slide
tags: Templates, Talk, slide
slideOptions:
  transition: convex
  theme: Sky
  allottedMinutes: 30
---

<!-- .slide: data-background="https://i.imgur.com/M2lCepI.jpg" data-background-color="#0E0047" data-background-opacity="0.5"-->

<style>
.reveal section img{border:0}

pre{text-align: left !important;}
.hljs {background: #fefefe;color: #777;}
pre code .gutter.linenumber {
    color: #bfbfbf !important;
    border-right: 3px solid #6DBFFF !important;
}
.reveal pre code {
    max-height: 500px;
}
</style>

# <span style="color: #00BFFF;">实时看板方案设计</span>
<span style="color: #001FFF;">lnk 
2022-08-30
</span>

---

<!-- .slide: data-background="https://i.imgur.com/M2lCepI.jpg" data-background-color="#0E0047" data-background-opacity="0.5"-->

# 上报链路

![](/uploads/upload_e5963e083b8d5362f0333edfce964477.png)

---

<!-- .slide: data-background="https://i.imgur.com/M2lCepI.jpg" data-background-color="#0E0047" data-background-opacity="0.5"-->

## ATTA数据源

https://atta.woa.com/#/dataManage/myData/attaIdDetail?attaid=y5c0dc03514

----

<!-- .slide: data-background="https://i.imgur.com/M2lCepI.jpg" data-background-color="#0E0047" data-background-opacity="0.5"-->

### 可用字段整理1

| 字段            | 备注                                     |
| --------------- | ---------------------------------------- |
| actiontimestamp | pc 注意 app端没有这个字段 使用reporttime |
| action          |                                          |
| courseid        |                                          |
| coursepackageid |                                          |
| platform        | 1-pc 2-h5 3-安卓 4-ios                   |
| module          |                                          |

----

<!-- .slide: data-background="https://i.imgur.com/M2lCepI.jpg" data-background-color="#0E0047" data-background-opacity="0.5"-->

### 可用字段整理2

| 字段     | 备注          |
| -------- | ------------- |
| page     | search 搜索页 |
| position | 位置          |
| testid   | 使用array特性 |
| uin      |               |
| ver2     | 搜索词        |


----

<!-- .slide: data-background="https://i.imgur.com/M2lCepI.jpg" data-background-color="#0E0047" data-background-opacity="0.5"-->

### 一些问题

pc端偶尔上报失败

![](/uploads/upload_058cbe0f38b63b0ccee837d0adfadea0.png)

----

<!-- .slide: data-background="https://i.imgur.com/M2lCepI.jpg" data-background-color="#0E0047" data-background-opacity="0.5"-->

app端上报失败，重启手机后有上报

---

<!-- .slide: data-background="https://i.imgur.com/M2lCepI.jpg" data-background-color="#0E0047" data-background-opacity="0.5"-->

## 数据接出-虫洞

![](/uploads/upload_e20fcd181889affb4da26187ad4a06d7.png)

----

<!-- .slide: data-background="https://i.imgur.com/M2lCepI.jpg" data-background-color="#0E0047" data-background-opacity="0.5"-->

### topic

搜索页上报attaid: y5c0dc03514

接出虫洞：U_TOPIC_y5c0dc03514

申请消费组：cg_go_edu_etl_svr

---

<!-- .slide: data-background="https://i.imgur.com/M2lCepI.jpg" data-background-color="#0E0047" data-background-opacity="0.5"-->

## why clickhouse

ClickHouse是一个用于OLAP的列式数据库管理系统(DBMS)。

OLAP: Online Analytical Processing

----

<!-- .slide: data-background="https://i.imgur.com/M2lCepI.jpg" data-background-color="#0E0047" data-background-opacity="0.5"-->

### OLAP场景的关键特征

![](/uploads/upload_9aa29c90514452f79a2414f96489dfda.png)

----

<!-- .slide: data-background="https://i.imgur.com/M2lCepI.jpg" data-background-color="#0E0047" data-background-opacity="0.5"-->

### 列式数据库更适合OLAP场景的原因

列式数据库更适合于OLAP场景(对于大多数查询而言，处理速度至少提高了100倍)

----

<!-- .slide: data-background="https://i.imgur.com/M2lCepI.jpg" data-background-color="#0E0047" data-background-opacity="0.5"-->

#### 行式

![](https://clickhouse.com/docs/assets/images/row-oriented-d515facb5bffb48cbd09dc7d064c8816.gif)

----

<!-- .slide: data-background="https://i.imgur.com/M2lCepI.jpg" data-background-color="#0E0047" data-background-opacity="0.5"-->

#### 列式

![](https://clickhouse.com/docs/assets/images/column-oriented-b992c529fa4085b63b57452fbbeb27ba.gif)

----

<!-- .slide: data-background="https://i.imgur.com/M2lCepI.jpg" data-background-color="#0E0047" data-background-opacity="0.5"-->

### 输入/输出

1. 针对分析类查询，通常只需要读取表的一小部分列。在列式数据库中你可以只读取你需要的数据。例如，如果只需要读取100列中的5列，这将帮助你最少减少20倍的I/O消耗。

2. 由于数据总是打包成批量读取的，所以压缩是非常容易的。同时数据按列分别存储这也更容易压缩。这进一步降低了I/O的体积。

3. 由于I/O的降低，这将帮助更多的数据被系统缓存。

----

<!-- .slide: data-background="https://i.imgur.com/M2lCepI.jpg" data-background-color="#0E0047" data-background-opacity="0.5"-->

### CPU


行式存储在查询时需要处理大量的行数据
列式存储使用向量引擎，所有的操作都为向量操作，多个操作之间的不再需要频繁的调用，并且调用的成本基本可以忽略不计


----


<!-- .slide: data-background="https://i.imgur.com/M2lCepI.jpg" data-background-color="#0E0047" data-background-opacity="0.5"-->

## clickhouse数据类型

![](/uploads/upload_690ebd7f66062f1155eefb7fc7c7c7eb.png)


----

<!-- .slide: data-background="https://i.imgur.com/M2lCepI.jpg" data-background-color="#0E0047" data-background-opacity="0.5"-->

## 创建表单

```sql
CREATE TABLE ketang_content.edu_etl_for_search
               (
                   `action_at` DateTime,
                   `action` String,
                   `content_type` String,
                   `content_id` String,
                   `platform` String,
                   `page` String,
                   `module` String,
                   `position` String,
                   `uin` String,
                   `keyword` String,
                   `testid` Array(String)
               )
               ENGINE = MergeTree
               partition by toYYYYMMDD(action_at)
               ORDER BY action_at
               SETTINGS index_granularity = 8192;

-- 时间按天分区
-- 按时间排序
-- 稀疏索引默认8192
```

----

<!-- .slide: data-background="https://i.imgur.com/M2lCepI.jpg" data-background-color="#0E0047" data-background-opacity="0.5"-->

## MergeTree

Clickhouse 中最强大的表引擎当属 MergeTree （合并树）引擎及该系列（*MergeTree）中的其他引擎。

MergeTree 系列的引擎被设计用于插入极大量的数据到一张表当中。数据可以以数据片段的形式一个接着一个的快速写入，数据片段在后台按照一定的规则进行合并。相比在插入时不断修改（重写）已存储的数据，这种策略会高效很多。

----

<!-- .slide: data-background="https://i.imgur.com/M2lCepI.jpg" data-background-color="#0E0047" data-background-opacity="0.5"-->

## clickhouse到站下车

TTL

[物化试图](https://clickhouse.com/docs/zh/sql-reference/statements/create/view)

[传送门](https://clickhouse.com/docs/en/intro/)

---

<!-- .slide: data-background="https://i.imgur.com/M2lCepI.jpg" data-background-color="#0E0047" data-background-opacity="0.5"-->

## go_edu_etl_svr

https://git.woa.com/csig_edu_data_service/go_edu_etl_svr

----

<!-- .slide: data-background="https://i.imgur.com/M2lCepI.jpg" data-background-color="#0E0047" data-background-opacity="0.5"-->

## 服务架构

![](/uploads/upload_2e5664ffa7d080f85b67d9e7c43fe3e8.png)

---

<!-- .slide: data-background="https://i.imgur.com/M2lCepI.jpg" data-background-color="#0E0047" data-background-opacity="0.5"-->

看板

http://dev.grafana.edu.woa.com/d/832ld8E7z/sou-suo-shi-shi-kan-ban?orgId=1&from=now-24h&to=now