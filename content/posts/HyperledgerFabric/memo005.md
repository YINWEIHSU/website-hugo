---
title: "Memo005"
date: 2022-12-15T15:00:02+08:00
draft: true
---
# 進階應用程式開發

## 客製化SDK功能
因為關注點分離（separation of concerns）的原則，應用程式專注在業務邏輯，而與區塊鏈互動的細節都交給SDK。但這樣非常多的邏輯都照著預設值在走，有時候不是那麼適合目前的業務。現在來看看有什麼事可以調整的。
### Connection Options
我們知道Gateway是連接的入口，只要透過connectionProfile以及connectionOptions即可。
現在來看看connectionOptions的內容，下面是範例
```js
const userName = 'pedroIdentityA@orgAexample.com';
const TlsName = 'pedroIdentityC@orgAexample.com';
const wallet = new FileSystemWallet('../identity/user/pedro/wallet');
connectionProfilePath = path.resolve(__dirname, 'registration-network.json');
connectionProfile1 = JSON.parse(fs.readFileSync(connectionProfilePath,
'utf8'));
connectionOptions1 = {
  identity: userName,
  wallet: wallet,
  clientTlsIdentity: TlsName,
  eventHandlerOptions: {
    commitTimeout: 100,
    strategy: EventStrategies.MSPID_SCOPE_ANYFORTX
  }
 };
await gateway1.connect(connectionProfile1, connectionOptions1);
regNetwork = await gateway1.getNetwork('vehicle-registration');
```
identity以及wallet是兩個必要的欄位，除了他們以外，還有其他可設定的欄位。
* commitTimeout：等待每個「必要的」節點成功寫入帳本的時間，預設值為300秒，這裡設定為100秒。
* strategy設定 MSPID_SCOPE_ANYFORTX 代表任何一個組織中的節點回傳結果即完成，是很樂觀的策略。與之相反的，NETWORK_SCOPE_ALLFORTX 代表網路中的所有節點都需要回傳才會視為完成共識。
* 在一般情況下，當Gateway成功連上網路時，會使用identity去進行任何交易，但如果對等元件（例如節點）被設定為需要使用安全溝通，則可以設定 clientTlsIdentity。

### 事件處理器（Event handler function）
如果是比較大型的網路，我們或許會想要有比 MSPID_SCOPE_ANYFORTX, MSPID_SCOPE_ALLFORTX,NETYWORK_SCOPE_ANYFORTX,NETWORK_SCOPE_ALLFORTX 這四個策略更多的設置，這時候可以客製事件處理器。原則上，使用原本預設的設定即可。
```js
{
 ...
 const connectOptions: GatewayOptions = {
 transaction: {
 strategy: MyEventhandler
 }
 }
 const gateway = new Gateway();
 await gateway.connect(connectionProfile, connectOptions);
 ...
}
import { TxEventHandler } from 'fabric-network';
class MyEventHandler implements TxEventHandler {
 ...
}
```

## 暫時性資料（Transient data）
在我們呼叫智能合約的函式時，或許有些敏感數據是合約必要，但不們不想被公開在所有的區塊鏈網路上。（例如，給特定商家的折扣成數）
```js
sellTransaction = await carContract.createTransaction('sellCar');
await sellTransaction.setTransient(transientData);
await sellTransaction.submit('CAR0001', 'Bob Buyer','10000');
```
在使用setTransient時，先透過createTransaction再submit是必須的，且效率較高（與submitTransaction相比）。

