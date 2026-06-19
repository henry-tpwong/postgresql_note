# PostgreSQL 深度學習筆記

> 部分內容參考自 [digoal (德哥) blog](https://github.com/digoal/blog)，經大幅改寫、補充 PG 9~18 版本演進、Mermaid 圖與新手導向說明。
> 適合 Developer 閱讀，著重觀念理解而非深入原始碼。每份 md 均由多篇原始筆記合併而成，依由淺到深排列，含標題層級（`# 一、` → `## 1.` → `### I.`）與 Mermaid 圖輔助說明。

---

## 目錄

| 章節 | 檔案 | 主題 |
|------|------|------|
| **Transaction 全攻略** | [`transaction.md`](transaction.md) | MVCC Snapshot 深度解析、四種隔離級別底層原理（SSI/SIREAD/2PL/S2PL）、Write Skew 重現、EvalPlanQual / Lost Update、VACUUM 影響、.NET Dapper/Polly Retry 實戰、**分佈式事務（2PC/XA/Saga/TCC/Outbox + .NET 完整範例）** |
| **索引全解析** | [`index.md`](index.md) | 六種掃描類型全解析（Seq/Index/Bitmap/Parallel/Index-Only）、Bitmap Heap Scan 詳解、BRIN/Bloom/GIN/GiST/SP-GiST/RUM、Covering Index、索引失效 20 場景 |
| **查詢深度解析** | [`query.md`](query.md) | 查詢生命週期、CBO 與 pg_hint_plan、GROUP BY 策略、IN/ANY/VALUES、分頁與計數、Recursive CTE 優化、死循環防禦、分組 Top-N（44x）、JOIN 冗餘膨脹 Early DISTINCT |
| **鎖（Lock）** | [`lock.md`](lock.md) | 隱式鎖、Lock Wait 追蹤、秒殺 Advisory Lock、高並發更新、Lock Flooding、max_locks_per_transaction、OLTP advisory lock、無間隙 ID 生成 |
| **監控與追溯** | [`monitoring.md`](monitoring.md) | pg_stat_activity 生產實戰（5 場景 + 決策圖）、wait_event Top 10、auto_explain、pg_stat_io、track_commit_timestamp |
| **Vacuum / Bloat** | [`vacuum.md`](vacuum.md) | 生產場景驅動核心概念、4 大 Bloat 成因與診斷 SQL、VM 深度解析（結構/失效/Index Entry）、VACUUM FULL vs pg_repack 二方案對比與決策樹 |
| **資料型別** | [`datatype.md`](datatype.md) | Float vs Numeric 效能對比（360x）、SIMD 向量化、`AT TIME ZONE` 語法解析與型別轉換陷阱 |
| **JSON/JSONB** | [`json.md`](json.md) | Scalar Types（大小寫敏感）、Type I/O 機制、BYTEA Escape 與 base64 方案、Tokenizer 原理、陣列提取與 GIN Index、JSONPath / json_table（PG 12→17）、方案選擇矩陣、生產排查與 App Dev 視角 |
| **系統底層** | [`system.md`](system.md) | Column Order 與 Byte Alignment 全鏈路效能、Bit 位運算標籤系統、Linux Page Fault 與 huge_pages / NUMA |
| **擴充功能** | [`extensions/extensions.md`](extensions/extensions.md) | 兩大分類 13 個 Extension：Non-Contrib（pg_partman / PgBouncer / pg_repack / pg_cron / pg_stat_kcache / hypopg）+ Contrib 內建（pg_stat_statements / auto_explain / pgcrypto / pg_trgm / pg_prewarm / pg_buffercache / btree_gin+btree_gist） |
| **分頁查詢** | [`pagination.md`](pagination.md) | OFFSET 效能退化、CURSOR 方案、Keyset Pagination、分頁優化策略 |
| **其他進階** | [`others.md`](others.md) | PG 17 開發規範（六大章節 + Npgsql 連線/交易/Retry/Prepared Statement/WAL 完整 C# 實戰）、Trigger Audit（DML hstore+JSONB/DDL Event Trigger + pgaudit 對比 + 架構決策速查）、12306 搶票系統全案設計（varbit/SKIP LOCKED/Array+GIN/CURSOR/pgrouting/Parallel Query/Sharding/Recursive CTE） |

---

## 各章節快速導覽

### 資料型別（Datatype）
- **一、Float vs Numeric**：Benchmark（四則/開根號/Pi 計算）、硬體加速 vs 軟體模擬的根本差異、SIMD 向量化優勢、選型建議、版本演進
- **二、AT TIME ZONE**：EXTRACT epoch 的時區陷阱、`gram.y` 語法規則、`timestamptz_part()` vs `timestamp_part()` 的 overload 選擇、三種 Case 逐步分解

### JSON/JSONB（json）
- **一、Value Types 與構造方法**：Scalar Types（大小寫敏感）、jsonb 內部無 Type 概念、Type I/O Function 機制、BYTEA Escape 限制與 base64 方案、Tokenizer 運作原理
- **二、陣列提取與查詢**：`->` / `->>` 操作符、`json_array_elements` + ARRAY 構造器、`@>` / `&&` 陣列操作、GIN on Expression、JSONPath（PG 12+）、`json_table`（PG 17+）、方案選擇矩陣、PG16 Best Practice

### 擴充功能（extensions）
- **# 一、Non-Contrib Extensions（需額外安裝）**
  - **pg_partman**：原生分區自動化管理、自動分區創建/清理、Retention Policy、Background Worker 驅動 partition lifecycle
  - **PgBouncer**：Transaction vs Session vs Statement Pooling、生產級 HAProxy 拓撲
  - **pg_repack**：四階段在線重組原理、vs VACUUM FULL / pg_squeeze 對比、配合 pg_cron 定時執行
  - **pg_cron**：PG 內建排程、定時 VACUUM / 分區維護 / 物化視圖刷新
  - **pg_stat_kcache**：getrusage() 實體 IO 統計、CPU 耗時分析、寫入放大檢測
  - **hypopg**：假設索引 zero-cost 試錯、分區表索引測試
- **# 二、Contrib Extensions（PG 內建，CREATE EXTENSION 即可）**
  - **pg_stat_statements**：查詢歸一化原理、queryid 計算、Top-N 慢查詢/JIT/WAL 分析、PG 16 JIT 計數器
  - **auto_explain**：自動記錄執行計劃、log_analyze/log_buffers/log_triggers/log_nested_statements、生產調校策略
  - **pgcrypto**：digest/hmac 數據校驗、crypt+gen_salt(bf) 密碼儲存、PGP 對稱/公鑰加密、C# Npgsql 實戰
  - **pg_trgm**：Trigram 原理、GIN/GiST 加速 LIKE、similarity() / word_similarity()、與 pg_bigm 對比
  - **pg_prewarm**：手動緩存預熱 + autoprewarm BGW（PG 11+）自動恢復、重啟冷啟動優化
  - **pg_buffercache**：緩存內容即時診斷、usagecount 時鐘演算法、pg_buffercache_summary()（PG 17）
  - **btree_gin / btree_gist**：GIN 多欄位複合類型索引、GiST EXCLUSION CONSTRAINT 擴展

### Vacuum / Bloat（vacuum）
- **一、Vacuum 原理與防止 Bloat**：生產場景驅動核心概念（MVCC/Dead Tuple/OldestXmin）、4 大 Bloat 成因（Long Transaction / 批量操作 / 觸發閾值 / 非 HOT 更新）附診斷 SQL、VM 深度解析（2-bit 結構 / 失效時序圖 / Index Entry 結構 + DDL + mermaid）
- **二、收縮膨脹表**：VACUUM FULL vs pg_repack 二方案對比（FILENODE swap 機制 / Lock 時間估算 / 執行時長參考）、生產決策樹、`REINDEX CONCURRENTLY`（PG 12+）

### 系統底層（System）
- **一、Column Order & Byte Alignment**：ADD COLUMN 永遠在末尾、Simple View 虛擬重排、Byte Alignment 對 Row Size 的影響（padding 可佔 41%）、全鏈路效能鏈式反應
- **二、Bit 位運算標籤系統**：5000 萬用戶/200 標籤實測、`bitand()` 無法用 Index 的瓶頸、替代方案（intarray + RD-Tree、roaringbitmap）、生產環境建議
- **三、Linux Page Fault**：MMU 與虛擬記憶體、Major/Minor/Invalid 三種 Page Fault、大 shared_buffers 啟動低潮案例（minor fault 風暴）、huge_pages / pg_prewarm / NUMA 現代化解方

### 鎖（Lock）
- **一、Lock 機制全景**：8 級 Lock Mode 衝突矩陣、Lock Queue 排隊機制（pending lock 可堵死後續請求）、Object-Level Lock 在 Transaction 結束才釋放、idle in transaction 偵測與預防
- **二、Advisory Lock 應用場景**：秒殺（231K TPS）、高並發全表更新（18x 加速）、OLTP 排隊控制、無間隙 ID 生成與 Lock Flooding 防禦、`max_locks_per_transaction` 配置

### 其他進階（others）
- **一、PG 17 開發規範**：六大章節完整規範體系（命名/設計/Query/管理/穩定性與效能）+ Npgsql 連線字串完整參數表（14 項）+ FK Action 三選一/Partial Index/B-tree 2000 byte 限制/Stored Procedure 減少 Roundtrip 詳解 + WAL 生命週期與 App Dev 防禦 + ETL 大批量載入（COPY/Binary COPY/批次 INSERT）+ Connection Pool/Polly Retry/Advisory Lock 完整 C# 實作 + Prepared Statement 兩種 Protocol/Named Statement `_p1` 誕生全過程/custom→generic plan 切換機制 + 章末 Developer 落地檢查清單（12 項）
- **二、Trigger Audit**：DML 審計（JSONB Row Trigger 欄位級 diff + key-level FULL OUTER JOIN 比對）+ DDL 審計（Event Trigger + pg_stat_activity context 快照）+ pgaudit C 層審計 vs plpgsql Trigger vs WAL Logical Decoding 三方案對比 + 架構決策速查表（5 場景）
- **三、12306 搶票系統**：10 大法寶全景設計 — varbit 座位區段 + Array + GIN 查詢車次（generated column station_pos）+ SKIP LOCKED 防 Lock 衝突（hash 分流防熱點）+ CURSOR 分頁（SCROLL CURSOR 前後翻頁 + WITH HOLD）+ pgrouting 路徑規劃 + **Parallel Query** 多核加速餘票統計（觸發條件/EXPLAIN 對比/PG 9.6→17 演進/parallel_leader_participation）+ **Sharding** 三方案對比（Citus/FDW/Application-Level + Shard Key 選擇陷阱 + 冷熱分離三層策略）+ **Recursive CTE** 圖遍歷（BFS/DFS/環路檢測 PG 14+ CYCLE 子句/跨版本相容寫法）

### 查詢深度解析（Query）
- **一、查詢生命週期**：Client Request → Parser → Analyzer → Planner → Executor 六階段逐層拆解、Process-per-Connection 架構設計、三層 Tree 轉換（Parse/Plan/Executor）
- **二、CBO 與 pg_hint_plan**：Cost-Based Optimizer 盲區分析、何時需要 Hint 介入、pg_hint_plan 使用方法與注意事項
- **三、GROUP BY 策略**：Sort (GroupAgg) vs Hash (HashAgg) 兩種實作路徑、Planner 選擇邏輯與調校
- **四、IN / =ANY / VALUES 效能對決**：四種等效寫法的底層差異、JOIN VALUES 展開陷阱
- **五、分頁與計數優化**：count(*) 替代策略、OFFSET 退化分析、Keyset 位點
- **六、Recursive CTE 優化**：Index Skip Scan 模擬、Top-N Per Group（44x 加速）
- **七、Recursive CTE 死循環防禦**：CYCLE 語法（PG 14+）、Production 防禦體系
- **八、分組 Top-N**：Recursive CTE + Index Loop vs Window Function（44x 加速）
- **九、JOIN 冗餘膨脹 Early DISTINCT**：笛卡爾乘積膨脹的根因、每層 JOIN 後立即去重、重現實驗、CTE 簡化寫法（數十秒→數十毫秒）

### 監控與追溯（Monitoring）
- **一、慢查詢追溯體系**：pg_stat_activity 生產實戰（5 場景 + 30 秒決策圖 + 鎖阻塞/Plan 不穩定/連線池滿/間歇性慢查詢）、wait_event Top 10、Snapshot vs Time-Series、應用層 backend_pid 記錄
- **二、track_commit_timestamp**：SLRU 儲存結構、logical replication / snapshot too old / CDC 應用場景、效能影響與 Production 取捨

### Transaction 全攻略（transaction）
- **一、隔離級別的底層：MVCC 與 Snapshot**：Snapshot 結構（xmin/xmax/xip[]）深度回顧、`GetTransactionSnapshot()` 內部機制、RC vs RR 的 Snapshot 獲取時序對比（stateDiagram）、PG 為何用 Snapshot 而非 Lock
- **二～五、四種隔離級別完整解析**：Read Uncommitted（PG 中等於 RC 的 MVCC 根本原因）、Read Committed（每個 Statement 新 Snapshot / 幻讀重現 / 生產場景選擇表）、Repeatable Read（gantt 時序圖 / PG 額外防止 Phantom Read 的 snapshot 邊界原理 / Write Skew 完整重現 + Mermaid)、Serializable（SSI 原理 / SIREAD lock page-level 限制 / wr/ww/rw 三種依賴 / serialization failure 觸發流程 / 效能 overhead 量化）
- **六、隔離級別對 VACUUM 的影響**：OldestXmin 計算、RR transaction 如何阻止 dead tuple 回收（Mermaid 災難鏈）、找出卡住 VACUUM 的 session、`old_snapshot_threshold` / `idle_in_transaction_session_timeout` 解法
- **三、分佈式事務實現方式（NEW）**：CAP/BASE 理論、2PC（PREPARE TRANSACTION）/ 3PC / XA + TransactionScope C# 範例、Saga（Choreography/Orchestration）、TCC（Try-Confirm-Cancel）、Outbox Pattern + BackgroundService + 冪等處理、Seata AT Mode、方案對比與選型決策圖
- **生產環境場景**：轉帳 Phantom（FOR UPDATE / RR / Serializable 解法矩陣）、Write Skew 案例（值班醫生 SQL 重現）、EvalPlanQual / Lost Update、隔離級別選擇決策圖（Mermaid flowchart + 速查對照表）
- **.NET 實戰**：Npgsql IsolationLevel enum 完整對照、Dapper 正確/錯誤寫法對比（FOR UPDATE + transaction 傳遞）、Polly Retry Pattern（`SqlState 40001` serialization failure）

