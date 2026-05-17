# PostgreSQL GIN Index + LIMIT 慢的原因與因應策略

> 來源：[digoal - PostgreSQL GIN索引limit慢的原因分析 (2016-05-07)](https://github.com/digoal/blog/blob/master/201605/20160507_02.md)
>
> 更新於 2026-05-17，補充 PG 10~18 GIN 演進與替代方案

---

## GIN Index 結構回顧

GIN（Generalized Inverted Index）是倒排索引：每個 element（陣列元素 / tsvector lexeme / JSONB key）對應一組 row number（TID list），而非 B-tree 那樣一行一個 index entry。

![GIN Index Structure](images/gin_structure.jpeg)

對高頻 element（如陣列中反覆出現的 `1`），TID 列表會被組織成 posting list（而非一個一個 pointer），再由 index 指向該 list。

![GIN Element → Row List](images/gin_element_list.jpeg)

---

## 核心問題：為什麼 GIN + LIMIT 慢？

GIN Index **只支援 Bitmap Index Scan**（不支援 plain Index Scan）。Bitmap Index Scan 的流程：

```
BitMap Index Scan → 收集所有匹配 TID → 排序 TID（按 page 順序）
  → BitMap Heap Scan → 依序訪問對應 data page → Recheck → 返回 row
```

**關鍵**：排序所有匹配 TID 是無法跳過的步驟。即使你寫了 `LIMIT 1`，GIN Index Scan 仍然會收集**所有**匹配該條件的 TID，全部排序後才去 Heap 取資料。

### 實驗

```sql
CREATE TABLE t3 (id INT, info INT[]);
INSERT INTO t3 SELECT generate_series(1, 10000), ARRAY[1,2,3,4,5];
CREATE INDEX idx_t3_info ON t3 USING GIN (info);
SET enable_seqscan = off;
```

**無 LIMIT — 10,000 row 全部匹配：**

```sql
EXPLAIN ANALYZE SELECT * FROM t3 WHERE info && ARRAY[1];

-- Bitmap Heap Scan on t3  (actual time=1.156..3.565 rows=10000 loops=1)
--   Recheck Cond: (info && '{1}'::integer[])
--   Heap Blocks: exact=94
--   ->  Bitmap Index Scan on idx_t3_info
--         (actual time=1.129..1.129 rows=10000 loops=1)
--         Index Cond: (info && '{1}'::integer[])
-- Execution time: 5.272 ms
```

**LIMIT 1 — 時間沒省多少：**

```sql
EXPLAIN ANALYZE SELECT * FROM t3 WHERE info && ARRAY[1] LIMIT 1;

-- Limit  (actual time=1.121..1.121 rows=1 loops=1)
--   ->  Bitmap Heap Scan on t3  (actual time=1.119..1.119 rows=1 loops=1)
--         Heap Blocks: exact=1
--         ->  Bitmap Index Scan on idx_t3_info
--               (actual time=1.095..1.095 rows=10000 loops=1)
--               Index Cond: (info && '{1}'::integer[])
-- Execution time: 1.175 ms
```

`LIMIT 1` 只省了 Heap Scan（Heap Blocks 94 → 1），但 **Bitmap Index Scan 仍然掃了全部 10,000 row 的 TID 並排序**（`rows=10000 loops=1` 沒變）。節省的比例很低：5.2ms → 1.2ms，遠不如 B-tree 的 `LIMIT 1` 能從百毫秒降到 0.03ms。

---

## 為什麼 GIN 只支援 Bitmap Scan — 設計取捨

GIN 的場景本質是**多對多**：一個 element 對應多行多 page，一個查詢可能匹配成千上萬個 page。若不用 Bitmap Scan 逐行跳轉，會產生恐怖的 random I/O。

Bitmap Scan 收集所有 TID → 排序的設計，反而是為了：

1. **Page deduplication**：同一 page 內有多行匹配 → 只讀一次，不重複讀取
2. **Sequential page reading**：TID 按 page 排序後順序讀取 → 最大化 sequential I/O

但當 `LIMIT` 很小（如 `LIMIT 1` 或 `LIMIT 10`）且 matched row 很多時，Bitmap Scan 的排序成本成為主導，而非 I/O 節省。

> 補充（Senior Dev）：這是一個經典的 **optimizer blind spot**：GIN 的 cost model 假設「matched row 不多」或「最昂貴的是 I/O 而非排序」。當 matched row 極多但只取少量時（如 `&& ARRAY[1] LIMIT 1`，但 `1` 出現在 90% 的 row 中），排序成本遠超實際需要。PG 至今（18）未引入 GIN Index Scan，社群討論過但實作複雜度極高（需保證返回順序與 Bitmap dedup 的正確性）。

---

## btree_gin：多列 GIN 複合索引

`btree_gin` extension 讓標準 scalar type（INT / TEXT / TIMESTAMP 等）支援 GIN，可用來建複合 GIN index。適合「單列選擇性差，但多列組合選擇性好」的場景：

```sql
CREATE EXTENSION btree_gin;

CREATE TABLE t4 (id INT, info INT[]);
INSERT INTO t4
SELECT trunc(random() * 1000),
       ARRAY_APPEND(ARRAY[1,2,3], trunc(random() * 1000)::INT)
FROM generate_series(1, 100000);

CREATE INDEX idx_t4 ON t4 USING GIN (id, info);

EXPLAIN (ANALYZE, VERBOSE, COSTS, TIMING, BUFFERS)
SELECT * FROM t4 WHERE id = 10 AND info && ARRAY[1,2,3];

-- Bitmap Heap Scan on public.t4  (actual time=1.572..1.737 rows=97 loops=1)
--   Recheck Cond: ((t4.id = 10) AND (t4.info && '{1,2,3}'::integer[]))
--   Heap Blocks: exact=92
--   Buffers: shared hit=179
--   ->  Bitmap Index Scan on idx_t4  (actual time=1.554..1.554 rows=97 loops=1)
--         Index Cond: ((t4.id = 10) AND (t4.info && '{1,2,3}'::integer[]))
--         Buffers: shared hit=87
```

此例中 `id = 10` 將結果縮小到 97 row，Bitmap Index Scan 的 TID 排序量可控（97 個 TID vs 若只用 `info && ...` 可能上萬），效能良好。

> 補充（Senior Dev）：btree_gin 複合索引的 trade-off：
> - 若 `id = 10` 已能縮小結果到**幾十 row**，GIN Bitmap Scan overhead 可忽略
> - 若 `id = 10` 無法有效過濾（id 基數太低），GIN 仍會收集大量 TID 並排序
> - 若查詢有 `ORDER BY id LIMIT N` 且 id 在 GIN index 第一列，PG 16+ 可部分優化（先按 id 過濾 TID，減少排序量）
> - 一般建議：如果 B-tree 已經能有效縮小範圍，優先使用 B-tree，GIN 的場景是「必須用 array / JSONB / tsvector 條件」，不是「替代 B-tree」

---

## 解法矩陣

### 1. gin_fuzzy_search_limit — 隨機採樣軟上限

```sql
SET gin_fuzzy_search_limit = 5000;

SELECT * FROM t3 WHERE info && ARRAY[1] LIMIT 10;
```

| 行為 | 說明 |
|------|------|
| 預設 | `0`（無限制） |
| 設定非 0 值 | GIN Index Scan 只掃描前 N 個隨機 TID，不回表取全部 |
| 結果特性 | 隨機子集（非精確排序、非全部結果） |
| 適用場景 | primary goal 是**快速返回「一些」結果**，不需全部也不需精確排序（如搜尋建議、無限滾動 feed） |
| 來源 | [PostgreSQL GIN tips](https://www.postgresql.org/docs/devel/static/gin-tips.html) |

從經驗，5000–20000 效果最好。注意：結果數量可能略偏離設定值（依賴 random generator 品質）。

### 2. 改用 B-tree（若條件不依賴陣列）

```sql
-- 若 info 能標準化為單值 → 用 B-tree
CREATE INDEX idx_t3_info_btree ON t3 (info_element);
SELECT * FROM t3 WHERE info_element = 1 LIMIT 1;  -- Index Scan, 極快
```

### 3. 改用 GiST（某些場景）

GiST 索引支援 plain Index Scan（非只有 Bitmap Scan）。對陣列、全文檢索、幾何等場景可選 GiST。但 GiST 體積比 GIN 大，寫入比 GIN 慢。取捨：

| 維度 | GIN | GiST |
|------|-----|------|
| Index Scan | ❌（僅 Bitmap） | ✅ |
| 寫入速度 | 快（pending list 機制） | 慢 |
| 查詢速度（大量匹配） | 快（Bitmap dedup） | 較慢 |
| Index 體積 | 小 | 大 |
| LIMIT 小結果 | 慢 | 快（可走 Index Scan） |

### 4. 改寫查詢（限制輸入規模）

```sql
-- 原寫法（BAD）：10,000 row 全排序
SELECT * FROM t3 WHERE info && ARRAY[1] LIMIT 1;

-- 改寫：先限制到最小可能的 pool
SELECT * FROM t3
WHERE id IN (SELECT unnest(info) FROM ...)  -- 先縮範圍
  AND info && ARRAY[1]
LIMIT 1;
```

### 5. ORDER BY 配合 GIN Index（PG 16+）

```sql
-- 若 GIN index 包含排序列，且查詢有 ORDER BY + LIMIT
-- PG 16+ optimizer 可能先按順序取 TID，減少 Bitmap 排序量
CREATE INDEX idx_t3_gin_id_info ON t3 USING GIN (id, info);

SELECT * FROM t3
WHERE info && ARRAY[1]
ORDER BY id LIMIT 5;
```

> 補充（Senior Dev）：PG 10+ 支援 GIN index 的 parallel scan，對大量 TID 排序有加速（多 CPU core 並行排序）。但**不解決根本問題**——仍要掃全部 TID。真正的解法是未來版本可能支援 GIN Index Scan + LIMIT early cutoff（社群討論了多年，實作挑戰在於 GIN posting list 的物理順序 ≠ TID 順序，需要重新設計內部結構）。PG 18 尚未實現。

---

## 版本演進

| 功能 | 版本 | 說明 |
|------|------|------|
| `gin_fuzzy_search_limit` | PG 9.4+ | 軟上限隨機採樣 |
| `gin_pending_list_limit` | PG 9.5+ | 寫入優化（控制 pending list 大小） |
| Parallel GIN Index Scan | PG 10 | 多 CPU 並行排序 TID |
| GIN fastupdate 改善 | PG 12 | 降低 pending list 合併頻率 |
| btree_gin 改善 | PG 14 | 複合 index 效能提升 |
| GIN scan memory optimization | PG 16 | 降低大結果集的排序 memory 壓力 |

## 參考

- [PostgreSQL Bitmap Scan IO 放大原理與優化](https://github.com/digoal/blog/blob/master/201801/20180119_03.md)
- [PostgreSQL GIN Tips](https://www.postgresql.org/docs/devel/static/gin-tips.html)
