# PostgreSQL BRIN 索引（Block Range INdex）

> 來源：[digoal - PostgreSQL 物联网黑科技 — 瘦身几百倍的索引(BRIN index) (2016-04-14)](https://github.com/digoal/blog/blob/master/201604/20160414_01.md)

---

## 背景：BTREE 為何不適合 IoT 場景

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

## BRIN 原理

BRIN 不逐 row 存 index entry。它將 table 中連續的 N 個 data page 視為一個 block range（預設 `pages_per_range = 128`），對每個 block range 僅存一份統計摘要：

```
min(val), max(val), has null?, all null?, left block id, right block id
```

![BRIN Principle](images/brin_principle.png)

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

## 效能實測

### 寫入效能

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

![Insert Performance](images/brin_perf_insert.png)

### Index Size

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

### 範圍查詢效能

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

### 精確點查效能

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

### 綜合性能對比圖

![Performance Summary 1](images/brin_perf_query.png)

![Performance Summary 2](images/brin_perf_summary.png)

---

## 適用場景與限制

| 適合 | 不適合 |
|------|--------|
| 海量、順序寫入的時序數據（IoT sensor、log、metrics） | 頻繁隨機點查（`WHERE id = X`） |
| 大範圍掃描（`BETWEEN`、`>=`、time range aggregation） | 資料物理順序與索引列不相關（如隨機 UUID） |
| Storage 受限環境（index size 是 BTREE 的 1/500+） | 頻繁 UPDATE / DELETE 導致 row 位置與 min/max 脫節 |

> 補充（Senior Dev）：BRIN 效能依賴資料的 **physical correlation**（索引列的值與物理存儲順序一致）。若資料按 `create_time` 順序插入，BRIN 效果極佳。但對於 `UPDATE` 頻繁的表，MVCC 產生的 new tuple 可能寫到不同 page，逐漸破壞 physical correlation。可通過 `CLUSTER` 或 `pg_repack` 定期重整，或搭配 Partition 來維持 locality。

---

## Oracle Exadata Storage Index

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

## 小結

1. BRIN 針對 IoT 場景：流式寫入、範圍查詢。對寫入影響微乎其微，index size 比 BTREE 小數百倍
2. 範圍查詢效能與 BTREE 差距在毫秒級，但 index 體積節省巨大
3. 精確點查不如 BTREE（差距 1000x），不應用作 point-lookup 的主 index
4. 結合 JSON / GiST 等功能，適合 IoT 混合分析場景
5. 德哥原文結論：DBA 應抓準各類 index 特性，匹配到合適場景（千里馬與伯樂）
