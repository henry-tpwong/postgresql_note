# PostgreSQL Float vs Numeric 性能對比

> 來源：[digoal - float和numeric性能对比 (2015-10-20)](https://github.com/digoal/blog/blob/master/201510/20151020_02.md)
>
> 更新於 2026-05-17，補充 JIT / SIMD 演進

---

## 測試環境

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

## Benchmark 結果

### 基本算術運算（+, *, /, -）

**NUMERIC（8 concurrent, 10 秒）：**

```
tps: ~35,028  (latency 0.227ms)
```

**FLOAT（8 concurrent, 10 秒）：**

```
tps: ~34,729  (latency 0.229ms)
```

小數值（4 位小數）的 basic arithmetic 兩者近乎持平，numeric 甚至略快。

### 數學函數（sqrt + cbrt + arithmetic）

**NUMERIC（8 concurrent, 6 秒）：**

```
tps: ~29,667  (latency 0.268ms)
```

**FLOAT（8 concurrent, 6 秒）：**

```
tps: ~30,528  (latency 0.261ms)
```

引入 `sqrt` / `cbrt`（`|/c1`, `||/c1`）後，float 開始輕微領先。

### Pi 計算（遞歸逼近，70 次迭代）

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

## 性能差異的根本原因

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

## Float 的額外優勢：SIMD 向量化

固定長度類型（`float4`, `float8`, `int4`, `int8`）支援 CPU SIMD 指令（SSE / AVX / AVX-512），可在一條 CPU 指令中同時處理多個數值。在海量數據分析（OLAP）場景中，這是巨量加速的來源。NUMERIC 因變長結構無法受益於 SIMD。

---

## 選擇建議

| 場景 | 推薦 | 原因 |
|------|------|------|
| 金融、會計（需精確小數） | NUMERIC | 任意精度，無 rounding error |
| 科學計算、統計 | FLOAT | 硬體加速，效能碾壓 |
| 高 TPS OLTP（精度 < 10 digits） | NUMERIC / FLOAT 均可 | 差異可忽略 |
| OLAP / 大規模聚合 | FLOAT | SIMD 向量化優勢 |
| 需要精準等值比對 | NUMERIC | float rounding error 會導致 `WHERE a = b` 誤判 |

---

## 版本演進

| 功能 | 版本 | 說明 |
|------|------|------|
| LLVM JIT compilation | PG 11 | 加速 WHERE / expression evaluation，float/numeric 均受益 |
| JIT for tuple deforming | PG 14 | 加速 column value 提取 |
| SIMD 優化（pg_lzcompress 等） | PG 16+ | PostgreSQL 逐步引入 SIMD 加速內部模塊 |
