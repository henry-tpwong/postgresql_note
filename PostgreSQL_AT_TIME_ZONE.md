# PostgreSQL AT TIME ZONE 與 EXTRACT 的語法解析

> 來源：[digoal - PostgreSQL timestamp parse in gram.y (' ' AT TIME ZONE ' ') (2015-04-30)](https://github.com/digoal/blog/blob/master/201504/20150430_01.md)

---

## 問題：EXTRACT epoch 在不同時區下的意外行為

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

## 語法解析：EXTRACT 和 AT TIME ZONE 的底層函數

### EXTRACT → date_part

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

### AT TIME ZONE → timezone

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

## 逐步分解

### Case 1：EXTRACT epoch FROM 'today'::timestamptz

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

### Case 2：EXTRACT epoch FROM 'today' AT TIME ZONE '0'

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

### Case 3：EXTRACT epoch FROM 'today' AT TIME ZONE '8'

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

## 底層函數對照

| 語法 | 解析結果 | 底層函數（src/backend/utils/adt/timestamp.c） |
|------|---------|----------------------------------------------|
| `EXTRACT(epoch FROM timestamptz)` | `date_part('epoch', timestamptz)` | `timestamptz_part()` — stable，依賴 session timezone |
| `EXTRACT(epoch FROM timestamp)` | `date_part('epoch', timestamp)` | `timestamp_part()` — immutable，視為 UTC |
| `timestamptz AT TIME ZONE zone` | `timezone(zone, timestamptz)` | `timestamptz_zone()` / `timestamptz_izone()` — 返回 `timestamp` |
| `timestamp AT TIME ZONE zone` | `timezone(zone, timestamp)` | `timestamp_zone()` / `timestamp_izone()` — 返回 `timestamptz` |

---

## 源碼參考

- `src/backend/parser/gram.y` — `EXTRACT` 語法規則與 `AT TIME ZONE` 語法規則
- `src/backend/utils/adt/timestamp.c` — `timestamptz_part()`、`timestamp_part()`、`timestamptz_zone()`

> [PG 版本註] 原文基於 PG 9.x（2015）。核心行為（`AT TIME ZONE` 的型別轉換、`date_part` 的 overload 選擇邏輯）在最新版本（PG 17+）完全一致，未變更。唯一變化：PG 12+ 新增 `date_trunc` 的 timezone 參數支援（`date_trunc('day', timestamptz, 'Asia/Shanghai')`），但與 `EXTRACT` / `AT TIME ZONE` 無直接關聯。
