# pg_hint_plan：PostgreSQL SQL Hint 的原理與使用

> 來源：[digoal - PostgreSQL SQL HINT的使用(pg_hint_plan) (2016-02-03)](https://github.com/digoal/blog/blob/master/201602/20160203_01.md)

---

## 背景：CBO 是否每次都能產生最優 Plan？

PostgreSQL 使用 Cost-Based Optimizer（CBO）。當關聯表數量超過 `geqo_threshold`（預設 12）時，會改用 Genetic Query Optimizer（GEQO）以避免窮舉帶來的 planning 開銷，但 GEQO 結果不保證最優。

本文聚焦 CBO：它是否**每次都**能輸出最優執行計劃？

### CBO 的成本計算因子

來源：`src/backend/optimizer/path/costsize.c`

| 因子類別 | 具體項目 | 影響 |
|---------|---------|------|
| 物理屬性 | table 的 data block 數量 | 影響 scan block 成本（Seq Scan / Index Scan） |
| 物理屬性 | table 的 row 數量 | 影響 CPU 處理 row 成本 |
| 成本因子 | `seq_page_cost` / `random_page_cost` | 連續 vs 隨機讀取單個 block 的成本 |
| 成本因子 | `cpu_tuple_cost` / `cpu_index_tuple_cost` | CPU 處理 heap/index 中一條 row 的成本 |
| 成本因子 | `cpu_operator_cost` | 執行一個 operator 或 function 的成本 |
| Memory | `work_mem` 等 | 影響 Index Scan 的計算成本 |
| 統計資訊 | correlation（物理順序與 index 順序的離散度） | 影響 Index Scan 成本估算 |
| 統計資訊 | column width、null fraction、n_distinct、MCV、histogram buckets | 影響選擇性（selectivity），從而影響估計 row 數 |
| 函數屬性 | 建立 function / operator 時設置的 cost | 影響含 function call 的查詢成本 |

### CBO 的盲區

**1. 已考慮但可能滯後的因素（auto-analyze 自動更新）：**

```
-- pg_class.relpages, pg_class.reltuples：block 數與 row 數
-- pg_statistic：column 統計資訊、correlation
```

自動 analyze 有延遲。若表在兩次 analyze 之間發生大量變更（bulk INSERT / DELETE），statistics 可能嚴重失準，導致 CBO 選擇錯誤 plan。

**2. 靜態配置因素（不會自動調整）：**

- **可用作 cache 的實際記憶體**：OS 上可能運行其他程式，database session 大量使用 `work_mem` 時也會擠壓可用 cache。`effective_cache_size` 是靜態設定，無法反映即時變化。
- **Function / operator cost**：function 內部邏輯或 SQL 變更後，實際執行時間可能已變化，但 cost 不自動更新。

**3. 未考慮的因素：**

- **Block device read-ahead**：一般一次讀取會預讀 128KB（`blockdev --getra` 查看）。如果要讀取的資料在連續的 128KB block 中，實際上只需一次 I/O，但 CBO 仍按逐個 block 計算成本。這對不同 block device（HDD vs SSD）或不同的 read-ahead 設定會有細微影響。
- `seq_page_cost` / `random_page_cost` 可以針對 tablespace 設置來補償不同設備差異，但不會動態追蹤 read-ahead 變更。

**4. Generic plan cache（plancache.c — `choose_custom_plan`）：**

PostgreSQL 比對 cached plan 的平均成本與 custom plan 的成本：若 cached plan 成本更高，會改用 custom plan。前提是**統計資訊必須準確**，否則無法及時發現 cached plan 的問題。

### 小結：何時需要 Hint

大多數情況下，合理配置 `random_page_cost`、定期 `ANALYZE`、調整 `default_statistics_target` 就足夠。需要 Hint 的場景：

- 靜態配置與實際不符（如 function cost 過時）
- 特定查詢的 optimizer 估計與實際偏差巨大，且無法僅靠調整 cost factor 修正
- 除錯：想對比不同 plan 的實際效率
- 緊急情況：production 中某個查詢突然選擇了差的 plan，需要立即凍結正確 plan

> 補充（Senior Dev）：Hint 是把雙面刃。寫在 application code 中的 hint 會在資料成長後過時（plan rot），導致原本正確的 hint 變成效能殺手。應優先嘗試非侵入式方案（調整 `enable_*` GUC、extended statistics、調整 `random_page_cost`），hint 作為最後手段。

---

## pg_hint_plan 簡介

`pg_hint_plan` 是 PostgreSQL extension，利用 PG 開放的 hook 介面（`_PG_init`），在不修改 PG source code 的前提下實現 Oracle-style SQL hint。

因不同 PG 版本 plan 層 source code 可能不一致，`pg_hint_plan` 需搭配對應 PG 版本編譯。

### 安裝（PG 9.4 示範）

```bash
wget http://iij.dl.sourceforge.jp/pghintplan/62456/pg_hint_plan94-1.1.3.tar.gz
tar -zxvf pg_hint_plan94-1.1.3.tar.gz
cd pg_hint_plan94-1.1.3
export PATH=/opt/pgsql/bin:$PATH
gmake clean && gmake && gmake install
```

```ini
# postgresql.conf
shared_preload_libraries = 'pg_hint_plan'
pg_hint_plan.enable_hint = on
pg_hint_plan.debug_print = on
pg_hint_plan.message_level = log
```

```bash
pg_ctl restart -m fast
```

```sql
CREATE EXTENSION pg_hint_plan;
```

> [PG 12+] 自 `pg_hint_plan` 對應 PG 12 的版本起，支援 **hint table**（`hint_plan.hints`），可在 table 中管理 hint 而無需修改 SQL text。這是 production 環境的關鍵功能——無需改 code，透過 INSERT/UPDATE hint table 即可動態調整 plan：
>
> ```sql
> CREATE EXTENSION pg_hint_plan;
> -- hint table 由 extension 自動建立
> INSERT INTO hint_plan.hints (norm_query_string, application_name, hints)
> VALUES ('SELECT * FROM orders WHERE create_time > $1', '', 'IndexScan(orders idx_orders_time)');
> ```
>
> hint table 支援 regex pattern matching，可針對不同 application / user / query pattern 設定 hint。

### Hint vs GUC Switch 對比

**原生 GUC 開關：**

```sql
SET enable_nestloop = off;
EXPLAIN SELECT a.*, b.* FROM a, b WHERE a.id = b.id AND a.id < 10;
-- 結果：Hash Join（禁用了 Nested Loop）

SET enable_nestloop = on;
EXPLAIN SELECT a.*, b.* FROM a, b WHERE a.id = b.id AND a.id < 10;
-- 結果：Nested Loop（恢復預設）
```

**pg_hint_plan hint：**

```sql
/*+
  HashJoin(a b)
  SeqScan(b)
*/
EXPLAIN SELECT a.*, b.* FROM a, b WHERE a.id = b.id AND a.id < 10;
-- 結果：Hash Join + Seq Scan on b，與 GUC 開關效果相同但更精確
```

Hint 相較 GUC 的優勢：GUC 是 session-level（影響該 session 中所有後續查詢），hint 是 query-level（只影響該條 SQL），粒度更精細。

```sql
-- Scan 類型控制
/*+ SeqScan(a) */
EXPLAIN SELECT * FROM a WHERE id < 10;
-- Seq Scan on a

/*+ BitmapScan(a) */
EXPLAIN SELECT * FROM a WHERE id < 10;
-- Bitmap Heap Scan on a
```

### 完整 Hint 列表

| Hint | 作用 |
|------|------|
| `SeqScan(table)` | 強制 Seq Scan |
| `TidScan(table)` | 強制 TID Scan |
| `IndexScan(table [index...])` | 強制 Index Scan，可指定特定 index |
| `IndexOnlyScan(table [index...])` | 強制 Index Only Scan（PG 9.2+） |
| `BitmapScan(table [index...])` | 強制 Bitmap Scan |
| `NoSeqScan(table)` | 禁止 Seq Scan |
| `NoTidScan(table)` | 禁止 TID Scan |
| `NoIndexScan(table)` | 禁止 Index Scan 與 Index Only Scan |
| `NoIndexOnlyScan(table)` | 禁止 Index Only Scan（PG 9.2+） |
| `NoBitmapScan(table)` | 禁止 Bitmap Scan |
| `NestLoop(table table [table...])` | 強制 Nested Loop Join |
| `HashJoin(table table [table...])` | 強制 Hash Join |
| `MergeJoin(table table [table...])` | 強制 Merge Join |
| `NoNestLoop(table table [table...])` | 禁止 Nested Loop Join |
| `NoHashJoin(table table [table...])` | 禁止 Hash Join |
| `NoMergeJoin(table table [table...])` | 禁止 Merge Join |
| `Leading(table table [table...])` | 強制 Join Order |
| `Leading((...))` | 強制 Join Order 與方向（巢狀括號語法） |
| `Rows(table table [table...] correction)` | 修正 join result 的 row estimate（支援 `#`、`+`、`-`、`*`） |
| `Set(GUC-param value)` | 在 planner 執行期間設定 GUC 參數 |

> 補充（Senior Dev）：實務中最常用的三個 hint：
> 1. **`Leading`** — 修復 join order 錯誤（CBO 最常見的失誤是 join order estimation 偏差，因 row estimate error 在 multi-join 中指數放大）
> 2. **`Rows`** — 直接修正 row estimate，比換 scan/join 方式更根本
> 3. **`Set`** — 臨時覆蓋 `enable_*` 或 `work_mem` 等參數，適合無法修改 global config 的 managed service 環境（如 RDS）

---

## CBO 成本計算的完整鏈路

```
table statistics (pg_class, pg_statistic)
         ↓
cost factors (seq_page_cost, random_page_cost, cpu_*_cost)
         ↓
scan cost estimation (costsize.c)
         ↓
join order enumeration (dynamic programming / GEQO)
         ↓
join method selection (NestLoop / HashJoin / MergeJoin)
         ↓
final plan selection (lowest total cost)
```

### plan cache 的 choose_custom_plan 邏輯

來源：`src/backend/utils/cache/plancache.c`

PostgreSQL 對 prepared statement 的 plan 有兩種模式：
- **custom plan**：每次重新規劃
- **generic plan**：使用緩存的通用 plan（首次 5 次執行使用 custom，第 6 次開始比較成本）

選擇邏輯：若 `cached_plan_cost > avg_custom_plan_cost`，改用 custom plan。這意味著即使使用了 prepared statement，PostgreSQL 仍有機會在統計資訊變化後自動切換 plan——前提是 `auto_analyze` 正常運作。

---

## 替代方案與最佳實踐

> 補充（Senior Dev）：在使用 pg_hint_plan 之前，應先嘗試以下方案，由輕到重：

| 優先級 | 方案 | 適用場景 |
|--------|------|---------|
| 1 | 調整 `random_page_cost` / `seq_page_cost` | SSD 環境 cost factor 不準確（SSD: `random_page_cost = 1.0-1.5`） |
| 2 | 增加 `default_statistics_target` | 單一 column 統計資訊不足 |
| 3 | CREATE STATISTICS（PG 10+ extended stats） | 多 column 之間有相關性（如 `country` 與 `city`） |
| 4 | `SET enable_nestloop = off` 等 GUC（per session/transaction） | 臨時性調整，不修改 code |
| 5 | 調整 `join_collapse_limit` / `from_collapse_limit` | 顯式 JOIN 順序被限制時 |
| 6 | pg_hint_plan hint table（PG 12+） | 需長期、可控的 plan 修正 |
| 7 | pg_hint_plan inline hint（`/*+ ... */`） | 最後手段，query-specific fix |

> 補充（Senior Dev）：PG 10+ 的 **extended statistics** 可以直接解決原文中 CBO 的「未考慮 cross-column correlation」盲區：
>
> ```sql
> CREATE STATISTICS s1 (dependencies, mcv) ON country, city FROM users;
> ANALYZE users;
> ```
>
> 這讓 CBO 能夠理解 `country='US'` 時 `city` 的分佈大大不同於全域分佈，從根本上修正 selectivity estimate，比用 `Rows` hint 手動修復更可靠。

---

## 源碼參考

1. `src/backend/optimizer/path/costsize.c` — CBO 成本計算（scan cost、join cost、全部 cost factor）
2. `src/backend/utils/cache/plancache.c` — `choose_custom_plan`：cached plan vs custom plan 選擇邏輯
3. [pg_hint_plan 官方文件](http://pghintplan.sourceforge.jp/pg_hint_plan-en.html)
4. [pg_hint_plan hint 列表](http://pghintplan.sourceforge.jp/hint_list.html)
5. [pg_hint_plan GitHub](https://github.com/ossc-db/pg_hint_plan/releases)
6. [pg_hint_plan BigSQL Docs](https://www.openscg.com/bigsql/docs/hintplan/)

> [PG 版本註] 原文基於 PG 9.4.1 + pg_hint_plan 1.1.3（2016）。核心 hint 語法與 CBO 成本計算模型在最新版本（PG 17+）不變。主要演進：
> - PG 10+ `CREATE STATISTICS`（extended stats）補上了 CBO 對 cross-column correlation 的盲區
> - PG 12+ pg_hint_plan 支援 hint table（`hint_plan.hints`），無需改 SQL text 即可管理 hint
> - PG 12+ planner 支援 `FETCH FIRST ... WITH TIES`
> - PG 14+ 改進了多 table join 時的 selectivity estimation
> - 原文中的全部 hint 類型在最新版 pg_hint_plan 均向後相容
