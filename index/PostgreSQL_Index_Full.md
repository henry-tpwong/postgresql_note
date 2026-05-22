# 一、PostgreSQL EXPLAIN 計畫中的掃描類型全解析

## 1 為什麼讀懂掃描類型是效能的關鍵

Postgres 查詢效能的秘密不僅在於 *查什麼*，更在於 **如何找到資料**。`EXPLAIN` 是理解查詢行為的核心工具，而其中最關鍵的就是 **掃描類型（Scan Type）**——它決定了查詢是毫秒級還是秒級。

本文涵蓋六種主要掃描類型、Planner 選擇邏輯、以及 Production 調校策略。

![Postgres EXPLAIN Plan](../images/01-header.jpg)

---

## 2 Sequential Scan（循序掃描）

![Sequential Scan](../images/02-seq-scan.jpg)

循序掃描是最基礎的資料存取方式：**從頭到尾逐行讀取整張表**，檢查每一行是否符合 `WHERE` 條件。

```sql
EXPLAIN SELECT * FROM accounts;

                            QUERY PLAN
-------------------------------------------------------------------
 Seq Scan on accounts  (cost=0.00..22.70 rows=1270 width=36)
(1 row)
```

### I. Planner 何時選擇 Seq Scan

Planner 並非「有索引就一定用」。選擇 Seq Scan 的典型場景：

| 條件 | 原因 |
|------|------|
| 小表（少於數百 rows） | 索引 lookup 的 random I/O overhead 比一次 sequential read 更貴 |
| 回傳大量 row（> ~5-10% 總 row 數） | 大量 random page fetch 累積成本超過一次性全表掃描 |
| 沒有合適索引 | Planner 別無選擇 |
| `WHERE` 條件 selectivity 極低 | 例如 `WHERE is_active = true` 且 99% rows 為 true |

### II. Senior Dev 補充：成本模型的直覺解讀

`cost=0.00..22.70` 中的兩個數字：
- **0.00** = startup cost（回傳第一行前消耗的 arbitrary unit）
- **22.70** = total cost（回傳所有行後的總成本）

Seq Scan 的 startup cost 通常是 0，因為不需要索引 lookup 就能直接產出第一行。total cost 與 page 數 + row 數成正比：

```
total_cost ≈ seq_page_cost * relpages + cpu_tuple_cost * reltuples
```

預設 `seq_page_cost = 1.0`、`cpu_tuple_cost = 0.01`，所以一個 1000 page 的表：
- page cost = 1000 * 1.0 = 1000
- row cost 相對微小

> **關鍵洞察**：`seq_page_cost` 設為 1.0 是基準值。`random_page_cost` 預設 4.0 的假設是「random read 比 sequential read 貴 4 倍」。但這只對 HDD 成立。SSD 或 cloud storage 中，許多 DBA 將 `random_page_cost` 降為 1.1-1.5，這會 **大幅改變 Planner 傾向 Index Scan 的意願**。

---

## 3 Index Scan（索引掃描）

![Index Scan](../images/03-index-scan.jpg)

索引掃描是 **兩階段** 操作：

1. **Index Lookup**：在 B-Tree 中快速定位符合條件的 index entry
2. **Heap Fetch**：用 index entry 中的 TID（ctid，即 page + offset）去 heap table 取完整 row

```sql
EXPLAIN SELECT * FROM accounts WHERE id = 5;

                                  QUERY PLAN
-------------------------------------------------------------------------------
 Index Scan using accounts_pkey on accounts  (cost=0.15..2.37 rows=1 width=36)
   Index Cond: (id = 5)
(2 rows)
```

Primary Key 自動建立 B-Tree 唯一索引，因此 PK 查詢幾乎總是用 Index Scan。

### I. Index Scan vs Seq Scan：Planner 的分界線

Planner 使用以下公式決定用 Index Scan 還是 Seq Scan：

```
random_page_cost * selectivity * relpages  vs  seq_page_cost * relpages
```

當 selectivity（選擇率）小於約 `seq_page_cost / random_page_cost` 時，Index Scan 勝出：
- HDD（random_page_cost=4.0）：門檻約 25%
- SSD（random_page_cost=1.1）：門檻約 90%

但 `effective_cache_size` 也影響決策：如果 Planner 認為大部分 index page 已在 shared buffer 中，random access 成本估算會降低。

### II. Senior Dev 補充：Index Scan 的 I/O 放大陷阱

Index Scan 看似高效，但存在一個容易被忽略的問題——**I/O 放大（Random I/O Amplification）**：

- 每個 heap tuple fetch 都是一次 random page access
- 如果 index 和 heap 的 physical ordering 不一致（correlation 低），每個 row 可能要讀一個不同的 data page
- 極端情況：回傳 1000 rows 可能觸發 1000 次 random disk read

Postgres 統計資訊中的 `pg_stats.correlation` 反映 column physical ordering 與 logical ordering 的吻合度（-1 到 1）。接近 -1 或 1 代表高度有序，Planner 會更傾向 Index Scan。接近 0 代表隨機分佈，Planner 可能轉向 Bitmap Scan。

```sql
-- 查看 correlation
SELECT attname, correlation FROM pg_stats
WHERE tablename = 'accounts' AND attname = 'id';
```

---

## 4 Bitmap Index Scan + Bitmap Heap Scan（點陣圖掃描）

![Bitmap Index Scan](../images/04-bitmap-scan.jpg)

Bitmap Scan 是 Index Scan 與 Seq Scan 之間的 **混合策略**。當查詢返回的行數太多讓 Index Scan 不划算，但又不足以讓 Seq Scan 成為最佳選擇時，Planner 會選用 Bitmap Scan。

### I. 兩階段機制

1. **Bitmap Index Scan**：掃描一個或多個索引，建立一個 in-memory **點陣圖（bitmap）**，標記所有*可能*包含目標行的 data page
2. **Bitmap Heap Scan**：根據 bitmap 依序讀取 heap page，每個 page 只讀一次（不像 Index Scan 可能重複讀取同一 page），再逐行檢查 `Recheck Cond`

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT customer_id, registration_date
FROM customer_records
WHERE gender = 'F'
  AND state_code = 'KS';

                                                               QUERY PLAN
----------------------------------------------------------------------------------------------------------------------------------------
 Bitmap Heap Scan on customer_records  (cost=835.78..8669.29 rows=49226 width=12) (actual time=5.717..38.642 rows=50184.00 loops=1)
   Recheck Cond: (state_code = 'KS'::bpchar)
   Filter: (gender = 'F'::bpchar)
   Rows Removed by Filter: 49682
   Heap Blocks: exact=6370
   ->  Bitmap Index Scan on idx_customer_state  (cost=0.00..823.48 rows=97567 width=0) (actual time=4.377..4.378 rows=99866.00 loops=1)
         Index Cond: (state_code = 'KS'::bpchar)
(10 rows)
```

### II. Recheck Cond 的必要性

Bitmap 以 page 為粒度標記（page 級別，而非 row 級別），所以 Bitmap Heap Scan 讀取的每個 page 中，**所有 row 都要重新檢查條件**（`Recheck Cond`）。這是因為一個 page 可能包含不符合條件的 row——bitmap 只知道「這個 page 可能有目標 row」，但不確定哪幾行是目標。

### III. Senior Dev 補充：多索引 Bitmap 組合（AND / OR）

Bitmap Scan 最強大的場景是多個 `WHERE` 條件各有獨立索引且用 `AND` / `OR` 連接：

```
WHERE state_code = 'KS' AND gender = 'F'
```

Planner 會：
1. 對 `idx_customer_state` 做 Bitmap Index Scan → bitmap A
2. 對 `idx_customer_gender` 做 Bitmap Index Scan → bitmap B
3. `AND` → bitmap A **AND** bitmap B（intersection）
4. `OR` → bitmap A **OR** bitmap B（union）
5. 用最終 bitmap 做 Bitmap Heap Scan，一次讀取 heap page

這在單一 Index Scan 無法同時利用兩個獨立索引時極為有用。EXPLAIN 中會看到多個 `BitmapOr` / `BitmapAnd` 節點：

```sql
-- 組合示例
EXPLAIN SELECT * FROM t WHERE a = 1 OR b = 2;

                          QUERY PLAN
---------------------------------------------------------------
 Bitmap Heap Scan on t
   Recheck Cond: ((a = 1) OR (b = 2))
   ->  BitmapOr
         ->  Bitmap Index Scan on idx_a
               Index Cond: (a = 1)
         ->  Bitmap Index Scan on idx_b
               Index Cond: (b = 2)
```

> **關鍵參數 `work_mem`**：bitmap 是存在記憶體中的。如果 `work_mem` 不夠容納完整 bitmap（或 heap page 超過 `effective_cache_size` 預估），Planner 可能放棄 Bitmap Scan 改用 Seq Scan。增大 `work_mem` 有助於讓 Bitmap Scan 出現在更合適的場景中。

---

## 5 Parallel Sequential Scan（平行循序掃描）

![Parallel Sequential Scan](../images/05-parallel-seq-scan.jpg)

當表非常大時，Postgres 可以啟動多個 **background worker**，每個 worker 掃描表的不同區塊，最後透過 `Gather` 或 `Gather Merge` 節點合併結果。

```sql
EXPLAIN (ANALYZE, VERBOSE, BUFFERS)
SELECT id, data_value
FROM parallel_test
WHERE data_value < 100000
ORDER BY data_value DESC
LIMIT 1000;

                                                                         QUERY PLAN
------------------------------------------------------------------------------------------------------------------------------------------------------------
 Limit  (cost=161310.11..161431.04 rows=1000 width=16)
   ->  Gather Merge  (cost=161310.11..220311.14 rows=487915 width=16)
         Workers Planned: 5
         Workers Launched: 5
         ->  Sort
               Sort Key: parallel_test.data_value DESC
               Sort Method: top-N heapsort  Memory: 163kB
               ->  Parallel Seq Scan on public.parallel_test
                     Filter: (parallel_test.data_value < '100000'::numeric)
                     Rows Removed by Filter: 750082
```

### I. 平行查詢的觸發條件

Planner 只在滿足以下條件時考慮 Parallel Scan：

| 條件 | 說明 |
|------|------|
| `max_parallel_workers_per_gather > 0` | 預設 2，控制單個 Gather 節點的 worker 數 |
| 表大小 > `min_parallel_table_scan_size` | 預設 8MB，太小的表不值得平行化 |
| `parallel_tuple_cost` 和 `parallel_setup_cost` 的估算 | 啟動 worker 和傳遞 tuple 的 overhead 能被規模效益抵消 |

### II. Senior Dev 補充：Gather vs Gather Merge

- **Gather**：各 worker 結果任意順序匯集，不對結果排序。適合無 `ORDER BY` 或 `ORDER BY` 由聚合函數處理的場景
- **Gather Merge**：每個 worker 先對自己區塊排序，Gather Merge 再合併 N 個有序流（類似 merge sort 的合併階段）。適合帶 `ORDER BY` + `LIMIT` 的查詢

> **Production 建議**：`max_parallel_workers_per_gather` 不是越大越好。每個 worker 消耗一個 connection slot 和 memory。典型配置為 2-4。`max_parallel_workers`（全局上限）通常設為 CPU core 數的一半。

---

## 6 Parallel Index Scan（平行索引掃描）

![Parallel Index Scan](../images/06-parallel-index-scan.jpg)

平行索引掃描的邏輯與 Parallel Seq Scan 類似，但 worker 們掃描的是 **索引的不同區段**（B-Tree 的 leaf page 是雙向鏈表，worker 可以各自負責一段）。

```sql
EXPLAIN (ANALYZE, VERBOSE, BUFFERS)
SELECT data_id, filler_text
FROM parallel_index_test
WHERE data_id BETWEEN 1000000 AND 2000000;

                                                                                QUERY PLAN
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Gather  (cost=0.43..34560.34 rows=995971 width=109)
   Workers Planned: 4
   Workers Launched: 4
   ->  Parallel Index Scan using idx_data_id on public.parallel_index_test
         Index Cond: ((data_id >= 1000000) AND (data_id <= 2000000))
         Index Searches: 1
```

### I. Senior Dev 補充：B-Tree Parallel Scan 的內部機制

Postgres 使用 **Parallel B-Tree Scan** 的流程如下：

1. Leader process 先讀取 index meta page，計算 index 的 page 範圍
2. 將 B-Tree leaf page 鏈表按 page 數平均分配給 N 個 worker
3. 每個 worker 從分配的起始 leaf page 開始，沿著雙向鏈表掃描
4. 每個 index entry 指向 heap tuple，worker 各自 fetch 對應的 heap data

關鍵限制：B-Tree 的平行掃描僅在 **leaf page 層級** 平行化。Root 到 leaf 的 traversal（descend）仍由每個 worker 獨立進行。

> **注意**：查看 `EXPLAIN (ANALYZE, BUFFERS)` 輸出時，留意 `Buffers` 的分佈。如果某個 worker 的 `Buffers` 明顯高於其他 worker（資料分佈偏斜），代表平行掃描的效益被不均衡的數據分佈削弱。這是 partition table 或重新設計索引策略的信號。

---

## 7 Index-Only Scan（純索引掃描）

![Index-Only Scan](../images/07-index-only-scan.jpg)

Index-Only Scan 是效能之星：**所有查詢所需的欄位都存在索引中**，完全不需要存取 heap table。

```sql
EXPLAIN (ANALYZE, VERBOSE, BUFFERS)
SELECT code, status
FROM index_only_test
WHERE code > 'CODE_050000'
ORDER BY code
LIMIT 100;

                                                                           QUERY PLAN
----------------------------------------------------------------------------------------------------------------------------------------------------------------
 Limit  (cost=0.42..2.60 rows=100 width=13)
   ->  Index Only Scan using idx_code_status on public.index_only_test
         Index Cond: (index_only_test.code > 'CODE_050000'::text)
         Heap Fetches: 0
         Index Searches: 1
