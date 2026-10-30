# 03 · Indexes & Query Plans

How Postgres finds rows, what an index buys and costs, and how to read an execution plan.

## 1. Storage basics
A table is a file of **8 KB pages** (the *heap*). Each row has a **TID** = (page, slot). Disks read whole pages, so reading one row costs reading its page. A sequential scan reads every page; an index stores sorted `(key → TID)` entries so a lookup reads only a few pages.

## 2. B-tree (default index)
Root → internal nodes → leaf pages, with leaves sorted and linked left to right.
- **Wide and shallow:** hundreds of keys per page (fan-out ~300), so ~3 levels cover tens of millions of rows → a lookup is ~3-4 page reads (a binary tree would need ~25).
- Sorted, linked leaves give **range queries, ORDER BY and LIMIT, and keyset pagination** almost for free.
- A B-tree cannot help with: a function on the column (`lower(email)`), a leading wildcard (`LIKE '%x'`), or a non-leading column of a composite index.

## 3. Scan types
| Scan | Typical when |
|---|---|
| Seq Scan | Tiny table, or the query needs a large share of rows |
| Index Scan | Few rows match |
| Bitmap Index + Bitmap Heap Scan | A medium number of rows: collect TIDs, sort by page, read each page once |
| Index Only Scan | The index holds every needed column (and the visibility map is current) |

**Why the planner skips an index at ~33% selectivity:** matching rows touch almost every page, but via random jumps; random reads cost more than sequential ones (default `random_page_cost = 4`), so one ordered pass wins.
- **Selectivity:** fraction of rows a condition matches. **Cardinality:** number of distinct values.

## 4. Composite indexes
- **Leftmost prefix:** an index on `(a, b, c)` serves `a`, `a,b`, `a,b,c`, not `b` or `c` alone. (Postgres 18's skip scan only helps when the leading column has very few distinct values.)
- **Equality columns first, then the range or sort column:** the matching entries then sit contiguously, already in sort order. With the reverse order they are scattered between other entries.
- `(user_id, created_at DESC)` serves `WHERE user_id = ? ORDER BY created_at DESC LIMIT 20` with no Sort node, and the scan stops after 20 rows.
- A prefix index (`(user_id)`) is redundant next to `(user_id, created_at)`. Mixed ASC/DESC needs an index declared with the same mix.

## 5. Covering, partial, expression
- **Covering:** `INCLUDE (col)` adds payload columns so the query can be an Index Only Scan. Heap Fetches mean the visibility map is stale → **VACUUM**. (`ANALYZE` refreshes planner statistics; `VACUUM` cleans dead rows and the visibility map.)
- **Partial:** `CREATE INDEX … WHERE status = 'failed'` is tiny and cheap; the query must repeat the condition. Good for unprocessed jobs, soft deletes, "unique among active rows".
- **Expression:** `CREATE INDEX … ON t (lower(email))`; the index must store what you search for.

## 6. Why isn't my index used?
| Cause | Fix |
|---|---|
| Table is tiny | Nothing: the planner is right |
| Matches too many rows | Composite/partial index, or accept the seq scan |
| Function on the column | Expression index |
| Leading wildcard | Trigram (GIN) or full-text index |
| Wrong column order | Reorder |
| Stale statistics | `ANALYZE` |
| Column compared as another type | Pass the correct type |
| `OR` across columns | Separate indexes (BitmapOr) |
| Sort mismatch | Match the index directions |

## 7. The price of indexes
Every insert writes to the heap **and every index**; an update of an indexed column updates the index. Indexes also take disk space, compete with table data for cache, and bloat until VACUUM. Production habits: `CREATE INDEX CONCURRENTLY` (no write blocking; cannot run in a transaction), find unused indexes with `pg_stat_user_indexes` (`idx_scan = 0`), index for measured queries only.

## 8. Planner and EXPLAIN
The planner estimates the cost of candidate plans from statistics and picks the cheapest. Stale statistics → bad estimates → bad plans (`ANALYZE` after bulk loads).

```
Limit  (cost=0.42..9.10 rows=20 width=48) (actual time=0.040..0.075 rows=20 loops=1)
  ->  Index Scan using idx_user_created on events (...)
        Index Cond: (user_id = 4242)
        Buffers: shared hit=24
Execution Time: 0.110 ms
```
- Plan = a tree of **nodes** (Limit, Sort, Index Scan); the deepest node runs first and feeds its parent.
- `cost` is the planner's estimate in arbitrary units. `rows=` is the estimate; `actual … rows=` is reality; `loops` multiplies time.
- `Buffers: shared hit` = pages found in memory; `read` = fetched from disk/OS. `Rows Removed by Filter` = wasted work (rows examined = returned + removed).
- **Estimate vs actual off by 10× or more → suspect stale statistics** (`ANALYZE`).
- `EXPLAIN` does not run the query; `EXPLAIN ANALYZE` does (wrap writes in `BEGIN … ROLLBACK`). Run twice: cold vs warm.
- A **Sort** node means the index did not deliver the needed order; a composite index `(filter, sort)` removes it.

**Join algorithms:** Nested Loop (small outer + indexed inner lookup), Hash Join (big unsorted `=` joins; spills to disk if memory is short), Merge Join (both inputs sorted). Without an index on a foreign-key column, joins and parent deletes scan the child table.

## 9. Other index types
B-tree (equality, range, sort) · Hash (equality only) · **GIN** (contains: full text, arrays, JSONB) · GiST/SP-GiST (geometry, ranges) · **BRIN** (huge append-only tables in time order) · **HNSW / IVFFlat** (pgvector approximate nearest-neighbor).
1M embeddings × 1,536 dims × 4 bytes ≈ 6 GB; without an index, exact search scans all of it per query. HNSW trades exactness (recall), write cost and memory for speed.

## 10. Index design recipe
1. List each query's `WHERE`, `ORDER BY`, `LIMIT`.
2. Equality columns first, then the range/sort column.
3. Merge: drop any index that is a prefix of another.
4. Rare conditions → partial index. Functions on columns → expression index.
5. Measure with `EXPLAIN ANALYZE`; every index must earn its place.

## Interview one-liners
- "B-trees are wide and shallow because disks charge per page: three levels cover tens of millions of rows."
- "The planner skips an index when most pages would be touched anyway; random I/O costs more than sequential."
- "Equality first, then range or sort, so matching entries are contiguous and already ordered."
- "Every index taxes every write and competes for cache, so I index measured queries only."
- "Estimate versus actual rows far apart means stale statistics: I run ANALYZE and re-check the plan."
