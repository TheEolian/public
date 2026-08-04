# Grain-Finding Exercise

A hands-on procedure for discovering and confirming the grain of an unfamiliar table. Run these in order on a real table. Replace `TABLE_NAME`, `col1`, `col2`, etc. with actual names.

Grain = the combination of columns at which one row is unique. "What does one row represent?"

---

## Step 0 — Size the table

Get a baseline row count. Every later check compares against this.

```sql
SELECT COUNT(*) AS total_rows
FROM TABLE_NAME;
```

---

## Step 1 — Test your first guess at the key

Guess the column(s) you think make a row unique. Group by them and look for duplicates.

```sql
SELECT col1, col2, COUNT(*) AS cnt
FROM TABLE_NAME
GROUP BY col1, col2
HAVING COUNT(*) > 1
ORDER BY cnt DESC;
```

- **Zero rows returned** → `col1, col2` is a valid grain (unique). Go to Step 4 to confirm.
- **Rows returned** → your guess is too coarse. There's another dimension. Go to Step 2.

---

## Step 2 — Inspect a duplicate to find the missing dimension

Pick one duplicated key from Step 1 and pull all its rows. Eyeball which column actually differs between them.

```sql
SELECT *
FROM TABLE_NAME
WHERE col1 = 'SOME_VALUE'   -- a value that showed cnt > 1
  AND col2 = 'SOME_VALUE'
ORDER BY col1, col2;
```

Look for what changes row to row — a version number, effective date, coverage code, vehicle number, transaction sequence, claimant id. That differing column is a missing part of the grain. Add it to your candidate list.

---

## Step 3 — Discover the grain by subtraction (when you have no good guess)

Put **all** plausible dimension columns in the `GROUP BY`. Then remove one column at a time and re-run. The moment `COUNT(*)` goes above 1, the column you just removed is part of the grain.

Start wide:

```sql
SELECT col1, col2, col3, col4, COUNT(*) AS cnt
FROM TABLE_NAME
GROUP BY col1, col2, col3, col4
HAVING COUNT(*) > 1;   -- expect zero rows if this full set is unique
```

Now remove `col4` and re-run:

```sql
SELECT col1, col2, col3, COUNT(*) AS cnt
FROM TABLE_NAME
GROUP BY col1, col2, col3
HAVING COUNT(*) > 1;
```

- If this **still returns zero rows**, `col4` was NOT part of the grain (removing it didn't create duplicates). Keep going — remove `col3` next.
- If this **now returns rows**, `col4` WAS part of the grain (removing it created duplicates). Put it back and it stays in the key.

Repeat, removing one column each pass, until you've tested every column. What's left that you *can't* remove without creating duplicates = the grain.

---

## Step 4 — Confirm the candidate key is truly unique

Once you think you have the grain, prove it two ways.

**4a. Distinct key combinations should equal total rows:**

```sql
SELECT
    COUNT(*)                                   AS total_rows,
    COUNT(DISTINCT col1 || '~' || col2)        AS distinct_keys
FROM TABLE_NAME;
```

If `total_rows = distinct_keys`, the key is unique. (Adjust the concatenation syntax to your dialect — Oracle uses `||`.)

**4b. The duplicate check returns nothing:**

```sql
SELECT col1, col2, COUNT(*)
FROM TABLE_NAME
GROUP BY col1, col2
HAVING COUNT(*) > 1;
```

Zero rows = confirmed.

---

## Step 5 — Check the key columns for NULLs

A NULL in a key column breaks uniqueness logic and causes trouble in joins.

```sql
SELECT
    SUM(CASE WHEN col1 IS NULL THEN 1 ELSE 0 END) AS col1_nulls,
    SUM(CASE WHEN col2 IS NULL THEN 1 ELSE 0 END) AS col2_nulls
FROM TABLE_NAME;
```

Any NULLs here mean the "key" isn't reliable — investigate before trusting it in a join.

---

## Step 6 — Write the grain down in plain words

Translate the confirmed key into a sentence. This is what you carry into join planning.

```
TABLE_NAME grain: one row per <col1> per <col2>
   e.g. "one row per policy per term"
        "one row per claim per coverage"
        "one row per vehicle per policy"
```

---

### Quick reference — the whole loop

1. Count total rows.
2. Guess a key → GROUP BY + HAVING COUNT(*) > 1.
3. Duplicates? Inspect one, find the differing column, add it.
4. No guess? Group by everything, remove one column at a time, watch for COUNT > 1.
5. Confirm: distinct keys == total rows, and duplicate check returns nothing.
6. Check key columns for NULLs.
7. Write the grain as a plain sentence.
