---
title: "Memo006"
date: 2022-12-23T15:05:08+08:00
draft: true
---
# 集合與狀態的背書政策（Collection and state endorsements policy）

背書政策會在三個層級上進行。分別是合約層、集合層、狀態層。一筆交易會依序通過狀態層，集合層以及合約層之後才會被認定為有效。

![Collection and state endorsements policy](/HyperledgerFabric/sample6-1.png)

上圖範例中可以看到每個組織都有一個私有數據集合，儲存VIN等資訊。
* 當car101被生產出來時，因為集合層的背書政策優先於合約層，因此由 policy T 指示必須簽署的組織，而不是 policy I 。
* 每個組織的集合有自己的背書策略，這意味著可以在使用同一個合約的同時，但可以客製背書政策來定義必須如何簽署有效交易。

再來我們看 car102，他被交給經銷商 Tesla Dealer 1(TD1)，任何針對該車輛的變更交易都需要 TD1 的簽章。這裡我們可以得知
* 基於狀態的背書政策在當車子移轉制經銷商時被添加，該交易變更由Info合約建立並根據集合層的背書政策 T 經由 Tesla 簽署。當此交易完成後，car201將會包含狀態層的背書政策。
* 由於狀態層的優先層級高於集合層。以car201為例，政策AND(TD1)現在新增了交易需要 TD1 簽章的規定。
* 即使在同一集合，不同車子也可以有不同的背書政策。舉例來說，car201的擁有者為TD1，而car204則為HD4。

INFO的狀態是由info合約控制，但決定由誰簽署變更交易是由狀態層來控制，並不是合約層或集合層的背書政策。這樣的設計允許我們變更供應鏈中特定資產的所有權。透過適當的背書政策，我們也能實現多方組織同時擁有資產的情況。

## 集合政策（Collection Policy）
集合政策不僅僅包含用於驗證交易的背書，還包含其他一些內容。這些內容對於應用程式設計師來說，通常不是特別重要的。

例如，集合政策定義了 maxPeerCount 和 requiredPeerCount，它們控制在返回給呼叫智能合約的應用程式前，必須有多少節點收到私有狀態更新。但這是管理員配置的，並且與應用程式如何工作無關。它可能會影響性能，但這不是應用程式或智能合約開發人員應該關心的單獨問題。

## 轉移式交易（The transfer-style transaction）
我們透過「驗證式」(verify-style) 和「插入式」(insert-style) 設計模式，來設計合約處理私有數據集合。我們也可以設計基於狀態以及集合背書政策的「轉移式」交易。
![The transfer-style transaction](/HyperledgerFabric/sample6-2.png)
上圖可以看到轉移式交易中最主要的交易（簡化）。下面根據函式分別說明。
### CreateInfo
* createInfo 開啟了 info 的生命週期。負責創建VIN資訊。
* createInfo 可以在每個組織的 Info 集合確認要建立的資產是否已經存在。與插入型交易的最大差別在於，插入型交易無法讀取私有數據庫因此在創建資產前不會確認是否已存在。
* putPrivateData 將新的物件寫入帳本，但同樣的，在達成共識前，不會真的被寫進帳本。
* 由 createInfo 創建的交易需要根據集合層背書政策進行簽署。例如，一台新的 Tesla 寫進去之前，要根據政策 T 進行簽署。由於新物件是要加進集合中，因此在合約層的背書政策 I 並沒有參與其中。
在設計智能合約時，必須考慮背書政策的要求，同時仍然具有彈性，以滿足每個組織的要求。具體來說，Tesla、Honda、Peugeot 和 Ford 都有相同的智能合約，確保它們的每個「INFO」集合的行為方式相同。但是，它們每個都有不同的背書政策，以便為每個組織的交易驗證提供定制功能。
* 我們無法通過檢查 createInfo 的內容來確定對於給定的對象適用的具體背書政策。因此，必須了解智能合約及其背書政策定義的完整交易生命周期，以及相關集合及其背書政策，才能確定哪個背書政策適用。這不是理想的做法，因為如果智能合約有某種元數據，可以用於幫助解決這個問題，那就會更好。正如我們稍後將看到的，這確實會對應用程式產生影響。
 
 ### TransferInfo
 * 此交易用來轉移 carInfo 中的物件的所有權。此方法用於簽署要轉移的汽車，原使用者和新使用者。從背書的角度來看，原使用者的簽名並非必要，但是智能合約通常封裝了管理商業對象轉移的業務規則。例如，將 car201 從 Tesla 轉移到 TD1 時，將檢查該汽車當前是否由 Tesla 擁有。
 * 我們經過集合層的背書政策創造了資產，而 transferInfo 可以透過 setPrivateDataValidationParameter() API 設定物件狀態的新背書政策。在範例中，car201 會新增一個 AND(TD1) 的背書政策。
 * 儘管由 transferInfo 生成的交易將 TD1 設為擁有者，但該交易依然要滿足背書政策，經由 Tesla 的簽署過後才會滿足，這是很合理的設定。
 * 當此交易被認定為有效後，該車輛將會擁有一個狀態層的背書策略，並且在接下來的狀態更改中發揮作用。
 * 與 createInfo 不同，transferInfo 的程式碼中可以看到哪個背書政策新增至指定物件。透過 setPrivateDataValidationParameter() 可以看到狀態層的背書政策啟用，並會在接下來的交易中發生作用。

 ### updateInfo
 * updateInfo 用來修改特定汽車的詳細資訊。很間單的確認車子的存在，並且透過 get 以及 put 帳本 API 去執行。
 * 因為 transferInfo 新增了狀態層的背書政策，updateInfo 生成的交易都必須經由個別的經銷商（例如 TD1）簽署。
 * 因為擁有者並沒有改變 updateInfo 並沒有使用 setPrivateDataValidationParameters()
