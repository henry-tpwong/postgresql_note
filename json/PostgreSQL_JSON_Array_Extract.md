# PostgreSQL JSON/JSONB 陣列提取與查詢

> 來源：[digoal - 如何从PostgreSQL json中提取数组 (2016-09-10)](https://github.com/digoal/blog/blob/master/201609/20160910_01.md)
>
> 更新於 2026-05-17，補充 JSONPath / SQL/JSON 標準函數 / json_table 演進

---

## JSON Value Type 與提取基礎

### json_typeof / jsonb_typeof

```sql
SELECT jsonb_typeof(c1), c1 FROM t3;
```

支援 type：`object`, `array`, `string`, `number`, `boolean`, `null`。

### -> 與 ->> 操作符

| 操作符 | 返回 type | 範例 |
|--------|----------|------|
| `->'key'` | json/jsonb（保留原始類型） | `c1->'f'` → `[1,2,3,4]` (jsonb) |
| `->>'key'` | text | `c1->>'f'` → `[1,2,3,4]` (text) |
| `->N` | json/jsonb（陣列索引） | `c1->'f'->0` → `1` |

完整演示（jsonb）：

```sql
CREATE TABLE t3 (c1 JSONB);
INSERT INTO t3 VALUES ('{
  "a":"v", "b":12, "c":{"ab":"hello"}, "d":12.3,
  "e":true, "f":[1,2,3,4], "g":["a","b"]
}');

SELECT pg_typeof(col), jsonb_typeof(col), col
FROM (SELECT c1->'a' col FROM t3) t;
--  pg_typeof | jsonb_typeof | col
-- -----------+--------------+-----
--  jsonb     | string       | "v"

SELECT pg_typeof(col), jsonb_typeof(col), col
FROM (SELECT c1->'f' col FROM t3) t;
--  pg_typeof | jsonb_typeof | col
-- -----------+--------------+--------------
--  jsonb     | array        | [1,2,3,4]
```

關鍵認知：`->` 返回的仍是 json/jsonb type（不是 native SQL type），即使 `jsonb_typeof` 判斷為 array，PG 仍視為 jsonb。

---

## 2016 年方案：json_array_elements + ARRAY 構造器

### Step 1：JSON array → row set

```sql
SELECT jsonb_array_elements_text('{"a":"B","b":[1,2,3,4,5,6]}'::jsonb->'b');
--  col
-- -----
--  1
--  2
--  3
--  4
--  5
--  6
-- pg_typeof: text
```

`jsonb_array_elements_text()` 將 JSON array 展開為 text row set。對應 json 版為 `json_array_elements_text()`。

### Step 2：row set → SQL array

```sql
SELECT ARRAY(
  SELECT jsonb_array_elements_text('{"a":"B","b":[1,2,3,4,5,6]}'::jsonb->'b')
);
--  {1,2,3,4,5,6}
-- pg_typeof: text[]

SELECT ARRAY(
  SELECT (jsonb_array_elements_text('{"a":"B","b":[1,2,3,4,5,6]}'::jsonb->'b'))::INT
);
--  {1,2,3,4,5,6}
-- pg_typeof: integer[]
```

### Step 3：封裝為可重複使用的 Helper Function

```sql
-- JSONB → text[]
CREATE OR REPLACE FUNCTION json_arr2text_arr(_js JSONB)
  RETURNS TEXT[] AS $$
  SELECT ARRAY(SELECT jsonb_array_elements_text(_js))
$$ LANGUAGE SQL IMMUTABLE;

-- JSON → text[]
CREATE OR REPLACE FUNCTION json_arr2text_arr(_js JSON)
  RETURNS TEXT[] AS $$
  SELECT ARRAY(SELECT json_array_elements_text(_js))
$$ LANGUAGE SQL IMMUTABLE;

-- JSONB → int[]
CREATE OR REPLACE FUNCTION json_arr2int_arr(_js JSONB)
  RETURNS INT[] AS $$
  SELECT ARRAY(SELECT (jsonb_array_elements_text(_js))::INT)
$$ LANGUAGE SQL IMMUTABLE;
```

使用：

```sql
SELECT json_arr2text_arr(c1->'g') FROM t3;  -- {a,b}
SELECT json_arr2int_arr(c1->'f') FROM t3;   -- {1,2,3,4}
```

---

## 陣列查詢：@> / && 等操作符

一旦 JSON array 轉為 native SQL array，即可使用 PostgreSQL 的陣列操作符：

```sql
-- @>  全包含（contains）
SELECT * FROM t3 WHERE json_arr2text_arr(c1->'g') @> ARRAY['a'];        -- 包含 a
SELECT * FROM t3 WHERE json_arr2int_arr(c1->'f') @> ARRAY[1,2];         -- 同時包含 1,2

-- &&  相交（overlaps）
SELECT * FROM t3 WHERE json_arr2int_arr(c1->'f') && ARRAY[1,6];         -- 包含 1 或 6

-- NOT &&  不相交
SELECT * FROM t3 WHERE NOT(json_arr2int_arr(c1->'f') && ARRAY[6]);      -- 不包含 6

-- 注意 type 一致：text 陣列 vs text value / int 陣列 vs int value
SELECT * FROM t3 WHERE NOT(json_arr2text_arr(c1->'f') && ARRAY['6']);   -- 對 text[] 用 '6'
```

---

## Index：GIN on Expression

```sql
-- 建立 function-based GIN index（將 JSON array 轉換結果索引化）
CREATE INDEX idx_t3_1 ON t3 USING GIN (json_arr2text_arr(c1->'f'));

SET enable_seqscan = off;

EXPLAIN SELECT * FROM t3 WHERE json_arr2text_arr(c1->'f') && ARRAY['1','6'];
--  Bitmap Heap Scan on t3  (cost=12.25..16.52 rows=1 width=32)
--    Recheck Cond: (json_arr2text_arr((c1 -> 'f'::text)) && '{1,6}'::text[])
--    ->  Bitmap Index Scan on idx_t3_1

EXPLAIN SELECT * FROM t3 WHERE json_arr2text_arr(c1->'f') @> ARRAY['1','6'];
--  Bitmap Heap Scan on t3
--    Recheck Cond: (json_arr2text_arr((c1 -> 'f'::text)) @> '{1,6}'::text[])
--    ->  Bitmap Index Scan on idx_t3_1
```

> 補充（Senior Dev）：function-based GIN index 要求 function 標記為 `IMMUTABLE`。若 function 不是 IMMUTABLE（如依賴 GUC），index 將無法建立。上面 `json_arr2text_arr` 確實是 IMMUTABLE（純 JSONB → text[] 轉換，無外部依賴）。

---

## 現代化方案：JSONPath（PG 12+）

2016 年的方案仍有效，但 PG 12 引入的 JSONPath 提供更強大的表達能力：

```sql
-- 查詢條件：JSON array 中包含數字 1
SELECT * FROM t3
WHERE c1 @? '$.f[*] ? (@ == 1)';

-- 提取所有匹配的元素
SELECT * FROM jsonb_path_query(
  c1,
  '$.f[*] ? (@ > 2)'
) FROM t3;

-- 返回原生 SQL 型別（自動處理）
SELECT * FROM t3
WHERE c1 @? '$.b ? (@ > 10)';
```

> 補充（Senior Dev）：JSONPath 的 query 在 GIN index 上的加速方式有別於手動轉換：
> - `c1 @? '$.f[*] ? (@ == 1)'` 可以使用 default `jsonb_ops` GIN index（因為 `@?` 內部轉為 `jsonb_path_match`）
> - 但 JSONPath 的 `@?` 不會自動利用「提取 array → text[] → GIN index」的路徑，需手動建立。對於大量陣列包含查詢，2016 年的 function-based GIN 方案可能仍有性能優勢。

---

## SQL/JSON 標準函數（PG 15+）

PG 15 引入 ISO SQL/JSON 標準函數，替代部分手動處理：

```sql
-- json_exists — 判斷值是否存在
SELECT * FROM t3 WHERE JSON_EXISTS(c1, '$.f[*] ? (@ == 1)');

-- json_value — 提取單一 scalar value（自動轉型）
SELECT JSON_VALUE(c1, '$.b' RETURNING INT) FROM t3;  -- → 12 (int)

-- json_query — 提取整個 object/array
SELECT JSON_QUERY(c1, '$.f') FROM t3;  -- → [1,2,3,4] (jsonb)
```

> 補充（Senior Dev）：`json_value` 解決了 2016 年的核心痛點——`->` 返回 jsonb 而非 native type。現在 `json_value(c1->'b' RETURNING INT)` 直接拿到 PostgreSQL integer，不需要額外 `::INT` cast。

---

## json_table（PG 17+）

PG 17 引入 `json_table`（又是 ISO SQL/JSON 標準），直接在 FROM 子句中將 JSON 陣列展開為 structured column：

```sql
SELECT jt.*
FROM t3,
     JSON_TABLE(
       c1, '$.f[*]'
       COLUMNS (
         val INT PATH '$'
       )
     ) AS jt;

-- 支持多層嵌套、多 column、type coercion
SELECT jt.id, jt.name, jt.price
FROM orders,
     JSON_TABLE(
       doc, '$.items[*]'
       COLUMNS (
         id    INT     PATH '$.product_id',
         name  TEXT    PATH '$.product_name',
         price NUMERIC PATH '$.price'
       )
     ) AS jt
WHERE jt.price < 100;
```

> 補充（Senior Dev）：`json_table` 將 JSON 陣列的 row-expansion 從「select-list subquery + array constructor」的兩步操作變成單一步驟，plan execution 更高效（無需 intermediate array materialization）。對大量 JSON 集的 OLAP 場景是 game changer。

---

## 三種方案的選擇矩陣

| 方案 | 適用 PG 版本 | 適用場景 | Trade-off |
|------|-------------|---------|-----------|
| `json_array_elements` + `ARRAY()` | 9.3+ | 需要為陣列建立 GIN index 進行 `@>` / `&&` 查詢 | 需 self-defined function，維護成本 |
| JSONPath `@?` / `*.?()` | 12+ | 條件查詢（不需要轉 SQL array） | 對 `@>` 語義支援弱於 native array operators |
| SQL/JSON (`json_value` / `json_table`) | 15+ / 17+ | 結構化提取、返回 native SQL type | 最新方案，生態文檔較少 |

---

## 版本演進

| 功能 | 版本 | 說明 |
|------|------|------|
| `json_array_elements_text` | PG 9.3 | JSON array → row set |
| `jsonb` type + GIN index | PG 9.4 | 二進制 JSON + 原生 GIN 加速 |
| `jsonb_path_query` / `@?` / `@@` | PG 12 | JSONPath 表達式 |
| SQL/JSON `json_exists` / `json_value` / `json_query` | PG 15 | ISO 標準函數 |
| SQL/JSON `json_table` | PG 17 | ISO 標準 inline table function |

## 參考

- [How to turn JSON array into Postgres array (DBA.SE)](http://dba.stackexchange.com/questions/54283/how-to-turn-json-array-into-postgres-array)
- [PostgreSQL JSON Functions](https://www.postgresql.org/docs/9.6/static/functions-json.html)
- [PostgreSQL Array Functions](https://www.postgresql.org/docs/9.6/static/functions-array.html)
