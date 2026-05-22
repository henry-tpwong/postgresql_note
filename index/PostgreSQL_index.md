# PostgreSQL Index 學習筆記

---

## 1. Bitmap Heap Scan

### Block = Page（8KB）

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

### 為什麼時間花在 Bitmap Heap Scan 上，而不是 Index Scan？

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

### 為什麼 BRIN 比 B-tree 更省空間？（圖解）

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

---

## 5. B+ Tree Leaf Page 與 Covering Index

### 什麼是 Leaf Page？

Leaf Page（葉子頁）是 B+ 樹索引中最底層的節點，也是實際存放「索引條目」的地方。

在 B+ 樹索引結構中：

- **根節點 / 內部節點**：只存鍵值 + 子頁指針，用來導航。
- **葉子頁（Leaf Page）**：位於最底層，存放完整的索引條目。
  - 若為聚簇索引（PostgreSQL 中不存在，但概念上等同於 table 本身），葉子頁本身就是數據頁。
  - 若為非聚簇索引，葉子頁存「索引鍵 + 指向數據行的指針（TID）」。
  - 所有葉子頁之間通常有雙向鏈結，支援範圍掃描。

#### 圖解：B+ Tree 結構

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

#### 如果有 Covering Index with INCLUDE

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

### Covering Index（INCLUDE）與普通 Composite Index 的差異

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

### Partition + BRIN 的組合應用（IoT 黃金組合）

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
