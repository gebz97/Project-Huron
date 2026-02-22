
## Chapter 9: Advanced SQL

### Boolean Operators

| Operator | Description |
|----------|-------------|
| `<` | Less than |
| `<=` | Less than or equal |
| `>` | Greater than |
| `>=` | Greater than or equal |
| `=` | Equal |
| `<>` or `!=` | Not equal |

### Logical Operators

| AND | TRUE | FALSE | NULL |
|-----|------|-------|------|
| TRUE | TRUE | FALSE | NULL |
| FALSE | FALSE | FALSE | FALSE |
| NULL | NULL | FALSE | NULL |

| OR | TRUE | FALSE | NULL |
|----|------|-------|------|
| TRUE | TRUE | TRUE | TRUE |
| FALSE | TRUE | FALSE | NULL |
| NULL | TRUE | NULL | NULL |

| NOT | Result |
|-----|--------|
| TRUE | FALSE |
| FALSE | TRUE |
| NULL | NULL |

### Mathematical Functions

| Function | Description | Example |
|----------|-------------|---------|
| `abs(x)` | Absolute value | `abs(-1)` |
| `cbrt(x)` | Cube root | `cbrt(9)` |
| `ceiling(x)` | Round up | `ceiling(4.2)` |
| `degrees(x)` | Radians to degrees | `degrees(1.047)` |
| `exp(x)` | e^x | `exp(1)` |
| `floor(x)` | Round down | `floor(4.2)` |
| `ln(x)` | Natural log | `ln(exp(1))` |
| `log(b, x)` | Base b log | `log(2, 64)` |
| `log2(x)` | Base 2 log | `log2(64)` |
| `log10(x)` | Base 10 log | `log10(140)` |
| `mod(n, m)` | Modulo | `mod(3, 2)` |
| `power(x, p)` | x^p | `pow(2, 6)` |
| `radians(x)` | Degrees to radians | `radians(60)` |
| `round(x)` | Round to integer | `round(pi())` |
| `round(x, d)` | Round to d decimals | `round(pi(), 2)` |
| `sqrt(x)` | Square root | `sqrt(64)` |
| `truncate(x)` | Truncate decimal | `truncate(e())` |

### Trigonometric Functions

| Function | Description |
|----------|-------------|
| `cos(x)` | Cosine |
| `acos(x)` | Arc cosine |
| `cosh(x)` | Hyperbolic cosine |
| `sin(x)` | Sine |
| `asin(x)` | Arc sine |
| `tan(x)` | Tangent |
| `atan(x)` | Arc tangent |
| `atan2(y, x)` | Arc tangent of y/x |
| `tanh(x)` | Hyperbolic tangent |

### Constants

| Function | Value |
|----------|-------|
| `e()` | 2.718281828459045 |
| `pi()` | 3.141592653589793 |
| `infinity()` | Infinity |
| `nan()` | Not a number |

### String Functions

| Function | Description | Example |
|----------|-------------|---------|
| `chr(n)` | Unicode code point to char | `chr(65)` |
| `codepoint(string)` | Char to code point | `codepoint('A')` |
| `concat(s1, ..., sN)` | Concatenate | `concat('a', 'b')` |
| `length(string)` | String length | `length('hello')` |
| `lower(string)` | Lowercase | `lower('ABC')` |
| `lpad(string, size, pad)` | Left pad | `lpad('A', 4, ' ')` |
| `ltrim(string)` | Trim leading spaces | `ltrim('  A')` |
| `replace(string, search, replace)` | Replace | `replace('555.555', '.', '-')` |
| `reverse(string)` | Reverse | `reverse('abc')` |
| `rpad(string, size, pad)` | Right pad | `rpad('A', 4, '#')` |
| `rtrim(string)` | Trim trailing spaces | `rtrim('A  ')` |
| `split(string, delimiter)` | Split to array | `split('a,b,c', ',')` |
| `split_to_map(string, entryDelim, kvDelim)` | Parse to map | `split_to_map('a=1&b=2', '&', '=')` |
| `split_to_multimap(string, entryDelim, kvDelim)` | Parse with duplicates | `split_to_multimap('a=1&a=2', '&', '=')` |
| `strpos(string, substring)` | Position of substring | `strpos('trino.io', '.io')` |
| `substr(string, start, length)` | Substring | `substr('trino.io', 1, 5)` |
| `trim(string)` | Trim both ends | `trim('  A  ')` |
| `upper(string)` | Uppercase | `upper('abc')` |
| `word_stem(word, lang)` | Stem word | `word_stem('trino', 'it')` |