```

關鍵指標 `Heap Fetches: 0`——代表完全沒碰 heap，純讀索引。這是 **最高效能** 的訊號。

### I. 何時該建立 Covering Index

適合建 covering index 的場景：

| 信號 | 說明 |
|------|------|
| 查詢頻率極高 | 大量重複查詢的微小改善會累積成顯著收益 |
| 只 SELECT 少數欄位 | 例如 20 欄的表只查 3 個，把這 3 個欄位放進索引 |
| `EXPLAIN` 顯示大量 Heap Fetch | 表示現有 Index Scan 耗費大量 random I/O 做 heap fetch |
| 寫入頻率低 | 每個索引都需在寫入時維護，高寫入表的 covering index 會造成寫入放大 |

### II. Senior Dev 補充：Visibility Map、MVCC 與 VACUUM — 為什麼 VACUUM 是 Index-Only Scan 的幕後功臣

> **實戰法則**：如果你建了 covering index 但 `Heap Fetches` 仍然很大，**問題不是索引設計，而是 VACUUM 策略**。增加 `autovacuum` 頻率或在大量寫入後手動 `VACUUM` 可以恢復 Index-Only Scan 的效能。

要理解為何 VACUUM 是 Index-Only Scan 的「幕後功臣」，得先從 PostgreSQL 的 MVCC（多版本並行控制）和 Visibility Map（可見性地圖）說起。

#### a 為什麼一行資料會變成「看不見」？

在 PostgreSQL 中，`UPDATE` 或 `DELETE` 並不會立刻移除舊資料，而是在原地留下一個 **死元組（dead tuple）**。這樣設計是為了讓其他正在執行的事務還能看見他們「應該看見」的版本。

- 當你執行 `UPDATE` 時，Postgres 會插入一個新版本的 row，並把舊版本標記為無效。
- 這些舊版本依然佔用磁碟空間，而且 heap page 上會同時存在「對某些事務可見」和「對某些事務不可見」的 tuple。

只有當沒有任何事務還需要這些舊版本時，它們才會被 `VACUUM` 清理，空間才能被回收重用。

#### b Visibility Map：Index-Only Scan 的「免檢金牌」

每個 heap table 都有一張對應的 **Visibility Map（VM）**，它記錄每一個 data page 的兩個重要狀態：

- **all-visible**：這個 page 裡的所有 tuple 對所有事務都已經是可見的（沒有未提交的插入、沒有死元組需要遮擋）。
- **all-frozen**：這個 page 已被凍結，防止事務 ID 回捲（進階主題）。

當查詢進行 Index-Only Scan 時，會發生以下流程：

```
                         ┌─────────────┐
                         │  索引找到   │
                         │  目標 row   │
                         │  所在 heap  │
                         │    page     │
                         └──────┬──────┘
                                │
                    ┌───────────▼───────────┐
                    │ 該 page 的 VM 標記    │
                    │   為 all-visible？    │
                    └───────────┬───────────┘
                                │
                 ┌──────────────┼──────────────┐
                 │ YES                         │ NO
                 │                             │
    ┌────────────▼────────────┐    ┌───────────▼───────────┐
    │ 直接從索引返回資料      │    │ 必須去 heap page 檢查  │
    │（Heap Fetches: 0）      │    │ tuple 的可見性         │
    └─────────────────────────┘    └───────────┬───────────┘
                                              │
                                    ┌─────────▼─────────┐
                                    │ 增加 Heap Fetches  │
                                    │       計數         │
                                    └─────────┬─────────┘
                                              │
                                    ┌─────────▼─────────┐
                                    │ 若 tuple 對目前    │
                                    │ 事務可見 → 回傳   │
                                    │ 否則 → 忽略       │
                                    └───────────────────┘
```

- 如果 VM 說 page 是 `all-visible`，Postgres 就完全信任它，連碰都不碰 heap，直接從 index 輸出結果。這就是 `Heap Fetches: 0` 的最高境界。
- 只要 page 不滿足 `all-visible` 條件（例如曾發生過 `UPDATE` 但尚未被 `VACUUM`），就必須實際讀取 heap page，逐一檢查每一行的可見性，產生 `Heap Fetches`。

#### c VACUUM 如何讓 VM 變成 all-visible？

`VACUUM` 會做兩件關鍵事：

1. **清理死元組**：將不再被任何事務需要的舊版本 tuple 標記為可用空間。
2. **更新 Visibility Map**：清完 dead tuple 後，該 page 上剩下的全部都是「對所有人可見」的 live tuple，此時 `VACUUM` 就會把這個 page 在 VM 中的 `all-visible` bit 設為 1。

之後的 Index-Only Scan 再碰到這個 page，就能直接跳過 heap fetch，因為 VM 已經保證「不用檢查了，全部可見」。

這就是那條實戰法則背後的原理：因為即使索引裡包含了所有需要的欄位，只要那些 heap page 還沒被 `VACUUM` 更新過 VM，Postgres 就不敢直接用索引的內容，被迫一次又一次地去 heap 確認，導致大量的 `Heap Fetches`。

#### d 實際監控與調整策略

**4.1 檢查死元組與 VACUUM 狀態**

```sql
SELECT relname,
       n_live_tup,
       n_dead_tup,
       round(100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 2) AS dead_pct,
       last_autovacuum,
       last_vacuum
FROM pg_stat_user_tables
WHERE relname = 'your_table_name';
```

- 如果 `dead_pct` 超過 5%~10%，且你的查詢經常使用 Index-Only Scan，就表示 VACUUM 的頻率需要提高。
- 觀察 `last_autovacuum` 和 `last_vacuum` 的時間，若太久沒執行，就是警訊。

**4.2 手動觸發 VACUUM 進行測試**

```sql
VACUUM (VERBOSE) your_table_name;
```

然後再執行一次你的查詢，查看 `EXPLAIN (ANALYZE, BUFFERS)` 中 `Heap Fetches` 是否大幅下降。通常立刻就能看到改善。

**4.3 調整 autovacuum 參數（單表級別）**

如果手動 `VACUUM` 有效，就代表 autovacuum 的預設設定對這張表來說太「寬鬆」。可以在單表上調整 storage parameter：

```sql
ALTER TABLE your_table_name SET (
    autovacuum_vacuum_scale_factor = 0.01,   -- 原本預設 0.2（20%），改成 1%
    autovacuum_vacuum_threshold = 1000       -- 死元組超過 1000 就觸發
);
```

- `autovacuum_vacuum_scale_factor`：表中有多大比例的 dead tuple 才觸發 autovacuum。對於大型表（上百萬行），`0.2` 可能會累積數十萬死元組才動作，改為 `0.01`~`0.05` 會讓 VACUUM 更頻繁。
- `autovacuum_vacuum_threshold`：絕對死元組數量門檻，避免極小表頻繁 vacuum。

**4.4 加快 Index-Only Scan 的輔助：VACUUM FREEZE**

如果你的表有大量寫入，也可以考慮稍微積極的凍結策略（讓 VM 也標記為 `all-frozen`），不過一般情況調高 autovacuum 頻率就足夠。

#### e 總結：VACUUM 是 Index-Only Scan 效能的真正守門員

- 沒有 `VACUUM`，Visibility Map 就不會更新，Index-Only Scan 永遠無法信任索引中的資料，必須頻繁回表檢查。
- 即使你精心設計了 covering index，只要寫入多、VACUUM 少，`Heap Fetches` 就會居高不下，索引效益大打折扣。
- 在生產環境中，監控 dead tuple 比例 + 調校 autovacuum 參數，和設計索引本身一樣重要。

### III. 包含欄位的索引（INCLUDE Index）

PG 11+ 支援 `INCLUDE` 子句，可以在不影響索引鍵（key column）排序的前提下，在 leaf page 中攜帶額外欄位：

```sql
CREATE INDEX idx_code_status ON index_only_test (code) INCLUDE (status);
```

優勢：
- `code` 決定 B-Tree 排序和搜尋路徑
- `status` 只存在 leaf page，不參與 branch page 的分叉決策
- 索引體積比把 `status` 也當成 key column 小，且不會影響唯一性約束

`INCLUDE` 索引特別適合：按 A 排序但只需要 A + B 兩個欄位的查詢。

---

## 8 掃描類型總覽與決策流程圖

![Summary of Scan Types](../images/08-summary.jpg)

### I. 六種掃描類型速查

| 掃描類型 | EXPLAIN 關鍵字 | 運作方式 | 最佳場景 |
|----------|---------------|---------|---------|
| Sequential Scan | `Seq Scan` | 讀取整表，逐行檢查 | 小表、回傳 > ~10% rows |
| Index Scan | `Index Scan` | B-Tree lookup → heap fetch | 精準查詢（PK / unique），回傳少量行 |
| Bitmap Scan | `Bitmap Index Scan` + `Bitmap Heap Scan` | 建立 page-level bitmap → 依序讀 heap | 中等選擇率、多條件 `AND`/`OR` |
| Parallel Seq Scan | `Parallel Seq Scan` + `Gather` | 多 worker 分塊掃表 | 大型全表掃描 |
| Parallel Index Scan | `Parallel Index Scan` + `Gather` | 多 worker 分區段掃描索引 | 大型索引範圍掃描 |
| Index-Only Scan | `Index Only Scan` | 純讀索引，不碰 heap | 所有查詢欄位都在索引中 + VM clean |

### II. Senior Dev 補充：掃描選擇的實戰除錯策略

當查詢效能不佳時，按以下步驟診斷：

**1. 先確認掃描類型**

```sql
EXPLAIN (ANALYZE, BUFFERS, TIMING) SELECT ...
```

重點觀察：
- `actual time` vs `estimated cost` 的差距（統計資訊是否過時）
- `Buffers` 的 `shared hit` vs `shared read`（cache hit ratio）
- `Heap Fetches`（Index-Only Scan 是否有效）

**2. 判斷 Planner 是否選錯掃描**

如果是 Seq Scan 但直覺該用 Index Scan：
```sql
-- 檢查 planner 參數
SHOW random_page_cost;     -- SSD 建議 1.1 - 1.5
SHOW effective_cache_size; -- 建議設為 RAM 的 50-75%
SHOW seq_page_cost;        -- 基準值 1.0

-- 更新統計資訊
ANALYZE table_name;

-- 查看 column 的 n_distinct 和 correlation
SELECT attname, n_distinct, correlation
FROM pg_stats
WHERE tablename = 'table_name' AND attname IN ('col1', 'col2');
```

 **3. 臨時性測試（只在 Session 層級）**

```sql
SET enable_seqscan = off;    -- 強制跳過 Seq Scan，驗證 Index Scan 成本
EXPLAIN (ANALYZE) SELECT ...;
-- 確認 Index Scan 是否真的更快，然後決定是否調整參數
```

> **警告**：**絕對不要在 production 全局關閉 `enable_seqscan`**。這只是一個診斷手段。如果 Index Scan 真的更優，問題是參數設定（`random_page_cost` 或 `effective_cache_size` 不準），應調整參數而非關閉 seq scan。

**4. 如果 Heap Fetches 過高**

```sql
-- 手動 VACUUM 更新 visibility map
VACUUM (VERBOSE) table_name;

