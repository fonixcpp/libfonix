FonixCo: fonix 啟動器
========================

## 簡介
  * FonixCo 為一個 console 程式
  * 負責: 建立 fonix 執行環境、管理簡單使用者認證、管理載入模組
  * TODO:
    * Windows: install service
    * Linux: daeman
    * Log path config

## 命令列參數
  * 設定擋路徑: `-c configPath` 或 `--cfg configPath`
  * 啟動後先進入管理員模式: `--admin`
    * 使用時機: 首次啟動尚未建立使用者、忘記管理員密碼

## 預設物件
  * `/SysEnv` 系統執行環境
    * `/SysEnv/_ConfigPath` 設定擋路徑, 依照底下順序尋找:
      * 最後一個設定執行時參數: `-c configPath` 或 `--cfg configPath`
      * 尋找環境變數: `fonixcfg=configPath`
      * 以上都找不到時, 使用預設值: `fonixcfg`
    * `/SysEnv/_ExecPath` 執行時路徑
      * 程式啟動時的現在路徑
  * `/AuthMgr` 簡單認證管理
    * `/AuthMgr/RoleMgr` 使用者角色管理員
    * `/AuthMgr/PoSeedACL` 使用者政策: seed機制的存取權限
    * `/AuthMgr/UserS1` 簡單使用者管理: 使用 PBKDF2 儲存密碼
    * `/AuthMgr/Sa_SCRAM_SHA_1` 支援 SASL SCRAM-SHA-1 機制的登入服務
    * 以上認證管理的設定:
      * 主檔儲存在: `fonixcfg/SimpleAuthDB.inn`
      * 異動同步檔, 輸出: `fonixsyn/SyncAuthDB/SynOut.log`
      * 異動同步檔, 輸入: `fonixsyn/SyncAuthDB/SynIn.log`
        * 需用額外機制, 從遠端主機下載同步檔, 合併後存入此處
  * `/DllMgr` 負責載入符合 `fonix.DllMgr` 規範的載入模組
    * 主檔儲存在: `fonixcfg/DllMgr.ini`
    * 上次載入成功的備份檔(UTC time): `fonixcfg/cfgbak/DllMgr.ini.yyyymmddHHMMSS.uuuuuu`
    * 提供給 `fonix.DllMgr` 載入的模組進入點, 請參考 `fonix/seed/DllTree.hpp: struct DllEntryProc;`
    * 設定範例:
      * `編號為1000` 的載入模組 `libfonoms`, 進入點為 `AddOmsCore`, 提供給進入點的啟動參數為 `TDay,../../forms/FormAll.cfg`
      * 新設定的載入模組 **不會** 立即啟動, 會在下次啟動時自動載入
      * 如果要在重啟前載入, 則需要針對該項目執行 `apply` 指令
```console
[fonwin] /DllMgr>ss,Enabled=Y|Dll=libfonoms|Proc=AddOmsCore|Args=TDay,../../forms/FormAll.cfg /DllMgr/1000
Enabled=Y
Dll=libfonoms
Proc=AddOmsCore
Args=TDay,../../forms/FormAll.cfg
[0] Config
        [0] Id=1000
        [1] Enabled=Y
        [2] Dll=libfonoms
        [3] Proc=AddOmsCore
        [4] Args=TDay,../../forms/FormAll.cfg

[fonwin] /DllMgr>ps 1000
[0] Config
        [0] Id=1000
        [1] Enabled=Y
        [2] Dll=libfonoms
        [3] Proc=AddOmsCore
        [4] Args=TDay,../../forms/FormAll.cfg
[1] Run
        [0] Enabled=
        [1] Dll=
        [2] Proc=
        [3] Args=
        [4] Level=0
        [5] Message=

[fonwin] /DllMgr>./1000 apply
20170616065917.173535  2496[INFO ]DllMgr.Loading:|fn=libfonoms|DllEntryId=1000
20170616065917.186669  2496[ERROR]DllMgr.Loaded:|fn=V:\devel\fonoms\build\vs2015\_out64\Debug\libfonoms.DLL|proc=AddOmsCore|err=proc not found|DllEntryId=1000
Dll config applied.
```

