---
title: "Memo003"
date: 2022-12-13T15:21:17+08:00
draft: true
---
# 智能合約
## 背書政策（Endorsement policy）
背書政策是跟隨智能合約走的，每個智能合約有特定的背書政策。這也是hyperledger fabric與比特鏈或以太鏈很不同的地方。然而，雖然背書政策與智能合約綁定，但兩著卻又是分開的，如果今天有新的組織加入網路，可以單獨更改背書政策，而不用更新智能合約。
背書政策可以設為以下幾種形式
```
AND('ORG1.Member', 'ORG2.Member')
OR('ORG1.Member', 'ORG2.Member', 'ORG3.Member')
OUTOF(2, 'ORG1.Member', 'ORG2.Member', 'ORG3.Member')
```
且雖然背書政策是對所有組織公開，但合約本身則不是。可以確保合約內部的業務邏輯是對所有組織隱藏的。

# 基於狀態的背書（State-based endorsement）
被書政策甚至可以針對單獨物件進行設定。舉例來說，下圖的CAR1如果要更新狀態，則限定必須要有Sara的簽章。且如果CAR1被轉移給了Bob，背書政策必須要相對應的修改。
![state-based endorsement](/HyperledgerFabric/sample2.png)

