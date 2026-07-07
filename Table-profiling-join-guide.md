# Profiling Unknown Tables & Finding Join Relationships

A practical checklist for reverse-engineering tables when you don't have
documentation — the archaeology that dimensional-modeling books assume away.
Examples use Oracle SQL but the logic ports to any dialect.

---

## 1. Profile each table alone first

Before joining anything, understand what one row *is*.

```sql
-- Row count + distinct count of your suspected key
SELECT COUNT(*)                              AS total_rows,
       COUNT(DISTINCT suspected_key)         AS distinct_keys,
       COUNT(*) - COUNT(suspected_key)       AS null_keys
FROM mystery_table;
```

- `total_rows = distinct_keys` and no nulls → candidate primary/unique key.
- `total_rows > distinct_keys` → the grain is finer than that column; something
  else (or a combination) defines the row.

---

## 2. Find the real grain

When no single column is unique, test combinations:

```sql
SELECT col_a, col_b, COUNT(*) AS n
FROM mystery_table
GROUP BY col_a, col_b
HAVING COUNT(*) > 1
ORDER BY n DESC;
```

- Empty result → `(col_a, col_b)` is the grain.
- Rows returned → still coarser than reality; add another column and retest.

---

## 3. Identify join keys by name + type + overlap

Name and datatype get you candidates; overlap confirms them.

```sql
-- How many of A's keys actually exist in B?
SELECT COUNT(*)        AS a_rows,
       COUNT(b.key)    AS matched_in_b
FROM table_a a
LEFT JOIN table_b b ON a.some_id = b.some_id;
```

A low match rate means the wrong key, the wrong direction, or a transformation
in between (zero-padding, prefixes, case, trimming). Watch for
`VARCHAR '00123'` vs `NUMBER 123` — silent killers.

---

## 4. Determine cardinality (the fan-out check)

This step prevents accidental row multiplication.

```sql
-- Does each A match at most one B? Check the B side's key uniqueness.
SELECT some_id, COUNT(*) AS dupes
FROM table_b
GROUP BY some_id
HAVING COUNT(*) > 1;
```

- B's key unique → **many-to-one** (safe, no fan-out).
- B's key has dupes → **many-to-many** → your join multiplies rows.

Always confirm cardinality *before* trusting an aggregate after a join.

---

## 5. Validate the join didn't change your grain

```sql
SELECT COUNT(*) FROM table_a;                          -- before
SELECT COUNT(*) FROM table_a a JOIN table_b b ON ...;  -- after
```

- Count grew but you expected many-to-one → your key assumption is wrong.
- Count shrank → an inner join is dropping unmatched rows; decide whether
  that's intended.

---

## 6. Sniff out FK relationships across the schema

When you don't even know which tables relate, scan the catalog for shared
column names:

```sql
SELECT table_name, column_name
FROM all_tab_columns
WHERE column_name LIKE '%\_ID' ESCAPE '\'
  AND owner = 'YOUR_SCHEMA'
ORDER BY column_name, table_name;
```

Columns sharing a name across tables are your join-graph candidates — then
confirm each with the overlap test in step 3.

---

## Habits worth keeping

- **Trust data over column names.** A column called `customer_id` may hold
  quote IDs.
- **Check nulls before assuming an inner join is lossless.**
- **Sample actual values** to catch format mismatches your eyes would miss:

```sql
SELECT DISTINCT some_id
FROM mystery_table
FETCH FIRST 20 ROWS ONLY;
```