### Unicode Functions

| Function | Description |
|----------|-------------|
| `chr(n)` | Unicode code point to char |
| `codepoint(string)` | Char to code point |
| `normalize(string)` | NFC normalization |
| `normalize(string, form)` | Specified normalization (NFD, NFC, NFKD, NFKC) |
| `to_utf8(string)` | Encode to UTF-8 varbinary |
| `from_utf8(binary)` | Decode UTF-8, replace invalid with U+FFFD |
| `from_utf8(binary, replace)` | Decode, replace invalid with custom |

**Example:**
```sql
SELECT u&'\00F1', u&'\006E\0303',
       u&'\00F1' = u&'\006E\0303',  -- false
       normalize(u&'\00F1') = normalize(u&'\006E\0303');  -- true
```

### Regular Expressions

| Function | Description |
|----------|-------------|
| `regexp_extract_all(string, pattern[, group])` | Array of matches |
| `regexp_extract(string, pattern[, group])` | First match |
| `regexp_like(string, pattern)` | Boolean if pattern exists |
| `regexp_replace(string, pattern[, replacement])` | Replace matches |
| `regexp_replace(string, pattern, function)` | Replace with lambda |
| `regexp_split(string, pattern)` | Split on pattern |

**Examples:**
```sql
SELECT regexp_extract_all('abbbbcccb', 'b');        -- [b,b,b,b,b]
SELECT regexp_extract_all('abbbbcccb', 'b+');      -- [bbbb,b]
SELECT regexp_replace('abc', '(b)(c)', '$2$1');    -- acb
```

### UNNEST (Complex Types)

```sql
-- Sample data
SELECT * FROM permissions;
-- matt | [[WebService_ReadWrite, Storage_ReadWrite], [Billing_Read]]
-- martin | [[WebService_ReadWrite, Storage_ReadWrite], [Billing_ReadWrite, Audit_Read]]

-- Unnest roles
SELECT user, t.roles
FROM permissions,
UNNEST(permissions.roles) AS t(roles);

-- Unnest permissions
SELECT user, permission
FROM permissions,
UNNEST(permissions.roles) AS t1(roles),
UNNEST(t1.roles) AS t2(permission);

-- Filter
SELECT user, permission
FROM permissions,
UNNEST(permissions.roles) AS t1(roles),
UNNEST(t1.roles) AS t2(permission)
WHERE permission = 'Audit_Read';
```

### JSON Functions

| Function | Description |
|----------|-------------|
| `is_json_scalar(json)` | Is value scalar? |
| `json_array_contains(json, value)` | Array contains value? |
| `json_array_length(json)` | Array length |

### Date/Time Functions

| Function | Description |
|----------|-------------|
| `current_timezone()` | Current time zone |
| `current_date` | Current date |
| `current_time` | Current time with zone |
| `current_timestamp` or `now()` | Current timestamp |
| `localtime` | Local time |
| `localtimestamp` | Local timestamp |
| `from_unixtime(unixtime)` | Unix time to timestamp |
| `to_unixtime(timestamp)` | Timestamp to Unix time |
| `to_milliseconds(interval)` | Interval to milliseconds |
| `from_iso8601_timestamp(string)` | ISO 8601 to timestamp |
| `from_iso8601_date(string)` | ISO 8601 to date |

