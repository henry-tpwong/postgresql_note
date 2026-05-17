# PostgreSQL Query 生命週期分析 — 從 Client Request 到 Result Return

> 來源：[digoal 轉載自 PipelinedB Wiki - PostgreSQL query 生命周期分析 (2015-10-16)](https://github.com/digoal/blog/blob/master/201510/20151016_01.md)
>
> 原出處：[PipelinedB Wiki - Lifecycle of a query](https://github.com/pipelinedb/pipelinedb/wiki/Lifecycle-of-a-query)

---

## 六階段總覽

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

---

## 1. Client Request

PostgreSQL 使用 **process-per-connection** 模型（single-threaded），所有 concurrency 發生在 process level。

### Wire Protocol

Client 與 Server 之間使用 binary message protocol：message 的第一個 byte 是 type code（如 `Q` = simple query、`P` = Parse、`B` = Bind、`E` = Execute），接下來 4 bytes 是 message length，message body 按 type 對應的格式編碼。

Message types 定義在 [PostgreSQL Protocol Message Formats](http://www.postgresql.org/docs/9.4/static/protocol-message-formats.html)，client-side wrapper 位於 `libpq`（`src/include/libpq`）。

### Backend Process 分配

Client 獲取 backend process 有兩種路徑：

1. **Reuse idle process**：backend process 空閒時 [持續嘗試讀取新 incoming message](https://github.com/pipelinedb/pipelinedb/blob/master/src/backend/tcop/postgres.c#L4166)，直接接管 client 請求
2. **Fork new process**：postmaster [fork 新 backend process](https://github.com/pipelinedb/pipelinedb/blob/master/src/backend/postmaster/postmaster.c#L1598)。Postmaster 本質上是 proxy server + child process supervisor

> 補充（Senior Dev）：Process-per-connection 是 PG 最經典的架構限制之一。每個 connection 至少消耗 ~5-10 MB private memory（取決於 `work_mem` / `temp_buffers`），所以 1000 connections = 5-10 GB。這也是為什麼 production 強烈建議使用 connection pooler（PgBouncer / Pgpool-II）。
>
> 關於 message protocol，有兩個模式：
> - **Simple Query Protocol**：`Q` message，一次性傳送整條 SQL → PG 內部走 `exec_simple_query()`
> - **Extended Query Protocol**：`P`(Parse) → `B`(Bind) → `E`(Execute) 三步，允許 parameterized query、cursor、batch execution。JDBC / Npgsql 預設使用 Extended Protocol
>
> 函數名 `exec_simple_query` **極具誤導性** —— 它處理絕大多數 query，包括極度複雜的查詢。`exec_standard_query` 或 `exec_query` 才是更好的命名。只有 cursor open / fetch、`BIND` message 等少數類型由各自專用函數處理。

---

## 2. Parser

Raw SQL string 被 parser 解析為 **parse tree**（由 query node 構成的 tree）。

入口函數：[`postgres.c:exec_simple_query`](https://github.com/pipelinedb/pipelinedb/blob/master/src/backend/tcop/postgres.c#L900)，其中 [parsing 是第一步](https://github.com/pipelinedb/pipelinedb/blob/master/src/backend/tcop/postgres.c#L955)。

| 概念 | 說明 |
|------|------|
| Parse Tree Node Types | `parsenodes.h`：`SelectStmt`、`InsertStmt`、`DeleteStmt`、`UpdateStmt`、`CreateStmt` 等 |
| 輸出 | 純語法結構，未做語義檢查 |
| Debug | `SET debug_print_parse = on`，輸出寫入 server log（路徑見 `pipelinedb.conf` / `postgresql.conf`） |

> 補充（Senior Dev）：`debug_print_*` 系列參數輸出極其詳細（包含完整 tree dump），在 production 開啟會 flood log。建議只在 dev session 中 `SET`（僅影響當前 connection）而非全域 `ALTER SYSTEM`。另外 `debug_pretty_print = on` 可以讓 tree dump 變成 indented 格式，大幅提升可讀性。
>
> Parser 使用 `gram.y` + `scan.l`（flex/bison）生成。PG 的 SQL 語法是 LALR(1)，有一個著名的 corner case：`SELECT 1 AS foo UNION SELECT 2` vs `SELECT 1 AS foo, 2 AS bar` —— parser 需要 lookahead context 來區分 `UNION` 關鍵字和 alias `UNION`，這導致了某些 ambiguous grammar 的 shift/reduce conflict。理解這點對閱讀 `gram.y` 原始碼很有幫助。

---

## 3. Analyzer (Semantic Analysis & Rewriting)

Parse tree 經過 [semantic analysis 和 transformation](https://github.com/pipelinedb/pipelinedb/blob/master/src/backend/tcop/postgres.c#L1112) 後被改寫為一個或多個等價的 Query object。

### View Rewriting 範例

```sql
CREATE VIEW example_view AS SELECT * FROM table WHERE table.column = 'foo';

-- Client 查詢 View
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
> 另外，PG 的 rewrite system 允許 recursive rewrite。當一個 query tree 被 rewrite 後，新的 tree 會再次經過 rewrite 階段，直到沒有更多 rule 可以觸發。這意味著 infinite recursion 是可能的（例如兩個互相轉換的 rule），PG 以 `max_rewrite_depth` 做保護（超出即報錯）。

---

## 4. Planner (Query Optimization)

Analyzed Query object [傳入 Planner](https://github.com/pipelinedb/pipelinedb/blob/master/src/backend/tcop/postgres.c#L1115)，生成 **plan tree**（node types 見 `plannodes.h`）。

```
Query → Planner → Plan Tree
```

### 三種掃描策略範例

```sql
CREATE TABLE planz (id integer, data text);

-- 假設 id 有 B-tree index
EXPLAIN ANALYZE SELECT * FROM planz WHERE id = 42;
```

Without index（Seq Scan）：
```
 Seq Scan on planz  (cost=0.00..34.00 rows=2400 width=4)
   (actual time=0.002..0.002 rows=0 loops=1)
```

With index（Bitmap Heap Scan）：
```
 Bitmap Heap Scan on planz  (cost=4.20..13.67 rows=6 width=36)
   Recheck Cond: (id = 42)
   ->  Bitmap Index Scan on planz_id_index
         (cost=0.00..4.20 rows=6 width=0)
         Index Cond: (id = 42)
```

Planner 根據 cost model 在 `IndexScan`（`plannodes.h:L333`）與 `SeqScan`（`plannodes.h:L287`）之間選擇，此處 `bitmap_scan_plan` 由 `createplan.c:create_bitmap_scan_plan` 構建。

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
> `effective_cache_size` 的影響極大但不直觀：它不分配任何 memory，只是讓 planner **假設這麼多 data 已在 cache**。值越大，planner 越傾向用 Index Scan（因為它認為 random read 的代價因為 cache hit 而變低）。`effective_cache_size` 應設為 `shared_buffers + OS filesystem cache` 的總和（通常是總 RAM 的 50-75%）。
>
> `geqo` (Genetic Query Optimizer) 的預設觸發閾值是 `geqo_threshold = 12`（超過 12 個 table 的 join 會切換為 genetic search）。對於 OLTP 這幾乎不觸發；對於 OLAP / data warehouse，你可能需要調高 `geqo_threshold` 讓 exhaustive search 跑久一點以獲取更好的 plan。
>
> `join_collapse_limit` 和 `from_collapse_limit` 控制 join order 的搜索範圍。預設值 `8` 對大多數場景足夠；如果需要 10+ table join，可以逐步調高（注意 plan time 會呈指數增長）。

---

## 5. Executor

Plan tree 傳入 Executor 後，[PortalRun](https://github.com/pipelinedb/pipelinedb/blob/master/src/backend/tcop/pquery.c#L719) 為入口點，最終進入 [`execMain.c:ExecutePlan`](https://github.com/pipelinedb/pipelinedb/blob/master/src/backend/executor/execMain.c#L1529)。

### Volcano Model（Pull-based）

Executor 採用 **pull-based iterator model**（Volcano model）。每個 plan node 都是一個 iterator，由 parent node 從 child node "pull" 資料：

```
ExecutePlan → 遞迴呼叫 execProcNode(node)
    → 每個 node 輸出 tuple
    → parent node 以此為 input
    → root node 輸出 = final query result
```

執行順序是 top-down（pull）+ bottom-up（return tuple）。

### Executor State Nodes

Executor 為每個 plan node 建立對應的 executor state node（`execnodes.h`），初始化在 [`execMain.c:InitPlan`](https://github.com/pipelinedb/pipelinedb/blob/master/src/backend/executor/execMain.c#L754)，遞迴呼叫 [`execProcNode.c:ExecInitNode`](https://github.com/pipelinedb/pipelinedb/blob/master/src/backend/executor/execProcnode.c#L139) 為每個 plan node 建立 executor node。

| 概念 | Source | 說明 |
|------|--------|------|
| Plan Node | `plannodes.h` | planner 的輸出，"what to do" |
| Executor Node | `execnodes.h` | executor 的 runtime state，"where we are" |
| Entry | `execProcnode.c:ExecProcNode` | [switch-case dispatch](https://github.com/pipelinedb/pipelinedb/blob/master/src/backend/executor/execProcnode.c#L448) 每個 node type 的分發邏輯 |

> 補充（Senior Dev）：Volcano model 的核心是每個 node 實現 `ExecInitNode`、`ExecProcNode`、`ExecEndNode` 三個接口。這使得 plan tree 可以任意嵌套組合（如 `Sort → HashAgg → SeqScan`），每個 node 只關心從 child pull 資料再做自己的處理。
>
> PG 9.6+ 引入 Parallel Query：`Gather` node 作為 plan tree 的 root，底下的 partial plan 被 `N` 個 parallel worker 同時執行。`ExecGather` 從每個 worker 的 `TupleQueue` pull tuple 合併輸出。Parallel Query 只在 Seq Scan、Hash Join（PG 11+）、B-tree Index Scan（PG 11+）等 subset 可用。
>
> PG 11+ 引入 JIT compilation（基於 LLVM）：ExecProcNode 的 tight loop（如 `WHERE` clause evaluation、`target list` expression）在滿足 `jit_above_cost` 閾值時被 JIT compile 為 native code，對 CPU-bound query 的加速顯著（通常 10-30%）。
>
> 關於 Portal：
> - **Unnamed Portal**：`exec_simple_query` 建立的預設 portal，一次性執行全部 plan
> - **Named Portal**：`DECLARE CURSOR` 建立，可分段 fetch（`FETCH 100 FROM cursor`）
> - Portal 本質是 executor state 的 holder，記錄目前執行到 plan tree 的哪個位置
>
> 理解 Portal 對於排查 cursor 相關效能問題很重要：cursor 的 query plan 是 executed lazily（每 `FETCH` 一次跑一輪 executor，而非一次性產生全部結果再 cache）。

---

## 6. Client Response

Result tuple 在 [`ExecutePlan` 內部](https://github.com/pipelinedb/pipelinedb/blob/master/src/backend/executor/execMain.c#L1489) 發送給 client。

流程：
1. `ExecProcNode` 產生每個 tuple
2. Tuple 傳給 [destination receiver 的 callback function](https://github.com/pipelinedb/pipelinedb/blob/master/src/backend/executor/execMain.c#L1572)
3. 最常見的 receiver 是 [regular client receiver](https://github.com/pipelinedb/pipelinedb/blob/master/src/backend/tcop/postgres.c#L1174)（在 `exec_simple_query` 中建立）
4. 用 [`printtup.c:printtup`](https://github.com/pipelinedb/pipelinedb/blob/master/src/backend/access/common/printtup.c) 將 tuple 格式化為 wire protocol message 發送

> 補充（Senior Dev）：Destination receiver 不只一種。除了 client receiver 外還有：
> - **SPI receiver**：PL/pgSQL function 內部用，tuple 存到 `SPITupleTable`
> - **Materialize receiver**：`Tuplestore`，用於 CTE (`WITH`)、cursor materialization
> - **COPY receiver**：`COPY TO` 輸出
>
> 理解 destination receiver 的抽象能幫你理解為什麼 `SELECT count(*)` 的 executor 不會把全部 row 送回 client：Aggregate node 攔截了所有 tuple 只輸出最終的 count=1 row，所以中間 tuple 根本沒經過 `printtup`。這也是 `EXPLAIN ANALYZE` 中 `actual time` 和 `rows` 分別記錄 executor 內部時間和最終傳輸量的原因。
>
> wire protocol 層面：`printtup` 將每個 tuple 序列化為 `DataRow ('D')` message。PG 14 引入了 pipeline mode（`libpq` 支援），client 可以在一條 connection 上連續發出多個 query 而不用等待前一個完成，大幅提升高延遲網路下的吞吐量（類似 HTTP pipelining），但每個 query 仍然在 server side 的單個 backend process 中依序執行。

---

## 關鍵 Source File 索引

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
