# PostgreSQL JSON / JSONB Value Types 與構造方法

> 來源：[digoal - PostgreSQL json jsonb 支持的value数据类型，如何构造一个jsonb (2015-09-24)](https://github.com/digoal/blog/blob/master/201509/20150924_03.md)

---

## 1. JSON 支援的 Scalar Types（區分大小寫）

PostgreSQL JSON 的 value 支援以下 5 種 scalar types，**嚴格區分大小寫**：

| Scalar Type | 合法寫法 | 非法寫法 |
|-------------|---------|---------|
| string | `"abc"` | — |
| number | `10.001` | — |
| boolean true | `true` | `TRUE`、`True` |
| boolean false | `false` | `FALSE`、`False` |
| null | `null` | `NULL`、`Null` |

實測：

```sql
SELECT jsonb '{"a": true}';       -- OK
SELECT jsonb '{"a": TRUE}';       -- ERROR: Token "TRUE" is invalid
SELECT jsonb '{"a": false}';      -- OK
SELECT jsonb '{"a": NULL}';       -- ERROR: Token "NULL" is invalid
SELECT jsonb '{"a": null}';       -- OK
SELECT jsonb '{"a": 10.001}';     -- OK
SELECT jsonb '{"a": "10.001"}';   -- OK (string, not number)
```

這源於 `src/backend/utils/adt/json.c` 中 `parse_scalar()` 的實作。Parser 對每個 token 做精確的 byte-level 匹配，`true` 必須是 4 bytes、`false` 必須是 5 bytes、`null` 必須是 4 bytes，且大小寫敏感：

```c
static inline void
parse_scalar(JsonLexContext *lex, JsonSemAction *sem)
{
    char       *val = NULL;
    json_scalar_action sfunc = sem->scalar;
    char      **valaddr;
    JsonTokenType tok = lex_peek(lex);

    valaddr = sfunc == NULL ? NULL : &val;

    /* a scalar must be a string, a number, true, false, or null */
    switch (tok)
    {
        case JSON_TOKEN_TRUE:
            lex_accept(lex, JSON_TOKEN_TRUE, valaddr);
            break;
        case JSON_TOKEN_FALSE:
            lex_accept(lex, JSON_TOKEN_FALSE, valaddr);
            break;
        case JSON_TOKEN_NULL:
            lex_accept(lex, JSON_TOKEN_NULL, valaddr);
            break;
        case JSON_TOKEN_NUMBER:
            lex_accept(lex, JSON_TOKEN_NUMBER, valaddr);
            break;
        case JSON_TOKEN_STRING:
            lex_accept(lex, JSON_TOKEN_STRING, valaddr);
            break;
        default:
            report_parse_error(JSON_PARSE_VALUE, lex);
    }
    if (sfunc != NULL)
        (*sfunc) (sem->semstate, val, tok);
}
```

Parser context 的 enum（只用於 parser 狀態管理，與 JSONB 內部存儲無關）：

```c
typedef enum  /* contexts of JSON parser */
{
    JSON_PARSE_VALUE,
    JSON_PARSE_STRING,
    JSON_PARSE_ARRAY_START,
    JSON_PARSE_ARRAY_NEXT,
    JSON_PARSE_OBJECT_START,
    JSON_PARSE_OBJECT_LABEL,
    JSON_PARSE_OBJECT_NEXT,
    JSON_PARSE_OBJECT_COMMA,
    JSON_PARSE_END
} JsonParseContext;
```

> 補充（Senior Dev）：JSONB 內部實際上是 binary format（`JsonbContainer` / `JsonbValue`），並非真正的純字串。文章說「JSONB 內部就是一個有一定規則的字串」是指 **從 text 解析為 jsonb 時** Postgres 只做 token 匹配，不保留原始 type metadata。例如 `10` 和 `10.0` 存入 jsonb 後都是 numeric scalar，無法區分原始是 integer 還是 float。這與 MongoDB BSON 保留 `int32` / `int64` / `double` 的設計不同。

---

## 2. jsonb 的內部類型本質

jsonb 內部**沒有**類型概念。Parser 看到的 TOKEN 只是字串匹配。關鍵理解：

- `JSON_TOKEN_TRUE` / `JSON_TOKEN_FALSE` / `JSON_TOKEN_NULL` / `JSON_TOKEN_NUMBER` / `JSON_TOKEN_STRING` 是 **parser token type**，不是 JSONB 的內部 type
- JSONB 存儲的是 binary representation，但在 text↔jsonb 轉換過程中，Postgres 不保留原始 PG type 信息
- 如果你需要把 `int8range`、`geometry`、`timestamptz` 等 PG 內部 type 塞進 jsonb 並**能原樣取回**，必須透過 type I/O function 做 text 橋接

> 補充（Senior Dev）：MongoDB BSON 有 `$type` operator 可查 field 的 binary type（string / int / double / date / objectid 等）。Postgres jsonb 沒有對等機制——`jsonb_typeof()` 只返回 `string` / `number` / `boolean` / `null` / `object` / `array`，無法區分 `number` 是 int 還是 float。這在跨系統數據交換時是需要注意的精度風險點。

---

## 3. Type I/O Function 機制

Postgres 每個 type 都有 `typinput` 和 `typoutput` function（定義在 `pg_type` catalog），負責 text ↔ binary 的轉換：

```sql
SELECT oid, typname, typinput, typoutput
FROM pg_type WHERE typname = 'timestamptz';
```

| 屬性 | 值 | 作用 |
|------|-----|------|
| `oid` | 1184 | type OID |
| `typinput` | `timestamptz_in` | text → binary |
| `typoutput` | `timestamptz_out` | binary → text |

用法：

```sql
SELECT timestamptz_out(now());
-- 2015-09-24 19:44:16.076233+08

SELECT timestamptz_in('2015-09-24 19:44:16.076233+08', 1184, 1);
-- 2015-09-24 19:44:16.1+08      (typmod=1, precision truncated)

SELECT timestamptz_in('2015-09-24 19:44:16.076233+08', 1184, 6);
-- 2015-09-24 19:44:16.076233+08 (typmod=6, full precision)
```

`typmod`（第三個參數）控制精度/長度（如 `varchar(N)` 的 N、`numeric(P,S)` 的 P,S、`timestamptz(P)` 的 P）。對 `jsonb_in()` 來說，typmod 固定為 -1（無限制）。

來自 `src/backend/utils/adt/varlena.c` 中的 `text_format()`：

```c
/*
 * Returns a formatted string
 */
Datum
text_format(PG_FUNCTION_ARGS)
{
    ...
}
```

> 補充（Senior Dev）：這個 I/O function 機制是 Postgres type system 的核心——每個 type 的 text ↔ binary 轉換都是 reversible（`typoutput ∘ typinput = identity`）。因此你可以把任何 type **經由 text** 塞進 jsonb 再取出，只要 text representation 沒變。這也是為什麼 jsonb 不存內部 type：它依賴 text 的 self-describing 特性。

---

## 4. format() + jsonb_in() 構造法

利用 `format()` 將任意 type 的 output 格式化為 json string，再用 `jsonb_in()` 解析為 jsonb：

### 基本模式

```sql
-- Step 1: 用 format 產生 json 字串
SELECT format('{"K": "%s"}', int8range(1,10));
-- {"K": "[1,10)"}

-- Step 2: jsonb_in 解析
SELECT jsonb_in(format('{"K": "%s"}', int8range(1,10))::cstring);
-- {"K": "[1,10)"}

-- Step 3: 取出 element
SELECT jsonb_in(format('{"K": "%s"}', int8range(1,10))::cstring) ->> 'K';
-- [1,10)

-- Step 4: 轉回原 type
SELECT (jsonb_in(format('{"K": "%s"}', int8range(1,10))::cstring) ->> 'K')::int8range;
-- [1,10)
```

### 完整建構 / 提取 / 還原 demo

```sql
-- 存入
SELECT jsonb_in(format('{"ts": "%s", "rng": "%s"}',
    now(),
    int8range(1,10)
)::cstring);
-- {"ts": "2015-09-24 19:44:16.076233+08", "rng": "[1,10)"}

-- 取出並還原
WITH src AS (
    SELECT jsonb_in(format('{"ts": "%s", "rng": "%s"}',
        now()::timestamptz,
        int8range(1,10)
    )::cstring) AS data
)
SELECT
    (data ->> 'ts')::timestamptz AS ts,
    (data ->> 'rng')::int8range AS rng
FROM src;
```

> 補充（Senior Dev）：現代 PG（9.4+）提供了更簡潔的 native 構造方式，在多數場景下應優先使用：
> ```sql
> -- PG 9.4+: jsonb_build_object
> SELECT jsonb_build_object('ts', now(), 'rng', int8range(1,10));
>
> -- PG 9.5+: 直接 concatenation
> SELECT jsonb_build_object('a', 1) || jsonb_build_object('b', 2);
>
> -- PG 11+: jsonb 支援 TRANSFORM FOR TYPE（plpgsql 中自動轉換）
> ```
> `format() + jsonb_in()` 模式的優勢在於：你**完全控制 text representation**（如 timestamp 格式、range 邊界符號），這在需要自訂序列化格式時有用。但劣勢是性能較差（多一次 text ↔ binary 轉換）且要處理 escape。

---

## 5. BYTEA 的 Escape 限制

試圖將 `bytea` 塞進 jsonb 時，會遇到 escape 衝突：

```sql
SELECT format('{"K": "%s"}', '你好'::bytea);
-- {"K": "\xe4bda0e5a5bd"}
```

`bytea_out()` 輸出的 hex format 使用 `\x` prefix，但 JSON parser 的 escape handler 不認得 `\x`：

```sql
\set VERBOSITY verbose
SELECT jsonb_in(format('{"K": "%s"}', '你好'::bytea)::cstring);
-- ERROR: 22P02: invalid input syntax for type json
-- DETAIL: Escape sequence "\x" is invalid.
```

Source code 中的 escape handler（`json_lex_string()`）：

```c
static inline void
json_lex_string(JsonLexContext *lex)
{
    ...
    switch (*s)
    {
        case '"':
        case '\\':
        case '/':
            appendStringInfoChar(lex->strval, *s);
            break;
        case 'b':
            appendStringInfoChar(lex->strval, '\b');
            break;
        case 'f':
            appendStringInfoChar(lex->strval, '\f');
            break;
        case 'n':
            appendStringInfoChar(lex->strval, '\n');
            break;
        case 'r':
            appendStringInfoChar(lex->strval, '\r');
            break;
        case 't':
            appendStringInfoChar(lex->strval, '\t');
            break;
        default:
            /* Not a valid string escape, so error out. */
            lex->token_terminator = s + pg_mblen(s);
            ereport(ERROR,
                (errcode(ERRCODE_INVALID_TEXT_REPRESENTATION),
                 errmsg("invalid input syntax for type json"),
                 errdetail("Escape sequence \"\\%s\" is invalid.",
                           extract_mb_char(s)),
                 report_json_context(lex)));
    }
```

Parser 只處理 JSON standard 的 escape：`"` `\\` `/` `\b` `\f` `\n` `\r` `\t`。任何其他 escape 都會報錯。

### Workaround：Output Escape 模式

修改 `json_lex_string()` 後（或使用 native 函數跳過 escape），可達成存入 bytea 並提取還原：

```sql
-- 取出為 hex string（無 \x prefix）
SELECT jsonb_in(format('{"K": "%s"}', '你好'::bytea)::cstring) ->> 'K';
-- e4bda0e5a5bd

-- 還原：手動補回 \x prefix
SELECT convert_from(
    byteain(('\x' || (
        jsonb_in(format('{"K": "%s"}', '你好'::bytea)::cstring) ->> 'K'
    ))::cstring),
    'utf8'::name
);
-- 你好
```

> 補充（Senior Dev）：PG 9.5+ 有更好的方案。`bytea` 的 output 可設定為 `escape` 格式（`SET bytea_output = 'escape'`），此時 `\x` 變為 `\` octet 表示法。PG 9.6+ 可以直接用 `encode()` / `decode()` 配合 `base64` 來避免 escape 衝突：
> ```sql
> -- PG 9.6+: 使用 base64 encode 避開 JSON escape
> SELECT jsonb_build_object('data', encode('你好'::bytea, 'base64'));
> -- {"data": "5L2g5aW9"}
>
> SELECT decode(('{"data": "5L2g5aW9"}'::jsonb ->> 'data'), 'base64');
> -- \xe4bda0e5a5bd
> ```
> 如果你的場景是將 binary data 存進 jsonb，`base64` encode 是通用且安全的做法，跨資料庫兼容性也好。
>
> 關於 `format()` 的另一個陷阱：`%s` 會呼叫類型的 `typoutput` function，而某些類型的 output 包含 control character 或 backslash（如 `point` 類型的 `(1,2)`、`box` 類型的 `(1,2),(3,4)`），這些括號/逗號不會被 JSON parser 誤解因為它們在 quoted string 內。但若類型的 output 本身帶 `"` 字元，format 產出的 JSON 就會損壞——這是一個邊際案例，建議在 production 中優先使用 `jsonb_build_object()` 來避免手動組裝 JSON string 的風險。

---

## 6. json_lex() 完整 Tokenizer 邏輯

`json_lex()` 是 JSON parser 的 tokenizer，負責將輸入 stream 切分為 token（跳過 whitespace `' '` `'\t'` `'\n'` `'\r'`）。其 switch-case 完整覆蓋：

- `{` `}` → `JSON_TOKEN_OBJECT_START` / `END`
- `[` `]` → `JSON_TOKEN_ARRAY_START` / `END`
- `,` `:` → `JSON_TOKEN_COMMA` / `COLON`
- `"` → `json_lex_string()` → `JSON_TOKEN_STRING`
- `-` → `json_lex_number()` with leading minus → `JSON_TOKEN_NUMBER`
- `0-9` → `json_lex_number()` → `JSON_TOKEN_NUMBER`
- 其他 alpha token：匹配 `true`(4) / `false`(5) / `null`(4)，否則報錯

```c
static inline void
json_lex(JsonLexContext *lex)
{
    char       *s;
    int         len;

    /* Skip leading whitespace. */
    s = lex->token_terminator;
    len = s - lex->input;
    while (len < lex->input_length &&
           (*s == ' ' || *s == '\t' || *s == '\n' || *s == '\r'))
    {
        if (*s == '\n')
            ++lex->line_number;
        ++s;
        ++len;
    }
    lex->token_start = s;

    /* Determine token type. */
    if (len >= lex->input_length)
    {
        lex->token_start = NULL;
        lex->prev_token_terminator = lex->token_terminator;
        lex->token_terminator = s;
        lex->token_type = JSON_TOKEN_END;
    }
    else
        switch (*s)
        {
            case '{': lex->token_type = JSON_TOKEN_OBJECT_START;  break;
            case '}': lex->token_type = JSON_TOKEN_OBJECT_END;    break;
            case '[': lex->token_type = JSON_TOKEN_ARRAY_START;   break;
            case ']': lex->token_type = JSON_TOKEN_ARRAY_END;     break;
            case ',': lex->token_type = JSON_TOKEN_COMMA;         break;
            case ':': lex->token_type = JSON_TOKEN_COLON;         break;
            case '"': json_lex_string(lex);
                      lex->token_type = JSON_TOKEN_STRING;         break;
            case '-': json_lex_number(lex, s + 1, NULL);
                      lex->token_type = JSON_TOKEN_NUMBER;         break;
            case '0' ... '9':
                      json_lex_number(lex, s, NULL);
                      lex->token_type = JSON_TOKEN_NUMBER;         break;
            default:
            {
                char *p;
                for (p = s; p - s < lex->input_length - len
                     && JSON_ALPHANUMERIC_CHAR(*p); p++) /* skip */;
                if (p == s)
                    report_invalid_token(lex);
                lex->prev_token_terminator = lex->token_terminator;
                lex->token_terminator = p;
                if (p - s == 4)
                {
                    if (memcmp(s, "true", 4) == 0)
                        lex->token_type = JSON_TOKEN_TRUE;
                    else if (memcmp(s, "null", 4) == 0)
                        lex->token_type = JSON_TOKEN_NULL;
                    else
                        report_invalid_token(lex);
                }
                else if (p - s == 5 && memcmp(s, "false", 5) == 0)
                    lex->token_type = JSON_TOKEN_FALSE;
                else
                    report_invalid_token(lex);
            }
        }
}
```

> 補充（Senior Dev）：注意 `true` 和 `null` 共用 4-byte 分支、`false` 獨佔 5-byte 分支。這意味著 `tree`、`nullx` 等 4-byte 非標準 token 會進入 `true`/`null` 的 `memcmp` 檢查後才報錯（而非直接被 `JSON_ALPHANUMERIC_CHAR` 循環找到的長度觸發），`falsy` 同理。這不會影響正確性但對 custom JSON extension（如 pg_cron 的 JSON config parser）是值得注意的實現細節。