-- 再次檢查 Index-Only Scan 是否達到 Heap Fetches: 0
EXPLAIN (ANALYZE, BUFFERS) SELECT ...;
```

### III. 影響 Planner 掃描選擇的關鍵 GUC 參數

| 參數 | 預設值 | 影響 | 建議 |
|------|--------|------|------|
| `random_page_cost` | 4.0 | 降低會讓 Index Scan 更容易勝出 | SSD 設 1.1-1.5 |
| `seq_page_cost` | 1.0 | 基準值，一般不需調整 | 保持 1.0 |
| `effective_cache_size` | 4GB | 影響 Planner 對 index page 在 cache 中的假設 | 設為 OS cached + shared_buffers 的 50-75% |
| `work_mem` | 4MB | 影響 Bitmap Scan 是否被選用 | 視 query 類型設 16MB-256MB |
| `min_parallel_table_scan_size` | 8MB | 低於此的表不做 Parallel Scan | 預設合理 |
| `max_parallel_workers_per_gather` | 2 | 單個 Gather 的最大 worker 數 | 2-4，不超過 CPU core 的一半 |

---

## 9 延伸閱讀與工具

- [PostgreSQL EXPLAIN 官方文件](https://www.postgresql.org/docs/current/using-explain.html)
- [explain.depesz.com](https://explain.depesz.com/) — 貼上 EXPLAIN 輸出，自動解析和上色顯示 cost/actual time 差異
- [explain.dalibo.com](https://explain.dalibo.com/) — 另一款 EXPLAIN 視覺化工具，支援多語系
- [CrunchyData - Parallel Queries in Postgres](https://www.crunchydata.com/blog/parallel-queries-in-postgres)
- [PG 17 新特性：B-Tree Index Deduplication 對掃描的影響](https://www.postgresql.org/docs/17/btree-implementation.html)


# 二、PostgreSQL Index 核心機制

## 1 Bitmap Heap Scan

### I. Block = Page（8KB）

Block 與 Page 在 PostgreSQL 中是完全同義詞，指向磁盤上最小的存儲與 I/O 單元，預設大小 8KB。

- **物理層面**：一張表（或 index）在磁盤上由一個或多個固定大小的 page 組成，每個 page 8KB。當 PostgreSQL 需要讀取數據時，不會一次只讀一行，而是以 page 為最小單位讀入 shared_buffers。

> **圖解補充：**

> ```mermaid
> graph TB
>     subgraph Disk ["💾 磁盤 (Disk)"]
>         direction TB
>         Table[(📄 表文件<br/>Heap / Index)]
>         Page1[📋 Page 1<br/>8KB]
>         Page2[📋 Page 2<br/>8KB]
>         Page3[📋 Page 3<br/>8KB]
>         PageX[📋 ...<br/>Page N]
>         
>         Table --- Page1
>         Table --- Page2
>         Table --- Page3
>         Table --- PageX
>         
>         Page1 --> Row1[Row A]
>         Page1 --> Row2[Row B]
>         Page1 --> Row3[Row C]
>     end
> 
>     subgraph Memory ["🧠 共享內存 (shared_buffers)"]
>         Buffer[🗂️ 緩衝區<br/>一次載入一個完整 Page]
>     end
> 
>     Page1 -->|"⚡ 最小 I/O 單位：<br/>1 個 Page (8KB)"| Buffer
>     Page2 -->|"⚡ 下一個 Page"| Buffer
>     Page3 -->|"⚡ 按需依序讀入"| Buffer
> 
>     linkStyle default stroke:#1a56db,stroke-width:4px
> ```

> **圖解說明：**
> 
> 1. **表由多個固定大小的 Page 組成**：左邊的「表文件」物理上是一連串的 8KB Page（Page 1, Page 2, Page 3 ...）。無論是 Heap 表還是 Index，底層存儲都用相同的 Page 結構。
> 2. **Page 是磁盤 I/O 的最小單位**：PostgreSQL 絕對不會「只讀一行」。即使查詢只要求一行數據（例如 `SELECT * FROM t WHERE id=100`），數據庫也會把 **包含那一行的整個 Page（8KB）** 從磁盤讀進記憶體。圖中箭頭標示了「最小 I/O 單位：1 個 Page (8KB)」。
> 3. **讀入 shared_buffers**：右側的 `shared_buffers` 是 PostgreSQL 的緩衝池（記憶體中的一塊區域）。讀取時 Page 會先被載入這裡，後續查詢若命中相同 Page 就可以直接從記憶體拿，不再讀磁盤。圖中每個 Page 按順序移進 shared_buffers，這也是 Bitmap Heap Scan 能實現「物理順序讀取」的基礎：先排好要讀哪些 Page，再一口氣批量載入，減少磁頭跳動。
> 4. **Page 裡面含有多行**：從 `Page 1` 中放大可以看到 Row A、Row B、Row C，表示一個 Page 通常會儲存數十到上百行（視行寬而定）。所以「以 Page 為單位讀取」也自然解釋了為何 Bitmap Heap Scan 的效能瓶頸在 Page 讀取量，而不只是行數。
> 
> 💡 **簡單記法**：
> - **磁盤上**：表 → 多個 8KB Page。
> - **讀取時**：一次一個 8KB Page 進 shared_buffers。
> - **行**：只是 Page 裡的內容，隨著 Page 一起上來。
> 
> 如果圖無法渲染，可以想像一本書：整本書是「表」，每頁是「Page」（固定大小），你想看某一行字，必須翻開整頁，不可能只把那一行單獨抽出來看。PostgreSQL 的 I/O 行為正是如此。

- **稱呼習慣**：核心代碼和文檔中經常混用。Page 偏重邏輯結構（內有 row、page header），Block 偏重物理存儲、I/O 操作（如 blkno）。在 execution plan 和日常討論中完全可以互換。

### II. B-tree 的 Bitmap Index Scan：預設 Row-Level，但可退化為 Lossy

- **預設情況（memory 充足）**：B-tree 構建的位圖是 row-level（TID bit array），每一位對應一個具體的 `(page, item)`，可以精確到 row。
- **memory 不足時**：如果 result set 很大，超出 `work_mem` 的限制，PostgreSQL 會把 bitmap 退化為 lossy bitmap（page-level），此時每個位對應一個 page，整個 page 要被讀出來逐行 Recheck。

從 `EXPLAIN (ANALYZE, BUFFERS)` 的輸出中，看 Bitmap Heap Scan 的 `Heap Blocks: exact=xxx lossy=yyy` 來判斷是否退化。lossy 不為零就說明退化發生了。

> 補充（Senior Dev）：預設 `work_mem = 4MB` 對於現代伺服器極度保守。一個 query 可能用到 `work_mem * number_of_parallel_workers` 的 memory（例如 Hash Join），而不是 `work_mem` 上限。在 OLAP 場景建議 `work_mem = 256MB ~ 1GB`，但 OLTP 場景保持較低以避免單個 connection 耗盡 memory。

### III. Bitmap 結構：1 bit = 1 page（非 1 row）

這是理解 Bitmap Heap Scan 的關鍵。Bitmap 不是按 row 構建的，而是按 page 構建。

假設一張表有 100 萬 row，存儲在 10,000 個 8KB 的 page 裡。Bitmap 就是一個長度為 10,000 的 bit array：第 i 位對應表文件中的第 i 個 data page（page number i）。

**標記過程（Bitmap Index Scan）**：掃描 index（如 BRIN 或 B-tree）時，並不是馬上回 Heap 讀取 row。Index Scan 會找出哪些 page 可能包含符合條件的 row，然後把對應的 bit 設為 1。

- 對於 B-tree：每個 index entry 都包含 row 所在的 page 號。掃描時，會把所有匹配 entry 引用的 page 號對應的 bit 設為 1。
- 對於 BRIN：每個 index summary 覆蓋一個 page range（預設 128 個 page）。如果 summary 滿足條件（例如時間範圍重疊），就會把那 128 個 page 對應的 bit 全部設為 1。

**讀取過程（Bitmap Heap Scan）**：Bitmap 建好後，PostgreSQL 會遍歷這個 bitmap，找到所有被標記為 1 的 page，然後按物理順序（page 號遞增）依次把它們讀入 shared_buffers。這一步才真正從 Heap 中讀取 row。因為是順序讀取 page，磁頭或 SSD 不需要隨機跳轉，I/O 效率極高——這正是 Bitmap Heap Scan 優於普通 Index Scan 的核心原因。

### IV. 為什麼用 page 作為 Bitmap 粒度，而不是 row？

- **I/O 單位決定**：無論如何，讀取數據最後都要落到 page。即使 index 能找到具體 row，最終讀取的還是包含該 row 的整個 page。所以用 page 作為 bitmap 粒度，能直接映射到物理讀取。
- **memory 和 CPU 效率**：1000 萬 row 如果用 row-level bitmap 需要 10M bit（~1.25MB），而用 page-level bitmap（假設每 page 100 row）只需要 100K bit（~12.5KB），體積小一個數量級，構建和遍歷都快得多。
- **與 BRIN 天然契合**：BRIN 本身就是 page-level summary index，它返回的就是 page range，直接填充 bitmap 特別自然。

### V. 為什麼時間花在 Bitmap Heap Scan 上，而不是 Index Scan？

很多人的直覺是：「既然 index 幫我定位了 row，為什麼大頭時間還在 Heap Scan？」原因在於：

1. **I/O 繞不開，最終數據要從 Heap 讀取**：Index 裡只存了 index key 和 row pointer（TID），並沒有存完整的 row 數據。無論 bitmap 是 row-level 還是 page-level，Bitmap Heap Scan 必須拿著這些 pointer 去 Heap 中取出整行數據。這個步驟就是慢的主要來源，尤其是：
   - 需要訪問的 data page 數量很大（即使每個 page 只取一行）。
   - 這些 page 大多不在 memory（shared_buffers 或 filesystem cache），需要物理 disk read。
2. **雖然避免了 random I/O，但讀取的 page 數可能依然很多**：Bitmap Heap Scan 會把需要訪問的 page 號排序，然後按物理順序批量讀取，這比普通 Index Scan 的隨機單 page 跳轉要快得多。但是，如果 query 本身需要讀取 10000 個不同的 data page，這 10000 次 read 仍然是實打實的 I/O 消耗。時間自然就堆在這個階段。
3. **Recheck Cond 帶來的 CPU 開銷（尤其在退化時）**：如果 bitmap 退化成了 page-level，或者使用了 BRIN 這類有損 index，那麼整個 page 內的所有 row 都要被重新檢查條件，這會產生可觀的 CPU 時間。對於 B-tree，如果 lossy 發生，同樣會因 page-level 讀取而多出大量無用的 row 檢查。
4. **Bitmap Heap Scan 是「真正幹重活」的節點**：在 plan 裡，Bitmap Index Scan 只負責掃 index、填 bitmap，幾乎純 CPU，速度很快，所以 actual time 非常小。而接下來 Bitmap Heap Scan 負責實際的表訪問和過濾輸出，是整個 query 的體力活，時間自然全部集中在這裡。

> 總結：B-tree 預設構建的是 row-level 的 bitmap（精確到 row），不是 page-level，除非 memory 不夠退化為 lossy。慢的主要原因是 Bitmap Heap Scan 要真正去讀 Heap data page，這一步驟集中了絕大多數的 I/O 和大量的 CPU 過濾。row-level bitmap 只幫你避免了「整 page Recheck」的 CPU 浪費，但無法避免「讀 page」這個動作本身。看到時間都在 Bitmap Heap Scan 上是正常的，關鍵在於判斷它是 I/O bound 還是 CPU bound，然後對症優化。
>
> #### I/O Bound vs CPU Bound（效能診斷基礎）
>
> 這對概念是效能診斷的基礎，直接決定你的優化方向。
>
> **什麼是 I/O Bound 與 CPU Bound？**
>
> 繼續用 `EXPLAIN (ANALYZE, BUFFERS)` 輸出為例：
>
> ```text
> Bitmap Heap Scan on orders  (actual time=0.123..10.456 rows=4800 loops=1)
>    Recheck Cond: ...
>    Rows Removed by Index Recheck: 12000
>    Heap Blocks: exact=800 lossy=200
>    Buffers: shared hit=200 read=800    -- 這裡是關鍵
> ```
>
> **1. 判斷為 I/O Bound**
>
> 當 `Buffers: read` 的數值很大時，代表絕大多數的 page 不在記憶體中，需要**物理磁盤讀取**。這就是典型的 I/O Bound。
>
> - **特徵**：查詢大部分時間花在等磁盤。
> - **解法**：
>   - 加大記憶體（`shared_buffers` 或伺服器本身記憶體），讓更多資料能留在快取中。
>   - 使用更快的磁盤（SSD）。
>   - 減少需讀取的資料量，例如改用能更快過濾的索引（B-tree 而非 BRIN），或是使用 Covering Index 達成 Index Only Scan，根本避免去讀 Heap Page。
>
> **2. 判斷為 CPU Bound**
>
> 當 `Buffers: shared hit` 佔絕大多數，但查詢依然很慢時，問題就在 CPU 計算上。此時瓶頸可能是：
>
> - **Recheck 開銷大**：`Rows Removed by Index Recheck` 數值遠大於最終回傳行數（例如 12000 vs 4800），表示 CPU 花費大量時間在過濾那些最後被丟棄的誤報行。
> - **資料處理複雜**：WHERE 條件本身計算複雜（如正則表達式），或回傳行數極多，需要大量運算。
>
> - **特徵**：資料都在記憶體了，但 CPU 使用率飆高，查詢還是慢。
> - **解法**：
>   - 升級更快的 CPU。
>   - 如果是 Recheck 導致，就對症下藥：加大 `work_mem` 避免精確點陣圖退化為 Lossy Page-Level 點陣圖（針對 B-tree）；或是調整 BRIN 的 `pages_per_range` 來降低誤報率。
>
> **一句話總結**
>
> > 看 `Buffers` 來判斷：`read` 多、等待久，就是 **I/O Bound**；`hit` 多、但 CPU 使用率飆高，就是 **CPU Bound**。這個判斷會直接告訴你該花錢買記憶體/SSD，還是該花時間優化 SQL 與索引。

### VI. Recheck Cond 詳解

Recheck Cond 是 PostgreSQL execution plan 中 Bitmap Heap Scan 節點上的一個關鍵步驟。其核心任務是：從 data page 中取出每一行後，再次用 index 條件核實它真的符合要求，把誤傷的 row 丟棄。

#### a 為什麼需要 Recheck？——有損性的根源

Index Scan 構建 bitmap 時，不一定能精確到「具體哪一行滿足條件」，有時只能告訴你「哪些 page 裡可能有」。這種不精確就叫有損。Recheck 就是為此而生，確保最終結果絕對正確。

有損主要來自兩種情況：

1. **Index 本身有損**：
   - **BRIN**：只記錄每個 page range 的 min / max。如果查詢 `create_time >= '2026-05-10'`，一個範圍 `min='2026-05-09', max='2026-05-11'` 的 page block 就會被標記。但這個 page block 裡很可能存在 `2026-05-09` 的 row，它們不滿足條件，必須 Recheck 過濾。
   - **GIN**：Inverted index 存儲了 key 和對應的 page 號列表，但一個 page 裡可能有多 row，有的符合全文查詢，有的不符合。
   - **GiST**：取決於 operator class。例如地理位置的 `&&`（bounding box overlap）是有損的，因為 bounding box 重疊不代表幾何體真正相交。

2. **Bitmap 本身精度降低（Lossy Bitmap）**：即使 index 是精確的（如 B-tree），bitmap 本身也可能變有損。PostgreSQL 首先構建 row-level bitmap（每個 bit 對應一個 TID），但這會消耗 memory（work_mem）。如果 result set 巨大，memory 不足，系統會把 bitmap 退化為 page-level bitmap——只記錄哪些 page 包含匹配 row，丟掉了具體 row 位置。退化的 page-level bitmap 也是有損的，導致掃描時必須把整個 page 讀出來逐行 Recheck。

> 補充（Senior Dev）：Lossy bitmap 發生時，不只 Recheck 成本增加，還可能導致原本可通過 BitmapAnd 精確合併的多 index 條件被迫在 page 層面做合併，進一步放大誤報。例如 `WHERE status = 'active' AND create_time > '...'` 兩個 Bitmap Index Scan 若都退化，BitmapAnd 後每個 page 內仍需逐 row 驗證兩個條件。

#### b Recheck Cond 在不同 Index 類型下的表現

| Index 類型 | Bitmap 精確度 | 是否需要 Recheck | 說明 |
|-----------|-------------|----------------|------|
| B-tree | 預設構建精確 row-level bitmap | 通常不需要（plan 可能不顯示 Recheck Cond） | 如果 work_mem 不足導致 bitmap 退化（lossy），會顯示 Recheck Cond |
| BRIN | 永遠是 page-level，有損 | 一定需要，plan 中總是出現 Recheck Cond | Index 只提供 page range，本來就不精確到 row |
| GIN | 總是 page-level，有損 | 一定需要 | Inverted index 只指向 page，page 內可能有多 row |
| GiST | 可能精確也可能有損，視 operator 而定 | 視情況而定 | 例如 `btree_gist` 的 `=` 可精確，但 `range @> elem` 通常有損 |

#### c Recheck 的具體機制

舉例查詢：

```sql
SELECT * FROM orders WHERE create_time >= '2026-05-10';
```

1. **構建 bitmap**：BRIN Scan 返回 page 號 100~200 等，構建 page-level bitmap。
2. **Bitmap Heap Scan 開始**：從 page 100 開始，把整個 8KB page 讀入 memory。
3. **Recheck 每一行**：遍歷 page 內所有 row（可能幾十到上百行），對每一行評估 `create_time >= '2026-05-10'`。
   - 符合條件 → 返回給上層。
   - 不符合 → 丟棄（在 EXPLAIN ANALYZE 中可能看到 `Rows Removed by Index Recheck`）。
4. 處理完 page 100，接著處理 page 101，按 page 號順序直到結束。

**關鍵點**：Recheck 的 CPU 開銷取決於候選 page 的總 row 數，而非最終結果 row 數。如果 BRIN index 定位的 page range 很寬，但實際符合條件的數據很少，就可能出現「讀了大量 page，Recheck 後丟棄絕大部分 row」的情況，這是性能不佳的徵兆。

> 補充（Senior Dev）：可以用以下 SQL 模擬 BRIN Recheck 的健康度，在自己表上測量「命中率」：
> ```sql
> SELECT
>   (SELECT count(*) FROM big_table WHERE ts BETWEEN '...' AND '...') AS actual_rows,
>   (SELECT reltuples::bigint * (pages_in_range::float / relpages)
>    FROM pg_class WHERE relname = 'big_table') AS estimated_scanned_rows;
> ```
> 若 `estimated_scanned_rows >> actual_rows`，BRIN 誤報嚴重，考慮調小 `pages_per_range`。

#### d Recheck Cond vs Filter

| 謂詞 | 所在節點 | 用途 |
|------|----------|------|
| Recheck Cond | 只在 Bitmap Heap Scan 中 | 重新驗證 index 條件，是對「可疑」row 的第二次篩查，確保 index 的有損性不影響最終正確性 |
| Filter | 幾乎任何節點都可以有 | Execution plan 中無法使用 index 的其他條件，或者 index 條件以外的過濾。例如 `WHERE create_time > ... AND status = 'done'`，status 的過濾可能就是 Filter |

在 plan 中你可能會同時看到兩者：index 負責時間範圍（Recheck Cond），然後對其他列進行 Filter。

#### e 小結 Recheck

- Recheck Cond 是「有損 index / 有損 bitmap」的糾錯機制，必須執行以保證結果正確。
- BRIN index 必然導致 Recheck Cond，因為 index 只能標記 page range。
- 它的開銷取決於誤報 page 中的總 row 數，通過 `Rows Removed by Index Recheck` 可以量化。
- 優化思路：讓 index 更精準（調整 BRIN 參數）或避免 bitmap 退化（加大 work_mem，但對 BRIN 先天 page-level 無效）。

---

## 2 EXPLAIN 分析與效能優化

### I. 重點觀察指標

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders
WHERE create_time BETWEEN '2026-05-01' AND '2026-05-07';
```

