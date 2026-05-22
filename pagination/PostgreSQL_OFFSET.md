# PostgreSQL OFFSET 原理與使用注意事項

> 來源：[digoal - PostgreSQL offset 原理，及使用注意事项 (2016-04-02)](https://github.com/digoal/blog/blob/master/201604/20160402_02.md)

---

## 核心發現：OFFSET 跳過的 row 仍會被計算

OFFSET 只過濾最終結果不發送給 client，但跳過的 row 仍然完整經過了 SELECT list 中的所有 expression 計算（包括 function call）。這不是 bug，而是 SQL 執行模型的設計結果。

> 註：此行為是 SQL 標準定義的邏輯執行順序所決定，非 PostgreSQL 特有。MySQL、SQL Server、Oracle 在 OFFSET 行為上完全一致：OFFSET 發生在 SELECT list 計算之後。Tom Lane 的回覆也確認了這一點。

### 實驗：VOLATILE function 與 OFFSET

創建一個 VOLATILE function，每次呼叫印出 `NOTICE: called`：

```sql
CREATE OR REPLACE FUNCTION f() RETURNS void AS $$
DECLARE
BEGIN
  RAISE NOTICE 'called';
END;
$$ LANGUAGE plpgsql STRICT VOLATILE;
```

```sql
SELECT f(), * FROM (VALUES (1), (2), (3), (4), (5), (6)) t(id) OFFSET 3 LIMIT 2;
-- NOTICE:  called
-- NOTICE:  called
-- NOTICE:  called
-- NOTICE:  called
-- NOTICE:  called
--  f | id
-- ---+----
--    |  4
--    |  5
-- (2 rows)
```

OFFSET 3 跳過了前 3 條 row，但 f() 被呼叫了 **5 次**（等於 OFFSET + LIMIT = 3 + 2），而不是預期的 2 次。連跳過的 row 也執行了 f()。

### STABLE function 同樣行為

將 function 改為 STABLE，結果不變：

```sql
ALTER FUNCTION f() STABLE;
SELECT f(), * FROM (VALUES (1), (2), (3), (4), (5), (6)) t(id) OFFSET 3 LIMIT 2;
-- NOTICE:  called
-- NOTICE:  called
-- NOTICE:  called
-- NOTICE:  called
-- NOTICE:  called
-- (same 5 calls)
```

### IMMUTABLE 的特殊行為

改為 IMMUTABLE 後，optimizer 在生成 execution plan 之前就把 function 常量化（constant folding），因此無論 OFFSET 值多大一律只呼叫 1 次：

```sql
ALTER FUNCTION f() IMMUTABLE;
SELECT f(), * FROM (VALUES (1), (2), (3), (4), (5), (6)) t(id) OFFSET 3 LIMIT 2;
-- NOTICE:  called
-- (only 1 call)
```

> 補充（Senior Dev）：IMMUTABLE 的這種行為是 optimizer 層面的常數折疊（constant folding），前提是 function 輸入參數也為常數。如果 IMMUTABLE function 接受 column 值作為參數，則仍會每 row 計算一次。此外，`STRICT` modifier 表示任一參數為 NULL 則不執行 function body 直接返回 NULL，與 OFFSET 行為無關。

### WHERE 的對比：WHERE 過濾的 row 不會被計算

```sql
SELECT f(), * FROM (VALUES (1), (2), (3), (4), (5), (6)) t(id) WHERE id = 1 LIMIT 5;
-- NOTICE:  called
--  f | id
-- ---+----
--    |  1
-- (1 row)
```

f() 只被呼叫 1 次。這是 WHERE 與 OFFSET 的根本差異：

| 機制 | 過濾階段 | 被跳過 row 是否計算 |
|------|----------|-------------------|
| WHERE | 在 SELECT list 計算之前（predicate pushdown） | 否 |
| OFFSET | 在 SELECT list 計算之後，僅不發送給 client | **是** |

> 補充（Senior Dev）：這反映了 SQL 邏輯執行順序：`FROM → WHERE → SELECT → OFFSET/LIMIT`。WHERE 在 SELECT 之前過濾 row，OFFSET 在 SELECT 之後丟棄結果。所以 OFFSET 節省的是 **網絡傳輸成本**，不是 **計算成本**。這也是為什麼 keyset pagination（`WHERE id > last_id ORDER BY id LIMIT N`）在大偏移量場景遠優於 `OFFSET N`。

---

## 實戰性能問題：大 OFFSET 帶來的隱性成本

當 OFFSET 值很大時（如 `OFFSET 100000 LIMIT 1`），被跳過的 100,000 條 row 全部會觸發 SELECT list 中的 expression 計算。如果 SELECT 中包含：

- VOLATILE function
- 複雜的 scalar subquery
- `CASE` / `COALESCE` 等 expression
- 其他非 IMMUTABLE 的計算

則這些計算會在 100,000 條 row 上白白執行一次再丟棄，造成極大的 CPU 浪費，也可能讓 performance troubleshooting 誤判瓶頸（因為 EXPLAIN 中的 actual time 包含了這些計算，但最終只返回 1 row，直覺上不該這麼慢）。

Tom Lane（PostgreSQL 核心開發者）的回覆：

> No, it's not a bug. OFFSET only results in the skipped tuples not being delivered to the client; it does not cause them not to be computed.
>
> You could probably do something with a two-level select with the OFFSET in the sub-select and the volatile function in the top level.
>
> — Tom Lane

---

## 解法：子查詢隔離 OFFSET 與計算

把 expensive function 放在外層 SELECT，OFFSET 限制在內層子查詢。內層子查詢只掃描 row（不執行 function），外層才對最終結果執行 function：

```sql
ALTER FUNCTION f() VOLATILE;

SELECT f(), *
FROM (
    SELECT * FROM (VALUES (1), (2), (3), (4), (5), (6)) t(id)
    OFFSET 3 LIMIT 2
) t;
-- NOTICE:  called
-- NOTICE:  called
--  f | id
-- ---+----
--    |  4
--    |  5
-- (2 rows)
```

f() 只被呼叫 **2 次**（等於最終返回的 row 數），OFFSET 3 跳過的 3 條 row 不再觸發 function 計算。

這個方案的關鍵在於：**內層子查詢只投影 column，不含 function call**。OFFSET 在內層執行時只需處理 column 值（幾乎零成本），外層才對篩選後的結果執行 function。

> 補充（Senior Dev）：在實際應用中，如果 function 接受的是 table column 值（如 `f(col)`），而非 `f()` 無參數，則此解法無效——因為內層必須投影該 column 才能傳給外層。這種情況下應考慮：
> 1. 改用 WHERE-based pagination（keyset / cursor）完全避免 OFFSET
> 2. 若 function 邏輯可用 CASE 改寫為 IMMUTABLE expression，讓 optimizer 處理
> 3. 若 function 為 STABLE 且結果可緩存，考慮 subquery 中先 `DISTINCT` 去重 column 值再 JOIN 回來，減少呼叫次數
>
> [PG 13+] **Incremental Sort**：當查詢包含 `ORDER BY x OFFSET N LIMIT M` 時，PG 13+ 的 Incremental Sort 可以分批排序（而非一次性全量排序），降低大 OFFSET + ORDER BY 場景的排序成本。但注意：這只優化排序階段，OFFSET 跳過的 row 仍然會計算 SELECT list 中的 expression——Incremental Sort 不改變 OFFSET 對計算成本的影響。
>
> [PG 12+] **FETCH FIRST 語法**：PG 12 起支援 SQL 標準語法 `FETCH FIRST n ROWS ONLY` 和 `OFFSET n ROWS FETCH FIRST m ROWS ONLY`，行為與 `LIMIT` / `OFFSET` 完全等價：
> ```sql
> SELECT * FROM tab ORDER BY id OFFSET 100 ROWS FETCH FIRST 10 ROWS ONLY;
> -- 等價於
> SELECT * FROM tab ORDER BY id OFFSET 100 LIMIT 10;
> ```