## 私有數據（Private data）
看起來感覺與 Transient data 很像，但實際上 Private data 的機制完全不同。Transient data 讓交易內容再輸入時不會公開在帳本上，而 Private data 讓交易輸出不會出現在帳本上。它在私有數據集合（private data collection）中存儲，但是只有特定的組織成員（透過collection policy定義）才能訪問它。在私有數據集合中的狀態只對應於該集合，即使他與世界狀態有同樣的key值，依然是兩個不同狀態。
對於SDK來說，他依然只是呼叫交易，使用私有數據的是智能合約。但這樣要怎麼確保大家帳本紀錄的資料是相同的，且資料是在區塊上的。實際上，私有數據在區塊上是以hash值的形式存在。
下圖可以很清楚看到，當呼叫createCarOwner時，存在私有數據集合裡的狀態是car1:{REG1, VIN1, YOGENDRA}，而存在區塊裡的是0A01C1:04395C，分別對應car1以及{REG1, VIN1, YOGENDRA}的hash（SHA256）值。
![private date in blockchain](/HyperledgerFabric/sample5-1.png)
這與其他區塊鏈技術中被稱為 off-chain 的機制是很像的。

## 驗證型交易（The verify-style transaction ）
可以操作私有數據集合的合約擁有與之相關的驗證型方法。
舉例來說，下面就是一段合約的程式碼就識做了verifyCarOwner的功能。
```ts
import { Context, Contract, Info, Returns, Transaction } from 'fabriccontract-api';
import { CarOwner } from './car-owner';
const collectionName: string = 'OwnerInfo';
@Info({title: 'CarOwner', description: 'Smart Contract for Vehicle owner data' })
export class CarOwnerContract extends Contract {
public async verifyCarOwner(ctx: Context,
 carOwnerId: string,
 verifyProperties: CarOwner):
Promise<boolean> {
 const hash: Buffer = await ctx.stub.getPrivateDataHash(collectionNa
me, carOwnerId);
 if (hash.length === 0) {
 throw new Error('No private data hash with the Key: ${carOwnerId}');
 }
 const actualHash: string = pdHashBytes.toString('hex');
 // Convert user provided object into a hash
 const propertiesHash: string = crypto.createHash('sha256')
 .update(JSON.stringify(verifyProperties))
 .digest('hex');
 // Compare the hash of object provided with the hash stored on
 // public ledger
 if (propertiesHash === actualHash) {
 return true;
 } else {
 return false;
 }
}
}
```
這讓我們可以實現下面的情境。
![verify-style transaction](/HyperledgerFabric/sample5-2.png)
在上圖中，汽車註冊機構以及警察都擁有同一份智能合約。假設今天警察攔停了一輛車，雖然他沒有辦法訪問PRIVATE1的資料，但他同樣可以詢問汽車的資訊，並透過verify方法去驗證資料的正確性。

## 私有數據的共識流程（Private data consensus）
![Private data consensus](/HyperledgerFabric/sample5-3.png)
透過上圖的情境來了解私有數據的共識流程。
整個流程完成一件事，讓car1的所有權移轉給Rebecca。

第一步，registration 應用程式提交了 updateOwner 交易。輸入的值為將 car1 的所有權從 Yogendra 轉為 Rebecca。應用程式使用暫時性資料形式來確保輸入內容不會被寫在帳本上。

第二步，SDK根據背書政策選擇Peer1來為交易背書。Peer1 將運行 updateOwner 方法並代表車輛註冊組織簽署回應。

第三步，智能合約執行並訪問 PRIVATE1 集合以創建反映車主為 Rebecca 的狀態 ` car1: {REG1, VIN1, REBECCA}.` 智能合約透過 putPrivateData API 達成上述的行為。與世界狀態交易的情況一樣，此時 PRIVATE1 中的 car1 沒有發生變化。當提交整個事務時，對 PRIVATE1 的更新將在稍後的步驟 8a 和 8b 發生。由於交易涉及私有數據，因此有來自 Peer1 的額外通信。它將提議的 car1 狀態更新分發給 Peer2，Peer2 也持有 PRIVATE1 的副本。在我們的範例中，PRIVATE1 由車輛登記機構獨家持有，因此分配非常簡單。如果多個組織皆持有的情況，將分發給多個組織。

第四步，Peer2接到了 car1 的更新。但同樣的，在交易被commit之前， PRIVATE1不會有任何更新。到目前為止，車輛登記機構中的兩個 peer 都具有 car1 的新值，但尚未將它們放進他們的 PRIVATE1 副本。