| 指標 | 含義 | 判斷 |
|------|------|------|
| `Buffers: shared hit=... read=...` | hit = 已在 memory，read = 物理 disk I/O | read 很大說明大量 data page 不在 memory，是最常見的主因 |
| `Heap Blocks: exact=xxx lossy=yyy` | exact = 精確 page 數（來自 index，無退化），lossy = 退化為 page-level bitmap 的 page 數 | lossy 很大 → 需要增加 work_mem（如 `SET work_mem = '256MB'`） |
| `Rows Removed by Index Recheck` | Recheck 丟棄了多少 row | 遠大於最終返回 row 數 → index 誤報率高，CPU 浪費嚴重，要麼 bitmap 退化嚴重，要麼有損 index |
| `actual time` | Bitmap Index Scan 的 actual time 很小（純 CPU），Bitmap Heap Scan 的 actual time 大 | 正常現象 |

範例 EXPLAIN 輸出：

```
Bitmap Heap Scan on orders  (actual time=0.123..10.456 rows=4800 loops=1)
   Recheck Cond: (create_time >= '2026-05-01'::date AND create_time <= '2026-05-07'::date)
   Rows Removed by Index Recheck: 12000
   Heap Blocks: exact=800 lossy=200
   Buffers: shared hit=1000
   ->  Bitmap Index Scan on idx_brin_orders_time  ...
```

解讀：`Rows Removed by Index Recheck: 12000` 遠大於實際返回的 4800 row，大量 row 被 Recheck 後丟棄。`Heap Blocks: lossy=200` 說明有 200 page 的 bitmap 退化了。`Buffers: shared hit=1000` 表示這些 page 都從 buffer cache 讀取——如果這裡的 read 很大，那說明物理 I/O 才是主因。

> 補充（Senior Dev）：`EXPLAIN (ANALYZE, BUFFERS, TIMING, COSTS, VERBOSE, SETTINGS)` 是所有參數的全開組合，建議寫成 alias。`SETTINGS` 會輸出當時的 GUC 參數（work_mem / random_page_cost 等），對跨環境比對 plan 差異極有用。另外，`auto_explain` extension 可以自動記錄慢查詢的 plan，比手動 `EXPLAIN ANALYZE` 更適合 production。

### II. 優化策略

| 問題 | 方案 |
|------|------|
| 位圖退化（lossy 很大） | 增加 `work_mem`，讓 bitmap 保持 row-level 精度，減少 page-level Recheck 開銷 |
| I/O read 很大（data page 不在 memory） | 加大 shared_buffers 和 OS memory，讓常用 data page 常駐 memory |
| 回表 I/O 過多 | 使用 Covering Index：`CREATE INDEX idx_orders_time_cover ON orders (create_time) INCLUDE (col1, col2)`，讓 query 變成 Index Only Scan，直接避開 Bitmap Heap Scan |
| 大範圍查詢、表極大、順序寫入 | 考慮 BRIN：極大減少 index 大小和 I/O，但必須接受一定比例的誤報 page Recheck |
| 單表數據量巨大 | Partition + Partition Pruning：按時間 partition，讓 query 只掃一個或幾個 partition，極大減少需要 bitmap 化的 page 數 |

> 補充（Senior Dev）：Covering Index（INCLUDE）與普通 composite index 的差異：INCLUDE column 不參與 index key 的排序和唯一性約束，只在 leaf page 存儲值。這意味著 INCLUDE 方式寫入成本更低、index 更小（因為 page split 只需按 key column 決定），但無法加速對 INCLUDE column 的 WHERE 過濾。真正需要加速過濾的 column 還是要放在 key 部分。
>
> Partition 與 BRIN 的組合應用：若按時間 partition 後，每個 partition 內數據按插入順序高度相關，BRIN 的效果會更好（因為 BRIN 依賴物理順序與邏輯順序的 correlation）。先分區、再 BRIN，是物聯網場景的黃金組合。

---

## 3 B+ Tree Leaf Page 與 Covering Index

### I. 什麼是 Leaf Page？

Leaf Page（葉子頁）是 B+ 樹索引中最底層的節點，也是實際存放「索引條目」的地方。

在 B+ 樹索引結構中：

- **根節點 / 內部節點**：只存鍵值 + 子頁指針，用來導航。
- **葉子頁（Leaf Page）**：位於最底層，存放完整的索引條目。
  - 若為聚簇索引（PostgreSQL 中不存在，但概念上等同於 table 本身），葉子頁本身就是數據頁。
  - 若為非聚簇索引，葉子頁存「索引鍵 + 指向數據行的指針（TID）」。
  - 所有葉子頁之間通常有雙向鏈結，支援範圍掃描。

#### a 圖解：B+ Tree 結構

下圖是一個簡化的 B+ 樹，索引鍵為 `user_id`：

```
                 [ 內部節點 ]
                 | 50 | 150 |
                /       |       \
           [內部]     [內部]     [內部]
           /    \     /    \     /    \
        [葉] [葉] [葉] [葉] [葉] [葉]     ← 葉子頁層，雙向鏈結
          ↕    ↕    ↕    ↕    ↕    ↕
        (1..49)(50..99)(100..149)(150..199)(200..249)(250..299)
```

放大其中一個葉子頁（假設非聚簇索引，並帶有 INCLUDE 欄位）：

```
+--------------------------------------------------+
|  Leaf Page (頁號 103)                             |
|  Prev Page: 102    Next Page: 104                 |
+--------------------------------------------------+
| 索引條目結構 (Key + 指標 + Included Columns)      |
|--------------------------------------------------|
| user_id=50 | row_ptr → (name, email, ...)        |  ← 只存指標，不存整行
| user_id=51 | row_ptr → ...                       |
| user_id=52 | row_ptr → ...                       |
| ...                                              |
+--------------------------------------------------+
```

#### b 如果有 Covering Index with INCLUDE

例如：

```sql
CREATE INDEX idx_cover ON orders (order_date) INCLUDE (amount, status);
```

則葉子頁內的條目會是：

```
[ order_date | row_ptr | amount | status ]
    ↑ key       ↑         ↑________↑
   排序依據    指向行    僅儲存在葉子頁，不參與排序
```

`amount`、`status` 的值直接嵌在葉子頁裡，查詢時若只需這些欄位，就不必回表。

### II. Covering Index（INCLUDE）與普通 Composite Index 的差異

核心差異在於**葉子頁儲存什麼**，以及**鍵的排序行為**。

| 特性 | 普通複合索引 (Composite Index) | 包含欄位索引 (INCLUDE) |
|------|-------------------------------|------------------------|
| 索引鍵定義 | `INDEX (A, B)` —— A、B 都在 key 中 | `INDEX (A) INCLUDE (B)` —— 只有 A 在 key |
| 排序與唯一性 | A、B 共同決定排序，參與唯一性約束 | 只有 A 決定排序，B 不參與 |
| 葉子頁內容 | `(A, B, row_ptr)` | `(A, row_ptr, B)` |
| WHERE 過濾 B 時 | 可以用索引快速定位 | 無法加速，B 僅在葉子頁中，無法當作導航鍵 |
| 寫入成本 & Page Split | 任何 key 欄位變動都可能觸發分裂，較重 | 僅依 A 決定分裂，成本更低 |
| 索引大小 | 通常較大（B 也影響樹結構） | 較小，樹結構只按 A 建立 |

**使用 INCLUDE 的時機**：

- 你想讓查詢變成覆蓋索引（Covering Index），但某些欄位不需要用於過濾或排序，只需輸出。
- 典型的例子：`WHERE order_date = ...`，但 `SELECT order_date, amount`，此時用 `INCLUDE (amount)` 即可，不用把 `amount` 放入 key。

### III. Partition + BRIN 的組合應用（IoT 黃金組合）

這跟 Leaf Page 沒有直接關係，但指向了另一個層面的儲存結構優化。

- **BRIN（Block Range INdex）** 依賴「物理儲存順序」與「邏輯值順序」的高度相關性。
- 當我們按 `time` 做 partition（例如每天一個分區），每個分區內的資料都是按時間順序插入，物理上極度有序。
- 此時在每個分區上建 BRIN（例如 `BRIN ON timestamp`），只需極小的索引就能快速跳過大量不相關的 block range，實現類似時序數據庫的效果。

流程圖：

```
原始大表 (時間跨度大，物理亂序)
      ↓ 按時間 partition
分區 p1 (day1, 物理有序)  ← BRIN 效果極佳
分區 p2 (day2, 物理有序)  ← BRIN 效果極佳
...
```

這樣既能利用 partition 做資料管理與淘汰，又能用 BRIN 在分區內高效過濾，是物聯網、日誌、監控等場景的經典設計。

**總結**：Leaf Page 是索引真正存放「數據與指標」的地方，理解它才能明白覆蓋索引、INCLUDE 的效益；而 Partition + BRIN 則是另一種繞開 B+ 樹、為時序數據極致壓縮索引大小的策略。

# 三、PostgreSQL BRIN 索引（Block Range INdex）

## 1 BRIN Index（Block Range INdex）

BRIN 是 page-level summary index。不 index 每行數據，而是按 data page 的物理存儲範圍（page range）來記錄，體積比 B-tree 小幾百甚至上千倍（可能從 GB 級降到 MB 甚至 KB 級）。每個 summary 存儲連續相鄰 page 的統計信息：`min(val), max(val), has null?, all null?`。

### I. 為什麼 BRIN 比 B-tree 更省空間？（圖解）

假設一張表，按時間順序插入，共 1 億行，每個 page 存 200 行，共 **500,000 個 data pages**。為 `time` 欄位分別建 B-tree 和 BRIN 索引來對比。

```
┌──────────────── 表的物理存儲（數據頁） ────────────────┐
│  Page 1  ... Page 128 │ Page 129 ... Page 256 │ ...  │
│  (200 rows)           │ (200 rows)            │      │
│  time: 00:00 ~ 00:05  │ time: 00:05 ~ 00:10   │      │
└───────────────────────┴───────────────────────┴──────┘
```

**B-tree 索引（為每一行建一個索引條目）**

B-tree 的每個 leaf page 會塞滿許多 (time, 行指針) 的條目，結構分多層（根、內部、葉子）。

```
         [根頁]
         /    \
    [內部頁] [內部頁]
     /   \      /   \
  [葉頁1] [葉頁2] ... [葉頁n]   ← 葉子頁之間雙向鏈結

每個葉子頁內：
┌───────────────────────────────┐
│ time=00:00:01 | row_ptr →    │
│ time=00:00:02 | row_ptr →    │
│ ...（共數百條）              │
└───────────────────────────────┘

總條目數 = 1 億條（因為一行一條）
索引大小 ≈ 1 億 × (鍵值 + 指針 + 頁開銷) → 數 GB 級
```

**BRIN 索引（為每「一組連續頁」建一條摘要）**

BRIN 不關心每一行，只把連續的 page range（例如每 128 個頁）做一次統計。每個 range 只存一行摘要。

