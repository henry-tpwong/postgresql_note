# PostgreSQL 深度學習筆記

> 部分內容參考自 [digoal (德哥) blog](https://github.com/digoal/blog)，經大幅改寫、補充 PG 9~18 版本演進、Mermaid 圖與新手導向說明。
> 適合 Developer 閱讀，著重觀念理解而非深入原始碼。每份 md 均由多篇原始筆記合併而成，依由淺到深排列，含標題層級（`# 一、` → `## 1.` → `### I.`）與 Mermaid 圖輔助說明。

---

## 目錄

| 章節 | 檔案 | 主題 |
|------|------|------|
| **資料型別** | [`datatype/PostgreSQL_Datatype.md`](datatype/PostgreSQL_Datatype.md) | Float vs Numeric 效能對比（360x）、SIMD 向量化、`AT TIME ZONE` 語法解析與型別轉換陷阱 |
| **JSON/JSONB** | [`json/PostgreSQL_JSON.md`](json/PostgreSQL_JSON.md) | JSONB Value Types、Type I/O 機制、陣列提取與 GIN Index、JSONPath / SQL/JSON / json_table（PG 12→17） |
| **全文檢索** | [`fulltext/PostgreSQL_Fulltext.md`](fulltext/PostgreSQL_Fulltext.md) | zhparser 中文分詞、Whole-Row FTS（Generated Column）、record_out + SCWS 逗號問題與解法 |
| **擴充功能** | [`extensions/PostgreSQL_Extensions.md`](extensions/PostgreSQL_Extensions.md) | IMPORT FOREIGN SCHEMA 跨庫連線、pg_pathman 高效分區（Custom Scan API）、pg_shard 分散式分片（Citus 前身） |
| **Vacuum / Bloat** | [`vacuum/PostgreSQL_Vacuum.md`](vacuum/PostgreSQL_Vacuum.md) | Bloat 8 大成因與測試驗證、預防措施、VACUUM FULL vs pg_repack vs pg_squeeze 三方案對比 |
| **系統底層** | [`system/PostgreSQL_System.md`](system/PostgreSQL_System.md) | Column Order 與 Byte Alignment 全鏈路效能、Bit 位運算標籤系統、Linux Page Fault 與 huge_pages / NUMA |
| **鎖（Lock）** | [`lock/PostgreSQL_Lock.md`](lock/PostgreSQL_Lock.md) | 隱式鎖、Lock Wait 追蹤、秒殺 Advisory Lock、高並發更新、Lock Flooding、max_locks_per_transaction、OLTP advisory lock、無間隙 ID 生成 |
| **其他進階** | [`others/PostgreSQL_Others.md`](others/PostgreSQL_Others.md) | PG 17 開發規範、Trigger Audit（DML+DDL）、JOIN 冗餘 Early DISTINCT、pgcrypto 加密、千億級 pg_trgm Regex、12306 搶票架構設計 |
| **查詢深度解析** | [`PostgreSQL_Query.md`](PostgreSQL_Query.md) | 查詢生命週期、CBO 與 pg_hint_plan、GROUP BY 策略、IN/ANY/VALUES、分頁與計數、Recursive CTE 優化、死循環防禦 |
| **監控與追溯** | [`PostgreSQL_Monitoring.md`](PostgreSQL_Monitoring.md) | 慢查詢追溯體系、`pg_stat_activity` / `auto_explain` / `pg_stat_io`、`track_commit_timestamp` |
| **索引全解析** | [`PostgreSQL_Index.md`](PostgreSQL_Index.md) | 掃描類型全解析（Seq/Index/Bitmap/Parallel/Index-Only）、Bitmap Heap Scan 詳解、BRIN/Bloom/GIN/GiST/SP-GiST/RUM、Covering Index |
| **分頁查詢** | [`PostgreSQL_pagination.md`](PostgreSQL_pagination.md) | OFFSET 效能退化、CURSOR 方案、Keyset Pagination、分頁優化策略 |

---

## 各章節快速導覽

### 資料型別（datatype）
- **一、Float vs Numeric**：Benchmark（四則/開根號/Pi 計算）、硬體加速 vs 軟體模擬的根本差異、SIMD 向量化優勢、選型建議、版本演進
- **二、AT TIME ZONE**：EXTRACT epoch 的時區陷阱、`gram.y` 語法規則、`timestamptz_part()` vs `timestamp_part()` 的 overload 選擇、三種 Case 逐步分解

### JSON/JSONB（json）
- **一、Value Types 與構造方法**：Scalar Types（大小寫敏感）、jsonb 內部無 Type 概念、Type I/O Function 機制、`format() + jsonb_in()` 構造法、BYTEA Escape 限制、`json_lex()` Tokenizer 完整邏輯
- **二、陣列提取與查詢**：`->` / `->>` 操作符、`json_array_elements` + ARRAY 構造器、`@>` / `&&` 陣列操作、GIN on Expression、JSONPath（PG 12+）、SQL/JSON 標準函數（PG 15+）、`json_table`（PG 17+）

### 全文檢索（fulltext）
- **一、zhparser 中文全文檢索**：SCWS 分詞引擎、Token Type Mapping（實詞 vs 虛詞）、分詞效果測試、完整部署流程、效能調校（GIN/GiST/RUM 選擇）
- **二、Whole-Row FTS（PG 17 視角）**：Generated Column + GIN 現代方案、Legacy IMMUTABLE 繞過法、`t::text` 格式限制、加權檢索、中文分詞 Extension 選擇
- **三、record_out + SCWS 逗號問題**：`record_out` 序列化格式、SCWS 將逗號解析為 auxiliary token 導致截斷、`replace(, → ' ')` 解法、分詞效能基準（4.44 萬字/s）

### 擴充功能（extensions）
- **一、IMPORT FOREIGN SCHEMA**：`LIMIT TO` / `EXCEPT` 過濾、View/Materialized View/Foreign Table 一併導入、串聯 FDW 注意事項、限制與適用範圍
- **二、pg_pathman**：Custom Scan API + HOOK 實作、`RuntimeAppend` runtime partition pruning、Range/Hash 分區 API、Split/Merge/Append/Prepend、非阻塞式資料遷移、Insert 4.1x 加速
- **三、pg_shard（Citus 前身）**：Hash Sharding、Metadata 三表（partition/shard/shard_placement）、Replica 與故障修復、`master_copy_shard_placement` full-copy 策略、限制清單

### Vacuum / Bloat（vacuum）
- **一、Vacuum 原理與防止 Bloat**：8 大 Bloat 成因（Long Transaction 為核心）、6 組測試驗證（XID/游標/長查詢/隔離級別/批量更新/naptime）、10 項預防措施、`OldestXmin` 原始碼分析
- **二、收縮膨脹表**：VACUUM FULL vs pg_repack vs pg_squeeze 三方案對比（鎖定時長/Delta 捕捉/效能影響/成熟度）、現代最佳實踐、`REINDEX CONCURRENTLY`（PG 12+）

### 系統底層（system）
- **一、Column Order & Byte Alignment**：ADD COLUMN 永遠在末尾、Simple View 虛擬重排、Byte Alignment 對 Row Size 的影響（padding 可佔 41%）、全鏈路效能鏈式反應
- **二、Bit 位運算標籤系統**：5000 萬用戶/200 標籤實測、`bitand()` 無法用 Index 的瓶頸、替代方案（intarray + RD-Tree、roaringbitmap）、生產環境建議
- **三、Linux Page Fault**：MMU 與虛擬記憶體、Major/Minor/Invalid 三種 Page Fault、大 shared_buffers 啟動低潮案例（minor fault 風暴）、huge_pages / pg_prewarm / NUMA 現代化解方

### 鎖（lock）
- **一、Lock 機制全景**：8 級 Lock Mode 衝突矩陣、Lock Queue 排隊機制（pending lock 可堵死後續請求）、Object-Level Lock 在 Transaction 結束才釋放、idle in transaction 偵測與預防
- **二、Advisory Lock 應用場景**：秒殺（231K TPS）、高並發全表更新（18x 加速）、OLTP 排隊控制、無間隙 ID 生成與 Lock Flooding 防禦、`max_locks_per_transaction` 配置

### 其他進階（others）
- **一、PG 17 開發規範**：命名/設計/Query/管理/穩定性五大類 50+ 條規則，標記 4 條已過時規則
- **二、Trigger Audit**：DML 審計（hstore + Row Trigger 記錄欄位級變更）+ DDL 審計（Event Trigger + hstore）
- **三、JOIN 冗餘膨脹 Early DISTINCT**：笛卡爾乘積膨脹的根因、每層 JOIN 後立即去重的解法、重現實驗與 CTE 簡化寫法
- **四、pgcrypto 加密**：密碼儲存（crypt+gen_salt）、PGP 對稱/公鑰加密、三種方案選擇矩陣
- **五、千億級 Regex 模糊查詢**：1,008 億行 pg_trgm + GIN 效能實測、Trigram 原理、四種查詢模式（Prefix/Suffix/中間/Regex）、pg_bigm 替代方案
- **六、12306 搶票架構**：varbit 座位區段銷售狀態、Array+GIN 車次查詢、SKIP LOCKED 避免 Lock 衝突、pgrouting 路徑規劃、10 大法寶

### 查詢深度解析（Query）
- **一、查詢生命週期**：Client Request → Parser → Analyzer → Planner → Executor 六階段逐層拆解、Process-per-Connection 架構設計、三層 Tree 轉換（Parse/Plan/Executor）
- **二、CBO 與 pg_hint_plan**：Cost-Based Optimizer 盲區分析、何時需要 Hint 介入、pg_hint_plan 使用方法與注意事項
- **三、GROUP BY 策略**：Sort (GroupAgg) vs Hash (HashAgg) 兩種實作路徑、Planner 選擇邏輯與調校
- **四、IN / =ANY / VALUES 效能對決**：四種等效寫法的底層差異、JOIN VALUES 展開陷阱
- **五、分頁與計數優化**：count(*) 替代策略、OFFSET 退化分析、Keyset 位點
- **六、Recursive CTE 優化**：Index Skip Scan 模擬、Top-N Per Group（44x 加速）
- **七、Recursive CTE 死循環防禦**：CYCLE 語法（PG 14+）、Production 防禦體系

### 監控與追溯（Monitoring）
- **一、慢查詢追溯體系**：pg_stat_activity 即時監測 → OS/DB 層級數據採集 → auto_explain / log_lock_waits 自動記錄 → 交叉分析還原案發經過
- **二、track_commit_timestamp**：SLRU 儲存結構、logical replication / snapshot too old / CDC 應用場景、效能影響與 Production 取捨

### 索引全解析（Index）
- **一、六種掃描類型全解析**：Seq Scan / Index Scan / Bitmap Heap Scan / Index-Only Scan / Parallel Scan / TID Scan 的 Planner 選擇邏輯與成本模型
- **二、Index 核心機制**：B-Tree / Hash 內部結構、Covering Index（INCLUDE）、Partial Index、Expression Index
- **三、BRIN Index**：Block Range Index 原理、適用場景（時序/ append-only）、與 B-Tree 的效能對比
- **四、Bloom Index**：單一索引支撐任意 Column 組合查詢、簽名長度調校
- **五～七、模糊查詢索引全景**：GIN / GiST / SP-GiST / RUM 四種索引的內部機制、全文檢索排序、GIN+LIMIT 慢的原因與 RUM 解法

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
- 專業名詞使用英文（TID、Recheck Cond、OldestXmin 等）
- 版本標注：`> 更新於 2026-05-17，補充 PG X~X 新增能力`

### License

MIT License