第五步，updateOwner 合約方法已將包含私有數據哈希的交易返回給 SDK。 SDK 隨後創建交易 1234 並將其提交給排序服務。最重要的是要注意交易 1234 不包含 PRIVATE1 或新 car1 狀態的任何顯式或可計算記錄，除了哈希： 
*  Key 0A01C1：PRIVATE1 car1 的 SHA256 Hash
*  Value 355BFE： {REG1, VIN1，麗貝卡} 的 SHA256 Hash
而因為我們輸入的方式是使用暫時性資料，因此交易回傳並不會包含任何對世界狀態集合的更新。智能合約只更新私有數據集合。編寫生成包含公共世界狀態和私有數據更新的交易的智能合約是完全合理的，只是目前的範例不需要。

第六步，交易1234被排序。並和其他交易一起被包在區塊中，由排序節點發送給每個組織的節點（節點1,2,3,4）驗證。在第 7a、7b 步，車輛登記機構節點驗證交易1234。在一般交易，使用背書政策來驗證。對於私有數據，此策略可以存在於個別私有數據狀態、集合策略或智能合約定義中。在第 7c、7d 步，可以看到驗證由保險公司與警察驗證。

在第 8a、8b 步，PRIVATE1已經更新了 car1 的狀態，透過加上星號來代表。

## 插入型交易（The insert-style transaction）
智能合約還有一另外一種設計模式，插入型交易方法。這種交易風格與我們到目前為止看到的標准或創建風格的方法最相似，但仍然有很大的不同。
![The insert-style transaction](/HyperledgerFabric/sample5-4.png)
在上圖中，可以看到 Top bank 以及 Big Bank 使用 Interbank securities network 來記錄他們與網路中銀行的股票交易紀錄。我們先將事情簡化，只看各銀行如何使用各自的私有數據集合。該集合專門用於記錄該組織與其他銀行的所有交易（證券買賣）。而集合的內容只有自己銀行的節點有權限存取，跟我們之前提到的驗證型交易一樣，如果其他銀行可以拿到交易細節，他可以用來驗證該筆交易是否存在。
如果我們反過來看 Big Bank 的交易集合，可以看到雨 Top Bank 的類似交易紀錄。同樣的，這些資料都只有他自己的節點可以存取，但他與 Top Bank 的交易集合都有一半的 Y2 交易，因此他可以透過這筆交易驗證交易是否正確的寫入對方的集合。

上圖中最重要的一筆交易是 W2 ，因為他並不是由 Top Bank 發起的交易。這與我們過往認知的交易流程都不太一樣。這筆交易是由 Big Bank 提出想要購買股票的掛單交易，而 Top Bank想要販售股票，因此 W2 交易由 Top Bank 提出販售的掛單交易。就像實際上由 Big Bank 向 Top Bank 提出「我想請你賣股票給我，細節如下，你同意嗎？」，Top Bank 會決定是否同意這筆交易，但該提案已由 Big bank 提交，並記錄在帳本中。
同樣值得注意的是，作為同一過程的一部分，Big bank 還使用 Peer9 上的交易合約的創建方法在 BIG TRADES 中創建了 W2 購買掛單。這意味著 Top bank 現在可以使用 Peer6 上的驗證方法驗證 Big bank 是否創建了交易，並可以將此作為驗證 W2 交易的一部分。
這個範例可以知道幾個重要的點：
* 此行為是由智能合約寫定，Big Bank 無法隨意寫入任何資料進入集合中。
* Big Bank**寫入**資料進入 Top Trade 集合，而不是**讀取**。對於私有數據來說，讀取是一個比較有價值的行為，因為其需要確保隱私被保障。
這是創建交易與插入交易概念上很大的分別，創建交易需要先讀取集合並確認目標物件存在，而插入交易則單純寫入交易而沒有讀取的行為。因為插入交易是在驗證階段執行的，因此可以由非擁有該私有數據集合的組織發起，但同時符合背書政策。
如果要比喻的話，可以將私有數據集合想像為信箱，其他組織可以發信（透過putPrivateData() API）給信箱（寫入交易），但無法讀取該信箱。因此，插入型交易的生命週期是由發起掛單（Pending）交易的時候開始。也由於有背書政策，我們可以確保只要符合背書政策，不一定只能由私有數據集合的組織才能寫入資料。

