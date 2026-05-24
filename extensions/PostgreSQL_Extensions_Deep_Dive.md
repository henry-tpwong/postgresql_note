# PostgreSQL Extensions 深入探討

> **閱讀順序：由淺入深，逐步構建分散式數據庫視野**

本書按 PostgreSQL 擴展能力的三個層次編排：

- **第一章（FDW — IMPORT FOREIGN SCHEMA）**：跨庫查詢基礎。從單一 remote server 的 foreign table 批量建立開始，掌握 PostgreSQL FDW 架構、schema 導入、type mapping 等核心概念。這是所有跨節點數據操作的地基——無論是分區路由還是 shard 路由，底層都建立在 FDW 機制之上。
- **第二章（pg_pathman — 單節點高效分區）**：單節點內的分區管理進階。深入 Custom Scan API 與 RuntimeAppend 的 runtime partition pruning 機制，了解如何在不改動 catalog 的前提下，以 O(1) 哈希查找或 O(log N) 二分查找取代傳統的 O(N) constraint exclusion。FDW 分區支援為第三章的多節點 sharding 鋪路。
- **第三章（pg_shard — 多節點分片）**：多節點分片與高可用。在掌握跨庫查詢（第一章）與分區路由（第二章）後，探索 hash sharding、replica placement、shard repair 等分散式數據庫核心主題。pg_shard 是 Citus 的技術前身，其 metadata 設計與修復機制仍深刻影響現代 Citus 架構。

---

# 一、PostgreSQL IMPORT FOREIGN SCHEMA — 一鍵創建 Foreign Table

