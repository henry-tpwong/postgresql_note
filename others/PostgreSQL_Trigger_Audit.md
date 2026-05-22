# PostgreSQL Trigger Audit — DML 欄位變更審計 + DDL 結構變更審計

> 來源（德哥 digoal Trigger 審計二部曲）：
> - [DDL 審計 — use event trigger record user who alter table (2014-12-11)](https://github.com/digoal/blog/blob/master/201412/20141211_02.md)
> - [DML 審計 — use trigger audit record which column modified (2014-12-14)](https://github.com/digoal/blog/blob/master/201412/20141214_01.md)

---

## 1. DML 審計：Row Trigger + hstore 記錄欄位級變更

### 1.1 審計表結構

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

### 1.2 通用觸發器函數

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

### 1.3 掛載 Trigger 及測試

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

### 1.4 審計結果

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

### 1.5 DML Trigger 效能考量

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

### 2.1 審計表與 Event Trigger

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

### 2.2 Event Trigger 函數

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

### 2.3 測試

```sql
CREATE TABLE test (id INT);

ALTER TABLE test ALTER COLUMN id TYPE INT8;
```

### 2.4 審計結果

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

### 2.5 原文注意事項

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
