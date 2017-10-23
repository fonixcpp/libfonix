# fonoms 回報
* [特殊欄位說明](#特殊欄位說明)
* [fonoms 回報訊息包含哪些資料](#fonoms-回報訊息包含哪些資料)
* [訂閱回報](#訂閱回報)
* [Client 應該如何處理回報? 以利後續的刪改查](#client-應該如何處理回報-以利後續的刪改查)

----------------------------------------------------------------------------

## 目前普遍的回報方式
目前常見的回報格式是「固定式」。
  * 通常分為「委託回報、成交回報」2種訊息格式。
  * 委託回報僅使用一種格式，對於一般「新單處理」問題不大，但會有底下的問題：
    1. 欄位長度的調整，增減欄位的需求：通常需要修改程式。
    2. 遇到「刪、改、查」的回報時，會造成一些混淆：
       * 如果改價失敗，回報訊息如何告知「失敗的改價要求」的內容?
    3. 無法回報更細緻的下單處理過程：
       * 「風控處理中」，「傳送中」，「排隊中」之類的處理過程。
    4. 難以區分此次回報對應哪個要求：
       * 如果同時發出2個「減量要求」，回報時難以區分這次是哪個要求的結果。
  * 成交回報較單純：告知原委託，及此次成交的內容。
    * 較難處理的是複式單成交，不同交易所回報的方式可能不同：
      * 一次告知多腳成交的內容。
      * 每隻腳一筆回報。

所以 fonoms 要設計「新的交易回報」機制，來解決傳統回報機制所遇到的難題。

----------------------------------------------------------------------------

## 特殊欄位說明
#### `OrdKey` = 全市場唯一編號，由底下的欄位組成
  * `Market`：上市、上櫃、興櫃、期權、外期...
  * `Session`：早盤、定盤、夜盤、零股...
  * `BrkNo`：券商代號
  * `OrdNo`：此欄位可能為空白，可能原因：
    * 新單欄位有誤
    * 新單風控失敗
    * 新單 Queuing 尚未送出：系統設定為送單時才打印委託書號。

#### `IvacKey` = 投資人帳號
  * `BrkNo`：券商代號
  * `IvacNo`：投資人帳號
  * `SubacNo`：子帳號

#### `UniId` = 字串，fonoms 編制的 Request 在[同券商內,包含分公司]的唯一序號
  * 在多 fonoms 環境下，保證同一個 `UniId` 對應同一筆 Request。
  * 即使是外部單，也保證 `UniId` 唯一。

#### `RequestSeq` = fonoms 自行依序編制的 [下單要求序號]
  * 同一筆 `UniId` 在不同的 fonoms 裡面，`RequestSeq` 不保證相同。

----------------------------------------------------------------------------

## fonoms 回報訊息包含哪些資料

底下的欄位僅為舉例說明之用，實際線上環境在訂閱回報時，會先提供「回報訊息包含哪些欄位」的資訊。

每筆回報可分為下列資料區段：

### Init 委託基本(原始)資料區
  * `RequestSeq`：原始下單要求的序號。
  * `UniId`：fonoms 編制的唯一序號，Client 可用來當作委託唯一序號。
  * `Time`：委託建立時的系統時間，或收到外部回報時的系統時間。
  * 基本資料：
    * `IvacKey`
    * `Symbol`
    * `Side`
  * 若不可改價，則此區包含價格欄位：`PriType`，`Pri`，`TIF`
  * 其餘不可改的必要欄位放在此區：
    * `OType`：台灣證券交易別：資、券、現...

### OrdSt 委託最後(現在)狀態區
  * `OrderSt`
  * `LeavesQty`，`CumQty`，`CumAmt`
  * 若允許改價，則此區包含價格欄位：`PriType`，`Pri`，`TIF`; 改價成功後才會變動。
  * `OrdNo`：交易所委託書號
  * 時間：
    * `TrxTime`：交易所委託簿的最後異動時間：[新、刪、改] 成功時間，不含查詢。
    * `LastFillTime`：最後成交時間。

### 「Fill 成交」、「Request 下單」內容區：
針對不同 [要求種類(`RequestKind`：新、刪、改、查、成交)] 而有不同的內容。
#### Fill 成交回報
  * `UniSeq`：唯一成交序號：在此委託唯一，其他委託可能使用相同序號。
  * `Time`
  * `Pri`
  * `Qty`
  * Legs... (複式單)

#### Request 下單，委託回報，包含 2 區段：
##### Request [下單要求] 的內容：
  * `RequestSeq`：此筆下單要求的序號，原始下單回報無此欄位。
  * `UniId`：[刪改查.下單要求]唯一編號，原始下單回報無此欄位。
  * `Time`：[刪改查.下單要求]建立時的系統時間，原始下單回報無此欄位。
  * `UserId`
  * `FromIp`
  * 新單要求：`Qty`(新單原始數量); 若允許改價，則需包含 `PriType`，`Pri`，`TIF`
  * 改量要求：`Qty`(改後數量 or 欲刪減數量)
  * 改價要求：`PriType`，`Pri`，`TIF`
  * 刪單要求：-
  * 查詢要求：-
##### ReqSt [下單要求] 的最後(現在)狀態：
提供 [下單要求] 執行過程(結果)的資訊。
  * `StepSeq` + `RequestSt`(Queuing，Sending，Sent，Reject，Done...)
    * 每個步驟都可能有 Queuing，Sending，Sent，Reject
    * 但只有最後一個步驟會 Done
  * `Time`：此次異動的系統時間
  * `Text`：此次異動的文字說明訊息

----------------------------------------------------------------------------

## 訂閱回報：
### Client 必須主動訂閱：
在尚未訂閱之前，fonoms 不會主動送出回報，
所以登入後，下單前，必須先訂閱(回補)委託，
收到委託回補完畢訊息之後，才開始下單。

訂閱時須提供的參數：

    回報檔Id + 起始序號 + Filter(依照權限：UserId，可用帳號IvacList)。
    * 首次連線(或需要從頭回補)，則 [回報檔Id] 填入 0。

### Server 回覆：
收到訂閱要求後，fonoms 會依序回覆底下訊息：

  * 本地的回報檔Id(HostId)
  * 回報檔格式：
    * Form(表格) 數量：每個表格代表一種格式，每個回報資料區段使用一個表格。
    * 目前分成底下幾種 Form
      1. Init form：[委託基本(原始)資料](#init-委託基本原始資料區)。
      2. OrdSt form：[委託現在狀態](#ordst-委託最後現在狀態區)
      3. Fill form：[成交內容](#fill-成交回報)
      4. Request form：[下單內容](#request-下單要求-的內容)
      5. ReqSt form：[下單要求現在狀態](#reqst-下單要求-的最後現在狀態)
      6. Abandon form：此筆下單要求無效：下單要求在尚未 [建立或綁定] 委託書之前，就遭遇失敗。
    * 回報檔格式回覆範例：
      * 共 7 個 Forms，不同的 form 可能會有相同的欄位名稱，但不會造成混淆，
        因為可以輕易地從 FormKind 判斷出欄位的涵義。
      * [0] `FormKind=Init    | Fields=RequestSeq,UniId,Time,Market,Session,BrkNo,IvacNo,SubacNo,Symbol,Side,OType`
        * `RequestSeq` = 原始下單要求的本機 fonoms 序號。
        * `UniId` = 原始下單要求的券商唯一序號。
        * `Time` = Init time = fonoms 建立此下單要求的系統時間。
      * [1] `FormKind=OrdSt   | Fields=OrdNo,LeavesQty,CumQty,CumAmt,TrxTime,LastFillTime`
      * [2] `FormKind=Fill    | Fields=UniSeq,Time,Pri,Qty`
        * `Time` = 成交時間。
        * `Pri` = 成交價。
        * `Qty` = 成交量。
      * [3] `FormKind=Request | Fields=RequestKind,UserId,FromIp,PriType,Pri,Qty`
        * 沒提供 `RequestSeq`，`UniId` 則表示與 Init 相同：此 form 是 Init 的其他內容。
        * `UserId` = 建立此筆原始下單的使用者。
        * `Pri` = 原始下單價。
        * `Qty` = 原始下單量。
      * [4] `FormKind=Request | Fields=RequestSeq,UniId,RequestKind,UserId,FromIp,Time`
        * 刪單、查詢：因為沒有其他可能異動的欄位(例：價、量)
        * `UserId` = 要求此筆 [刪單 or 查詢] 的使用者。
        * `Time` = 建立此筆要求的時間。
      * [5] `FormKind=ReqSt   | Fields=StepSeq,RequestSt,Time,StatusCode,Text`
        * `Time` = 此次異動的 fonoms 系統時間，或交易所(券商主機)的最後處理時間。
      * [6] `FormKind=Abandon | Fields=RequestSeq,UniId,Market,Session,BrkNo,IvacNo,SubacNo,Symbol,Side,OType,RequestKind,RequestSt,Qty,PriType,Pri,Text`
  * Client要求的回報檔Id(HostId) == LocalHostId
    * 從指定序號開始回補
  * Client要求的回報檔Id(HostId) != LocalHostId
    * 從第一筆開始回補。
  * 回補完畢訊息
  * 等候新回報，並送出。

----------------------------------------------------------------------------

## Client 應該如何處理回報? 以利後續的刪改查
* 用委託基本資料的 `Init.UniId` 欄位，來識別此筆回報屬於哪筆委託書。
* 在委託書裡面保留 `Init.RequestSeq`：刪改查時可能會用到。

	**`請注意`** 每次重新連線時，如果連到與上次不同的 fonoms，
	則同一筆 `Init.UniId` 委託對應的 `RequestSeq` 不保證相同。
	所以，重新連線回補委託回報時，一定要更新 `RequestSeq`。

* 刪改查時，提供給 fonoms 尋找原委託所需的欄位：
  * 如果 `OrdNo` 已編碼，則提供 `OrdKey`。
  * 或是使用 `RequestSeq` + `Init.UniId`
  * 例如：新單正在排隊(Quueuing)，尚未打印 `OrdNo`，所以無法提供 `OrdKey`，
    此時如果要刪單(改量、改價)，則只能用此方式找到要刪改的委託。

* 如果 [刪改查要求]，需要提供基本資料(`IvacKey`，`Symbol`，`Side`...)，
  則委託書也要儲存這些欄位。

----------------------------------------------------------------------------

