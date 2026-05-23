# PostgreSQL 資料型別深度探討

本文由淺入深，依序探討兩個 PostgreSQL 資料型別相關主題：

1. **第一章「Float vs Numeric 性能對比」**：從實戰角度出發，比較兩種數值型別的效能差異、底層原因與選型建議。適合快速了解何時該用哪種型別。
2. **第二章「AT TIME ZONE 語法解析」**：深入 parser 內部（gram.y）、型別系統與函數 overload 選擇邏輯，用一個違反直覺的案例逐步拆解 `EXTRACT epoch` 在不同時區下的行為。適合想理解 PostgreSQL 型別系統底層運作的讀者。

建議依序閱讀：先建立對數值型別的實戰認識，再進入時間型別的 parser 層級分析。

---

# 一、Float vs Numeric 性能對比

> 來源：[digoal - float和numeric性能对比 (2015-10-20)](https://github.com/digoal/blog/blob/master/201510/20151020_02.md)
>
> 更新於 2026-05-17，補充 JIT / SIMD 演進

---

## 1. 測試環境

```sql
CREATE TABLE tt (c1 NUMERIC, c2 NUMERIC);
ALTER TABLE tt ALTER COLUMN c1 SET STORAGE PLAIN;  -- 避免 TOAST overhead
ALTER TABLE tt ALTER COLUMN c2 SET STORAGE PLAIN;
INSERT INTO tt VALUES (1.1111, 1.1111);

CREATE TABLE tf (c1 FLOAT, c2 FLOAT);
INSERT INTO tf VALUES (1.1111, 1.1111);
```

`SET STORAGE PLAIN` 確保 numeric 不經過 TOAST 壓縮/外部存儲，消除 storage overhead 對 benchmark 的干擾。

---

## 2. Benchmark 結果

### I. 基本算術運算（+, *, /, -）

**NUMERIC（8 concurrent, 10 秒）：**

```
tps: ~35,028  (latency 0.227ms)
```

**FLOAT（8 concurrent, 10 秒）：**

```
tps: ~34,729  (latency 0.229ms)
```

小數值（4 位小數）的 basic arithmetic 兩者近乎持平，numeric 甚至略快。

### II. 數學函數（sqrt + cbrt + arithmetic）

**NUMERIC（8 concurrent, 6 秒）：**

```
tps: ~29,667  (latency 0.268ms)
```

**FLOAT（8 concurrent, 6 秒）：**

```
tps: ~30,528  (latency 0.261ms)
```

引入 `sqrt` / `cbrt`（`|/c1`, `||/c1`）後，float 開始輕微領先。

### III. Pi 計算（遞歸逼近，70 次迭代）

```sql
-- NUMERIC: 完整精度
WITH RECURSIVE pi(lv, c) AS (
  SELECT 1::NUMERIC lv, 1::NUMERIC c
  UNION ALL
  SELECT lv + 1,
         SQRT((c/2)*(c/2) + (1-SQRT(1-(c/2)*(c/2)))*(1-SQRT(1-(c/2)*(c/2)))) c
  FROM pi WHERE lv < 70
)
SELECT 3 * POWER(2, lv) * c / 2 p FROM pi WHERE lv = 70;

-- Result: 3.14159265358979323846264338327950288419717321...
--          ...(數千位精度，完整展開)
-- Time: 513.449 ms
```

```sql
-- FLOAT: 僅 double precision
WITH RECURSIVE pi(lv, c) AS (
  SELECT 1::FLOAT lv, 1::FLOAT c
  UNION ALL
  SELECT lv + 1,
         SQRT((c/2)*(c/2) + (1-SQRT(1-(c/2)*(c/2)))*(1-SQRT(1-(c/2)*(c/2)))) c
  FROM pi WHERE lv < 70
)
SELECT 3 * POWER(2, lv) * c / 2 p FROM pi WHERE lv = 70;

-- Result: 3.14159265358979
-- Time: 1.431 ms
```

**NUMERIC 513ms vs FLOAT 1.4ms = ~360x 差距。**

---

## 3. 性能差異的根本原因

| 維度 | FLOAT (float8) | NUMERIC |
|------|---------------|---------|
| 存儲 | 固定 8 bytes（IEEE 754） | 變長（每 4 digits 佔 2 bytes + overhead） |
| CPU 指令 | 直接用 FPU / SSE / AVX 指令 | 軟體實現（C 函數逐 digit 運算） |
| CPU 向量化 | 支援 SIMD（MMX/SSE/AVX/AVX-512） | 不支援 |
| 精度 | 15-17 decimal digits | 任意精度（受 memory 限制） |
| 範圍 | ~1e-308 ~ 1e+308 | 不限（可達 131072 digits before decimal, 16383 after） |

**核心差異**：float 是 CPU native type，硬體直接運算。numeric 是軟體模擬的任意精度運算——每次 arithmetic 都要遍歷 struct 中的 digit array，無硬體加速。

小數值（4 位）時 numeric struct overhead 很小，arithmetic 與 float 持平。一旦涉及 `sqrt` 等迭代逼近函數，軟體實現的代價急劇放大（360x）。

> 補充（Senior Dev）：PG 16+ 的實測中，`double precision` 在 `pgbench` 單行 benchmark 下 numeric vs float 差距取決於 digit 數量。當 numeric precision > 20 digits 時，差距開始顯著。典型 OLTP 場景（金額計算，精度 < 10 digits）兩者差異可忽略。PG 11+ LLVM JIT compilation 對兩者都有提速，但對 numeric 的 `sqrt` 等內部迴圈加速有限（因核心在 C library 層，非 JIT 範圍）。

---

## 4. Float 的額外優勢：SIMD 向量化

固定長度類型（`float4`, `float8`, `int4`, `int8`）支援 CPU SIMD 指令（SSE / AVX / AVX-512），可在一條 CPU 指令中同時處理多個數值。在海量數據分析（OLAP）場景中，這是巨量加速的來源。NUMERIC 因變長結構無法受益於 SIMD。

---

## 5. 選擇建議

| 場景 | 推薦 | 原因 |
|------|------|------|
| 金融、會計（需精確小數） | NUMERIC | 任意精度，無 rounding error |
| 科學計算、統計 | FLOAT | 硬體加速，效能碾壓 |
| 高 TPS OLTP（精度 < 10 digits） | NUMERIC / FLOAT 均可 | 差異可忽略 |
| OLAP / 大規模聚合 | FLOAT | SIMD 向量化優勢 |
| 需要精準等值比對 | NUMERIC | float rounding error 會導致 `WHERE a = b` 誤判 |

---

## 6. 版本演進

| 功能 | 版本 | 說明 |
|------|------|------|
| LLVM JIT compilation | PG 11 | 加速 WHERE / expression evaluation，float/numeric 均受益 |
| JIT for tuple deforming | PG 14 | 加速 column value 提取 |
| SIMD 優化（pg_lzcompress 等） | PG 16+ | PostgreSQL 逐步引入 SIMD 加速內部模塊 |

---

# 二、AT TIME ZONE 與 EXTRACT 的語法解析

> 來源：[digoal - PostgreSQL timestamp parse in gram.y (' ' AT TIME ZONE ' ') (2015-04-30)](https://github.com/digoal/blog/blob/master/201504/20150430_01.md)

---

## 1. 問題：EXTRACT epoch 在不同時區下的意外行為

當前 session timezone 為 `PRC`（即 `Asia/Shanghai`，UTC+8）：

```
postgres=# show timezone;
 TimeZone
----------
 PRC
(1 row)
```

以下兩個 epoch 結果完全相同，違反直覺：

```
postgres=# select extract(epoch from 'today'::timestamptz);
 date_part
------------
 1430323200
(1 row)

postgres=# select extract(epoch from 'today' at time zone '0');
 date_part
------------
 1430323200
(1 row)
```

直覺上 `AT TIME ZONE '0'` 應該把時間轉換到 UTC+0 再取 epoch，結果應該不同，但這裡卻一樣。而指定 UTC+8 反而結果不同：

```
postgres=# select extract(epoch from 'today' at time zone '8');
 date_part
------------
 1430294400
(1 row)
```

---

## 2. 語法解析：EXTRACT 和 AT TIME ZONE 的底層函數

### I. EXTRACT → date_part

`EXTRACT(field FROM expr)` 在 parser（`src/backend/parser/gram.y`）中解析為 `date_part` 函數調用：

```
| EXTRACT '(' extract_list ')'
    {
        $$ = (Node *) makeFuncCall(SystemFuncName("date_part"), $3, @1);
    }
```

`date_part` 的 overload 列表（`\df+ date_part`）：

| Argument Type | Volatility |
|---------------|------------|
| `text, timestamp with time zone` | stable |
| `text, timestamp without time zone` | immutable |
| `text, date` | immutable |
| `text, time with time zone` | immutable |
| `text, time without time zone` | immutable |
| `text, interval` | immutable |
| `text, abstime` | stable |
| `text, reltime` | stable |

關鍵區別：`date_part(text, timestamptz)` 是 **stable**（依賴 session timezone），`date_part(text, timestamp)` 是 **immutable**（純數學計算，不依賴 timezone）。

### II. AT TIME ZONE → timezone

`expr AT TIME ZONE zone` 在 parser 中解析為 `timezone` 函數調用：

```
| a_expr AT TIME ZONE a_expr %prec AT
    {
        $$ = (Node *) makeFuncCall(SystemFuncName("timezone"),
                                    list_make2($5, $1), @2);
    }
```

注意參數順序：`timezone(zone, expr)` —— `AT TIME ZONE` 右側的 zone 在前。

`timezone` 的 overload 列表（`\df+ timezone`）：

| Result Type | Argument Types |
|-------------|---------------|
| `timestamp without time zone` | `interval, timestamp with time zone` |
| `timestamp with time zone` | `interval, timestamp without time zone` |
| `timestamp without time zone` | `text, timestamp with time zone` |
| `timestamp with time zone` | `text, timestamp without time zone` |
| `time with time zone` | `interval, time with time zone` |
| `time with time zone` | `text, time with time zone` |

---

## 3. 逐步分解

### I. Case 1：EXTRACT epoch FROM 'today'::timestamptz

```
postgres=# select
    extract(epoch from 'today'::timestamptz),
    date_part('epoch', 'today'::timestamptz),
    'today'::timestamptz;

 date_part  | date_part  |      timestamptz
------------+------------+------------------------
 1430323200 | 1430323200 | 2015-04-30 00:00:00+08
```

- `'today'::timestamptz` 在 PRC 時區被解析為 `2015-04-30 00:00:00+08`
- 調用 `timestamptz_part(text, timestamptz)`（source: `src/backend/utils/adt/timestamp.c`）
- epoch 結果 = UTC timestamp 對應的秒數

```
postgres=# select pg_typeof('today'::timestamptz);
        pg_typeof
--------------------------
 timestamp with time zone
```

### II. Case 2：EXTRACT epoch FROM 'today' AT TIME ZONE '0'

```
postgres=# select
    extract(epoch from 'today' at time zone '0'),
    date_part('epoch', timezone('0', 'today'::timestamptz)),
    timezone('0', 'today'::timestamptz),
    timezone('0', 'today');

 date_part  | date_part  |      timezone       |      timezone
------------+------------+---------------------+---------------------
 1430323200 | 1430323200 | 2015-04-29 16:00:00 | 2015-04-29 16:00:00
(1 row)
```

步驟：
1. `timezone('0', 'today'::timestamptz)` 將 `2015-04-30 00:00:00+08` 轉換為 UTC+0 → `2015-04-29 16:00:00`
2. 返回類型是 `timestamp without time zone`：

```
postgres=# select pg_typeof(timezone('0', 'today'));
          pg_typeof
-----------------------------
 timestamp without time zone
```

3. 此時調用的是 `timestamp_part(text, timestamp)`（immutable 版本），直接把 `2015-04-29 16:00:00` 視為 UTC 時間計算 epoch
4. `2015-04-29 16:00:00 UTC` 的 epoch 等於 `2015-04-30 00:00:00+08` 的 epoch——兩者描述的是**同一個物理時刻**，所以結果一致

**結果一致的根源**：`timezone('0', timestamptz)` 做的是時區轉換（物理時刻不變），而 `date_part('epoch', timestamp)` 把沒有 timezone 的 timestamp 當成 UTC 計算。物理時刻相同 → epoch 相同。

### III. Case 3：EXTRACT epoch FROM 'today' AT TIME ZONE '8'

```
postgres=# select
    date_part('epoch', timezone('8', 'today')),
    timezone('8', 'today');

 date_part  |      timezone
------------+---------------------
 1430294400 | 2015-04-29 08:00:00
(1 row)
```

1. `timezone('8', 'today')` → 將 `'today'`（隱含在 PRC 時區）轉換為 UTC+8，得到 `2015-04-29 08:00:00`（note：當前 session timezone 是 PRC = UTC+8，所以 `'today'::timestamptz` 是 `2015-04-30 00:00:00+08`，但如果裸寫 `'today'` 不強制轉型，行為需視上下文而定）
2. `date_part('epoch', timestamp)` 將其當成 UTC 計算 → epoch 不同

> 補充（Senior Dev）：整個混淆的本質是 **timestamp with time zone 和 timestamp without time zone 之間的型別轉換鏈**。`AT TIME ZONE` operator 的行為取決於輸入類型——輸入是 `timestamptz` 則返回 `timestamp`（去掉 timezone，表示該時區的 wall-clock time）；輸入是 `timestamp` 則返回 `timestamptz`（將該 wall-clock time 解釋為指定時區，轉成 UTC）。而 `EXTRACT(epoch FROM ...)` 對 `timestamp` 的處理是「將該值視為 UTC」，對 `timestamptz` 則是「先轉 UTC 再計算」。所以當 `AT TIME ZONE '0'` 把 `timestamptz` 轉成 `timestamp` 後，兩者描述的是同一 UTC 時刻，epoch 自然相同。這不是 bug，而是型別系統的精確行為，但極易踩坑。

---

## 4. 底層函數對照

| 語法 | 解析結果 | 底層函數（src/backend/utils/adt/timestamp.c） |
|------|---------|----------------------------------------------|
| `EXTRACT(epoch FROM timestamptz)` | `date_part('epoch', timestamptz)` | `timestamptz_part()` — stable，依賴 session timezone |
| `EXTRACT(epoch FROM timestamp)` | `date_part('epoch', timestamp)` | `timestamp_part()` — immutable，視為 UTC |
| `timestamptz AT TIME ZONE zone` | `timezone(zone, timestamptz)` | `timestamptz_zone()` / `timestamptz_izone()` — 返回 `timestamp` |
| `timestamp AT TIME ZONE zone` | `timezone(zone, timestamp)` | `timestamp_zone()` / `timestamp_izone()` — 返回 `timestamptz` |

---

## 5. 源碼參考

- `src/backend/parser/gram.y` — `EXTRACT` 語法規則與 `AT TIME ZONE` 語法規則
- `src/backend/utils/adt/timestamp.c` — `timestamptz_part()`、`timestamp_part()`、`timestamptz_zone()`

> [PG 版本註] 原文基於 PG 9.x（2015）。核心行為（`AT TIME ZONE` 的型別轉換、`date_part` 的 overload 選擇邏輯）在最新版本（PG 17+）完全一致，未變更。唯一變化：PG 12+ 新增 `date_trunc` 的 timezone 參數支援（`date_trunc('day', timestamptz, 'Asia/Shanghai')`），但與 `EXTRACT` / `AT TIME ZONE` 無直接關聯。
