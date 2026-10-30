# 02 · Relational Databases & SQL

Why a database exists, how to model data so it stays correct, and how to query it: joins, grouping, window functions.

## 1. Why a database (not an in-memory array)
Durability (survives crashes and restarts), one shared truth for every server (servers are stateless), fast lookup (indexes), safe concurrent access (transactions) and rules the data must obey (constraints). Postgres is itself a server (port 5432): each connection costs a TCP/TLS handshake plus a backend process, which is why applications use connection pools.

## 2. Types and keys
- Money → `NUMERIC` (exact decimals), never `FLOAT` (binary rounding error). Timestamps → `TIMESTAMPTZ`. Use `TEXT` unless there is a real length limit.
- **Primary key:** unique, not null identity of a row. Prefer a surrogate key (`BIGINT GENERATED ALWAYS AS IDENTITY` or a UUID) over a natural key such as email (emails change and get reused, and every reference would have to change too). Keep natural values as separate `UNIQUE` columns. Sequential ids are guessable, so never rely on secret ids for authorization.
- **Composite primary key:** for example `order_items(order_id, product_id)`.
- **Foreign key:** the value must exist in the referenced table, so there are no orphan rows.

| `ON DELETE` | Meaning | Fits |
|---|---|---|
| `RESTRICT` / `NO ACTION` | Refuse the delete | `orders → users` (protect history) |
| `CASCADE` | Delete the dependents too | `order_items → orders` |
| `SET NULL` | Keep the row, clear the link | Optional links |

Cost of foreign keys: an extra check on every insert/delete, and **Postgres does not index the FK column automatically** (add the index yourself).

## 3. Constraints
`NOT NULL`, `UNIQUE`, `CHECK (price >= 0)`, `DEFAULT now()`, `PRIMARY KEY`, `REFERENCES`.
Application validation is a courtesy; constraints are a guarantee under bugs, multiple servers and scripts. Idempotency keys rely on a `UNIQUE` constraint.

## 4. SQL essentials
**Logical execution order:** `FROM/JOIN → WHERE → GROUP BY → HAVING → SELECT (+ window functions) → ORDER BY → LIMIT`.
- Aliases from `SELECT` are not visible in `WHERE`; aggregates cannot be used in `WHERE` (use `HAVING`).
- `NULL` means unknown: `= NULL` is never true (use `IS NULL`); `<> 'x'` skips NULL rows (use `IS DISTINCT FROM`); `COUNT(*)` counts rows, `COUNT(col)` counts non-NULL values, `COUNT(DISTINCT col)` counts unique non-NULL values.
- Always use parameterized queries (`$1`) so input is data, never SQL (injection).
- `UPDATE` / `DELETE` without `WHERE` affects every row.

## 5. Modeling relationships
- One-to-many: foreign key on the "many" side.
- Many-to-many: a **junction table** (`user_groups(user_id, group_id)`, composite PK, both FKs).

## 6. Joins
| Join | Keeps |
|---|---|
| `INNER` | Rows matching on both sides |
| `LEFT` | All left rows + matches (NULLs when none) |
| `RIGHT` | Mirror of LEFT (swap tables and use LEFT) |
| `FULL` | All rows from both sides (reconciling two lists) |
| `CROSS` | Every combination (rows multiply: 1,000 × 1,000 = 1,000,000) |
| Self join | A table joined to itself (employee → manager) |

**Traps**
- **Fan-out:** a join multiplies the "one" side's rows, so `SUM` of its columns inflates. Sum before joining, or do not join.
- **LEFT JOIN + `COUNT(*)`** counts the NULL row as 1; use `COUNT(child.id)`.
- **LEFT JOIN + `WHERE` on the right table** silently turns it into an INNER JOIN. Put right-table conditions in `ON`.
- Anti-join ("rows with no match"): `LEFT JOIN … WHERE right.id IS NULL`.

## 7. GROUP BY and HAVING
`GROUP BY` collapses rows into groups; every selected column must be grouped or aggregated. `WHERE` filters rows **before** grouping; `HAVING` filters groups **after**.

## 8. Window functions (standard SQL)
`function() OVER (PARTITION BY group_col ORDER BY sort_col)` computes a value per row from related rows **without collapsing them**.
- `ROW_NUMBER()` 1,2,3 (use a tie-breaker). `RANK()` leaves gaps after ties (1,2,2,4). `DENSE_RANK()` has no gaps (1,2,2,3).
- Running total: `SUM(x) OVER (ORDER BY day)`. Previous / next row: `LAG(x)` / `LEAD(x)` (NULL at the edges).
- Computed at the SELECT step, so you cannot filter on them in the same query: wrap in a subquery or CTE (`WITH name AS (...)`).
- `PARTITION BY` restarts the calculation per group; without it the whole table is one partition.

## 9. Normalization and denormalization
**Anomalies of one big table:** *update* (the same fact edited inconsistently), *insert* (cannot store a fact without another), *delete* (removing one thing destroys a different fact).
- **1NF:** one value per cell, no lists or repeating columns.
- **2NF:** every non-key column depends on the whole key.
- **3NF:** no non-key column depends on another non-key column. ("The key, the whole key, and nothing but the key.")

**Denormalize only for a measured reason.**
| Kind | Example | What you owe |
|---|---|---|
| Snapshot of history | `order_items.unit_price` (price paid ≠ current price) | Nothing: correct modeling |
| Derived counter | `posts.comment_count` | Keep in sync |
| Summary / materialized view | `daily_revenue` | Refresh; accept staleness |
| Copied filter column | `tenant_id` on RAG chunks | Keep equal to the parent |

Keep copies correct: same transaction, trigger, periodic recompute (repair job), or compute on read + cache. Pitfalls: read-modify-write race (use `SET n = n + 1`), a missed write path, partial failure, a hot row on viral items.

## 10. SQL patterns cheat sheet
```sql
-- anti-join
SELECT u.* FROM users u LEFT JOIN orders o ON o.user_id = u.id WHERE o.id IS NULL;
-- count per parent, including zero
SELECT u.id, COUNT(o.id) FROM users u LEFT JOIN orders o ON o.user_id = u.id GROUP BY u.id;
-- filter rows, then filter groups
SELECT user_id, SUM(total) FROM orders WHERE status <> 'cancelled'
GROUP BY user_id HAVING SUM(total) > 6000;
-- top row per group
SELECT * FROM (
  SELECT o.*, ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY created_at DESC, id DESC) AS rn
  FROM orders o) t WHERE rn = 1;
-- keyset (cursor) page
SELECT * FROM items WHERE (created_at, id) < (:ts, :id)
ORDER BY created_at DESC, id DESC LIMIT 21;
```

## Interview one-liners
- "Constraints are guarantees; application validation is a courtesy."
- "Normalize first; denormalize only for a measured read path, and then I owe consistency (atomic updates plus a repair job)."
- "After a join I check for fan-out before I SUM or COUNT."
- "Windows keep the rows; GROUP BY collapses them."
