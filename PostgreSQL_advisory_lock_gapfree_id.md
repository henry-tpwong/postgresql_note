# Advisory Lock 實現無縫自增 ID

> 來源：[digoal - PostgreSQL 无缝自增ID的实现 - by advisory lock (2016-10-20)](https://github.com/digoal/blog/blob/master/201610/20161020_02.md)

---

## 背景：SEQUENCE 的空洞問題

PostgreSQL 的 `SERIAL` / `SEQUENCE` 提供遞增數值，但有一個本質特性：**已消耗的值不會回收**。如果 transaction rollback，序列值已被取走，產生空洞。

```sql
CREATE TABLE seq_test(id SERIAL, info text);

INSERT INTO seq_test (info) VALUES ('test');  -- id=1

BEGIN;
INSERT INTO seq_test (info) VALUES ('test');  -- id=2
ROLLBACK;                                      -- id=2 被消耗，永不復用

INSERT INTO seq_test (info) VALUES ('test');  -- id=3
SELECT * FROM seq_test;
--  id | info
-- ----+------
--   1 | test
--   3 | test
-- (2 rows)  ← id=2 永久消失
```

這是 SEQUENCE 的設計取捨：為效能最大化（無 lock contention），犧牲連續性。`nextval()` 是非交易性的（不可 rollback），確保高並發下不會互相阻塞。

> 補充（Senior Dev）：Oracle 的 SEQUENCE 行為與 PG 完全一致；MySQL InnoDB 的 `AUTO_INCREMENT` 在 rollback 時同樣不回補。空洞是關聯式資料庫的普遍行為，不是 bug。

---

## 何時需要無縫 ID？

| 場景 | 是否需要無縫 | 原因 |
|------|------------|------|
| 發票號碼 (invoice number) | **是** | 法規 / 審計要求連續 |
| 銀行流水號 | **是** | 監管要求不可跳號 |
| PK (Primary Key) | 否 | 空洞不影響功能 |
| 對外 API 的 ID | 否 | 用 UUID 更好 |
| Log / audit trail ID | 否 | 時間戳 + 序號即可 |

> 補充（Senior Dev）：即使業務需要無縫 ID，也應分離 PK 與業務 ID。PK 用 `SERIAL` / `BIGSERIAL` 確保寫入效能；業務 ID 用本文方案單獨生成。兩者解耦後，PK 的效能不受業務 ID 序列化瓶頸影響。

---

## Advisory Lock 機制

PostgreSQL 的 Advisory Lock 是 application-level lock，不與 table/row 綁定，只認一個 `bigint key`（或兩個 `int key`）。

完整函數列表（來源：[官方文檔 9.6](https://www.postgresql.org/docs/9.6/static/functions-admin.html#FUNCTIONS-ADVISORY-LOCKS)）：

### Session-Level Lock（會話鎖）

| 函數 | 說明 |
|------|------|
| `pg_advisory_lock(key)` | 獲取排他 session lock（blocking） |
| `pg_advisory_lock_shared(key)` | 獲取共享 session lock（blocking） |
| `pg_try_advisory_lock(key)` | 嘗試獲取排他 session lock（non-blocking，returns bool） |
| `pg_try_advisory_lock_shared(key)` | 嘗試獲取共享 session lock（non-blocking） |
| `pg_advisory_unlock(key)` | 釋放排他 session lock |
| `pg_advisory_unlock_shared(key)` | 釋放共享 session lock |
| `pg_advisory_unlock_all()` | 釋放當前 session 的所有 advisory lock |

### Transaction-Level Lock（交易鎖）

| 函數 | 說明 |
|------|------|
| `pg_advisory_xact_lock(key)` | 獲取排他 transaction lock（blocking，transaction 結束時自動釋放） |
| `pg_advisory_xact_lock_shared(key)` | 獲取共享 transaction lock（blocking） |
| `pg_try_advisory_xact_lock(key)` | 嘗試獲取排他 transaction lock（non-blocking） |
| `pg_try_advisory_xact_lock_shared(key)` | 嘗試獲取共享 transaction lock（non-blocking） |

關鍵區別：
- **Session lock**：需手動釋放（`pg_advisory_unlock`），或 session 斷開時自動釋放。適合跨多個 transaction 的協調。
- **Transaction lock**：transaction commit/rollback 時自動釋放，不需手動管理。適合本文場景。

---

## 實作：Advisory Lock 無縫 ID Generator

以 `id` 值本身作為 advisory lock key——若 `pg_try_advisory_xact_lock(N)` 返回 `true`，表示當前的 transaction 獨佔了 ID=N，可以安全插入。

### Function 邏輯

```
LOOP:
  1. SELECT MAX(id) + 1 → 得出下一個可用 ID
  2. pg_try_advisory_xact_lock(newid)
     - 成功 → INSERT → RETURN newid
     - 失敗（被其他 session 搶先）→ CONTINUE loop
  3. PK 衝突（極端競爭）→ 遞迴重試
```

### SQL 實作

```sql
CREATE TABLE uniq_test (id int PRIMARY KEY, info text);

CREATE OR REPLACE FUNCTION f_uniq(i_info text) RETURNS int AS $$
DECLARE
  newid int;
  i int := 0;
  res int;
BEGIN
  LOOP
    IF i > 0 THEN
      PERFORM pg_sleep(0.2 * random());  -- 衝突時隨機退縮，降低競爭
    ELSE
      i := i + 1;
    END IF;

    -- 獲取當前最大 ID + 1
    SELECT max(id) + 1 INTO newid FROM uniq_test;

    IF newid IS NOT NULL THEN
      -- 嘗試獲取 transaction-level advisory lock（key = newid）
      IF pg_try_advisory_xact_lock(newid) THEN
        INSERT INTO uniq_test (id, info) VALUES (newid, i_info);
        RETURN newid;
      ELSE
        CONTINUE;  -- 鎖被搶走，重試下一個 ID
      END IF;
    ELSE
      -- 第一條記錄的特殊處理（max(id) 為 NULL）
      IF pg_try_advisory_xact_lock(1) THEN
        INSERT INTO uniq_test (id, info) VALUES (1, i_info);
        RETURN 1;
      ELSE
        CONTINUE;
      END IF;
    END IF;
  END LOOP;

  -- PK 衝突（極端競爭下的 race condition）→ 遞迴重試
  EXCEPTION WHEN OTHERS THEN
    SELECT f_uniq(i_info) INTO res;
    RETURN res;
END;
$$ LANGUAGE plpgsql STRICT;
```

### 關鍵設計點

| 設計 | 原因 |
|------|------|
| `pg_try_advisory_xact_lock`（非 blocking） | 避免 deadlock；失敗時 loop 重試而不用等待 |
| `_xact_`（transaction-level） | commit 自動釋放，不需手動 unlock |
| `pg_sleep(0.2 * random())` | 退縮策略（backoff），避免多 session 在競爭窗口內同步重試 |
| `EXCEPTION WHEN OTHERS` | 處理 race condition：兩個 session 可能同時取得不同 ID 但 PK 衝突（`max(id)` 返回瞬態值），遞迴重試即可 |

---

## 效能實測

```sql
-- test.sql
SELECT f_uniq('test');
```

```
pgbench -M prepared -n -r -P 1 -f ./test.sql -c 164 -j 164 -T 10
```

| 指標 | 數值 |
|------|------|
| 並發數 | 164 clients |
| TPS | **11,730** |
| Avg Latency | 13.79 ms |
| 錯誤率 | 0 |

```
postgres=# SELECT count(*), max(id) FROM uniq_test;
 count  |  max
--------+--------
 119565 | 119565   ← count = max，完美無空洞
```

### 與原生 SEQUENCE 的對比

> 補充（Senior Dev）：

| 維度 | SERIAL / SEQUENCE | Advisory Lock 方案 |
|------|-------------------|-------------------|
| TPS (單表) | ~50,000-100,000+ | ~12,000 |
| 空洞 | 有（rollback 產生） | 無 |
| Lock contention | 極低 (`nextval()` 幾乎無競爭） | 高（每個 ID 需 `pg_try_advisory_xact_lock`，存在搶鎖 loop） |
| `SELECT max(id)` | 不需要 | 每次都要（全表 Seq Scan 或 Index Only Scan），隨 table 增大而變慢 |
| 適用場景 | 99% 的 OLTP | 業務 ID（invoice no. 等） |

效能差距的核心：原生 SEQUENCE 使用 shared memory counter，`nextval()` 只需 atomic increment + WAL 記錄；Advisory Lock 方案需要 `max(id)` scan + lock acquisition loop + potential retry。

---

## 優化方向與生產注意事項

> 補充（Senior Dev）：

### `SELECT max(id)` 的瓶頸

隨著 table 增大，`max(id)` 需要 Index Only Scan on PK（或 Seq Scan），每次調用都掃一次。解法：

```sql
-- 在另一張 counter table 中維護 current max
CREATE TABLE id_counter (current_max int);
INSERT INTO id_counter VALUES (0);

-- 替代 SELECT max(id)：用 UPDATE ... RETURNING 原子遞增
CREATE OR REPLACE FUNCTION f_uniq_v2(i_info text) RETURNS int AS $$
DECLARE
  newid int;
BEGIN
  UPDATE id_counter SET current_max = current_max + 1
    RETURNING current_max INTO newid;
  INSERT INTO uniq_test (id, info) VALUES (newid, i_info);
  RETURN newid;
END;
$$ LANGUAGE plpgsql;
```

但這個方案在 `UPDATE id_counter` 時變成 single-row hotspot（所有 session 排隊等同一行 lock），TPS 會驟降到 ~2,000。因此需按場景取捨。

### PG 10+ IDENTITY Column

PG 10 引入了 SQL 標準的 `GENERATED AS IDENTITY`，行為等同 `SERIAL`，同樣會有空洞：

```sql
CREATE TABLE t (
    id int GENERATED ALWAYS AS IDENTITY,
    info text
);
```

```sql
-- PG 10+
CREATE TABLE t (
    id int GENERATED BY DEFAULT AS IDENTITY,
    info text
);
```

這只是語法糖，底層仍使用 SEQUENCE。無縫需求仍需本文方案。

### 分離 PK 與業務 ID（推薦生產實踐）

```sql
CREATE TABLE invoices (
    pk_id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,  -- 內部 PK，用於 FK / join
    invoice_no int NOT NULL UNIQUE,                          -- 業務 ID，本文方案生成
    amount numeric,
    issued_at timestamptz DEFAULT now()
);

-- 業務方只查 invoice_no，內部 join 用 pk_id
```

### Advisory Lock 的系統限制

- Lock 數量無上限（不佔 `max_locks_per_transaction` slot，因為 advisory lock 使用獨立的 shared memory hash table）
- `pg_advisory_unlock_all()` 只在 session 結束時自動調用；`pg_advisory_xact_lock` 在 transaction 結束時自動釋放
- PG 9.6+ 可透過 `pg_locks` 查看 advisory lock 狀態：`SELECT * FROM pg_locks WHERE locktype = 'advisory'`
- 兩個 `int` key 版本（`pg_try_advisory_xact_lock(key1, key2)`）可用於多維度協調（如 `(table_oid, id)` 避免不同 table 共用相同 key 的 ID 衝突）

---

## 小結

| 觀點 | 說明 |
|------|------|
| 原生 SEQUENCE 空洞是設計取捨 | 為效能放棄連續性，99% 場景不需要無縫 |
| Advisory Lock 是輕量級的序列化工具 | transaction-level 不需手動釋放，non-blocking try 模式避免 deadlock |
| 效能差距顯著 | ~12K TPS vs ~50K+ TPS，取決於並發度與硬體 |
| 生產最佳實踐 | PK 用 SERIAL，業務 ID 用本文方案分開管理 |
| 替代方案 | counter table（更慢但更簡單）、外部 ID service（Redis INCR、Snowflake）、UUID v7（sortable、no gap concern） |

---

## 源碼與參考

1. [Advisory Lock 官方文檔](https://www.postgresql.org/docs/9.6/static/functions-admin.html#FUNCTIONS-ADVISORY-LOCKS)
2. `src/backend/storage/lmgr/lock.c` — lock manager 實作（advisory lock 走同一條 `LockAcquire` 路徑）
3. [PG 10 IDENTITY Column 文檔](https://www.postgresql.org/docs/10/sql-createtable.html)

> [PG 版本註] 原文基於 PG 9.6（2016）。`advisory lock` API 在所有後續版本（PG 10~17+）中完全向後相容，無任何 breaking change。PG 10+ 新增 `GENERATED AS IDENTITY` 語法，但對無縫 ID 需求無影響。PG 14+ 引入了 pipeline mode（`libpq` batch operation），可配合 advisory lock 減少 round-trip overhead。
