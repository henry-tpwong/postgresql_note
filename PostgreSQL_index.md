# PostgreSQL Index 學習筆記

> 原始筆記來源：多段問答對話整理而成

---

## 1. Bitmap Heap Scan

### Block = Page（8KB）

Block 與 Page 在 PostgreSQL 中是完全同義詞，指向磁盤上最小的存儲與 I/O 單元，預設大小 8KB。

- **物理層面**：一張表（或 index）在磁盤上由一個或多個固定大小的 page 組成，每個 page 8KB。當 PostgreSQL 需要讀取數據時，不會一次只讀一行，而是以 page 為最小單位讀入 shared_buffers。
- **稱呼習慣**：核心代碼和文檔中經常混用。Page 偏重邏輯結構（內有 row、page header），Block 偏重物理存儲、I/O 操作（如 blkno）。在 execution plan 和日常討論中完全可以互換。

> 補充（Senior Dev）：shared_buffers 中的 page 管理採用 LRU clock-sweep 算法，不是標準 LRU。理解這一點對判斷「哪些 page 會被踢出 buffer」很重要——頻繁訪問的小表 page 可能因為 clock-sweep 的近似性被意外踢出。

### B-tree 的 Bitmap Index Scan：預設 Row-Level，但可退化為 Lossy

- **預設情況（memory 充足）**：B-tree 構建的位圖是 row-level（TID bit array），每一位對應一個具體的 `(page, item)`，可以精確到 row。
- **memory 不足時**：如果 result set 很大，超出 `work_mem` 的限制，PostgreSQL 會把 bitmap 退化為 lossy bitmap（page-level），此時每個位對應一個 page，整個 page 要被讀出來逐行 Recheck。

從 `EXPLAIN (ANALYZE, BUFFERS)` 的輸出中，看 Bitmap Heap Scan 的 `Heap Blocks: exact=xxx lossy=yyy` 來判斷是否退化。lossy 不為零就說明退化發生了。

> 補充（Senior Dev）：預設 `work_mem = 4MB` 對於現代伺服器極度保守。一個 query 可能用到 `work_mem * number_of_parallel_workers` 的 memory（例如 Hash Join），而不是 `work_mem` 上限。在 OLAP 場景建議 `work_mem = 256MB ~ 1GB`，但 OLTP 場景保持較低以避免單個 connection 耗盡 memory。

### Bitmap 結構：1 bit = 1 page（非 1 row）

這是理解 Bitmap Heap Scan 的關鍵。Bitmap 不是按 row 構建的，而是按 page 構建。

假設一張表有 100 萬 row，存儲在 10,000 個 8KB 的 page 裡。Bitmap 就是一個長度為 10,000 的 bit array：第 i 位對應表文件中的第 i 個 data page（page number i）。

**標記過程（Bitmap Index Scan）**：掃描 index（如 BRIN 或 B-tree）時，並不是馬上回 Heap 讀取 row。Index Scan 會找出哪些 page 可能包含符合條件的 row，然後把對應的 bit 設為 1。

- 對於 B-tree：每個 index entry 都包含 row 所在的 page 號。掃描時，會把所有匹配 entry 引用的 page 號對應的 bit 設為 1。
- 對於 BRIN：每個 index summary 覆蓋一個 page range（預設 128 個 page）。如果 summary 滿足條件（例如時間範圍重疊），就會把那 128 個 page 對應的 bit 全部設為 1。

**讀取過程（Bitmap Heap Scan）**：Bitmap 建好後，PostgreSQL 會遍歷這個 bitmap，找到所有被標記為 1 的 page，然後按物理順序（page 號遞增）依次把它們讀入 shared_buffers。這一步才真正從 Heap 中讀取 row。因為是順序讀取 page，磁頭或 SSD 不需要隨機跳轉，I/O 效率極高——這正是 Bitmap Heap Scan 優於普通 Index Scan 的核心原因。

### 為什麼用 page 作為 Bitmap 粒度，而不是 row？

