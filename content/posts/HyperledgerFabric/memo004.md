---
title: "Memo004"
date: 2022-12-13T15:55:46+08:00
draft: true
---
# 應用程式開發
應用程式主要可以完成以下任務。
* 查詢帳本
* 提交新交易
* 接收帳本通知訊息
 
## 應用程式
目前的fabric-network package可以實現很多舊版的 fabric-client 無法實現的功能，強烈建議不要使用fabric-client。
應用程式也不用去注意到智能合約的背書政策，因為SDK的discovery service會為我們處理這件事。
### 查詢帳本
因為分散式帳本的關係，應用程式可以在網路中的任何帳本上查到相對應的結果，但原則上，應用程式會優先查詢自己組織節點的帳本。
### 提交交易
![submit transaction process](/HyperledgerFabric/sample4.png)
整個交易提交流程就是產生共識的過程。也可以直接將整個流程稱呼為，共識。透過SDK可以很簡單的完成整個流程，但現在來探究一下背後的機制。
整個交易提交可以分成三個階段，**交易簽章（transaction signing）、分配（distribution）以及驗證（validation）**。
#### 交易簽章
在上圖的範例中，app1想要紀錄一筆車輛銷售的交易（id:1234）。整個交易起始於包含交易定義以及簽章的交易提案。該提案會被發送到網路中，並用於構建多方交易。
SDK可以判斷合約A須符合背書政策A，因此會將交易提案發送給組織A與組織B的節點（2a, 3a），而兩個組織都安裝了合約A，因此可以產生經過簽章的交易回覆。
雖然節點4沒有安裝相對應的合約，因此無法產生交易回覆，但他知道背書政策A，因此可以驗證交易的有效性。
在這個階段，Discovery扮演重要的角色，Discovery幫助SDK認知到合約安裝在哪些節點上，且知道背書政策A的內容。
在步驟2b，3b，SDK可以將兩個已簽章的交易回傳值打包進原本的交易提案中（帳本狀態尚未被修改）。SDK提交以及收集回傳值都是並行的，並且會自動避開未運行或錯誤的節點。
#### 分配
接著必須要將交易發送給所有的節點，而這件事是由排序節點來處理。SDK會將已經過簽章的多方交易傳送給排序節點（5a）。在交易送到排序節點後，就是共識流程開始之時，而SDK要一直等到交易寫入帳本後才會回傳給App。排序節點會將交易打包成區塊，並發送給各個節點（6），且發送是併發的。
#### 驗證
每個節點會將收到的區塊加到自己的區塊鏈上，並開始驗證區塊中交易的有效性。經過有效交易的改變的物件將會寫入狀態資料庫。在目前的範例中，交易1234必須經過兩步驟驗證，第一步需要驗證被書政策，確認交易是經過組織A與B簽章的。第二步要驗證交易的before-image與狀態資料庫的物件狀態符合，在符合的情況下，after-image的內容才會改寫狀態資料庫。
當節點完成整個上鏈的程序後，他會通知所有要求通知的應用程式。在8a中，事件通知是由peer2產生，8b是SDK收到通知。應用程式不必設定特定節點接收通知，因為SDK會預設向所有組織中的節點註冊通知。當然，通知策略也是可以客製化的。當SDK收到通知後，他會將控制權交還給應用程式，並指示交易內容是有效還是無效。
### 接收帳本通知訊息（Requesting ledger notification ）
應用程式很常有查詢帳本的需求，與查詢相反，帳本通知本身是個非同步的過程。通知本身是個很簡單的流程，應用程式只需向節點發送通知請求即可。當帳本有更動時，節點會通知註冊請求的應用程式。而通知有兩種形式，事件通知（event notification）以及區塊通知（block notification）。當應用程式收到通知時，會馬上向帳本發起查詢，確認帳本的最新狀態。

## 錢包（Wallet）與身份（Identity）
### 使用身份
身份在區塊鏈網路中扮演核心的角色，應用程式必須選擇該使用何種身份。而不止應用程式（使用者的身份），每個區塊鏈網路中的元件都有自己的獨特身份，包含節點、排序節點、CA、組織。在Hyperledger Fabric中，身份都是以X509證書以及相關連的私鑰存在。理論上，其他證書類型也是可能的，但不常見。每個參與者的 X.509 證書均由其組織的證書頒發機構（CA）頒發。
所有的元件透過網路通道結合在一起，且相關資訊被配置在一組MSP定義（MSP definitions）中。該定義描述了每個組織以及其關鍵角色。透過MSP，每個在通道中的人都可以很快地知道對方身份代表的組織。

