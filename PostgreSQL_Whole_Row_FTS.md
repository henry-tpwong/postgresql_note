# PostgreSQL 行級全文檢索 (Whole-Row Full-Text Search) — PG 17 視角

> 來源：[digoal - PostgreSQL 行級全文檢索 (2016-04-19)](https://github.com/digoal/blog/blob/master/201604/20160419_01.md)
>
> 本文基於原始文章，以 PG 17 (2026) 為基準更新：generated column 為首要推薦方案，IMMUTABLE hack 僅保留作為 legacy PG < 12 的參考。

---

## 場景：任意欄位全文檢索的痛點

當需要對 table 中所有 column 做全文檢索——例如前端搜尋框「在全部欄位中匹配關鍵字」——傳統寫法需要逐個 column 疊加 OR 條件，既繁瑣又難以維護：

```sql
-- 每個 column 都要寫一組條件，精準匹配 + 模糊匹配混雜
SELECT * FROM t
WHERE phonenum = 'digoal'
   OR info ~ 'digoal'
   OR c1::text = 'digoal'
   OR ...;  -- N 個 column 就要 N 個 OR
```

若 column 數量多、型別混雜（text / int / timestamp），寫 SQL 非常痛苦，且每個 OR 條件都會在 query planner 中產生額外的 filter clause。

解決方案：**將整個 row 轉成單一 text，對其建立 GIN full-text search index**，一條 `@@` 搞定所有 column 的全文檢索。

---

## 現代推薦方案：Generated Column + GIN (PG 12+)

自 PG 12 起，generated column 是最簡潔的方案，不需手動標記 IMMUTABLE，不需包裝 function：

```sql
ALTER TABLE t ADD COLUMN fts tsvector
    GENERATED ALWAYS AS (to_tsvector('jiebacfg', t::text)) STORED;

CREATE INDEX idx_t_fts ON t USING GIN (fts);
```

查詢直接使用該 column，`@@` operator 自動走 index：

```sql
SELECT * FROM t WHERE fts @@ to_tsquery('digoal & 阿里巴巴');
-- EXPLAIN 會顯示 Bitmap Index Scan on idx_t_fts
```

**優點：**
- expression 不需 IMMUTABLE，`to_tsvector` + `::text` 直接可用
- INSERT / UPDATE 時自動重算 `fts` 值，無額外 trigger
- 查詢 SQL 簡潔，不需呼叫包裝 function

> 補充（Senior Dev）：若 table 已存在且數據量大，建議用 background migration 逐步新增 column + index，避免長時間鎖表。`ALTER TABLE ... ADD COLUMN ... GENERATED ALWAYS AS ... STORED` 對既有 row 會觸發全表重寫（PG 17 起支援 incremental build，但首次仍需 full scan）。

---

## 原始實作步驟（PG < 12 或需完全手控場景）

以下是 2016 年的原始做法，步驟較多但對沒有 generated column 的舊版 PG 是唯一方案。

測試 table：

```sql
CREATE TABLE t (
    phonenum text,
    info     text,
    c1       int,
    c2       text,
    c3       text,
    c4       timestamp
);

INSERT INTO t VALUES (
    '13888888888',
    'i am digoal, a postgresqler',
    123,
    'china',
    '中华人民共和国，阿里巴巴，阿',
    now()
);
```

### Step 1：row 轉 text —— `t::text`

PostgreSQL 的 composite type 支援 `::text` cast，輸出格式為 `(col1_val, col2_val, col3_val, ...)`：

```sql
SELECT t::text FROM t;
-- (13888888888,"i am digoal, a postgresqler",123,china,中华人民共和国，阿里巴巴，阿,"2016-04-19 11:15:55.208658")
```

### Step 2：對 row text 做 tsvector 分詞

使用中文分詞 extension（此例為 jieba / pg_jieba）：

- [pg_jieba (GitHub)](https://github.com/jaiminpan/pg_jieba)
- [pg_scws (GitHub)](https://github.com/jaiminpan/pg_scws)

兩者均支援自訂辭典。

```sql
SELECT to_tsvector('jiebacfg', t::text) FROM t;
-- ' ':6,8,11,13,33 '04':30 '11':34 '123':17 '13888888888':2 ...
-- 'digoal':9 'postgresqler':14 'china':19 '中华人民共和国':21 '阿里巴巴':23 ...
```

整個 row 的所有 column 值被合併成一個 tsvector，各詞條標記了在整段 text 中的位置。

### Step 3：全文檢索查詢

```sql
-- digoal AND china（命中）
SELECT to_tsvector('jiebacfg', t::text) @@ to_tsquery('digoal & china') FROM t;
-- t

-- digoal AND post（不命中）
SELECT to_tsvector('jiebacfg', t::text) @@ to_tsquery('digoal & post') FROM t;
-- f
```

實際查詢：

```sql
SELECT * FROM t
WHERE to_tsvector('jiebacfg', t::text) @@ to_tsquery('digoal & china');
--  phonenum   |            info             | c1  |  c2   |              c3              |             c4
-- -------------+-----------------------------+-----+-------+------------------------------+----------------------------
--  13888888888 | i am digoal, a postgresqler | 123 | china | 中华人民共和国，阿里巴巴，阿 | 2016-04-19 11:15:55.208658

SELECT * FROM t
WHERE to_tsvector('jiebacfg', t::text) @@ to_tsquery('digoal & 阿里巴巴');
-- 同上，命中
```

---

## 建立 GIN Index：繞過 IMMUTABLE 限制（Legacy：PG < 12）

直接對 `to_tsvector('jiebacfg', t::text)` 建立 index 會失敗，因為 `t::text` 背後依賴 `record_out()` 和 `textin()` 兩個 function，它們預設是 **STABLE**（非 IMMUTABLE），而 expression index 要求 expression 必須是 IMMUTABLE。

### Step 1：包裝 IMMUTABLE function

```sql
CREATE OR REPLACE FUNCTION f1(regconfig, text) RETURNS tsvector AS $$
  SELECT to_tsvector($1, $2);
$$ LANGUAGE sql IMMUTABLE STRICT;

CREATE OR REPLACE FUNCTION f1(text) RETURNS tsvector AS $$
  SELECT to_tsvector($1);
$$ LANGUAGE sql IMMUTABLE STRICT;
```

### Step 2：將依賴的 system function 標記為 IMMUTABLE

```sql
ALTER FUNCTION record_out(record) IMMUTABLE;
ALTER FUNCTION textin(cstring) IMMUTABLE;
```

> 補充（Senior Dev）：`record_out` 和 `textin` 預設為 STABLE 是有原因的——某些 locale / encoding 設定下輸出行為可能因 session 環境而異。如果確認你的 DB 環境一致（固定 collation、encoding），標記為 IMMUTABLE 是安全的。但注意：若後續改動這些設定，需重建 index。PG 12+ 可改用 generated column 避開此問題（見末尾補充）。

### Step 3：建立 GIN index

```sql
CREATE INDEX idx_t_1 ON t USING GIN (f1('jiebacfg'::regconfig, t::text));
```

### Step 4：驗證 index 生效

```sql
-- 少量 row 時 PG 可能選擇 Seq Scan，可強制關閉確認 index 被使用：
SET enable_seqscan = off;

EXPLAIN SELECT * FROM t
WHERE f1('jiebacfg'::regconfig, t::text) @@ to_tsquery('digoal & 阿里巴巴');

--                                                    QUERY PLAN
-- ----------------------------------------------------------------------------------------------------------------
--  Bitmap Heap Scan on t  (cost=12.25..16.77 rows=1 width=140)
--    Recheck Cond: (to_tsvector('jiebacfg'::regconfig, (t.*)::text) @@ to_tsquery('digoal & 阿里巴巴'::text))
--    ->  Bitmap Index Scan on idx_t_1  (cost=0.00..12.25 rows=1 width=0)
--          Index Cond: (to_tsvector('jiebacfg'::regconfig, (t.*)::text) @@ to_tsquery('digoal & 阿里巴巴'::text))
```

```sql
-- 恢復預設行為
SET enable_seqscan = on;
```

> 補充（Senior Dev）：`SET enable_seqscan = off` 僅用於測試 small table 的 index 行為。Production 中若 row 數正常，planner 會自然選擇 index。不要永久關閉 seqscan。

---

## Senior Dev 實戰注意事項（PG 17 更新）

### 1. `t::text` 格式的局限

`t::text` 輸出格式為 `(val1, val2, ...)`，其中 text column 的值以雙引號包裹、逗號分隔。這意味著：
- 逗號本身會被當作 token 分隔符，不影響分詞
- 但若 column 值內含特殊字符（如括號、引號），可能產生意外 token。建議先對 sample data 用 `to_tsvector()` 檢查 token 列表

> 補充（Senior Dev）：PG 14+ 中 `(table_alias.*)::text` 在某些 edge case 下的 escaping 行為有微調，建議升級後重新驗證 token 輸出。

### 2. 不能做加權檢索（Weighted Search）

row-level tsvector 將所有 column 合為一體，無法區分「title 匹配權重 A、body 匹配權重 B」。如需加權，應使用 `setweight()` 對各 column 分別指定權重再合併：

```sql
SELECT setweight(to_tsvector('english', coalesce(title, '')), 'A') ||
       setweight(to_tsvector('english', coalesce(body, '')),  'B');
```

### 3. UPDATE 成本

任何 column 的 UPDATE 都會觸發整行 tsvector 重建 + GIN index 更新。對高寫入頻率的 table 需評估 overhead。使用 generated column 時同樣會觸發重算。

> 補充（Senior Dev）：GIN index 的 `fastupdate` 參數（預設 on）可將多個 pending insert 合併寫入，減少 UPDATE-heavy workload 的 index 維護成本。但 pending list 過大會增加查詢時的 scan overhead，可透過 `gin_pending_list_limit` 調整閾值（PG 9.5+，預設 4MB）。

### 4. PG 14~17 新特性

| 版本 | 相關改進 |
|------|---------|
| PG 14 | `REINDEX` 支援 `TABLESPACE` 變更；GIN index 的 `gin_clean_pending_list` 效能提升 |
| PG 15 | `jsonb_to_tsvector` 可以直接從 JSONB column 提取 text 做 FTS（不需先 `::text`） |
| PG 16 | `tsvector` 的 `||` operator 效能優化；支援 `ts_headline` 的 `MaxWords` / `MinWords` options |
| PG 17 | GIN index 支援 incremental build（`CREATE INDEX ... WITH (fastupdate = on)` 可用於 concurrent build） |

### 5. 中文分詞 extension 選擇（2026 現狀）

| Extension | 引擎 | 適合場景 | 維護狀態 (2026) |
|-----------|------|---------|----------------|
| **zhparser** | SCWS | 簡體中文 | 社群最活躍，Alibaba RDS PG 內建支援 |
| pg_jieba | jieba（結巴） | 簡體中文 | 原文章示例，更新較慢 |
| pg_scws | SCWS | 簡體中文 | 詞典較豐富 |
| pg_bigm | 2-gram | 中日韓混合文本 | 無需辭典，適合多語言混合場景 |
| pgroonga | Groonga | 中日韓全文檢索 | 效能極佳，支援 N-gram + tokenizer |
| 內建 default | snowball stemmer | 英文 / 歐洲語言 | PG 原生 |

> 補充（Senior Dev）：2026 年中文 PG 場景中 **zhparser** 是事實標準，Alibaba Cloud RDS PG 直接內建`zhparser` extension，`pgroonga` 則適合需要高效 CJK 全文檢索 + JSON 混用的 OLTP 場景。如果不需要精確分詞且追求零配置，`pg_bigm` 的 2-gram 是開箱即選。
