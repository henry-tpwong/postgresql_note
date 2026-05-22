# PostgreSQL 欄位順序：ADD COLUMN 的物理限制、View 虛擬修改與 Byte Alignment 效能影響

> 來源：
> - [digoal - PostgreSQL 將字段加入指定位置 — 表字段位置的"虛擬修改"實現 (2016-02-29)](https://github.com/digoal/blog/blob/master/201602/20160229_01.md)
> - 延伸討論：Byte Alignment 對 Row Width、Page I/O、Recheck Cond 的全鏈路效能影響

---

DBA 問「欄位加在哪裡」表面上是順序問題，實質觸及三個層面：
1. PostgreSQL **物理儲存機制**：ADD COLUMN 永遠加在末尾
2. **虛擬修改方案**：Simple View 重排欄位外觀的 trade-off
3. **更深層的效能問題**：欄位排列順序如何影響 Byte Alignment → Row Width → Page Density → I/O / Memory / CPU

---

## 1. PostgreSQL 的物理現實：ADD COLUMN 永遠在末尾

在 PostgreSQL 中，tuple 的物理 storage layout 是固定的，欄位的解釋順序由 system catalog `pg_attribute` 中的 `attnum` 定義。

```sql
SELECT attname, attnum, attisdropped
FROM pg_attribute
WHERE attrelid = 'tbl'::regclass;
```

```
 attname  | attnum | attisdropped
----------+--------+--------------
 tableoid |     -7 | f
 cmax     |     -6 | f
 xmax     |     -5 | f
 cmin     |     -4 | f
 xmin     |     -3 | f
 ctid     |     -1 | f
 id       |      1 | f
 info     |      2 | f
 crt_time |      3 | f
 c1       |      4 | f
(10 rows)
```

- 前 6 個是 system column（`tableoid`, `cmax`, `xmax`, `cmin`, `xmin`, `ctid`），`attnum` 為負數
- 用戶欄位從 `attnum = 1` 開始遞增
- `ALTER TABLE ... ADD COLUMN` 只會將新欄位設為最大的 `attnum`，即**永遠加在末尾**
- 不支援 MySQL 的 `ALTER TABLE t ADD COLUMN c1 INT AFTER id` 語法

> 補充（Senior Dev）：`pg_attribute.attnum` 是 int16，欄位一旦被 DROP（變為 `attisdropped = t`），其 `attnum` 不會被回收復用。長期頻繁 ADD/DROP column 的表會因為 `attnum` 耗盡（上限 1600）而需要 `VACUUM FULL` 或重建表。

---

## 2. Simple View 虛擬修改：外觀順序 vs 物理順序

PostgreSQL 的 Simple View 是一個輕量方案——View 本身會被自動加上 rewrite rule（INSERT / UPDATE / DELETE），對 View 的讀寫行為與對基表幾乎一致。

### 2.1 實作

```sql
CREATE TABLE tbl (id INT, info TEXT, crt_time TIMESTAMP);
ALTER TABLE tbl ADD COLUMN c1 INT;

-- 用 View 重排欄位：c1 被「放到」crt_time 之前
CREATE VIEW v_tbl AS
  SELECT id, info, c1, crt_time FROM tbl;
```

```sql
INSERT INTO v_tbl VALUES (1, 'test', 2, now());

SELECT * FROM v_tbl;
--  id | info | c1 |          crt_time
-- ----+------+----+----------------------------
--   1 | test |  2 | 2016-02-29 14:07:19.171928

SELECT * FROM tbl;
--  id | info |          crt_time          | c1
-- ----+------+----------------------------+----
--   1 | test | 2016-02-29 14:07:19.171928 |  2
```

View 中的 `c1` 在 `crt_time` 之前；基表中 `c1` 仍在末尾。`pg_attribute` 物理順序不變。

### 2.2 優點

- 滿足應用層欄位邏輯排列需求，不需改動 SQL
- 避免昂貴的表重寫（`ALTER TABLE ... ALTER COLUMN ... SET DATA TYPE` 或 `VACUUM FULL`），View 創建是瞬間的 metadata 操作

### 2.3 代價

**僅是 View，非物理真實**：任何繞過 View 直接查詢基表的操作看到的是原始末尾順序。

**DDL 傳染性**：對基表執行 DDL 會 cascade 到 View：

```sql
ALTER TABLE tbl DROP COLUMN info;
-- ERROR: cannot drop table tbl column info because other objects depend on it
-- DETAIL: view v_tbl depends on table tbl column info
```

必須 `DROP ... CASCADE` 後重建 View：

```sql
ALTER TABLE tbl DROP COLUMN info CASCADE;
-- NOTICE: drop cascades to view v_tbl

CREATE VIEW v_tbl AS SELECT id, c1, crt_time FROM tbl;
```

被 DROP 的欄位在 `pg_attribute` 中變為 `attisdropped = t`，`attnum` 仍佔用不釋放：

```
 attname            | attnum | attisdropped
--------------------+--------+--------------
 id                 |      1 | f
 ........pg.dropped.2........ |      2 | t   -- DROP 後仍佔位
 crt_time           |      3 | f
 c1                 |      4 | f
```

**維護成本**：每次基表結構變更都需同步更新 View 定義。德哥原話：「万不得已，也不要这么用。除非业务上不想改SQL。」

> 補充（Senior Dev）：Simple View 的自動 rewrite rule 在 PG 9.3+ 的運作原理：當 View 是 `SELECT * FROM single_table`（無 JOIN、無 DISTINCT、無 GROUP BY、無 LIMIT）時，PG 會將其標記為 auto-updatable，自動生成 INSERT/UPDATE/DELETE rule。這讓 View 行為接近真正的 table，但 rule-based rewrite 在複雜條件下可能產生非直覺行為（如 `RETURNING` clause 在不同 PG 版本的表現不一致）。若需要更可控的 DML 行為，建議用 `INSTEAD OF` trigger on View。

---

## 3. Byte Alignment：欄位順序如何影響 Row Size 與全鏈路效能

DBA 關心「欄位加在哪裡」，表面是順序問題，**深層是物理儲存效率**。

### 3.1 數據頁（Page）是 I/O 的最小單位

PostgreSQL 讀寫數據以 **8KB page** 為最小單位，不是以 row 為單位。查詢一行，會把包含該行的整個 8KB page 從 disk 讀入 shared_buffers。

**Row Width ↑ → Rows per Page ↓ → I/O ↑ → Memory Cache Efficiency ↓**

### 3.2 為什麼 Row Width 受 Column Order 影響？

CPU 存取不同 data type 需要**記憶體對齊（Byte Alignment）**：

| Type | Size | Alignment |
|------|------|-----------|
| `bigint` / `timestamp` / `timestamptz` / `double precision` | 8 bytes | 8-byte boundary |
| `integer` / `real` / `date` | 4 bytes | 4-byte boundary |
| `smallint` | 2 bytes | 2-byte boundary |
| `boolean` / `char` / `"char"` | 1 byte | 1-byte boundary |

如果一個 8-byte 對齊的 column 前面的 column 總長度不是 8 的倍數，PG 會在**中間插入無用的 padding byte** 來對齊。

### 3.3 具體例子

假設四個 column：`c1 bool` (1B), `c2 bigint` (8B), `c3 bool` (1B), `c4 int` (4B)

**差順序：`bool → bigint → bool → int`**

```
Offset 0:  [c1: 1B]
Offset 1:  [PADDING: 7B]  ← 強制對齊，讓 c2 從 8 的倍數開始
Offset 8:  [c2: 8B]
Offset 16: [c3: 1B]
Offset 17: [PADDING: 3B]  ← 強制對齊，讓 c4 從 4 的倍數 (20) 開始
Offset 20: [c4: 4B]
Offset 24: [PADDING: 0-4B] ← row header alignment (MAXALIGN)
```
**Row Size = 1 + 7 + 8 + 1 + 3 + 4 + alignment ≈ 24~28 bytes，padding 佔 41%**

**好順序：`bigint → int → bool → bool`**

```
Offset 0:  [c2: 8B]   ← 本身就是 8 的倍數
Offset 8:  [c4: 4B]   ← 8 是 4 的倍數
Offset 12: [c1: 1B]
Offset 13: [c3: 1B]
Offset 14: [PADDING: 2B] ← 僅 tail alignment
Offset 16: row end
```
**Row Size = 8 + 4 + 1 + 1 + 2 ≈ 16 bytes，padding 僅佔 12.5%**

**同樣的資料，僅欄位順序不同 → 每 row 從 24 bytes 降至 16 bytes，一個 8KB page 多裝 ~50% 的 row。**

### 3.4 效能鏈式反應

這個差異不是單一環節的，而是**全鏈路**：

| 環節 | 影響 |
|------|------|
| **Disk I/O** | Page 固定 8KB。Row 越大，讀取相同 row 數需搬更多 page → 物理 disk read 次數增加 |
| **shared_buffers** | Cache 中能存放的 row 數減少 → cache hit ratio 下降 |
| **Page 讀入後的所有操作** | SQL 引擎需要解析 row 內每個 column 的 offset，跳過 padding byte → CPU 指令增加 |
| **Bitmap Heap Scan 的 Recheck Cond** | Lossy bitmap 把整個 page 讀入後逐 row 重新驗證條件。Row 越寬、padding 越多，逐 row 檢查的 CPU 耗費越大 |
| **Seq Scan 的 Filter** | 沒有 Recheck Cond，但仍有 Filter。同樣需要解析 row 結構、跳過 padding |
| **Index Scan** | 從 index 拿到 TID 後回 Heap 取 row 時，同樣要解析該 page 內的 target row |

> 關鍵澄清：Recheck Cond 只是 bitmap scan 這個特定場景下的二次驗證。但 byte alignment 帶來的效能損耗**不只發生在 Recheck Cond**。任何需要讀取、解析、過濾 row 的操作（Filter、JOIN、aggregate）都會因為 row 內部 padding 過多而增加 CPU 成本。

### 3.5 最佳實踐：Column Order 規則

```
1. Fixed-width, high-alignment columns FIRST
   bigint(8), timestamp(8), double precision(8), integer(4), date(4), real(4)

2. Variable-width columns MIDDLE
   text, varchar, numeric, bytea, jsonb

3. Fixed-width, low-alignment columns LAST
   smallint(2), boolean(1), char(1)

4. NULL-able columns AFTER their NOT NULL counterparts of the same alignment
   (nullable columns may need extra null-bitmap handling)
```

> 補充（Senior Dev）：
>
> **怎麼驗證自己的表有 alignment waste？**
> ```sql
> -- 檢查實際 row width
> SELECT
>   relname,
>   reltuples,
>   relpages,
>   (relpages * 8192) / GREATEST(reltuples, 1) AS avg_row_bytes,
>   pg_size_pretty(pg_relation_size(oid)) AS table_size
> FROM pg_class WHERE relname = 'your_table';
> ```
> 若 `avg_row_bytes` 明顯大於「所有 column 的 data type size 總和」，說明 padding 浪費了可觀空間。
>
> **修正方案**：PG 本身不提供 `ALTER TABLE ... MODIFY COLUMN ... AFTER ...`。修正順序唯一辦法是重建表：
> ```sql
> CREATE TABLE tbl_new AS SELECT col_a, col_b, ... FROM tbl;
> -- 或使用 pg_repack / pg_squeeze 線上重建
> ```
>
> **生產中要不要刻意優化 column order？**：
> - 寬表（>20 columns）且 QPS / latency 敏感的 OLTP 表 → 值得。每 row 省 8-16 bytes，百億行級別 = 省 TB 級儲存
> - 窄表或分析型（少數 column 的查詢） → 收益有限，不優先
> - 最有效的仍是只 SELECT 需要的 column（避免 `SELECT *`）、用 Covering Index 繞過 Heap access
>
> **NULL bitmap 的額外考量**：每個 tuple 有一個 NULL bitmap（每 8 個 column 用 1 byte）。如果表有 9-16 個 column（部分 nullable），NULL bitmap 本身佔 2 bytes。把 NOT NULL column 集中排列可以讓 NULL bitmap 更緊湊，但這個影響遠小於 byte alignment 的 padding。

---

## 4. 總結：DBA 問「加在哪」的三層含義

| 層級 | 問題 | PG 現實 | 方案 |
|------|------|---------|------|
| 邏輯層 | 欄位順序不好看 | ADD COLUMN 永遠在末尾，`pg_attribute` 定義順序不可移動 | Simple View 虛擬重排（短期，代價為 DDL 傳染） |
| 物理層 | 能否真的在中间插 column？ | 不能。唯一方法是 `CREATE TABLE new AS SELECT` 重建整表 | 重建表 OR 接受物理順序（讓應用層按名引用欄位而非 `SELECT *` 靠位置） |
| 效能層 | 現有順序浪費 IO/CPU？ | 差順序 → byte alignment padding → 每 row 膨脹 30-50% | 重建表按 alignment-optimal 順序排列 column |

德哥原結論：「万不得已，也不要这么用。除非业务上不想改SQL。」——對 View 方案的總結。但 DBA 問的根本不只是順序，而是**byte alignment 對全鏈路效能的影響**。

---

## 參考

1. [阿里雲 RDS PostgreSQL 最佳實踐 — 表字段順序](https://yq.aliyun.com/articles/7176)
2. [PostgreSQL Documentation — Simple View / Auto-updatable Views](https://www.postgresql.org/docs/current/sql-createview.html)
3. [PostgreSQL Source — src/include/access/tupmacs.h (MAXALIGN, att_align)](https://github.com/postgres/postgres/blob/master/src/include/access/tupmacs.h)
