libfonix 基礎建設程式庫
=======================

## 基本說明
  * libfonix 是 **風言軟體工程有限公司** 獨力開發的「C++ 跨平台基礎建設」程式庫,
  * 使用 C++11
    * VS 2015 (VC 19.00.23506)
    * g++ 4.8.4

## 目標
  * [程式即文件、一致的程式碼風格規範](Overview/CodingStyle.md)
  * 基本工具
    * 錯誤處理機制 fonix::ResultCode
    * 文數字處理工具:
      * 傳遞字串 fonix::StrArg
      * 字串解析 fonix::StrParser
      * 固定小數數字 fonix::Decimal
      * 文字轉數字 fonix::StrToNum(), fonix::StrTo()
      * 數字格式化輸出 fonix::ToStrRev(), fonix::ToStr()
      * 格式化輸出 fonix::Format
      * 簡易時間處理 fonix::TimeStamp
    * 緩衝區機制
      * 區塊記憶體管理 fonix::MemBlock
      * fonix::BufferNode, fonix::BufferList, fonix::Buffer
      * 從 **後往前** 填入資料的緩衝區: fonix::BufferListReverse
      * 從 **前往後** 填入資料的緩衝區: fonix::BufferListForward
      * 輸出文字到緩衝區(型別安全) fonix::ToBufferRev() 可支援: integer, Decimal, string, 自訂型別...
    * 簡易檔案處理 fonix::File
      * 路徑處理 fonix::FilePath
      * 檔名處理 fonix::TimedFileName 根據時間(or 序號)決定檔名
      * 根據時間(or檔案大小)自動換檔 fonix::TimedFile
    * 簡易 Log
      * 預設為 stdout 輸出
      * 可透過 fonix::InitLogWriteToFile() 設定使用 fonix::LogFile
        * 可根據時間、檔案大小自動換檔
    * 簡易 thread 工具
      * Inter thread message queue(concurrent queue): fonix::CoMQ, fonix::CoMQ_MPSC
    * 簡易 signal/slot 工具
      * fonix::Subject
      * 提供給[評測工具](https://github.com/NoAvailableAlias/signal-slot-benchmarks)用的程式: `ext/sigslot/benchmark/*`
  * 物件管理基礎建設 `fonix::seed`
    * 物件管理, 從一棵樹開始 `fonix::seed::NsTree MaTree_;`
      * 提供「高效能、方便使用、方便管理」的資料處理能力
    * 應用實例:
      * 載入模組機制 `fonix::seed::DllMgr`
  * 通訊基礎建設 `fonix::io`
    * 隱藏各種通訊的複雜性
    * 提供方便的通訊擴充能力
    * TCP、UDP... 皆可使用擴充方式提供, 處理程序不用理會實際通訊細節
    * 低延遲、高傳輸量

## Thread
  * 使用 C++11 的 `std::thread`
  * 只在仔細清楚的規劃，並輔以詳細的文件說明，之後，才將 **特定工作** 放到 **獨立Thread**
  * 在 Multi Thread 環境下, 使用 libfonix 的基本原則是: 只在 **可控制的地方** 告知物件、變數的所在
    * 盡量不用全域變數, 如果無法避免則盡可能使用 const 或 constexpr
    * Singleton 物件: 在 function 裡面使用 static std::shared_ptr<> 建立
      * C++11 有保證 function 裡面的 static object 建構是 thread-safe.
        * VS 2015 才支援(MSVC 稱它為 Magic statics): https://msdn.microsoft.com/en-us/library/hh567368.aspx
  * 在 fonix 裡面保護物件大致有底下幾種方式:
    * 使用 fonix::MustLock 鎖定後才能操作物件: 須注意要避免同時鎖定多個物件造成 deadlock
    * 使用 fonix::CoMQ_MPSC 在特定 thread 裡面操作物件
    * 先 try lock 如果成功(且Queue為空)則可立即操作物件, 否則把操作要求丟到 Queue 裡, 然後到指定的 thread 操作物件
  * AsyncOp 的 callback 得到的東西通常是暫時性的
    * 只能保證在該 callback 可以安全操作
    * 一旦結束 callback 則物件就不可繼續使用
    * 除非有額外說明, 且經過安全的處理, 例: ...
