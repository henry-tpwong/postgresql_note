# PostgreSQL 深度學習筆記

> 整理自 [digoal (德哥) blog](https://github.com/digoal/blog)，輔以 PG 9~18 版本演進更新與 senior developer 實戰見解。

---

## 目錄

### Index 索引

| 筆記 | 主題 |
|------|------|
| [PostgreSQL_index.md](PostgreSQL_index.md) | Bitmap Heap Scan / Recheck Cond / EXPLAIN 分析 / BRIN / 索引總覽 / Leaf Page 與 Covering Index |
| [PostgreSQL_BRIN.md](PostgreSQL_BRIN.md) | BRIN 索引原理、體積對比（540x 小於 B-tree）、物聯網場景 |
| [PostgreSQL_Bloom_Index.md](PostgreSQL_Bloom_Index.md) | Bloom filter 索引：單索引支援多列組合查詢 |
| [PostgreSQL_RUM_index.md](PostgreSQL_RUM_index.md) | RUM index：支援 ranking / phrase search / 雙欄位複合 |
| [PostgreSQL_GIN_LIMIT_Slow.md](PostgreSQL_GIN_LIMIT_Slow.md) | GIN index + LIMIT 慢的原因與 5 種解法 |
| [PostgreSQL_Fuzzy_Search_Index.md](PostgreSQL_Fuzzy_Search_Index.md) | GIN / GiST / SP-GiST / RUM 模糊查詢原理與選擇決策 |
| [PostgreSQL_Scan_Types_EXPLAIN.md](PostgreSQL_Scan_Types_EXPLAIN.md) | EXPLAIN 掃描類型全解析：Seq Scan / Index Scan / Bitmap Scan / TID Scan |

### 查詢效能優化

| 筆記 | 主題 |
|------|------|
| [PostgreSQL_IN_vs_ANY_vs_VALUES.md](PostgreSQL_IN_vs_ANY_vs_VALUES.md) | IN / =ANY(ARRAY) / =ANY(VALUES) / JOIN VALUES 效能對決（Datadog 100x case） |
| [PostgreSQL_GroupAgg_vs_HashAgg.md](PostgreSQL_GroupAgg_vs_HashAgg.md) | GROUP BY Sort vs Hash 的 cost model 與選擇邏輯 |
| [PostgreSQL_count_estimate_keyset.md](PostgreSQL_count_estimate_keyset.md) | COUNT 估算與 Keyset Pagination |
| [PostgreSQL_Group_TopN_Recursive.md](PostgreSQL_Group_TopN_Recursive.md) | Group TopN 與遞迴查詢優化 |
| [PostgreSQL_Recursive_Optimization.md](PostgreSQL_Recursive_Optimization.md) | Recursive CTE 效能優化 |
| [PostgreSQL_Recursive_Cycle.md](PostgreSQL_Recursive_Cycle.md) | Recursive CTE CYCLE 檢測 |
| [PostgreSQL_pg_hint_plan.md](PostgreSQL_pg_hint_plan.md) | pg_hint_plan 強制執行計劃 |
| [PostgreSQL_Query_Lifecycle.md](PostgreSQL_Query_Lifecycle.md) | 查詢生命週期（parser → planner → executor） |

### Vacuum / Bloat / 空間回收

| 筆記 | 主題 |
|------|------|
| [PostgreSQL_Vacuum_Bloat.md](PostgreSQL_Vacuum_Bloat.md) | Vacuum 原理、8 大 bloat 成因、預防措施、`OldestXmin` 機制 |
| [PostgreSQL_Table_Bloat_Repack_Squeeze.md](PostgreSQL_Table_Bloat_Repack_Squeeze.md) | 表膨脹檢測與 pg_repack / pg_squeeze 回收 |

### Lock / 並發

| 筆記 | 主題 |
|------|------|
| [PostgreSQL_Implicit_Lock.md](PostgreSQL_Implicit_Lock.md) | 隱式鎖請求：`pg_get_indexdef` 等函數的隱性 table lock 陷阱與 lock queue pileup |
| [PostgreSQL_Lock_Wait_Trace.md](PostgreSQL_Lock_Wait_Trace.md) | Lock wait 追蹤三法：`log_lock_waits` / `pg_stat_activity` / `LOCK_DEBUG` |
| [PostgreSQL_Flash_Sale_Advisory_Lock.md](PostgreSQL_Flash_Sale_Advisory_Lock.md) | 秒殺場景：Advisory Lock vs FOR UPDATE NOWAIT vs SKIP LOCKED（231K TPS） |
| [PostgreSQL_High_Concurrency_Update.md](PostgreSQL_High_Concurrency_Update.md) | 高並發全表更新：Advisory Lock / SKIP LOCKED / 分區方案（18x 加速） |
| [PostgreSQL_Lock_Flooding.md](PostgreSQL_Lock_Flooding.md) | Lock flooding 現象與 `max_locks_per_transaction` |
| [PostgreSQL_max_locks_per_transaction.md](PostgreSQL_max_locks_per_transaction.md) | `max_locks_per_transaction` 配置指南 |
| [PostgreSQL_OLTP_advisory_lock.md](PostgreSQL_OLTP_advisory_lock.md) | OLTP 高並發 advisory lock 場景 |
| [PostgreSQL_advisory_lock_gapfree_id.md](PostgreSQL_advisory_lock_gapfree_id.md) | Advisory lock + 無間隙 ID 生成 |

### JSON / JSONB

| 筆記 | 主題 |
|------|------|
| [PostgreSQL_JSON_Array_Extract.md](PostgreSQL_JSON_Array_Extract.md) | JSON array→SQL array 轉換、GIN 索引、JSONPath / SQL/JSON 標準（到 PG 17 `json_table`） |
| [PostgreSQL_JSONB_Construct.md](PostgreSQL_JSONB_Construct.md) | JSONB 建構與效能 |

### 全文檢索 (FTS)

| 筆記 | 主題 |
|------|------|
| [PostgreSQL_Chinese_FTS_zhparser.md](PostgreSQL_Chinese_FTS_zhparser.md) | 中文全文檢索：zhparser + SCWS 安裝、token type mapping、雲端相容矩陣 |
| [PostgreSQL_FTS_whole_row.md](PostgreSQL_FTS_whole_row.md) | 整行 FTS 索引 |
| [PostgreSQL_Whole_Row_FTS.md](PostgreSQL_Whole_Row_FTS.md) | 整行全文檢索方案 |

### 系統底層

| 筆記 | 主題 |
|------|------|
| [PostgreSQL_Page_Fault.md](PostgreSQL_Page_Fault.md) | Linux page fault 對 PG 性能影響、huge_pages / THP / NUMA / `pg_prewarm` |
| [PostgreSQL_bit_ops_perf.md](PostgreSQL_bit_ops_perf.md) | Bit 操作效能對比 |
| [PostgreSQL_Column_Order_Alignment.md](PostgreSQL_Column_Order_Alignment.md) | Column order 與記憶體對齊對 tuple 大小的影響 |

### 監控 / 排查

| 筆記 | 主題 |
|------|------|
| [PostgreSQL_Slow_Query_Trace.md](PostgreSQL_Slow_Query_Trace.md) | 慢查詢追溯體系：`pg_stat_activity` / `auto_explain` / `pg_stat_io` / wait_event |
| [PostgreSQL_slow_query_tracing.md](PostgreSQL_slow_query_tracing.md) | 慢查詢追蹤方法 |
| [PostgreSQL_track_commit_timestamp.md](PostgreSQL_track_commit_timestamp.md) | `track_commit_timestamp` 功能與應用 |

### 分頁 / OFFSET

| 筆記 | 主題 |
|------|------|
| [PostgreSQL_OFFSET.md](PostgreSQL_OFFSET.md) | OFFSET 原理：跳過 row 仍被計算、`VOLATILE` function 陷阱 |
| [PostgreSQL_OFFSET_Gap.md](PostgreSQL_OFFSET_Gap.md) | OFFSET 造成的查詢缺口問題 |
| [PostgreSQL_分頁優化.md](PostgreSQL_分頁優化.md) | 分頁效能優化 |

### 資料型別 / Function

| 筆記 | 主題 |
|------|------|
| [PostgreSQL_Float_vs_Numeric.md](PostgreSQL_Float_vs_Numeric.md) | Float vs Numeric 性能對比（360x sqrt 差距）、SIMD 向量化 |
| [PostgreSQL_AT_TIME_ZONE.md](PostgreSQL_AT_TIME_ZONE.md) | `AT TIME ZONE` 行為詳解 |

### Partition / Extension / FDW

| 筆記 | 主題 |
|------|------|
| [PostgreSQL_pg_pathman.md](PostgreSQL_pg_pathman.md) | pg_pathman 分區管理 extension |
| [PostgreSQL_pg_shard.md](PostgreSQL_pg_shard.md) | pg_shard 分散式方案 |
| [PostgreSQL_FDW_ImportForeignSchema.md](PostgreSQL_FDW_ImportForeignSchema.md) | `IMPORT FOREIGN SCHEMA` 用法 |

### 其他

| 筆記 | 主題 |
|------|------|
| [PostgreSQL_12306_Ticket_System.md](PostgreSQL_12306_Ticket_System.md) | 12306 訂票系統設計思路 |
| [PostgreSQL_JOIN_Redundancy.md](PostgreSQL_JOIN_Redundancy.md) | JOIN 冗餘分析 |
| [PostgreSQL_pgcrypto.md](PostgreSQL_pgcrypto.md) | pgcrypto 加密 extension |
| [PostgreSQL_Trigger_Audit.md](PostgreSQL_Trigger_Audit.md) | Trigger 審計 |
| [PostgreSQL_pg_trgm_Regex_100Billion.md](PostgreSQL_pg_trgm_Regex_100Billion.md) | pg_trgm 正則 100 億級別案例 |
| [PostgreSQL_Dev_Standards.md](PostgreSQL_Dev_Standards.md) | PG 17 開發規範：命名、索引、SQL 編寫、schema 設計（標記過時規則） |

---

## 關於本 Repo

### 內容來源

筆記原始內容主要來自 [digoal (德哥) 的 PostgreSQL blog](https://github.com/digoal/blog)，經結構化整理、去除重複、補充 PG 9~18 版本演進資訊與 senior developer 實戰見解。


### 使用方式

- 每篇 `.md` 獨立成章，可直接閱讀
- `images/` 目錄存放相關圖片
- 專業名詞使用英文（TID、Recheck Cond、OldestXmin 等）
- 版本標注：`> 更新於 2026-05-17，補充 PG 14~18 新增能力`

### License

原始內容版權歸 [digoal](https://github.com/digoal) 所有。整理、補充部分以 MIT License 釋出。
