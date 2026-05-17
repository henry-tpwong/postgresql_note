# PostgreSQL 數據庫開發規範 — PG 17 更新版

> 來源：[digoal - PostgreSQL 数据库开发规范 (2016-09-26)](https://github.com/digoal/blog/blob/master/201609/20160926_01.md)
>
> 2026 更新：標記 4 條已過時規則、補充 PG 14-17 對應改進、新增 Senior Dev 實戰筆記。

---

## 目錄

- [1. 命名規範](#1-命名規範)
- [2. 設計規範](#2-設計規範)
- [3. Query 規範](#3-query-規範)
- [4. 管理規範](#4-管理規範)
- [5. 穩定性與效能規範](#5-穩定性與效能規範)
- [6. 阿里雲 RDS 規範](#6-阿里雲-rds-規範)
- [7. PG 14-17 過時規則追蹤](#7-pg-14-17-過時規則追蹤)

---

## 1. 命名規範

### 【強制】

| # | 規則 |
|---|------|
| 1 | DB / table / column name 總長度 ≤ 63 字符 |
| 2 | Object name 只用小寫字母、底線、數字。不以 `pg` 開頭、不以數字開頭、不用保留字（[PG 17 reserved words](https://www.postgresql.org/docs/17/sql-keywords-appendix.html)） |
| 3 | Query alias 只用 `[a-z0-9_]`，不用中文或其他字符 |

### 【推薦】

| # | 規則 |
|---|------|
| 4 | PK index → `pk_*`，UK index → `uk_*`，普通 index → `idx_*` |
| 5 | Temp table → `tmp_*`；分區子表以規則結尾（`tbl_2016`、`tbl_2017`...） |
| 6 | DB name：`<部門>_<功能>`（如 `xxx_yyy`） |
| 7 | DB name 與應用名稱一致或便於辨識 |
| 8 | 避免 `public` schema；每應用分配獨立 schema，`schema_name` 與 username 一致 |
| 9 | Comment 不用中文（編碼不一致時 `pg_dump` 可能亂碼） |

---

## 2. 設計規範

### 【強制】

| # | 規則 | SQL 範例 |
|---|------|---------|
| 1 | 跨表同名/同義 column 必須名稱、型別一致 | — |
| 2 | B-tree index 欄位 ≤ 2000 bytes；超過用 hash / 分詞 index | — |
| 3 | FK column 必須手動建 index（否則影響 referenced table 的 UPDATE/DELETE） | 見下方 #1 |
| 4 | FK 必須設 action：`ON DELETE/UPDATE CASCADE \| SET NULL \| SET DEFAULT` | 見下方 #2 |
| 5 | 頻繁更新的 table 設 `fillfactor = 85`（預留 15% 給 HOT update） | 見下方 #3 |
| 6 | Index `NULLS FIRST/LAST` 必須與 `ORDER BY` 一致，否則 index 可能無法使用 | — |
| 7 | Column data type 必須與 application 定義一致；跨表 collation 必須一致 | — |

```sql
-- #3 FK 手動建 index
CREATE TABLE tbl(id int PRIMARY KEY, info text);
CREATE TABLE tbl1(id int REFERENCES tbl(id), info text);
-- 此時 tbl1.id 沒有 index！
CREATE INDEX idx_tbl1_id ON tbl1(id);

-- #4 FK action
CREATE TABLE tbl2(id int REFERENCES tbl(id) ON DELETE CASCADE ON UPDATE CASCADE, info text);
CREATE INDEX idx_tbl2_id ON tbl2(id);
INSERT INTO tbl VALUES (1,'test');
INSERT INTO tbl2 VALUES (1,'test');
UPDATE tbl SET id = 2;
SELECT * FROM tbl2;  -- id 自動變成 2

-- #5 fillfactor
CREATE TABLE test123(id int, info text) WITH (fillfactor = 85);
```

### 【推薦】

| # | 規則 |
|---|------|
| 8 | Partition table 全域唯一 PK：多個 sequence，不同 `INCREMENT BY` 或不同 `MINVALUE/MAXVALUE` 範圍 |
| 9 | 定期刪除歷史數據 → 按時間分區，`DROP`/`TRUNCATE` partition，不用 `DELETE` |
| 10 | 所有字串用 UTF-8；區分 `length()`（字元數）vs `octet_length()`（byte 數） |
| 11 | 線性相關數據（streaming、自增、timestamp）+ 範圍查詢 → BRIN index |
| 12 | 選擇合適 data type：數字不用字串、樹型不用字串（PG 支援 numeric / float / money / string / date / time / boolean / enum / geometry / network / UUID / XML / JSON / array / composite / range / ltree / cube / earth / hstore / pg_trgm / PostGIS...） |
| 13 | 避免全表掃描；PG 幾乎所有 data type 都支援 index（btree / hash / gin / gist / sp-gist / brin / rum / bloom） |
| 14 | 複雜 business logic + 高 RT 場景：盡量用 stored procedure（plpgsql）或內建函數，減少 DB-app roundtrip |
| 15 | 避免用 trigger（複雜化邏輯，不利 debug） |
| 16 | 常訪問大結果集（如 100 row）→ 設法聚合為少量 row；無法聚合則用高 IOPS 磁盤 |
| 17 | Streaming 即時統計防 parallel transaction 空洞：parallel insert 到不同分表，單分表內 serial insert |
| 18 | 範圍查詢 → 用 range type + GIST index（如 IP range `int8range` + GIST，比 `BETWEEN` 快 20x） |
| 19 | 未引用的大對象必須同時刪除數據部分（或用 `vacuumlo`） |
| 20 | 固定條件的查詢 → partial index（`CREATE INDEX ... WHERE id = 1`） |
| 21 | Expression 查詢 → expression/function index |
| 22 | Debug 複雜邏輯 → plpgsql `DO` block，非 function |
| 23 | 中文分詞 → `zhparser` 或 `jieba`；分詞欄位用 GIN index |
| 24 | Regex / text similarity → `pg_trgm` GIN index（覆蓋 `LIKE '%xxx%'`） |
| 25 | ~~PG ≤ 9.4：GIN 高併發寫入設 `fillfactor=70`~~ ⛔ **PG 10+ 已優化 GIN lock，此條過時** |
| 26 | Prefix / suffix 模糊查詢：`col ~ '^abc'`（prefix）或 `reverse(col) ~ '^def'`（suffix） |
| 27 | 大表（> 8GB 或 > 1000 萬 row）應分區，提升查詢/更新/備份/恢復/index 效率 |
| 28 | ~~分區數不宜過多（PG < 10 的 optimizer 開銷）~~ ⛔ **PG 10+ declarative partitioning + PG 11-17 pruning 優化，此限制大幅放寬** |
| 29 | Table schema 謹慎規劃，避免頻繁 `ALTER TABLE ADD COLUMN`（可能觸發 rewrite）；不可預測 schema 用 `jsonb` |

```sql
-- #8 方案一：不同步長
CREATE SEQUENCE seq_tab1 INCREMENT BY 10000 START WITH 1;
CREATE SEQUENCE seq_tab2 INCREMENT BY 10000 START WITH 2;
CREATE SEQUENCE seq_tab3 INCREMENT BY 10000 START WITH 3;
CREATE TABLE tab1 (id int PRIMARY KEY DEFAULT nextval('seq_tab1') CHECK (mod(id,10000)=1), info text);
CREATE TABLE tab2 (id int PRIMARY KEY DEFAULT nextval('seq_tab2') CHECK (mod(id,10000)=2), info text);
CREATE TABLE tab3 (id int PRIMARY KEY DEFAULT nextval('seq_tab3') CHECK (mod(id,10000)=3), info text);

-- #8 方案二：不同範圍
CREATE SEQUENCE seq_tb1 INCREMENT BY 1 MINVALUE 1 MAXVALUE 100000000 START WITH 1 NO CYCLE;
CREATE SEQUENCE seq_tb2 INCREMENT BY 1 MINVALUE 100000001 MAXVALUE 200000000 START WITH 100000001 NO CYCLE;

-- #10 UTF-8 長度
SELECT length('阿里巴巴');       -- 4（字元數）
SELECT octet_length('阿里巴巴'); -- 12（byte 數）

-- Available length functions:
-- array_length, bit_length, char_length/character_length,
-- json_array_length, jsonb_array_length, length, octet_length

-- #11 BRIN index
CREATE INDEX idx ON tbl USING brin (id);

-- #18 Range type + GIST IP 範例
CREATE TABLE ip_address_pool_3 (
    id serial8 PRIMARY KEY,
    start_ip inet NOT NULL,
    end_ip inet NOT NULL,
    province varchar(128) NOT NULL,
    city varchar(128) NOT NULL,
    region_name varchar(128) NOT NULL,
    company_name varchar(128) NOT NULL,
    ip_decimal_segment int8range
);
CREATE INDEX ip_address_pool_3_range ON ip_address_pool_3 USING gist (ip_decimal_segment);
SELECT province, ip_decimal_segment FROM ip_address_pool_3 WHERE ip_decimal_segment @> :ip::int8;

-- #20 Partial index
CREATE INDEX idx ON tbl (col) WHERE id = 1;

-- #21 Expression index
CREATE INDEX idx ON tbl (exp);

-- #22 DO block for debugging
DO LANGUAGE plpgsql $$
DECLARE
BEGIN
    -- logical code
END;
$$;

-- #26 Prefix / suffix fuzzy query
SELECT * FROM tbl WHERE col ~ '^abc';          -- prefix
SELECT * FROM tbl WHERE reverse(col) ~ '^def'; -- suffix
```

> 補充（Senior Dev）：#8 的 multi-sequence 方案在 PG 10+ 有了更好的替代——`IDENTITY` column + `GENERATED BY DEFAULT AS IDENTITY`，搭配 `CACHE` 參數可減少 sequence 競爭。若用 PG 17 native partitioning + `GENERATED ALWAYS AS IDENTITY`，sequence 管理由 PG 內部處理，不需手動調步長。

---

## 3. Query 規範

### 【強制】

| # | 規則 | SQL 範例 |
|---|------|---------|
| 1 | 永遠用 `count(*)`（SQL92 標準，統計所有 row 含 NULL）；不用 `count(col)` 或 `count(常數)` | — |
| 2 | 多 column count 必須加括號：`count((col1,col2,col3))`。即使所有 column 為 NULL 該 row 仍被計數 | — |
| 3 | `count(distinct col)` 只計非 NULL 不重複值，NULL 不計 | — |
| 4 | `count(distinct (col1,col2,...))` 計數 NULL，且 NULL = NULL 視為相同 | — |
| 5 | `count(col)` 對 NULL column 返回 0，而 `sum(col)` 返回 NULL → NPE guard：`COALESCE(SUM(g), 0)` | — |
| 6 | NULL = UNKNOWN；任何與 NULL 的比較返回 NULL（非 true/false）：`NULL <> NULL` → NULL，`NULL = NULL` → NULL | — |
| 7 | 除非 ETL，否則避免返回大數據量給 client | — |
| 8 | 永遠不用 `SELECT *`，必須指定 column list | — |

```sql
-- #2~5 count behavior
CREATE TABLE t123(c1 int, c2 int, c3 int);
INSERT INTO t123 VALUES
    (null,null,null), (null,null,null),
    (1,null,null), (2,null,null),
    (null,1,null), (null,2,null);

SELECT count((c1,c2)) FROM t123;   -- 6（全部 row）
SELECT count((c1)) FROM t123;      -- 2（只計 c1 非 NULL）
SELECT count(distinct c1) FROM t123;  -- 2
SELECT count(distinct (c1,c2)) FROM t123;  -- 5（NULL 被計入且 NULL=NULL）
SELECT count(distinct (c1,c2,c3)) FROM t123;  -- 5
SELECT count(c1), sum(c1) FROM t123 WHERE c1 IS NULL;  -- 0 | NULL
SELECT COALESCE(SUM(g), 0) FROM table;  -- NPE guard
```

---

## 4. 管理規範

### 【強制】

| # | 規則 | SQL |
|---|------|-----|
| 1 | 數據訂正：先 `SELECT` 確認，再 `DELETE`/`UPDATE` | — |
| 2 | DDL 及重鎖操作（`VACUUM FULL`、`CREATE INDEX`...）必須設 `lock_timeout` | 見下方 |
| 3 | 涉及數據變更的 `EXPLAIN ANALYZE` 必須在 transaction 中執行後 rollback | 見下方 |
| 4 | `CREATE INDEX CONCURRENTLY` 避免阻塞 DML | 見下方 |
| 5 | 複雜密碼：小寫字母 + 數字 + 底線；字母開頭，字母/數字結尾 | — |
| 6 | 業務/開發/測試帳號不得用 superuser | — |
| 7 | `archive_mode=on` 必須設 `archive_command`，監控 `pg_wal` 空間防止歸檔失敗堆積 | — |
| 8 | Standby + replication slot：必須監控備機延遲和 slot 狀態，防止 WAL 堆積 | — |

```sql
-- #2 lock_timeout
BEGIN;
SET LOCAL lock_timeout = '10s';
-- DDL query;
END;

-- #3 EXPLAIN in transaction
BEGIN;
EXPLAIN ANALYZE query;
ROLLBACK;

-- #4 CONCURRENTLY
CREATE INDEX CONCURRENTLY idx ON tbl(id);

-- #13 maintenance_work_mem
BEGIN;
SET LOCAL maintenance_work_mem = '2GB';
CREATE INDEX idx ON tbl(id);
END;
```

### 【推薦】

| # | 規則 |
|---|------|
| 9 | 多業務共用 PG cluster → 每業務一個 DB；若需跨業務 data 互動，在 application 層處理 |
| 10 | 每業務分配獨立 DB account，禁止共用 |
| 11 | Failover 後，新 primary 開放前用 `pg_prewarm` 預熱 shared buffer |
| 12 | 快速數據載入：關閉 autovacuum → 刪除 index → 載入 → analyze + 重建 index |
| 13 | 加速建 index：加大 `maintenance_work_mem`（考慮可用記憶體） |
| 14 | 防止長連接佔用過多 relcache/syscache：定期重連（如每小時）。RDS PG 提供 in-session cache release API |
| 15 | 大批量入庫用 `COPY` 或 `INSERT INTO ... VALUES (),(),...()` |
| 16 | 大批量 insert/update/delete 後，若 autovacuum 關閉，手動 `VACUUM VERBOSE ANALYZE` |
| 17 | 大批量 delete/update 分多個 transaction；操作完檢查 bloat rate，必要時用 `pg_repack` |

> 補充（Senior Dev）：#12 的「關閉 autovacuum + 刪除 index 再重建」是 ETL 經典手法。PG 12+ 的 `REINDEX CONCURRENTLY` 可在不鎖表情況下重建 index。對於超大表，建議用 `pg_repack`（社群工具）在線重組，避免 `VACUUM FULL` 的獨佔鎖。

---

## 5. 穩定性與效能規範

### 【強制】

| # | 規則 | SQL |
|---|------|-----|
| 1 | 分頁邏輯：count = 0 時立即 return，不執行後續分頁 | — |
| 2 | Cursor 使用後立即關閉 | — |
| 3 | 2PC transaction 必須及時 commit/rollback，否則導致 DB bloat | — |
| 4 | 不用 `DELETE` 全表 → 用 `TRUNCATE`（DDL，設 `lock_timeout`） | — |
| 5 | Application 必須開啟 autocommit；避免框架 auto-BEGIN 空 transaction | — |
| 6 | 高併發必須用 prepared statement（bind variable），防止硬解析 CPU overhead | — |
| 7 | ~~不用 hash index（不寫 WAL，crash 不可恢復）~~ ⛔ **PG 10+ hash index 已寫 WAL，此限制解除** | — |
| 8 | 秒殺場景：必須用 `pg_try_advisory_xact_lock(id)` 鎖定後再更新；拿不到鎖重試 | 見下方 |
| 9 | 不要用 `count(*)` 判斷是否存在 → 用 `SELECT 1 ... LIMIT 1` | 見下方 |
| 10 | 高併發必須用 connection pool（程式層或 middleware：pgbouncer / pgpool-II） | — |
| 11 | 必須有重連機制；長時間空閒連接前用 `SELECT 1` 探測 | — |
| 12 | KNN 近鄰查詢必須建 GIST / SP-GIST index | 見下方 |
| 13 | 避免頻繁 create/drop temp table（造成 system catalog 碎片） | — |
| 14 | 只用最低需求的 isolation level；READ COMMITTED 足夠時不用 REPEATABLE READ / SERIALIZABLE | — |

```sql
-- #8 Advisory lock for flash sale
CREATE OR REPLACE FUNCTION public.f(i_id integer)
RETURNS void
LANGUAGE plpgsql
AS $function$
DECLARE
    a_lock boolean := false;
BEGIN
    SELECT pg_try_advisory_xact_lock(i_id) INTO a_lock;
    IF a_lock THEN
        UPDATE t1 SET count = count - 1 WHERE id = i_id;
    END IF;
    EXCEPTION WHEN others THEN
        RETURN;
END;
$function$;

SELECT f(id) FROM tbl WHERE id = ? AND count > 0;

-- #9 LIMIT 1 instead of count(*)
SELECT 1 FROM tbl WHERE xxx LIMIT 1;
-- if found → 存在
-- else     → 不存在

-- #12 KNN with GIST
CREATE INDEX idx ON tbl USING gist(col);
SELECT * FROM tbl ORDER BY col <-> '(0,100)';
```

### 【推薦】

| # | 規則 |
|---|------|
| 15 | 高峰期避免 `ALTER TABLE ADD COLUMN ... DEFAULT`（觸發 rewrite）；先加 column 不加 default，application 層後處理 |
| 16 | 空間查詢（contains, intersects）：用大面積多邊形提效；無法時 split polygon + UNION ALL |
| 17 | 不需精確分頁數 → 用 EXPLAIN-based 快速估算 `countit()` function |
| 18 | 分頁優化 → 用 cursor（`DECLARE ... CURSOR` + `FETCH`）；後翻用 `SCROLL` cursor |
| 19 | 可預估執行時間的 query → 設 `statement_timeout`，防止雪崩和長時間持鎖 |
| 20 | `TRUNCATE` 是 DDL，鎖粒度大 → 開發程式碼中不用，除非加 `lock_timeout` |
| 21 | 將 DDL 封裝在 transaction 中以便 rollback；但必須短 transaction |
| 22 | 用 `INSERT/UPDATE/DELETE ... RETURNING` 減少 DB roundtrip |
| 23 | 自增欄位用 sequence（`serial2/4/8`）；禁用 trigger 產生序列值 |
| 24 | 多 column 任意匹配查詢 → row-level full-text index（`to_tsvector(row::text)`）或 row-to-array index |
| 25 | 中文分詞必須設 token mapping（`ALTER TEXT SEARCH CONFIGURATION ... ADD MAPPING FOR a,b,c,... WITH simple`）；配置 zhparser 參數 |
| 26 | Tree query → recursive CTE；確保終止條件；避免 tree data 重複 |
| 27 | 避免長 transaction（造成 garbage bloat） |
| 28 | 多維分析 → `GROUPING SETS`、`CUBE`、`ROLLUP`（減少 data rescan） |
| 29 | 快速近似 UV（`count distinct`）→ HLL extension |
| 30 | Upsert → `INSERT ... ON CONFLICT DO NOTHING / DO UPDATE SET ...` |
| 31 | 常訪問的大表子集/OLAP → `MATERIALIZED VIEW`；支援 `REFRESH MATERIALIZED VIEW CONCURRENTLY` |
| 32 | 避免頻繁 update 寬表（MVCC write amplification）；將常更新 column 拆分到獨立表，PK join |
| 33 | 用 window function 減少 app-DB roundtrip |
| 34 | Application 層防止 deadlock：同一 user 的數據在單一 thread/session 處理 |
| 35 | OLTP：避免頻繁 aggregation → 定期聚合 + caching |
| 36 | 數據去重：row-level → `DELETE WHERE ctid NOT IN (SELECT min(ctid) FROM tbl GROUP BY tbl::text)`；有 PK → `row_number() OVER (PARTITION BY col ORDER BY pk)` |
| 37 | 快速隨機取記錄：方法一 → random id sampling from `generate_series`；方法二 → plpgsql function |
| 38 | Online schema change（`ADD COLUMN`, `CREATE INDEX`）在離峰時段進行 |
| 39 | OLTP：高峰期禁止長 SQL、大事務、大批量操作 |
| 40 | Query 條件與 index 類型匹配（expression → expression index；column → column index） |
| 41 | 比較值含 NULL 時用 `IS DISTINCT FROM` / `IS NOT DISTINCT FROM`（NULL 視為相等） |
| 42 | UDF/DO block 中逐行處理用 cursor + `WHERE CURRENT OF` |
| 43 | 避免 `!=` / `<>` 在 WHERE（可能無法用 index）；解法：partial index、partition、`constraint_exclusion=on` |
| 44 | 頻繁變動的表：降低 `autovacuum_vacuum_scale_factor`（如 0.005）、`autovacuum_analyze_scale_factor`（如 0.01）、設 `autovacuum_vacuum_cost_delay=0` |
| 45 | `WHERE col IN (...)` / `WHERE col=a OR col=b` → PG 自動用 bitmap OR，不需手動改 UNION |
| 46 | 優先使用 `EXISTS (SELECT 1 ...)` 而非 `IN (SELECT ...)` |
| 47 | 優先使用 array variable 而非 temp table（除非數據量極大） |
| 48 | 避免全表掃描：`WHERE` 和 `ORDER BY` 欄位建 index；用 `EXPLAIN (verbose,costs,timing,buffers,analyze)` 驗證 |
| 49 | 控制 JOIN order：`join_collapse_limit=1` + `geqo=off` 固定 explicit JOIN order；或讓 optimizer 找最佳順序再改寫 SQL |
| 50 | 控制 subquery flattening：`from_collapse_limit=1` + `geqo=off` 防止 subquery 提升為 JOIN |
| 51 | GIN write 優化：`fastupdate=on`，設適當 `gin_pending_list_limit` (KB) |
| 52 | B-tree：避免 UUID 等高離散值作 index key（過多 page split）；調整 `fillfactor` |
| 53 | BRIN：依數據相關性設 `pages_per_range`；用 `ANALYZE tbl; SELECT reltuples/relpages` 評估 |

### 精選 SQL 範例

```sql
-- #17 快速估算分頁數
CREATE OR REPLACE FUNCTION countit(text)
RETURNS float4
LANGUAGE plpgsql AS $$
DECLARE v_plan json;
BEGIN
    EXECUTE 'EXPLAIN (FORMAT JSON) ' || $1 INTO v_plan;
    RETURN v_plan #>> '{0,Plan,"Plan Rows"}';
END;
$$;
SELECT countit('SELECT * FROM t1234 WHERE id < 1000');  -- 954

-- #18 Cursor pagination
DECLARE cur1 CURSOR FOR SELECT * FROM sbtest1 WHERE id BETWEEN 100 AND 1000000 ORDER BY id;
FETCH 100 FROM cur1;
DECLARE cur1 SCROLL CURSOR FOR ...;  -- 支援前後翻頁

-- #19 statement_timeout
BEGIN;
SET LOCAL statement_timeout = '10s';
-- query;
END;

-- #22 RETURNING
CREATE TABLE tbl4(id serial, info text);
INSERT INTO tbl4 (info) VALUES ('test') RETURNING *;
UPDATE tbl4 SET info = 'abc' RETURNING *;
DELETE FROM tbl4 RETURNING *;

-- #24 Row-level full-text index
CREATE OR REPLACE FUNCTION f1(text) RETURNS tsvector AS $$
    SELECT to_tsvector($1);
$$ LANGUAGE sql IMMUTABLE STRICT;
ALTER FUNCTION record_out(record) IMMUTABLE;
ALTER FUNCTION textin(cstring) IMMUTABLE;
CREATE INDEX idx_t_1 ON t USING gin (f1('jiebacfg'::regconfig, t::text));
SELECT * FROM t WHERE f1('jiebacfg'::regconfig, t::text) @@ to_tsquery('digoal & china');

-- #25 Chinese token mapping
ALTER TEXT SEARCH CONFIGURATION testzhcfg
    ADD MAPPING FOR a,b,c,d,e,f,g,h,i,j,k,l,m,n,o,p,q,r,s,t,u,v,w,x,y,z WITH simple;

-- zhparser options:
-- zhparser.punctuation_ignore = f
-- zhparser.seg_with_duality = f
-- zhparser.dict_in_memory = f
-- zhparser.multi_short = f
-- zhparser.multi_duality = f
-- zhparser.multi_zmain = f
-- zhparser.multi_zall = f

-- #26 Recursive CTE (tree)
CREATE TABLE tbl_test (ID numeric, NAME text, PID numeric DEFAULT 0);
INSERT INTO tbl_test VALUES ('1','10','0'),('2','11','1'),('3','20','0'),('4','12','1'),('5','121','2');

-- Root → leaf:
WITH RECURSIVE t_result AS (
    SELECT * FROM tbl_test WHERE id = 1
    UNION ALL
    SELECT t2.* FROM t_result t1 JOIN tbl_test t2 ON t1.id = t2.pid
) SELECT * FROM t_result;

-- Leaf → root:
WITH RECURSIVE t_result AS (
    SELECT * FROM tbl_test WHERE id = 5
    UNION ALL
    SELECT t2.* FROM t_result t1 JOIN tbl_test t2 ON t1.pid = t2.id
) SELECT * FROM t_result;

-- #28 Multi-dimensional analysis with CUBE + GROUPING SETS
CREATE TABLE tab5(c1 int, c2 int, c3 int, c4 int, crt_time timestamp);
INSERT INTO tab5 SELECT
    trunc(100*random()), trunc(1000*random()),
    trunc(10000*random()), trunc(100000*random()),
    clock_timestamp() + (trunc(10000*random())||' hour')::interval
FROM generate_series(1, 1000000);

CREATE TABLE stat_tab5 (
    c1 int, c2 int, c3 int, c4 int,
    time1 text, time2 text, time3 text, time4 text,
    cnt int8, bitmap text
);

INSERT INTO stat_tab5
SELECT c1,c2,c3,c4, t1,t2,t3,t4, cnt,
    '' ||
    CASE WHEN c1 IS NULL THEN 0 ELSE 1 END ||
    CASE WHEN c2 IS NULL THEN 0 ELSE 1 END ||
    CASE WHEN c3 IS NULL THEN 0 ELSE 1 END ||
    CASE WHEN c4 IS NULL THEN 0 ELSE 1 END ||
    CASE WHEN t1 IS NULL THEN 0 ELSE 1 END ||
    CASE WHEN t2 IS NULL THEN 0 ELSE 1 END ||
    CASE WHEN t3 IS NULL THEN 0 ELSE 1 END ||
    CASE WHEN t4 IS NULL THEN 0 ELSE 1 END
FROM (
    SELECT c1,c2,c3,c4,
        to_char(crt_time, 'yyyy') t1,
        to_char(crt_time, 'yyyy-mm') t2,
        to_char(crt_time, 'yyyy-mm-dd') t3,
        to_char(crt_time, 'yyyy-mm-dd hh24') t4,
        count(*) cnt
    FROM tab5
    GROUP BY CUBE(c1,c2,c3,c4),
        GROUPING SETS(
            to_char(crt_time, 'yyyy'),
            to_char(crt_time, 'yyyy-mm'),
            to_char(crt_time, 'yyyy-mm-dd'),
            to_char(crt_time, 'yyyy-mm-dd hh24')
        )
) t;
-- INSERT 0 49570486, Time: 172374 ms

CREATE INDEX idx_stat_tab5_bitmap ON stat_tab5 (bitmap);

-- 查詢維度 c1,c3,c4,t3 → bitmap='10110010'
SELECT c1,c3,c4,time3,cnt FROM stat_tab5 WHERE bitmap = '10110010' LIMIT 10;
-- Time: 0.528 ms

-- #29 HLL approximate UV
CREATE TABLE access_date (acc_date date UNIQUE, userids hll);
INSERT INTO access_date
    SELECT current_date, hll_add_agg(hll_hash_integer(user_id))
    FROM generate_series(1, 10000) t(user_id);
SELECT *, total_users - COALESCE(lag(total_users,1) OVER (ORDER BY rn), 0) AS new_users
FROM (
    SELECT acc_date, row_number() OVER date AS rn,
        #hll_union_agg(userids) OVER date AS total_users
    FROM access_date
    WINDOW date AS (ORDER BY acc_date ASC ROWS UNBOUNDED PRECEDING)
) t;

-- #30 ON CONFLICT
INSERT INTO tbl VALUES (1, 'info')
    ON CONFLICT ON CONSTRAINT tbl_pkey
    DO UPDATE SET info = excluded.info;

-- #31 MATERIALIZED VIEW
CREATE MATERIALIZED VIEW mv_tbl AS SELECT xx,xx,xx FROM tbl WHERE xxx WITH DATA;
REFRESH MATERIALIZED VIEW CONCURRENTLY mv_tbl WITH DATA;

-- #36 Dedup
-- Row-level:
DELETE FROM tbl WHERE ctid NOT IN (SELECT min(ctid) FROM tbl GROUP BY tbl::text);
-- Column-level with PK:
DELETE FROM tbl WHERE pk IN (
    SELECT pk FROM (
        SELECT pk, row_number() OVER (PARTITION BY col ORDER BY pk) rn FROM tbl
    ) t WHERE t.rn > 1
);

-- #41 IS DISTINCT FROM
SELECT NULL IS DISTINCT FROM NULL;  -- f
SELECT NULL IS DISTINCT FROM 1;     -- t

-- #42 Cursor in DO block
DO LANGUAGE plpgsql $$
DECLARE
    cur refcursor;
    rec record;
BEGIN
    OPEN cur FOR SELECT * FROM tbl WHERE id > 1;
    LOOP
        FETCH cur INTO rec;
        IF found THEN
            RAISE NOTICE '%', rec;
            UPDATE tbl SET info = 'ab' WHERE CURRENT OF cur;
        ELSE
            CLOSE cur;
            EXIT;
        END IF;
    END LOOP;
END;
$$;

-- #44 Autovacuum tuning
CREATE TABLE t21(id int, info text) WITH (
    autovacuum_enabled = on,
    toast.autovacuum_enabled = on,
    autovacuum_vacuum_scale_factor = 0.005,
    toast.autovacuum_vacuum_scale_factor = 0.005,
    autovacuum_analyze_scale_factor = 0.01,
    autovacuum_vacuum_cost_delay = 0,
    toast.autovacuum_vacuum_cost_delay = 0
);

-- #46 EXISTS > IN
SELECT num FROM a WHERE EXISTS (SELECT 1 FROM b WHERE num = a.num);

-- #47 Array variable > temp table (small data)
-- #51 GIN fastupdate
CREATE INDEX idx_1 ON tbl USING gin (tsvector)
    WITH (fastupdate=on, gin_pending_list_limit=10240);

-- #53 BRIN evaluation
ANALYZE tbl;
SELECT reltuples / relpages FROM tbl;
```

### #37 快速隨機取記錄

```sql
-- 方法一：random ID sampling
SELECT * FROM tbl_user
WHERE id IN (
    SELECT floor(random() * (max_id - min_id))::int + min_id
    FROM generate_series(1, 5),
        (SELECT max(id) AS max_id, min(id) AS min_id FROM tbl_user) s1
) LIMIT 5;

-- 方法二：plpgsql function
CREATE OR REPLACE FUNCTION f_get_random(i_range int) RETURNS SETOF record AS $$
DECLARE
    v_result record;
    v_max_id int;
    v_min_id int;
    v_random numeric;
BEGIN
    SELECT random() INTO v_random;
    SELECT max(id), min(id) INTO v_max_id, v_min_id FROM tbl_user;
    FOR v_result IN
        SELECT * FROM tbl_user
        WHERE id BETWEEN (v_min_id+(v_random*(v_max_id-v_min_id))::int)
                  AND (v_min_id+(v_random*(v_max_id-v_min_id))::int + i_range)
    LOOP
        RETURN NEXT v_result;
    END LOOP;
    RETURN;
END;
$$ LANGUAGE plpgsql;
```

### #49-50 JOIN Order & Subquery Control

```sql
-- 固定 explicit JOIN order
BEGIN;
SET LOCAL join_collapse_limit = 1;
SET LOCAL geqo = off;
-- your query with explicit JOIN order
END;

-- 讓 optimizer 找最佳 JOIN order（設高 join_collapse_limit），再改寫 SQL
SET join_collapse_limit = 100;
SET geqo = off;
EXPLAIN SELECT * FROM t2 JOIN t1 USING (id) JOIN t3 USING (id) ...;
-- → 根據 EXPLAIN 輸出改寫 SQL 為最佳 order

-- 固定 subquery 不 flatten（from_collapse_limit = 1）
SET from_collapse_limit = 1;
```

> 補充（Senior Dev）：#49-50 的 `join_collapse_limit` / `from_collapse_limit` 控制技巧是 DBA 排查 plan regression 的核心武器。PG 14+ 的 optimizer 對 multi-JOIN 的處理已大幅改進（特別是 `geqo_threshold` 預設值從 PG 16 起調整），大部分場景不需手動控制。若 production 遇到 join order 不穩，先用 `auto_explain` + `pg_stat_statements` 抓出問題 query，再用 `pg_hint_plan` pin 住最佳 plan，而非靠 `join_collapse_limit`。

---

## 6. 阿里雲 RDS 規範（全【推薦】）

| # | 規則 |
|---|------|
| 1 | DB > 2TB → 冷熱分離：用 OSS_EXT external table plugin 存冷數據 |
| 2 | 高 RT 場景 → SLB 或 PROXY passthrough 模式連接 |
| 3 | RDS region 必須與 application region 一致（禁止跨 region） |
| 4 | 配置多個告警接收人，設適當告警門檻 |
| 5 | 設適當白名單保障訪問安全 |
| 6 | 關閉公網訪問；若必須開，必須設白名單 |
| 7 | 數據傾斜導致 bind-variable plan skew → `pg_hint_plan` extension pin 執行計劃 |

```sql
-- #7 pg_hint_plan
CREATE EXTENSION pg_hint_plan;
ALTER ROLE ALL SET session_preload_libraries = 'pg_hint_plan';

/*+ SeqScan(test) */
EXPLAIN SELECT * FROM test WHERE id = 1;
-- → Seq Scan on test

/*+ BitmapScan(test) */
EXPLAIN SELECT * FROM test WHERE id = 1;
-- → Bitmap Heap Scan on test
--    → Bitmap Index Scan on test_pkey
```

---

## 7. PG 14-17 過時規則追蹤

| 原規則 | 版本限制 | PG 17 現狀 |
|--------|---------|-----------|
| 設計 #25：GIN `fillfactor=70` | PG ≤ 9.4 的 lock 問題 | **過時**。PG 10 起 GIN lock 已優化，PG 14+ 進一步改進 |
| 設計 #28：分區數不宜過多 | PG < 10 的 optimizer 開銷 | **大幅放寬**。PG 10 declarative partitioning + 11-17 pruning 優化 |
| 穩定性 #7：禁用 hash index | PG < 10 不寫 WAL | **過時**。PG 10+ hash index 已寫 WAL，crash-safe |
| 命名 #2：保留字參考 PG 9.5 | 連結過時 | 改用 [PG 17 reserved words](https://www.postgresql.org/docs/17/sql-keywords-appendix.html) |

### PG 14-17 新增相關能力

| 版本 | 與本規範相關的新功能 |
|------|---------------------|
| PG 14 | `CYCLE` / `SEARCH` clause（recursive CTE）；`REINDEX` TABLESPACE 變更 |
| PG 15 | `MERGE` statement；`jsonb` 支援 SQL/JSON path；`LOGIN` event trigger |
| PG 16 | `pg_stat_io`（I/O 監控）；parallel `VACUUM`；`REINDEX` on partitioned tables |
| PG 17 | `GENERATED ALWAYS AS IDENTITY` 改進；`EXPLAIN (MEMORY)`；incremental `CREATE INDEX` |
