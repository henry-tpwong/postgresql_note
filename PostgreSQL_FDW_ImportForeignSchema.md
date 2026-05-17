# PostgreSQL IMPORT FOREIGN SCHEMA — 一鍵創建 Foreign Table

> 來源：[digoal - PostgreSQL 9.5 使用 import foreign schema 语法一键创建外部表 (2015-04-09)](https://github.com/digoal/blog/blob/master/201504/20150409_02.md)

---

## 1. 核心語法與背景

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