### 使用錢包
身份訊息是被放在錢包裡的，一個使用者的錢包中可以有不同的身份訊息，分別代表他在區塊鏈網路中可以執行的權限。錢包本身的儲存可能是以下的形式。
* 本地（Local memory）：當應用程式運作在網頁上時很有用。
* 資料庫
* 硬體安全模組 （Hardware Security Module (HSM)）:當應用程式想要非常安全的儲存X509證書以及私鑰時。
錢包是透過SDK提供的gateway來與通道交互。以下是範例
```js
const userName = 'pedroId1@orgA.example.com'
const wallet = new FileSystemWallet('../identity/user/pedro/wallet')
const connectionOptions = {
 identity: userName,
 wallet: wallet,
 eventHandlerOptions: {
 commitTimeout: 100,
 strategy: EventStrategies.MSPID_SCOPE_ANYFORTX
 }
 }
await gateway.connect(connectionProfile, connectionOptions)
```
### Gateways
Gateway的用途是能夠將所有的組件合併再一起讓我們容易操作。讓我們可以專注在開發應用程式所需的三個主要功能，查找、提交以及通知。
可以將Gateway視為使用者與網路互動的連接點。我們只需要定義以下兩點。
1. connectionProfile：定義組織的連線資訊。
2. connectionOptions：定義應用程式與網路互動的設定，例如身份或錢包。
接著即可透過API`gateway.connect()`將所有的一切串接再一起。
下面是一個connectionProfile的範例：
```
name: vehicle-networks
version: 1.0.0
organizations:
 OrgA:
  mspid: OrgA.MSP
  peers:
   - peer1.orgA.example.com
  certificateAuthorities:
   - ca1.example.com
peers:
 peer1.orgA.example.com:
 url: grpc://peer1.orgA.example.com:7051
certificateAuthorities:
 ca1.example.com:
 url: http://ca1.example.com:7054
 caName: ca1.orgA.example.com
```
我們只需要定義簡單的資訊，當SDK連上區塊鏈網路時，discovery會自動拓墣（topology），將剩下的資訊補足並取代原本的connectionProfile。
### Discovery
因為Discovery，我們可以使用很簡單的connectionProfile便可以連上區塊鏈網路。Discovery本身是個利用gossip的通訊協定去搜尋其他網路元件行為的稱呼。我們只需要定義網路中的一個節點，discovery便可以為我們拿回該節點連接的所有通道相關資訊。SDK會找到其他節點並可以知道他們是否安裝著合約，預設上，SDK會假定有安裝合約的節點為背書節點。
Discovery不僅止於節點。他也可以透過定義在通道配置中的 **錨節點（anchor peer）** 找到其他的組織。
透過connectionProfile、Gateway以及Discovery，可以很簡單的實現與區塊鏈網路互動的功能。

### evaluate transaction 與 submit transaction 的不同
因為每個分散式帳本的內容理論上都會是相同的。因此SDK透過evaluateTransaction可以讓和約只在特病的節點帳本上查詢，不用將結果送給背書節點，其他節點也不會知道這次的查詢。亦即不用經過共識的流程。
而submitTransaction則會在不同節點上的合約執行。且會經過共識流程並將結果寫入帳本。
我們當然也可以透過evaluateTransaction去執行會改寫帳本的交易，但SDK並不會將這筆交易提交到區塊鏈網路中，我們只會知道這筆交易執行的結果，但不會改動帳本的值。