**Examples:**
```sql
SELECT TIME '12:00' + INTERVAL '1' HOUR;                       -- 13:00:00.000
SELECT INTERVAL '1' YEAR + INTERVAL '15' MONTH;                -- 2-3
SELECT TIME '02:56:15 UTC' AT TIME ZONE '-08:00';              -- 18:56:15.000 -08:00
SELECT from_iso8601_timestamp('2019-03-17T21:05:19Z');         -- 2019-03-17 21:05:19.000 UTC
SELECT from_iso8601_timestamp('2019-W10');                      -- 2019-03-04 00:00:00.000 UTC
```

### Histograms

```sql
SELECT count(*) count, year, width_bucket(year, 2010, 2020, 4) bucket
FROM flights_orc
WHERE year >= 2010
GROUP BY year;
```

### Aggregate Functions

| Function | Description |
|----------|-------------|
| `count(*)` | Count rows |
| `count(x)` | Count non-null x |
| `sum(x)` | Sum |
| `min(x)` | Minimum |
| `max(x)` | Maximum |
| `avg(x)` | Average |

### Map Aggregate Functions

| Function | Description |
|----------|-------------|
| `histogram(x)` | Count of each value |
| `map_agg(key, value)` | Map from key/value (random if duplicate) |
| `map_union(x(K,V))` | Union of maps (random if key conflict) |
| `multimap_agg(key, value)` | Map of key → array of values |

**Examples:**
```sql
SELECT histogram(floor(petal_length_cm)) FROM memory.default.iris;
-- {1.0=50, 4.0=43, 5.0=35, 3.0=11, 6.0=11}

SELECT multimap_agg(species, petal_length_cm) FROM memory.default.iris;
-- {versicolor=[4.7, 4.5, 4.9...], virginica=[6.0, 5.1, 5.9...], setosa=[1.4, 1.4, 1.3...]}
```

### Approximate Aggregate Functions

| Function | Description |
|----------|-------------|
| `approx_distinct(x)` | Approximate distinct count |
| `approx_percentile(x, percentile)` | Approximate percentile |

### Window Functions

```sql
-- Overall average
SELECT species, sepal_length_cm,
       avg(sepal_length_cm) OVER() AS avgsepal
FROM memory.default.iris;

-- Partition by species
SELECT species, sepal_length_cm,
       avg(sepal_length_cm) OVER(PARTITION BY species) AS avgsepal
FROM memory.default.iris;

-- Multiple window functions
SELECT DISTINCT species,
       min(sepal_length_cm) OVER(PARTITION BY species) AS minsepal,
       avg(sepal_length_cm) OVER(PARTITION BY species) AS avgsepal,
       max(sepal_length_cm) OVER(PARTITION BY species) AS maxsepal
FROM memory.default.iris;
```

### Lambda Expressions

```sql
SELECT zip_with(ARRAY[1, 2, 6, 2, 5],
                ARRAY[3, 4, 2, 5, 7],
                (x, y) -> x + y);
-- [4, 6, 8, 7, 12]
```

### Geospatial Functions

| Function | Description |
|----------|-------------|
| `ST_GeometryFromText(varchar)` | Geometry from WKT |
| `ST_GeomFromBinary(varbinary)` | Geometry from WKB |
| `ST_Point(double, double)` | Create point |
| `ST_Contains(Geometry, Geometry)` | Contains? |
| `ST_Distance(Geometry, Geometry)` | Distance |
| `ST_Area(SphericalGeography)` | Area |

### Prepared Statements

```sql
-- Prepare
PREPARE delay_query FROM
SELECT dayofweek, avg(depdelayminutes) AS delay
FROM flights_orc
WHERE month = ? AND origincityname LIKE ?
GROUP BY dayofweek
ORDER BY dayofweek;

-- Execute with parameters
EXECUTE delay_query USING 2, '%Boston%';

-- Describe input/output
DESCRIBE INPUT delay_query;
DESCRIBE OUTPUT delay_query;

-- Deallocate
DEALLOCATE PREPARE delay_query;
```

---
