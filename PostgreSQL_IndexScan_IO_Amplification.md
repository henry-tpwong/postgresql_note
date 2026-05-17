# Index Scan IO Amplification — Correlation 與 Heap Page Scan 放大的統計學原理

> 來源：[digoal - 索引顺序扫描引发的堆扫描IO放大背后的统计学原理与解决办法 (2014-04-26)](https://github.com/digoal/blog/blob/master/201404/20140426_01.md)

---

## 1. 問題本質：Correlation 如何導致 Index Scan IO 放大

當 B-tree Index Scan 的 index column 與物理存儲（ctid）的 linear correlation 很低時，會引發 **heap page scan 的 IO 放大**：每個 index entry 指向的 heap page 幾乎都不同，導致每次 index lookup 都伴隨一次 heap page random read。

```
高 correlation (≈1)： Index entry → page 0 → page 0 → page 1 → page 1 → ...
                        ↓                (連續，page cache 友好)

低 correlation (≈0)： Index entry → page 5 → page 200 → page 3 → page 180 → ...
                        ↓                (跳躍，每次都要讀不同 page)
```

後果：
- 如果 row 數足夠多（如 N 條匹配 row），heap page scan 次數 ≈ N（最壞情況）
- 大 `block_size`（如 32KB vs 8KB）會進一步放大這個問題
- 即使所有 data 在 shared_buffers 中，CPU 端的 page lookup 開銷依然存在

> 補充（Senior Dev）：這個問題的數學本質是 **coupon collector problem** 的變體。假設表有 P 個 page、查詢返回 N 個 row，每個 row 落在哪個 page 是均勻隨機的，則需要掃描的 distinct page 數量期望值為 `P × (1 - (1 - 1/P)^N)`。當 N 接近 P 時，幾乎所有 page 都會被訪問；當 N = P 時，期望值 ≈ `P × (1 - 1/e)` ≈ 63% 的 page。這也解釋了為什麼對於大範圍查詢，即使 correlation = 0，Seq Scan 的 IO 總量（P 個 sequential read）可能小於 Index Scan 的 IO 總量（N 個 random read），從而 planner 應選 Seq Scan。

---

## 2. 實驗證明

### 成本因子設定

```sql
shared_buffers = 8192MB
random_page_cost = 1.0        -- 刻意壓低以凸顯問題
cpu_index_tuple_cost = 0.005
effective_cache_size = 96GB
```

### 場景 A：高 Correlation（correlation = 1）

```sql
CREATE TABLE test_indexscan (id int, info text);
INSERT INTO test_indexscan
    SELECT generate_series(1, 5000000), md5(random()::text);
CREATE INDEX idx_test_indexscan_id ON test_indexscan (id);
```

```
 relpages
----------
    10396   -- table pages
     3402   -- index pages
```

驗證 correlation 和物理順序：

```sql
SELECT correlation FROM pg_stats
WHERE tablename = 'test_indexscan' AND attname = 'id';
-- correlation: 1

SELECT ctid, id FROM test_indexscan LIMIT 10;
 ctid   | id
--------+----
 (0,1)  | 1
 (0,2)  | 2
 (0,3)  | 3
 ...完全與 ctid 順序一致...
```

Index Scan 結果：**無放大**

```sql
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM test_indexscan WHERE id > 90000;

 Index Scan using idx_test_indexscan_id on test_indexscan
   (cost=0.43..99518.57 rows=4912065 width=37)
   (actual time=0.180..2172.949 rows=4910000 loops=1)
   Index Cond: (id > 90000)
   Buffers: shared hit=10209 read=3338
 Total runtime: 2674.637 ms
```

Buffers ≈ 10209 + 3338 = 13547，接近 table + index 總 page 數（10396 + 3402 = 13798）。每個 heap page 只被讀一次。

### 場景 B：低 Correlation（correlation ≈ 0.01）

```sql
TRUNCATE test_indexscan;
INSERT INTO test_indexscan
    SELECT (random()*5000000)::int, md5(random()::text)
    FROM generate_series(1, 100000);

-- correlation: 0.00986802
```

物理順序完全隨機：

```sql
SELECT ctid, id FROM test_indexscan LIMIT 10;
 ctid   |   id
--------+---------
 (0,1)  | 4217216
 (0,2)  | 2127868
 (0,3)  | 2072952
 (0,4)  |   62641
 (0,5)  | 4927312
 ...全部亂序...
```

大小：

```
 relpages
----------
      208   -- table pages
       86   -- index pages
```

Index Scan 結果：**IO 放大 335x**

```sql
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM test_indexscan WHERE id > 90000;

 Index Scan using idx_test_indexscan_id on test_indexscan
   (cost=0.29..2035.38 rows=99719 width=37)
   (actual time=0.027..87.456 rows=98229 loops=1)
   Index Cond: (id > 90000)
   Buffers: shared hit=97837            ← 遠大於 table+index (208+86=294)！
 Total runtime: 97.370 ms
```

Buffers = 97837，實際 table + index = 294 page，IO 放大了 **~333 倍**。原因是 query 返回 98,229 row，每個 row 的 ctid 指向不同的 heap page → 至少 98,229 次 heap page 讀取。

> 補充（Senior Dev）：這裡的 97.37ms 看似不慢，但前提是 **所有 data 已經在 shared_buffers 中**（`shared hit` 而非 `read`）。若 data 不在 memory，physical random read 會讓 runtime 變成 **幾十秒甚至分鐘級**。這也是為什麼在 production 中 index scan IO 放大經常伴隨著突然的 latency spike —— 原本 data 在 cache 時一切正常，cache eviction 後效能崩潰。

---

## 3. Cost Model 剖析：Planner 為何選錯 Plan

### `cost_index()` 核心公式

來自 `src/backend/optimizer/path/costsize.c`：

```c
/*
 * Now interpolate based on estimated index order correlation
 * to get total disk I/O cost for main table accesses.
 */
csquared = indexCorrelation * indexCorrelation;

run_cost += max_IO_cost + csquared * (min_IO_cost - max_IO_cost);
```

含義：
- **max_IO_cost**：最壞情況（correlation = 0），每個 index entry 對應一個不同的 heap page
  - `≈ index_pages × random_page_cost + N_tuples × random_page_cost`
- **min_IO_cost**：最佳情況（correlation = 1），所有匹配 row 在同一個或少數連續 page 中
  - `≈ index_pages × random_page_cost + 1 × random_page_cost`（極簡化）
- **實際 cost**：在 max 和 min 之間按 `correlation²` 做二次方插值

這意味著 **correlation 在 cost model 中的影響是非線性的**。correlation = 0.01 → csquared = 0.0001 → 幾乎貼近 max_IO_cost。correlation = 0.5 → csquared = 0.25 → cost 已有明顯折扣。

### 問題案例：Planner 低估

在上述場景 B 中，planner 只估計了約 293 個 data block 的讀取：

```sql
-- random_page_cost=2 時的 total cost: 2305.73
-- random_page_cost=1 時的 total cost: 2012.75
-- cost 差值: 2305 - 2012 = 293 (data block estimate)
```

實際讀了 97,837 個 buffer，planner 低估了約 **334 倍**。

**根本原因**：
- `random_page_cost = 1`（等於 `seq_page_cost`）→ planner 認為 random read 沒有懲罰
- `effective_cache_size = 96GB >> table_size` → planner 認為所有 page 都在 cache，random vs sequential 成本無差異
- correlation 很低但 planner 的低估被上面兩個因素掩蓋

> 補充（Senior Dev）：`cost_index()` 的 `max_IO_cost` 計算細節：
>
> `max_IO_cost` 實際上是假設 index 中每個匹配 entry 對應的 heap tuple 都位於不同的 page。具體來說：
> - `index_pages_fetched`：根據 selectivity 估算要掃多少 index page
> - `heap_pages_fetched`：根據 Mackert-Lohman 公式估算（考慮 buffer cache 效應和重複訪問同一 page 的機率），而非簡單的 `min(rows_returned, table_pages)`。
>
> planner 的誤差來源主要有三層：
> 1. **selectivity estimate 偏差**：`pg_stats` 的 MCV / histogram 不準
> 2. **correlation estimate 偏差**：`pg_stats.correlation` 基於抽樣，可能誤差大
> 3. **cost parameter 虛假**：`random_page_cost = 1` 在人造情境下抹平了 random/seq 差異
>
> Double-check 你的 cost model 可用這個查詢：
> ```sql
> SELECT
>     tablename, attname,
>     correlation,
>     n_distinct,
>     array_length(most_common_vals, 1) AS mcv_count,
>     array_length(histogram_bounds, 1) - 1 AS histogram_buckets
> FROM pg_stats
> WHERE tablename = 'your_table' AND attname = 'your_column';
> ```

---

## 4. 成本因子校準實驗

以下展示在**不同成本因子組合**下 planner 的選擇變化。環境：table 219 MB，index 64 MB。

### Default cost factors

```sql
seq_page_cost = 1          random_page_cost = 4
cpu_tuple_cost = 0.01      cpu_index_tuple_cost = 0.005
cpu_operator_cost = 0.0025 effective_cache_size = 128MB
```

### 實驗矩陣

| random_page_cost | effective_cache_size | enable_seqscan | enable_bitmapscan | Planner 選擇 | Buffers | Runtime |
|:-:|:-:|:-:|:-:|------|--------|--------|
| 10 | < table+index | on | on | **Seq Scan** ✓ | 28,038 | 2012ms |
| 10 | < table+index | off | on | **Bitmap Heap Scan** ✓ | 36,229 | 3586ms |
| 10 | < table+index | off | off | **Index Scan** (cost高) | 3,005,084 | 6173ms |
| 1 | < table+index | on | on | **Seq Scan** ✓ | 28,038 | 2249ms |
| 1 | < table+index | off | on | **Bitmap Heap Scan** ✓ | 36,229 | 2956ms |
| 1 | **283MB** (> table+index) | off | on | **Index Scan** ✗ 選錯！ | 3,005,084 | 5690ms |
| 10 | 283MB | off | on | **Bitmap Heap Scan** ✓ 恢復正常 | 36,229 | 2698ms |

關鍵觀察：

1. `random_page_cost = 10` 時 planner 始終選對（Seq Scan 或 Bitmap Scan，取決於 `enable_seqscan`）。
2. `random_page_cost = 1` + `effective_cache_size > table+index` → planner 錯誤選擇 Index Scan，實際 Buffers 從 36K 暴漲到 300 萬，runtime 翻倍。
3. 即使 `random_page_cost = 1`，只要 `effective_cache_size < table_size`，planner 仍正確（因為它還認為有 cache miss 的懲罰）。

> 補充（Senior Dev）：**成本因子校準黃金法則**：
>
> | 設備 | `random_page_cost` | `seq_page_cost` | 備註 |
> |------|:-:|:-:|------|
> | HDD (7200 rpm) | 4.0 - 10.0 | 1.0 | `random_page_cost` 取決於 seek time。7200rpm HDD 的 random read 約 100-150 IOPS，sequential 可達 100-200 MB/s |
> | SATA SSD | 1.5 - 2.0 | 1.0 | random read latency 約 0.1ms |
> | NVMe SSD | 1.1 - 1.3 | 1.0 | random read latency 約 0.05ms |
> | Cloud Block (gp2/gp3) | 2.0 - 4.0 | 1.0 | burst 結束後 random IOPS 急劇下降 |
> | Full Memory (無 disk IO) | 1.6 | 1.0 | 即使全在 RAM，random page lookup 仍有 CPU cache miss 成本，絕對不能設為 1 |
>
> 校準方法：實際測量你的硬體（用 `pg_test_fsync` + `fio`），然後在 `postgresql.conf` 中設定全域值。針對特定 query 可以用 `SET random_page_cost = ...` 在 session level 臨時調整。
>
> 另一個容易踩的坑：`effective_cache_size` **不分配 memory**，只影響 planner assumption。設得太大會讓 planner 過度樂觀地選擇 Index Scan。經驗法則：`effective_cache_size = shared_buffers × 2`（考慮 OS filesystem cache）。

---

## 5. 三種解決方案

### 方案 A：讓 Planner 選 Bitmap Scan（成本因子校正）

Bitmap Heap Scan 在取得所有 index entry 後，**按 ctid（heap physical order）排序**後再依序讀取 heap page。這確保了每個 heap page 只被讀一次，徹底消除 IO 放大：

```sql
SET enable_indexscan = off;   -- 強制測試用

EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM test_indexscan WHERE id > 90000;

 Bitmap Heap Scan on test_indexscan
   (cost=846.77..2282.96 rows=98255 width=37)
   (actual time=15.291..35.911 rows=98229 loops=1)
   Recheck Cond: (id > 90000)
   Buffers: shared hit=292              ← 解決！(vs Index Scan 的 97837)
   ->  Bitmap Index Scan on idx_test_indexscan_id
         (cost=0.00..822.21 rows=98255 width=0)
         (actual time=15.202..15.202 rows=98229 loops=1)
         Index Cond: (id > 90000)
         Buffers: shared hit=84
 Total runtime: 45.838 ms               ← 從 97ms 降到 46ms（純 memory 場景）
```

正確的成本因子（`random_page_cost = 10`）會讓 planner 自動選 Bitmap Scan。

### 方案 B：CLUSTER（物理重排）

```sql
CLUSTER test_indexscan USING idx_test_indexscan_id;
```

將 table 按 index 的邏輯順序重新物理排列，使 correlation 回到 1。之後 Index Scan 不再有 IO 放大。

> 補充（Senior Dev）：`CLUSTER` 的代價很大：
> - **Exclusive lock**（`ACCESS EXCLUSIVE`）—— 執行期間 table 不可讀寫
> - **會 rewrite 整張表**—— 需要至少 table size 的 free disk space
> - **之後的 INSERT / UPDATE 會慢慢破壞 correlation**，需要定期重跑 CLUSTER
> - **Index 會被重建**（CLUSTER 後所有 index 都是 fresh compact state）
>
> Online 替代方案：**pg_repack**（原名 pg_reorg）。它在背景建立新 table + trigger 同步變更，最後在短暫 exclusive lock 下切換。對於 production 場景，`pg_repack` 是唯一可行的 CLUSTER 等價操作。
>
> 另一個輕量方案：如果表是 append-only（只 INSERT 不 UPDATE/DELETE），而且 INSERT 順序就符合 index 順序，那 correlation 天然保持高位，不需要 CLUSTER。

### 方案 C：Core 理解 —— 三方案決策流程

```
correlation ≈ 1？
  ├─ 是 → Index Scan 最佳（無需任何處理）
  └─ 否 →
      ├─ 查詢結果 row 數多 → 傾向 Seq Scan（sequential read 更快）
      ├─ 查詢結果 row 數中等 → 確保 planner 選 Bitmap Scan（校準成本因子）
      ├─ 查詢結果 row 數少 → Index Scan 放大有限，可接受
      └─ 若必須用 Index Scan且相關性差 → CLUSTER / pg_repack
```

---

## 6. 關聯文章

原文引用的相關資料：

| 主題 | 連結 |
|------|------|
| 成本因子校準 | [digoal - PostgreSQL explain cost constants alignment to timestamp](https://github.com/digoal/blog/blob/master/201311/20131126_03.md) |
| 線性相關性計算 | [digoal - PostgreSQL 计算任意类型字段之间的线性相关性](https://github.com/digoal/blog/blob/master/201604/20160403_01.md) |
| 統計資訊中的 correlation | [digoal - PostgreSQL 统计信息之逻辑与物理存储的线性相关性](https://github.com/digoal/blog/blob/master/201502/20150228_01.md) |
| Community discussion | [pgsql-hackers: Index Scan cost model](http://www.postgresql.org/message-id/flat/13668.1398541533@sss.pgh.pa.us) |

> 補充（Senior Dev）：`pg_stats.correlation` 是一個介於 -1 到 1 的值，由 `src/backend/commands/analyze.c` 中的 `compute_scalar_stats()` 計算。其演算法是對 `pg_statistic` 中抽樣的 `stanumbers1` 做公式計算，而非全表掃描。在 `default_statistics_target = 100` 的情況下，抽樣約 3,000 行來推估算整個 table 的 correlation。對於大表，這個 estimate 可能不準（抽樣誤差）。若有明顯的 plan regression 懷疑 correlation 估算錯誤，可以 `ALTER TABLE t ALTER COLUMN c SET STATISTICS 5000` 提高 sampling 精度（代價是 `ANALYZE` 變慢）。在 PG 10+，也可以用 `CREATE STATISTICS` 做 multi-column correlation tracking。