API 。但是，任何對該物件所做的改變都必須通過 transferInfo 時透過 setPrivateDataValidationParameters()
API 設定的背書政策。
* 與 createInfo 相同，我們無法在 updateInfo 中知道關於背書政策的設定。

## 應用程式提交轉移式交易
下面來看一段應用程式。
### createInfo
```js
const createDetails = {
 owningOrg: Buffer.from('TESLA'),
 carId: Buffer.from('Car101'),
 carVin: Buffer.from('VIN101'),
 carYear: Buffer.from('2020')
};
createInfo = await infoContract.createTransaction('createInfo');
await createInfo.setTransient(createDetails);
endorsingOrgs = ['TESLA'];
await createInfo.setEndorsingOrganizations(...endorsingOrgs);
await createInfo.submit();
```
上面的程式碼中可以看奧應用程式將細部資料放入暫時性資料中。但根據集合層的背書政策，createInfo必須由Tesla進行背書，因此透過 setEndorsingOrganizations 來標示背書組織。這個步驟是必須的，因為發現服務沒辦法幫我們完成這個步驟，要將 SDK 與 Tesla info 集合連接，必須由應用程式來實現這樣的細節。

### transferInfo
```js
const transferDetails = {
 owningOrg: Buffer.from('TESLA'),
 newOwner: Buffer.from('TD1'),
 carId: Buffer.from('Car201'),
};
transferInfo = await infoContract.createTransaction('transferInfo');
await transferInfo.setTransient(transferDetails);

endorsingOrgs = ['TESLA'];
await transferInfo.setEndorsingOrganizations(...endorsingOrgs);
await transferInfo.submit();
```
雖然 transferInfo 交易會根據 newOwner 設定新的背書政策 TD1 ，但我們依然需要 Tesla 的簽章才能成功寫進帳本，這邊一樣可以看到我們透過 setEndorsingOrganizations 來實現。

### updateInfo
```js
const updateDetails = {
 owningOrg: Buffer.from('TD1'),
 carId: Buffer.from('Car201'),
 carModification: Buffer.from('FILTER4'),
};
updateInfo = await infoContract.createTransaction('updateInfo');
await updateInfo.setTransient(updateDetails);
endorsingOrgs = ['TD1'];
await updateInfo.setEndorsingOrganizations(...endorsingOrgs);
await updateInfo.submit();
```
這裡最主要的差別是被書組織改為設定 TD1。

我們並沒有花太多時間專注在智能合約上的商業邏輯。先單純看上面三個交易。
* createInfo 可以確保 VIN 符合格式且之前尚未發布過。
* transferInfo 可以確保交易是由車子的擁有者發起，且提交者擁有該組織賦予的足夠權限。
* updateInfo 可以確保交易是由車輛的擁有者提交並且更改是有效的。


