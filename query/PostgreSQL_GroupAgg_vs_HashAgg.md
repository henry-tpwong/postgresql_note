# PostgreSQL GROUP BY：Sort (GroupAgg) vs Hash (HashAgg) 的選擇邏輯

> 來源：[digoal - PostgreSQL sort or not sort when group by? (2015-08-13)](https://github.com/digoal/blog/blob/master/201508/20150813_02.md)
>
> 更新於 2026-05-17，補充 PG 11~18 GroupAgg / HashAgg 演進

---

## 核心問題：GROUP BY 為什麼有時用 Sort？

Optimizer 選擇 Agg 策略的唯一標準是 cost。`GROUP BY` 有兩種 aggregation strategy：

| Strategy | 說明 |
|----------|------|
| **GroupAgg (AGG_SORTED)** | 先 Sort 再分組聚合。input 已有序時 skip Sort（如來自 index scan） |
| **HashAgg (AGG_HASHED)** | 建 hash table 聚合。需等全部 input 就緒才開始輸出 |

兩者 total CPU cost 在 cost 模型中相同，但 **GroupAgg startup cost 更低**（可 on-the-fly 輸出），HashAgg startup cost = input total cost（必須等所有 input 讀完）。

### 實驗：work_mem 影響策略選擇

```sql
CREATE TABLE t1 (c1 INT, c2 INT, c3 INT, c4 INT);
INSERT INTO t1 SELECT generate_series(1, 100000), 1, 1, 1;
```

**work_mem = 4MB（預設）→ external sort + GroupAgg：**

```sql
SHOW work_mem;  -- 4MB

EXPLAIN (ANALYZE, VERBOSE, BUFFERS, COSTS, TIMING)
SELECT c1, c2, c3, c4 FROM t1 GROUP BY c1, c2, c3, c4;

-- Group  (cost=9845.82..11095.82 rows=100000 width=16)
--        (actual time=340.382..384.324 rows=100000 loops=1)
--   Group Key: t1.c1, t1.c2, t1.c3, t1.c4
--   Buffers: shared hit=544, temp read=318 written=318
--   ->  Sort  (cost=9845.82..10095.82 rows=100000 width=16)
--             (actual time=340.379..353.887 rows=100000 loops=1)
--         Sort Key: t1.c1, t1.c2, t1.c3, t1.c4
--         Sort Method: external sort  Disk: 2544kB
--         Buffers: shared hit=544, temp read=318 written=318
--         ->  Seq Scan on public.t1  (cost=0.00..1541.00 rows=100000 width=16)
--               (actual time=0.025..26.641 rows=100000 loops=1)
-- Execution time: 392.131 ms
```

Sort Method: **external sort Disk: 2544kB** — 4MB `work_mem` 不夠（100,000 row × 16 bytes avg width），溢寫磁盤，total execution 392ms。

**work_mem = 1GB → HashAggregate：**

```sql
SET work_mem = '1GB';

EXPLAIN (ANALYZE, VERBOSE, BUFFERS, COSTS, TIMING)
SELECT c1, c2, c3, c4 FROM t1 GROUP BY c1, c2, c3, c4;

-- HashAggregate  (cost=2541.00..3541.00 rows=100000 width=16)
--                (actual time=74.786..97.568 rows=100000 loops=1)
--   Group Key: t1.c1, t1.c2, t1.c3, t1.c4
--   Buffers: shared hit=541
--   ->  Seq Scan on public.t1  (cost=0.00..1541.00 rows=100000 width=16)
--         (actual time=0.037..16.179 rows=100000 loops=1)
-- Execution time: 104.705 ms
```

HashAgg 無 Sort step，total execution 104ms（vs Sort+Group 392ms）。

**強制關閉 HashAgg → quicksort + GroupAgg：**

```sql
SET enable_hashagg = off;

EXPLAIN (ANALYZE, VERBOSE, BUFFERS, COSTS, TIMING)
SELECT c1, c2, c3, c4 FROM t1 GROUP BY c1, c2, c3, c4;

-- Group  (cost=9845.82..11095.82 rows=100000 width=16)
--        (actual time=28.217..62.442 rows=100000 loops=1)
--   Group Key: t1.c1, t1.c2, t1.c3, t1.c4
--   Buffers: shared hit=541
--   ->  Sort  (cost=9845.82..10095.82 rows=100000 width=16)
--             (actual time=28.214..35.161 rows=100000 loops=1)
--         Sort Key: t1.c1, t1.c2, t1.c3, t1.c4
--         Sort Method: quicksort  Memory: 7760kB
--         Buffers: shared hit=541
-- Execution time: 68.409 ms
```

Sort Method: **quicksort Memory: 7760kB** — 1GB `work_mem` 夠用，in-memory sort。total execution 68ms，反而比 HashAgg（104ms）還快。但在正常 GUC 設定下 optimizer 選 HashAgg，因為 cost 估算更低。

---

## 原始碼分析：cost_agg() — 兩種策略的 Cost 計算

來源：`src/backend/optimizer/path/costsize.c`

```c
void
cost_agg(Path *path, PlannerInfo *root,
         AggStrategy aggstrategy, const AggClauseCosts *aggcosts,
         int numGroupCols, double numGroups,
         Cost input_startup_cost, Cost input_total_cost,
         double input_tuples)
```

**AGG_SORTED（GroupAgg）**：

```c
startup_cost = input_startup_cost;           // Sort 的 startup cost
total_cost   = input_total_cost;             // Sort 的 total cost
total_cost  += aggcosts->transCost.startup;  // aggregate transition fn startup（一次）
total_cost  += aggcosts->transCost.per_tuple * input_tuples;  // per-tuple trans
total_cost  += (cpu_operator_cost * numGroupCols) * input_tuples;  // per-group 比較
total_cost  += aggcosts->finalCost * numGroups;  // per-group final fn
total_cost  += cpu_tuple_cost * numGroups;       // 每個輸出 group
```

**AGG_HASHED（HashAgg）**：

```c
startup_cost = input_total_cost;             // 必須等所有 input（無 on-the-fly）
startup_cost += aggcosts->transCost.startup;
startup_cost += aggcosts->transCost.per_tuple * input_tuples;
startup_cost += (cpu_operator_cost * numGroupCols) * input_tuples;
total_cost   = startup_cost;
total_cost  += aggcosts->finalCost * numGroups;
total_cost  += cpu_tuple_cost * numGroups;
```

**關鍵差異**：HashAgg 的 startup_cost = `input_total_cost`，意味著在第一個 output row 產生前必須讀完所有 input 並建好 hash table。GroupAgg 的 startup_cost = `input_startup_cost`（通常是 Sort 的 startup cost），可越早輸出（對 LIMIT 查詢有利）。

原始碼註釋說明了取捨原則：

```
 * Note: in this cost model, AGG_SORTED and AGG_HASHED have exactly the
 * same total CPU cost, but AGG_SORTED has lower startup cost.  If the
 * input path is already sorted appropriately, AGG_SORTED should be
 * preferred (since it has no risk of memory overflow).
```

> 補充（Senior Dev）：當 input 已有序時（如來自 BTREE index scan 或 merge join），AGG_SORTED 可以跳過 Sort step，此時 GroupAgg 的 total cost 會顯著低於 HashAgg。這是為什麼 `GROUP BY indexed_column` 常看到 GroupAgg 而非 HashAgg 的原因。

---

## cost_sort() — Sort 策略的成本分岔

來源：`src/backend/optimizer/path/costsize.c`

Sort 成本計算根據 `output_bytes` vs `sort_mem_bytes` 的分岔點選擇三種演算法：

```
1. output_bytes > sort_mem_bytes  →  external disk sort (polyphase merge)
    cost ≈ comparison_cost × tuples × log₂(tuples)
         + disk I/O: 2 × npages × log_M(nruns)
         + 75% sequential I/O, 25% random I/O

2. tuples > 2 × output_tuples  →  bounded heap sort (LIMIT 場景)
    僅保留 top-K tuples 在 heap 中

3. 其他 →  quicksort (全部 in-memory)
    cost ≈ comparison_cost × tuples × log₂(tuples)
```

這解釋了實驗中 work_mem=4MB 時落 external sort（Disk），work_mem=1GB 時落 quicksort 的原因。

---

## GroupAgg vs HashAgg 選擇的實戰考量

| 維度 | GroupAgg | HashAgg |
|------|----------|---------|
| requires sort | 是（除非 input 已有序） | 否 |
| startup cost | 低 | 高（=input total cost） |
| memory risk | 取決於 sort 階段 memory | 取決於 #distinct groups |
| LIMIT 友好 | 是（early output） | 否（需建完整 hash table） |
| 大量 distinct groups | 不適合（sort 龐大） | 適合（若 hash table fits memory） |
| input 已有序 | **最佳**（skip sort, 零風險） | 仍需建 hash table |

> 補充（Senior Dev）：當 `work_mem` 不足造成 HashAgg 的 hash table spill 到 disk 時（即 hash batch overflow，寫入 temp file），成本會急劇上升。這情況下 optimizer 可能低估了 HashAgg 的實際成本。可透過 `SET enable_hashagg = off` 強制切換 GroupAgg 對比真實 performance，或增加 `work_mem` 直到 `Hash Batches: 1`。

---

## 相關參數

| 參數 | 預設 | 說明 |
|------|------|------|
| `enable_hashagg` | on | 允許 optimizer 選擇 HashAgg |
| `enable_sort` | on | 允許 optimizer 選擇 Sort |
| `work_mem` | 4MB | sort 與 hash table 的 memory 上限（per operation） |
| `hash_mem_multiplier` | 2.0 (PG 16+) | Hash 操作的 memory = `work_mem × hash_mem_multiplier`，可讓 hash 使用更多 memory 而 sort 不受影響 |

> 補充（Senior Dev）：PG 16+ 的 `hash_mem_multiplier` 讓 HashAgg 可獲得比 Sort 更多的 work_mem（預設：HashAgg 獲得 `work_mem × 2.0`），解決過去「為了讓 HashAgg 有足夠 memory 而把 `work_mem` 開太大 → Sort 也用很多 memory → memory 總消耗暴增」的困境。

---

## 版本演進

| 功能 | 版本 | 說明 |
|------|------|------|
| hash_mem_multiplier | PG 16 | Hash 操作的獨立 memory multiplier |
| parallel hash join / hashagg | PG 11+ | 多 worker 共用同一個 hash table |
| disk-based HashAgg | PG 13+ | Hash table 可溢寫到 disk batch file |
| plan-time hash sizing | PG 14+ | Optimizer 在 planning 時估算 hash table 大小 |

## 原始碼參考

1. `src/backend/optimizer/path/costsize.c` — `cost_agg()` / `cost_sort()`
2. `src/backend/executor/nodeAgg.c` — GroupAgg / HashAgg 的實際執行邏輯
3. `src/backend/executor/nodeHash.c` — Hash table 建構與 probing
