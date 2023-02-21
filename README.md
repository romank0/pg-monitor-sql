pg-monitor-sql
==============

PostrgreSQL monitoring queries

[Kill backend](#kill)

[Reload config](#reload)

Overview
-------

```
SELECT
  d.datname,
  pg_size_pretty(pg_database_size(d.datname)) as size,
  numbackends,
  xact_commit,
  xact_rollback,
  blks_read,
  blks_hit,
  blks_hit::float / ( blks_hit + blks_read ) * 100.0 as cache_hit_ratio,
  tup_fetched,
  tup_returned,
  tup_inserted,
  tup_updated,
  tup_deleted
FROM
  pg_stat_database d
RIGHT JOIN
  pg_database on d.datname = pg_database.datname
WHERE
  not datistemplate and d.datname != 'postgres';
```

### Cache usage

```
SELECT
  d.datname::text,
  100.0 * blks_hit / (blks_read + blks_hit) AS cache_hit_ratio
FROM
  pg_stat_database d
RIGHT JOIN
  pg_database on d.datname = pg_database.datname
WHERE
 not datistemplate and d.datname != 'postgres';
```

### Sequential vs index scans

```
SELECT
  SUM(seq_scan) / SUM(idx_scan),
  SUM(seq_tup_read) / SUM(idx_tup_fetch)
  FROM pg_stat_all_tables;
```


### Statements distribution
```
SELECT
 d.datname::text,
 case when
   (tup_returned + tup_inserted + tup_updated + tup_deleted) > 0
 then round(1000000.0 * tup_returned / (tup_returned + tup_inserted + tup_updated + tup_deleted)) / 10000
 else 0
 end::numeric(1000, 4) as select_pct,

 case when (tup_returned + tup_inserted + tup_updated + tup_deleted) > 0
 then
 round(1000000.0 * tup_inserted / (tup_returned + tup_inserted + tup_updated + tup_deleted)) / 10000
 else 0
 end::numeric(1000, 4) as insert_pct ,

 case when (tup_returned + tup_inserted + tup_updated + tup_deleted) > 0
 then
 round(1000000.0 * tup_updated / (tup_returned + tup_inserted + tup_updated + tup_deleted)) / 10000
 else 0
 end::numeric(1000, 4) as update_pct,

 case when (tup_returned + tup_inserted + tup_updated + tup_deleted) > 0
 then round(1000000.0 * tup_deleted / (tup_returned + tup_inserted + tup_updated + tup_deleted)) / 10000
 else 0
 end::numeric(1000, 4) as delete_pct
from
  pg_stat_database d
right join
  pg_database on d.datname=pg_database.datname
where
  not datistemplate and d.datname != 'postgres';
```


### Size with toast

```
-- This seems wrong, not MECE, because it adds index size to tables, but then shows indexes.

SELECT
  nspname as schemaname,
  c.relname::text,
  case
    when
      c.relkind='r'
    then
      'table'
    when
      c.relkind='i'
    then
      'index'
    else
      lower(c.relkind)
  end as "type",
  pg_size_pretty(pg_relation_size(c.oid)) as "size",
  pg_size_pretty
  (
   case when c.reltoastrelid > 0
   then
   pg_relation_size(c.reltoastrelid)
   else 0 end
   +
   case when t.reltoastidxid > 0
   then
   pg_relation_size(t.reltoastidxid)
   else 0 end
  ) as toast,
  pg_size_pretty(cast((
    SELECT
      coalesce(sum(pg_relation_size(i.indexrelid)), 0)
    FROM
      pg_index i
    WHERE
      i.indrelid = c.oid
  )
  as int8)) as associated_idx_size,
  pg_size_pretty(pg_total_relation_size(c.oid)) as "total"
FROM
 pg_class c
LEFT JOIN
 pg_namespace n ON (n.oid = c.relnamespace)
LEFT JOIN
  pg_class t on (c.reltoastrelid=t.oid)
WHERE
  nspname not in ('pg_catalog', 'information_schema') AND
  nspname !~ '^pg_toast' AND
  -- c.relkind in ('r','i')
  c.relkind in ('r')
ORDER BY
  pg_total_relation_size(c.oid) DESC
LIMIT
  100;
```

### Table sizes

```
SELECT
    table_name,
    pg_size_pretty(table_size) AS table_size,
    pg_size_pretty(indexes_size) AS indexes_size,
    pg_size_pretty(total_size) AS total_size
FROM (
    SELECT
        table_name,
        pg_table_size(table_name) AS table_size,
        pg_indexes_size(table_name) AS indexes_size,
        pg_total_relation_size(table_name) AS total_size
    FROM (
        SELECT ('"' || table_schema || '"."' || table_name || '"') AS table_name
        FROM information_schema.tables
    ) AS all_tables
    ORDER BY total_size DESC
    LIMIT 10
) AS pretty_sizes;
```

### Table records estimation based on statistics

This is useful when the table is exclusively locked for example during the DDL
that modifies its structure:


```
select * from pg_stat_all_tables where relname = 'a';
```

### Indexes sizes

```
select
    pg_size_pretty(pg_table_size(indexrelid)),
    indexrelname index_name,
    relname table_name,
    idx_tup_read,
    idx_tup_fetch
from pg_stat_all_indexes
order by pg_table_size(indexrelid)
desc limit 20;
```


### Table and index hit %, hot tables (top 50)

```
SELECT
 schemaname::text,
 relname::text,
 heap_blks_read,
 heap_blks_hit,

 case when (heap_blks_hit + heap_blks_read) > 0
 then
 round(100 * heap_blks_hit / (heap_blks_hit + heap_blks_read))
 else 0 end as hit_pct,

 idx_blks_read,
 idx_blks_hit,

 CASE WHEN (idx_blks_hit + idx_blks_read) > 0
 then
 round(100 * idx_blks_hit / (idx_blks_hit + idx_blks_read))
 else 0 end as idx_hit_pct
FROM
  pg_statio_user_tables
WHERE
  (heap_blks_hit + heap_blks_read + idx_blks_hit + idx_blks_read) > 0
ORDER BY
  (heap_blks_read + heap_blks_hit) DESC
limit
  100;
```

### Read activity by table (top 50)

Break down total read activity in the database on a per-table basis.
Tthat's not completely fair because the size of the reads isn't considered,
but it does give a rough idea where activity is happening.

```
SELECT
  schemaname::text,
  relname::text,
  seq_tup_read,
  idx_tup_fetch,
  seq_tup_read + idx_tup_fetch as total_reads,
  round(100 * idx_tup_fetch / (seq_tup_read + idx_tup_fetch)) as idx_read_pct,
  pg_size_pretty(pg_total_relation_size(relid)) as total_size
FROM
  pg_stat_user_tables
JOIN
  pg_stat_database ON datname = current_database()
WHERE
  (seq_tup_read + idx_tup_fetch > 0) AND
  tup_returned > 0
ORDER BY
  (seq_tup_read + idx_tup_fetch) desc -- A.K.A. "total_reads"
limit
  50;
```

### Index usage counts

Rank how much all indexes are used, looking for unused ones.

```
SELECT
    schemaname::text,
    relname::text,
    indexrelname::text,
    idx_scan,
    pg_size_pretty(pg_relation_size(i.indexrelid)) as index_size

FROM
  pg_stat_user_indexes i
JOIN
  pg_index using (indexrelid)
WHERE
  indisunique is false
ORDER BY
  idx_scan desc, pg_relation_size(i.indexrelid) desc;
```

## Statistics

See http://www.postgresql.org/docs/9.2/static/monitoring-stats.html

### Reset statistics for current database

```
select pg_stat_reset(); -- resets all statistics just for the current database. 
```

`pg_stat_reset_shared('bgwriter')` can be used to reset `pg_stat_bgwriter`. 

`pg_stat_reset_single_table_counters()` and `pg_stat_reset_single_function_counters()` can be used to reset individual table, index, or function statistics.


### Buffer, background writer, and checkpoint activity

```
CREATE VIEW pg_stat_bgwriter AS
SELECT
    pg_stat_get_bgwriter_timed_checkpoints() AS checkpoints_timed,
    pg_stat_get_bgwriter_requested_checkpoints() AS checkpoints_req,
    pg_stat_get_bgwriter_buf_written_checkpoints() AS buffers_checkpoint,
    pg_stat_get_bgwriter_buf_written_clean() AS buffers_clean,
    pg_stat_get_bgwriter_maxwritten_clean() AS maxwritten_clean,
    pg_stat_get_buf_written_backend() AS buffers_backend,
    pg_stat_get_buf_alloc() AS buffers_alloc;



CREATE TABLE pg_stat_bgwriter_snapshot AS SELECT current_timestamp,* FROM pg_stat_bgwriter;

delete from pg_stat_bgwriter_snapshot;
INSERT INTO pg_stat_bgwriter_snapshot (SELECT current_timestamp,* FROM pg_stat_bgwriter);

SELECT
    cast(date_trunc('minute',start) AS timestamp) AS start,
    date_trunc('second',elapsed) AS elapsed,
    date_trunc('second',elapsed / (checkpoints_timed + checkpoints_req)) AS avg_checkpoint_interval,
    (100 * checkpoints_req) / (checkpoints_timed + checkpoints_req) AS checkpoints_req_pct,
    100 * buffers_checkpoint / (buffers_checkpoint + buffers_clean + buffers_backend) AS checkpoint_write_pct,
    100 * buffers_backend / (buffers_checkpoint + buffers_clean + buffers_backend) AS backend_write_pct,
    pg_size_pretty(buffers_checkpoint * block_size / (checkpoints_timed + checkpoints_req)) AS avg_checkpoint_write,
    pg_size_pretty(cast(block_size * (buffers_checkpoint + buffers_clean + buffers_backend) / extract(epoch FROM elapsed) AS int8)) AS written_per_sec,
    pg_size_pretty(cast(block_size * (buffers_alloc) / extract(epoch FROM elapsed) AS int8)) AS alloc_per_sec
FROM
(
    SELECT
        one.now AS start,
        two.now - one.now AS elapsed,
        two.checkpoints_timed - one.checkpoints_timed AS checkpoints_timed,
        two.checkpoints_req - one.checkpoints_req AS checkpoints_req,
        two.buffers_checkpoint - one.buffers_checkpoint AS buffers_checkpoint,
        two.buffers_clean - one.buffers_clean AS buffers_clean,
        two.maxwritten_clean - one.maxwritten_clean AS maxwritten_clean,
        two.buffers_backend - one.buffers_backend AS buffers_backend,
        two.buffers_alloc - one.buffers_alloc AS buffers_alloc,
        (SELECT cast(current_setting('block_size') AS integer)) AS block_size
    FROM pg_stat_bgwriter_snapshot one
        INNER JOIN pg_stat_bgwriter_snapshot two
    ON two.now > one.now
) bgwriter_diff
WHERE (checkpoints_timed + checkpoints_req) > 0;
```

## Buffer cache

```
create extension "pg_buffercache";
```

### Summary by usage count

```
SELECT usagecount,count(*),isdirty
FROM pg_buffercache
GROUP BY isdirty,usagecount
ORDER BY isdirty,usagecount;
```

### Buffer usage by table
```
SELECT c.relname, count(*) AS buffers
             FROM pg_buffercache b INNER JOIN pg_class c
             ON b.relfilenode = pg_relation_filenode(c.oid) AND
                b.reldatabase IN (0, (SELECT oid FROM pg_database
                                      WHERE datname = current_database()))
             GROUP BY c.relname
             ORDER BY 2 DESC
             LIMIT 10;
```

### Buffer content summary
```
SELECT 
    c.relname,
    pg_size_pretty(count(*) * 8192) as buffered,
    round(100.0 * count(*) /
    (SELECT setting FROM pg_settings
        WHERE name='shared_buffers')::integer,1) AS buffers_percent,
    round(100.0 * count(*) * 8192 / pg_relation_size(c.oid),1) AS percent_of_relation
FROM pg_class c
    INNER JOIN pg_buffercache b
    ON b.relfilenode = c.relfilenode
    INNER JOIN pg_database d
    ON (b.reldatabase = d.oid AND d.datname = current_database())
GROUP BY c.oid,c.relname
ORDER BY 3 DESC
LIMIT 10;
```

## Running sessions

```
select * from pg_stat_activity
```

## Running queries

```
SELECT pid, age(clock_timestamp(), query_start), usename, xact_start, wait_event, wait_event_type, state, query FROM pg_stat_activity
WHERE coalesce(state, '') != 'idle' AND query NOT ILIKE '%pg_stat_activity%'
ORDER BY query_start desc;
```

## <a name="kill"></a>Kill backend
Cancel query:

```
SELECT pg_cancel_backend(procpid);
```

Kill:
```
select pg_terminate_backend(<pid>)
```

## <a name="reload"></a>Reload config
```
SELECT pg_reload_conf();
```

## Locks 
```
select * from pg_locks
```

# System level monitoring

## Overall
```
vmstat 1
```

## IO
```
iostat -k 2
```

# Queries

## Get table indices

```
select tablename,indexname,tablespace,indexdef  from pg_indexes where tablename = ''
```

## Get table columns

```
select column_name, data_type, character_maximum_length, column_default, is_nullable from INFORMATION_SCHEMA.COLUMNS where table_name = ''
```
