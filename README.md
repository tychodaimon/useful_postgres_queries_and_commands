# Useful PostgreSQL Queries & Commands

A collection of handy SQL queries and CLI commands for PostgreSQL diagnostics, maintenance, and optimization.

---

## 🔍 Query Monitoring

### Show Running Queries
```sql
SELECT pid, age(clock_timestamp(), query_start), usename, ltrim(query, E'\n' || ' ') as query
FROM pg_stat_activity 
WHERE state != 'idle' 
  AND query NOT ILIKE '%pg_stat_activity%' 
  AND usename IS NOT NULL 
  AND query NOT ILIKE '%replica%'
ORDER BY query_start DESC;
```

### Cancel a Running Query
```sql
SELECT pg_cancel_backend(pid);
```

### Terminate an Idle Query
```sql
SELECT pg_terminate_backend(pid);
```

### Is this connection repplica or master?
```sql
SELECT current_setting('transaction_read_only') = 'on' as is_replica;
```

---

## 📊 Table & Index Insights

### Table Size Breakdown
```sql
WITH table_sizes AS (
  SELECT *, total_bytes - index_bytes - COALESCE(toast_bytes, 0) AS table_bytes
  FROM (
    SELECT c.oid,
           nspname AS table_schema,
		   case when c.relkind = 'm' then 'view' else 'table' end as "type", 
           relname AS table_name,
           c.reltuples AS row_estimate,
           pg_total_relation_size(c.oid) AS total_bytes,
           pg_indexes_size(c.oid) AS index_bytes,
           pg_total_relation_size(reltoastrelid) AS toast_bytes
    FROM pg_class c
    LEFT JOIN pg_namespace n ON n.oid = c.relnamespace
    WHERE c.relkind IN ('r','m') AND n.nspname NOT IN ('pg_catalog', 'information_schema')
  ) AS stats
  ORDER BY total_bytes DESC
)
SELECT *,
       pg_size_pretty(total_bytes) AS total,
       pg_size_pretty(index_bytes) AS index,
       pg_size_pretty(toast_bytes) AS toast,
       pg_size_pretty(table_bytes) AS table
FROM table_sizes;
```

### Unused Indexes
```sql
SELECT
    s.schemaname,
    s.relname AS tablename,
    s.indexrelname AS indexname,
    pg_relation_size(s.indexrelid) AS index_size,
	pg_size_pretty(pg_relation_size(s.indexrelid)) AS size_pretty,
    s.idx_scan AS index_scans,
    s.idx_tup_read AS tuples_read,
    s.idx_tup_fetch AS tuples_fetched
FROM pg_stat_user_indexes s
JOIN pg_index i ON s.indexrelid = i.indexrelid
WHERE s.idx_scan = 0
  AND 0 <> ALL (i.indkey)
  AND NOT i.indisunique
  AND NOT EXISTS (SELECT 1 FROM pg_constraint c WHERE c.conindid = s.indexrelid)
ORDER BY pg_relation_size(s.indexrelid) DESC;
```

---

## 🧹 Vacuum & Bloat

### Vacuum Statistics
```sql
SELECT schemaname,
       relname,
       n_live_tup,
       n_dead_tup,
       n_dead_tup / GREATEST(n_live_tup + n_dead_tup, 1)::float * 100 AS dead_percentage,
       last_vacuum,
       last_autovacuum,
       last_autoanalyze,
       vacuum_count,
       autovacuum_count
FROM pg_stat_user_tables
WHERE n_dead_tup > 0
ORDER BY n_dead_tup DESC, last_vacuum DESC, last_autovacuum DESC;
```

---

## 🔐 Locking

### Detect Locks and Blocked Queries
```sql
SELECT pid, pg_blocking_pids(pid) AS blocked_by, query AS blocked_query
FROM pg_stat_activity
WHERE pg_blocking_pids(pid)::text != '{}';
```

---

## 💾 Backup & Restore

### Dump Remote Database to File
```bash
pg_dump -U username -h hostname \
  --format plain --no-owner --encoding UTF8 \
  --no-privileges --no-tablespaces --no-unlogged-table-data \
  "databasename" > dump.sql
```

### Import Dump into Existing Database
```bash
psql -d newdb -f dump.sql
```

---

## 🧽 Maintenance

### Full Vacuum Freeze & Analyze
```bash
vacuumdb -afz
```

---

## 📘 Notes
- Be cautious with `pg_terminate_backend()` — it forcibly stops a session and may cause rollbacks.
- Always run backups before large-scale index drops or vacuums.
