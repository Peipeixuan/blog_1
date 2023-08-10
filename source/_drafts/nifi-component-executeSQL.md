---
title: Nifi component - executeSQL 介紹
tags: Nifi
---

當我們開始使用 Nifi ，最常使用的莫過於從資料庫撈取資料了。

如果要使用 SQL 從關聯式資料庫取得資料，我們可以使用 ExecuteSQL 或是 ExecuteSQLRecord。

這兩個的差異在於：ExecuteSQL 取出來的資料格式都是 Avro ， ExecuteSQLRecord 則是可以依照需求改變格式。

> Avro 是一種資料格式，在 Hadoop、 Kafka、 Spark 都可以使用。他有像 JSON 那樣易讀且可以儲存各種資料格式的特性，卻是使用 binary 的方式儲存，因此他的讀取速度是更快的。

因此如果後續並沒有一定得使用 JSON 去處理資料的話，使用 ExecuteSQL 是更好的選擇。

## 參數

以 ExecuteSQL 來說，有幾個參數是最重要的。

1. Database Connection Pooling Service 
    
    這個參數要放 **Controller Service** ，才知道他要和哪個資料庫溝通。
    
2. SQL select query
    
    這裡就是放 SQL query 的語句了，他會對你的 Database 執行此 query ，並將結果放到之後的 flowfile，且忽略前一個 processor 所傳來的所有東西；
    
    不過這裡也可以是空的，那他就會期待你前一個 processor 是傳來一段 SQL，並且去執行他。 
    
3. SQL Pre-Query、SQL Post-Query
    
    這兩個就是在你的主 Query 前後執行的語句。比如說你需要對有 foreign key 的資料表做一些動作。你就會需要在前面執行 `SET FOREIGN_KEY_CHECKS = 0;` ，在之後執行`SET FOREIGN_KEY_CHECKS = 1;` ，就可以放在這裡處理。
    
4. Max Rows Per Flow File
    
    當我們把資料 SELECT 出來，他會被放到 flow filw 裡，這個參數定義的就是一個 flow file 要放多少 row 的資料。假如我的 SELECT 結果有 50 rows，這個欄位設定成 10 ，那就會獲得 5 個 flow file。那這個參數會影響到我們往下去處理的結果，假如我們想要再對這些結果做 query ，那可能會希望所有的資料可以一起處理，也有可能是我們希望可以一筆一筆做其他轉換，那可能就會設定成 1。
    

基本上以上就是我們在使用 executeSQL 最常會去調整的參數。

## 參考

[Nifi ExecuteSQL](https://nifi.apache.org/docs/nifi-docs/components/org.apache.nifi/nifi-standard-nar/1.21.0/org.apache.nifi.processors.standard.ExecuteSQL/index.html)

[Nifi ExecuteSQLRecord](https://nifi.apache.org/docs/nifi-docs/components/org.apache.nifi/nifi-standard-nar/1.21.0/org.apache.nifi.processors.standard.ExecuteSQLRecord/index.html)

[Difference between Nifi QueryDatabaseTable and QueryDatabaseTableRecord
](https://community.cloudera.com/t5/Support-Questions/Difference-between-Nifi-QueryDatabaseTable-and/td-p/370170)

[A Detailed Introduction to the Avro Data Format](https://sqream.com/blog/a-detailed-introduction-to-the-avro-data-format/)

[Why Avro for Kafka Data?
](https://www.confluent.io/blog/avro-kafka-data/)