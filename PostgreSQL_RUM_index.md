# PostgreSQL RUM Index — 全文檢索的非 Bitmap Index Scan 與相似度排序

> 來源：[digoal - PostgreSQL 全文檢索加速快到沒有朋友 — RUM索引接口 (2016-10-19)](https://github.com/digoal/blog/blob/master/201610/20161019_01.md)
>
> 官方：[Postgres Professional — RUM GitHub](https://github.com/postgrespro/rum)

---

GIN 是 PostgreSQL 全文檢索的主力 index type，但它有兩個瓶頸：
1. 查詢使用 Bitmap Heap Scan，結果集需要 SORT（尤其是 `ORDER BY` + `OFFSET` 場景）
2. 不支援文本相似度排序 (`ORDER BY relevance`)

RUM（**R**UM access method，比 GIN 多存 **U**niversal **M**etadata）解決了這兩個問題。

---

## 1. 問題場景：逗號分隔字串的全文檢索反模式

常見錯誤用法：逗號分隔元素存為單一 `text` column，再用 `LIKE '%1%' OR LIKE '%502%'` 搜尋。

```sql
CREATE TABLE test (c1 TEXT);
INSERT INTO test VALUES ('1,100,2331,344,502,...');
SELECT * FROM test WHERE c1 LIKE '%1%' OR c1 LIKE '%502%' AND c1 LIKE '%2331%';
-- 全表掃描，無法用 index
```

### 正確做法 1：Array + GIN

```sql
CREATE TABLE arr_test (c1 INT[]);
CREATE INDEX idx_arr_test ON arr_test USING GIN (c1);

-- 查詢
SELECT * FROM arr_test WHERE c1 && ARRAY[1, 2];
```

### 正確做法 2：tsvector + GIN

```sql
CREATE TABLE gin_test (c1 TSVECTOR);
CREATE INDEX idx_gin_test ON gin_test USING GIN (c1);

-- 查詢
SELECT * FROM gin_test WHERE c1 @@ to_tsquery('english', '1 | 2');
```

### 正確做法 3：tsvector + RUM

```sql
CREATE TABLE rum_test (c1 TSVECTOR);
CREATE INDEX rumidx ON rum_test USING rum (c1 rum_tsvector_ops);

-- 查詢
SELECT * FROM rum_test WHERE c1 @@ to_tsquery('english', '1 | 2');
```

---

## 2. 效能基準（10M row / 每 row 100 element = 1B total token）

測試數據：每 row 含 100 個 0-100000 範圍的隨機數字。查詢搜尋 token `1` 或 `2`（命中約 19,900 row / 10M）。

```sql
INSERT INTO rum_test
SELECT to_tsvector(string_agg(c1::text, ','))
FROM (SELECT (100000 * random())::int FROM generate_series(1,100)) t(c1);

SET maintenance_work_mem = '64GB';
CREATE INDEX rumidx ON rum_test USING rum (c1 rum_tsvector_ops);
```

### 2.1 無 ORDER BY 查詢

| Index | Scan Method | Execution Time |
|-------|------------|----------------|
| **RUM** | Index Scan | **26.1 ms** |
| GIN (tsvector) | Bitmap Heap Scan | 35.3 ms |
| GIN (int[]) | Bitmap Heap Scan | 32.8 ms |

無 ORDER BY 時，RUM vs GIN 差距約 1.2-1.35x。

```sql
-- RUM：Index Scan，無 Bitmap 轉換
EXPLAIN ANALYZE SELECT * FROM rum_test
WHERE c1 @@ to_tsquery('english', '1 | 2');

-- Index Scan using rumidx on rum_test
--   Index Cond: (c1 @@ '''1'' | ''2'''::tsquery)
-- Execution time: 26.086 ms

-- GIN：Bitmap Heap Scan
EXPLAIN ANALYZE SELECT * FROM gin_test
WHERE c1 @@ to_tsquery('english', '1 | 2');

-- Bitmap Heap Scan on gin_test
--   Recheck Cond: (c1 @@ '''1'' | ''2'''::tsquery)
--   Heap Blocks: exact=19764
--   ->  Bitmap Index Scan on idx_gin_test
-- Execution time: 35.279 ms
```

### 2.2 有 ORDER BY + OFFSET 查詢（關鍵差異）

```sql
SELECT * FROM ... WHERE ... ORDER BY c1 OFFSET 19000 LIMIT 100;
```

| Index | Scan Method | Sort | Execution Time |
|-------|------------|------|----------------|
| **RUM** | Index Scan | **None** (Index 已排序) | **59.2 ms** |
| GIN (tsvector) | Bitmap Heap Scan | **external merge Disk: 26,968 kB** | 126.1 ms |
| GIN (int[]) | Bitmap Heap Scan | **external merge Disk: 8,440 kB** | 93.1 ms |

**RUM 的核心優勢在這裡**。GIN 的 Bitmap Heap Scan 返回的是未排序的 row → 必須在 memory 中 SORT → 超過 `work_mem` 時寫入 disk（external merge）。RUM 直接走 Index Scan，index entry 已按 `tsvector` 順序排列，無需額外 SORT。

```sql
-- RUM：Index Scan 自帶排序，無 Sort 節點
EXPLAIN ANALYZE SELECT * FROM rum_test
WHERE c1 @@ to_tsquery('english', '1 | 2')
ORDER BY c1 <=> to_tsquery('english', '1 | 2') OFFSET 19000 LIMIT 100;

-- Index Scan using rumidx on rum_test
--   Index Cond: (c1 @@ '''1'' | ''2'''::tsquery)
--   Order By: (c1 <=> '''1'' | ''2'''::tsquery)
-- Execution time: 59.220 ms

-- GIN：Bitmap → Sort (external) → Limit
-- Sort Method: external merge  Disk: 26968kB
-- Execution time: 126.122 ms
```

> 補充（Senior Dev）：RUM 能走 Index Scan 而非 Bitmap Scan 的根本原因：RUM index 的 leaf page 中每個 entry 除了存 TID 之外，還存了 **附加的排序 metadata**（如 `tsvector` 的位置權重資訊）。這讓 RUM 能夠在 index traversal 階段就決定 row 的訪問順序，不需要 Bitmap 的中間層來做物理排序。代價是 RUM index 比 GIN 大約 **2-3 倍**（因為需要存 metadata），且寫入成本更高。

---

## 3. RUM 的相似度排序（`<=>` operator）

RUM 獨有功能：文本相似度排序。`<=>` (distance operator) 返回 query 與 document 的 distance score，越小越相關。

```sql
-- 分詞結果
SELECT * FROM to_tsvector('jiebacfg',
  '小明硕士毕业于中国科学院计算所，后在日本京都大学深造');
-- '中国科学院':5 '小明':1 '日本京都大学':10 '毕业':3 '深造':11 '硕士':2 '计算所':6

-- OR 相似度：匹配越多 token → distance 越小 → 越相關
SELECT rum_ts_distance(
  to_tsvector('jiebacfg', '小明硕士毕业于中国科学院计算所，后在日本京都大学深造'),
  to_tsquery('计算所 | 硕士')
);
-- 8.22467

-- AND 相似度：強制兩個 token 都存在 → distance 更大（更嚴格）
SELECT rum_ts_distance(
  to_tsvector('jiebacfg', '小明硕士毕业于中国科学院计算所，后在日本京都大学深造'),
  to_tsquery('计算所 & 硕士')
);
-- 32.8987

-- 不相關 → Infinity
SELECT rum_ts_distance(
  to_tsvector('jiebacfg', '小明硕士毕业于中国科学院计算所，后在日本京都大学深造'),
  to_tsquery('计算')
);
-- Infinity
```

### 相似度排序查詢（RUM 的核心 killer feature）

```sql
CREATE TABLE test15 (c1 TSVECTOR);
INSERT INTO test15 VALUES
  (to_tsvector('jiebacfg', 'hello china, i''m digoal')),
  (to_tsvector('jiebacfg', 'hello world, i''m postgresql')),
  (to_tsvector('jiebacfg', 'how are you, i''m digoal'));

CREATE INDEX idx_test15 ON test15 USING rum (c1 rum_tsvector_ops);

-- 按相似度排序（直接 Index Scan + Order By，無 Sort 節點）
EXPLAIN SELECT *, c1 <=> to_tsquery('postgresql') FROM test15
ORDER BY c1 <=> to_tsquery('postgresql');

-- Index Scan using idx_test15 on test15
--   Order By: (c1 <=> to_tsquery('postgresql'::text))
```

> 補充（Senior Dev）：
>
> **RUM vs GIN + `ts_rank()` 的相似度排序差異**：
> - `ts_rank()` 是 PostgreSQL 內建相似度函數，GIN index 無法加速它——查詢必須對所有匹配 row 計算 `ts_rank()` 再 SORT
> - RUM 的 `<=>` 是 index-aware 的距離運算，計算結果內嵌在 index entry 中，Index Scan 階段就能按距離排序 output row
> - 實際效果：`ORDER BY ts_rank() LIMIT 10` 在 GIN 上需讀取全部匹配 row → 計算 rank → sort → limit；RUM 上只需讀取距離最小的前 10 row（top-N stop key optimization）
>
> **RUM index 的寫入成本**：
> - RUM 的 pending list（類似 GIN fastupdate）預設也是啟用的
> - 每次 INSERT/UPDATE 需更新 index entry 中的 metadata（position info），寫入成本約為 GIN 的 **1.5-3x**（視 column 數量和 token 密度）
> - 高吞吐 append-only 場景（如 log 搜尋）建議：普通 GIN for 寫入 + RUM for 定期重建後的查詢（或在 standby 上使用 RUM）
>
> **PG 版本演進**：
> - PG 9.6：RUM 作為外部 extension 首次可用
> - PG 13+：GIN 有部分改進（更高效的 pending list cleanup），但 core 仍不支援相似度索引排序
> - RUM 至今仍是外部 extension，由 Postgres Professional 維護，未合入 PG core

---

## 4. 方案選擇矩陣

| 場景 | 推薦方案 | 理由 |
|------|---------|------|
| 純 filter（無 ORDER BY） | GIN | 寫入成本低，查詢效能僅略慢於 RUM |
| `ORDER BY relevance LIMIT N` | **RUM** | Index Scan + 相似度排序，無 SORT |
| `ORDER BY column OFFSET N LIMIT M` | **RUM** | 無 external sort，深頁仍快 |
| 高吞吐 append-only 寫入 | GIN（寫入端）+ RUM（查詢端或 standby） | 平衡寫入與查詢 |
| 全文搜尋引擎替代 | RUM + 分詞插件（jieba / zhparser） | PG native solution，無數據同步延遲 |
| 需要複雜 ranking model（BM25 / TF-IDF 自訂） | 外部搜索引擎（ES / Solr） | RUM 的 ranking 較簡單，不及專業搜索引擎 |

> 補充（Senior Dev）：
> - `rum_tsvector_ops` vs `rum_tsvector_addon_ops`：前者存 position + weight 資訊，支援 `<=>` 距離查詢；後者是輕量版，不存 metadata，體積更小。若不需相似度排序，用 `addon_ops` 減少 index 體積
> - RUM 在 PostgreSQL 雲端 RDS 的可用性取決於廠商——阿里雲 RDS PG 支援 RUM extension，AWS RDS 不支援（需用 Aurora PG 或自建）

---

## 參考

1. [RUM GitHub — Postgres Professional](https://github.com/postgrespro/rum)
2. [德哥：RUM 索引測試詳細](https://yq.aliyun.com/articles/59212)
3. [pg_jieba](https://github.com/jaiminpan/pg_jieba) / [zhparser](https://github.com/amutu/zhparser)