```
BRIN 索引結構（類似一個極小的列表）：

┌─ Range 1 ─────────────────────────────────────────┐
│ 涵蓋頁: 1 ~ 128                                   │
│ min(time) = 2024-01-01 00:00:00                   │
│ max(time) = 2024-01-01 00:04:59                   │
│ has null? = false, all null? = false              │
│ 指針: 指向頁 1                                    │
└───────────────────────────────────────────────────┘
┌─ Range 2 ─────────────────────────────────────────┐
│ 涵蓋頁: 129 ~ 256                                 │
│ min(time) = 2024-01-01 00:05:00                   │
│ max(time) = 2024-01-01 00:09:59                   │
│ ...                                              │
└───────────────────────────────────────────────────┘
...（總共 500,000 頁 / 128 ≈ 3906 個 range）

總摘要數 ≈ 3906 條（每條約幾十字節）
索引大小 ≈ 3906 × 幾十字節 → 數百 KB，甚至更小
```

**直觀對比（放大看同一個頁範圍）**

```
表的物理頁（128 頁一組）
┌───────┬───────┬───────┬─────┬───────┐
│Page 1 │Page 2 │Page 3 │ ... │Page128│  共 25,600 行
└───────┴───────┴───────┴─────┴───────┘

B-tree 對這 25,600 行：
  建立 25,600 個葉子條目，佔用數十個葉子頁 + 上層導航頁

BRIN 對這 25,600 行：
  只建立 1 條摘要  (min, max, null flags, 指向 Page1)
```

BRIN 就是用 **1 條摘要** 代替 **2.5 萬條索引條目**，這就是體積縮小幾百倍甚至上千倍的根本原因。

**為什麼可以這樣？條件是什麼？**

- 物理順序必須和邏輯順序高度相關（即 `time` 相近的行在硬盤上也是連續存放的）。
- 查詢 `WHERE time = '某值'` 時，BRIN 只能告訴你「這個範圍的 min～max 有沒有包含該值」，無法像 B-tree 那樣精確定位到某一行。若命中了，還是要掃描整個範圍的頁（但可以跳過完全不符的範圍，這就是 lossy index）。
- 因此 BRIN 主要適合 **順序性好、數據量巨大、查詢多為範圍掃描** 的場景（如 IoT、日誌），正是「先分區再 BRIN」的完美舞臺。

**總結一句話**

> B-tree = 為每一行存一個指路牌（精確但體積大）  
> BRIN = 為每一大片連續頁存一張「統計小紙條」（粗糙但極小）

當數據物理上本身就是有序的，這種「小紙條」就足以快速跳過不需要的區塊，同時索引大小從 GB 級直接摔到 MB 甚至 KB 級。

- **適用場景**：海量、順序增長的時間序列數據（如 log 表）。
- **不適用場景**：頻繁的隨機點查。因為 BRIN 是有損 index，精確定位能力不如 B-tree。

BRIN 查詢 plan 中始終是 Bitmap Index Scan + Bitmap Heap Scan 的組合——因為 BRIN 本身返回 page range，直接填充 bitmap，天然對應 page-level 粒度。

> 補充（Senior Dev）：BRIN 有效的前提是物理存儲順序與 index column 的邏輯順序高度相關（correlation ≈ 1 或 -1）。可以用以下查詢檢查 correlation：
> ```sql
> SELECT correlation FROM pg_stats
> WHERE tablename = 'your_table' AND attname = 'your_column';
> ```
> 若 correlation 接近 0，說明數據在磁盤上隨機分佈，BRIN 會因為誤報率極高而幾乎退化為 Seq Scan。此時應先 `CLUSTER` 或 `pg_repack` 按該 column 重排表，再建 BRIN。
>
> `pages_per_range` 的取值權衡：預設 128 適合大多數場景。值越小 → summary 越多 → index 越大但誤報越少（適合頻繁點查）；值越大 → summary 越少 → index 極小但誤報越多（適合純大範圍掃描）。對於 IoT 場景，128~256 通常是 sweet spot。

---

