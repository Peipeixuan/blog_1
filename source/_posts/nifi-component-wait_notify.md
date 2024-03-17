---
title: Nifi component - wait & notify 介紹
date: 2023-08-15 14:36:34
tags: Nifi
---
使用 Nifi 的時候，資料流就像水龍頭一樣，只要上面的 Connector 有資料，下面的 Processor 就會開始處理，雖然 Processor 也可以設定 cron 之類的來讓他定時處理，但如果我們是希望前面的 Processor 處理完全部資料再處理下一步資料可以怎麼做呢？

## 使用 Notify & Wait

當時會用到這兩個 Processor 的情境是這樣：

我們的資料庫分成資料呈現撈取的資料庫，以及資料流處理的資料庫，分成兩個資料庫的原因是避免資料流在處理時可能會暫時讓使用者讀不到資料，所以我只要在資料流處理完之後再把兩個資料庫對調就可以了。

這時候遇到的問題就是：我要怎麼知道他已經處理完了？是否有機制可以通知我呢？

![nifi-wait-notify](img1.png)
（先偷看一下最後的成果，等下來解釋裡面的參數設定）

## 前置需求

要使用 Notify & Wait ，我們需要在要當前的 Processor Group 新增兩個 Controller Service。

- DistributedMapCacheClientService
- DistributedMapCacheServer

要注意這兩個 Controller Service 的 scope 只能是 Processor Group。

如果有其他 Processor Group 也要使用 Notify & Wait ，就要在那個 Processor Group 也建立一組 DistributedMapCacheServer & DistributedMapCacheClientService，port 也要使用不同的 port。
![DistributedMapCache](img2.png)

## Notify

Notify 顧名思義就是負責「通知」，他會把訊號丟到 Cache Service ，讓 Wait 去檢查。

參數解釋：

- Release Signal Identifier：要儲存在 Cache server 的 key 值。
- Signal Conter Name：訊號計數器的名稱。
    - 當初在看這個參數時一直搞不懂他跟 Release Signal Identifier 有什麼差別，但後來發現他並非一定要使用，如果沒改的話就是 Default ，因此我會把當理解成在同一個 Release Signal Identifier 底下再分成不同的 counter 去紀錄的話，就可以使用這個參數。
    - 如：Release Signal Identifier 設定為 flowfile 的 filename，因為整個 flow 裡面的資料 filename 都一樣，那我想要分別為 success 和 failed 做不同的 notify & wait 就可以利用 Signal Conter Name 。
- Signal Counter Delta：訊號在計數時增加的數量。而 0 代表將計數器歸零。
- Signal Buffer Count：在訊號傳遞給 Cache Service 前可以緩衝的數量。根據文件，數量越大效能越好，因為他可以把同一個 Release Signal Identifier Grouping 起來，減少跟 Cache Service 的互動。
![Nifi-notify](img3.png)

## Wait

Wait 從字面上就知道，他負責等待。等待設定的條件達成後就執行接下的 Processor。

參數解釋：

- Release Signal Identifier：和 Notify 相同的參數，也是他們之間溝通的 key 值。
- Target Signal Count：收到多少次訊號要啟動，就是門檻的概念。
- Signal Counter Name：也是和 Notify 相同的參數。
- Wait Buffer Count：根據官方文件說法是「指定緩衝以檢查其是否可以向前移動的最大傳入FlowFiles 數。 更多的緩衝區可以提供更好的性能。」概念和 Notify 的 Signal Buffer Count 類似，但它應該是指 Buffer 住的要檢查訊號數（？）。
- Releasable FlowFile Count：當達到門檻後要釋放的 flowfile 數量，如果我們只有要做一次就寫 1。
- Expiration Duration：被放到 expired 的 flowfile 的存活時間。
- Attribute Copy Mode：是否要修改 flowfile 的屬性。
- Wait Mode：設定等在 wait 的 flowfile 是否要放到 wait 的 relation 或是就讓他留在上游。
![Nifi-wait](img4.png)

---
稍微了解參數後，我們再看一次整個流程。
所以底下做的事就是：
1. 當我 ExucuteSQL 時，flowfile 會同時流進 wait 和 PutDatabaseRecord（也就是 insert 到資料庫裡）
2. insert 到資料庫成功後，成功的 flowfile 會再流進 notify，notify 就會依照設定參數「丟訊號」給 wait
3. 等到 wait 收到規定數量的訊號，他就會釋放你所設定的 flowfile 數到下一個 processor
![nifi-wait-notify](img1.png)

有幾個地方要注意：
- 以我們的案例來說，我們希望 wait 達到門檻後只要做一件事就好，因此我讓其 Releasable FlowFile Count 只有 1，一次只會是放一個 flowfile ，但上游的 flowfile 可能還有很多，因此可以在 connector 的地方加上到期時間，讓他超過一定時間後就清空消失。

![nifi-wait-notify](img5.png)

- Release Signal Identifier 設定成 `${filename}` 也就是當前 flowfile 的名字，filename 是只要從同一份檔案或同一個 query 來的資料都會相同。
    - 雖然也可以自訂固定數值，但要注意的是如果都是固定數值，下一次有新的一批資料進來時，相同訊號會一直往上加，沒辦法如實紀錄這一次執行的數量
## 總結

使用 Notify 釋放帶有特定參數的訊號， Wait 會等待收到正確數量訊號後釋放 flowfile 到 success，就可以有條件的執行接下來的步驟。以上就是簡單的 Wait 和 Notify 介紹～

## 參考
[Wait and Notify Processor Data Flow | Apache Nifi | Part 2](https://www.youtube.com/watch?v=p8iVzaIyJgo)
[Notify](https://nifi.apache.org/docs/nifi-docs/components/org.apache.nifi/nifi-standard-nar/1.12.1/org.apache.nifi.processors.standard.Notify/)
[Wait](http://nifi.incubator.apache.org/docs/nifi-docs/components/org.apache.nifi/nifi-standard-nar/1.22.0/org.apache.nifi.processors.standard.Wait/index.html)
[Notify和Wait](https://www.modb.pro/db/157355)