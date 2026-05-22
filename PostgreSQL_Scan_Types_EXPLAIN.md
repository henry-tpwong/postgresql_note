# PostgreSQL EXPLAIN 計畫中的掃描類型全解析

> 原文：[CrunchyData - Elizabeth Christensen, "Postgres Scan Types in EXPLAIN Plans" (2025-12-04)](https://www.crunchydata.com/blog/postgres-scan-types-in-explain-plans)
>
> 本文基於原文架構，補充 Production 實戰經驗、Planner 決策機制、參數調校建議與掃描類型選擇法則。

---

## 為什麼讀懂掃描類型是效能的關鍵

Postgres 查詢效能的秘密不僅在於 *查什麼*，更在於 **如何找到資料**。`EXPLAIN` 是理解查詢行為的核心工具，而其中最關鍵的就是 **掃描類型（Scan Type）**——它決定了查詢是毫秒級還是秒級。

本文涵蓋六種主要掃描類型、Planner 選擇邏輯、以及 Production 調校策略。

![Postgres EXPLAIN Plan](images/01-header.jpg)

---

## Sequential Scan（循序掃描）

![Sequential Scan](images/02-seq-scan.jpg)

循序掃描是最基礎的資料存取方式：**從頭到尾逐行讀取整張表**，檢查每一行是否符合 `WHERE` 條件。

```sql
EXPLAIN SELECT * FROM accounts;

                            QUERY PLAN
-------------------------------------------------------------------
 Seq Scan on accounts  (cost=0.00..22.70 rows=1270 width=36)
(1 row)
```

### Planner 何時選擇 Seq Scan

Planner 並非「有索引就一定用」。選擇 Seq Scan 的典型場景：

| 條件 | 原因 |
|------|------|
| 小表（少於數百 rows） | 索引 lookup 的 random I/O overhead 比一次 sequential read 更貴 |
| 回傳大量 row（> ~5-10% 總 row 數） | 大量 random page fetch 累積成本超過一次性全表掃描 |
| 沒有合適索引 | Planner 別無選擇 |
| `WHERE` 條件 selectivity 極低 | 例如 `WHERE is_active = true` 且 99% rows 為 true |

### Senior Dev 補充：成本模型的直覺解讀

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

## Index Scan（索引掃描）

![Index Scan](images/03-index-scan.jpg)

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

### Index Scan vs Seq Scan：Planner 的分界線

Planner 使用以下公式決定用 Index Scan 還是 Seq Scan：

```
random_page_cost * selectivity * relpages  vs  seq_page_cost * relpages
```

當 selectivity（選擇率）小於約 `seq_page_cost / random_page_cost` 時，Index Scan 勝出：
- HDD（random_page_cost=4.0）：門檻約 25%
- SSD（random_page_cost=1.1）：門檻約 90%

但 `effective_cache_size` 也影響決策：如果 Planner 認為大部分 index page 已在 shared buffer 中，random access 成本估算會降低。

### Senior Dev 補充：Index Scan 的 I/O 放大陷阱

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

## Bitmap Index Scan + Bitmap Heap Scan（點陣圖掃描）

![Bitmap Index Scan](images/04-bitmap-scan.jpg)

Bitmap Scan 是 Index Scan 與 Seq Scan 之間的 **混合策略**。當查詢返回的行數太多讓 Index Scan 不划算，但又不足以讓 Seq Scan 成為最佳選擇時，Planner 會選用 Bitmap Scan。

### 兩階段機制

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

### Recheck Cond 的必要性

Bitmap 以 page 為粒度標記（page 級別，而非 row 級別），所以 Bitmap Heap Scan 讀取的每個 page 中，**所有 row 都要重新檢查條件**（`Recheck Cond`）。這是因為一個 page 可能包含不符合條件的 row——bitmap 只知道「這個 page 可能有目標 row」，但不確定哪幾行是目標。

### Senior Dev 補充：多索引 Bitmap 組合（AND / OR）

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

## Parallel Sequential Scan（平行循序掃描）

![Parallel Sequential Scan](images/05-parallel-seq-scan.jpg)

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

### 平行查詢的觸發條件

Planner 只在滿足以下條件時考慮 Parallel Scan：

| 條件 | 說明 |
|------|------|
| `max_parallel_workers_per_gather > 0` | 預設 2，控制單個 Gather 節點的 worker 數 |
| 表大小 > `min_parallel_table_scan_size` | 預設 8MB，太小的表不值得平行化 |
| `parallel_tuple_cost` 和 `parallel_setup_cost` 的估算 | 啟動 worker 和傳遞 tuple 的 overhead 能被規模效益抵消 |

### Senior Dev 補充：Gather vs Gather Merge

- **Gather**：各 worker 結果任意順序匯集，不對結果排序。適合無 `ORDER BY` 或 `ORDER BY` 由聚合函數處理的場景
- **Gather Merge**：每個 worker 先對自己區塊排序，Gather Merge 再合併 N 個有序流（類似 merge sort 的合併階段）。適合帶 `ORDER BY` + `LIMIT` 的查詢

> **Production 建議**：`max_parallel_workers_per_gather` 不是越大越好。每個 worker 消耗一個 connection slot 和 memory。典型配置為 2-4。`max_parallel_workers`（全局上限）通常設為 CPU core 數的一半。

---

## Parallel Index Scan（平行索引掃描）

![Parallel Index Scan](images/06-parallel-index-scan.jpg)

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

### Senior Dev 補充：B-Tree Parallel Scan 的內部機制

Postgres 使用 **Parallel B-Tree Scan** 的流程如下：

1. Leader process 先讀取 index meta page，計算 index 的 page 範圍
2. 將 B-Tree leaf page 鏈表按 page 數平均分配給 N 個 worker
3. 每個 worker 從分配的起始 leaf page 開始，沿著雙向鏈表掃描
4. 每個 index entry 指向 heap tuple，worker 各自 fetch 對應的 heap data

關鍵限制：B-Tree 的平行掃描僅在 **leaf page 層級** 平行化。Root 到 leaf 的 traversal（descend）仍由每個 worker 獨立進行。

> **注意**：查看 `EXPLAIN (ANALYZE, BUFFERS)` 輸出時，留意 `Buffers` 的分佈。如果某個 worker 的 `Buffers` 明顯高於其他 worker（資料分佈偏斜），代表平行掃描的效益被不均衡的數據分佈削弱。這是 partition table 或重新設計索引策略的信號。

---

## Index-Only Scan（純索引掃描）

![Index-Only Scan](images/07-index-only-scan.jpg)

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

### 何時該建立 Covering Index

適合建 covering index 的場景：

| 信號 | 說明 |
|------|------|
| 查詢頻率極高 | 大量重複查詢的微小改善會累積成顯著收益 |
| 只 SELECT 少數欄位 | 例如 20 欄的表只查 3 個，把這 3 個欄位放進索引 |
| `EXPLAIN` 顯示大量 Heap Fetch | 表示現有 Index Scan 耗費大量 random I/O 做 heap fetch |
| 寫入頻率低 | 每個索引都需在寫入時維護，高寫入表的 covering index 會造成寫入放大 |

### Senior Dev 補充：Visibility Map、MVCC 與 VACUUM — 為什麼 VACUUM 是 Index-Only Scan 的幕後功臣

> **實戰法則**：如果你建了 covering index 但 `Heap Fetches` 仍然很大，**問題不是索引設計，而是 VACUUM 策略**。增加 `autovacuum` 頻率或在大量寫入後手動 `VACUUM` 可以恢復 Index-Only Scan 的效能。

要理解為何 VACUUM 是 Index-Only Scan 的「幕後功臣」，得先從 PostgreSQL 的 MVCC（多版本並行控制）和 Visibility Map（可見性地圖）說起。

#### 1. 為什麼一行資料會變成「看不見」？

在 PostgreSQL 中，`UPDATE` 或 `DELETE` 並不會立刻移除舊資料，而是在原地留下一個 **死元組（dead tuple）**。這樣設計是為了讓其他正在執行的事務還能看見他們「應該看見」的版本。

- 當你執行 `UPDATE` 時，Postgres 會插入一個新版本的 row，並把舊版本標記為無效。
- 這些舊版本依然佔用磁碟空間，而且 heap page 上會同時存在「對某些事務可見」和「對某些事務不可見」的 tuple。

只有當沒有任何事務還需要這些舊版本時，它們才會被 `VACUUM` 清理，空間才能被回收重用。

#### 2. Visibility Map：Index-Only Scan 的「免檢金牌」

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

#### 3. VACUUM 如何讓 VM 變成 all-visible？

`VACUUM` 會做兩件關鍵事：

1. **清理死元組**：將不再被任何事務需要的舊版本 tuple 標記為可用空間。
2. **更新 Visibility Map**：清完 dead tuple 後，該 page 上剩下的全部都是「對所有人可見」的 live tuple，此時 `VACUUM` 就會把這個 page 在 VM 中的 `all-visible` bit 設為 1。

之後的 Index-Only Scan 再碰到這個 page，就能直接跳過 heap fetch，因為 VM 已經保證「不用檢查了，全部可見」。

這就是那條實戰法則背後的原理：因為即使索引裡包含了所有需要的欄位，只要那些 heap page 還沒被 `VACUUM` 更新過 VM，Postgres 就不敢直接用索引的內容，被迫一次又一次地去 heap 確認，導致大量的 `Heap Fetches`。

#### 4. 實際監控與調整策略

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

#### 5. 總結：VACUUM 是 Index-Only Scan 效能的真正守門員

- 沒有 `VACUUM`，Visibility Map 就不會更新，Index-Only Scan 永遠無法信任索引中的資料，必須頻繁回表檢查。
- 即使你精心設計了 covering index，只要寫入多、VACUUM 少，`Heap Fetches` 就會居高不下，索引效益大打折扣。
- 在生產環境中，監控 dead tuple 比例 + 調校 autovacuum 參數，和設計索引本身一樣重要。

### 包含欄位的索引（INCLUDE Index）

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

## 掃描類型總覽與決策流程圖

![Summary of Scan Types](images/08-summary.jpg)

### 六種掃描類型速查

| 掃描類型 | EXPLAIN 關鍵字 | 運作方式 | 最佳場景 |
|----------|---------------|---------|---------|
| Sequential Scan | `Seq Scan` | 讀取整表，逐行檢查 | 小表、回傳 > ~10% rows |
| Index Scan | `Index Scan` | B-Tree lookup → heap fetch | 精準查詢（PK / unique），回傳少量行 |
| Bitmap Scan | `Bitmap Index Scan` + `Bitmap Heap Scan` | 建立 page-level bitmap → 依序讀 heap | 中等選擇率、多條件 `AND`/`OR` |
| Parallel Seq Scan | `Parallel Seq Scan` + `Gather` | 多 worker 分塊掃表 | 大型全表掃描 |
| Parallel Index Scan | `Parallel Index Scan` + `Gather` | 多 worker 分區段掃描索引 | 大型索引範圍掃描 |
| Index-Only Scan | `Index Only Scan` | 純讀索引，不碰 heap | 所有查詢欄位都在索引中 + VM clean |

### Senior Dev 補充：掃描選擇的實戰除錯策略

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

### 影響 Planner 掃描選擇的關鍵 GUC 參數

| 參數 | 預設值 | 影響 | 建議 |
|------|--------|------|------|
| `random_page_cost` | 4.0 | 降低會讓 Index Scan 更容易勝出 | SSD 設 1.1-1.5 |
| `seq_page_cost` | 1.0 | 基準值，一般不需調整 | 保持 1.0 |
| `effective_cache_size` | 4GB | 影響 Planner 對 index page 在 cache 中的假設 | 設為 OS cached + shared_buffers 的 50-75% |
| `work_mem` | 4MB | 影響 Bitmap Scan 是否被選用 | 視 query 類型設 16MB-256MB |
| `min_parallel_table_scan_size` | 8MB | 低於此的表不做 Parallel Scan | 預設合理 |
| `max_parallel_workers_per_gather` | 2 | 單個 Gather 的最大 worker 數 | 2-4，不超過 CPU core 的一半 |

---

## 延伸閱讀與工具

- [PostgreSQL EXPLAIN 官方文件](https://www.postgresql.org/docs/current/using-explain.html)
- [explain.depesz.com](https://explain.depesz.com/) — 貼上 EXPLAIN 輸出，自動解析和上色顯示 cost/actual time 差異
- [explain.dalibo.com](https://explain.dalibo.com/) — 另一款 EXPLAIN 視覺化工具，支援多語系
- [CrunchyData - Parallel Queries in Postgres](https://www.crunchydata.com/blog/parallel-queries-in-postgres)
- [PG 17 新特性：B-Tree Index Deduplication 對掃描的影響](https://www.postgresql.org/docs/17/btree-implementation.html)
