---
title: "[Issue]要記得Buffer在Javascript中也是物件"
date: 2022-12-06T18:36:14+08:00
draft: true
---
## 問題描述
目前的程式碼中有一段功能，是用來接收chaincode回傳的查詢結果。
按照文件上的溝通，若查無結果會回傳一個空物件。因此當時為了用來判斷是否為空，使用下面這段程式碼。

```javascript
if (!Object.keys(queryResult).length) { return res.send({}) }
```

若是空物件，則key值陣列的長度會為零。藉此來判斷。
但今天查詢時發現這條判斷是並沒有判斷成功，長度回傳值為2。
試著將`Object.keys(queryResult)`的結果印出來，發現是`[0, 1]`。
而如果透過`Object.values(queryResult)`則會印出`[ 123, 125 ]`。

## 結果
發現實際接收到的直並非空物件，而是{}的buffer，因此透過`Object.values(queryResult)`回傳的是`{`以及`}`的ASCII編碼。