- **I/O 單位決定**：無論如何，讀取數據最後都要落到 page。即使 index 能找到具體 row，最終讀取的還是包含該 row 的整個 page。所以用 page 作為 bitmap 粒度，能直接映射到物理讀取。
- **memory 和 CPU 效率**：1000 萬 row 如果用 row-level bitmap 需要 10M bit（~1.25MB），而用 page-level bitmap（假設每 page 100 row）只需要 100K bit（~12.5KB），體積小一個數量級，構建和遍歷都快得多。
- **與 BRIN 天然契合**：BRIN 本身就是 page-level summary index，它返回的就是 page range，直接填充 bitmap 特別自然。

> 補充（Senior Dev）：`effective_cache_size` 不影響 Bitmap Heap Scan 內部行為，但影響 planner 是否選擇 Bitmap Heap Scan。planner 假設 `effective_cache_size` 的 page 已在 OS page cache 中，如果這個值過小，planner 會高估 random page read 成本，傾向選 Seq Scan 而非 Bitmap Heap Scan。

### 為什麼時間花在 Bitmap Heap Scan 上，而不是 Index Scan？

很多人的直覺是：「既然 index 幫我定位了 row，為什麼大頭時間還在 Heap Scan？」原因在於：

1. **I/O 繞不開，最終數據要從 Heap 讀取**：Index 裡只存了 index key 和 row pointer（TID），並沒有存完整的 row 數據。無論 bitmap 是 row-level 還是 page-level，Bitmap Heap Scan 必須拿著這些 pointer 去 Heap 中取出整行數據。這個步驟就是慢的主要來源，尤其是：
   - 需要訪問的 data page 數量很大（即使每個 page 只取一行）。
   - 這些 page 大多不在 memory（shared_buffers 或 filesystem cache），需要物理 disk read。
2. **雖然避免了 random I/O，但讀取的 page 數可能依然很多**：Bitmap Heap Scan 會把需要訪問的 page 號排序，然後按物理順序批量讀取，這比普通 Index Scan 的隨機單 page 跳轉要快得多。但是，如果 query 本身需要讀取 10000 個不同的 data page，這 10000 次 read 仍然是實打實的 I/O 消耗。時間自然就堆在這個階段。
3. **Recheck Cond 帶來的 CPU 開銷（尤其在退化時）**：如果 bitmap 退化成了 page-level，或者使用了 BRIN 這類有損 index，那麼整個 page 內的所有 row 都要被重新檢查條件，這會產生可觀的 CPU 時間。對於 B-tree，如果 lossy 發生，同樣會因 page-level 讀取而多出大量無用的 row 檢查。
4. **Bitmap Heap Scan 是「真正幹重活」的節點**：在 plan 裡，Bitmap Index Scan 只負責掃 index、填 bitmap，幾乎純 CPU，速度很快，所以 actual time 非常小。而接下來 Bitmap Heap Scan 負責實際的表訪問和過濾輸出，是整個 query 的體力活，時間自然全部集中在這裡。

> 總結：B-tree 預設構建的是 row-level 的 bitmap（精確到 row），不是 page-level，除非 memory 不夠退化為 lossy。慢的主要原因是 Bitmap Heap Scan 要真正去讀 Heap data page，這一步驟集中了絕大多數的 I/O 和大量的 CPU 過濾。row-level bitmap 只幫你避免了「整 page Recheck」的 CPU 浪費，但無法避免「讀 page」這個動作本身。看到時間都在 Bitmap Heap Scan 上是正常的，關鍵在於判斷它是 I/O bound 還是 CPU bound，然後對症優化。

### Recheck Cond 詳解

Recheck Cond 是 PostgreSQL execution plan 中 Bitmap Heap Scan 節點上的一個關鍵步驟。其核心任務是：從 data page 中取出每一行後，再次用 index 條件核實它真的符合要求，把誤傷的 row 丟棄。

#### 為什麼需要 Recheck？——有損性的根源

Index Scan 構建 bitmap 時，不一定能精確到「具體哪一行滿足條件」，有時只能告訴你「哪些 page 裡可能有」。這種不精確就叫有損。Recheck 就是為此而生，確保最終結果絕對正確。

有損主要來自兩種情況：

