# PostgreSQL WITH RECURSIVE 死循環防禦與解法

> 來源：[digoal - PostgreSQL 递归死循环案例及解法 (2016-07-23)](https://github.com/digoal/blog/blob/master/201607/20160723_01.md)
>
> 2026 更新：PG 14+ `CYCLE` clause 徹底解決方案、temp 相關 PG 17 改進、Production 防禦體系。

---

## 問題：Recursive CTE 遇到數據迴圈（Cycle）

WITH RECURSIVE 是處理樹形 / 圖形查詢的標準語法，但如果數據中存在循環引用（例如 `c1 = 1, c2 = 1`，自己指向自己），recursive CTE 會無限迭代，不斷產生中間結果、寫入 temp file，最終耗盡磁盤空間影響業務。

### 迴圈數據範例

```sql
CREATE TABLE test(c1 int, c2 int, info text);
CREATE INDEX idx_test_01 ON test(c1);
CREATE INDEX idx_test_02 ON test(c2);

INSERT INTO test VALUES
    (9,8,'test'), (8,7,'test'), (7,6,'test'),
    (6,5,'test'), (5,4,'test'), (4,3,'test'),
    (3,2,'test'), (2,1,'test'),
    (1,1,'test');  -- ← 這裡形成 loop：c1=1 的 parent 也是 1
```

### 死循環觸發

```sql
WITH RECURSIVE t(c1, c2, info) AS (
    SELECT * FROM test WHERE c1 = 9
    UNION ALL
    SELECT t2.* FROM test t2 JOIN t ON (t.c2 = t2.c1)
)
SELECT count(*) FROM t;
-- 永遠不會結束：9→8→7→6→5→4→3→2→1→1→1→1→...
```

### Temp File 暴增過程

```bash
# 開始後不久
$ ls -lh $PGDATA/base/pgsql_tmp/
-rw------- pgsql_tmp8997.5  89M

# 數分鐘後
-rw------- pgsql_tmp8997.5 575M

# 繼續增長
-rw------- pgsql_tmp8997.5 1.0G
-rw------- pgsql_tmp8997.6 1.0G
-rw------- pgsql_tmp8997.7 435M
```

`log_temp_files` 只在 query 結束時才記錄 log，中途不會有警告——這是 PG 社區版的行為限制（原文作者建議階段性記錄，至今未實作）。

---

## 解法 1：`temp_file_limit` 硬限制（2016 年做法）

### 相關參數

```ini
# postgresql.conf
temp_buffers = 8MB            # temp buffer 大小，min 800kB
temp_file_limit = -1          # 單個 session 最大 temp file 空間 (kB)，-1 = 無限制
log_temp_files = 102400       # temp file 超過此大小 (kB) 時記錄 log（query 結束後記錄）
temp_tablespaces = ''         # temp 表空間列表，'' = 預設
```

### 會話級限制

```sql
SET temp_file_limit = '10MB';

WITH RECURSIVE t(c1, c2, info) AS (
    SELECT * FROM test WHERE c1 = 9
    UNION ALL
    SELECT t2.* FROM test t2 JOIN t ON (t.c2 = t2.c1)
)
SELECT count(*) FROM t;
-- ERROR: 53400: temporary file size exceeds temp_file_limit (10240kB)
-- LOCATION: FileWrite, fd.c:1491
```

設定後死循環在 temp file 達到 10MB 時被強制中斷，不再無限增長。

### 手動取消

```sql
-- Ctrl+C 或從另一個 session：
SELECT pg_cancel_backend(pid);
-- ERROR: 57014: canceling statement due to user request
```

取消後可看到 temp file 紀錄（query 結束後才輸出）：

```
LOG: temporary file: path "base/pgsql_tmp/pgsql_tmp8997.7", size 1073741824
LOG: temporary file: path "base/pgsql_tmp/pgsql_tmp8997.6", size 1073741824
LOG: temporary file: path "base/pgsql_tmp/pgsql_tmp8997.5", size 1073741824
```

### pg_hint_plan 逐 Query 指定限制

```sql
/*+
Set (temp_file_limit '10MB')
*/
WITH RECURSIVE t(c1, c2, info) AS (...)
SELECT count(*) FROM t;
```

---

## 解法 2：`CYCLE` Clause — PG 14+ 根本方案

PG 14 引入了 SQL 標準的 `CYCLE` clause，從遞迴引擎內部檢測 cycle，無需外部限制：

```sql
WITH RECURSIVE t(c1, c2, info) AS (
    SELECT * FROM test WHERE c1 = 9
    UNION ALL
    SELECT t2.* FROM test t2 JOIN t ON (t.c2 = t2.c1)
)
CYCLE c1 SET is_cycle USING path  -- ← 追蹤 c1，檢測重複
SELECT * FROM t;
-- 遇到 cycle 時自動停止該分支的遞迴，is_cycle = true 標記循環行
```

**輸出：**

| c1 | c2 | info | is_cycle | path |
|----|----|------|----------|------|
| 9 | 8 | test | f | {(9)} |
| 8 | 7 | test | f | {(9),(8)} |
| ... | ... | ... | ... | ... |
| 1 | 1 | test | f | {(9),(8),(7),(6),(5),(4),(3),(2),(1)} |
| 1 | 1 | test | **t** | {(9),(8),(7),(6),(5),(4),(3),(2),(1),(1)} |

> 補充（Senior Dev）：`CYCLE` clause 是 **PG 14 新增的 SQL 標準語法**，徹底解決了 recursive CTE 的死循環問題。內部實作維護一個 hash table 追蹤已訪問節點，當 cycle column 重複出現時停止該分支。相比 `temp_file_limit` 的「暴力中斷」，CYCLE 是「優雅終止」——不需要猜測 limit 值，不會誤殺正常的大遞迴查詢。

---

## Senior Dev：Production 防禦體系

### 1. 三層防禦

| 層級 | 機制 | 適用場景 |
|------|------|---------|
| **SQL 層**（根本） | `CYCLE` clause (PG 14+) | 所有 recursive CTE 都應加 |
| **SQL 層**（相容） | 手動 breadcrumb `array[c1]` + `NOT (c1 = ANY(path))` | PG < 14 |
| **Session 層**（兜底） | `SET LOCAL temp_file_limit = '1GB'` | 防止未知 CTE 炸庫 |
| **DB 層**（全域） | `temp_file_limit` in postgresql.conf | 預防性全域上限 |

### 2. PG < 14 的手動 Cycle Detection（breadcrumb 模式）

```sql
WITH RECURSIVE t(c1, c2, info, path, depth) AS (
    SELECT *, ARRAY[c1], 0 FROM test WHERE c1 = 9
    UNION ALL
    SELECT t2.*, t.path || t2.c1, t.depth + 1
    FROM test t2
    JOIN t ON (t.c2 = t2.c1 AND NOT (t2.c1 = ANY(t.path)))  -- ← 手動 cycle 檢測
)
SELECT * FROM t;
```

> 補充（Senior Dev）：breadcrumb 模式雖然有效，但缺點是 (a) 每次遞迴都要 scan 整個 `path` array（O(n)），大型樹深度大時 CPU 成本高；(b) `path` array 隨遞迴增長，佔用更多 temp space。PG 14 的 `CYCLE` clause 內部用 hash table，O(1) 查重。

### 3. `SET LOCAL` vs `SET SESSION`

```sql
-- 僅影響當前 transaction（最安全）
SET LOCAL temp_file_limit = '1GB';

-- 影響整個 session 的所有後續 query（小心 side effect）
SET SESSION temp_file_limit = '1GB';
```

建議在 application 層以 `SET LOCAL` 包裹 recursive CTE，不影響同一 connection pool connection 的其他 query。

### 4. Temp File 清理機制

- Query 結束時自動清理（無論正常或異常終止）
- DB startup 時，startup process 會清理殘留 temp file
- `pg_cancel_backend()` 會觸發清理；`pg_terminate_backend()` 可能留下孤兒 temp file（crash recovery 時清理）

### 5. Temp File 監控（PG 17）

```sql
-- 查看當前 temp file 使用量（PG 14+）
SELECT pg_stat_get_temp_files();

-- 查看 session 級 temp 寫入量
SELECT temp_files, temp_bytes
FROM pg_stat_database WHERE datname = current_database();
```

### 6. PG 14-17 Recursive CTE 相關改進

| 版本 | 改進 |
|------|------|
| PG 14 | `CYCLE` clause（根本解法） |
| PG 14 | `SEARCH` clause（`BREADTH FIRST` / `DEPTH FIRST` 排序） |
| PG 15 | Recursive CTE 的 materialization 策略優化（`MATERIALIZED` / `NOT MATERIALIZED` hint） |
| PG 16 | Recursive CTE 內部 hash table 可 spill to disk（避免 memory overflow） |
| PG 17 | `WORK_MEM` 對 recursive CTE hash table 的分配更精細 |

### 7. `SEARCH` Clause 搭配 `CYCLE`（PG 14+）

```sql
WITH RECURSIVE t(c1, c2, info) AS (
    SELECT * FROM test WHERE c1 = 9
    UNION ALL
    SELECT t2.* FROM test t2 JOIN t ON (t.c2 = t2.c1)
)
SEARCH DEPTH FIRST BY c1 SET order_col   -- ← 深度優先，c1 排序
CYCLE c1 SET is_cycle USING path
SELECT * FROM t ORDER BY order_col;
```

---

## 原文 RDS PG 改進建議（2016 年，回顧）

| 建議 | 2026 現狀 |
|------|---------|
| 動態 `temp_file_limit`（依剩餘空間自動調整） | **未實作**，仍為靜態值 |
| Group / User / DB 級別的 temp 限制 | PG 16 引入 `pg_db_role_setting` 可設 DB+Role 級參數，部分解決 |
| 階段性記錄 temp file log | **未實作**，仍在 query 結束時記錄 |
| Greenplum Resource Queue 式資源管控 | PG 17 仍無此機制，需靠 connection pooler + OS cgroup 輔助 |

> 補充（Senior Dev）：`temp_file_limit` 的單一 session 模型在 multi-tenant / connection pooling 場景有盲點——多個小 query 各自不超過 limit 但疊加後仍可能炸庫。建議在 OS 層用 cgroup v2 對 PG process group 設 `MemoryMax` + IO limit 作為最後防線。
