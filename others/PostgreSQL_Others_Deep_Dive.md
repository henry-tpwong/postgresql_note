# PostgreSQL 深度探索：從開發規範到千億級實戰

> 本文按由淺入深的閱讀順序，將六篇 PostgreSQL 深度文章合併為一冊：
>
> **第一章** 從開發規範入手，建立 PostgreSQL 的正確使用習慣；
> **第二章** 進入 Trigger 審計的實作細節，掌握 DML 與 DDL 的變更追蹤；
> **第三章** 學習 JOIN 冗餘膨脹的排查與 Early DISTINCT 優化策略；
> **第四章** 深入 pgcrypto 加密體系，從 Hash、密碼儲存到 PGP 對稱/非對稱加密；
> **第五章** 挑戰千億級數據的模糊查詢，剖析 pg_trgm + GIN 的內部機制；
> **第六章** 以 12306 搶票系統為案例，綜合運用 varbit、SKIP LOCKED、Array、pgrouting 等特性完成高併發架構設計。

---

# 一、PostgreSQL 數據庫開發規範 — PG 17 更新版

> 來源：[digoal - PostgreSQL 数据库开发规范 (2016-09-26)](https://github.com/digoal/blog/blob/master/201609/20160926_01.md)
>
> 2026 更新：標記 4 條已過時規則、補充 PG 14-17 對應改進、新增 Senior Dev 實戰筆記。

---

## 1. 命名規範

### I. 【強制】

| # | 規則 |
|---|------|
| 1 | DB / table / column name 總長度 ≤ 63 字符 |
| 2 | Object name 只用小寫字母、底線、數字。不以 `pg` 開頭、不以數字開頭、不用保留字（[PG 17 reserved words](https://www.postgresql.org/docs/17/sql-keywords-appendix.html)） |
| 3 | Query alias 只用 `[a-z0-9_]`，不用中文或其他字符 |

### II. 【推薦】

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

### I. 【強制】

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

### II. 【推薦】

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

### I. 【強制】

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

### I. 【強制】

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

### II. 【推薦】

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

### I. 【強制】

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

### II. 【推薦】

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

### III. 精選 SQL 範例

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

### IV. #37 快速隨機取記錄

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

### V. #49-50 JOIN Order & Subquery Control

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

### I. PG 14-17 新增相關能力

| 版本 | 與本規範相關的新功能 |
|------|---------------------|
| PG 14 | `CYCLE` / `SEARCH` clause（recursive CTE）；`REINDEX` TABLESPACE 變更 |
| PG 15 | `MERGE` statement；`jsonb` 支援 SQL/JSON path；`LOGIN` event trigger |
| PG 16 | `pg_stat_io`（I/O 監控）；parallel `VACUUM`；`REINDEX` on partitioned tables |
| PG 17 | `GENERATED ALWAYS AS IDENTITY` 改進；`EXPLAIN (MEMORY)`；incremental `CREATE INDEX` |

---

# 二、PostgreSQL Trigger Audit — DML 欄位變更審計 + DDL 結構變更審計

> 來源（德哥 digoal Trigger 審計二部曲）：
> - [DDL 審計 — use event trigger record user who alter table (2014-12-11)](https://github.com/digoal/blog/blob/master/201412/20141211_02.md)
> - [DML 審計 — use trigger audit record which column modified (2014-12-14)](https://github.com/digoal/blog/blob/master/201412/20141214_01.md)

---

## 1. DML 審計：Row Trigger + hstore 記錄欄位級變更

### I. 審計表結構

```sql
CREATE EXTENSION hstore;

CREATE TABLE table_change_rec (
  id serial8 PRIMARY KEY,
  relid oid,
  table_schema name,
  table_name name,
  when_tg text,
  level text,
  op text,
  old_rec hstore,
  new_rec hstore,
  diff_old_rec text,
  diff_new_rec text,
  crt_time timestamp without time zone DEFAULT now(),
  username name,
  client_addr inet,
  client_port int
);
```

> 補充（Senior Dev）：`hstore` 是 PostgreSQL 的 key-value extension，在 PG 9.4 後建議改用 `JSONB`。JSONB 的優勢：支援 nested structure、index（GIN）、更豐富的 operator。若改用 JSONB：
> ```sql
> old_rec JSONB,
> new_rec JSONB,
> ```
> trigger 中的 `hstore(OLD.*)` 改為 `to_jsonb(OLD.*)`。本文保留 hstore 以忠實原文，生產環境可自行切換。

### II. 通用觸發器函數

```sql
CREATE OR REPLACE FUNCTION dml_trace()
RETURNS trigger
LANGUAGE plpgsql
AS $BODY$
DECLARE
  v_new_rec hstore;
  v_old_rec hstore;
  v_diff_new_rec text;
  v_diff_old_rec text;
  v_username text := session_user;
  v_client_addr inet := inet_client_addr();
  v_client_port int := inet_client_port();
BEGIN
  CASE TG_OP
  WHEN 'DELETE' THEN
    v_old_rec := hstore(OLD.*);
    INSERT INTO table_change_rec
      (relid, table_schema, table_name, when_tg, level, op, old_rec,
       username, client_addr, client_port)
    VALUES (tg_relid, tg_table_schema, tg_table_name, tg_when, tg_level,
            tg_op, v_old_rec, v_username, v_client_addr, v_client_port);

  WHEN 'INSERT' THEN
    v_new_rec := hstore(NEW.*);
    INSERT INTO table_change_rec
      (relid, table_schema, table_name, when_tg, level, op, new_rec,
       username, client_addr, client_port)
    VALUES (tg_relid, tg_table_schema, tg_table_name, tg_when, tg_level,
            tg_op, v_new_rec, v_username, v_client_addr, v_client_port);

  WHEN 'UPDATE' THEN
    v_old_rec := hstore(OLD.*);
    v_new_rec := hstore(NEW.*);
    -- 計算哪些欄位發生了變更（diff）
    SELECT array_agg(o)::text, array_agg(n)::text
    INTO v_diff_old_rec, v_diff_new_rec
    FROM (
      SELECT row_number() OVER () AS rn, o
      FROM each(v_old_rec) AS o
    ) AS o
    JOIN (
      SELECT row_number() OVER () AS rn, n
      FROM each(v_new_rec) AS n
    ) AS n
    ON o.rn = n.rn AND o <> n;

    INSERT INTO table_change_rec
      (relid, table_schema, table_name, when_tg, level, op,
       old_rec, new_rec, diff_old_rec, diff_new_rec,
       username, client_addr, client_port)
    VALUES (tg_relid, tg_table_schema, tg_table_name, tg_when, tg_level,
            tg_op, v_old_rec, v_new_rec, v_diff_old_rec, v_diff_new_rec,
            v_username, v_client_addr, v_client_port);
  ELSE
    RETURN null;
  END CASE;
  RETURN null;
END;
$BODY$ STRICT;
```

> 補充（Senior Dev）：diff 計算的核心邏輯分析：
>
> 使用 `each(hstore)` 展開為 `(key, value)` pair，再透過 `row_number()` 模擬 array index 做 position-based JOIN。
>
> **潛在問題**：`each()` 返回的順序是不保證的！如果 `hstore(OLD.*)` 和 `hstore(NEW.*)` 的 key 展開順序不同，就會把 column `c1` 的 OLD 值與 column `c2` 的 NEW 值比對，產生錯誤的 diff。這個寫法只有在表中 column 數量/順序固定時才能「碰巧」正確。
>
> **正確 diff 寫法（使用 key-level JOIN）**：
> ```sql
> SELECT coalesce(jsonb_agg(old_col ORDER BY key)::text, '{}'),
>        coalesce(jsonb_agg(new_col ORDER BY key)::text, '{}')
> INTO v_diff_old_rec, v_diff_new_rec
> FROM (
>   SELECT o.key,
>          jsonb_build_object(o.key, o.value) AS old_col,
>          jsonb_build_object(n.key, n.value) AS new_col
>   FROM jsonb_each(to_jsonb(OLD)) o
>   FULL OUTER JOIN jsonb_each(to_jsonb(NEW)) n ON o.key = n.key
>   WHERE o.value IS DISTINCT FROM n.value
> ) t;
> ```
> 這樣透過 key name JOIN 而非 position，確保 c1 比對 c1、c2 比對 c2，且能正確處理新增/刪除欄位的情況。

### III. 掛載 Trigger 及測試

```sql
CREATE TABLE test (id INT, c1 INT, c2 TEXT, c3 TIMESTAMP);

CREATE TRIGGER tg
  AFTER DELETE OR INSERT OR UPDATE ON test
  FOR EACH ROW EXECUTE PROCEDURE dml_trace();
```

執行一系列 DML：

```sql
TRUNCATE table_change_rec;

INSERT INTO test VALUES (1, 1, 'digoal', now());
INSERT INTO test VALUES (2, 2, 'digoal', now());

-- 全部設為 NULL
UPDATE test SET id = NULL, c1 = NULL, c2 = NULL, c3 = NULL WHERE id = 1;

-- 只改部分欄位
UPDATE test SET c1 = 2, c3 = now() WHERE id = 2;

DELETE FROM test;
```

### IV. 審計結果

**INSERT 記錄**（Rec 1 & 2）：`old_rec` 為空，`new_rec` 含完整 row：

```
id=11 | op=INSERT
new_rec: "c1"=>"1", "c2"=>"digoal", "c3"=>"2014-12-15 03:43:12.135139", "id"=>"1"
diff_old_rec / diff_new_rec: NULL

id=12 | op=INSERT
new_rec: "c1"=>"2", "c2"=>"digoal", "c3"=>"2014-12-15 03:43:14.803063", "id"=>"2"
```

**UPDATE 全部設 NULL**（Rec 3）：所有欄位都記錄為 diff：

```
id=13 | op=UPDATE
old_rec: "c1"=>"1", "c2"=>"digoal", "c3"=>"2014-12-15 03:43:12.135139", "id"=>"1"
new_rec: "c1"=>NULL, "c2"=>NULL, "c3"=>NULL, "id"=>NULL
diff_old_rec: {"(c1,1)","(c2,digoal)","(c3,\"2014-12-15 03:43:12.135139\")","(id,1)"}
diff_new_rec: {"(c1,)","(c2,)","(c3,)","(id,)"}
```

**UPDATE 只改部分欄位**（Rec 4）：只有變更的欄位出現在 diff，未變更的不記錄：

```
id=14 | op=UPDATE
old_rec: "c1"=>"2", "c2"=>"digoal", "c3"=>"2014-12-15 03:43:14.803063", "id"=>"2"
new_rec: "c1"=>"2", "c2"=>"digoal", "c3"=>"2014-12-15 03:43:35.977674", "id"=>"2"
diff_old_rec: {"(c3,\"2014-12-15 03:43:14.803063\")"}         -- 只有 c3 變更
diff_new_rec: {"(c3,\"2014-12-15 03:43:35.977674\")"}
```

**DELETE 記錄**（Rec 5 & 6）：`new_rec` 為空，`old_rec` 含被刪除的 row：

```
id=15 | op=DELETE | old_rec: "c1"=>NULL, "c2"=>NULL, "c3"=>NULL, "id"=>NULL
id=16 | op=DELETE | old_rec: "c1"=>"2", "c2"=>"digoal", "c3"=>"2014-12-15 03:43:35.977674", "id"=>"2"
```

> 補充（Senior Dev）：`AFTER` trigger 的審計完整性：資料在觸發點已經成功寫入（或刪除），審計記錄不會因 constraint violation 而產生「假警報」的 rollback 數據。但 transaction 仍可能被上層邏輯 rollback —— 此時審計記錄會一起 rollback（因為審計 INSERT 在同一 transaction 中）。如果需要保留 rollback 的審計，應使用 `dblink` 或 `LISTEN/NOTIFY` 將審計寫入獨立 transaction。

### V. DML Trigger 效能考量

> 補充（Senior Dev）：
>
> | 考量點 | 影響 | 緩解 |
> |--------|------|------|
> | `hstore(ROW.*)` 轉換 | 每 row 一次 key-value serialization，O(n_columns) | 大表應只審計關鍵欄位而非 `*` |
> | Trigger 內 INSERT | 每個 DML row 一次額外 write（WAL 加倍） | 批量操作時考慮 batch audit（statement-level trigger + transition table，PG 10+） |
> | `each()` + JOIN diff 計算 | 每 row O(n_columns)，高併發下 CPU 壓力 | 生產中改為 JSONB + key-level JOIN（O(n_changed) 而非 O(n_columns²)） |
> | Audit 表膨脹 | 審計表增速 = 業務表寫入量 | Partition audit table by month；定期歸檔或刪除 |
> | Trigger 本身 latency | ~0.1-1ms per row（視 column 數量） | 對 latency-sensitive OLTP，改為 async audit（pg_background / pgaudit） |
>
> **現代方案替代**：
> - `pgaudit` extension（PG 官方）：在 C 層面攔截，效能遠優於 plpgsql trigger，支援 session / object 級別
> - PG 10+ statement-level trigger + transition table：一次性處理批量變更，而非逐 row
> - `pg_recvlogical` + logical decoding：完全 non-invasive 的 WAL-based audit

---

## 2. DDL 審計：Event Trigger + hstore 記錄 ALTER TABLE

### I. 審計表與 Event Trigger

```sql
CREATE EXTENSION hstore;

-- 審計表（DDL 審計用）
CREATE TABLE aud_alter (
    id SERIAL PRIMARY KEY,
    crt_time TIMESTAMP DEFAULT now(),
    ctx hstore
);
```

> 補充（Senior Dev）：`ctx hstore` 把整條 `pg_stat_activity` 的 row 序列化存入單一 column。這種設計的優點是「不需要預先定義審計欄位」，未來 `pg_stat_activity` 新增欄位時自動納入。缺點是無法直接對 key-value 內的欄位建 index（如 `WHERE ctx->'usename' = 'app_user'` 無法走 b-tree）。若需頻繁查詢特定欄位，建議把 `usename`、`query` 等拉出獨立 column 並建 index。

### II. Event Trigger 函數

```sql
CREATE OR REPLACE FUNCTION ef_alter() RETURNS event_trigger AS $$
DECLARE
  rec hstore;
BEGIN
  SELECT hstore(pg_stat_activity.*)
  INTO rec
  FROM pg_stat_activity
  WHERE pid = pg_backend_pid();

  INSERT INTO aud_alter (ctx) VALUES (rec);
END;
$$ LANGUAGE plpgsql STRICT;

-- 掛載 event trigger：僅在 ALTER TABLE 完成後觸發
CREATE EVENT TRIGGER e_alter
  ON ddl_command_end
  WHEN TAG IN ('ALTER TABLE')
  EXECUTE PROCEDURE ef_alter();
```

> 補充（Senior Dev）：
>
> **Trigger 時機選擇**：
> - `ddl_command_start`：DDL 執行前觸發。若審計寫在 start 但 DDL 失敗 rollback，審計記錄仍會留在審計表（trigger 內 INSERT 不會被 rollback 因為在 subtransaction 外...實際上 event trigger 在 start 時執行，DDL 失敗會回滾整個 transaction 包括審計記錄）
> - `ddl_command_end`：DDL 執行後觸發。本文的選擇，確保審計記錄的 DDL 已成功執行。
> - `sql_drop`：物件被刪除時觸發（PG 9.5+）
> - `table_rewrite`：ALTER TABLE 觸發 full table rewrite 時觸發（PG 10+）
>
> **Event Trigger 支援的 TAG（PG 9.3+）**：`ALTER TABLE`、`CREATE TABLE`、`DROP TABLE` 等。完整列表見 `pg_event_trigger_ddl_commands()` 的 `command_tag` 欄位。

### III. 測試

```sql
CREATE TABLE test (id INT);

ALTER TABLE test ALTER COLUMN id TYPE INT8;
```

### IV. 審計結果

```sql
SELECT * FROM aud_alter;
```

```
id=1 | crt_time=2014-12-12 05:43:42.840327
ctx:
"pid"=>"48406"
"datid"=>"12949"
"query"=>"alter table test alter column id type int8;"
"state"=>"active"
"datname"=>"postgres"
"usename"=>"postgres"
"waiting"=>"f"
"usesysid"=>"10"
"xact_start"=>"2014-12-12 05:43:42.840327+08"
"client_addr"=>NULL
"client_port"=>"-1"
"query_start"=>"2014-12-12 05:43:42.840327+08"
"state_change"=>"2014-12-12 05:43:42.840331+08"
"backend_start"=>"2014-12-12 05:38:37.084733+08"
"client_hostname"=>NULL
"application_name"=>"psql"
```

展開審計內容（每個 key-value pair）：

```sql
SELECT each(ctx) FROM aud_alter WHERE id = 1;
```

```
(pid,48406)
(datid,12949)
(query,"alter table test alter column id type int8;")  -- 完整 DDL SQL
(state,active)
(datname,postgres)
(usename,postgres)
(waiting,f)
(usesysid,10)
(xact_start,"2014-12-12 05:43:42.840327+08")
(client_addr,)
(client_port,-1)
(query_start,"2014-12-12 05:43:42.840327+08")
(state_change,"2014-12-12 05:43:42.840331+08")
(backend_start,"2014-12-12 05:38:37.084733+08")
(client_hostname,)
(application_name,psql)
```

審計記錄包含：執行者（`usename`）、來源 IP/Port（`client_addr`、`client_port`）、連線資料庫（`datname`）、完整 DDL SQL（`query`）、執行時間。

### V. 原文注意事項

如果使用者在 function 內部封裝 DDL，則 `pg_stat_activity.query` 記錄的是 function call 而非內部 DDL SQL。

> 德哥建議：更嚴格的做法是從 parsetree 中取出被修改的欄位和類型。Event Trigger 可透過 `TG_EVENT`、`TG_TAG` 變數及 `pg_event_trigger_ddl_commands()` 函數獲取結構化 DDL 資訊：
> ```sql
> -- EventTriggerData 結構（src/include/commands/event_trigger.h）
> typedef struct EventTriggerData {
>     NodeTag     type;
>     const char *event;      /* event name */
>     Node       *parsetree;  /* parse tree */
>     const char *tag;        /* command tag */
> } EventTriggerData;
> ```
>
> 在 PL/pgSQL event trigger 中可用 `CREATE EVENT TRIGGER ... WHEN TAG IN ('ALTER TABLE')` 搭配 `pg_event_trigger_ddl_commands()` 獲取 object identity、schema、object type 等結構化欄位。

> 補充（Senior Dev）：
>
> **DDL Audit 方案對比**：
>
> | 方案 | 優點 | 缺點 | 適用場景 |
> |------|------|------|----------|
> | Event Trigger (本文) | 精確攔截特定 TAG、可記錄完整 context、DB 層級 | 寫在 trigger function 內，有小小效能開銷；封裝在 function 內的 DDL 不可見內部 SQL | 精細化審計、需記錄 IP/User 等 context |
> | `log_statement = 'ddl'` | 零效能、零開發、自動記錄 | 寫在 PostgreSQL log file，format 不結構化、不容易檢索；無法過濾 TAG | 基本合規需求 |
> | `pgaudit` extension | C 層面極低開銷、session/object 級控制、結構化輸出 | 需編譯安裝 extension（部分雲端 RDS 不支援）；PG 14+ `pgaudit` 為 contrib | Production 多數場景 |
> | `pg_stat_statements` | 自動聚合、內建 | 不記錄單次執行細節、不記錄執行者 | 輔助診斷 |
>
> **Event Trigger 的版本演進**：
> - PG 9.3：`ddl_command_start` / `ddl_command_end` + `TAG IN (...)` 過濾
> - PG 9.5：新增 `sql_drop` event
> - PG 10：新增 `table_rewrite` event
> - PG 13+：可從 `pg_event_trigger_ddl_commands()` 獲得的資訊更豐富（`object_identity`、`in_extension` 等）
>
> **Production 建議**：
> - 基本合規：`log_statement = 'ddl'` + log 集中收集（如 ELK / Loki）
> - 精細化 Audit：`pgaudit`（物件級 + session 級，效能最優）
> - 客製化需求：Event Trigger + 寫入專用 audit schema（本文方案），適合在某些 RDS 無法安裝 extension 時作為替代
> - 不論哪種方案，DDL audit 記錄需設定保留期並定期歸檔（DDL 變更少，audit 表通常不會過大）

---

## 參考

1. [Event Triggers (official doc)](http://www.postgresql.org/docs/9.3/static/event-triggers.html)
2. [CREATE EVENT TRIGGER (official doc)](http://www.postgresql.org/docs/9.3/static/sql-createeventtrigger.html)
3. [Event Trigger Interface (C-level)](http://www.postgresql.org/docs/9.3/static/event-trigger-interface.html)
4. [PL/pgSQL Trigger Functions](http://www.postgresql.org/docs/9.3/static/plpgsql-trigger.html)
5. [hstore extension](http://www.postgresql.org/docs/9.3/static/hstore.html)
6. [德哥：USE hstore store table's trace record](https://github.com/digoal/blog/blob/master/201206/20120625_01.md)

---

# 三、PostgreSQL JOIN 冗餘膨脹導致慢查詢優化 — Early DISTINCT 策略

> 來源：[digoal - 冗餘數據 JOIN 導致的慢 SQL 優化一例 (2016-08-17)](https://github.com/digoal/blog/blob/master/201608/20160817_03.md)

---

## 1. 問題：每表僅數千 row，4 表 JOIN 卻數十秒無結果

```sql
SELECT DISTINCT abc.pro_col1, abc.col3
FROM t0 p
INNER JOIN t1 abc ON p.id = abc.par_col2
INNER JOIN t2 s   ON s.col3 = abc.col3
INNER JOIN t3 po  ON po.id = s.col4
WHERE p.state = 2 AND po.state = 3
ORDER BY abc.pro_col1, abc.col3;
```

### I. 根因分析

語意上是一條取兩個欄位 distinct 值的簡單查詢。但執行時每一次 JOIN 都因為 JOIN key 的 **cardinality 過低**而產生大量 intermediate row：

```
JOIN 1: t0(p) ⋈ t1(abc) → 可能 1×N 膨脹（單個 p.id 對應多個 abc.par_col2）
JOIN 2: result ⋈ t2(s)   → 再次膨脹（abc.col3 可能對應多個 s.col3）
JOIN 3: result ⋈ t3(po)  → 再次膨脹
```

膨脹效果是乘數關係：若每層 JOIN 膨脹 10×，4 層就是 10³× = 1000×，最後才用 DISTINCT 收斂到幾條結果。中間耗費的 memory / CPU 全屬浪費。

---

## 2. 解法：Early DISTINCT — 每層 JOIN 後立即去重

```sql
SELECT DISTINCT pro_col1, col3
FROM (
    SELECT DISTINCT t1.pro_col1, t1.col3, s.col4
    FROM (
        SELECT DISTINCT abc.pro_col1, abc.col3
        FROM t1 abc
        INNER JOIN t0 p ON (p.id = abc.par_col2 AND p.state = 2)
    ) t1
    INNER JOIN t2 s ON (s.col3 = t1.col3)
) t2
INNER JOIN t3 po ON (po.id = t2.col4 AND po.state = 3)
ORDER BY t2.pro_col1, t2.col3;
```

**核心邏輯**：在每一層 JOIN 之後立刻用子查詢 + `DISTINCT` 把 intermediate result set 降到最小，阻止膨脹效應傳播到下一層 JOIN。

修改後從數十秒降至數十毫秒。

> 補充（Senior Dev）：這與 SQL 執行計劃中的「predicate pushdown」邏輯相同，只是這裡推的是 DISTINCT 而非 WHERE。PostgreSQL 的 planner 不會自動推 DISTINCT 進子查詢（因為 DISTINCT 與 JOIN 不滿足交換律：先去重再 JOIN vs 先 JOIN 再去重，結果可能不同）。只有在 query 的語意等價時才能手動改寫。判斷何時安全：
> 1. DISTINCT 的 column 集合等於或包含後續 JOIN 的 key
> 2. DISTINCT 不改變後續 JOIN 的 cardinality（即去重後 key 值不丟失）

---

## 3. 重現實驗

```sql
CREATE TABLE rt1 (id INT, info TEXT);
CREATE TABLE rt2 (id INT, info TEXT);
CREATE TABLE rt3 (id INT, info TEXT);
CREATE TABLE rt4 (id INT, info TEXT);

-- rt1: 1000 row (id = 1~1000)
INSERT INTO rt1 SELECT generate_series(1, 1000), 'test';

-- rt2/rt3/rt4: 每個 id=1 有 1000 行（全是 id=1）
INSERT INTO rt2 SELECT 1, 'test' FROM generate_series(1, 1000);
INSERT INTO rt3 SELECT 1, 'test' FROM generate_series(1, 1000);
INSERT INTO rt4 SELECT 1, 'test' FROM generate_series(1, 1000);
```

### I. 原寫法（不斷膨脹）

```sql
EXPLAIN SELECT DISTINCT rt1.id
FROM rt1 JOIN rt2 ON rt1.id = rt2.id
         JOIN rt3 ON rt2.id = rt3.id
         JOIN rt4 ON rt3.id = rt4.id;
```

```
HashAggregate (rows=1000)
  -> Hash Join (rows=1000)
       Hash Cond: (rt4.id = rt1.id)
       -> Seq Scan on rt4 (rows=1000)
       -> Hash (rows=1000)
            -> Hash Join (rows=1000)
                 Hash Cond: (rt3.id = rt1.id)
                 -> Seq Scan on rt3 (rows=1000)
                 -> Hash (rows=1000)
                      -> Hash Join (rows=1000)
                           Hash Cond: (rt2.id = rt1.id)
                           -> Seq Scan on rt2 (rows=1000)
                           -> Hash (rows=1000)
                                -> Seq Scan on rt1 (rows=1000)
```

關鍵觀察：每層 Hash Join 的預估 row 數都是 1000，但因為 rt2/rt3/rt4 的 id 全是 1，實際 JOIN 產生的是 **笛卡爾乘積**。最終 intermediate result 遠超 1000 row，但 DISTINCT 壓回 1 row。

### II. Early DISTINCT 改寫

```sql
EXPLAIN SELECT DISTINCT t2.id
FROM (
    SELECT DISTINCT t1.id
    FROM (
        SELECT DISTINCT rt1.id FROM rt1 JOIN rt2 ON rt1.id = rt2.id
    ) t1
    JOIN rt3 ON t1.id = rt3.id
) t2
JOIN rt4 ON t2.id = rt4.id;
```

```
HashAggregate (rows=1000)
  Group Key: rt1.id
  -> Hash Join (rows=1000)
       Hash Cond: (rt4.id = rt1.id)
       -> Seq Scan on rt4 (rows=1000)
       -> Hash (rows=1000)
            -> HashAggregate (rows=1000)       ← Early DISTINCT
                 Group Key: rt1.id
                 -> Hash Join (rows=1000)
                      Hash Cond: (rt3.id = rt1.id)
                      -> Seq Scan on rt3 (rows=1000)
                      -> Hash (rows=1000)
                           -> HashAggregate (rows=1000)  ← Early DISTINCT
                                Group Key: rt1.id
                                -> Hash Join (rows=1000)
                                     Hash Cond: (rt2.id = rt1.id)
                                     -> Seq Scan on rt2 (rows=1000)
                                     -> Hash (rows=1000)
                                          -> Seq Scan on rt1 (rows=1000)
```

每層 JOIN 後都插入 `HashAggregate (Group Key: rt1.id)`，確保向下傳遞的 row 數不膨脹。

```sql
-- 實際執行：2.052 ms，結果 1 row
SELECT DISTINCT t2.id FROM (
    SELECT DISTINCT t1.id FROM (
        SELECT DISTINCT rt1.id FROM rt1 JOIN rt2 ON rt1.id = rt2.id
    ) t1
    JOIN rt3 ON t1.id = rt3.id
) t2
JOIN rt4 ON t2.id = rt4.id;
-- id
-- ----
--   1
-- (1 row)
```

> 補充（Senior Dev）：
>
> **何時應該用 Early DISTINCT vs 其他方案？**
>
> | 場景 | 方案 |
> |------|------|
> | JOIN key cardinality 極低（如 status='active' 佔 80% 表） | Early DISTINCT（本文） |
> | 只需要 EXISTS 而不需要 JOIN 的 column | 改用 `WHERE EXISTS (SELECT 1 FROM t2 WHERE ...)`，徹底避免 row 膨脹 |
> | 外層 DISTINCT + ORDER BY | 考慮複合 index 讓 planner 走 Index Scan 跳過 redundant row（Index Only Scan + DISTINCT 可能直接用 index skip scan，PG 17+ `enable_indexskipscan`） |
> | 子查詢結果需要傳遞 column 到外層 | 本文的 subquery + DISTINCT |
> | PG 12+ | CTE `MATERIALIZED` 可作為 explicit optimization fence，但 planner 不會自動推 DISTINCT |
>
> **生產中如何發現此問題**：
> ```sql
> EXPLAIN (ANALYZE, BUFFERS) <your_query>;
> ```
> 觀察 `actual rows` vs `estimated rows` 的偏差。若某個 Hash Join / Nested Loop 的 `actual rows` 遠大於 `rows` estimate（如估 1000 但實際 10,000,000），就是膨脹信號。進一步檢查該 JOIN key 的 `n_distinct`（`pg_stats.n_distinct`）。
>
> **內核層改進（德哥建議）**：利用 `pg_stats.n_distinct` 統計資訊，planner 可以在 JOIN key cardinality 極低時自動推入 Early DISTINCT rewrite。目前 PostgreSQL 尚未實現，但這是 query optimizer 的合理改進方向（類似於 MySQL 8.0 的 `derived_condition_pushdown` 或 Oracle 的 cost-based query transformation）。
>
> **簡化寫法（PG 9.4+ 可用 `WITH` 結構化）**：
> ```sql
> WITH
> j1 AS (SELECT DISTINCT abc.pro_col1, abc.col3
>        FROM t1 abc JOIN t0 p ON p.id = abc.par_col2 AND p.state = 2),
> j2 AS (SELECT DISTINCT j1.pro_col1, j1.col3, s.col4
>        FROM j1 JOIN t2 s ON s.col3 = j1.col3)
> SELECT DISTINCT j2.pro_col1, j2.col3
> FROM j2 JOIN t3 po ON po.id = j2.col4 AND po.state = 3
> ORDER BY j2.pro_col1, j2.col3;
> ```
> CTE 版本與 subquery 版本在 planner 行為上等價（PG 12 之前 CTE 預設 materialized fence，12+ 預設 `NOT MATERIALIZED` 讓 planner 可推入 CTE 內），但可讀性更好。

---

# 四、PostgreSQL pgcrypto — 數據加密全面指南

> 來源：[digoal - 固若金湯 — PostgreSQL pgcrypto 加密插件 (2016-07-27)](https://github.com/digoal/blog/blob/master/201607/20160727_02.md)

---

## 1. 加密架構：Server-Side vs Client-Side

pgcrypto 提供的是 **Server-Side 加密**——加密/解密在資料庫服務端執行。

| 加密位置 | 網路傳輸內容 | 防禦範圍 |
|----------|-------------|---------|
| Server-Side（pgcrypto） | 明文 | Data-at-rest（存儲層） |
| Client-Side（應用層加密） | 密文 | Data-at-rest + Data-in-transit |
| **Server-Side + TLS** | 加密通道傳輸明文 | Data-at-rest + Data-in-transit（完整防禦） |

> 德哥建議：服務端加解密 + 傳輸層加密（TLS/SSL）是生產環境的標準組合。單純依靠 pgcrypto 無法防止網路竊聽。

> 補充（Senior Dev）：現代 PostgreSQL（PG 10+）預設啟用 SCRAM-SHA-256 密碼驗證和 TLS 1.3。建議 `ssl = on` + `ssl_min_protocol_version = 'TLSv1.2'` + `pgcrypto` 形成纵深防禦。若只加密個別敏感欄位（如 SSN、信用卡號），pgcrypto 是合適方案；若全表加密需求，考量 PG TDE（PG 16+ `pg_tde` extension 或雲端 RDS 自帶）更透明。

---

## 2. Hash 函數：digest() 與 hmac()

```sql
CREATE EXTENSION pgcrypto;

-- digest(data, algorithm) → bytea
-- 支援: md5, sha1, sha224, sha256, sha384, sha512
-- 若編譯時帶 --with-openssl 可支援更多算法
SELECT digest('I am digoal.', 'md5');
-- \xc3b0fb1147858d2259d92f20668fc8f9

-- hmac(data, key, algorithm) → bytea
-- 比 digest 多一個 key 參數：相同 data + 不同 key → 不同 hash
-- 用於防止 data + hash 同時被篡改（攻擊者無 key 無法重算 hash）
SELECT hmac('I am digoal.', 'this is a key', 'md5');
-- \xc70d0fd2af2382ea8e0a7ffd9edcbd58
```

**重要**：`digest()` 和 `hmac()` 有 text 和 bytea 兩個 overload。若參數類型不明確會調用不同 function，結果不同：

```sql
SELECT digest('\xffffff'::bytea, 'md5');
-- \x8597d4e7e65352a302b63e07bc01a7da
SELECT digest('\xffffff', 'md5');
-- \xd721f40e22920e0fd8ac7b13587aa92d   ← 不同結果！調用了 digest(text, text)
```

> `hmac` 若 key 長度超過 hash 算法的 block size，key 會被先 hash 一次再使用。

> 補充（Senior Dev）：`digest()` / `hmac()` 每次輸入相同 → 輸出相同（deterministic）。這意味著它們**不適合儲存密碼**——rainbow table 攻擊可逆向。應使用 `crypt()` + `gen_salt()`。但 `hmac(key, data, 'sha256')` 適合用於 API Token / HMAC 簽名校驗。

---

## 3. 密碼加密：crypt() + gen_salt()

`crypt()` + `gen_salt()` 的核心設計目標是**加大破解難度**（slow hash + random salt）。每次同一密碼產生的 hash 都不同。

### I. 使用

```sql
-- gen_salt(type [, iter_count]) → salt
-- type: des, xdes, md5, bf
-- iter_count: xdes 需為奇數 (1-16777215, default 725); bf (4-31, default 6)
SELECT gen_salt('bf', 10);
-- $2a$10$qY0amXGalzj14rooSpTf5e

SELECT crypt('this is a pwd source', gen_salt('bf', 10));
-- $2a$10$tvNK2H9mPu1tU5L6oAHdSeze5Hlz7G0y4oEKNg9SlGa06J2sywZHu

SELECT crypt('this is a pwd source', gen_salt('bf', 10));
-- $2a$10$cgJiTAs55vMBqYR1kGMGXuMKZI4dsayna4wgEL4K7duYkD0g25ufW  ← 不同！
```

### II. 密碼驗證

`crypt(password, stored_hash)` → 若密碼正確返回與 `stored_hash` 相同的值。驗證方式：

```sql
SELECT crypt('this is a pwd source', pwd) = pwd
FROM userpwd WHERE userid = 1;
-- 正確密碼 → t, 錯誤密碼 → f
```

**原理**：`crypt()` 從 `stored_hash` 的第二個參數位擷取 salt（格式：`$algorithm$iter$salt$hash`），用相同的 algorithm / iter / salt 重算 hash，與儲存值比對。

### III. 算法對比

| Algorithm | Max Pwd Len | Adaptive | Salt Bits | 說明 |
|-----------|------------|----------|-----------|------|
| bf (Blowfish) | 72 | ✓ | 128 | **推薦**，iter 可控 |
| md5 | unlimited | ✗ | 48 | 次選 |
| xdes | 8 | ✓ | 24 | iter 需奇數 |
| des | 8 | ✗ | 12 | 不建議，古老 |

### IV. 破解速度基準

| Algorithm | Hashes/sec (Pentium 4 1.5GHz) | 破解 8-char \[a-z\] | 破解 8-char \[A-Za-z0-9\] |
|-----------|------------------------------|---------------------|--------------------------|
| crypt-bf/8 (iter=256) | 28 | 246 年 | 251,322 年 |
| crypt-bf/7 | 57 | 121 年 | 123,457 年 |
| crypt-bf/6 | 112 | 62 年 | 62,831 年 |
| crypt-bf/5 | 211 | 33 年 | 33,351 年 |
| crypt-md5 | 2,681 | 2.6 年 | 2,625 年 |
| crypt-des | 362,837 | 7 天 | 19 年 |
| sha1 | 590,223 | 4 天 | 12 年 |
| md5 | 2,345,086 | 1 天 | 3 年 |

**iter_count 選擇原則**：以當前 CPU 實測為準，讓 `crypt()` 執行時間在 **4-100 ms** 之間（每秒 10-250 次），平衡安全性與 user login latency。

> 補充（Senior Dev）：
> - `bf` (Blowfish/bcrypt) 的 iter_count 對應的是 2^iter 次迭代。iter=10 → 1024 次。現代 CPU 上 iter=10 約 50-100ms，iter=12 約 200-400ms。PG 14+ 可考慮 `scram-sha-256`（內建密碼驗證機制，不需要 pgcrypto 處理密碼欄位）
> - `crypt()` 的 salt 長度：bf 為 128-bit (22 char Base64)，md5 為 48-bit。salt 太短會降低抗碰撞性
> - **Timing Attack 注意**：`crypt()` 使用 constant-time comparison，但若應用層用 `SELECT ... WHERE pwd = crypt(...)` 而非 `crypt(...) = stored_hash`，可能洩漏 timing info。務必使用 constant-time 比對方式

---

## 4. PGP 對稱加密

遵循 OpenPGP (RFC 4880)。對稱加密用同一密鑰加解密。

```sql
-- pgp_sym_encrypt(data, psw [, options]) → bytea
-- pgp_sym_decrypt(msg, psw [, options]) → text
```

```sql
SELECT pgp_sym_encrypt(
  'i am digoal', 'pwd',
  'cipher-algo=aes256, compress-algo=2, compress-level=9'
);

SELECT pgp_sym_decrypt(
  '\xc30d040403024581...'::bytea,
  'pwd'
);
-- → i am digoal
```

**options** 支援：`cipher-algo`（aes256 / aes128 / bf / 3des / cast5）、`compress-algo`（0=none, 1=ZIP, 2=ZLIB）、`compress-level`（0-9）、`convert-crlf`、`disable-mdc`、`s2k-mode`、`s2k-digest-algo`、`s2k-cipher-algo`。

> 補充（Senior Dev）：PGP symmetric encrypt 內部流程：隨機產生 session key → 用 session key 對稱加密 data → 用 password + S2K（String-to-Key）派生 key 加密 session key → 組裝 PGP message。這意味著即使兩次用相同 password 加密相同 data，輸出也不同（因為 session key 是隨機的）。預設 `cipher-algo=aes128`；建議生產用 `cipher-algo=aes256`。

---

## 5. PGP 公鑰加密

### I. GPG 金鑰生成

```bash
gpg --gen-key
# 選擇: (1) DSA and Elgamal (default)
# keysize: 2048
# expire: 0 (never)
# Real name: digoal

# 導出公鑰和私鑰
gpg -a --export digoal > public.key
gpg -a --export-secret-keys digoal > secret.key
```

### II. 金鑰格式轉換（dearmor）

GPG 匯出的金鑰是 ASCII-armor 格式（`-----BEGIN PGP PUBLIC KEY BLOCK-----`），pgcrypto 需要 bytea 格式。使用 `dearmor()` 轉換：

```sql
SELECT dearmor('-----BEGIN PGP PUBLIC KEY BLOCK-----
Version: GnuPG v1.4.5 (GNU/Linux)

mQGiBFGgIDgRBADALXrWA...
=XGEV
-----END PGP PUBLIC KEY BLOCK-----');
-- → \x9901a20451a02038110400c02d7ad... (raw bytea)
```

### III. 加解密

```sql
-- 公鑰加密
SELECT pgp_pub_encrypt('i am digoal', '<public_key_bytea>');

-- 私鑰解密
SELECT pgp_pub_decrypt('<encrypted_bytea>', '<private_key_bytea>');
```

若 GPG 生成私鑰時設了 passphrase，解密需傳入：

```sql
pgp_pub_decrypt(msg bytea, key bytea, psw text [, options text])
```

### IV. 輔助函數

```sql
-- 查詢加密數據使用對稱密鑰還是公鑰
SELECT pgp_key_id('<encrypted_bytea>');
-- SYMKEY 或公鑰 ID (如 1B437BCCD670D845)

-- bytea ↔ ASCII-armor 轉換
SELECT armor(data bytea);     -- → text (PGP ASCII-armor)
SELECT dearmor(data text);    -- → bytea

-- 產生隨機加密用 bytea
SELECT gen_random_bytes(32);  -- 32 random bytes
```

> 補充（Senior Dev）：
>
> **公鑰加密的生產考量**：
> 1. **Key Rotation**：公鑰/私鑰應定期輪換。需維護 key version 欄位與歷史私鑰的安全備份（解密歷史數據需要舊私鑰）
> 2. **私鑰存儲**：絕對不能存在同一資料庫中。推薦方案：私鑰存在 HashiCorp Vault / AWS KMS / Azure Key Vault，應用層呼叫 KMS decrypt → 拿到明文後才送入 PG（或由應用層直接解密不回傳私鑰給 PG）
> 3. **效能**：PGP 公鑰加密/解密比對稱加密慢約 10-50×（RSA/Elgamal 運算）。大批量加密場景建議用 envelope encryption 模式（generate random symmetric key → encrypt data with symmetric key → encrypt symmetric key with public key）
> 4. **現代替代**：`pg_sodium` / `pgsodium` extension 基於 libsodium，提供 `crypto_box()` / `crypto_secretbox()` 等更現代的 AEAD（Authenticated Encryption），比 pgcrypto 的 PGP 實現更新、更安全、更快。但 pgcrypto 是 contrib 內建，相容性最好
>
> **對稱加密的 Key Management**：
> - 絕對不要把 key 寫死在 SQL / stored procedure 中
> - 使用 `SET SESSION myapp.encryption_key = '...'`（Custom GUC，PG 9.2+）在 connection pool 初始化時設定，function 內用 `current_setting('myapp.encryption_key')` 讀取，避免 key 出現在 query log / pg_stat_activity 中
> - 或將加密 key 存在另一張受限權限的 table 中，搭配 Row-Level Security 限制訪問
>
> **gen_random_bytes 的 entropy 來源**：PG 10+ 使用 OpenSSL 的 CSPRNG（`RAND_bytes()`），在 Linux 上從 `/dev/urandom` 獲取 entropy；PG 16+ 改用内置 CSPRNG，不再依赖 OpenSSL。

---

## 6. 三種加密方案適用場景總結

| 方案 | 適用場景 | 特點 |
|------|----------|------|
| `crypt()` + `gen_salt()` | 密碼儲存 | Slow hash + random salt，每次不同輸出；高破解難度；適合短字串（密碼 ≤ 72 chars） |
| PGP 對稱加密 | 大量數據加密（資料欄位、文件） | 效能好；需管理對稱密鑰（建議存在外部 KMS / custom GUC） |
| PGP 公鑰加密 | 高安全場景（僅應用層持私鑰） | 加密與解密分離；私鑰不存 DB；適合合規需求（PCI-DSS、HIPAA） |

---

## 參考

1. [PostgreSQL pgcrypto 官方文檔](http://www.postgresql.org/docs/9.3/static/pgcrypto.html)
2. [GnuPG Manual](http://www.gnupg.org/gph/en/manual.html)
3. [德哥 blog — pgcrypto 相關系列](http://blog.163.com/digoal@126/blog/static/163877040201342233131835/)

---

# 五、PostgreSQL 千億級 Regex / 模糊查詢 — pg_trgm + GIN 效能實測

> 來源：[digoal - PostgreSQL 1000亿数据量 正则匹配 速度与激情 (2016-03-07)](https://github.com/digoal/blog/blob/master/201603/20160307_01.md)
>
> 2026 更新：補充 pg_trgm 內部機制、PG 14-17 演進、pg_bigm 替代方案、Production 層級建議。

---

## 測試規模

| 項目 | 數值 |
|------|------|
| Cluster | 8 台實體主機（16 core / host），共 **240 個 data node** |
| Total rows | **1,008 億** (100.8 billion) |
| Table size | **4,158 GB** (~4.1 TB) |
| Data characteristic | 12-char hex string（`md5(random()::text)` 前 48-bit），83.7% 唯一值 |
| B-tree index (info) | **2,961 GB** |
| B-tree index (reverse(info)) | **2,961 GB** |
| GIN index (gin_trgm_ops) | **2,300 GB** |

![Regex Performance 100 Billion](images/regex_perf_100billion.png)

---

## 數據生成

```bash
# 創建 distributed table（基於 PG 的 MPP 架構）
psql -c "CREATE TABLE t_regexp_100billion DISTRIBUTED RANDOMLY"

# 分批生成 1008 億行，每批 1 億行
for ((i=1; i<=1008; i++)); do
  psql -c "COPY (
    SELECT substring(md5(random()::text), 1, 12)
    FROM generate_series(1, 100000000)
  ) TO stdout" \
  | psql -c "COPY t_regexp_100billion FROM stdin"
done

# 創建三組索引
psql -c "SET maintenance_work_mem = '4GB';
         CREATE INDEX idx_t_regexp_100billion_1
         ON t_regexp_100billion(info)"

psql -c "SET maintenance_work_mem = '4GB';
         CREATE INDEX idx_t_regexp_100billion_2
         ON t_regexp_100billion(reverse(info))"

psql -c "SET maintenance_work_mem = '4GB';
         CREATE INDEX idx_t_regexp_100billion_gin
         ON t_regexp_100billion USING gin (info gin_trgm_ops)"
```

### I. 數據特徵

```sql
SELECT count(*) FROM t_regexp_100billion;
-- 100800000000 (1,008 億)
-- Time: 228,721 ms (~3.8 min for COUNT(*))

-- 抽樣驗證唯一性（隨機 offset 100 萬行中僅 250 個重複）
SELECT count(DISTINCT info)
FROM (SELECT * FROM t_regexp_100billion
      OFFSET 1299422811 LIMIT 1000000) t;
-- count: 999750

-- psql 統計資訊
ALTER TABLE t_regexp_100billion ALTER COLUMN info SET STATISTICS 10000;
ANALYZE t_regexp_100billion;

-- n_distinct: -0.836834   → 約 83.68% 唯一值
-- most_common_freqs: 最高頻值佔比 1e-06（每值約 10 萬次出現）

-- 實際驗證「最高頻值」7f68d12d2205
SELECT count(*) FROM t_regexp_100billion WHERE info = '7f68d12d2205';
-- count: 54（而非統計預估的 10 萬次 — 採樣偏差）
```

> 補充（Senior Dev）：`n_distinct = -0.837` 是柱狀圖估算值，負值表示「比例」而非「絕對數」。此處由於 random MD5 分佈極均勻，GIN index 每個 trigram 的 posting list 長度非常接近理論值（1008 億 ÷ 16^3 ≈ 2460 萬 / trigram），selectivity 可精確預測。

---

## 內部機制：pg_trgm 如何讓 Regex 變快

### II. Trigram 原理

`pg_trgm` 將每個字串拆成長度為 3 的連續子串（trigram）。例如 `'80ebcdd47006'`：

```
Trigrams: 80e, 0eb, ebc, bcd, cdd, dd4, d47, 470, 700, 006
```

GIN index 的 key = trigram，value = 包含該 trigram 的所有 row 的 posting list。

### III. 查詢流程（以 `info ~ 'e7add04871'` 為例）

```
1. PG 將 regex 轉換為 trigram 集合：
   'e7add04871' → {e7a, 7ad, add, dd0, d04, 048, 487, 871}

2. GIN scan 對每個 trigram 取出 posting list，取交集：
   ∩ {rows with 'e7a', '7ad', 'add', ...}

3. Bitmap Heap Scan：將交集結果中的 page 全部讀出

4. Recheck：用原始 regex 對候選 row 進行精確匹配
   （GIN + pg_trgm 是 lossy，trigram 交集只保證候選集包含答案）
```

> 補充（Senior Dev）：pg_trgm 的最小有效 pattern 長度取決於 `pg_trgm.word_similarity_threshold` 和 GIN 的 selectivity。單個字符或兩個字符的 regex（如 `~ 'a.'`）無法產生足夠的 trigram 過濾，會退化成 Seq Scan。實務上 pattern ≥ 3 characters 才能有效利用 GIN 過濾。

---

## 四種查詢模式效能實測

### IV. Prefix 匹配（前綴查詢 `^`）

```sql
SELECT ctid, tableoid, info
FROM t_regexp_100billion
WHERE info ~ '^80ebcdd47';
-- 返回 52 rows，3,226 ms
```

實際使用 B-tree index（`info` column），`~ '^...'` 被 optimizer 轉換為 B-tree range scan（`>= '80ebcdd47' AND < '80ebcdd48'`）。

```
EXPLAIN (ANALYZE, VERBOSE, BUFFERS, COSTS, TIMING)
SELECT ... WHERE info ~ '^80ebcdd47';
-- 240 個 node 並行執行
-- Planning time: 0.061 ms
-- Execution time: 3,112 ms
```

| 指標 | 數值 |
|------|------|
| 模式 | B-tree index scan（prefix → range） |
| 時間 | **3.1 秒** |
| 結果數 | 52 rows（1,008 億中） |

---

### V. Suffix 匹配（後綴查詢 `reverse()` 技巧）

```sql
-- 查詢以 'f42d12089b' 結尾的 row
-- 做法：反轉字串後做 prefix 查詢
SELECT ctid, tableoid, info
FROM t_regexp_100billion
WHERE reverse(info) ~ '^f42d12089b';
-- 返回 46 rows，3,141 ms
```

> 補充（Senior Dev）：這是 PG 中處理 suffix matching 的經典技巧。原理是把 `LIKE '%abc'` 轉為 `reverse(col) LIKE 'cba%'`，再用 B-tree index 加速。**不需要 pg_trgm**，只需一個 `reverse()` B-tree index。在 PG 17 中，若 `reverse()` 被標記為 IMMUTABLE（確實如此），expression index 直接可用。此技巧對任意長度的後綴都有效，不像 pg_trgm 受 trigram 最小長度限制。

```
EXPLAIN (ANALYZE, VERBOSE, BUFFERS, COSTS, TIMING)
SELECT ... WHERE reverse(info) ~ '^f42d12089b';
-- 240 個 node 並行執行
-- Execution time: 3,112 ms
```

| 指標 | 數值 |
|------|------|
| 模式 | `reverse()` B-tree index |
| 時間 | **3.1 秒** |
| 結果數 | 46 rows |

---

### VI. 中間包含匹配（任意位置 substring）

無法用 B-tree prefix trick，必須依賴 GIN + pg_trgm：

```sql
SELECT ctid, tableoid, info
FROM t_regexp_100billion
WHERE info ~ 'e7add04871';
-- 返回 32 rows，4,971 ms
```

```
EXPLAIN (ANALYZE, VERBOSE, BUFFERS, COSTS, TIMING)
SELECT ... WHERE info ~ 'e7add04871';
-- 240 個 node 並行執行
-- Execution time: 4,898 ms
```

| 指標 | 數值 |
|------|------|
| 模式 | GIN (gin_trgm_ops) → Bitmap Index Scan |
| 時間 | **4.9 秒** |
| 結果數 | 32 rows |

> 補充（Senior Dev）：4.9 秒中有相當一部分花在 Recheck 上。因為 pattern 長度僅 10 字符（8 個 trigram），candidate set 較大。若 pattern 更長，trigram 更多，交集更精確，速度會更快。此例最慢是因為需要從 1,008 億行中搜尋任意位置 substring——沒有 prefix/suffix 的 shortcut。

---

### VII. 正則表達式匹配（通用 regex）

```sql
-- . 匹配任意字符
SELECT ctid, tableoid, info
FROM t_regexp_100billion
WHERE info ~ '.3918.209f';
-- 返回 38 rows，3,581 ms
```

```
EXPLAIN (ANALYZE, VERBOSE, BUFFERS, COSTS, TIMING)
SELECT ... WHERE info ~ '.3918.209f';
-- 240 個 node 並行執行
-- Execution time: 3,622 ms
```

```sql
-- 複雜 regex：字符類 + 選擇 + 量詞
SELECT ctid, tableoid, info
FROM t_regexp_100billion
WHERE info ~ 'ab2..d[1|f]3c8';
-- 返回 95 rows，4,718 ms
```

```
EXPLAIN (ANALYZE, VERBOSE, BUFFERS, COSTS, TIMING)
SELECT ... WHERE info ~ 'ab2..d[1|f]3c8';
-- 240 個 node 並行執行
-- Execution time: 4,649 ms
```

| Pattern | 匹配數 | 時間 |
|---------|--------|------|
| `.3918.209f` | 38 rows | **3.6 秒** |
| `ab2..d[1|f]3c8` | 95 rows | **4.6 秒** |

> 補充（Senior Dev）：pg_trgm 會從 regex pattern 中提取所有「確定性字符序列」生成 trigram。例如 `ab2..d[1|f]3c8` → trigram 包括 `ab2`, `b2d`, `2d1`, `d13`, `13c`, `3c8` 等（`.` 和 `[]` 視為不可確定符號，不貢獻 trigram）。Regex 越具體 / 確定性字符越多，trigram 越多 → 候選集越小 → 越快。

---

## 效能總結

| 查詢模式 | 索引類型 | 時間 | 千億中命中 |
|----------|----------|------|-----------|
| Prefix `^` | B-tree (info) | 3.1s | 52 rows |
| Suffix (reverse) | B-tree (reverse(info)) | 3.1s | 46 rows |
| 中間包含 | GIN (gin_trgm_ops) | 4.9s | 32 rows |
| 簡單 regex `.3918.209f` | GIN (gin_trgm_ops) | 3.6s | 38 rows |
| 複雜 regex `ab2..d[1` | GIN (gin_trgm_ops) | 4.6s | 95 rows |

> 補充（Senior Dev）：三種索引總體積 = 2,961 + 2,961 + 2,300 = **8,222 GB**（約 table 體積的 2x）。若只需 regex / 模糊查詢，GIN 一個索引就涵蓋 prefix、suffix、中間匹配三種場景（pg_trgm 自動處理 prefix/suffix），不需額外 B-tree。原文章建三個索引是為了對比測試。

---

## PG 14-17 pg_trgm 演進

| 版本 | 改進 |
|------|------|
| PG 14 | GIN index 的 `gin_clean_pending_list` 效能提升；`pg_trgm.word_similarity_threshold` 更靈活 |
| PG 15 | `pg_trgm` 支援 ICU collation 下的 trigram 提取；regex engine 內部改寫提升 `~` operator 效能 |
| PG 16 | `pg_trgm.strict_word_similarity` 支援 multi-byte encoding 優化 |
| PG 17 | GIN parallel index build 支援 `pg_trgm`，大規模建索引更快 |

---

## Senior Dev：Production 設計建議

### VIII. Index 選擇決策

| 場景 | 推薦 | 原因 |
|------|------|------|
| 只需 prefix 查詢 | B-tree `(info text_pattern_ops)` | 最輕量，不需 pg_trgm |
| 只需 suffix 查詢 | B-tree `(reverse(info))` | 比 GIN 小、更快（無 Recheck） |
| Regex / 模糊 / 任意位置 | GIN `gin_trgm_ops` | 唯一的 regex 加速方案 |
| 中日韓文本混合 regex | GIN `pg_bigm`（2-gram） | 比 pg_trgm 更適合 CJK，不需辭典 |

### IX. GIN + pg_trgm 的參數調優（PG 14+）

```sql
-- 控制 GIN fast update pending list 大小（預設 4MB）
-- 寫入密集型：減小以降低查詢時 pending list scan 開銷
-- 查詢密集型：增大以減少 index 寫入次數
SET gin_pending_list_limit = '2MB';

-- pg_trgm 相似度門檻（預設 0.3）
-- 用於 similarity() / word_similarity() 函數，不影響 ~ operator
SET pg_trgm.similarity_threshold = 0.5;
```

### X. Regex 查詢優化技巧

```sql
-- 壞寫法：LIKE '%abc%'（無法用 B-tree，退化成 Seq Scan）
-- 好寫法：用 pg_trgm GIN index
SELECT * FROM t WHERE info ~ 'abc';

-- 壞寫法：regex 太短（< 3 characters），trigram 過濾無效
-- SELECT * FROM t WHERE info ~ 'a.';  -- 退化成 Seq Scan

-- 好寫法：regex pattern 確保 ≥ 3 個確定性字符
SELECT * FROM t WHERE info ~ 'abc.*xyz';
```

### XI. 大規模部署考量

**分佈式查詢（MPP / Citus / Greenplum）：**
- GIN index 在每個 shard 上獨立運作，查詢效能線性擴展
- 瓶頸在「最慢的 shard」，而非總數據量
- Recheck 是 shard-local 操作，不受跨節點 network 影響

**Single-node PG 17 + Parallel Query：**
- 若數據量在 TB 級（非千億），single-node PG 17 的 parallel bitmap heap scan 可有效利用 multi-core
- `max_parallel_workers_per_gather` 建議設為 CPU core 數的 50-75%

### XII. `pg_bigm` vs `pg_trgm`（PG 17 對比）

| 面向 | pg_trgm | pg_bigm |
|------|---------|---------|
| 最小 token | 3-gram | 2-gram |
| CJK 支援 | 需配合 ICU collation | 原生支援 |
| Regex `~` 加速 | ✓ | ✓ |
| LIKE `%abc%` 加速 | ✓ | ✓ |
| Index 大小 | 較大（3-gram） | 更大（2-gram，更多 token） |
| 查詢速度 | 較快（token 少） | 較慢（token 多，但 selectivity 更高） |

選擇：純英文/數字 → pg_trgm；CJK/多語言混合 → pg_bigm。

---

# 六、PostgreSQL × 12306 搶火車票 — varbit / SKIP LOCKED / Array 架構設計

> 來源：[digoal - PostgreSQL 與 12306 搶火車票的思考 (2016-11-24)](https://github.com/digoal/blog/blob/master/201611/20161124_02.md)
>
> 相關：[門禁廣告銷售系統與 PostgreSQL 實現](https://github.com/digoal/blog/blob/master/201611/20161124_01.md)

---

這篇文章從 12306 的業務場景出發，剖析高併發購票系統的設計痛點，並用 PostgreSQL 的 10 個特性逐一對應解法。核心挑戰：**查詢餘票的高併發、購票的 row lock 衝突、座位空洞最小化、路徑規劃**。

---

## 1. 業務場景與設計痛點

### I. 核心功能

| 功能 | 類型 | 挑戰 |
|------|------|------|
| 查詢餘票 | 高併發 read + aggregate | CPU / IO 密集，幾億 user 同時查 |
| 購票 | 高併發 write（庫存遞減） | row lock 衝突，同一座位可能被多人搶 |
| 中途票 | 區段購買，避免空洞 | 同一座位不同區段可賣給不同人 |
| 路徑規劃 | 無直達時推薦中轉 | Graph traversal（pgrouting） |
| 對賬 / 退改簽 | 非同步 / 批量 | 一致性要求 |

![餘票查詢介面](images/20161124_02_pic_001.png)

### II. 核心矛盾

> 「眼見不一定為實」——高併發期間，用戶看到的餘票資訊可能在付款前就已失效。因此**餘票查詢可以是非同步統計結果**，允許一定延遲。

![12306 餘票大盤](images/20161124_02_pic_002.png)

![流量尖峰](images/20161124_02_pic_004.png)

### III. 座位空洞問題

從北京到上海，途經天津、徐州、南京、蘇州。如果天津→南京被人買了，剩下的北京→天津、南京→上海**還能繼續賣**。目標是最大化利用率，減少「有票不能賣」的情況。

---

## 2. PostgreSQL 10 大法寶

### IV. 法寶 1：varbit — 座位區段銷售狀態

每個座位用 varbit 表示途經站點的已售狀態。6 個站點 → 6-bit：

```
'000000'         -- 全程未售
'010000'         -- 天津→徐州已售（只設起點到終點-1 的 bit）
```

**餘票統計**：對 varbit 做 bitwise AND + count 即可。統計任意組合站點的餘票（北京→天津、北京→徐州...蘇州→上海）。

```sql
-- UDF：統計指定起始站餘票
udf_count(varbit, start_pos, end_pos) RETURNS record

-- 本質是：bitrange = 全 0 的座位數量
SELECT count(*) FROM train_sit
WHERE getbit(station_bit, start, end-1) = repeat('0', end-start)::varbit;
```

### V. 法寶 2：Array + GIN — 快速查詢可搭乘車次

```sql
CREATE TABLE train (
  id INT PRIMARY KEY,
  go_date DATE,
  train_num NAME,
  station TEXT[]  -- 途經站點陣列
);

-- GIN index 支援 @> (包含) 運算
CREATE INDEX ON train USING GIN (station);

-- 查詢從北京到南京的所有車次
SELECT train_num FROM train
WHERE station @> ARRAY['北京', '南京'];
```

每秒可處理數十萬次查詢。

> 補充（Senior Dev）：`@>` 只能確認「車次途經這兩個站」，不能確保「北京在南京之前」。需要再加 `array_pos(station, '北京') < array_pos(station, '南京')` 條件。實務中可將此邏輯預先計算為 generated column。

### VI. 法寶 3：SKIP LOCKED — 避免購票 Lock 衝突

核心購票邏輯：同一車次同一座位可能被多人同時搶，傳統 `FOR UPDATE` 會讓其他人等待。用 `SKIP LOCKED` 跳過已被鎖定的 row：

```sql
SELECT * FROM train_sit
WHERE train_num = 'G1921'
  AND go_date = '2026-01-20'
  AND sit_level = '二等座'
  AND getbit(station_bit, from_pos, to_pos-1) = repeat('0', to_pos-from_pos)::varbit
ORDER BY station_bit DESC  -- 優先售賣已售中途票的座位（減少空洞）
FOR UPDATE SKIP LOCKED
LIMIT 1;
```

> 補充（Senior Dev）：
> - `SKIP LOCKED` 在 PG 9.5+ 可用，本質是 work queue pattern（跳過被其他 worker 正在處理的 item）
> - `ORDER BY station_bit DESC` 的設計意圖：`111000` 的座位比 `110000` 更優先售出（已售區段越多 → 剩餘區段越少 → 先清掉減少空洞），符合鐵路最大化利用率的目標
> - `NOWAIT` vs `SKIP LOCKED`：NOWAIT 遇到 locked row 直接報錯（需 application retry），SKIP LOCKED 透明跳過。購票場景 SKIP LOCKED 更合適
> - **熱點問題**：即使有 SKIP LOCKED，如果同一車次只剩少數座位，大量 connection 仍會競爭同一批 row。德哥原方案中有 `mod(pg_backend_pid(), 100) = mod(pk, 100)` 的 hash 分流技巧（將不同 PID 分流到不同 row range），但有「部分 PID 賣完後找不到票」的風險

### VII. 法寶 4：CURSOR

大批量查詢使用 CURSOR 減少重複掃描。

### VIII. 法寶 5：路徑規劃（pgrouting）

當直達車無票時，自動計算中轉路線（時間最短 / 價格最低 / 中轉次數最少）。

![鐵路網](images/20161124_02_pic_003.gif)

```sql
-- pgrouting 支援 Dijkstra / A* / 多種路徑演算法
SELECT * FROM pgr_dijkstra(
  'SELECT id, source, target, cost FROM railway_edges',
  start_station_id, end_station_id
);
```

### IX. 法寶 6-10

| # | 法寶 | 用途 |
|---|------|------|
| 6 | Parallel Query | 餘票統計時多核加速計算 |
| 7 | Resource Isolation (cgroups) | 尖峰時刻確保關鍵業務（購票）有足夠 CPU/IO/Memory |
| 8 | Sharding（plproxy / Citus / pg-xl） | 全國鐵路數據分庫儲存 |
| 9 | Recursive CTE | 查詢鐵路網中可達的所有站點 / 轉乘路徑 |
| 10 | MPP（Greenplum / PG-XL） | 春節運力預測、加開車次數據挖掘 |

---

## 3. 資料庫設計（偽代碼）

### X. 核心表結構

```sql
-- 列車資訊
CREATE TABLE train (
  id INT PRIMARY KEY,
  go_date DATE,
  train_num NAME,
  station TEXT[]     -- 途經站點陣列
);

-- 座位資訊（每個座位一條 row）
CREATE TABLE train_sit (
  id SERIAL8 PRIMARY KEY,
  tid INT REFERENCES train(id),
  bno INT,                -- 車廂號
  sit_level TEXT,         -- 席別
  sit_no INT,             -- 座位號
  station_bit VARBIT      -- 途經站點 BIT 位
);
```

### XI. 購票函數

```sql
CREATE OR REPLACE FUNCTION buy(
  INOUT i_train_num NAME,
  INOUT i_fstation TEXT,
  INOUT i_tstation TEXT,
  INOUT i_go_date DATE,
  INOUT i_sits INT,          -- 購買張數
  OUT o_slevel TEXT,
  OUT o_bucket_no INT,
  OUT o_sit_no INT,
  OUT o_order_status BOOLEAN
) ...
```

核心步驟：
1. 從 `train` 表查詢站點陣列 + 起終點位置
2. 用 `FOR UPDATE SKIP LOCKED` 選取符合條件的座位
3. 用 `set_bit()` 更新 `station_bit`，標記已售站點範圍
4. `ORDER BY station_bit DESC` 優先清掉已售中途票的座位

### XII. `array_pos()` 輔助函數

```sql
CREATE OR REPLACE FUNCTION array_pos(a ANYARRAY, b ANYELEMENT) RETURNS INT AS $$
DECLARE
  i INT;
BEGIN
  FOR i IN 1..array_length(a, 1) LOOP
    IF b = a[i] THEN RETURN i; END IF;
  END LOOP;
  RETURN NULL;
END;
$$ LANGUAGE plpgsql;
```

> 補充（Senior Dev）：生產中建議用 C function 實作 array_pos（O(1) pointer arithmetic vs plpgsql 的 O(n) loop），或用 PG 內建的 `array_position()`（PG 9.5+）。

### XIII. Partial Index 設計

```sql
-- 只索引還有空位（非全 1）的座位，減少掃描
CREATE INDEX idx_train_sit_station_bit
  ON train_sit (station_bit)
  WHERE station_bit <> repeat('1', 14)::varbit;
```

> 補充（Senior Dev）：partial index 的關鍵：`WHERE station_bit <> repeat('1', 14)::varbit` 確保只有未售完的座位在 index 中。在 10M row 只賣出 10% 的場景，index 只掃 1M row 而非 10M。

---

## 4. 效能基準（PL/pgSQL buy() 函數）

測試環境：1 趟車、14 個站點、196M 座位（200 萬車廂 × 98 座位）。

| 模式 | TPS | Latency |
|------|-----|---------|
| 不加 NOWAIT | 1,831 tps | 8.7 ms |
| 加 NOWAIT (`FOR UPDATE NOWAIT`) | **7,818 tps** | 2.0 ms |

NOWAIT 模式大幅降低鎖等待時間（失敗直接報錯 → application retry，而非在 DB 內排隊）。

### XIV. 原始 PL/pgSQL buy() 函數（完整版）

```sql
CREATE OR REPLACE FUNCTION buy(
  INOUT i_train_num NAME,
  INOUT i_fstation TEXT,
  INOUT i_tstation TEXT,
  INOUT i_go_date DATE,
  OUT o_slevel TEXT,
  OUT o_bucket_no INT,
  OUT o_sit_no INT,
  OUT o_order_status BOOLEAN
) RETURNS RECORD AS $$
DECLARE
  curs1 REFCURSOR;
  curs2 REFCURSOR;
  v_row INT;
  v_station TEXT[];
  v_train_id INT;
  v_train_bucket_id INT;
  v_train_sit_id INT;
  v_from_station_idx INT;
  v_to_station_idx INT;
  v_station_len INT;
BEGIN
  SET enable_seqscan = off;
  v_row := 0;
  o_order_status := false;

  -- 查詢列車資訊與站點位置
  SELECT array_length(station,1), station, id,
         array_pos(station, i_fstation),
         array_pos(station, i_tstation)
  INTO v_station_len, v_station, v_train_id,
       v_from_station_idx, v_to_station_idx
  FROM train
  WHERE train_num = i_train_num AND go_date = i_go_date;

  IF NOT found OR
     v_from_station_idx IS NULL OR
     v_to_station_idx IS NULL OR
     v_from_station_idx >= v_to_station_idx THEN
    RETURN;
  END IF;

  -- Cursor 1：優先找已有中途票的座位（station_bit 非全 0 也非全 1）
  OPEN curs2 FOR
    SELECT tid, tbid, sit_no FROM train_sit
    WHERE (station_bit &
           bitsetvarbit(repeat('0', v_station_len-1)::varbit,
             v_from_station_idx-1,
             v_to_station_idx - v_from_station_idx, 1))
          = repeat('0', v_station_len-1)::varbit
      AND station_bit <> repeat('1', v_station_len-1)::varbit
    LIMIT 1
    FOR UPDATE NOWAIT;

  LOOP
    FETCH curs2 INTO v_train_id, v_train_bucket_id, o_sit_no;
    IF found THEN
      UPDATE train_sit
      SET station_bit = bitsetvarbit(station_bit,
            v_from_station_idx-1,
            v_to_station_idx - v_from_station_idx, 1)
      WHERE CURRENT OF curs2;
      GET DIAGNOSTICS v_row = ROW_COUNT;
      IF v_row = 1 THEN
        SELECT sit_level, bno INTO o_slevel, o_bucket_no
        FROM train_bucket WHERE id = v_train_bucket_id;
        CLOSE curs2;
        o_order_status := true;
        RETURN;
      END IF;
    ELSE
      CLOSE curs2;
      EXIT;
    END IF;
  END LOOP;

  -- Cursor 2：找全新空座位
  OPEN curs1 FOR
    SELECT id, tid, strpos(sit_bit::text, '0'), sit_level, bno
    FROM train_bucket
    WHERE sit_remain > 0
    LIMIT 1
    FOR UPDATE NOWAIT;
  ...
END;
$$ LANGUAGE plpgsql;
```

> `pgrowlocks()` 用於解決熱點鎖等待的嘗試被放棄：德哥原文標註改用 pgrowlocks 查詢已鎖定 row 的成本約 300ms，不如 NOWAIT + application retry 划算。

> 補充（Senior Dev）：
>
> **現代改進方向**：
> 1. **Range Type + Exclusion Constraint**（PG 9.2+）：替代 varbit，`int4range(from_pos, to_pos, '[)')` + `EXCLUDE USING GIST (train_id WITH =, seat_range WITH &&)`，讓 DB kernel 層面保證同一座位區段不重複售出，而非 application-level 的 SKIP LOCKED
> 2. **Advisory Lock per Seat**：對於極熱門車次的最後幾個座位，SKIP LOCKED 也無能為力（所有 connection 都 SKIP 同一批 row → 沒 row 可搶）。此時可用 `pg_try_advisory_xact_lock(seat_id)` 做 per-seat 排隊
> 3. **Queue-based 購票**：由一個 producer 合併購票請求（`LISTEN/NOTIFY` + pg_background），將隨機競爭變成 FIFO 排隊（減少 DB lock contention）
> 4. **真正的 12306 技術路線**：12306 最終採用的不是傳統 RDBMS 的 row lock 方案，而是**記憶體庫存計算** + **排隊系統**（使用 GemFire / 自研分散式記憶體網格），只在最後扣庫存時寫 DB。PG 的方案是 demo 層面的架構探討，不應直接用於億級 concurrent 的生產環境

---

## 5. varbit 優先級策略：最大化利用率

座位選擇時的優先級規則：已售中途票越多的座位 → 優先選。

```
座位 A: 111000 (前 3 站已售)
座位 B: 110000 (前 2 站已售)
查詢:   最後 2 站 (000011)

座位 A: 111000 | 000011 = 111011 ← 選這個（剩前 4 站可賣）
座位 B: 110000 | 000011 = 110011 ← 留著（更多彈性）

ORDER BY station_bit DESC 實現此優先級
```

---

## 6. 阿里雲 RDS PG 客製化增強

| 功能 | 說明 |
|------|------|
| `bit_count_range_zero(varbit, start, end)` | 統計指定範圍內 bit=0 的數量（餘票統計） |
| `array_pos()` C 版本 | O(1) 效能 |
| `set_bit()` / `get_bit()` 的 varbit 批量操作 | 購票一次更新多位 |

---

## 參考

1. [setbitvarbit UDF](http://blog.163.com/digoal@126/blog/static/163877040201302192427651/)
2. [阿里雲 RDS PG 用戶畫像推薦系統](https://github.com/digoal/blog/blob/master/201610/20161021_01.md)
3. [pgrouting 動態路徑規劃](https://github.com/digoal/blog/blob/master/201607/20160710_01.md)
4. [門禁廣告銷售系統與 PG](https://github.com/digoal/blog/blob/master/201611/20161124_01.md)