> 來源：[digoal - PostgreSQL 物联网黑科技 — 瘦身几百倍的索引(BRIN index) (2016-04-14)](https://github.com/digoal/blog/blob/master/201604/20160414_01.md)

---

## 2 背景：BTREE 為何不適合 IoT 場景

IoT 場景特徵：海量數據流式寫入、查詢以範圍掃描為主（FIFO）。在此場景下 BTREE 有兩個致命問題：

**1. Index size 過大**

BTREE 逐 row 存儲 indexed column value + TID（row pointer），導致 index size 與 table size 接近甚至更大：

```
postgres=# \dt+ tab
                    List of relations
 Schema | Name | Type  |  Owner   |  Size   | Description
--------+------+-------+----------+---------+-------------
 public | tab  | table | postgres | 3438 MB |
(1 row)

postgres=# \di+ idx_tab_id
                           List of relations
 Schema |    Name    | Type  |  Owner   | Table |  Size   | Description
--------+------------+-------+----------+-------+---------+-------------
 public | idx_tab_id | index | postgres | tab   | 2125 MB |
(1 row)
```

4.1 GB 的 table，index 就佔了 2.5 GB。

**2. 寫入效能大幅下降**

```sql
CREATE UNLOGGED TABLE tab(id serial8, info text, crt_time timestamp);
CREATE INDEX idx_tab_id ON tab(id);
```

```sql
-- test.sql
INSERT INTO tab (info) SELECT '' FROM generate_series(1, 10000);
```

有 BTREE index，每秒入庫 **28.45 萬行**：

```
pgbench -M prepared -n -r -P 1 -f ./test.sql -c 48 -j 48 -T 100
tps = 28.453983 (excluding connections establishing)
```

無 index，每秒入庫 **66.88 萬行**：

```
postgres=# DROP INDEX idx_tab_id;
```

```
pgbench -M prepared -n -r -P 1 -f ./test.sql -c 48 -j 48 -T 100
tps = 66.880260 (excluding connections establishing)
```

BTREE index 讓寫入效能腰斬（28.5 vs 66.9 萬/s）。對流式入庫場景來說，這意味著需要 2 倍以上的硬體資源才能跟上寫入速度。

> 補充（Senior Dev）：這是 BTREE 的結構性代價。每次 INSERT 不僅要寫 table heap page，還要寫 index page，且 BTREE 是平衡樹結構，資料若非完全遞增插入會觸發 page split，進一步加劇 write amplification。在 NVMe SSD 上，BTREE 的 random write 仍是瓶頸，相比順序寫入（append-only heap）有數量級的差距。BRIN 正是針對這兩點設計的替代方案。

---

## 3 BRIN 原理

BRIN 不逐 row 存 index entry。它將 table 中連續的 N 個 data page 視為一個 block range（預設 `pages_per_range = 128`），對每個 block range 僅存一份統計摘要：

```
min(val), max(val), has null?, all null?, left block id, right block id
```

![BRIN Principle](../images/brin_principle.png)

若 table 有 10,000 個 page、`pages_per_range = 128`，BRIN 只需存 **79 份摘要**，而非逐 row 構建數百萬個 index entry。這解釋了為什麼 BRIN size 可從 GB 降到 MB/KB 級別。

查詢時的行為：
1. BRIN 掃描摘要，找到 min-max 與查詢條件重疊的 block range
2. 標記對應 page 到位圖（Bitmap Index Scan）
3. 按物理順序讀取 page，逐 row Recheck Cond（Bitmap Heap Scan）

BRIN 的有損性（只知哪個 page range 可能滿足，不知具體 row）決定了它**一定需要 Recheck**，也說明它不適合精確點查。

> [PG 10+] **autosummarize**：原文撰寫時期（9.5），BRIN 摘要需要手動 `VACUUM` 或 `brin_summarize_new_values()` 來更新。自 PG 10 起，`CREATE INDEX ... WITH (autosummarize = on)` 讓 autovacuum 自動更新 BRIN 摘要，對持續寫入的 table 不再需要排程手動重整。
>
> ```sql
> CREATE INDEX idx_brin ON tab USING BRIN (col)
>     WITH (pages_per_range = 128, autosummarize = on);
> ```
>
> [PG 14+] **multi-minmax**：原文時期每個 block range 只存一組 `(min, max)`。自 PG 14 起，`WITH (n_distinct_per_range = N)` 可在一個 block range 內存儲多組 min/max 值，大幅降低 value cluster 分散時的 false positive。例如 data 在 128 個 page 中出現多個不相鄰的 value cluster 時，舊版 BRIN 會用全域 min/max 涵蓋整個 range（包含大量不匹配的 gap），PG 14+ 則用多組 min/max 精確定位每個 cluster。
>
> ```sql
> CREATE INDEX idx_brin_mm ON tab USING BRIN (col)
>     WITH (pages_per_range = 128, n_distinct_per_range = 8);
> ```
>
> [PG 11+] **bloom operator class**：BRIN 預設只支援 min/max 運算（`<`, `<=`, `=`, `>=`, `>`），不支援多列複合條件中的精確等值。PG 11 引入 `brin_bloom` opclass，在每個 block range 存儲 bloom filter，使 BRIN 能處理 `=`、`IN` 查詢，且支援多列複合（`idx ON t USING brin (col1 bloom_ops, col2 bloom_ops)`）。bloom opclass 參數 `n_distinct_per_range`（與 multi-minmax 語法相同但語義不同）控制 bloom filter 大小，值越大 false positive 越低但 index 越大。
>
> ```sql
> CREATE INDEX idx_brin_bloom ON tab USING BRIN (status int4_bloom_ops)
>     WITH (pages_per_range = 128);
> ```

---

## 4 效能實測

### I. 寫入效能

BRIN index：

```sql
CREATE INDEX idx_tab_id ON tab USING BRIN (id) WITH (pages_per_range = 1);
```

```
pgbench -M prepared -n -r -P 1 -f ./test.sql -c 48 -j 48 -T 100
tps = 62.838701 (excluding connections establishing)
```

| 場景 | Insert TPS | 相對無 Index |
|------|-----------|-------------|
| No Index | **66.88 萬/s** | 100% |
| BRIN | **62.84 萬/s** | 94% |
| BTREE | 28.45 萬/s | 43% |

BRIN 寫入效能僅比無 index 低 6%，而 BTREE 損失 57%。

![Insert Performance](../images/brin_perf_insert.png)

### II. Index Size

Table 大小：4,163 MB

```
postgres=# \di+ idx_tab_btree_id
                              List of relations
 Schema |       Name       | Type  |  Owner   | Table |  Size   | Description
--------+------------------+-------+----------+-------+---------+-------------
 public | idx_tab_btree_id | index | postgres | tab   | 2491 MB |
(1 row)

postgres=# \di+ idx_tab_id
                           List of relations
 Schema |    Name    | Type  |  Owner   | Table |  Size   | Description
--------+------------+-------+----------+-------+---------+-------------
 public | idx_tab_id | index | postgres | tab   | 4608 kB |
(1 row)

postgres=# \dt+ tab
                    List of relations
 Schema | Name | Type  |  Owner   |  Size   | Description
--------+------+-------+----------+---------+-------------
 public | tab  | table | postgres | 4163 MB |
(1 row)
```

| Index Type | Size | 與 Table 比例 |
|-----------|------|--------------|
| BTREE | **2,491 MB** | 59.8% |
| BRIN | **4,608 KB** (4.6 MB) | 0.11% |

BRIN 比 BTREE 小 **540 倍**。4.6 MB 的 index 幾乎可以全部載入 L3 cache，scan 時幾乎零 disk I/O。

### III. 範圍查詢效能

> 註：原文中 `/*+ seqscan(tab) */` / `/*+ bitmapscan(...) */` / `/*+ indexscan(...) */` 為 `pg_hint_plan` extension 的 hint 語法，非 PG 內建。原生 PG 使用 `SET enable_seqscan = off` 等 GUC 控制 plan choice。

全表掃描（force Seq Scan）：

```
postgres=# /*+ seqscan(tab) */ EXPLAIN (ANALYZE, BUFFERS, TIMING, COSTS, VERBOSE)
SELECT count(*) FROM tab WHERE id BETWEEN 1 AND 100000;
                                                           QUERY PLAN
--------------------------------------------------------------------------------------------------------------------------------
 Aggregate  (cost=1891578.12..1891578.13 rows=1 width=0) (actual time=11353.057..11353.058 rows=1 loops=1)
   Output: count(*)
   Buffers: shared hit=133202
   ->  Seq Scan on public.tab  (cost=0.00..1891352.00 rows=90447 width=0) (actual time=1660.445..11345.123 rows=100000 loops=1)
         Output: id, info, crt_time
         Filter: ((tab.id >= 1) AND (tab.id <= 100000))
         Rows Removed by Filter: 117110000
         Buffers: shared hit=133202
 Planning time: 0.048 ms
 Execution time: 11353.080 ms
(10 rows)
```

BRIN index（Bitmap Heap Scan）：

```
postgres=# /*+ bitmapscan(tab idx_tab_id) */ EXPLAIN (ANALYZE, BUFFERS, TIMING, COSTS, VERBOSE)
SELECT count(*) FROM tab WHERE id BETWEEN 1 AND 100000;
                                                             QUERY PLAN
-------------------------------------------------------------------------------------------------------------------------------------
 Aggregate  (cost=70172.91..70172.92 rows=1 width=0) (actual time=63.735..63.735 rows=1 loops=1)
   Output: count(*)
   Buffers: shared hit=298
   ->  Bitmap Heap Scan on public.tab  (cost=1067.08..69946.79 rows=90447 width=0) (actual time=40.700..55.868 rows=100000 loops=1)
         Output: id, info, crt_time
         Recheck Cond: ((tab.id >= 1) AND (tab.id <= 100000))
         Rows Removed by Index Recheck: 893
         Heap Blocks: lossy=111
         Buffers: shared hit=298
         ->  Bitmap Index Scan on idx_tab_id  (cost=0.00..1044.47 rows=90447 width=0) (actual time=40.675..40.675 rows=1110 loops=1)
               Index Cond: ((tab.id >= 1) AND (tab.id <= 100000))
               Buffers: shared hit=187
 Planning time: 0.049 ms
 Execution time: 63.755 ms
(14 rows)
```

BTREE index：

```
postgres=# /*+ bitmapscan(tab idx_tab_btree_id) */ EXPLAIN (ANALYZE, BUFFERS, TIMING, COSTS, VERBOSE)
SELECT count(*) FROM tab WHERE id BETWEEN 1 AND 100000;
                                                                 QUERY PLAN
--------------------------------------------------------------------------------------------------------------------------------------------
 Aggregate  (cost=76817.88..76817.89 rows=1 width=0) (actual time=23.780..23.780 rows=1 loops=1)
   Output: count(*)
   Buffers: shared hit=181
   ->  Bitmap Heap Scan on public.tab  (cost=1118.87..76562.16 rows=102286 width=0) (actual time=6.569..15.950 rows=100000 loops=1)
         Output: id, info, crt_time
         Recheck Cond: ((tab.id >= 1) AND (tab.id <= 100000))
         Heap Blocks: exact=111
         Buffers: shared hit=181
         ->  Bitmap Index Scan on idx_tab_btree_id  (cost=0.00..1093.30 rows=102286 width=0) (actual time=6.530..6.530 rows=100000 loops=1)
               Index Cond: ((tab.id >= 1) AND (tab.id <= 100000))
               Buffers: shared hit=70
 Planning time: 0.099 ms
 Execution time: 23.798 ms
(13 rows)
```

| 策略 | Execution Time | Buffers |
|------|---------------|---------|
| Seq Scan | **11,353 ms** | 133,202 |
| BRIN | **64 ms** | 298 |
| BTREE | **24 ms** | 181 |

範圍查詢下 BRIN 與 BTREE 差距 40 ms（64 vs 24），但 Buffers 閱讀量僅差 117 個 page。在 disk I/O 場景（非 shared hit）中，BRIN 的 index 體積優勢更能體現，因為 BRIN 的摘要 page 幾乎一定在 cache 中。

關鍵觀察：BRIN plan 中 `Heap Blocks: lossy=111`，表示所有 page 都退化了。原因在此測試中 `pages_per_range = 1` 且 scan 結果跨多個 block range，Bitmap Index Scan 返回的是 block range 級別（有損），因此沒有 `exact=`。

### IV. 精確點查效能

全表掃描（force Seq Scan）：

```
postgres=# /*+ seqscan(tab) */ EXPLAIN (ANALYZE, BUFFERS, TIMING, COSTS, VERBOSE)
SELECT count(*) FROM tab WHERE id = 100000;
                                                      QUERY PLAN
-----------------------------------------------------------------------------------------------------------------------
 Aggregate  (cost=1598327.00..1598327.01 rows=1 width=0) (actual time=8297.589..8297.589 rows=1 loops=1)
   Output: count(*)
   Buffers: shared hit=133202
   ->  Seq Scan on public.tab  (cost=0.00..1598327.00 rows=2 width=0) (actual time=1221.359..8297.582 rows=1 loops=1)
         Output: id, info, crt_time
         Filter: (tab.id = 100000)
         Rows Removed by Filter: 117209999
         Buffers: shared hit=133202
 Planning time: 0.113 ms
 Execution time: 8297.619 ms
(10 rows)
```

BRIN index：

```
postgres=# /*+ bitmapscan(tab idx_tab_id) */ EXPLAIN (ANALYZE, BUFFERS, TIMING, COSTS, VERBOSE)
SELECT count(*) FROM tab WHERE id = 100000;
                                                          QUERY PLAN
-------------------------------------------------------------------------------------------------------------------------------
 Aggregate  (cost=142.04..142.05 rows=1 width=0) (actual time=38.498..38.498 rows=1 loops=1)
   Output: count(*)
   Buffers: shared hit=189
   ->  Bitmap Heap Scan on public.tab  (cost=140.01..142.04 rows=2 width=0) (actual time=38.432..38.495 rows=1 loops=1)
         Output: id, info, crt_time
         Recheck Cond: (tab.id = 100000)
         Rows Removed by Index Recheck: 1811
         Heap Blocks: lossy=2
         Buffers: shared hit=189
         ->  Bitmap Index Scan on idx_tab_id  (cost=0.00..140.01 rows=2 width=0) (actual time=38.321..38.321 rows=20 loops=1)
               Index Cond: (tab.id = 100000)
               Buffers: shared hit=187
 Planning time: 0.102 ms
 Execution time: 38.531 ms
(14 rows)
```

BTREE index（force Index Scan）：

```
postgres=# /*+ indexscan(tab idx_tab_btree_id) */ EXPLAIN (ANALYZE, BUFFERS, TIMING, COSTS, VERBOSE)
SELECT count(*) FROM tab WHERE id = 100000;
                                                            QUERY PLAN
-----------------------------------------------------------------------------------------------------------------------------------
 Aggregate  (cost=2.76..2.77 rows=1 width=0) (actual time=0.018..0.018 rows=1 loops=1)
   Output: count(*)
   Buffers: shared hit=4
   ->  Index Scan using idx_tab_btree_id on public.tab  (cost=0.44..2.76 rows=2 width=0) (actual time=0.015..0.016 rows=1 loops=1)
         Output: id, info, crt_time
         Index Cond: (tab.id = 100000)
         Buffers: shared hit=4
 Planning time: 0.049 ms
 Execution time: 0.036 ms
(9 rows)
```

| 策略 | Execution Time | Buffers |
|------|---------------|---------|
| Seq Scan | **8,297 ms** | 133,202 |
| BRIN | **39 ms** | 189 |
| BTREE | **0.036 ms** | 4 |

精確點查是 BRIN 的弱項。BTREE 直接定位到具體 row（Index Scan 只需 4 個 buffer），BRIN 需要掃完整個對應 block range 的所有 row（`Rows Removed by Index Recheck: 1811`），然後 Recheck 過濾。

> 補充（Senior Dev）：`pages_per_range = 1` 是極端配置（每個 page 一個摘要）。生產環境建議預設 `128`，此時 BRIN size 會進一步縮小（4.6 MB / 128 ≈ 36 KB），但精確點查的 Recheck 開銷也會等比放大。如果業務混雜少量點查 + 大量範圍掃描，可考慮 BRIN + 少量 partial BTREE index 的混合策略。

### V. 綜合性能對比圖

![Performance Summary 1](../images/brin_perf_query.png)

![Performance Summary 2](../images/brin_perf_summary.png)

---

## 5 適用場景與限制

| 適合 | 不適合 |
|------|--------|
| 海量、順序寫入的時序數據（IoT sensor、log、metrics） | 頻繁隨機點查（`WHERE id = X`） |
| 大範圍掃描（`BETWEEN`、`>=`、time range aggregation） | 資料物理順序與索引列不相關（如隨機 UUID） |
| Storage 受限環境（index size 是 BTREE 的 1/500+） | 頻繁 UPDATE / DELETE 導致 row 位置與 min/max 脫節 |

> 補充（Senior Dev）：BRIN 效能依賴資料的 **physical correlation**（索引列的值與物理存儲順序一致）。若資料按 `create_time` 順序插入，BRIN 效果極佳。但對於 `UPDATE` 頻繁的表，MVCC 產生的 new tuple 可能寫到不同 page，逐漸破壞 physical correlation。可通過 `CLUSTER` 或 `pg_repack` 定期重整，或搭配 Partition 來維持 locality。

---

## 6 Oracle Exadata Storage Index

Oracle 有類似機制，但僅限 Exadata 一體機（Storage Index 在 storage cell 層面，對 SQL 層透明）。PostgreSQL BRIN 是開源免費、任何硬體均可用的對等方案。

Oracle Storage Index 連結：[https://docs.oracle.com/cd/E50790_01/doc/doc.121/e50471/concepts.htm#SAGUG20984](https://docs.oracle.com/cd/E50790_01/doc/doc.121/e50471/concepts.htm#SAGUG20984)

---

> [PG 11+] **multi-column BRIN**：PG 11 起 BRIN 支援不同 column 使用不同 opclass，實現真正的複合 BRIN index：
> ```sql
> CREATE INDEX idx_brin_multi ON tab USING BRIN (
>     create_time timestamptz_minmax_ops,
>     status int4_bloom_ops
> ) WITH (pages_per_range = 128, autosummarize = on);
> ```
>
> [PG 13+] **parallel BRIN scan**：PG 13 起 BRIN index scan 支援 parallel query，對於大範圍掃描 + 多 CPU 環境可進一步加速。
>
> [PG 版本差異] 原文 EXPLAIN 輸出來自 PG 9.5，與最新版本有幾處格式差異：
> - `Planning time`（小寫 t）在 PG 13 改為 `Planning Time`（大寫 T）
> - PG 12+ 新增 `JIT` 資訊行（若 JIT 啟用）、PG 13+ 新增 `Memory` 用量、PG 14+ 新增 `Worker` 資訊
>
> 各項核心指標（`Buffers`、`Heap Blocks`、`Rows Removed by Index Recheck`）格式不變，原文 benchmark 結論在最新版本仍然有效。

## 7 小結

1. BRIN 針對 IoT 場景：流式寫入、範圍查詢。對寫入影響微乎其微，index size 比 BTREE 小數百倍
2. 範圍查詢效能與 BTREE 差距在毫秒級，但 index 體積節省巨大
3. 精確點查不如 BTREE（差距 1000x），不應用作 point-lookup 的主 index
4. 結合 JSON / GiST 等功能，適合 IoT 混合分析場景
5. 德哥原文結論：DBA 應抓準各類 index 特性，匹配到合適場景（千里馬與伯樂）

# 四、PostgreSQL Bloom Index — 一個索引支撐任意 Column 組合查詢

## 1 Bloom Index 原理與執行流程

作爲開發者，在設計數據訪問層（DAL）時，面對運營後臺那種幾十個字段任意組合的篩選需求，幾乎是一場索引噩夢。

爲了讓你更直觀地理解，下面是傳統 btree 索引方案在處理"多字段組合查詢"時面臨的窘境：

```mermaid
---
title: 傳統多字段組合查詢的索引爆炸困境
---
flowchart TD
    A[用戶發起查詢<br>可選條件：c1, c2, c3] --> B{規劃器評估索引}
    B -->|方案1| C[全表掃描]
    C --> D[❌ 數據量一大查詢就崩潰]

    B -->|方案2| E[單個聯合索引<br>（如 idx_c1_c2_c3）]
    E --> F{查詢 WHERE c2 = ?}
    F --> G[❌ 未命中前導列，索引失效]

    B -->|方案3| H[窮舉所有組合<br>（建立 2^N 個獨立索引）]
    H --> I[❌ 存儲爆炸、寫入性能災難]
```

PostgreSQL 提供的 Bloom Index 是解決此場景的利器。它專爲 WHERE c1=v1 AND c5=v5 AND ... 這類等值組合查詢設計，用極小的存儲成本實現高效過濾。

💡 一句話原理：一個"有損壓縮"的黑名單

你可以把 Bloom 索引理解成給每一行數據生成一個"有損壓縮的指紋"（Signature）。查詢時，先用指紋快速排除絕對不相關的行，再去表中精確覈對。

```mermaid
---
title: PostgreSQL Bloom Index 執行流程
---
flowchart TD
    subgraph A[寫入：生成簽名]
        direction LR
        A1[索引列值 c1,c2,…,cN] --> A2[多個哈希函數] --> A3[Signature<br>變長bit串]
    end

    subgraph B[查詢：快速篩查]
        direction LR
        B1[查詢條件 WHERE c1=v1 AND c5=v5] --> B2[計算查詢指紋] --> B3[與索引簽名做位圖掃描]
    end

    B3 --> C{匹配?}
    C -->|❌ 不匹配| D[丟棄，一定不存在<br>零損耗]
    C -->|⚠️ 匹配| E[回表獲取實際行]
    E --> F[Recheck 過濾誤報]
    F --> G[返回最終結果]
```

⚠️ 核心特性：精準的"不在"和模糊的"在"

它的"有損"特性決定了兩個關鍵行爲：

- 絕對可靠 (False Negative = 0%)：如果指紋沒命中，數據庫中絕對不存在符合條件的行。
- 可能誤報 (False Positive > 0%)：如果指紋命中，數據可能存在，也可能不存在。這就是所謂的"假陽性"。

---

## 2 實戰指南：DDL 與參數配置

首先確認 bloom 擴展已啓用：

```sql
-- 作爲標準擴展，通常一句 SQL 即可啓用
CREATE EXTENSION IF NOT EXISTS bloom;
```

**1. 索引 DDL**

和創建普通索引類似，但必須在 WITH 子句中配置參數來控制精度和大小。

```sql
CREATE INDEX idx_user_bloom ON users
USING bloom (
    first_name, last_name, email, department, status
)
WITH (
    length = 100,         -- 總指紋長度（bit）
    col1 = 5,             -- first_name 用5位
    col2 = 5,             -- last_name 用5位
    col3 = 6,             -- email 用6位
    col4 = 3,             -- department 用3位
    col5 = 2              -- status 用2位
);
```

**2. 參數完全解析**

WITH (...) 子句裏可以配置的參數如下：

| 參數 | 默認值 | 最大值 | 說明 |
|------|--------|--------|------|
| `length` | 80 | 4096 | 每行數據的簽名長度（bit），會向上取整爲16的倍數。控制全局精度。 |
| `col1 ~ col32` | 2 | 4095 | 分別爲第1到第32個索引列分配的bit數。數值越大，該列的查詢越精確。 |

**3. 精度與空間成本**

- Signature Length（指紋長度）：全局精度調節旋鈕。`length` 越大，指紋碰撞概率越低，查詢越精確，索引體積越大。
- Column Bits（字段位數）：局部精度調節旋鈕。爲高區分度字段（如 email）分配更多位，低區分度字段（如 status）少分配些。

**4. 匹配驗證：用 EXPLAIN 確認命中**

務必通過 EXPLAIN (ANALYZE, BUFFERS) 確認查詢確實用上了索引。

```sql
-- PostgreSQL 16 下，當查詢條件匹配索引時，計劃會顯示 Bitmap Index Scan
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM users
WHERE first_name = 'Bruce' AND department = 'Engineering';
```

- 重點觀察 `Rows Removed by Index Recheck`：此行數值代表被 Bloom 索引誤報（False Positive）的行數。數值越大，說明索引精度越低，需調整參數。
- `Filter` 行：如果查詢中有未包含在 Bloom 索引中的列，會顯示爲 `Filter` 步驟，此過濾在回表後進行。

---

## 3 調優與場景選擇

僅僅"能用"還不夠，"用好"Bloom索引纔是區分普通開發者和資深工程師的分水嶺。

**1. 參數優化建議**

- 壓榨性能：如果磁盤空間充裕且查詢耗時非常敏感，可將 `length` 增至 200 甚至 400。將高基數字段的 `colN` 值設爲 4 或 8。
- 節省空間：如果磁盤資源緊張，可適當降低 `length`（如 40 或 60），並將低基數字段的 `colN` 保持默認或降低。不推薦的寫法：`length=100, col1=20`。應通過增加 `length` 來提高精度，而非讓個別字段擠壓其他字段的bit位。

**2. 調優實戰策略：應對複雜查詢**

對於帶一個非索引列的查詢：

```sql
EXPLAIN (ANALYZE)
SELECT * FROM users
WHERE first_name = 'Bruce' AND department = 'Engineering' AND signup_date > '2023-01-01';
```

查看 EXPLAIN 輸出中的 `Filter` 行。如果 `signup_date` 的過濾性不強，此計劃可以接受；如果它過濾掉了大量數據，應考慮爲其創建獨立索引或調整查詢邏輯。

**3. Bloom vs. 其他索引：如何選擇？**

| 索引類型 | 典型體積 | 等值查詢 (`=`) | 範圍查詢 (`>`, `<`) | ORDER BY / MIN / MAX | 任意組合查詢 |
|----------|----------|:---:|:---:|:---:|:---:|
| Bloom | 極小 (MB級) | ✔️ | ❌ | ❌ | ✔️ (極佳) |
| 多個 B-tree | 巨大 (GB級) | ✔️ | ✔️ | ✔️ | ✔️ (磁盤空間與寫入壓力巨大) |
| 聯合 B-tree | 大 (GB級) | ✔️ | ✔️ | ✔️ | ❌ (受限於最左前綴) |
| GIN | 大 | ✔️ | ❌ | ❌ | ✔️ |
| BRIN | 極小 (KB級) | ✔️ (弱) | ✔️ | ❌ | ❌ |

決策指南：

- 海量日誌/事件表 → BRIN 索引（如果數據與物理存儲順序相關）。
- JSONB、數組、全文搜索 → GIN 索引。
- 需要保證排序或唯一性 → B-tree 索引。
- ✅ 最適合Bloom的場景：
  - 表有大量字段，且查詢條件組合無法預測。
  - 僅需等值查詢（`=` 或 `IN`）。
  - 對存儲成本極其敏感。

**4. 實現細節與演進：從 PG 9.6 到 PG 16**

- PG 9.6：Bloom索引誕生，引入 `length` 參數。
- PG 10 - 12：增強了並行查詢能力，對Bloom索引的 Bitmap Heap Scan 階段有顯著提升。
- PG 13 - 14：支持 parallel index build，可以並行創建Bloom索引。
- PG 15 - 16：優化了優化器對Bloom索引的成本估算模型，使計劃選擇更準確。`length` 參數上限提升至 4096 bits。

---

## 4 總結

PostgreSQL Bloom Index 是應對"多字段任意組合查詢"場景的利器，它以微小的磁盤空間和極高的寫入效率，換取了查詢性能的極大飛躍。Bloom Index 的定位是 B-tree 等傳統索引的強力補充，而非替代品。 在一個成熟的數據模型中，完全可以根據業務場景，將 Bloom 與其他索引類型組合使用，以達成空間與性能的完美平衡。

如果你對文中提到的 GIN 或 BRIN 索引的細節，或者如何在一個複雜項目中組合使用多種索引感興趣，我可以爲你展開講講。


# 五、PostgreSQL 模糊查詢索引全景：GIN / GiST / SP-GiST / RUM 原理與選擇

## 1 模糊查詢的索引困局

| 查詢模式 | B-tree | GIN (pg_trgm) | GiST (pg_trgm) | RUM |
|---------|--------|---------------|----------------|-----|
| `col = 'abc'` | ✅ | ❌ | ❌ | ❌ |
| `col LIKE 'abc%'`（後模糊） | ✅（Index Scan） | — | — | — |
| `col LIKE '%abc'`（前模糊） | ✅（reverse index） | — | — | — |
| `col LIKE '%abc%'`（前後模糊） | ❌ Seq Scan | ✅ Bitmap | ✅ Index Scan | ✅ Index Scan |
| `col ~ 'abc.*def'`（regex） | ❌ | ✅ | ✅ | ❌ |
| `col LIKE '%abc%' ORDER BY ts` | ❌ | ❌ | ❌ | ✅ |
| 短 pattern（1~2 字元） | ❌ | ⚠️ 大量 false match | ⚠️ 大量 false match | — |

---

## 2 GIN Index（Generalized Inverted Index）

### I. 結構

將 column 中的每個 element（token / trigram / array element）取出，存入樹狀結構（類似 B+Tree：key + TID list）。高頻值的 TID list 存到獨立的 posting list page 中。

![GIN Detail](../images/gin_structure_detail.jpg)

### II. Fast Update（GIN 寫入優化）

插入/更新時可能涉及大量 element 變更，GIN 使用 pending list buffer（類似 MySQL 索引組織表）先緩衝寫入，再定期合併到主樹：

![GIN Complete](../images/gin_complete_structure.jpg)

查詢不會堵塞合併——可直接查詢 pending list + 主樹。buffer 大小由 `gin_pending_list_limit`（預設 4MB）控制。

### III. 核心限制

- **只支援 Bitmap Index Scan**：收集所有匹配 TID → 排序 → Heap Scan。即使 `LIMIT 1`，仍需排序全部 TID。
- **不存位置資訊**：無法支援 index-level ranking、phrase search（`'a' <-> 'b'`）、或複合欄位排序（tsvector + timestamp）。

### IV. 適用場景

- 大量資料的 array / JSONB / tsvector 索引
- pg_trgm 模糊查詢（中間結果少時最佳）
- FTS 索引（不要求 ranking / phrase search）

---

## 3 GiST Index（Generalized Search Tree）

### I. 結構：聚集（Clustering）為核心

以 range type 為例，每條記錄對應一個線段（範圍）：

![GiST Ranges](../images/gist_ranges.jpg)

GiST 將數據聚集到 group（類似 K-Means），每個 group 對應一個 index page：

![GiST Clustering](../images/gist_clustering.jpg)

二級結構：上級存下級 index page 的範圍 union，搜尋時先過濾上級再檢查下級：

![GiST Two-Level](../images/gist_two_level.jpg)

### II. 關鍵抽象層：PickSplit / Choose

GiST 的性能取決於自定義 data type 的 `Picksplit` 和 `Choose` function 如何聚集 key。這是 PG 開放索引介面的核心價值——你定義 data type 時自己實作聚集演算法。

```
Performance depends on how well the user-defined
Picksplit and Choose functions can group keys
```

### III. GiST vs GIN 對比

| 維度 | GIN | GiST |
|------|-----|------|
| Index Scan | ❌（僅 Bitmap） | ✅（可 Index Scan） |
| 寫入 | 快（pending list） | 慢（即時 insert into tree） |
| 體積 | 小 | 大（~1.7x of GIN） |
| LIMIT 查詢 | 慢 | 快 |
| 支援 range / geometry | ❌ | ✅ |
| 支援 fuzzy search | ✅（pg_trgm） | ✅（pg_trgm） |

---

## 4 SP-GiST（Space-Partitioned GiST）

### I. 結構

- **Nodes 無交叉**：不同 index page 的內容互不重疊（vs GiST 節點有重疊）
- **Index 深度可變**：根據數據分佈深度可深可淺
- **一個 physical page 可含多個 nodes**

![SP-GiST QuadTree](../images/spgist_quadtree.jpg)

### II. 支援檢索類型

- **Kd-tree**：點查（points only；shape 會重疊不適用）
- **Prefix tree（Trie）**：text preifx 搜尋

### III. 適用場景

- 地理點查詢（`point` type）
- 網絡路由（`inet` / `cidr`）
- Prefix 搜尋

---

## 5 RUM Index（RUM Access Method）

### I. RUM 解決了 GIN 的哪些限制

GIN 不存 token 位置資訊，導致無法支援索引級的：

1. **Ranking**：需回 Heap 讀取完整 tsvector 計算 `ts_rank`，CPU 成本高
2. **Phrase search**：`'a' <-> 'b'`（a 和 b 相鄰出現）無法在 index 層判斷
3. **複合欄位排序**：如 `tsvector + timestamp` 雙欄位複合查詢，必須回表讀 timestamp

RUM 在 posting tree 中**存入附加資訊**（position / timestamp / 自訂 fields），解決以上三點：

![RUM Structure](../images/rum_structure.png)

### II. RUM trade-off

- Index 體積比 GIN 大（存更多資訊）
- Insert / update 比 GIN 慢（更多 WAL / 更多資料結構維護）
- **但查詢複雜度高的場景（phrase search / ranking / 雙欄位排序）是唯一選擇**

---

## 6 Benchmark：GIN vs GiST vs RUM（100 萬 row，md5 string）

### I. 測試參數

```sql
CREATE TABLE test (id INT, info TEXT);
INSERT INTO test SELECT id, md5(random()::text) FROM generate_series(1, 1000000) t(id);

-- Index size comparison
-- GIN:  102 MB,  build 19s
-- GiST: 177 MB,  build 31s
-- RUM:  86 MB,   build 133s
```

String → tsvector / tsquery 轉換函數（給 RUM 用）：

```sql
-- string → tsvector（帶 position）
CREATE OR REPLACE FUNCTION string_to_tsvector(v TEXT) RETURNS tsvector AS $$
DECLARE
  x INT := 1;
  res TEXT := '';
  i TEXT;
BEGIN
  FOR i IN SELECT regexp_split_to_table(v, '')
  LOOP
    res := res || ' ' || chr(92) || i || ':' || x;
    x := x + 1;
  END LOOP;
  RETURN res::tsvector;
END;
$$ LANGUAGE plpgsql STRICT IMMUTABLE;

-- string → tsquery（帶 <-> adjacent）
CREATE OR REPLACE FUNCTION string_to_tsquery(v TEXT) RETURNS tsquery AS $$
DECLARE
  x INT := 1;
  res TEXT := '';
  i TEXT;
BEGIN
  FOR i IN SELECT regexp_split_to_table(v, '')
  LOOP
    IF x > 1 THEN
      res := res || ' <-> ' || chr(92) || i;
    ELSE
      res := chr(92) || i;
    END IF;
    x := x + 1;
  END LOOP;
  RETURN res::tsquery;
END;
$$ LANGUAGE plpgsql STRICT IMMUTABLE;
```

### II. Test Case 1：中間結果大（`~ 'a'`，873K row，無 LIMIT）

| Index | Plan | Time |
|-------|------|------|
| **GiST** | Index Scan using trgm_idx2 | **1,692 ms** |
| **RUM** | Index Scan using rum_idx1 | 624 ms |
| **GIN** | Bitmap Heap Scan on trgm_idx1 | 3,426 ms |

GiST Index Scan 比 GIN Bitmap Scan 快 2x。RUM 最快但需轉換 tsquery overhead。

### III. Test Case 2：中間結果大 + LIMIT 100

| Index | Plan | Time |
|-------|------|------|
| **GiST** | Index Scan using trgm_idx2 | **0.68 ms** |
| **RUM** | Index Scan using rum_idx1 | 303 ms |
| **GIN** | Bitmap Heap Scan | 2,404 ms |

GiST 碾壓 GIN（由於 GIN 必須排序全部 1M TID）。GiST 0.68ms vs GIN 2,404ms = **3,500x 差距**。RUM 慢於 GiST 因 tsquery overhead。

### IV. Test Case 3：中間結果小（`~ '2e9a2c'`，2 row）

| Index | Plan | Time |
|-------|------|------|
| **GIN** | Bitmap Heap Scan | **2.54 ms** |
| **GiST** | Index Scan | 294 ms |
| **RUM** | Index Scan | 864 ms |

GIN 碾壓 GiST（TID 排序量極少（2 TID）→ Bitmap overhead 可忽略）。GIN 2.5ms vs GiST 294ms = **115x 差距**。

---

## 7 選擇決策：pg_hint_plan + Explain-Based

```sql
ALTER TABLE test ALTER COLUMN info SET STATISTICS 1000;
VACUUM ANALYZE test;

-- 使用 pg_hint_plan 分別強制三個 index，比較 estimated rows
EXPLAIN /*+ BitmapScan(test trgm_idx1) */ SELECT * FROM test WHERE info ~ '2e9';
--  Bitmap Heap Scan on test  (cost=59.30..5079.87 rows=7006 width=37)

EXPLAIN /*+ IndexScan(test trgm_idx2) */ SELECT * FROM test WHERE info ~ '2e9';
--  Index Scan using trgm_idx2  (cost=... rows=7006 width=37)
```

**決策規則**：

| 中間結果 rows（estimated） | 有 LIMIT / 游標？ | 選擇 |
|--------------------------|-------------------|------|
| < 數百 | — | **GIN** |
| 數百 ~ 數萬 | — | **GiST** |
| 任何 | 有 `LIMIT` 小值 | **GiST** |
| 需要 ranking / phrase / 雙欄位 | — | **RUM** |

---

## 8 PG 索引方法全覽（截至 PG 18）

| Index Method | 簡述 | 支援 scan 模式 |
|-------------|------|---------------|
| **B-tree** | 通用首選、排序 | Index Scan / Bitmap Scan |
| **Hash** | 純等值 `=` | Index Scan |
| **GIN** | 倒排索引（array/JSONB/tsvector） | 僅 Bitmap Scan |
| **GiST** | 聚集樹（range/geometry/pg_trgm） | Index Scan / Bitmap Scan |
| **SP-GiST** | 空間分割（point/prefix） | Index Scan |
| **BRIN** | 塊範圍摘要 | Bitmap Scan |
| **RUM** | GIN + position/timestamp | Index Scan / Bitmap Scan |
| **Bloom** | 多列任意組合過濾 | Bitmap Scan |

PG 開放索引介面（Index Access Method API）允許自訂新的 index method，如上表所有非 B-tree 索引均透過此 API 實作。

---

## 9 版本演進

| 功能 | 版本 | 說明 |
|------|------|------|
| GIN fastupdate | PG 9.4 | pending list buffer 機制 |
| `gin_pending_list_limit` | PG 9.5 | 可控 pending list 大小 |
| pg_trgm GiST/GIN regex support | PG 9.3+ | 正則表達式索引加速 |
| RUM extension | PG 9.6+ | position-aware GIN |
| Bloom index | PG 9.6 | 多列組合過濾 |
| GIN parallel builds | PG 10+ | 多 CPU 並行建立 GIN |
| GiST/GIN covering (INCLUDE) | PG 11+ | covering index on GiST/GIN |
| `pg_trgm` strict_word_similarity | PG 11+ | 單詞級相似度 |
| BRIN multi-minmax | PG 14+ | 多列 BRIN |
| `btree_gin` / `btree_gist` improvements | PG 15+ | composite 效能提升 |
| GIN index cleanup tuning | PG 16+ | 更精細 vacuum 控制 |

## 10 參考

- [德哥: pg_trgm 模糊查詢](https://github.com/digoal/blog/blob/master/201605/20160506_02.md)
- [德哥: RUM 全文檢索加速](https://github.com/digoal/blog/blob/master/201610/20161019_01.md)
- [德哥: 正則與相似度](https://github.com/digoal/blog/blob/master/201611/20161118_01.md)
- [德哥: pg_hint_plan 用法](https://github.com/digoal/blog/blob/master/201604/20160401_01.md)


# 六、PostgreSQL GIN Index + LIMIT 慢的原因與因應策略

## 1 GIN Index 結構回顧

GIN（Generalized Inverted Index）是倒排索引：每個 element（陣列元素 / tsvector lexeme / JSONB key）對應一組 row number（TID list），而非 B-tree 那樣一行一個 index entry。

![GIN Index Structure](../images/gin_structure.jpeg)

對高頻 element（如陣列中反覆出現的 `1`），TID 列表會被組織成 posting list（而非一個一個 pointer），再由 index 指向該 list。

![GIN Element → Row List](../images/gin_element_list.jpeg)

---

## 2 核心問題：為什麼 GIN + LIMIT 慢？

GIN Index **只支援 Bitmap Index Scan**（不支援 plain Index Scan）。Bitmap Index Scan 的流程：

```
BitMap Index Scan → 收集所有匹配 TID → 排序 TID（按 page 順序）
  → BitMap Heap Scan → 依序訪問對應 data page → Recheck → 返回 row
```

**關鍵**：排序所有匹配 TID 是無法跳過的步驟。即使你寫了 `LIMIT 1`，GIN Index Scan 仍然會收集**所有**匹配該條件的 TID，全部排序後才去 Heap 取資料。

### I. 實驗

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

## 3 為什麼 GIN 只支援 Bitmap Scan — 設計取捨

GIN 的場景本質是**多對多**：一個 element 對應多行多 page，一個查詢可能匹配成千上萬個 page。若不用 Bitmap Scan 逐行跳轉，會產生恐怖的 random I/O。

Bitmap Scan 收集所有 TID → 排序的設計，反而是為了：

1. **Page deduplication**：同一 page 內有多行匹配 → 只讀一次，不重複讀取
2. **Sequential page reading**：TID 按 page 排序後順序讀取 → 最大化 sequential I/O

但當 `LIMIT` 很小（如 `LIMIT 1` 或 `LIMIT 10`）且 matched row 很多時，Bitmap Scan 的排序成本成為主導，而非 I/O 節省。

> 補充（Senior Dev）：這是一個經典的 **optimizer blind spot**：GIN 的 cost model 假設「matched row 不多」或「最昂貴的是 I/O 而非排序」。當 matched row 極多但只取少量時（如 `&& ARRAY[1] LIMIT 1`，但 `1` 出現在 90% 的 row 中），排序成本遠超實際需要。PG 至今（18）未引入 GIN Index Scan，社群討論過但實作複雜度極高（需保證返回順序與 Bitmap dedup 的正確性）。

---

## 4 btree_gin：多列 GIN 複合索引

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

## 5 解法矩陣

### I. gin_fuzzy_search_limit — 隨機採樣軟上限

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

### II. 改用 B-tree（若條件不依賴陣列）

```sql
-- 若 info 能標準化為單值 → 用 B-tree
CREATE INDEX idx_t3_info_btree ON t3 (info_element);
SELECT * FROM t3 WHERE info_element = 1 LIMIT 1;  -- Index Scan, 極快
```

### III. 改用 GiST（某些場景）

GiST 索引支援 plain Index Scan（非只有 Bitmap Scan）。對陣列、全文檢索、幾何等場景可選 GiST。但 GiST 體積比 GIN 大，寫入比 GIN 慢。取捨：

| 維度 | GIN | GiST |
|------|-----|------|
| Index Scan | ❌（僅 Bitmap） | ✅ |
| 寫入速度 | 快（pending list 機制） | 慢 |
| 查詢速度（大量匹配） | 快（Bitmap dedup） | 較慢 |
| Index 體積 | 小 | 大 |
| LIMIT 小結果 | 慢 | 快（可走 Index Scan） |

### IV. 改寫查詢（限制輸入規模）

```sql
-- 原寫法（BAD）：10,000 row 全排序
SELECT * FROM t3 WHERE info && ARRAY[1] LIMIT 1;

-- 改寫：先限制到最小可能的 pool
SELECT * FROM t3
WHERE id IN (SELECT unnest(info) FROM ...)  -- 先縮範圍
  AND info && ARRAY[1]
LIMIT 1;
```

### V. ORDER BY 配合 GIN Index（PG 16+）

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

## 6 版本演進

| 功能 | 版本 | 說明 |
|------|------|------|
| `gin_fuzzy_search_limit` | PG 9.4+ | 軟上限隨機採樣 |
| `gin_pending_list_limit` | PG 9.5+ | 寫入優化（控制 pending list 大小） |
| Parallel GIN Index Scan | PG 10 | 多 CPU 並行排序 TID |
| GIN fastupdate 改善 | PG 12 | 降低 pending list 合併頻率 |
| btree_gin 改善 | PG 14 | 複合 index 效能提升 |
| GIN scan memory optimization | PG 16 | 降低大結果集的排序 memory 壓力 |

## 7 參考

- [PostgreSQL Bitmap Scan IO 放大原理與優化](https://github.com/digoal/blog/blob/master/201801/20180119_03.md)
- [PostgreSQL GIN Tips](https://www.postgresql.org/docs/devel/static/gin-tips.html)


# 七、PostgreSQL RUM Index — 全文檢索的非 Bitmap Index Scan 與相似度排序

GIN 是 PostgreSQL 全文檢索的主力 index type，但它有兩個瓶頸：
1. 查詢使用 Bitmap Heap Scan，結果集需要 SORT（尤其是 `ORDER BY` + `OFFSET` 場景）
2. 不支援文本相似度排序 (`ORDER BY relevance`)

RUM（**R**UM access method，比 GIN 多存 **U**niversal **M**etadata）解決了這兩個問題。

---

## 1 問題場景：逗號分隔字串的全文檢索反模式

常見錯誤用法：逗號分隔元素存為單一 `text` column，再用 `LIKE '%1%' OR LIKE '%502%'` 搜尋。

```sql
CREATE TABLE test (c1 TEXT);
INSERT INTO test VALUES ('1,100,2331,344,502,...');
SELECT * FROM test WHERE c1 LIKE '%1%' OR c1 LIKE '%502%' AND c1 LIKE '%2331%';
-- 全表掃描，無法用 index
```

### I. 正確做法 1：Array + GIN

```sql
CREATE TABLE arr_test (c1 INT[]);
CREATE INDEX idx_arr_test ON arr_test USING GIN (c1);

-- 查詢
SELECT * FROM arr_test WHERE c1 && ARRAY[1, 2];
```

### II. 正確做法 2：tsvector + GIN

```sql
CREATE TABLE gin_test (c1 TSVECTOR);
CREATE INDEX idx_gin_test ON gin_test USING GIN (c1);

-- 查詢
SELECT * FROM gin_test WHERE c1 @@ to_tsquery('english', '1 | 2');
```

### III. 正確做法 3：tsvector + RUM

```sql
CREATE TABLE rum_test (c1 TSVECTOR);
CREATE INDEX rumidx ON rum_test USING rum (c1 rum_tsvector_ops);

-- 查詢
SELECT * FROM rum_test WHERE c1 @@ to_tsquery('english', '1 | 2');
```

---

## 2 效能基準（10M row / 每 row 100 element = 1B total token）

測試數據：每 row 含 100 個 0-100000 範圍的隨機數字。查詢搜尋 token `1` 或 `2`（命中約 19,900 row / 10M）。

```sql
INSERT INTO rum_test
SELECT to_tsvector(string_agg(c1::text, ','))
FROM (SELECT (100000 * random())::int FROM generate_series(1,100)) t(c1);

SET maintenance_work_mem = '64GB';
CREATE INDEX rumidx ON rum_test USING rum (c1 rum_tsvector_ops);
```

### I. 1 無 ORDER BY 查詢

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

### II. 2 有 ORDER BY + OFFSET 查詢（關鍵差異）

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

## 3 RUM 的相似度排序（`<=>` operator）

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

### I. 相似度排序查詢（RUM 的核心 killer feature）

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

## 4 方案選擇矩陣

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

## 5 參考

1. [RUM GitHub — Postgres Professional](https://github.com/postgrespro/rum)
2. [德哥：RUM 索引測試詳細](https://yq.aliyun.com/articles/59212)
3. [pg_jieba](https://github.com/jaiminpan/pg_jieba) / [zhparser](https://github.com/amutu/zhparser)


# 附錄、索引選擇速查表

> 以下為 PostgreSQL 各索引類型的場景對照與 SQL Server 對應概念，適合作為快速查閱。

### I. 針對 create_time 類型字段的選擇

| Index | 適用場景 | 優勢 | 劣勢 |
|-------|----------|------|------|
| **B-tree** | 通用最普適、最穩妥 | 支援所有比較運算、排序 | 體積大（大表可達 GB） |
| **BRIN** | 海量順序數據 + 大範圍掃描 | 體積極小（MB/KB），寫入效能接近無 index | 有損，不適合精確點查 |
| **Hash** | 純等值查詢 `WHERE col = X` | 結構精煉，理論比 B-tree 更輕更快 | 完全不支援範圍查詢（`>`, `<`, `BETWEEN`）和排序 |
| **GIN** | 多列複合場景（搭配 `btree_gin` extension） | 可在 GIN 中混用 JSONB 等複雜列和時間列 | 不直接加速單列時間查詢 |
| **GiST** | 多列複合場景（搭配 `btree_gist` extension） | 可同時包含全文檢索或空間列與時間列的複合 index | 不直接加速單列時間查詢 |

SP-GiST 不推薦作為普通單列時間字段的首選。

> 補充（Senior Dev）：Hash index 在 PG 10 之前不寫 WAL，crash 後需要 REINDEX，因此在 PG 10+ 才算生產可用。另外 Hash index 不支援 UNIQUE constraint，如果需要唯一約束只能用 B-tree。
>
> GIN index 的 `fastupdate` 參數（預設 on）：pending list 用於緩衝寫入（類似 buffer pool），達到 `gin_pending_list_limit` 時才合併到主 index。寫入頻繁時 pending list 過大會拖慢查詢（查詢需要掃 pending list + 主 index），此時可調小 `gin_pending_list_limit` 或手動 `VACUUM`。

### II. SQL Server 對照

SQL Server 裡沒有叫 "Bitmap Heap Scan" 的 operator，但查詢引擎中有一個完全對應的概念：**Bitmap Filter**，通常在 execution plan 中看到 `Bitmap Create` + `Bitmap Probe`。

> 補充（Senior Dev）：SQL Server 的 bitmap 是 hash-based（基於 row hash 值映射到 bitmap），與 PostgreSQL 的 page-based bitmap 不同。PG 的 bitmap 直接對應物理 page 號，所以天然支援物理順序讀取；SQL Server 的 hash bitmap 需要額外的 sort/merge 步驟才能達到類似效果。

| PostgreSQL | SQL Server | 說明 |
|------------|------------|------|
| Bitmap Index Scan | Index Scan + Bitmap Create | 掃描 index，構建 bitmap |
| BitmapAnd / BitmapOr | 位圖合併（內部） | 支援多 index 掃描結果的 bitmap 邏輯合併 |
| Bitmap Heap Scan | Bitmap Probe + Clustered Index Scan / Heap Scan | 用 bitmap 探測數據 row，按物理順序讀取 |
| Recheck Cond | 顯式 Filter | 讀取 page 後再次應用過濾條件，確保準確性 |

SQL Server 的 Bitmap Filter 還常出現在 parallel Hash Join 中：Probe side 生成一個 bitmap 傳給 Build side，用來快速排除不可能匹配的 row，減少數據傳輸。這是 PostgreSQL 沒有的用法。

無論 PostgreSQL 還是 SQL Server，引入 bitmap scan 都是為了解決同一個矛盾：
- 普通 Index Scan：直接從 index 跳轉到對應的 row（Random Access），跳躍次數一多，random I/O 會拖垮性能。
- Seq Scan（全表掃描）：順序讀取很快，但如果只需要 5% 的 row，讀全部又太浪費。

Bitmap scan 折中了兩者：用 index 快速找到「哪些 page 可能有用」（減少讀取量），然後按物理順序讀取這些 page（保證 I/O 效率）。對 SSD 和 HDD 都很友好。

---