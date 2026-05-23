# PostgreSQL 查詢深度解析 — 從生命週期到進階優化

> **閱讀指引**：本文由 `/query/` 目錄下 8 篇獨立筆記合併而成，依 **由淺到深** 排序：
>
> - **第一章** 從一條 SQL 的完整生命週期開始，理解 PostgreSQL 內部到底發生了什麼
> - **第二章** 探討 CBO 的盲區以及何時需要 Hint 介入
> - **第三～五章** 深入常見查詢模式：GROUP BY 策略、IN 寫法、分頁計數
> - **第六～八章** 進入 Recursive CTE 進階技法：Skip Scan 模擬、Top-N 加速、死循環防禦
>
> 每章均附有 **原始碼參考** 和 **Senior Dev 實戰補充**，適合中階以上 PostgreSQL 開發者。
>
> 主要來源：[digoal (德哥) PostgreSQL 部落格](https://github.com/digoal/blog)

---

# 一、查詢生命週期：從 Client Request 到 Result Return

> 來源：[PipelinedB Wiki - Lifecycle of a query](https://github.com/pipelinedb/pipelinedb/wiki/Lifecycle-of-a-query)
> 原出處：[digoal 轉載 (2015-10-16)](https://github.com/digoal/blog/blob/master/201510/20151016_01.md)

## 1. 六階段總覽

一個 query 從 client 發起請求到接收結果，經歷六個階段：

```
Client Request → Parser → Analyzer → Planner → Executor → Client Response
```

每個階段輸出不同的 tree 結構：
- **Parser** → parsed query tree（`parsenodes.h`）
- **Analyzer** → rewritten Query object
- **Planner** → plan tree（`plannodes.h`）
- **Executor** → executor node tree（`execnodes.h`），遞迴執行 plan tree 每個 node，結果返回 client

> 補充（Senior Dev）：這是 PG 最重要的三層 tree 轉換：
> | Tree Type | Header | 目的 |
> |-----------|--------|------|
> | Parse Tree | `parsenodes.h` | 語法結構 —— 「使用者說了什麼」 |
> | Plan Tree | `plannodes.h` | 執行策略 —— 「怎麼最省成本」 |
> | Executor State Tree | `execnodes.h` | 執行狀態 —— 「跑到哪裡了」 |
>
> Parse tree 的 node types（如 `SelectStmt`、`InsertStmt`）直接對應 SQL 語法結構（raw parse tree 不做任何語義校驗，連 column 存在與否都不檢查）。Plan tree 才是真正決定 I/O 策略的關鍵。了解這三層的區別，才能理解 `EXPLAIN` output 的 cost/row estimate 來源、以及為什麼有時 planner 的 estimate 與 actual 差異巨大（通常是 `pg_statistic` 過時或 correlation 不準）。

## 2. Client Request

PostgreSQL 使用 **process-per-connection** 模型（single-threaded），所有 concurrency 發生在 process level。

### I. Wire Protocol

Client 與 Server 之間使用 binary message protocol：message 的第一個 byte 是 type code（如 `Q` = simple query、`P` = Parse、`B` = Bind、`E` = Execute），接下來 4 bytes 是 message length，message body 按 type 對應的格式編碼。

Message types 定義在 [PostgreSQL Protocol Message Formats](http://www.postgresql.org/docs/9.4/static/protocol-message-formats.html)，client-side wrapper 位於 `libpq`（`src/include/libpq`）。

### II. Backend Process 分配

Client 獲取 backend process 有兩種路徑：

1. **Reuse idle process**：backend process 空閒時持續嘗試讀取新 incoming message，直接接管 client 請求
2. **Fork new process**：postmaster fork 新 backend process。Postmaster 本質上是 proxy server + child process supervisor

> 補充（Senior Dev）：Process-per-connection 是 PG 最經典的架構限制之一。每個 connection 至少消耗 ~5-10 MB private memory（取決於 `work_mem` / `temp_buffers`），所以 1000 connections = 5-10 GB。這也是為什麼 production 強烈建議使用 connection pooler（PgBouncer / Pgpool-II）。
>
> 關於 message protocol，有兩個模式：
> - **Simple Query Protocol**：`Q` message，一次性傳送整條 SQL → PG 內部走 `exec_simple_query()`
> - **Extended Query Protocol**：`P`(Parse) → `B`(Bind) → `E`(Execute) 三步，允許 parameterized query、cursor、batch execution。JDBC / Npgsql 預設使用 Extended Protocol
>
> 函數名 `exec_simple_query` **極具誤導性** —— 它處理絕大多數 query，包括極度複雜的查詢。

## 3. Parser

Raw SQL string 被 parser 解析為 **parse tree**（由 query node 構成的 tree）。

入口函數：`postgres.c:exec_simple_query`。

| 概念 | 說明 |
|------|------|
| Parse Tree Node Types | `parsenodes.h`：`SelectStmt`、`InsertStmt`、`DeleteStmt`、`UpdateStmt`、`CreateStmt` 等 |
| 輸出 | 純語法結構，未做語義檢查 |
| Debug | `SET debug_print_parse = on`，輸出寫入 server log |

> 補充（Senior Dev）：`debug_print_*` 系列參數輸出極其詳細（包含完整 tree dump），在 production 開啟會 flood log。建議只在 dev session 中 `SET`（僅影響當前 connection）而非全域 `ALTER SYSTEM`。另外 `debug_pretty_print = on` 可以讓 tree dump 變成 indented 格式，大幅提升可讀性。
>
> Parser 使用 `gram.y` + `scan.l`（flex/bison）生成。PG 的 SQL 語法是 LALR(1)，有一個著名的 corner case：`SELECT 1 AS foo UNION SELECT 2` vs `SELECT 1 AS foo, 2 AS bar` —— parser 需要 lookahead context 來區分 `UNION` 關鍵字和 alias `UNION`，這導致了某些 ambiguous grammar 的 shift/reduce conflict。

## 4. Analyzer (Semantic Analysis & Rewriting)

Parse tree 經過 semantic analysis 和 transformation 後被改寫為一個或多個等價的 Query object。

### I. View Rewriting

```sql
CREATE VIEW example_view AS SELECT * FROM table WHERE table.column = 'foo';
SELECT * FROM example_view;
-- Analyzer 改寫為：
SELECT * FROM table WHERE table.column = 'foo';
```

`analyze.c:transformStmt` 根據 query type 進行不同的分析和正規化處理，輸出統一的 Query object 給 Planner。

| 概念 | 說明 |
|------|------|
| View 改寫 | 非 materialized view → 展開為 base table 查詢 |
| 正規化 | 統一各種 query type 的表示形式 |
| 輸出 | Query object（`parsenodes.h:Query`） |
| Debug | `SET debug_print_rewritten = on` |

> 補充（Senior Dev）：Analyzer 不只處理 View，還處理：
> - **Column name resolution**：`*` → 展開為實際 column list
> - **Type inference & implicit cast**：`WHERE col = 42` 如果 col 是 text → 自動推斷需要 `42::text`
> - **Default expression resolution**：INSERT 漏掉的 column 補 default value
> - **Rule system**：`CREATE RULE` 定義的 rewrite rule（PG 內部用 rule 實現 View——`CREATE VIEW` 本質上就是建立一個 `_RETURN` rule）
>
> 重要區別：`EXPLAIN (ANALYZE, VERBOSE)` 中的 `Output` 欄位顯示的是 analyzer 處理後的 column list（已展開、已 cast），不是 raw SQL 的原始寫法。
>
> PG 的 rewrite system 允許 recursive rewrite。當一個 query tree 被 rewrite 後，新的 tree 會再次經過 rewrite 階段，直到沒有更多 rule 可以觸發。PG 以 `max_rewrite_depth` 做保護（超出即報錯）。

## 5. Planner (Query Optimization)

Analyzed Query object 傳入 Planner，生成 **plan tree**（node types 見 `plannodes.h`）。

```
Query → Planner → Plan Tree
```

### I. 三種掃描策略範例

```sql
CREATE TABLE planz (id integer, data text);
EXPLAIN ANALYZE SELECT * FROM planz WHERE id = 42;
```

Without index（Seq Scan）：
```
 Seq Scan on planz  (cost=0.00..34.00 rows=2400 width=4)
```

With index（Bitmap Heap Scan）：
```
 Bitmap Heap Scan on planz  (cost=4.20..13.67 rows=6 width=36)
   Recheck Cond: (id = 42)
   ->  Bitmap Index Scan on planz_id_index
         (cost=0.00..4.20 rows=6 width=0)
         Index Cond: (id = 42)
```

Planner 根據 cost model 在 `IndexScan` 與 `SeqScan` 之間選擇。

| 概念 | 說明 |
|------|------|
| 演算法 | cost-based（`geqo` 參數控制 table 數量超過閾值時切換為 genetic algorithm） |
| 輸出 | plan tree，每個 node 對應一種 execution strategy |
| Debug | `SET debug_print_plan = on` |

> 補充（Senior Dev）：Planner 的核心成本參數：
>
> | GUC | 預設 | 影響 |
> |-----|------|------|
> | `random_page_cost` | 4.0 | random disk read 成本（SSD 建議 1.1-1.5） |
> | `seq_page_cost` | 1.0 | sequential disk read 成本 |
> | `cpu_tuple_cost` | 0.01 | 處理每個 row 的 CPU 成本 |
> | `cpu_index_tuple_cost` | 0.005 | 處理每個 index entry 的成本 |
> | `effective_cache_size` | 4GB | planner 假設在 OS page cache 中的 page 數 |
>
> `effective_cache_size` 不分配任何 memory，只是讓 planner **假設這麼多 data 已在 cache**。值越大，planner 越傾向用 Index Scan。應設為 `shared_buffers + OS filesystem cache` 的總和（通常是總 RAM 的 50-75%）。
>
> `geqo` (Genetic Query Optimizer) 的預設觸發閾值是 `geqo_threshold = 12`。對於 OLAP / data warehouse，你可能需要調高讓 exhaustive search 跑久一點以獲取更好的 plan。
>
> `join_collapse_limit` 和 `from_collapse_limit` 控制 join order 的搜索範圍。預設值 `8` 對大多數場景足夠；10+ table join 可以逐步調高（注意 plan time 會呈指數增長）。

## 6. Executor

Plan tree 傳入 Executor 後，最終進入 `execMain.c:ExecutePlan`。

### I. Volcano Model（Pull-based）

Executor 採用 **pull-based iterator model**（Volcano model）。每個 plan node 都是一個 iterator，由 parent node 從 child node "pull" 資料：

```
ExecutePlan → 遞迴呼叫 execProcNode(node)
    → 每個 node 輸出 tuple
    → parent node 以此為 input
    → root node 輸出 = final query result
```

執行順序是 top-down（pull）+ bottom-up（return tuple）。

### II. Executor State Nodes

Executor 為每個 plan node 建立對應的 executor state node（`execnodes.h`）。

| 概念 | Source | 說明 |
|------|--------|------|
| Plan Node | `plannodes.h` | planner 的輸出，"what to do" |
| Executor Node | `execnodes.h` | executor 的 runtime state，"where we are" |
| Entry | `execProcnode.c:ExecProcNode` | switch-case dispatch 每個 node type 的分發邏輯 |

> 補充（Senior Dev）：Volcano model 的核心是每個 node 實現 `ExecInitNode`、`ExecProcNode`、`ExecEndNode` 三個接口。這使得 plan tree 可以任意嵌套組合（如 `Sort → HashAgg → SeqScan`），每個 node 只關心從 child pull 資料再做自己的處理。
>
> PG 9.6+ 引入 Parallel Query：`Gather` node 作為 plan tree 的 root，底下的 partial plan 被 `N` 個 parallel worker 同時執行。`ExecGather` 從每個 worker 的 `TupleQueue` pull tuple 合併輸出。
>
> PG 11+ 引入 JIT compilation（基於 LLVM）：ExecProcNode 的 tight loop（如 `WHERE` clause evaluation）在滿足 `jit_above_cost` 閾值時被 JIT compile 為 native code，對 CPU-bound query 的加速顯著（通常 10-30%）。
>
> 關於 Portal：
> - **Unnamed Portal**：`exec_simple_query` 建立的預設 portal，一次性執行全部 plan
> - **Named Portal**：`DECLARE CURSOR` 建立，可分段 fetch
> - Portal 本質是 executor state 的 holder，記錄目前執行到 plan tree 的哪個位置

## 7. Client Response

Result tuple 在 `ExecutePlan` 內部發送給 client。

流程：
1. `ExecProcNode` 產生每個 tuple
2. Tuple 傳給 destination receiver 的 callback function
3. 最常見的 receiver 是 regular client receiver
4. 用 `printtup.c:printtup` 將 tuple 格式化為 wire protocol message 發送

> 補充（Senior Dev）：Destination receiver 不只一種。除了 client receiver 外還有：
> - **SPI receiver**：PL/pgSQL function 內部用，tuple 存到 `SPITupleTable`
> - **Materialize receiver**：`Tuplestore`，用於 CTE (`WITH`)、cursor materialization
> - **COPY receiver**：`COPY TO` 輸出
>
> 理解 destination receiver 的抽象能幫你理解為什麼 `SELECT count(*)` 的 executor 不會把全部 row 送回 client：Aggregate node 攔截了所有 tuple 只輸出最終的 count=1 row，所以中間 tuple 根本沒經過 `printtup`。
>
> PG 14 引入了 pipeline mode（`libpq` 支援），client 可以在一條 connection 上連續發出多個 query 而不用等待前一個完成，大幅提升高延遲網路下的吞吐量。

## 8. 關鍵 Source File 索引

| Module | Source File(s) | 作用 |
|--------|---------------|------|
| Wire Protocol | `src/include/libpq/`、`libpq` | client-server message format |
| Backend Entry | `src/backend/tcop/postgres.c` | `exec_simple_query`、message dispatch |
| Postmaster | `src/backend/postmaster/postmaster.c` | process forking, supervision |
| Parser | `src/backend/parser/`、`gram.y`、`scan.l` | raw SQL → parse tree |
| Parse Nodes | `src/include/nodes/parsenodes.h` | `SelectStmt`, `InsertStmt`, `Query` |
| Analyzer/Rewrite | `src/backend/parser/analyze.c` | `transformStmt`, view rewriting |
| Planner | `src/backend/optimizer/plan/createplan.c` | cost model, strategy selection |
| Plan Nodes | `src/include/nodes/plannodes.h` | `SeqScan`, `IndexScan`, `BitmapHeapScan` |
| Executor Entry | `src/backend/executor/execMain.c` | `InitPlan`, `ExecutePlan` |
| Executor Dispatch | `src/backend/executor/execProcnode.c` | `ExecProcNode`, `ExecInitNode` |
| Executor Nodes | `src/include/nodes/execnodes.h` | runtime state for each plan node |
| Result Output | `src/backend/access/common/printtup.c` | `printtup`, wire protocol serialization |
| Portal | `src/backend/tcop/pquery.c` | `PortalRun`, cursor management |

---

# 二、CBO 與 pg_hint_plan：何時需要 Hint？

> 來源：[digoal - PostgreSQL SQL HINT的使用 (2016-02-03)](https://github.com/digoal/blog/blob/master/201602/20160203_01.md)

## 1. 背景：CBO 是否每次都能產生最優 Plan？

PostgreSQL 使用 Cost-Based Optimizer（CBO）。當關聯表數量超過 `geqo_threshold`（預設 12）時，會改用 Genetic Query Optimizer（GEQO）以避免窮舉帶來的 planning 開銷，但 GEQO 結果不保證最優。

### I. CBO 的成本計算因子

來源：`src/backend/optimizer/path/costsize.c`

| 因子類別 | 具體項目 | 影響 |
|---------|---------|------|
| 物理屬性 | table 的 data block 數量 | 影響 scan block 成本 |
| 物理屬性 | table 的 row 數量 | 影響 CPU 處理 row 成本 |
| 成本因子 | `seq_page_cost` / `random_page_cost` | 連續 vs 隨機讀取單個 block 的成本 |
| 成本因子 | `cpu_tuple_cost` / `cpu_index_tuple_cost` | CPU 處理 heap/index 中一條 row 的成本 |
| 成本因子 | `cpu_operator_cost` | 執行一個 operator 或 function 的成本 |
| Memory | `work_mem` 等 | 影響 Index Scan 的計算成本 |
| 統計資訊 | correlation | 影響 Index Scan 成本估算 |
| 統計資訊 | column width、null fraction、n_distinct、MCV、histogram buckets | 影響選擇性（selectivity） |
| 函數屬性 | 建立 function / operator 時設置的 cost | 影響含 function call 的查詢成本 |

### II. CBO 的盲區

**a. 已考慮但可能滯後的因素（auto-analyze 自動更新）：**

`pg_class.relpages` / `pg_class.reltuples`、`pg_statistic` 統計資訊——自動 analyze 有延遲，bulk INSERT/DELETE 後可能嚴重失準。

**b. 靜態配置因素（不會自動調整）：**

- OS cache 實際可用記憶體（`effective_cache_size` 是靜態設定）
- Function / operator cost（函數內部邏輯變更後不自動更新）

**c. 未考慮的因素：**

- Block device read-ahead（一次讀取預讀 128KB，但 CBO 按逐個 block 計算）
- `seq_page_cost` / `random_page_cost` 可針對 tablespace 設置，但不會動態追蹤 read-ahead 變更

**d. Generic plan cache（`choose_custom_plan`）：**

PostgreSQL 比對 cached plan 的平均成本與 custom plan 的成本：若 cached plan 成本更高，會改用 custom plan。前提是**統計資訊必須準確**。

### III. 何時需要 Hint

大多數情況下，合理配置 `random_page_cost`、定期 `ANALYZE`、調整 `default_statistics_target` 就足夠。需要 Hint 的場景：

- 靜態配置與實際不符（如 function cost 過時）
- 特定查詢的 optimizer 估計與實際偏差巨大
- 除錯：想對比不同 plan 的實際效率
- 緊急情況：production 中某個查詢突然選擇了差的 plan

> 補充（Senior Dev）：Hint 是把雙面刃。寫在 application code 中的 hint 會在資料成長後過時（plan rot），導致原本正確的 hint 變成效能殺手。應優先嘗試非侵入式方案（調整 `enable_*` GUC、extended statistics、調整 `random_page_cost`），hint 作為最後手段。

## 2. pg_hint_plan 簡介

`pg_hint_plan` 是 PostgreSQL extension，利用 PG 開放的 hook 介面實現 Oracle-style SQL hint。

### I. 安裝（PG 9.4 示範）

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

```sql
CREATE EXTENSION pg_hint_plan;
```

> PG 12+ 支援 **hint table**（`hint_plan.hints`），可在 table 中管理 hint 而無需修改 SQL text：
> ```sql
> INSERT INTO hint_plan.hints (norm_query_string, application_name, hints)
> VALUES ('SELECT * FROM orders WHERE create_time > $1', '', 'IndexScan(orders idx_orders_time)');
> ```

### II. Hint vs GUC Switch 對比

**原生 GUC 開關：**

```sql
SET enable_nestloop = off;
EXPLAIN SELECT a.*, b.* FROM a, b WHERE a.id = b.id AND a.id < 10;
-- 結果：Hash Join（禁用了 Nested Loop）
```

**pg_hint_plan hint：**

```sql
/*+
  HashJoin(a b)
  SeqScan(b)
*/
EXPLAIN SELECT a.*, b.* FROM a, b WHERE a.id = b.id AND a.id < 10;
```

Hint 相較 GUC 的優勢：GUC 是 session-level，hint 是 query-level，粒度更精細。

### III. 完整 Hint 列表

| Hint | 作用 |
|------|------|
| `SeqScan(table)` | 強制 Seq Scan |
| `TidScan(table)` | 強制 TID Scan |
| `IndexScan(table [index...])` | 強制 Index Scan |
| `IndexOnlyScan(table [index...])` | 強制 Index Only Scan |
| `BitmapScan(table [index...])` | 強制 Bitmap Scan |
| `NoSeqScan(table)` / `NoIndexScan(table)` 等 | 禁止某種 scan |
| `NestLoop(table ...)` / `HashJoin(...)` / `MergeJoin(...)` | 強制 join 方式 |
| `Leading(table table [table...])` | 強制 Join Order |
| `Rows(table ... correction)` | 修正 join result 的 row estimate |
| `Set(GUC-param value)` | 在 planner 執行期間設定 GUC 參數 |

> 實務中最常用的三個 hint：
> 1. **`Leading`** — 修復 join order 錯誤
> 2. **`Rows`** — 直接修正 row estimate
> 3. **`Set`** — 臨時覆蓋 `enable_*` 或 `work_mem`

## 3. CBO 成本計算的完整鏈路

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

### I. plan cache 的 choose_custom_plan 邏輯

來源：`src/backend/utils/cache/plancache.c`

- **custom plan**：每次重新規劃
- **generic plan**：使用緩存的通用 plan（首次 5 次執行使用 custom，第 6 次開始比較成本）

選擇邏輯：若 `cached_plan_cost > avg_custom_plan_cost`，改用 custom plan。

## 4. 替代方案與最佳實踐

> 在使用 pg_hint_plan 之前，應先嘗試以下方案，由輕到重：

| 優先級 | 方案 | 適用場景 |
|--------|------|---------|
| 1 | 調整 `random_page_cost` / `seq_page_cost` | SSD 環境 cost factor 不準確 |
| 2 | 增加 `default_statistics_target` | 單一 column 統計資訊不足 |
| 3 | CREATE STATISTICS（PG 10+ extended stats） | 多 column 之間有相關性 |
| 4 | `SET enable_nestloop = off` 等 GUC | 臨時性調整，不修改 code |
| 5 | 調整 `join_collapse_limit` / `from_collapse_limit` | 顯式 JOIN 順序被限制時 |
| 6 | pg_hint_plan hint table（PG 12+） | 需長期、可控的 plan 修正 |
| 7 | pg_hint_plan inline hint | 最後手段 |

---

# 三、GROUP BY 策略：Sort (GroupAgg) vs Hash (HashAgg)

> 來源：[digoal - PostgreSQL sort or not sort when group by? (2015-08-13)](https://github.com/digoal/blog/blob/master/201508/20150813_02.md)

## 1. 核心問題：GROUP BY 為什麼有時用 Sort？

Optimizer 選擇 Agg 策略的唯一標準是 cost。`GROUP BY` 有兩種 aggregation strategy：

| Strategy | 說明 |
|----------|------|
| **GroupAgg (AGG_SORTED)** | 先 Sort 再分組聚合。input 已有序時 skip Sort（如來自 index scan） |
| **HashAgg (AGG_HASHED)** | 建 hash table 聚合。需等全部 input 就緒才開始輸出 |

兩者 total CPU cost 在 cost 模型中相同，但 **GroupAgg startup cost 更低**（可 on-the-fly 輸出），HashAgg startup cost = input total cost（必須等所有 input 讀完）。

### I. 實驗：work_mem 影響策略選擇

```sql
CREATE TABLE t1 (c1 INT, c2 INT, c3 INT, c4 INT);
INSERT INTO t1 SELECT generate_series(1, 100000), 1, 1, 1;
```

**work_mem = 4MB → external sort + GroupAgg（392ms）：**

```
Sort Method: external sort  Disk: 2544kB
```

**work_mem = 1GB → HashAggregate（104ms）：**

```
HashAggregate  -- 無 Sort step，total execution 104ms
```

**強制關閉 HashAgg → quicksort + GroupAgg（68ms）：**

```
Sort Method: quicksort  Memory: 7760kB  -- 反而比 HashAgg 快
```

但在正常 GUC 設定下 optimizer 選 HashAgg，因為 cost 估算更低。

## 2. 原始碼分析：cost_agg() — 兩種策略的 Cost 計算

來源：`src/backend/optimizer/path/costsize.c`

**AGG_SORTED（GroupAgg）**：

```c
startup_cost = input_startup_cost;           // Sort 的 startup cost
total_cost   = input_total_cost;             // Sort 的 total cost
total_cost  += aggcosts->transCost.per_tuple * input_tuples;
total_cost  += (cpu_operator_cost * numGroupCols) * input_tuples;
total_cost  += aggcosts->finalCost * numGroups;
```

**AGG_HASHED（HashAgg）**：

```c
startup_cost = input_total_cost;             // 必須等所有 input
startup_cost += aggcosts->transCost.per_tuple * input_tuples;
total_cost   = startup_cost;
total_cost  += aggcosts->finalCost * numGroups;
```

**關鍵差異**：HashAgg 的 startup_cost = `input_total_cost`。GroupAgg 的 startup_cost = `input_startup_cost`（可越早輸出，對 LIMIT 查詢有利）。

原始碼註釋：
```
* AGG_SORTED and AGG_HASHED have exactly the same total CPU cost,
* but AGG_SORTED has lower startup cost. If the input path is
* already sorted appropriately, AGG_SORTED should be preferred.
```

> 當 input 已有序時（如來自 BTREE index scan 或 merge join），AGG_SORTED 可以跳過 Sort step，此時 GroupAgg 的 total cost 會顯著低於 HashAgg。

## 3. cost_sort() — Sort 策略的成本分岔

來源：`src/backend/optimizer/path/costsize.c`

```
1. output_bytes > sort_mem_bytes  →  external disk sort (polyphase merge)
2. tuples > 2 × output_tuples  →  bounded heap sort (LIMIT 場景)
3. 其他 →  quicksort (全部 in-memory)
```

## 4. GroupAgg vs HashAgg 選擇的實戰考量

| 維度 | GroupAgg | HashAgg |
|------|----------|---------|
| requires sort | 是（除非 input 已有序） | 否 |
| startup cost | 低 | 高（=input total cost） |
| memory risk | 取決於 sort 階段 memory | 取決於 #distinct groups |
| LIMIT 友好 | 是（early output） | 否（需建完整 hash table） |
| 大量 distinct groups | 不適合 | 適合（若 hash table fits memory） |
| input 已有序 | **最佳**（skip sort） | 仍需建 hash table |

> 當 `work_mem` 不足造成 HashAgg 的 hash table spill 到 disk 時，成本會急劇上升。可透過 `SET enable_hashagg = off` 強制切換 GroupAgg 對比真實 performance。

## 5. 相關參數與版本演進

| 參數 | 預設 | 說明 |
|------|------|------|
| `enable_hashagg` | on | 允許 optimizer 選擇 HashAgg |
| `work_mem` | 4MB | sort 與 hash table 的 memory 上限 |
| `hash_mem_multiplier` | 2.0 (PG 16+) | Hash 操作的 memory = `work_mem × hash_mem_multiplier` |

| 功能 | 版本 |
|------|------|
| parallel hash join / hashagg | PG 11+ |
| disk-based HashAgg | PG 13+ |
| plan-time hash sizing | PG 14+ |
| hash_mem_multiplier | PG 16 |

---

# 四、IN / =ANY(ARRAY) / =ANY(VALUES) / JOIN VALUES 效能對決

> 來源：[digoal - PostgreSQL IN/VALUES 優化 (2014-10-16)](https://github.com/digoal/blog/blob/master/201410/20141016_01.md)
> 案例：[Datadog: 100x faster Postgres performance by changing 1 line](https://www.datadoghq.com/2013/08/100x-faster-postgres-performance-by-changing-1-line/)

## 1. 四種寫法與 Execution Plan

```sql
-- 1) IN 列表
SELECT * FROM t2 WHERE id IN (1,2,3,100,1000,0);

-- 2) =ANY(ARRAY) → optimizer 內部改寫為 ScalarArrayOpExpr
SELECT * FROM t2 WHERE id = ANY ('{1,2,3,100,1000,0}'::integer[]);

-- 3) =ANY(VALUES) → 產生 HashAggregate + Nested Loop
SELECT * FROM t2 WHERE id = ANY (VALUES (1),(2),(3),(100),(1000),(0));

-- 4) JOIN VALUES → Nested Loop（不產生 HashAggregate）
SELECT t2.* FROM t2 JOIN (VALUES (1),(2),(3),(100),(1000),(0)) AS t(id) ON t2.id = t.id;
```

## 2. Datadog 實戰案例：22s → 263ms（100x 提速）

### I. 原始問題

15 百萬 row 的表，查詢 11,000 個 primary key 時花了 22 秒。執行計劃核心節點是 **Bitmap Heap Scan + Filter**，Filter 逐行檢查 11,000 個 key 的 membership。

### II. 根因

大陣列 `=ANY(ARRAY[...])` 讓 optimizer 選擇 **Bitmap Heap Scan + Filter** 而非 **Index Scan**。Bitmap 將所有符合條件的 page 讀出，然後在 Filter 階段逐行比對 key ∈ array → 產生巨大的 Recheck overhead。

### III. 解法（一行差異）

```sql
-- Before: 22s
WHERE c.key = ANY (ARRAY[15368196, ...])
-- After: 263ms (100x faster)
WHERE c.key = ANY (VALUES (15368196), ...)
```

新計劃使用 **Nested Loop Index Scan**（一次性逐 key 精確查表）。

## 3. PG 14 實測：work_mem 對 VALUES 策略的影響

`=ANY(VALUES(...))` 的效能受 **work_mem** 影響：低 work_mem 時 VALUES 去重用 external sort → disk I/O，高 work_mem 時用 in-memory HashAggregate。但即使 external sort，仍遠快於 Bitmap Heap Scan + Filter 方案。

> PG 14 起引入 `MIN_ARRAY_SIZE_FOR_HASHED_SAOP`（預設 9），當 `IN` 列表超過 9 個元素時，optimizer 自動將 `INDEX = ANY(ARRAY[...])` 從 sequential scan 切換為 hash table probe，大幅改善中等大小 IN 列表的效能。

## 4. 效能矩陣：何時用哪種寫法

| 寫法 | 適合場景 | 注意 |
|------|---------|------|
| `IN (1,2,3)` | ≤ 9 個元素（PG 14+ 自動 hash） | 不適合動態拼接 |
| `=ANY(ARRAY[...])` | ≤ 數百個元素 | 大陣列可能選 Bitmap Scan |
| `=ANY(VALUES(...))` | 數百~數萬個元素 | 大列表穩定性極佳 |
| `JOIN (VALUES(...))` | 需保留排序 | 不保證唯一回傳 |

### I. 選擇決策

```
元素數 ≤ 9     → IN (...)
元素數 10~500  → =ANY(ARRAY[...])
元素數 500~50K → =ANY(VALUES(...))
元素數 > 50K   → TEMPORARY TABLE + JOIN + ANALYZE
```

## 5. Multi-Column IN 的寫法

```sql
-- 多列 IN：只有 =ANY(VALUES) 可用
WHERE (col_a, col_b) = ANY(VALUES (1,'x'), (2,'y'), (3,'z'))
```

## 6. 版本演進

| 功能 | 版本 |
|------|------|
| `=ANY(VALUES)` 基本支援 | PG 8.4+ |
| `plan_cache_mode = force_custom_plan` | PG 12 |
| `MIN_ARRAY_SIZE_FOR_HASHED_SAOP` | PG 14 |
| `enable_presorted_aggs` | PG 17 |

---

# 五、分頁與計數優化：count(*) 替代 + OFFSET 退化 + Keyset 位點

> 來源：[digoal - 论 count 与 offset 使用不当的罪名 (2016-05-06)](https://github.com/digoal/blog/blob/master/201605/20160506_01.md)

## 1. count(\*) 的替代：EXPLAIN Row Estimate

### I. 問題

典型分頁流程：先 `count(*)` 算總數 → 算頁數 → 再 `SELECT ... LIMIT N OFFSET M` 取數據。這等於同一批數據掃了兩遍。

### II. count_estimate() 函數

從 `EXPLAIN` 輸出中擷取 planner 的 row estimate，完全避免實際掃描數據：

```sql
CREATE FUNCTION count_estimate(query text) RETURNS INTEGER AS
$func$
DECLARE
    rec   record;
    ROWS  INTEGER;
BEGIN
    FOR rec IN EXECUTE 'EXPLAIN ' || query LOOP
        ROWS := SUBSTRING(rec."QUERY PLAN" FROM ' rows=([[:digit:]]+)');
        EXIT WHEN ROWS IS NOT NULL;
    END LOOP;
    RETURN ROWS;
END
$func$ LANGUAGE plpgsql;
```

### III. 精度對比

```sql
SELECT count_estimate('SELECT * FROM sbtest1 WHERE id BETWEEN 100 AND 100000');
-- → 102,166

SELECT count(*) FROM sbtest1 WHERE id BETWEEN 100 AND 100000;
-- → 99,901  （誤差 ≈ 2.3%）
```

### IV. 精度依賴

> **何時 count_estimate 不可靠？**
> - 最後一次 `ANALYZE` 後有大規模 INSERT/DELETE
> - 查詢包含複雜的 multi-column filter
> - `WHERE` 中有 function call（planner 使用 default selectivity = 0.005）
> - 剛 `TRUNCATE` + 重新 `INSERT` 但尚未 ANALYZE 的表
>
> **更輕量的近似 count 方案**：
> ```sql
> SELECT reltuples::bigint FROM pg_class WHERE relname = 'your_table';  -- 零成本
> SELECT n_live_tup FROM pg_stat_user_tables WHERE relname = 'your_table';
> ```
>
> **實務建議**：前端頁碼導航的總頁數不需要精確到個位數——用 `reltuples` 或 `count_estimate()`，設定 TTL 緩存（Redis 30-60s）。只有需要精確合規報表才用真實 `count(*)`。

## 2. OFFSET 效能退化與 CURSOR 對比

### I. 問題重現

```sql
-- OFFSET 0：極快 (0.088ms)
-- OFFSET 900000：退化 3700x (462ms)
```

Index Scan 掃了 900,100 row 才能跳過 900,000 → 丟棄 → 再取最後 100 row。

### II. CURSOR 方案

```sql
BEGIN;
DECLARE cur1 CURSOR FOR
  SELECT * FROM sbtest1 WHERE id BETWEEN 100 AND 1000000 ORDER BY id;
FETCH 100 FROM cur1;
```

CURSOR 的一次性 MOVE 成本等同 OFFSET，但後續 FETCH 成本極低。但 CURSOR 綁定在 backend connection 上，HTTP 請求之間無法保持，web 場景基本不可行。

### III. 三種分頁方案對比

| 方案 | 深頁效能 | Memory | 長 Transaction | 任意跳頁 |
|------|---------|--------|---------------|---------|
| OFFSET/LIMIT | 線性退化 | 低 | 無 | 支援 |
| CURSOR | MOVE 一次成本，後續 O(1) | 低 | **嚴重** | 方向可控 |
| Keyset Pagination | O(1) seek | 低 | 無 | **不支援** |

## 3. 自建位點法：當 ORDER BY column 沒有 PK/UK 時

### I. 問題

當排序欄位不是 PK 或 UK 時，同一 `crt_time` 可能對應多條 row，keyset 位點無法唯一指定。

### II. 解法：強制加入自增 PK 作為 tie-breaker

```sql
CREATE TABLE test1 (
    pk SERIAL8 PRIMARY KEY,
    id INT, c1 INT, c2 INT, c3 INT, c4 INT,
    crt_time TIMESTAMP
);
CREATE INDEX idx_test1_1 ON test1(c1, crt_time, pk);

-- keyset pagination
SELECT * FROM test1
WHERE c1 = 1 AND c2 BETWEEN 1 AND 10
  AND crt_time >= '2018-07-25 18:31:14.860328'
  AND pk > 1048412
ORDER BY crt_time, pk LIMIT 10;
```

### III. 效能對比

| 指標 | OFFSET | Keyset | 改善 |
|------|--------|--------|------|
| Execution time | 53.4 ms | **0.075 ms** | **712x** |
| shared_buffers hit | 8,506 | **9** | 945x |
| Row scanned | 100,010 | **~10** | 10,000x |

### IV. 設計要點

1. `ORDER BY` 必須包含 tie-breaker（PK）以確保順序唯一
2. Index key 順序必須是 `(filter_column, order_column, pk)`
3. PK 必須是 append-only（不可 UPDATE），否則位點會漂移

> **自建 PK 的生產考量**：
> - PG 10+ 建議用 `GENERATED ALWAYS AS IDENTITY` 替代 `SERIAL`
> - 大表遷移：`ALTER TABLE ADD COLUMN pk BIGSERIAL PRIMARY KEY` 會鎖全表，需用 `pg_repack` 或建立新表 + 批次遷移
> - 如果原始業務表已有 business key（如 `order_id + line_item`），優先使用 business key 作為 tie-breaker

---

# 六、Recursive CTE 優化技法：Index Skip Scan 模擬 + Top-N Per Group

> 來源：[digoal - 递归优化CASE (2012-09-14)](https://github.com/digoal/blog/blob/master/201209/20120914_01.md)
> [digoal - distinct xx和count(distinct xx)的变态递归优化方法 (2016-11-28)](https://github.com/digoal/blog/blob/master/201611/20161128_02.md)

## 1. 問題本質：全表 GROUP BY / DISTINCT 的掃描浪費

### I. 經典問題 SQL

```sql
SELECT a.skyid, a.giftpackageid, a.giftpackagename, a.intimestamp
FROM tbl_anc_player_win_log a
GROUP BY a.skyid, a.giftpackageid, a.giftpackagename, a.intimestamp
ORDER BY a.intimestamp DESC LIMIT 10;
```

這個查詢看似取 10 row，但 `GROUP BY` 會先對全表做排序/聚合，然後才取 Top 10。

```sql
SELECT COUNT(DISTINCT sex) FROM sex;  -- sex 只有 'm'/'w' 兩個值
```

在 2000 萬 row 的表上，即使建立 index，仍需掃完整個 index 才能做完 GROUP BY / DISTINCT —— PostgreSQL Index Only Scan 不支援「看到 unique value 就停」的 lazy aggregation。

> B-tree index 的本質限制：B-tree 是 **range-scan oriented**，不支援直接跳到「下一個不同值」。Oracle 的 **Index Skip Scan** 能做這件事，PostgreSQL 直到 PG 17（2024）才加入了部分支援。本文介紹的 recursive CTE 技巧本質上是用 **application-level loop** 模擬了 Skip Scan。

## 2. 技法 1：Subquery 縮小 GROUP BY 範圍

```sql
SELECT * FROM (
    SELECT user_id, listid, apkid, get_time
    FROM user_download_log
    ORDER BY get_time DESC LIMIT 1000
) AS t
GROUP BY user_id, listid, apkid, get_time
ORDER BY get_time DESC LIMIT 10;
```

| 方法 | Plan | Runtime | 提速 |
|------|------|--------|:---:|
| 原始 | Seq Scan + Sort + Group | 97.7ms | 1x |
| Subquery 縮小 | Index Scan Backward + HashAgg | 2.0ms | **49x** |

弊端：內層 `LIMIT` 的值不好估算。

## 3. 技法 2：Cursor + Array（PL/pgSQL）

用 cursor 逐行 fetch，用 array 做 in-memory dedup，取滿 LIMIT 後停止。

| 方法 | Runtime | 提速 |
|------|--------|:---:|
| 原始 | 97.7ms | 1x |
| Cursor + array | 0.81ms | **121x** |

> `ANY(array)` 對每條 fetch 做 O(n) 掃描。如果 distinct 組合極多（如數千），可改用 `hstore` 或 `jsonb` key 做 hash-based dedup。

## 4. 技法 3：Trigger 維護物化表

Trigger 在每次 INSERT 時檢查並維護一張去重物化表。

```sql
-- 查詢直接命中物化表
SELECT * FROM user_download_uk ORDER BY get_time DESC LIMIT 10;
-- 0.36ms（271x faster）
```

| 方法 | Runtime | 提速 | 代價 |
|------|--------|:---:|------|
| Trigger 物化 | 0.36ms | **271x** | Insert 從 ~4s 升到 ~4.9s（+23%） |

## 5. 技法 4：Recursive CTE — 核心 Index Skip Scan 模擬

### I. 原理

```
1. Non-recursive term：用 index scan 取 min(column)
2. Recursive term：用 index scan 取 「> 上一個值」的 min(column)
3. 每次遞迴只掃一個 index leaf → O(distinct_values) × O(log N)
4. Stop condition：WHERE column IS NOT NULL 阻斷 NULL
```

### II. Sparse Column 場景（低基數，如 sex / status）

```sql
WITH RECURSIVE skip AS (
    (SELECT MIN(t.sex) AS sex FROM sex t WHERE t.sex IS NOT NULL)
    UNION ALL
    (SELECT (SELECT MIN(t.sex) FROM sex t
             WHERE t.sex > s.sex AND t.sex IS NOT NULL)
     FROM skip s WHERE s.sex IS NOT NULL)
)
SELECT COUNT(DISTINCT sex) FROM skip;
-- 0.173 ms（原始 47254ms 的 1/273,000，36,390x speedup）
```

**關鍵：每次遞迴只掃 1 個 index leaf**。

### III. Dense Column 場景（高基數，不適合）

| Scenario | Distinct Values | Original | Recursive CTE | 適合？ |
|----------|:---:|---------|--------------|:---:|
| sex | 2 | 47,254ms | **0.17ms** | ✓ |
| user_id | 10,000,001 | 4,009ms | **186,909ms** | ✗ |

Recursive CTE 的每一層 overhead 約 0.01-0.02ms。因此 distinct count > ~50,000 時成本就會超過全表掃描。

> 快速評估：`SELECT COUNT(DISTINCT col) FROM pg_stats WHERE tablename = 'your_table' AND attname = 'your_col'`。若 distinct_count < 500 → recursive CTE 極有效；> 100,000 → 不適用。

## 6. 技法 5：Recursive CTE 做 Top-N Per Group

將問題拆成兩步：遞迴枚舉所有 distinct group → 對每個 group 取 Top N。

```sql
WITH RECURSIVE skip AS (
    (SELECT t.username FROM test t ORDER BY t.username LIMIT 1)
    UNION ALL
    (SELECT (SELECT MIN(t2.username) FROM test t2
             WHERE t2.username > s.username)
     FROM skip s WHERE s.username IS NOT NULL)
),
with_data AS (
    SELECT ARRAY(
        SELECT t FROM test t
        WHERE t.username = s.username
        ORDER BY t.some_ts DESC LIMIT 5
    ) AS rows
    FROM skip s WHERE s.username IS NOT NULL
)
SELECT (unnest(rows)).* FROM with_data;
-- 1.924 ms（原始 11,329ms，5,900x）
```

> LATERAL subquery 的等價寫法（PG 13+ 推薦，可讀性更高）：
> ```sql
> SELECT u.username, t.*
> FROM (SELECT DISTINCT username FROM test) u
> JOIN LATERAL (
>     SELECT * FROM test t WHERE t.username = u.username
>     ORDER BY t.some_ts DESC LIMIT 5
> ) t ON true;
> ```

## 7. Recursive CTE 的限制與注意事項

### I. Aggregate Function 不能在 Recursive Term 直接使用

必須將 aggregate 包在 scalar subquery 內。

### II. 中止條件必須明確

每個 recursive CTE 都要有終止保護：`SET max_recursion_depth = 500;`

## 8. 技法選擇決策

```
需要 GROUP BY + ORDER BY + LIMIT？
├─ Order by column 有 index + 數據均勻？
│   → 技法 1：Subquery LIMIT 縮小範圍（49x）
├─ 複雜多列 unique 組合 + 接受 PL/pgSQL？
│   → 技法 2：Cursor + Array（121x）
├─ DQL >> DML + 可接受 DML overhead？
│   → 技法 3：Trigger 物化表（271x）
├─ 單列 sparse column（distinct < 500）+ COUNT/GROUP？
│   → 技法 4：Recursive CTE Skip Scan（36,390x）
├─ Top-N per group（distinct group 少）？
│   → 技法 5：Recursive CTE + ARRAY unnest（5,900x）
│   或 LATERAL subquery（PG 13+ 推薦）
└─ Dense column（distinct > 100K）？
    → 接受全表 scan，或考慮 BRIN / 分區 / covering index
```

---

# 七、分組 Top-N：Recursive CTE + Index Loop 替代 Window Function（44x 加速）

> 來源：[digoal - CTE 递归查询分組TOP性能提升44倍 (2016-08-15)](https://github.com/digoal/blog/blob/master/201608/20160815_04.md)

## 1. 問題：Window Function 全表掃描

業務需求：每個 group 取 TOP N（每位歌手下載量 TOP 10、每個城市納稅前 10 名）。

### I. 傳統寫法

```sql
SELECT * FROM (
  SELECT row_number() OVER (PARTITION BY c1 ORDER BY c2) AS rn, *
  FROM tbl
) t
WHERE t.rn <= 10;
```

### II. 實驗：10,000 groups / 10M rows

```sql
CREATE TABLE tbl (c1 INT, c2 INT, c3 INT);
CREATE INDEX idx1 ON tbl (c1, c2);
INSERT INTO tbl SELECT mod(trunc(random()*10000)::int,10000),
       trunc(random()*10000000) FROM generate_series(1,10000000);
```

**WindowAgg 掃描了全部 10M row**，淘汰 99%（9.9M row 被 `Rows Removed by Filter` 丟棄）。耗時 **20.8 秒**。

> WindowAgg 是 blocking operator——必須把整個 partition 的 row 讀完才能確定 `row_number()` 排序。即使 index 提供排序，`PARTITION BY c1` 的 boundary 跨越讓 planner 無法做 early termination。

## 2. 優化方案：Recursive CTE + Per-Group Index Seek

### I. 核心思路

1. 用 recursive CTE 取出所有 distinct `c1` 值（index skip scan）
2. 對每個 `c1` 執行 `SELECT * FROM tbl WHERE c1 = v ORDER BY c2 LIMIT 10`
3. 每個 group 的查詢只需要讀取 index 中的 10 條 entry（non-blocking）

### II. PL/pgSQL Function 封裝

```sql
CREATE OR REPLACE FUNCTION f() RETURNS SETOF tbl AS $$
DECLARE v INT;
BEGIN
  FOR v IN
    WITH RECURSIVE t1 AS (
      (SELECT min(c1) c1 FROM tbl)
      UNION ALL
      (SELECT (SELECT min(tbl.c1) FROM tbl WHERE tbl.c1 > t.c1)
       FROM t1 t WHERE t.c1 IS NOT NULL)
    )
    SELECT * FROM t1
  LOOP
    RETURN QUERY SELECT * FROM tbl WHERE c1 = v ORDER BY c2 LIMIT 10;
  END LOOP;
END;
$$ LANGUAGE plpgsql STRICT;
```

### III. 效能結果

| 指標 | Window Function | Recursive CTE + Function |
|------|----------------|--------------------------|
| Execution time | 20,834 ms | **464 ms（44x）** |
| shared_buffers hit | 10,035,535 | 170,407 (98% 減少) |

> **此方案成立的前提條件**：
> 1. **Group 數量 << Total row 數**
> 2. **Index 必須是 `(group_col, order_col)`**
> 3. **TOP N 遠小於 group 內 row 數**

## 3. 現代替代方案（PG 9.3+ / PG 11+ / PG 14+）

### I. LATERAL + DISTINCT（PG 9.3+）

```sql
SELECT t.*
FROM (SELECT DISTINCT c1 FROM tbl) groups,
LATERAL (SELECT * FROM tbl WHERE tbl.c1 = groups.c1 ORDER BY c2 LIMIT 10) t;
```

### II. CTE MATERIALIZED 控制（PG 12+）

```sql
WITH groups AS MATERIALIZED (
  SELECT DISTINCT c1 FROM tbl
)
SELECT t.* FROM groups,
LATERAL (SELECT * FROM tbl WHERE tbl.c1 = groups.c1 ORDER BY c2 LIMIT 10) t;
```

### III. 方案對比總結

| 方案 | PG 版本 | 優點 | 劣勢 |
|------|---------|------|------|
| Window Function | 8.4+ | 寫法簡單 | 全表掃描 O(G×R) |
| Recursive CTE + Function | 9.0+ | N << R 時極佳 | 需 function 封裝 |
| LATERAL + DISTINCT | 9.3+ | 純 SQL | Planner 可能選 HashAgg |
| CTE MATERIALIZED + LATERAL | 12+ | Planner 行為可控 | 暫存 distinct group 結果 |

> **實際生產選擇**：9.3+ 用 `LATERAL + DISTINCT`，12+ 加 `MATERIALIZED`。Index 設計的黃金組合：`(c1, c2)`——`c1` 用於 group filter，`c2` 用於 order + limit。

---

# 八、Recursive CTE 死循環防禦：CYCLE 語法與 Production 防禦體系

> 來源：[digoal - PostgreSQL 递归死循环案例及解法 (2016-07-23)](https://github.com/digoal/blog/blob/master/201607/20160723_01.md)

## 1. 問題：Recursive CTE 遇到數據迴圈（Cycle）

WITH RECURSIVE 是處理樹形 / 圖形查詢的標準語法，但如果數據中存在循環引用（例如 `c1 = 1, c2 = 1`，自己指向自己），recursive CTE 會無限迭代，不斷產生中間結果、寫入 temp file，最終耗盡磁盤空間。

### I. 迴圈數據範例

```sql
CREATE TABLE test(c1 int, c2 int, info text);
INSERT INTO test VALUES
    (9,8,'test'), (8,7,'test'), (7,6,'test'),
    (6,5,'test'), (5,4,'test'), (4,3,'test'),
    (3,2,'test'), (2,1,'test'), (1,1,'test');  -- ← loop
```

### II. 死循環觸發

```sql
WITH RECURSIVE t(c1, c2, info) AS (
    SELECT * FROM test WHERE c1 = 9
    UNION ALL
    SELECT t2.* FROM test t2 JOIN t ON (t.c2 = t2.c1)
)
SELECT count(*) FROM t;
-- 永遠不會結束：9→8→7→6→5→4→3→2→1→1→1→1→...
```

### III. Temp File 暴增過程

```
pgsql_tmp8997.5  89M → 575M → 1.0G → 2.4G → ...
```

`log_temp_files` 只在 query 結束時才記錄 log，中途不會有警告。

## 2. 解法 1：`temp_file_limit` 硬限制

```sql
SET temp_file_limit = '10MB';

WITH RECURSIVE t(...) ...
-- ERROR: 53400: temporary file size exceeds temp_file_limit (10240kB)
```

### I. pg_hint_plan 逐 Query 指定限制

```sql
/*+ Set (temp_file_limit '10MB') */
WITH RECURSIVE t(...) SELECT count(*) FROM t;
```

## 3. 解法 2：`CYCLE` Clause — PG 14+ 根本方案

PG 14 引入了 SQL 標準的 `CYCLE` clause，從遞迴引擎內部檢測 cycle：

```sql
WITH RECURSIVE t(c1, c2, info) AS (
    SELECT * FROM test WHERE c1 = 9
    UNION ALL
    SELECT t2.* FROM test t2 JOIN t ON (t.c2 = t2.c1)
)
CYCLE c1 SET is_cycle USING path
SELECT * FROM t;
```

| c1 | c2 | info | is_cycle | path |
|----|----|------|----------|------|
| 9 | 8 | test | f | {(9)} |
| 8 | 7 | test | f | {(9),(8)} |
| ... | ... | ... | ... | ... |
| 1 | 1 | test | **t** | {(9),(8),...,(1),(1)} |

> `CYCLE` clause 內部實作維護一個 hash table 追蹤已訪問節點。相比 `temp_file_limit` 的「暴力中斷」，CYCLE 是「優雅終止」——不需要猜測 limit 值，不會誤殺正常的大遞迴查詢。

## 4. Production 防禦體系

### I. 三層防禦

| 層級 | 機制 | 適用場景 |
|------|------|---------|
| **SQL 層**（根本） | `CYCLE` clause (PG 14+) | 所有 recursive CTE 都應加 |
| **SQL 層**（相容） | 手動 breadcrumb `NOT (c1 = ANY(path))` | PG < 14 |
| **Session 層**（兜底） | `SET LOCAL temp_file_limit = '1GB'` | 防止未知 CTE 炸庫 |
| **DB 層**（全域） | `temp_file_limit` in postgresql.conf | 預防性全域上限 |

### II. PG < 14 的手動 Cycle Detection

```sql
WITH RECURSIVE t(c1, c2, info, path, depth) AS (
    SELECT *, ARRAY[c1], 0 FROM test WHERE c1 = 9
    UNION ALL
    SELECT t2.*, t.path || t2.c1, t.depth + 1
    FROM test t2
    JOIN t ON (t.c2 = t2.c1 AND NOT (t2.c1 = ANY(t.path)))
)
SELECT * FROM t;
```

> breadcrumb mode 的缺點：(a) 每次遞迴都要 scan 整個 `path` array（O(n)）；(b) `path` array 隨遞迴增長。PG 14 的 `CYCLE` clause 內部用 hash table，O(1) 查重。

### III. `SET LOCAL` vs `SET SESSION`

```sql
SET LOCAL temp_file_limit = '1GB';     -- 僅影響當前 transaction
SET SESSION temp_file_limit = '1GB';   -- 影響整個 session
```

建議在 application 層以 `SET LOCAL` 包裹 recursive CTE。

### IV. Temp File 清理機制

- Query 結束時自動清理（無論正常或異常終止）
- DB startup 時，startup process 會清理殘留 temp file
- `pg_cancel_backend()` 會觸發清理；`pg_terminate_backend()` 可能留下孤兒 temp file

### V. Temp File 監控（PG 17）

```sql
SELECT pg_stat_get_temp_files();
SELECT temp_files, temp_bytes FROM pg_stat_database WHERE datname = current_database();
```

### VI. PG 14-17 Recursive CTE 相關改進

| 版本 | 改進 |
|------|------|
| PG 14 | `CYCLE` clause、`SEARCH` clause（BFS/DFS 排序） |
| PG 15 | Recursive CTE materialization 策略優化 |
| PG 16 | Recursive CTE hash table 可 spill to disk |
| PG 17 | `WORK_MEM` 對 recursive CTE hash table 的精細分配 |

### VII. `SEARCH` Clause 搭配 `CYCLE`（PG 14+）

```sql
WITH RECURSIVE t(c1, c2, info) AS (
    SELECT * FROM test WHERE c1 = 9
    UNION ALL
    SELECT t2.* FROM test t2 JOIN t ON (t.c2 = t2.c1)
)
SEARCH DEPTH FIRST BY c1 SET order_col
CYCLE c1 SET is_cycle USING path
SELECT * FROM t ORDER BY order_col;
```

---

# 附錄：統合參考文獻

1. [PipelinedB Wiki - Lifecycle of a query](https://github.com/pipelinedb/pipelinedb/wiki/Lifecycle-of-a-query)
2. [pg_hint_plan 官方文件](http://pghintplan.sourceforge.jp/pg_hint_plan-en.html)
3. [Datadog: 100x faster Postgres performance by changing 1 line](https://www.datadoghq.com/2013/08/100x-faster-postgres-performance-by-changing-1-line/)
4. [digoal - 递归优化CASE](https://github.com/digoal/blog/blob/master/201209/20120914_01.md)
5. [digoal - distinct xx和count(distinct xx)的变态递归优化方法](https://github.com/digoal/blog/blob/master/201611/20161128_02.md)
6. [digoal - CTE 递归查询分組TOP性能提升44倍](https://github.com/digoal/blog/blob/master/201608/20160815_04.md)
7. [digoal - PostgreSQL 递归死循环案例及解法](https://github.com/digoal/blog/blob/master/201607/20160723_01.md)
8. [digoal - 论 count 与 offset 使用不当的罪名](https://github.com/digoal/blog/blob/master/201605/20160506_01.md)
9. [digoal - PostgreSQL sort or not sort when group by?](https://github.com/digoal/blog/blob/master/201508/20150813_02.md)
10. [digoal - PostgreSQL IN/VALUES 優化](https://github.com/digoal/blog/blob/master/201410/20141016_01.md)
11. [PostgreSQL Protocol Message Formats](http://www.postgresql.org/docs/9.4/static/protocol-message-formats.html)