### 索引全解析（Index）
- **一、六種掃描類型全解析**：Seq Scan / Index Scan / Bitmap Heap Scan / Index-Only Scan / Parallel Scan / TID Scan 的 Planner 選擇邏輯與成本模型
- **二、Index 核心機制**：B-Tree / Hash 內部結構、Covering Index（INCLUDE）、Leaf Page 圖解、Recheck Cond 詳解
- **三、BRIN Index**：Block Range Index 原理（540x 小於 B-tree）、適用場景（時序/append-only）
- **四、Bloom Index**：單一索引支撐任意 Column 組合查詢、簽名長度調校
- **五～七、模糊查詢索引全景**：GIN / GiST / SP-GiST / RUM 內部機制、全文檢索排序、GIN+LIMIT 慢的原因與 RUM 解法
- **八、索引失效 20 場景**：每個場景附 EXPLAIN ANALYZE 輸出對比與可行解法

### 分頁查詢（Pagination）
- **一、OFFSET 基本原理**：OFFSET 不是「跳過」而是「計算後丟棄」、VOLATILE function 的放大效應、SQL 邏輯執行順序
- **二、OFFSET 的質變**：數據斷層引發掃描量跳升、Rows Removed by Filter 暴增、Production 真實案例
- **三、分頁優化方案**：Keyset Pagination / CURSOR / ismax 標記列四種方案對比與選擇指南

---

## 關於本 Repo

### 內容來源

部分內容參考自 [digoal (德哥) 的 PostgreSQL blog](https://github.com/digoal/blog)，經大幅改寫、去除重複、補充 PG 9~18 版本演進資訊、Mermaid 圖視覺化與新手導向說明。

### 使用方式

- 每份 md 為該主題的完整學習手冊，由淺到深排列
- `images/` 目錄存放相關圖片
- `AGENTS.md` 為本專案的協作規範（標題層級、Mermaid 風格、提交規範等）
- 專業名詞使用英文（TID、Recheck Cond、OldestXmin 等）
- 版本標注：`> 更新於 2026-05-17，補充 PG X~X 新增能力`

### License

MIT License