> 來源：[digoal - PostgreSQL 9.5 使用 import foreign schema 语法一键创建外部表 (2015-04-09)](https://github.com/digoal/blog/blob/master/201504/20150409_02.md)

---

## 1. 核心語法與背景

> **初學者導讀**：FDW（Foreign Data Wrapper，外部資料包裝器）是 PG 內建的跨資料庫查詢機制。你可以把它想像成在本地資料庫中建立一個「窗口」——透過這個窗口，你能直接查詢另一台 PG 伺服器上的表格，就像它們存在本地一樣。`IMPORT FOREIGN SCHEMA` 是這個窗口的「快速建立工具」：你只需要一行指令，它就會自動掃描遠端 schema 中有哪些表格、每個表格有哪些欄位，然後一口氣在本地建立對應的「代理表格」（foreign table）。這省去了手寫數十甚至數百行 `CREATE FOREIGN TABLE` 的麻煩。

> 生活比喻：你在台北辦公室（本地 PG）需要頻繁查詢上海機房（遠端 PG）的 20 張業務表。沒有 `IMPORT FOREIGN SCHEMA` 之前，你得手動為這 20 張表逐一寫建表語句，還要確保每個欄位的型態對應正確。有了它，你只要告訴 PG：「去上海把整個 schema 的表結構都搬過來」，一鍵搞定。

```mermaid
flowchart LR
    subgraph L["本地 PG（9.5+）"]
        FT["foreign table\n（代理表格）"]
        LS["local_schema"]
    end
    subgraph R["遠端 PG 伺服器"]
        RT["實際表格\n（存有真實資料）"]
        IS["information_schema.\ncolumns"]
    end
    U["IMPORT FOREIGN\nSCHEMA 指令"] -->|"步驟1: 查詢遠端\ninformation_schema"| IS
    IS -->|"步驟2: 取得欄位定義\n(table name, column, type)"| U
    U -->|"步驟3: 在本地逐一執行\nCREATE FOREIGN TABLE"| FT
    FT -->|"使用者查詢時\n自動轉發到遠端"| RT
```

```mermaid
flowchart TD
    A["IMPORT FOREIGN SCHEMA rmt\nFROM SERVER rmt\nINTO local_schema"] 
    A --> B["連線到遠端伺服器\n（透過事先定義的 FOREIGN SERVER）"]
    B --> C["查詢遠端的 information_schema.columns\n取得該 schema 內所有表格的欄位定義"]
    C --> D["遍歷每個表格：\n讀取 table name、column name、data type"]
    D --> E["在本地逐表執行\nCREATE FOREIGN TABLE ...\n設定欄位對應（column_name mapping）"]
    E --> F["完成！本地 schema 內出現\n所有對應的 foreign tables\n可直接 SELECT 查詢"]
```

PostgreSQL 9.5 引入 `IMPORT FOREIGN SCHEMA`，可以將 remote server 上整個 schema 的所有 table（或指定子集）一次性批量創建為 local 的 foreign table。在此之前，需要手寫 DDL 或用自訂 function 從 metadata 中抽取 table name、column name、type 來生成建表語句。

```sql
IMPORT FOREIGN SCHEMA remote_schema
    [ { LIMIT TO | EXCEPT } ( table_name [, ...] ) ]
    FROM SERVER server_name
    INTO local_schema
    [ OPTIONS ( option 'value' [, ... ] ) ]
```

> 補充（Senior Dev）：這個命令本質是 DDL 操作的批量捷徑，不是持續同步機制。執行後 remote schema 的結構變更（新增 column、改 type、加 table）不會自動傳播到 local，需要重新 `IMPORT FOREIGN SCHEMA` 或手動調整。它的底層實現是查詢 remote 的 `information_schema.columns` 獲取欄位定義後逐表生成 `CREATE FOREIGN TABLE`，所以在 PG 跨版本場景（如 9.4→9.5）也能正常映射 type，前提是兩端的 type name 一致。自訂 type、enum、array of composite 等非標準 type 可能無法自動映射，需事後手動修正。

---

## 2. 完整操作示範

> **初學者導讀**：在動手之前，先建立整體流程的心理地圖。整個操作分為三個階段：
> 1. **遠端準備**：在遠端 PostgreSQL 建立 schema、表格和測試資料。
> 2. **本地配置**：在本地 PG 安裝 postgres_fdw extension、建立連線到遠端的「foreign server」、設定認證（user mapping）。
> 3. **一鍵導入**：執行 `IMPORT FOREIGN SCHEMA`，批量建立 foreign tables，然後驗證查詢。

> 關鍵點理解：`CREATE SERVER` 不是真的建立一個伺服器，而是定義一個「連線目標」的設定（主機位址、port、資料庫名稱）。`CREATE USER MAPPING` 則是告訴 PG 用什麼帳號密碼去連那個遠端。完成這兩步後，`IMPORT FOREIGN SCHEMA` 才知道要去哪裡、用誰的身份讀取表格結構。

```mermaid
flowchart TD
    subgraph R["遠端 PG 9.4.1（Remote）"]
        RS["Schema: rmt"]
        T1["Table: rmt\n(id, info, crt_time)"]
        T2["Table: rmt1\n(id, info, crt_time)"]
        T3["Table: rmt2\n(id, info, crt_time)"]
        RS --> T1
        RS --> T2
        RS --> T3
    end

    subgraph L["本地 PG 9.5（Local）"]
        EXT["CREATE EXTENSION postgres_fdw"]
        SRV["CREATE SERVER rmt\n→ host: 127.0.0.1:1921"]
        UM["CREATE USER MAPPING\n→ user: postgres"]
        IMP["IMPORT FOREIGN SCHEMA rmt\nFROM SERVER rmt INTO r1"]
        FT1["r1.rmt  (foreign table)"]
        FT2["r1.rmt1 (foreign table)"]
        FT3["r1.rmt2 (foreign table)"]
        EXT --> SRV --> UM --> IMP
        IMP --> FT1
        IMP --> FT2
        IMP --> FT3
    end

    FT1 -.->|"查詢時自動轉發"| T1
    FT2 -.->|"查詢時自動轉發"| T2
    FT3 -.->|"查詢時自動轉發"| T3
```

環境：remote = PostgreSQL 9.4.1，local = PostgreSQL 9.5。

**Remote 端準備（9.4.1）：**

```sql
-- 建立 schema 與 table
CREATE SCHEMA rmt;
CREATE TABLE rmt (id int, info text, crt_time timestamp);
CREATE TABLE rmt1 (id int, info text, crt_time timestamp);
CREATE TABLE rmt2 (id int, info text, crt_time timestamp);

-- 灌入測試資料
INSERT INTO rmt SELECT generate_series(1,100000), md5(random()::text), clock_timestamp();
INSERT INTO rmt1 SELECT generate_series(1,100000), md5(random()::text), clock_timestamp();
INSERT INTO rmt2 SELECT generate_series(1,100000), md5(random()::text), clock_timestamp();

-- 設定 constraints（這些不會被 import，詳見 3. Constraints 說明）
ALTER TABLE rmt ADD CONSTRAINT pk PRIMARY KEY (id);
ALTER TABLE rmt ADD CONSTRAINT ck CHECK (length(info) > 1);

-- 移到目標 schema
ALTER TABLE rmt SET SCHEMA rmt;
ALTER TABLE rmt1 SET SCHEMA rmt;
ALTER TABLE rmt2 SET SCHEMA rmt;
```

**Local 端操作（9.5）：**

```sql
-- 啟用 postgres_fdw extension
CREATE EXTENSION postgres_fdw;

-- 建立 foreign server（指向 remote）
CREATE SERVER rmt FOREIGN DATA WRAPPER postgres_fdw
    OPTIONS (hostaddr '127.0.0.1', port '1921', dbname 'postgres');

-- 建立 user mapping（remote 端的認證）
CREATE USER MAPPING FOR postgres SERVER rmt
    OPTIONS (user 'postgres', password 'postgres');

-- 建立 local schema 接收 foreign table
CREATE SCHEMA r1;

-- 一鍵導入整個 remote schema
IMPORT FOREIGN SCHEMA rmt FROM SERVER rmt INTO r1;
```

驗證結果：

```
           List of foreign tables
 Schema | Table | Server |              FDW Options
--------+-------+--------+----------------------------------------
 r1     | rmt   | rmt    | (schema_name 'rmt', table_name 'rmt')
 r1     | rmt1  | rmt    | (schema_name 'rmt', table_name 'rmt1')
 r1     | rmt2  | rmt    | (schema_name 'rmt', table_name 'rmt2')
```

Foreign table 結構自動對應 remote column：

```
                         Foreign table "r1.rmt"
  Column  |            Type             | Modifiers |       FDW Options
----------+-----------------------------+-----------+--------------------------
 id       | integer                     | not null  | (column_name 'id')
 info     | text                        |           | (column_name 'info')
 crt_time | timestamp without time zone |           | (column_name 'crt_time')
Server: rmt
FDW Options: (schema_name 'rmt', table_name 'rmt')
```

查詢正常：

```sql
SELECT count(*) FROM r1.rmt1;   -- 100000
SELECT count(*) FROM r1.rmt;    -- 100000
SELECT count(*) FROM r1.rmt2;   -- 100000
```

> 補充（Senior Dev）：`IMPORT FOREIGN SCHEMA` 會沿用 remote table 的 NOT NULL constraints（見上例 `id` 的 `not null`），但 PRIMARY KEY、UNIQUE、CHECK、FOREIGN KEY 等 constraints 不會被導入。這是 FDW 架構的限制：local 端無法驗證 remote 端的約束完整性。如果需要 local 查詢優化（如 join pushdown、parameterized path），可用 `OPTIONS (use_remote_estimate 'true')` 讓 planner 查詢 remote 的 `pg_stats` 估算成本，而非依賴 local 的 default estimate。

---

## 3. LIMIT TO 與 EXCEPT 過濾

> **初學者導讀**：在真實場景中，遠端 schema 可能有 100 張表，但你只需要其中的 3 張。`LIMIT TO` 和 `EXCEPT` 就是為此設計的兩個過濾器——一個是「白名單」（只導入這些），一個是「黑名單」（不要導入這些）。兩者不能同時使用，每次只能選一種策略。

> **何時用哪個？** 如果你只關心少數特定表 → 用 `LIMIT TO`（更安全，不會意外導入不相干的表）。如果你擁有整個 schema 的完整控制權，只是要排除一兩張不需要的表 → 用 `EXCEPT`（更省事）。

```mermaid
flowchart LR
    subgraph R["遠端 Schema: rmt"]
        A["rmt"]
        B["rmt1"]
        C["rmt2"]
        D["其他表..."]
    end
    LIMIT["LIMIT TO (rmt)"] -->|"只導入"| A
    EXCEPT["EXCEPT (rmt)"] -->|"排除 rmt\n導入其餘全部"| B & C
    style A fill:#4a9,stroke:#333
    style LIMIT fill:#eef,stroke:#333
    style EXCEPT fill:#fee,stroke:#333
```

可以只導入指定 table，或排除特定 table：

```sql
-- 只導入 rmt 這一張 table
DROP FOREIGN TABLE r1.rmt;
DROP FOREIGN TABLE r1.rmt1;
DROP FOREIGN TABLE r1.rmt2;
IMPORT FOREIGN SCHEMA rmt LIMIT TO (rmt) FROM SERVER rmt INTO r1;
-- 結果：只有 r1.rmt 一張 foreign table

-- 排除 rmt 後導入其餘
DROP FOREIGN TABLE r1.rmt;
IMPORT FOREIGN SCHEMA rmt EXCEPT (rmt) FROM SERVER rmt INTO r1;
-- 結果：r1.rmt1, r1.rmt2 兩張 foreign table
```

> 補充（Senior Dev）：`LIMIT TO` 與 `EXCEPT` 互斥，不能同時使用。在實際應用中，若 remote 是共享數據庫包含大量不相干的 table，`LIMIT TO` 是更安全的做法（只導入你確定的 table），`EXCEPT` 適合 remote schema 由你完全控制的場景。

---

## 4. 注意事項：View、Materialized View、Foreign Table 也會被導入

> **初學者導讀**：`IMPORT FOREIGN SCHEMA` 不只導入普通表格，它會把遠端 schema 中「所有看起來像表的東西」都導入——包括 view（檢視表）、materialized view（實體化檢視表）、以及遠端自己已有的 foreign table。這對初學者來說可能是驚喜也可能是陷阱：
> - **驚喜**：你不需要額外操作就能查詢遠端的 view。
> - **陷阱**：導入的 materialized view 只是一個指向快照資料的 foreign table，查詢時拿到的不是「即時重算」的結果，而是 materialized view 最後一次 refresh 的快照。如果遠端有串聯的 foreign table（遠端 A 的 foreign table 指向更遠的 B），本地導入後也會形成「串聯查詢」，效能會顯著下降。

```mermaid
flowchart TD
    subgraph R["遠端 PG"]
        VT["普通 table"]
        VW["view\n（虛擬表，查詢時即時計算）"]
        MV["materialized view\n（儲存快照結果）"]
        FT_R["foreign table\n（指向另一個遠端）"]
    end

    IMP["IMPORT FOREIGN SCHEMA"] -->|"全部導入為 foreign table"| L_FT["本地 foreign table"]

    L_FT -->|"查詢 view 時\n遠端即時返回計算結果"| VW
    L_FT -->|"查詢 materialized view 時\n返回最後一次 refresh 的快照"| MV
    L_FT -->|"查詢 foreign table 時\n形成串聯查詢（耗時倍增）"| FT_R
    L_FT -->|"查詢普通表"| VT

    style MV fill:#f9a,stroke:#333
    style FT_R fill:#f96,stroke:#333
```

`IMPORT FOREIGN SCHEMA` 不只導入普通 table，**連 remote 端的 view、materialized view、foreign table 都會一併導入**（除非用 `EXCEPT` 排除）。

**Remote 端準備：**

```sql
-- 建立 view
CREATE VIEW rmt.v1 AS SELECT * FROM test;

-- 建立 foreign table（指向同一 remote 的其他 schema）
CREATE SERVER rmt FOREIGN DATA WRAPPER postgres_fdw
    OPTIONS (hostaddr '127.0.0.1', port '1921', dbname 'postgres');
CREATE USER MAPPING FOR postgres SERVER rmt
    OPTIONS (user 'postgres', password 'postgres');
CREATE FOREIGN TABLE rmt.ft1 (id int, info text)
    SERVER rmt OPTIONS (schema_name 'public', table_name 'test');
```

**Local 端導入結果：**

```
           List of foreign tables
 Schema | Table | Server |              FDW Options
--------+-------+--------+----------------------------------------
 r1     | ft1   | rmt    | (schema_name 'rmt', table_name 'ft1')
 r1     | rmt1  | rmt    | (schema_name 'rmt', table_name 'rmt1')
 r1     | rmt2  | rmt    | (schema_name 'rmt', table_name 'rmt2')
 r1     | v1    | rmt    | (schema_name 'rmt', table_name 'v1')
```

remote 端的 view `v1` 和 foreign table `ft1` 都被導入為 local 的 foreign table，且查詢正常：

```sql
SELECT * FROM r1.v1;   -- 返回 remote view 的查詢結果
SELECT * FROM r1.ft1;  -- 返回 remote foreign table 的查詢結果
```

> 補充（Senior Dev）：這意味著你可以建立「串聯 FDW」——local foreign table → remote foreign table → remote 的另一個 server。每層跳轉都會增加 query latency，且 query planning 複雜度疊加。在 production 中建議避免超過一層 FDW 串聯；如果需要跨多層數據源，考慮在 remote 端用 materialized view 扁平化後再導入。
>
> Materialized view 被導入為 foreign table 後，local 端查詢時 remote 端返回的是 materialized view 的快照數據（不是實時重算結果），這與直接導入普通 view 的行為有本質差異，需注意數據新鮮度。

---

## 5. 限制與適用範圍

> **初學者導讀**：本章的核心知識總結如下：

```mermaid
flowchart TD
    subgraph SUPPORT["支援情況"]
        PGFDW["postgres_fdw ✅\n完整支援 IMPORT FOREIGN SCHEMA"]
        OTHER["其他 FDW ❌\n(mysql_fdw, mongo_fdw, file_fdw)\n需各自實作此介面"]
    end
    subgraph IMPORTS["導入內容"]
        TAB["普通 table ✅"]
        COL["欄位定義 + NOT NULL ✅"]
        VW["view / materialized view ✅"]
        FT["foreign table ✅"]
    end
    subgraph SKIP["不導入的內容"]
        PK["PRIMARY KEY ❌"]
        UN["UNIQUE ❌"]
        CK["CHECK ❌"]
        FK["FOREIGN KEY ❌"]
    end
    subgraph RISKS["潛在風險"]
        PWD["user mapping 密碼明文存在\npg_user_mapping 目錄中"]
        VER["跨 PG major version 時\ntype name 可能不相容"]
        CHAIN["多層 FDW 串聯\n效能嚴重下降"]
    end
```

- 目前只有 `postgres_fdw` 支援 `IMPORT FOREIGN SCHEMA` 語法。其他 FDW（如 `mysql_fdw`、`mongo_fdw`、`file_fdw`）需要各自 implement 此接口，否則會報 `feature not supported`。
- 官方 commit（`59efda3e`）由 Ronan Dunklau 和 Michael Paquier 提交，核心變更在於提供 server-side infrastructure，讓各 FDW 可以選擇性 implement。

> 補充（Senior Dev）：`IMPORT FOREIGN SCHEMA` 的 OPTIONS clause 可以傳遞 FDW-specific 參數，例如 `postgres_fdw` 支援 `import_default`（是否導入 default value）、`import_not_null`（是否導入 NOT NULL）。從 PG 11 開始還可以 `IMPORT FOREIGN SCHEMA ... LIMIT TO (table_name) OPTIONS (import_default 'false')` 來控制行為。
>
> User mapping 中的 password 以明文存儲在 `pg_user_mapping` catalog 中（superuser 可讀），在生產環境建議改為 `pg_hba.conf` 的 trust / cert / scram-sha-256 認證，或使用 `pg_service.conf` 配合 `password_required=false` 選項搭配 peer 認證。
>
> 如果是跨 PG major version 的 FDW（如 local PG 16 → remote PG 9.4），需要注意：
> 1. remote 的 `information_schema` 返回的 type name 必須能在 local 端成功解析。
> 2. 若 remote 有自訂 type 或在 PG 版本間 type name 有變更（如 `xid` → `xid8`），import 會報錯。
> 3. 遇到此類問題，可以先用 `LIMIT TO` 單表導入調試，成功後再批量，可避免大量報錯難以排查。

---

# 二、PostgreSQL pg_pathman — 高效分區表 Plugin（Custom Scan API）

> 來源：[digoal - PostgreSQL 9.5+ 高效分区表实现 - pg_pathman (2016-10-24)](https://github.com/digoal/blog/blob/master/201610/20161024_01.md)

---

## 1. 背景與架構

> **初學者導讀**：在深入 pg_pathman 之前，先建立一個關鍵理解：PostgreSQL 的 partition（分區）功能經歷了三代演進：
> 1. **第一代（PG 9.x 及之前）**：使用 table inheritance（繼承）+ CHECK constraint + trigger，把一張大表「手動拆成」多張小表。查詢時靠 PG 的 constraint exclusion 機制決定掃哪些子表，但這個決策是逐表檢查的（O(N)），分區越多越慢。
> 2. **第二代（PG 10-11）**：引入 declarative partitioning（宣告式分區），不再需要手寫 trigger，但底層的 partition pruning（分區過濾）仍然有限。
> 3. **pg_pathman（PG 9.5+ 的第三方方案）**：不走 PG 內建的繼承機制，而是利用 Custom Scan API 直接接管查詢計畫的生成。它用 binary search 或 hash lookup 定位分區，速度從 O(N) 降到 O(log N) 或 O(1)。

```mermaid
flowchart LR
    subgraph TRAD["傳統方案（PG 內建繼承）"]
        direction TB
        TQ["查詢: WHERE id = 500"]
        TAPP["Append node\n逐表掃描"]
        T1["分區1\nCHECK id 1-100 ❌"]
        T2["分區2\nCHECK id 101-200 ❌"]
        T3["分區3\nCHECK id 201-300 ❌"]
        T4["分區4\nCHECK id 301-400 ❌"]
        T5["分區5 ✅\n找到目標"]
        TQ --> TAPP
        TAPP --> T1 & T2 & T3 & T4 & T5
        style T1 fill:#faa,stroke:#333
        style T2 fill:#faa,stroke:#333
        style T3 fill:#faa,stroke:#333
        style T4 fill:#faa,stroke:#333
        style T5 fill:#4a9,stroke:#333
    end

    subgraph PATH["pg_pathman 方案"]
        direction TB
        PQ["查詢: WHERE id = 500"]
        PRT["RuntimeAppend\nBinary Search O(log N)"]
        P1["分區5 ✅\n直接命中目標"]
        PQ --> PRT --> P1
        style P1 fill:#4a9,stroke:#333
    end
```

```mermaid
flowchart TD
    subgraph ARCH["pg_pathman 架構"]
        HOOK["ProcessUtility_hook\n（攔截 COPY 指令）"]
        PLANNER["planner_hook\n（攔截查詢計畫生成）"]
        EXEC["ExecutorStart_hook\n（執行期動態選分區）"]
        FILTER["PartitionFilter HOOK\n（攔截 INSERT 指令）"]
        BG["Background Worker\n（非阻塞資料遷移）"]
        CACHE["Memory Cache\n從 pathman_config 載入分區定義"]
    end

    APP["使用者 SQL 查詢"] --> PLANNER
    PLANNER -->|"替換 Append → RuntimeAppend"| EXEC
    FILTER -->|"跳過 trigger\n直接路由到目標分區"| TARGET["目標分區表"]
    HOOK -->|"COPY 直接寫入\n目標分區"| TARGET
    BG -->|"批次遷移資料\n不鎖定主表"| TARGET
```

### I. 傳統分區表的問題

PG 社區版的分區功能長期依賴 **inheritance + CHECK constraint + trigger/rule**：

- **Insert**：trigger 或 rule 重寫 → 逐行判斷分區 → 效能差
- **Select/Update/Delete**：依賴 `constraint_exclusion = partition`，走 constraint check 逐區過濾（非 binary search）→ 分區越多越慢
- **Subquery 過濾分區不受支援**

商業版 EDB 和 Greenplum 有較好的分區支援。2015 年 GP 開源後，阿里雲 RDS PG 曾將 GP 的分區功能 port 到 PG 9.4（需改動 catalog，效能提升近百倍）。但 pg_pathman 走的是另一條路——**不改 catalog，用 Custom Scan API + HOOK 實作**。

### II. pg_pathman 設計原理

pg_pathman 由 PostgreSQL 核心貢獻者 Oleg Bartunov 所在的 **postgrespro** 公司開發。

| 技術 | 實作方式 |
|------|---------|
| 分區定義存儲 | `pathman_config` table + memory cache（不走 catalog 改動） |
| Range 分區定位 | **Binary search**（O(log N)） |
| Hash 分區定位 | **Hash search**（O(1)） |
| Plan node 替換 | `RuntimeAppend` → 取代 `Append`，runtime 才決定掃哪些分區 |
| Merge plan | `RuntimeMergeAppend` → 取代 `MergeAppend` |
| Insert 優化 | `PartitionFilter` HOOK → 取代 trigger/rule，in-place insert |
| COPY 優化 | `ProcessUtility_hook` → 直接寫入目標分區 |
| 並發遷移 | background worker，鎖競爭時 retry（sleep + retry loop） |

> 補充（Senior Dev）：Custom Scan API（PG 9.5+）是 pg_pathman 能繞過繼承+約束方案的關鍵。它允許 extension 註冊自訂的 plan node，在 planner 和 executor 層取代標準的 `Append` / `MergeAppend` node。`RuntimeAppend` 的核心優勢是 **runtime partition pruning**：planner 階段不決定掃哪些 partition，executor 階段根據實際查詢參數值（包括 subquery 的結果）動態選擇——這是傳統 `constraint_exclusion` 做不到的。
>
> 關於 shared_preload_libraries 順序：由於 pg_pathman 使用了 PG 內部的多個 HOOK（`ProcessUtility_hook`、`planner_hook`、`ExecutorStart_hook`），如果其他 extension（如 `pg_stat_statements`、`auto_explain`）也使用相同 HOOK，pg_pathman **必須排在最後**註冊，否則 HOOK chain 可能斷裂。

---

## 2. 核心特性

> **初學者導讀**：pg_pathman 的 13 項核心特性可以歸納為三大類：
> - **寫入加速**：特性 5/7/13 — 繞過傳統 trigger 機制，讓 INSERT、COPY 和分區鍵 UPDATE 直接命中目標分區。
> - **查詢加速**：特性 4/6 — 用二分搜尋（Range）或雜湊查表（Hash）取代逐表檢查，連子查詢條件都能正確定位分區。
> - **維運便利**：特性 2/8/10/11/12 — 自動擴展、非阻塞遷移、動態分割合併，降低 DBA 的人工負擔。

```mermaid
mindmap
  root((pg_pathman\n13項核心特性))
    寫入加速
      PartitionFilter HOOK
        繞過 trigger
        直接路由 INSERT
      COPY Hook
        COPY 直接寫目標分區
      分區鍵 UPDATE
        自動遷移行到正確分區
    查詢加速
      RuntimeAppend
        Binary Search O(log N)
        替代 Append 逐表掃描
      RuntimeMergeAppend
        替代 MergeAppend
      Subquery Pruning
        WHERE col = (SELECT ...)
        動態決定掃哪些分區
    維運便利
      自動分區擴展
        新值超出範圍時自動建新分區
      非阻塞遷移
        後台 worker 逐批搬資料
      分區 Split/Merge
        線上分割與合併
      FDW 分區
        分區可放在遠端伺服器
      自訂 Callback
        分區建立時自動建 index
```

| # | 特性 | 說明 |
|:-:|------|------|
| 1 | Range & Hash 分區 | 兩種分區模式，支援自動/手動管理 |
| 2 | 自動分區擴展 | Insert 新值超出既有分區範圍 → 自動建立新分區（僅 range） |
| 3 | 分區 column type | int, float, date, timestamp, 以及自訂 domain |
| 4 | Custom Scan | `RuntimeAppend` + `RuntimeMergeAppend` 動態分區選擇 |
| 5 | PartitionFilter HOOK | Insert 直接寫入目標分區，不用 trigger |
| 6 | Subquery 分區過濾 | `WHERE col = (SELECT ...)` 也能正確 pruning |
| 7 | COPY 直接寫分區 | `COPY FROM/TO` 不走主表觸發器 |
| 8 | 非阻塞式資料遷移 | `partition_table_concurrently()` 後台遷移，不鎖表 |
| 9 | FDW 支援 | 分區可放在 remote server（`postgres_fdw` 或任意 FDW） |
| 10 | Custom callback | `set_init_callback()` 設定分區建立時的回呼函數 |
| 11 | Split/Merge 分區 | range 分區支援分裂與合併 |
| 12 | Append/Prepend 分區 | 向前向後動態新增分區 |
| 13 | 分區 column 更新 | 支援更新分區鍵（觸發自動遷移到正確分區） |

---

## 3. 安裝與配置

> **初學者導讀**：pg_pathman 是一個需要編譯安裝的 C 語言 extension（非純 SQL）。它必須在 PostgreSQL 啟動前就載入到記憶體中（透過 `shared_preload_libraries` 設定），因為它需要在 PG 啟動時掛載多個內部 hook（攔截點），才能在查詢執行時介入並改寫執行計畫。這與一般只需 `CREATE EXTENSION` 就能用的 SQL-only extension（如 `pgcrypto`）有本質差異。

> **shared_preload_libraries 順序很重要！** pg_pathman 必須排在最後。原因是多個 extension 可能都想掛載同一個 hook（例如 planner_hook）。PG 的 hook 機制是單向鏈表——排在後面的 extension 會覆蓋前面的。但如果後面的 extension 沒有正確呼叫前一個 hook，鏈就會斷掉，導致其他 extension 失效。pg_pathman 內部有正確的 hook chain 處理，所以排在最後即可。

```mermaid
flowchart TD
    subgraph INSTALL["安裝步驟"]
        S1["1. git clone\n從 GitHub 下載原始碼"]
        S2["2. make + make install\n編譯 C 原始碼並安裝到 PG lib 目錄"]
        S3["3. 編輯 postgresql.conf\nshared_preload_libraries = '..., pg_pathman'"]
        S4["4. pg_ctl restart\n重啟 PostgreSQL 讓新設定生效"]
        S5["5. CREATE EXTENSION pg_pathman\n在目標資料庫內啟用"]
        S1 --> S2 --> S3 --> S4 --> S5
    end

    subgraph HOOKS["啟動時掛載的 Hook"]
        H1["ProcessUtility_hook\n攔截 COPY/DROP 等 DDL 操作"]
        H2["planner_hook\n攔截查詢計畫生成"]
        H3["ExecutorStart_hook\n執行期動態選分區"]
    end

    S4 --> HOOKS
```

```bash
git clone https://github.com/postgrespro/pg_pathman
cd pg_pathman
make USE_PGXS=1
make USE_PGXS=1 install
```

```ini
# postgresql.conf（pg_pathman 必須排在最後）
shared_preload_libraries = 'pg_stat_statements, pg_pathman'
```

```sql
CREATE EXTENSION pg_pathman;
```

### I. GUC 參數

| GUC | 預設 | 說明 |
|-----|:----:|------|
| `pg_pathman.enable` | on | 全域開關 |
| `pg_pathman.enable_runtimeappend` | on | RuntimeAppend custom node |
| `pg_pathman.enable_runtimemergeappend` | on | RuntimeMergeAppend custom node |
| `pg_pathman.enable_partitionfilter` | on | PartitionFilter insert hook |
| `pg_pathman.enable_auto_partition` | on | 自動擴展分區（per session） |
| `pg_pathman.insert_into_fdw` | postgres | FDW insert 支援：`disabled` / `postgres` / `any_fdw` |
| `pg_pathman.override_copy` | on | COPY hook |

---

## 4. 內部結構：Catalog Tables & Views

> **初學者導讀**：pg_pathman 最巧妙的設計之一是「不改動 PG 內建系統表」。傳統分區方案需要把分區資訊寫進 `pg_inherits`（繼承關係表）和 `pg_constraint`（約束表），這在升級 PG 版本或遷移 extension 時很容易出問題。pg_pathman 把分區定義全部存在自己建立的 `pathman_config` 表中，並且在記憶體中快取一份，查詢時直接從記憶體查——這比從系統表讀取快得多。

```mermaid
flowchart TD
    subgraph META["pg_pathman 內部 Metadata"]
        PC["pathman_config\n（分區定義主表）\n- 主表 OID（哪張表被分區）\n- 分區欄位名\n- 分區類型（1=Hash, 2=Range）\n- Range 間隔值"]
        PCP["pathman_config_params\n（可選參數覆蓋表）\n- 是否啟用主表掃描\n- 是否自動擴展\n- 自訂初始化回呼函數"]
        PL["pathman_partition_list（View）\n查詢所有分區及其邊界"]
        CT["pathman_concurrent_part_tasks（View）\n查詢正在執行的背景遷移任務"]
        CACHE["Memory Cache\n（啟動時從 config 表載入\n查詢時直接從記憶體讀取）"]
    end

    PC --> CACHE
    PCP --> CACHE
    CACHE --> PL
    CACHE --> CT

    style CACHE fill:#4a9,stroke:#333,color:#fff
```

### I. pathman_config（分區定義主表）

```sql
CREATE TABLE pathman_config (
    partrel        REGCLASS NOT NULL PRIMARY KEY,  -- 主表 OID
    attname        TEXT NOT NULL,                  -- 分區 column
    parttype       INTEGER NOT NULL,               -- 1=hash, 2=range
    range_interval TEXT                            -- range 分區的 interval
);
```

### II. pathman_config_params（可選覆蓋參數）

```sql
CREATE TABLE pathman_config_params (
    partrel        REGCLASS NOT NULL PRIMARY KEY,
    enable_parent  BOOLEAN NOT NULL DEFAULT TRUE,  -- planner 是否包含主表
    auto           BOOLEAN NOT NULL DEFAULT TRUE,  -- 是否自動擴展分區
    init_callback  REGPROCEDURE NOT NULL DEFAULT 0 -- 分區建立時回呼
);
```

### III. 管理 Views

| View | 用途 |
|------|------|
| `pathman_partition_list` | 列出所有分區、parent、range boundary（hash partition 為 NULL） |
| `pathman_concurrent_part_tasks` | 列出正在執行的背景遷移任務（user, pid, db, rel, processed rows, status） |

```sql
SELECT * FROM pathman_partition_list;
--  parent | partition | parttype | partattr | range_min | range_max

SELECT * FROM pathman_concurrent_part_tasks;
--  userid | pid | dbid | relid | processed | status
```

---

## 5. Range 分區管理 API

> **初學者導讀**：Range 分區適合時間序列資料——例如按月份或年份分割訂單表。每個分區負責一個連續範圍內的資料（如 2024 年 1 月的資料在分區 1，2 月在分區 2...）。pg_pathman 提供兩種建立方式：
> - **方式一（指定起始值 + 間隔 + 個數）**：告訴它「從 1 月開始、每月一個分區、建 12 個」，由它自動計算結束值。
> - **方式二（指定起始值 + 結束值 + 間隔）**：告訴它明確的開始和結束，由它自動計算需要多少個分區。
> 
> 核心 API 函數及其作用：
> - **建立分區**：從起始值按照固定間隔時間（如每月）自動建立一組連續的 range 分區
> - **非阻塞遷移**：在背景逐批搬移主表中的現有資料到正確的分區，不鎖定主表，應用程式可以繼續讀寫
> - **禁用主表**：遷移完成後告訴查詢計畫器「不用再掃描主表了」，確保查詢只命中分區表
> - **分裂分區**：將一個已有的 range 分區從中間切開，變成兩個更小的分區（資料自動重新分配）
> - **合併分區**：將兩個相鄰的 range 分區合併成一個（前提是範圍連續）
> - **向後新增**：在現有分區鏈的尾端再新增一個分區（使用原始設定的間隔大小）
> - **向前新增**：在現有分區鏈的頭端再新增一個分區

```mermaid
flowchart TD
    subgraph CREATE["建立 Range 分區（兩種方式）"]
        A["方式一：指定 start + interval + count\n例：從 10/25 起，每月一個，共 24 個"]
        B["方式二：指定 start + end + interval\n例：從 10/25 到 12/25，每月一個"]
    end

    subgraph MIGRATE["資料遷移流程"]
        C["Step 1: 建立主表\n（分區欄位必須 NOT NULL）"]
        D["Step 2: 插入既有資料"]
        E["Step 3: 建立分區定義\n（不立即遷移資料）"]
        F["Step 4: 非阻塞遷移\n（背景 worker 逐批搬資料）"]
        G["Step 5: 禁用主表\n（確保查詢只命中分區）"]
        C --> D --> E --> F --> G
    end

    subgraph MAINTAIN["分區維護操作"]
        SPLIT["分裂\n一個 ↦ 兩個"]
        MERGE["合併\n兩個 ↦ 一個"]
        APPEND["向後新增一個"]
        PREPEND["向前新增一個"]
    end

    CREATE --> MIGRATE --> MAINTAIN
```

### I. 建立 Range 分區

**方式一：指定起始值 + interval + 分區數**（核心 API：建立 range 分區）

```sql
create_range_partitions(
    relation       REGCLASS,     -- 主表 OID
    attribute      TEXT,         -- 分區 column 名
    start_value    ANYELEMENT,   -- 起始值
    p_interval     ANYELEMENT,   -- interval（任意 type）
    p_count        INTEGER DEFAULT NULL,
    partition_data BOOLEAN DEFAULT TRUE  -- 是否立即遷移資料
);

-- 時間類型專用多態版本（p_interval 為 INTERVAL type）
create_range_partitions(
    relation       REGCLASS, 
    attribute      TEXT,
    start_value    ANYELEMENT,
    p_interval     INTERVAL,     -- 用 PostgreSQL INTERVAL type（'1 month'）
    p_count        INTEGER DEFAULT NULL,
    partition_data BOOLEAN DEFAULT TRUE
);
```

**方式二：指定起始值 + 終值 + interval**

```sql
create_partitions_from_range(
    relation       REGCLASS,
    attribute      TEXT,
    start_value    ANYELEMENT,
    end_value      ANYELEMENT,
    p_interval     ANYELEMENT,   -- 或 INTERVAL
    partition_data BOOLEAN DEFAULT TRUE
);
```

### II. Range 分區完整示例

```sql
-- Step 1: 建立主表（分區 column 必須 NOT NULL）
CREATE TABLE part_test (
    id int,
    info text,
    crt_time timestamp NOT NULL
);

-- Step 2: 插入已有資料
INSERT INTO part_test
    SELECT id, md5(random()::text),
           clock_timestamp() + (id || ' hour')::interval
    FROM generate_series(1, 10000) t(id);

-- Step 3: 建立分區（24 個，每月一個；不立即遷移資料）
SELECT create_range_partitions(
    'part_test'::regclass,
    'crt_time',
    '2016-10-25 00:00:00'::timestamp,
    interval '1 month',
    24,
    false  -- 不遷移資料
);

-- Step 4: 非阻塞式遷移
SELECT partition_table_concurrently(
    'part_test'::regclass,
    10000,  -- batch_size
    1.0     -- sleep_time
);
-- 停止遷移：SELECT stop_concurrent_part_task('part_test');

-- Step 5: 遷移完成後禁用主表（planner 不再考慮主表）
SELECT set_enable_parent('part_test'::regclass, false);
```

遷移後查詢只有目標分區被掃描：

```sql
EXPLAIN SELECT * FROM part_test
WHERE crt_time = '2016-10-25 00:00:00'::timestamp;
--  Append
--    ->  Seq Scan on part_test_1
--          Filter: (crt_time = '2016-10-25 ...')
```

**強制建議：**
1. 分區 column 必須有 NOT NULL constraint
2. 分區個數必須能覆蓋所有已存在記錄
3. 使用非阻塞式遷移（`partition_table_concurrently`）
4. 遷移完成後調用 `set_enable_parent(rel, false)` 禁用主表

### III. Split / Merge / Append / Prepend

```sql
-- 分裂範圍分區：在 split_value 處切成兩半
SELECT split_range_partition(
    'part_test_1'::regclass,        -- 分區 OID
    '2016-11-10 00:00:00'::timestamp, -- 分裂值
    'part_test_1_2'                 -- 新分區名
);
-- 分裂後：part_test_1  [2016-10-25, 2016-11-10)
--         part_test_1_2 [2016-11-10, 2016-11-25)
-- 資料自動遷移

-- 合併相鄰分區（必須相鄰）
SELECT merge_range_partitions(
    'part_test_1'::regclass,
    'part_test_1_2'::regclass
);

-- 向後新增分區（按原始 interval）
SELECT append_range_partition('part_test'::regclass);
-- 返回: public.part_test_25

-- 向前新增分區
SELECT prepend_range_partition('part_test'::regclass);
-- 返回: public.part_test_26
```

> 補充（Senior Dev）：`append_range_partition` 和 `prepend_range_partition` 使用 `pathman_config.range_interval` 中儲存的原始 interval。如果你需要在特定 tablespace 中建立分區，可以在 session 層面預先 `SET local default_tablespace = 'tbs_cold'`。冷熱分離是分區表的常見場景——近期分區放 SSD tablespace、歷史分區放 HDD。

---

## 6. Hash 分區管理 API

> **初學者導讀**：Hash 分區與 Range 分區的核心差異在於「資料如何決定要去哪個分區」：
> - **Range 分區**：根據分區欄位的值落在哪個範圍內來決定（如 2024-06-15 落在 6 月分區）。適合有順序性的資料，查詢時常用範圍條件（BETWEEN、>=、<）。
> - **Hash 分區**：對分區欄位的值做雜湊計算，根據雜湊結果分配到不同分區。適合需要均勻分散的場景（如按 user_id 分區），但不支援範圍查詢的 pruning。
>
> pg_pathman 的 Hash 分區有一個巧妙設計：**使用者不需要在 WHERE 中寫雜湊表達式**。你只需要寫 `WHERE crt_time = '2024-01-01'`，pg_pathman 內部會自動計算雜湊值並定位到正確的分區——這對使用者完全透明。

```mermaid
flowchart TD
    subgraph HASH["Hash 分區原理"]
        Q["查詢: WHERE crt_time = '2024-01-01'"]
        HF["內部雜湊函數\n將 crt_time 轉換為雜湊值"]
        MOD["雜湊值 % 分區數\n（如 128 個分區則 mod 128）"]
        PIDX["得到分區編號\n如: 分區 70"]
        SCAN["只掃描分區 70"]
        Q --> HF --> MOD --> PIDX --> SCAN
    end

    subgraph COMPARE["Range vs Hash 定位速度"]
        RNG["Range 分區\n二分搜尋 O(log N)\n適合時間序列、有序查詢"]
        HSH["Hash 分區\n雜湊查找 O(1)\n適合均勻分散、等值查詢"]
    end

    style HF fill:#4a9,stroke:#333,color:#fff
    style PIDX fill:#4a9,stroke:#333,color:#fff
```

**建立 Hash 分區的核心 API：**

```sql
create_hash_partitions(
    relation         REGCLASS,   -- 主表 OID
    attribute        TEXT,       -- 分區 column（不限 int，自動 hash）
    partitions_count INTEGER,    -- 分區數
    partition_data   BOOLEAN DEFAULT TRUE
);
```

### I. Hash 分區完整示例

```sql
CREATE TABLE part_test (id int, info text, crt_time timestamp NOT NULL);

INSERT INTO part_test
    SELECT id, md5(random()::text),
           clock_timestamp() + (id || ' hour')::interval
    FROM generate_series(1, 10000) t(id);

-- 建立 128 個 hash 分區
SELECT create_hash_partitions(
    'part_test'::regclass, 'crt_time', 128, false);

-- 非阻塞遷移 + 禁用主表
SELECT partition_table_concurrently('part_test'::regclass, 10000, 1.0);
SELECT set_enable_parent('part_test'::regclass, false);
```

Hash 分區的特點：
- pg_pathman **不需要在 WHERE 中使用 hash 表達式**（`SELECT * FROM part_test WHERE crt_time = '...'::timestamp` 也能正確 pruning）——內部自動計算 hash 值對應的分區
- CHECK constraint 是 `get_hash_part_idx(timestamp_hash(crt_time), 128) = N`
- 分區 column 不限 int type，會自動用 hash 函數轉換

---

## 7. 效能對比數據

> **初學者導讀**：這是最能體現 pg_pathman 價值的章節。兩個數字說明一切：寫入速度快 4 倍、子查詢也能正確定位分區（傳統方案根本不支援）。關鍵原因：
> - **寫入快**：傳統方案的 trigger 對每一行 INSERT 都要執行 PL/pgSQL 函數來判斷該去哪個分區，這是逐行處理的。pg_pathman 在 C 層面的 hook 中直接攔截 INSERT 指令，一次性決定目標分區並重寫執行計畫，效能天差地別。
> - **子查詢 pruning**：傳統方案的 constraint exclusion 發生在查詢計畫階段，那時候子查詢的結果還沒出來，所以無法判斷。pg_pathman 的 RuntimeAppend 把決策延後到執行階段，此時子查詢已經有結果了，自然能正確定位。

```mermaid
flowchart LR
    subgraph INSERT_PERF["INSERT 效能對比（2000 萬行，64 連線）"]
        TRAD["傳統 Trigger 方案\nTPS: 54,517"]
        PATH["pg_pathman PartitionFilter\nTPS: 223,484"]
        TRAD -.->|"4.1 倍"| PATH
        style PATH fill:#4a9,stroke:#333,color:#fff
        style TRAD fill:#faa,stroke:#333
    end

    subgraph QUERY["子查詢 Pruning 對比"]
        TQ["傳統方案\nWHERE id = (SELECT ...)\n❌ 掃描所有分區"]
        PQ["pg_pathman RuntimeAppend\nWHERE id = (SELECT ...)\n✅ 只掃一個分區"]
        style PQ fill:#4a9,stroke:#333,color:#fff
        style TQ fill:#faa,stroke:#333
    end
```

### I. Insert 效能（2000 萬 row 表，64 connections）

| 方案 | TPS | 說明 |
|------|------|------|
| Traditional (trigger) | 54,517 | 每個 insert 觸發 trigger 判斷分區 |
| **pg_pathman (PartitionFilter)** | **223,484** | HOOK 直接寫入目標分區 |

pg_pathman 的寫入效能是傳統方案的 **4.1 倍**。

### II. 查詢效能：Subquery 過濾

傳統 inheritance + constraint 方案 **不支援 subquery partition pruning**。pg_pathman 的 `RuntimeAppend` 在 executor 階段根據 subquery 結果動態選擇分區：

```sql
CREATE TABLE partitioned_table (id INT NOT NULL, payload REAL);
INSERT INTO partitioned_table
    SELECT generate_series(1, 1000), random();
SELECT create_hash_partitions('partitioned_table', 'id', 100);
SELECT set_enable_parent('partitioned_table', false);
```

```sql
-- Subquery in WHERE：只掃相關分區
EXPLAIN (COSTS OFF, ANALYZE)
SELECT * FROM partitioned_table
WHERE id = (SELECT * FROM some_table LIMIT 1);

--  Custom Scan (RuntimeAppend) (actual time=0.051..0.053 rows=1 loops=1)
--    InitPlan 1 (returns $0)
--      ->  Limit (actual time=0.017..0.017 rows=1 loops=1)
--            ->  Seq Scan on some_table
--    ->  Seq Scan on partitioned_table_70  ← 只掃一個分區！
--          Filter: (id = $0)
```

```sql
-- ANY (subquery)：對每個 hash value 只掃對應分區
EXPLAIN (COSTS OFF, ANALYZE)
SELECT * FROM partitioned_table
WHERE id = ANY (SELECT * FROM some_table LIMIT 10);

--  Nested Loop (rows=10 loops=1)
--    ->  HashAggregate (rows=10)
--    ->  Custom Scan (RuntimeAppend) (rows=1 loops=10)
--          ->  Seq Scan on partitioned_table_88 (loops=2)   ← 只掃 8 個分區（非 100 個）
--          ->  Seq Scan on partitioned_table_72 (loops=1)
--          ...
```

> 補充（Senior Dev）：Prepared statement 與 simple query 在 pg_pathman 下的行為差異：prepared statement 時，custom scan 的 pruning 在 `EXPLAIN` 中不可見（顯示為掃描所有分區），但 `EXPLAIN ANALYZE` 中顯示實際只掃了一個分區。這是因為 prepared statement 的 plan 在 bind 階段才獲得參數值，planner 的 static plan 保守地包含所有分區，executor 的 RuntimeAppend 在執行時才做 pruning。從 perf top 來看，prepared statement 下的 LWLock 競爭更高（尤其 `LWLockAcquire`），與 `LockReleaseAll` 在每個 bind 週期都觸發有關。

---

## 8. Callback 機制

> **初學者導讀**：當 pg_pathman 自動建立一個新分區（例如寫入超出範圍的資料觸發自動擴展），新分區預設只是一個空的表格，沒有 index、沒有 storage 參數。Callback（回呼函數）就是為了解決這個問題：你可以註冊一個自訂函數，每當新分區建立時，pg_pathman 會自動呼叫它，讓你在新分區上自動建立 index、設定 fillfactor 等。

```mermaid
flowchart TD
    A["新資料寫入\n超出既有分區範圍"] --> B["pg_pathman 自動建立新分區"]
    B --> C["檢查是否有註冊 callback"]
    C -->|"有"| D["呼叫自訂 callback 函數\n傳入分區資訊 (JSONB)"]
    D --> E["callback 執行自訂邏輯\n例如: CREATE INDEX, SET fillfactor"]
    C -->|"無"| F["新分區為空表\n無 index 等設定"]
    E --> G["新分區就緒\n資料寫入"]
    F --> G

    style D fill:#4a9,stroke:#333,color:#fff
```

註冊 callback 的核心步驟（設定分區初始化函數）：

```sql
-- 註冊 callback
SELECT set_init_callback('part_test'::regclass, 'my_partition_init');

-- callback function signature
CREATE FUNCTION my_partition_init(args JSONB) RETURNS VOID AS $$
BEGIN
    -- args for range partition:
    -- {"parent": "part_test", "parttype": "2",
    --  "partition": "part_test_4", "range_max": "401", "range_min": "301"}

    -- args for hash partition:
    -- {"parent": "part_test", "parttype": "1", "partition": "part_test_0"}

    EXECUTE format('CREATE INDEX ON %I (id)', args->>'partition');
END;
$$ LANGUAGE plpgsql;
```

---

## 9. 注意事項與限制

> **初學者導讀**：在使用 pg_pathman 之前，有幾個「踩坑點」需要特別注意：
> 1. **分區欄位必須是 NOT NULL**：NULL 值無法確定屬於哪個分區（哪個範圍都不包含 NULL，哪個 hash bucket 也不對應 NULL），所以 pg_pathman 強制要求 NOT NULL。
> 2. **EXPLAIN 可能誤導你**：使用 prepared statement 時，`EXPLAIN`（不帶 ANALYZE）會顯示掃描所有分區，讓你以為沒做 pruning。但 `EXPLAIN ANALYZE` 會顯示真實行為（只掃一個分區）。原因是 prepared statement 的參數值在執行時才知道，planner 在計畫階段只能保守地全部包含。
> 3. **必須排在 shared_preload_libraries 最後**：因為它要掛載多個 PostgreSQL 內部 hook，如果排在前面，後面的 extension 可能覆蓋它的 hook，導致功能失效。

```mermaid
flowchart TD
    subgraph MUST["必要條件"]
        N1["PG 9.5+（需要 Custom Scan API）"]
        N2["分區欄位 NOT NULL"]
        N3["shared_preload_libraries 中\npg_pathman 排最後"]
    end
    subgraph CAUTION["注意事項"]
        C1["EXPLAIN 可能誤導\nprepared statement 下\n顯示掃所有分區"]
        C2["非阻塞遷移 vs 立即遷移\n建議用非阻塞版\n避免長時間鎖表"]
        C3["分區上限\n自動擴展無上限時\n可能產生過多分區\n建議定期合併"]
        C4["PG 10+ 內建分區\npg_pathman 仍可使用\n但功能逐漸重疊"]
    end
    MUST -->|"滿足後使用"| CAUTION
```

| 事項 | 說明 |
|------|------|
| **PG 版本** | 僅支援 9.5+（需要 Custom Scan API） |
| **prepared statement** | `EXPLAIN` 靜態 plan 顯示掃所有分區（誤導），`EXPLAIN ANALYZE` 才顯示實際 pruning 結果。此與 LWLock 競爭有關（已提 issue） |
| **shared_preload_libraries 順序** | pg_pathman 必須排最後 |
| **分區 column** | 必須有 NOT NULL constraint |
| **分區數上限** | range 分區的自動擴展若無上限可能產生極多 partition，建議定期合併 |
| **tablespace** | 可用 `SET local default_tablespace = 'tbs_name'` 在建立分區前指定 |
| **disable_pathman_for** | 無 session 級別關閉，需用 `SET pg_pathman.enable = off` |
| **資料遷移** | 建議使用 `partition_table_concurrently()`，避免 `partition_data = true` 時的 blocking 遷移 |
| **FDW partition** | 支援透過 `pg_pathman.insert_into_fdw` 控制 write routing |
| **PG 10+ 原生分區** | PG 10 引入 declarative partitioning，pg_pathman 仍可在 PG 10 使用。PG 11+ 的原生分區逐步補強（hash partition PG 11、default partition PG 11、partition pruning improvement PG 12） |

---

## 參考

1. [pg_pathman GitHub](https://github.com/postgrespro/pg_pathman)
2. [Oleg Bartunov - pg_pathman RuntimeAppend Blog](http://akorotkov.github.io/blog/2016/06/15/pg_pathman-runtime-append/)
3. [PostgreSQL Custom Scan API Wiki](https://wiki.postgresql.org/wiki/CustomScanAPI)
4. [PG 9.6 FDW docs](https://www.postgresql.org/docs/9.6/static/postgres-fdw.html)
5. `\sf` 查看 pg_pathman 內部 function 原始碼

---

# 三、PostgreSQL pg_shard — 資料庫分片（Sharding）

> 來源：[digoal - pg_shard PostgreSQL 數據庫分片 (2015-09-28)](https://github.com/digoal/blog/blob/master/201509/20150928_01.md)
>
> 官方：[CitusData / pg_shard v1.2.2](https://github.com/citusdata/pg_shard/tree/v1.2.2)

---

> 補充（Senior Dev）：pg_shard 是 Citus 的前身。2016 年 CitusData 將 pg_shard 合併進 Citus extension（開源版），Citus 現已成為 PostgreSQL 生態最成熟的分佈式方案。pg_shard 的架構（hash sharding + metadata table + master_copy_shard_placement）基本奠定了 Citus 的核心設計。本文整理的安裝流程和限制對理解 Citus 底層仍有參考價值。現代部署中 pg_shard 已被 Citus 取代，不建議在新項目使用。

---

## 1. 環境準備與安裝

> **初學者導讀**：在進入 pg_shard 的具體操作前，先建立一個高層次的心智模型——分散式資料庫的核心概念：
> - **Shard（分片）**：把一張大表的資料切成多份，每份放在不同機器上。切的方法通常是對某個欄位（如 customer_id）做雜湊，決定每行資料歸哪個 shard。
> - **Master/Coordinator（主節點）**：負責接收 SQL 查詢、解析後決定要發給哪些 worker、收集結果回傳給用戶。
> - **Worker（工作節點）**：實際存放 shard 資料的機器，只聽 master 的指令執行查詢。
> - **Replica（副本）**：同一個 shard 的多份拷貝，放在不同 worker 上。當一個 worker 掛掉，可以從另一個 worker 的副本繼續服務。

```mermaid
flowchart TD
    subgraph CLUSTER["pg_shard 叢集架構"]
        MASTER["Master（主節點）\n- 接收 SQL 查詢\n- 解析並路由到 worker\n- 彙整結果回傳\n- 管理 metadata"]
        W1["Worker 1 (port 1922)\n存放部分 shard 資料"]
        W2["Worker 2 (port 1923)\n存放部分 shard 資料"]
        W3["Worker 3 (port 1924)\n存放部分 shard 資料"]
        W4["Worker 4 (port 1925)\n存放部分 shard 資料"]
        MASTER -->|"查詢路由"| W1 & W2 & W3 & W4
    end

    subgraph DATA["資料分佈示意（Hash Sharding on customer_id）"]
        TBL["customer_reviews 總表\n（分佈式表，共 16 個 shard）"]
        S1["Shard 0-3 在 Worker 1/2"]
        S2["Shard 4-7 在 Worker 2/3"]
        S3["Shard 8-11 在 Worker 3/4"]
        S4["Shard 12-15 在 Worker 4/1"]
        TBL --> S1 & S2 & S3 & S4
    end
```

### I. GCC 版本要求

pg_shard 使用了 GCC 4.6+ 的特性。若系統 GCC < 4.6，需先安裝高版本：

```bash
yum install -y gmp mpfr libmpc libmpc-devel

wget http://gcc.cybermirror.org/releases/gcc-4.9.3/gcc-4.9.3.tar.bz2
tar -jxvf gcc-4.9.3.tar.bz2
cd gcc-4.9.3
./configure --prefix=/opt/gcc4.9.3
make && make install

# 配置 linker 路徑
vi /etc/ld.so.conf
# 加入:
/opt/gcc4.9.3/lib
/opt/gcc4.9.3/lib64

ldconfig
ldconfig -p | grep gcc

# 配置 PATH
vi /etc/profile
# 加入:
export PATH=/opt/gcc4.9.3/bin:$PATH
```

### II. 安裝 pg_shard v1.2.2

```bash
git clone https://github.com/citusdata/pg_shard.git
cd pg_shard/
git checkout master  # v1.2.2 at commit 7e6103f

. /home/postgres/.bash_profile
make clean; make; make install
```

> 補充（Senior Dev）：現代 Citus 安裝只需 `apt install postgresql-17-citus` 或 `CREATE EXTENSION citus;`，不再需要手動編譯。但 pg_shard 時期的手編流程對理解 extension 載入機制有幫助——`shared_preload_libraries` 中載入的 extension 會在 PostgreSQL 啟動時掛載 hook（planner hook / executor hook），這是 Citus 能攔截並重寫 distributed query 的技術基礎。

---

## 2. 叢集配置

> **初學者導讀**：在開始使用 pg_shard 之前，必須先建立一個「叢集」——也就是告訴 master 節點有哪些 worker 可以存放資料。這跟單機 PG 的根本差異在於：pg_shard 需要知道資料該存到哪台機器上。配置包含三個步驟：
> 1. **定義 worker 列表**（pg_worker_list.conf）：告訴 master 有哪些 worker 節點可用。
> 2. **設定認證**（pg_hba.conf）：確保 master 可以無密碼連線到所有 worker（在本地測試環境中）。
> 3. **載入 extension**：在 master 的 shared_preload_libraries 中加入 pg_shard，讓它在 PG 啟動時掛載 hook。

```mermaid
flowchart TD
    subgraph SETUP["叢集配置步驟"]
        S1["1. 建立 pg_worker_list.conf\n寫入所有 worker 的 host:port"]
        S2["2. 設定 pg_hba.conf\n在所有 worker 設 trust 認證"]
        S3["3. 修改 postgresql.conf\nshared_preload_libraries = 'pg_shard'"]
        S4["4. 重啟 Master\npg_ctl restart"]
        S5["5. 在所有節點建立一致的\nrole / database / schema"]
        S6["6. CREATE EXTENSION pg_shard\n在 master 端啟用"]
        S1 --> S2 --> S3 --> S4 --> S5 --> S6
    end

    subgraph RESULT["配置完成後的拓撲"]
        M["Master\n(port 1921)"]
        W1A["Worker 1\n(port 1922)"]
        W2A["Worker 2\n(port 1923)"]
        W3A["Worker 3\n(port 1924)"]
        W4A["Worker 4\n(port 1925)"]
        M -->|"讀寫路由"| W1A & W2A & W3A & W4A
    end
```

### I. 拓撲：1 Master + 4 Worker

```bash
# 在 Master 的 $PGDATA 中建立 worker 列表
vi pg_worker_list.conf
```

```
localhost 1922
localhost 1923
localhost 1924
localhost 1925
```

### II. 確保連線無密碼

所有 worker 節點的 `pg_hba.conf` 對 local 連線設為 trust：

```
local   all   all   trust
```

### III. Master 端載入 pg_shard

```bash
vi $PGDATA/postgresql.conf
# 加入:
shared_preload_libraries = 'pg_shard'

pg_ctl restart -m fast
```

### IV. 建立分佈式環境

在所有節點（master + workers）建立一致的 role、database、schema。然後在 master 建立 extension：

```sql
CREATE EXTENSION pg_shard;
```

---

## 3. 建立分佈式表

> **初學者導讀**：在 pg_shard 中，將一張普通表變成「分佈式表」需要三個步驟，每一步都對應一個核心概念：
> 1. **建立普通表結構**：跟一般 CREATE TABLE 一模一樣，但要注意——分佈鍵（distribution column）的選擇極其重要，它決定了資料如何散佈到各個 shard。好的分佈鍵應該讓資料均勻分散（避免熱點），且查詢時常用在 WHERE 和 JOIN 條件中。
> 2. **註冊為分佈式表**：告訴 pg_shard 這張表要分佈，以及用哪個欄位做分佈鍵（此例用 customer_id）。pg_shard 內部會將 customer_id 做雜湊計算，映射到 int32 的數值空間。
> 3. **建立實際的 shard**：在 worker 節點上建立實際存放資料的表格（shard table）。每個 shard 管理一個區段的雜湊值範圍。例如 16 個 shard 就把整個 int32 空間（約 43 億個值）均分為 16 段，每段約 2.68 億個值。

```mermaid
flowchart TD
    subgraph STEPS["三步驟建立分佈式表"]
        A["Step 1: CREATE TABLE\n定義表結構\n（跟一般建表一樣）"]
        B["Step 2: 註冊為分佈式表\n指定分佈鍵（distribution column）\n用於決定每行資料去哪個 shard"]
        C["Step 3: 建立 worker shard\n在 worker 節點上實際建表\n指定 shard 數量 + replica 數量"]
        A --> B --> C
    end

    subgraph HASHING["雜湊分佈原理"]
        ROW["一行資料\ncustomer_id = 'ABC123'"]
        HASH_FN["雜湊函數\n將 customer_id 轉為 int32 值"]
        RANGE["對應到 shard 的範圍\n例如值落在 -536870918 ~ -268435464\n→ Shard 10006"]
        WORKER["Shard 10006 的兩個副本\n分別在 Worker 3 和 Worker 4"]
        ROW --> HASH_FN --> RANGE --> WORKER
    end

    subgraph META["Metadata 三張表"]
        PART["pgs_distribution_metadata.partition\n記錄: 哪張表、分佈方法(hash)、分佈鍵"]
        SHARD["pgs_distribution_metadata.shard\n記錄: 每個 shard 的 hash 值範圍"]
        PLACE["pgs_distribution_metadata.shard_placement\n記錄: 每個 shard 實際放在哪個 worker"]
        PART --> SHARD --> PLACE
    end
```

### I. 建立測試表

```sql
CREATE TABLE customer_reviews
(
    customer_id TEXT NOT NULL,
    review_date DATE,
    review_rating INTEGER,
    review_votes INTEGER,
    review_helpful_votes INTEGER,
    product_id CHAR(10),
    product_title TEXT,
    product_sales_rank BIGINT,
    product_group TEXT,
    product_category TEXT,
    product_subcategory TEXT,
    similar_product_ids CHAR(10)[]
);
```

> 建議：在定義 worker table 前將所有 index、constraint 確定好。後續添加 index 需在所有 worker 節點手動執行。

### II. 設為分佈式表（Hash Sharding on customer_id）

```sql
SELECT master_create_distributed_table('customer_reviews', 'customer_id');
```

### III. 在 Worker 節點建立 Shard

16 個 shard，每個 shard 2 個 replica：

```sql
SELECT master_create_worker_shards('customer_reviews', 16, 2);
```

### IV. 查看 Metadata

```sql
\dt pgs_distribution_metadata.*
```

```
         Schema           |      Name       | Type  |  Owner
--------------------------+-----------------+-------+----------
 pgs_distribution_metadata | partition       | table | postgres
 pgs_distribution_metadata | shard           | table | postgres
 pgs_distribution_metadata | shard_placement | table | postgres
(3 rows)
```

**partition 表**（記錄分佈方法）：

```sql
SELECT * FROM pgs_distribution_metadata.partition;
```

```
 relation_id | partition_method |     key
-------------+------------------+-------------
       42067 | h                | customer_id
(1 row)
```

```sql
SELECT 42067::regclass;  -- customer_reviews
```

**shard 表**（記錄每個 shard 的 hash range，int32 空間均分為 16 段）：

```sql
SELECT * FROM pgs_distribution_metadata.shard;
```

```
  id   | relation_id | storage |  min_value  |  max_value
-------+-------------+---------+-------------+-------------
 10000 |       42067 | t       | -2147483648 | -1879048194
 10001 |       42067 | t       | -1879048193 | -1610612739
 10002 |       42067 | t       | -1610612738 | -1342177284
 10003 |       42067 | t       | -1342177283 | -1073741829
 10004 |       42067 | t       | -1073741828 | -805306374
 10005 |       42067 | t       | -805306373  | -536870919
 10006 |       42067 | t       | -536870918  | -268435464
 10007 |       42067 | t       | -268435463  | -9
 10008 |       42067 | t       | -8          | 268435446
 10009 |       42067 | t       | 268435447   | 536870901
 10010 |       42067 | t       | 536870902   | 805306356
 10011 |       42067 | t       | 805306357   | 1073741811
 10012 |       42067 | t       | 1073741812  | 1342177266
 10013 |       42067 | t       | 1342177267  | 1610612721
 10014 |       42067 | t       | 1610612722  | 1879048176
 10015 |       42067 | t       | 1879048177  | 2147483647
(16 rows)
```

> 分佈方式：對 `customer_id` 做 hash → 映射到 int32 空間，按 range 均分到 16 個 shard。

**shard_placement 表**（記錄每個 shard 的 replica 位置，32 row = 16 shard × 2 replica）：

```sql
SELECT * FROM pgs_distribution_metadata.shard_placement;
```

```
 id | shard_id | shard_state | node_name | node_port
----+----------+-------------+-----------+-----------
  1 |    10000 |           1 | localhost |      1922
  2 |    10000 |           1 | localhost |      1923
  3 |    10001 |           1 | localhost |      1923
  4 |    10001 |           1 | localhost |      1924
  5 |    10002 |           1 | localhost |      1924
  6 |    10002 |           1 | localhost |      1925
  7 |    10003 |           1 | localhost |      1925
  8 |    10003 |           1 | localhost |      1922
  9 |    10004 |           1 | localhost |      1922
 10 |    10004 |           1 | localhost |      1923
 11 |    10005 |           1 | localhost |      1923
 12 |    10005 |           1 | localhost |      1924
 13 |    10006 |           1 | localhost |      1924
 14 |    10006 |           1 | localhost |      1925
 15 |    10007 |           1 | localhost |      1925
 16 |    10007 |           1 | localhost |      1922
 17 |    10008 |           1 | localhost |      1922
 18 |    10008 |           1 | localhost |      1923
 19 |    10009 |           1 | localhost |      1923
 20 |    10009 |           1 | localhost |      1924
 21 |    10010 |           1 | localhost |      1924
 22 |    10010 |           1 | localhost |      1925
 23 |    10011 |           1 | localhost |      1925
 24 |    10011 |           1 | localhost |      1922
 25 |    10012 |           1 | localhost |      1922
 26 |    10012 |           1 | localhost |      1923
 27 |    10013 |           1 | localhost |      1923
 28 |    10013 |           1 | localhost |      1924
 29 |    10014 |           1 | localhost |      1924
 30 |    10014 |           1 | localhost |      1925
 31 |    10015 |           1 | localhost |      1925
 32 |    10015 |           1 | localhost |      1922
(32 rows)
```

- `shard_state = 1` 表示正常
- replica 分佈策略確保任意 worker 掛掉時，每個 shard 仍有一個健康副本在其他 worker

---

## 4. 限制（Limitations）

> **初學者導讀**：pg_shard 是一個「早期產品」，它的限制非常多，但這些限制恰恰幫助我們理解分散式資料庫的核心挑戰：
> - **為什麼不支援子查詢和 prepared statement？** 因為 pg_shard 需要完整解析 SQL 語句才能決定路由——子查詢的結果在解析時還不知道，而 prepared statement 的參數值在執行時才知道。現代 Citus 透過 adaptive executor（自適應執行器）解決了這些問題。
> - **為什麼不支援跨 shard 的分散式交易？** 因為要確保跨機器的資料一致性，需要兩階段提交（2PC）等分散式協議，這在 pg_shard 的設計範圍之外。Citus 從 6.0 開始支援 2PC。
> - **Master 單點瓶頸**：所有查詢都要經過 master 節點轉發，當流量大時 master 的網路頻寬和 CPU 會成為瓶頸。這也是為什麼後來 Citus 推出了 MX 模式（每個 worker 也能直接接受查詢）。

```mermaid
flowchart TD
    subgraph SHORT["短期限制（v1.2.2，後續版本預計解決）"]
        S1["不支援 DDL (ALTER TABLE)\n需手動在所有 worker 執行"]
        S2["不支援 DROP TABLE\n無自動清理 shard 機制"]
        S3["不支援 INSERT INTO ... SELECT ..."]
    end

    subgraph ARCH["架構層級限制（設計決定，不打算支援）"]
        A1["❌ 跨 shard 分散式交易\n(A→B 轉帳，shard key 不同)"]
        A2["❌ 非分佈鍵的唯一約束/外鍵"]
        A3["❌ 分散式 JOIN\n（需改用 CitusDB）"]
    end

    subgraph BUG["使用層級限制"]
        B1["❌ 子查詢 (Subquery)"]
        B2["❌ 非常數表達式 (now(), random())"]
        B3["❌ Prepared Statement / Bind Variable"]
        B4["❌ Extended Query Protocol\n（pgbench -M extended 失敗）"]
    end

    subgraph BOTTLENECK["架構瓶頸"]
        BN["Master 單點瓶頸\n所有查詢必經 master\n無對等備援"]
    end
```

### I. 不支援子查詢

```sql
INSERT INTO customer_reviews SELECT generate_series(1, 100);
-- ERROR: 0A000: cannot perform distributed planning for the given query
-- DETAIL: Subqueries are not supported in distributed queries.
```

### II. 不支援 non-constant expression / 變數

```sql
INSERT INTO customer_reviews VALUES ('a', now());
-- ERROR: 0A000: cannot plan sharded modification containing values
-- which are not constants or constant expressions
```

### III. 不支援 Prepared Statement / Bind Variable

```sql
PREPARE a (text) AS INSERT INTO customer_reviews VALUES ($1);
-- ERROR: 0A000: PREPARE commands on distributed tables are unsupported
```

使用 `pgbench -M extended`（extended query protocol）會直接失敗：

```
Client 2 aborted in state 1: ERROR: unrecognized node type: 2100
```

必須使用 `-M simple`（simple query protocol）才能正常寫入。

### IV. 效能測試

`pgbench -M simple` 寫入測試（8 client / 8 thread / 1000s）：

```
progress: 2.8 s, 0.7 tps,   lat 2568.361 ms
progress: 3.2 s, 68.3 tps,  lat 578.633 ms
progress: 3.2 s, 264.5 tps, lat 8.263 ms
progress: 4.0 s, 1193.6 tps, lat 8.561 ms
progress: 5.0 s, 1255.6 tps, lat 6.376 ms
progress: 6.0 s, 1277.5 tps, lat 6.263 ms
```

暖機後穩定在 ~1200-1400 tps。

### V. 完整限制清單

**中長期（架構決定，不打算支援）：**

- 跨 shard 的分散式 transaction（如 A 轉帳到 B，shard key 不同）
- 非 distribution column 的 Unique constraint / Foreign Key constraint
- 分散式 JOIN

> 原文官方說明：If you'd like to run complex analytic queries, please consider upgrading to CitusDB.

**短期（未來版本預計支援）：**

- 不支援 DDL（ALTER TABLE）：需手動在所有 worker 節點 propagate
- 不支援 DROP TABLE（未來版本將加入 shard cleanup command）
- 不支援 `INSERT INTO ... SELECT ...`

**單點瓶頸：Master 無法做對等備援**

Master 同時處理查詢路由和 metadata 管理，容易成為 network bandwidth 和 CPU 瓶頸。德哥認為連接池代理（connection pool proxy）方案更好——輕量、易於對等部署、效能線性增長。

> 德哥觀點：JDBC 9.4（支援 load balance + failover）+ plproxy 可完美實現真正效能線性增長的資料庫分片，前提是不需要跨庫 transaction / JOIN / 唯一約束。
>
> 參考：[JDBC 9.4 load balancing and failover](http://blog.163.com/digoal@126/blog/static/16387704020158241250463/)

### VI. 官方 Limitations 原文

> pg_shard is intentionally limited in scope during its first release, but is fully functional within that scope. We classify pg_shard's current limitations into two groups:
>
> **Medium-term (architectural):**
> - Transactional semantics for queries that span across multiple shards
> - Unique constraints on columns other than the partition key, or foreign key constraints
> - Distributed JOINs — If you'd like to run complex analytic queries, please consider upgrading to CitusDB
>
> **Short-term:**
> - Table alterations: customers accomplish them by using a script that propagates such changes to all worker nodes
> - DROP TABLE does not have any special semantics on a distributed table. An upcoming release will add a shard cleanup command
> - Queries such as `INSERT INTO foo SELECT bar, baz FROM qux` are not supported
>
> We keep an open discussion on GitHub issues to hear what you have to say.

> 補充（Senior Dev）：這些限制大部分在 Citus 6.0+ 已解決。分散式 JOIN 在 Citus 中通過 co-located join（相同 distribution column 的表可本地 JOIN）和 broadcast join（小表廣播到所有 worker）實現；分散式 transaction 通過 2PC 實現；subquery 在 Citus 的 adaptive executor 中支援。但 single coordinator 瓶頸在 Citus 中仍然存在——大規模部署需要 Citus MX（每個 worker 也可接受查詢）或 Citus 11+ 的 "query from any node" 特性。

---

## 5. Shard 修復（Repair）

> **初學者導讀**：分散式系統中「機器掛掉」是日常事件，不是意外。pg_shard 的修復流程是理解分散式資料庫高可用性的絕佳入門：
> 1. **自動容錯**：當一個 worker 掛掉，pg_shard 會自動把查詢路由到健康的 replica，使用者查詢不受影響（只會看到警告訊息）。
> 2. **資料不一致**：但如果在故障期間有寫入操作，故障 worker 上的 shard 副本就不會收到這些新資料——形成資料落後。
> 3. **全量修復**：pg_shard 的修復是「全量複製」策略——從健康副本把整個 shard 的資料完整複製到故障副本。這在 shard 很大時非常耗時，且修復期間無法對故障 shard 寫入。現代 Citus 已經改用基於 WAL（預寫日誌）的增量同步來解決這個問題。

```mermaid
flowchart TD
    subgraph FAIL["階段 1: Worker 故障"]
        F1["Worker 1922 掛掉"]
        F2["pg_shard 自動路由\n到健康 replica"]
        F3["查詢正常返回\n但有 WARNING 訊息"]
        F1 --> F2 --> F3
    end

    subgraph WRITE["階段 2: 故障期間寫入"]
        W1["pgbench 持續寫入"]
        W2["1922 上的 shard 副本\n收不到新資料"]
        W3["資料不一致：\n健康副本有新資料\n故障副本缺少新資料"]
        W1 --> W2 --> W3
    end

    subgraph REPAIR["階段 3: 修復流程"]
        R1["重啟故障 Worker (1922)"]
        R2["查詢 shard_state = 3 的 shard\n（需要修復）"]
        R3["從健康副本 (source)\n全量複製到故障副本 (target)"]
        R4["驗證: shard_state 恢復為 1\n資料數量一致"]
        R1 --> R2 --> R3 --> R4
    end

    FAIL --> WRITE --> REPAIR

    subgraph WARN["修復注意事項"]
        N1["⚠️ source/target 不能搞反\n（有狀態檢查保護）"]
        N2["⚠️ 全量複製，shard 大時很慢"]
        N3["⚠️ 修復期間無法寫入故障 shard"]
    end
```

### I. 模擬 Worker 故障

關閉 worker node `localhost:1922`：

```bash
pg_ctl stop -m fast -D /data01/pg_root_1922
```

查詢仍然成功（pg_shard 自動路由到健康 replica），但會輸出一系列連線失敗警告：

```sql
SELECT count(*) FROM customer_reviews;
```

```
WARNING:  Connection failed to localhost:1922
DETAIL:  Remote message: could not connect to server: Connection refused
...
 count
-------
  6296
(1 row)
```

### II. 查看 Shard 狀態

```sql
SELECT * FROM pgs_distribution_metadata.shard_placement;
```

掛掉的 worker 上的所有 shard 狀態變為 `3`（需要修復）：

```
 id | shard_id | shard_state | node_name | node_port
----+----------+-------------+-----------+-----------
  2 |    10000 |           1 | localhost |      1923
  3 |    10001 |           1 | localhost |      1923
 ...
  8 |    10003 |           3 | localhost |      1922   -- 掛了
 32 |    10015 |           3 | localhost |      1922   -- 掛了
  9 |    10004 |           3 | localhost |      1922   -- 掛了
 ...
```

`shard_state = 1` 健康，`shard_state = 3` 需修復。

### III. 在故障期間繼續寫入（造成資料不一致）

```bash
pgbench -M simple -n -r -P 1 -f ./test.sql -c 8 -j 8 -T 1000
```

```
WARNING: Connection failed to localhost:1922 ...
progress: 1.0 s, 167.9 tps
progress: 2.0 s, 1306.0 tps
progress: 3.0 s, 1451.7 tps
...
```

這段時間寫入的資料不會同步到掛掉的 worker 1922 上的 shard 副本，造成資料落後。

### IV. 重啟故障 Worker 並修復

```bash
pg_ctl start -D /data01/pg_root_1922
```

在所有 worker 節點建立 extension（修復函數 `master_copy_shard_placement` 需要在目標端也存在 pg_shard）：

```bash
psql -h 127.0.0.1 -p 1922 -c "CREATE EXTENSION pg_shard;"
psql -h 127.0.0.1 -p 1923 -c "CREATE EXTENSION pg_shard;"
psql -h 127.0.0.1 -p 1924 -c "CREATE EXTENSION pg_shard;"
psql -h 127.0.0.1 -p 1925 -c "CREATE EXTENSION pg_shard;"
```

### V. 找出需修復的 Shard（從健康副本複製到故障副本）

找出 `shard_state = 3` 的 shard 及其健康副本配對：

```sql
SELECT t1.shard_id, t1.node_name, t1.node_port, t2.node_name, t2.node_port
FROM pgs_distribution_metadata.shard_placement t1,
     pgs_distribution_metadata.shard_placement t2
WHERE t1.shard_id = t2.shard_id
  AND (t1.node_name || t1.node_port) <> (t2.node_name || t2.node_port)
  AND t1.shard_state = 3;
```

```
 shard_id | node_name | node_port | node_name | node_port
----------+-----------+-----------+-----------+-----------
    10003 | localhost |      1922 | localhost |      1925
    10004 | localhost |      1922 | localhost |      1923
    10008 | localhost |      1922 | localhost |      1923
    10011 | localhost |      1922 | localhost |      1925
    10012 | localhost |      1922 | localhost |      1923
    10015 | localhost |      1922 | localhost |      1925
(6 rows)
```

### VI. 執行修復

```sql
-- master_copy_shard_placement(
--   shard_id, source_node, source_port, target_node, target_port
-- )
-- 注意：source = 健康副本，target = 故障副本，不能搞反

SELECT master_copy_shard_placement(
    t1.shard_id, t2.node_name, t2.node_port,  -- 從健康副本 t2
    t1.node_name, t1.node_port                 -- 複製到故障副本 t1
)
FROM pgs_distribution_metadata.shard_placement t1,
     pgs_distribution_metadata.shard_placement t2
WHERE t1.shard_id = t2.shard_id
  AND (t1.node_name || t1.node_port) <> (t2.node_name || t2.node_port)
  AND t1.shard_state = 3;
```

若方向搞反會報錯（pg_shard 有保護）：

```
ERROR: 22023: source placement must be in finalized state
```

修復完成後所有 shard 狀態恢復為 `1`：

```sql
SELECT * FROM pgs_distribution_metadata.shard_placement;
-- 全部 32 row, shard_state = 1
```

### VII. 驗證資料一致性

```sql
SELECT count(*) FROM customer_reviews;
-- 17729 (與故障前一致)
```

> 補充（Senior Dev）：pg_shard 的修復機制是基於 `master_copy_shard_placement` 的 full-copy 策略——直接從健康副本把整個 shard 的數據複製到故障節點，沒有 incremental / WAL-based 同步。這在 shard 很大時非常耗時（全量傳輸），且修復期間故障 shard 的寫入會被跳過。現代 Citus 支援基於 logical replication 的 shard rebalancing（`citus_rebalance_start()`）和 streaming replication 的 worker HA，本質上解決了這個問題。

---

## 參考

1. [pg_shard GitHub v1.2.2](https://github.com/citusdata/pg_shard/tree/v1.2.2)
2. [pg_shard Quick Start Guide](https://www.citusdata.com/citus-products/pg-shard/pg-shard-quick-start-guide)
3. [JDBC 9.4 load balancing and failover](http://blog.163.com/digoal@126/blog/static/16387704020158241250463/)
