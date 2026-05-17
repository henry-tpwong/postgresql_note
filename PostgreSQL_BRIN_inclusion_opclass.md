# PostgreSQL BRIN Inclusion Opclass — R-Tree 風格索引策略

> 來源：[digoal - PostgreSQL 9.5 new feature: BRIN with R-Tree-like indexing strategies (2015-05-26)](https://github.com/digoal/blog/blob/master/201505/20150526_01.md)

---

## 背景與原理

PostgreSQL 9.5 新增了 BRIN（Block Range INdex）訪問方法，儲存每個連續 data page block 的邊界值信息：`min(val)`, `max(val)`, `has null?`, `all null?`。非常適合流式數據的自增列（例如 sequence、timestamp）索引，index 可以做到很小且定位精準。

PG 9.5 在此基礎上進一步新增了類似 R-Tree 的索引策略，稱為 **inclusion opclass**，讓 BRIN 不再局限於純量（scalar）的 min/max 比對，而是支援包含、相交、在旁等空間關係 operator。

當時實現的類型：
- `cidr` / `inet`
- `range`（anyrange）
- `box`（`point` 的 opclass 因浮點數比較一致性问题被移除）

未來還會添加更多類型的 operator，PostGIS 中的 geometry 類型也可能實現 BRIN。

官方 commit message：

> Add BRIN infrastructure for "inclusion" opclasses. This lets BRIN be used with R-Tree-like indexing strategies. Also provided are operator classes for range types, box and inet/cidr. The infrastructure provided here should be sufficient to create operator classes for similar datatypes; for instance, opclasses for PostGIS geometries should be doable, though we didn't try to implement one.
>
> Author: Emre Hasegeli. Review: Andreas Karlsson

> 補充（Senior Dev）：Inclusion opclass 的本質是將每個 page range 的 min/max 概念從「標量值」擴展到「包含範圍 / 邊界框」。對 box 類型來說，summary 存的是整個 page range 內所有 box 的最小 bounding box（union box）；對 range 類型則是 union range。這使得 BRIN 在處理「是否有任何 row 的 IP 範圍包含此 IP」或「是否有 box 與此區域相交」這類查詢時能以極小 index 實現有效過濾。但跟 scalar BRIN 一樣，前提是物理存儲順序與查詢 pattern 有關聯（correlation）。例如 IP 數據按 `inet` 排序存儲，BRIN 效果最好。
>
> 注意（版本更新）：本文寫於 PG 9.5 時期。在後續版本中，BRIN opclass 持續擴展——PG 10 加入了 `minmax-multi` opclass（每個 page range 存多組 min/max 而非一組），PG 11 加入 `bloom` opclass，PG 14 可對 BRIN 指定多個 column 各自不同的 opclass。PostGIS 也從某個版本開始正式支援 BRIN（`geography` / `geometry` 類型的 `brin_geometry_inclusion_ops` 等）。

---

## BRIN 支援的完整 Operator 列表

以下查詢列出當時 PG 9.5 BRIN 支援的所有 operator（共 237 行）：

```sql
SELECT oprname, gettypname(oprleft), gettypname(oprright)
FROM pg_operator
WHERE oid IN (
    SELECT amopopr FROM pg_amop
    WHERE amopmethod = (SELECT oid FROM pg_am WHERE amname = 'brin')
);

CREATE OR REPLACE FUNCTION gettypname(oid) RETURNS name AS $$
  SELECT typname FROM pg_type WHERE oid = $1;
$$ LANGUAGE sql STRICT;
```

### 純量類型（B-tree 風格 min/max operator）

| 類型 | Operator |
|------|----------|
| integer（int2 / int4 / int8） | `=` `<` `>` `<=` `>=`（含跨類型，如 int4 = int8） |
| char / name / text / bpchar | `=` `<` `>` `<=` `>=` |
| float4 / float8 / numeric | `=` `<` `>` `<=` `>=`（含跨類型） |
| date / time / timetz / timestamp / timestamptz | `=` `<` `>` `<=` `>=`（含跨類型如 date = timestamp） |
| interval / abstime / reltime / oid / tid / macaddr / uuid / pg_lsn | `=` `<` `>` `<=` `>=` |
| bit / varbit / bytea | `=` `<` `>` `<=` `>=` |

### box 類型（空間 operator）

| Operator | 含義 |
|----------|------|
| `<<` | 嚴格在左 (strictly left of) |
| `&<` | 不向右延伸 (does not extend to the right of) |
| `&>` | 不向左延伸 (does not extend to the left of) |
| `>>` | 嚴格在右 (strictly right of) |
| `<@` | 被包含 (contained in or on) |
| `@>` | 包含 (contains)，含 `@> point` |
| `~=` | 相同 (same as) |
| `&&` | 相交 (overlaps, one point in common makes this true) |
| `<<|` | 嚴格在下 (is strictly below) |
| `&<|` | 不向上延伸 (does not extend above) |
| `|&>` | 不向下延伸 (does not extend below) |
| `|>>` | 嚴格在上 (is strictly above) |

### inet / cidr 類型（網路 operator）

| Operator | 含義 |
|----------|------|
| `=` `<` `<=` `>` `>=` | 標準比較 |
| `<<` | 被包含 (is contained by) |
| `<<=` | 被包含或等於 (is contained by or equals) |
| `>>` | 包含 (contains) |
| `>>=` | 包含或等於 (contains or equals) |
| `&&` | 包含或被包含 (contains or is contained by) |

### anyrange 類型（範圍 operator）

| Operator | 含義 |
|----------|------|
| `=` `<` `<=` `>` `>=` | 標準比較 |
| `&&` | 重疊 (overlap, have points in common) |
| `@>` | 包含 (contains range / contains element) |
| `<@` | 被包含 (range is contained by) |
| `<<` | 嚴格在左 (strictly left of) |
| `>>` | 嚴格在右 (strictly right of) |
| `&<` | 不向右延伸 (does not extend to the right of) |
| `&>` | 不向左延伸 (does not extend to the left of) |
| `-|-` | 相鄰 (is adjacent to) |

> 補充（Senior Dev）：BRIN 的 spatial/inclusion operator 與對應的 GiST index operator 在語義上完全一致（如 `&&` 對 box 的含義相同），這意味著你可以通過簡單的 `USING BRIN` 替代 `USING GiST` 來實驗空間索引大小與查詢性能的 trade-off。在實際應用中：
> - 對於 IP 地理位置庫（如 GeoIP）這種百萬到千萬級的 static / append-only 表，BRIN on inet 可以將 GiST index 從數 GB 壓到數 MB，查詢從「幾百 ms + random I/O」變成「幾十 ms + 順序 read + 少量 Recheck」
> - 對於 range 類型的 BRIN（如預約系統的時間段 overlap 查詢），效果取決於 range column 的物理排序程度——建議按 `lower(range_col)` 或 `upper(range_col)` cluster 表後再建 BRIN
> - BRIN inclusion 的 `pages_per_range` 權衡與 scalar BRIN 一致：值越小 → summary 越精細 → 誤報越少，但 index 越大

## 參考

1. [Git commit: BRIN inclusion opclass infrastructure](http://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=b0b7be61337fc64147f2ad0af5bf2c0e6b8a709f)
2. [PostgreSQL Official Docs: Functions and Operators](http://www.postgresql.org/docs/devel/static/functions.html)
3. [德哥：BRIN 索引基礎文章](http://blog.163.com/digoal@126/blog/static/163877040201531931956500/)
