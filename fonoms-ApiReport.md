# fonoms 回報
* [特殊欄位說明](#特殊欄位說明)
* [fonoms 回報訊息包含哪些資料](#fonoms-回報訊息包含哪些資料)
* [訂閱回報](#訂閱回報)
* [Client 應該如何處理回報? 以利後續的刪改查](#client-應該如何處理回報-以利後續的刪改查)

----------------------------------------------------------------------------

## 特殊欄位說明
#### OrdKey = 全市場唯一編號, 由底下的欄位組成
    * Market (期權、上市、上櫃、興櫃、外期...)
    * Session (早盤、定盤、夜盤、零股...)
    * BrkNo
    * OrdNo: 此欄位可能為空白, 可能原因:
      * 新單欄位有誤
      * 新單風控失敗
      * 新單 Queuing 尚未送出: 系統設定為送單時才打印委託書號.

#### IvacKey = 投資人帳號
    * BrkNo
    * IvacNo
    * SubacNo

#### ReqUniId = 字串, fonoms 編制的 Request 唯一序號
    * 在多 fonoms 環境下, 保證同一個 ReqUniId 對應同一筆 Request.
    * 即使是外部單, 也保證 ReqUniId 唯一.
    * OrdUniId = 新單的 ReqUniId.

----------------------------------------------------------------------------

## fonoms 回報訊息包含哪些資料
底下的欄位僅為舉例說明之用, 實際線上環境, 在訂閱回報時, 會先提供「回報訊息包含哪些欄位」的資訊.

每筆回報可分為下列資料區段:
### 一、委託基本(原始)資料區
    * OrdUniId(字串):
      * fonoms 編制的唯一序號, Client 可用來當作委託唯一序號.
      * 等於 [新單要求] 的 ReqUniId, 所以新單回報不提供 ReqUniId.
    * CreateTime: 委託建立時的系統時間.
    * 基本資料:
      * IvacKey
      * Symbol
      * Side
    * 若不可改價, 則此區包含價格欄位: PriType, Pri, TIF
    * 其餘不可改的必要欄位放在此區:
      * OType: 台灣證券交易別: 資、券、現...

### 二、委託最後(現在)狀態區
    * OrderSt
    * LeavesQty, CumQty, CumAmt
    * 若允許改價, 則此區包含價格欄位: PriType, Pri, TIF; 改價成功後才會變動.
    * 交易所委託書號: OrdNo
    * 時間:
      * TrxTime: 交易所委託簿的最後異動時間: [新、刪、改] 成功時間, 不含查詢.
      * LastDealTime: 最後成交時間.

### 三、「成交 Request、下單 Request」內容區:
針對不同 [要求種類(RequestKind:新、刪、改、查、成交)] 而有不同的內容.
#### 成交回報(成交 Request)
    * 唯一成交序號: 在此委託唯一, 其他委託可能使用相同序號.
    * DealTime
    * Pri
    * Qty
    * Legs... (複式單)

#### 委託回報(下單 Request), 包含 2 區段:
##### [下單要求] 的內容:
    * RequestSeq: 此台 fonoms 自行依序編制的 [下單要求序號]
      同一筆 ReqUniId 在不同的 fonoms 裡面, RequestSeq 不會相同!
    * ReqUniId(字串): [刪改查.下單要求]唯一編號, 新單無此欄位(因為等於OrdUniId).
    * ReqCreateTime: [刪改查.下單要求]建立時的系統時間, 新單無此欄位(因為等於委託CreateTime).
    * UserId
    * FromIp
    * 新單要求: Qty(新單原始數量); 若允許改價, 則在此需包含 PriType, Pri, TIF
    * 改量要求: Qty(改後數量 or 欲刪減數量)
    * 改價要求: PriType, Pri, TIF
    * 刪單要求: -
    * 查詢要求: -
##### [下單要求] 的最後(現在)狀態: 提供 [下單要求] 執行過程(結果)的資訊.
    * StepSeq + RequestSt(Queuing, Sending, Sent, Reject, Done...)
      * 每個步驟都可能有 Queuing, Sending, Sent, Reject
      * 但只有最後一個步驟會 Done
    * UpdateTime
    * Text

----------------------------------------------------------------------------

## 訂閱回報:
### Client 必須主動訂閱:
在尚未訂閱之前, fonoms 不會主動送出回報,
所以登入後, 下單前, 必須先訂閱(回補)委託,
收到委託回補完畢訊息之後, 才開始下單.

訂閱時須提供的參數:

    回報檔Id + 起始序號 + Filter(依照權限: UserId, 可用帳號IvacList).
    * 首次連線(或需要從頭回補), 則 [回報檔Id] 填入 0.

### Server 回覆:
收到訂閱要求後, fonoms 會依序回覆底下訊息:

    * 本地的回報檔Id(HostId)
    * 回報檔格式.
    * Client要求的回報檔Id(HostId) == LocalHostId
      * 從指定序號開始回補
    * Client要求的回報檔Id(HostId) != LocalHostId
      * 從第一筆開始回補.
    * 回補完畢訊息
    * 等候新回報, 並送出.

----------------------------------------------------------------------------

## Client 應該如何處理回報? 以利後續的刪改查
* 用委託基本資料的 OrdUniId 欄位, 來識別此筆回報屬於哪筆委託書.
* 在委託書裡面保留新單的 RequestSeq: 刪改查時可能會用到.

	**`請注意`** 每次重新連線時, 如果連到與上次不同的 fonoms,
	則同一筆 OrdUniId 委託對應的 RequestSeq 不會相同!
	所以, 重新連線回補委託回報時, 一定要更新 RequestSeq.

* 刪改查時, 提供給 fonoms 尋找原委託所需的欄位:

      * 如果 OrdNo 已編碼, 則提供 OrdKey.
      * 或是使用 RequestSeq + OrdUniId
        例如: 新單正在排隊(Quueuing), 尚未打印 OrdNo, 所以無法提供 OrdKey,
              此時如果要刪單(改量、改價), 則只能用此方式找到要刪改的委託.

* 如果 [刪改查要求], 需要提供基本資料(IvacKey, Symbol, Side...),
  則委託書也要儲存這些欄位.

----------------------------------------------------------------------------