### 事件（Events）與通知（Notifications）
這裡要來談談帳本通知。通知是補足整個交易流程的最後一塊拼圖。與查詢或提交的流程不同，通知本身是非同步的。應用程式可以隨時註冊通知，但何時以及是否接收通知不是應用程式可以控制的。
事件是由合約所創造，包含在output當中，事件本身沒有before-image或是after-image，他僅僅傳達事實。
下面是包含事件的交易：
```
Registration updateRegistration transaction:
 identifier: 1234567890
 proposal:
  input: {CAR1, Bob, {51.06,-1.32}}
  signature: input signed by Pedro
 response:
  output:
    {CAR1.currentOwner = Sara Seller,
     CAR1.currentOwner = Bob Buyer,
    event:{"regEvent",{car: CAR1, location: 51.06,-1.32}}}
 signatures:
  output signed by Sara Seller
  output signed by Bob Buyer

```
同樣的，在合約中新增事件也是一件簡單的事。
```ts
import { Context, Contract, Info, Returns, Transaction } from 'fabriccontract-api';
import { Car } from './car';
import { Registration } from './registration';
import { Location } from './location';
@Info({title: 'RegistrationContract', description: 'The registration smart contract' })
export class RegistrationContract extends Contract {
//...
 @Transaction()
 public async updateRegistration(ctx: Context, CarId: string,
 newReg: Registration,
 coord: GpsLocation)
 : Promise<Registration> {
 const exists = await this.registrationExists(ctx, carId);
 if (!exists) {
 throw new Error(`Cannot find registration for car ${carId}.`);
 }
 const updatedReg = new Registration();
 updatedReg.value = newReg;
 const buffer = Buffer.from(JSON.stringify(updatedReg));
 await ctx.stub.putState(CarId, buffer);
 const eventPayload: Buffer = Buffer.from(
 {'CarId': CarId, 'location': coord});
 await ctx.stub.setEvent('regEvent', eventPayload);
 return updatedReg;
 }
//...
```
上面的合約在交易時會產生一個叫做RegEvent的事件，且事件是獨立於交易寫入的。此事件也會包含在回傳的簽章中。
再來看看應用程式如何接收到事件。
```js
const userName = yogendraId@orgB.example'.com;
const wallet = new FileSystemWallet('../identity/user/yogendra/
wallet');
connectionProfilePath = path.resolve(__dirname, 'registration-network.
json');
connectionProfile = JSON.parse(fs.readFileSync(connectionProfilePath,
'utf8'));
connectionOptions = {
 identity: userName,
 wallet: wallet,
 eventHandlerOptions: {
 commitTimeout: 100,
 strategy: EventStrategies.MSPID_SCOPE_ANYFORTX
 }
 };
 await gateway.connect(connectionProfile, connectionOptions);
regNetwork = await gateway.getNetwork('vehicle-registration');
contract = await network.getContract('RegistrationPackage',
'Registration');
// Listen for regEvent notifications
console.log(`Listening for registration event.`);
const listener =
 await contract.addContractListener('reg-listener',
 'regEvent',
(error: Error, event: any) => {
 if (error) {
 console.log('Error from event: ${error.toString()}');
 return;
 }
 const eventString: string = 'chaincode_id: ${event.chaincode_id},
 tx_id: ${event.tx_id}, event_name: "${event.event_name}",
 payload: ${event.payload.toString()}';
 console.log('Event caught: ${eventString}');
});
```
上面的`addContractListener()`傳入三個參數，監聽器的名字、監聽的事件以及事件處理函式。當`addContractListener()`完成後，節點就會記得我們發出的通知要求，當有事件產生時，就會透過SDK回傳兩個參數（error, event）回來。而event的結構如下：
* chaincodeId 是智能合約組合的名字。事件是包含在交易中，而交易則是透過合約生成的。 
* tx_id 是交易ID
* event_name 是產生事件的名字。
* event_payload 則是事件的內容。
當應用程式停止，監聽器會自然被移除，但事件依然會生成。兩者是解耦合的。

每個節點都可以產生事件通知，但還是可以透過設定擋設定負責產生通知的節點（event hub），他節點即使收到事件，也不會產生通知。

## 應用程式的設計
1. 應用程式同步：最簡單也最通用的機制，應用程式發出請情並等待交易成功寫道帳本後，SDK回傳並取回控制權。
2. 應用程式非同步：應用程式不會等待交易回傳，而會執行其他交易，必須確保交易彼此之間是沒有關連性的。
3. 交易非同步：應用程式可以同時處理交易以及事件監聽。（可以將兩個看做兩個分開的應用程式）
