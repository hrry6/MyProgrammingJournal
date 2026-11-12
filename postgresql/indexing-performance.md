# ⚡ Indexing & Performance

[🏠 Home](../README.md) > [🐘 PostgreSQL](./README.md)

## 📌 Index Types & When to Use Them

| Index Type | Best For | Example Operators |
| :--- | :--- | :--- |
| **B-Tree** *(Default)* | Exact matches, sorting, and ranges | `=`, `<`, `>`, `BETWEEN`, `ORDER BY` |
| **Composite** | Filtering by multiple columns together | `WHERE a = 1 AND b = 2` |
| **Partial** | Subsets of data (e.g., soft-deleted tables) | `WHERE deleted_at IS NULL` |
| **GIN** | Multi-value columns (`JSONB`, Arrays, Full-Text) | `@>`, `?`, `@@` |

```sql
-- 1. Standard B-Tree index
CREATE INDEX idx_users_email ON users(email);

-- 2. Composite index (Left-to-Right rule: works for (user_id) or (user_id, status), NOT (status) alone)
CREATE INDEX idx_orders_user_status ON orders(user_id, status);

-- 3. Partial index (smaller & faster: ignores soft-deleted rows)
CREATE INDEX idx_active_users_email ON users(email) WHERE deleted_at IS NULL;

-- 4. GIN index (for JSONB containment queries or arrays)
CREATE INDEX idx_events_payload ON events USING GIN (payload);

-- 5. Non-blocking index creation (prevents write locks on live production tables)
CREATE INDEX CONCURRENTLY idx_logs_created_at ON logs(created_at);
```

> **💡 Rule of Thumb:** Don't index everything. Every index speeds up `SELECT`, but slows down `INSERT`/`UPDATE`/`DELETE` because Postgres has to update the index too.

---

## 🔬 Query Analysis (`EXPLAIN ANALYZE`)

Use `EXPLAIN (ANALYZE, BUFFERS)` to actually run the query and inspect real execution stats (avoid running `ANALYZE` on `DELETE`/`UPDATE` unless wrapped in a `ROLLBACK` transaction).

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM users WHERE email = 'dev@example.com';
```

### How to Read the Output

* **`Seq Scan`:** Reads the entire table row-by-row. Normal for small tables, red flag for large ones.
* **`Index Scan`:** Finds row pointers in the index, then fetches the full row from the table.
* **`Index Only Scan`:** Fastest scan — all requested columns already exist inside the index (no table lookup needed).
* **`Bitmap Heap Scan`:** Combines multiple indexes or fetches many matching rows in bulk.
* **`Buffers (shared hit vs read)`:** `hit` = read from fast RAM cache; `read` = fetched from slower disk.
* **`Execution Time`:** Total real time spent running the query (in milliseconds).

---

## 🧹 Maintenance & Inspection

```sql
-- Check total size of a table including all its indexes
SELECT pg_size_pretty(pg_total_relation_size('users'));

-- Check individual index sizes on a table
SELECT indexname, pg_size_pretty(pg_relation_size(indexname::regclass))
FROM pg_indexes
WHERE tablename = 'users';

-- Update query planner statistics & reclaim space from dead rows
VACUUM ANALYZE users;

-- Rebuild a bloated or corrupted index without locking writes
REINDEX INDEX CONCURRENTLY idx_users_email;
```