1. **Index 本身有損**：
   - **BRIN**：只記錄每個 page range 的 min / max。如果查詢 `create_time >= '2026-05-10'`，一個範圍 `min='2026-05-09', max='2026-05-11'` 的 page block 就會被標記。但這個 page block 裡很可能存在 `2026-05-09` 的 row，它們不滿足條件，必須 Recheck 過濾。
   - **GIN**：Inverted index 存儲了 key 和對應的 page 號列表，但一個 page 裡可能有多 row，有的符合全文查詢，有的不符合。
   - **GiST**：取決於 operator class。例如地理位置的 `&&`（bounding box overlap）是有損的，因為 bounding box 重疊不代表幾何體真正相交。

2. **Bitmap 本身精度降低（Lossy Bitmap）**：即使 index 是精確的（如 B-tree），bitmap 本身也可能變有損。PostgreSQL 首先構建 row-level bitmap（每個 bit 對應一個 TID），但這會消耗 memory（work_mem）。如果 result set 巨大，memory 不足，系統會把 bitmap 退化為 page-level bitmap——只記錄哪些 page 包含匹配 row，丟掉了具體 row 位置。退化的 page-level bitmap 也是有損的，導致掃描時必須把整個 page 讀出來逐行 Recheck。

> 補充（Senior Dev）：Lossy bitmap 發生時，不只 Recheck 成本增加，還可能導致原本可通過 BitmapAnd 精確合併的多 index 條件被迫在 page 層面做合併，進一步放大誤報。例如 `WHERE status = 'active' AND create_time > '...'` 兩個 Bitmap Index Scan 若都退化，BitmapAnd 後每個 page 內仍需逐 row 驗證兩個條件。

#### Recheck Cond 在不同 Index 類型下的表現

| Index 類型 | Bitmap 精確度 | 是否需要 Recheck | 說明 |
|-----------|-------------|----------------|------|
| B-tree | 預設構建精確 row-level bitmap | 通常不需要（plan 可能不顯示 Recheck Cond） | 如果 work_mem 不足導致 bitmap 退化（lossy），會顯示 Recheck Cond |
| BRIN | 永遠是 page-level，有損 | 一定需要，plan 中總是出現 Recheck Cond | Index 只提供 page range，本來就不精確到 row |
| GIN | 總是 page-level，有損 | 一定需要 | Inverted index 只指向 page，page 內可能有多 row |
| GiST | 可能精確也可能有損，視 operator 而定 | 視情況而定 | 例如 `btree_gist` 的 `=` 可精確，但 `range @> elem` 通常有損 |

#### Recheck 的具體機制

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

#### Recheck Cond vs Filter

| 謂詞 | 所在節點 | 用途 |
|------|----------|------|
| Recheck Cond | 只在 Bitmap Heap Scan 中 | 重新驗證 index 條件，是對「可疑」row 的第二次篩查，確保 index 的有損性不影響最終正確性 |
| Filter | 幾乎任何節點都可以有 | Execution plan 中無法使用 index 的其他條件，或者 index 條件以外的過濾。例如 `WHERE create_time > ... AND status = 'done'`，status 的過濾可能就是 Filter |

在 plan 中你可能會同時看到兩者：index 負責時間範圍（Recheck Cond），然後對其他列進行 Filter。

#### 小結 Recheck

- Recheck Cond 是「有損 index / 有損 bitmap」的糾錯機制，必須執行以保證結果正確。
- BRIN index 必然導致 Recheck Cond，因為 index 只能標記 page range。
- 它的開銷取決於誤報 page 中的總 row 數，通過 `Rows Removed by Index Recheck` 可以量化。
- 優化思路：讓 index 更精準（調整 BRIN 參數）或避免 bitmap 退化（加大 work_mem，但對 BRIN 先天 page-level 無效）。

---

## 2. EXPLAIN 分析與效能優化

### 重點觀察指標

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

### 優化策略

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

## 3. BRIN Index（Block Range INdex）

BRIN 是 page-level summary index。不 index 每行數據，而是按 data page 的物理存儲範圍（page range）來記錄，體積比 B-tree 小幾百甚至上千倍（可能從 GB 級降到 MB 甚至 KB 級）。每個 summary 存儲連續相鄰 page 的統計信息：`min(val), max(val), has null?, all null?`。

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

## 4. 索引類型總覽

### 針對 create_time 類型字段的選擇

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

### SQL Server 對照

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