## 基本指令
  * `exit` 結束此 user, 回到登入狀態
  * `quit` 結束 FonixCo
  * `cs NewPath` 切換 seed 路徑
    * 如果沒提供 NewPath, 則印出現在路徑
  * `ls[,OriginKey] [Path]` seed 物件列表, 例:
```console
[AdminMode] />ls
AuthMgr
DllMgr
SysEnv
keys: 3/3

[AdminMode] />ls,DllMgr
DllMgr
SysEnv
keys: 2/3
```
  * `pl [Path]` 列出物件 layout
```console
[AdminMode] />pl /AuthMgr/UserS1
[0] UserDB
        [0] UserId
        [1] RoleId
        [2] Salt
        [3] SaltedPass
        [4] Flags
        [5] AlgParam
        [6] NotBefore
        [7] NotAfter
        [8] ChgPassTime
        [9] ChgPassFrom
        [10] LastAuthTime
        [11] LastAuthFrom
        [12] LastErrTime
        [13] LastErrFrom
        [14] ErrCount

[AdminMode] />pl /DllMgr
[0] Config
        [0] Id
        [1] Enabled
        [2] Dll
        [3] Proc
        [4] Args
[1] Run
        [0] Enabled
        [1] Dll
        [2] Proc
        [3] Args
        [4] Level
        [5] Message
```
  * `ps [Path]` 印出物件內容
```console
[AdminMode] />ps /AuthMgr/UserS1/fonwin
[0] UserDB
        [0] UserId=fonwin
        [1] RoleId=admin
        [2] Salt=JbEZpd17eelHk9Bn
        [3] SaltedPass=DRiUmXvyhl42pfj9/UhPDtvFlVM=
        [4] Flags=x1
        [5] AlgParam=10000
        [6] NotBefore=
        [7] NotAfter=
        [8] ChgPassTime=
        [9] ChgPassFrom=
        [10] LastAuthTime=
        [11] LastAuthFrom=
        [12] LastErrTime=
        [13] LastErrFrom=
        [14] ErrCount=0
```
  * `ss,field=value[|field2=value2] [Path]` 設定物件內容, 如果不存在則建立
```console
[AdminMode] />cs /AuthMgr/UserS1
/AuthMgr/UserS1
[AdminMode] /AuthMgr/UserS1>ss,RoleId=trader|Flags=1 tony
RoleId=trader
Flags=1
[0] UserDB
        [0] UserId=tony
        [1] RoleId=trader
        [2] Salt=
        [3] SaltedPass=
        [4] Flags=x1
        [5] AlgParam=0
        [6] NotBefore=
        [7] NotAfter=
        [8] ChgPassTime=
        [9] ChgPassFrom=
        [10] LastAuthTime=
        [11] LastAuthFrom=
        [12] LastErrTime=
        [13] LastErrFrom=
        [14] ErrCount=0
```
  * `rs,key[^coatName] [Path]` 移除物件
```console
[AdminMode] /AuthMgr/UserS1>ls
fonwin
tony
keys: 2/2

[AdminMode] /AuthMgr/UserS1>rs,tony
fonix::seed::SeedMgr
removed_pod: tony
[AdminMode] /AuthMgr/UserS1>ls
fonwin
keys: 1/1
```
  * `[Path] cmd` 物件指令, 由各物件自行提供, 例如:
```console
[AdminMode] /AuthMgr/UserS1>./fonwin repw
The new password is: 7IH%7?R9VbmS
```
在現在路徑上執行物件指令, 要加上 `.`
```console
[fonwin] /AuthMgr/UserS1/fonwin>. repw newpw
The new password is: newpw
```