接下來看一段插入型交易的智能合約：
```ts
import { Context, Contract, Info, Returns, Transaction } from 'fabriccontract-api';
import { Trade } from './trade';
@Info({title: 'Trade', description: 'Smart Contract for trades'})
export class TradeContract extends Contract {
  public async insertTrade (ctx: Context): Promise<void> {
  // Get trade details from transient data
  tradeId = transientData.get('tradeId').toString('utf8');
  buyer = transientData.get('buyerCode').toString('utf8');
  newTrade.seller = transientData.get('sellerCode').toString('utf8');
  newTrade.price = transientData.get('price').toString('utf8');
  ...
  // Check trade details
  ...
  // Insert the transaction
  const tradeBuffer = Buffer.from(newTrade);
  collectionName = newTrade.sellerCode;
  await ctx.stub.putPrivateData(collectionName, tradeId, tradeBuffer);
  }
}
```
* insertTrade()沒有包含任何輸入值，這是為了確保揭露出來的資料最小化，避免區塊鏈紀錄了任何不可洩漏的資訊。當然，我們也可以選擇包含輸入值，但要記得這些值都會被通道中的其他組織看到。
* 集合名稱是計算得出，而非寫死的。且在此方法中幾乎沒有宣告任何常量，這在真實世界的智能合約中是很典型的，因為智能合約被設計可以靈活的處理各種輸入與條件，而不是固定值。在我們的範例中，網路中的每個銀行都有一個單獨的集合，用於儲存它們與網路中其他銀行的所有交易。這意味著當一家新的銀行加入網路時，它只需要加入一個單獨的集合即可參與。只要每個集合都包含與其他銀行的所有交易，就不管是否有用於不同類型交易（如購買、銷售和待處理交易）的單獨集合都沒有關係。這種方法避免了每個對手方都有資料的情況，在這種情況下，銀行會使用不同的集合來存儲與不同對手方的交易。相反，這種方法允許每家銀行都有一個單獨的集合來存儲它們與其他銀行的所有交易，這是一種更有效且具有擴展性的模式。在網路中有幾個策略性的多邊集合是可以接受的（被多個參與者共享的集合），但是不要以系統化或有組織的方式使用這些集合。
* 智能合約中使用 putPrivateData() API 去Top Trade生成掛單交易。除非該組織有設定 memberOnlyWrite: false 不然預設是可以讓非組織成寫入。需再次重申，智能合約並沒有去讀取集合，而僅僅是寫入。

下面是應用程式呼叫，insertTrade()的範例：
```js
const tradeDetails = {
 tradeId: Buffer.from('TRADE-W2'),
 counterparty: Buffer.from('BIG'),
 units: Buffer.from('900'),
 symbol: Buffer.from('AAPL'),
 price: Buffer.from('1200')
};
insertTrade = await tradeContract.createTransaction('insertTrade');
await insertTrade.setTransient(transientData);
await insertTrade.submit();
```
在應用程式端，我們不會知道合約實際是透過插入型交易執行，應用程式單純地將交易細節放入暫時性資料中並提交至insertTrade()。這種資料隔離方式很重要，他允許應用程式以最小的負擔參與維護訊息的安全性，所有隱私問題都由智能合約來處理。

我們可以看到插入型交易的讀取與寫入是非對稱的，透過讀取獲取資料是及時的，但寫入資料會一直到交易被commit之後才完成。因此，一般來說同常會認為智能合約無法在同一個交易中讀取自己寫入的值。因為讀取是透過底層資料庫（LevelDB、CouchDB等），而寫入須等到驗證後才被寫入。但是，Fabric可以結合資料庫中的資料結合還未被驗證的寫入值去計算新的寫入值，以此方式來實現讀取自己的寫入。