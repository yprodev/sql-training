# SQL Snippets



> [2024/05/18] | [00:28:00]
> DB2 — Chapter #06 — Video #21 — Dynamic buffer behavior observed via pg_buffercache

---

### Chapter 6: Buffering and Caching

-- **PostgreSQL** --



#### Inspect Dynamic Buffer Behavior

PostgreSQL offers extension `pg_buffercache`, providing a tabular view (NB: This is only a tabular representation of the buffer descriptors. Internally, the buffer and its descriptors are implemented as C arrays) of the system's buffer cache descriptors:

```sql
SELECT  b.bufferid, b.relblocknumber, b.isdirty, b.usagecount
FROM 	pg_buffercache AS b
[ WHERE b.relfilenode = <tbl> ]; -- focus on table <tbl> only
```



#### EXPLAIN: Buffer Hits and Misses

`EXPLAIN` can be instructed to show whether the DBMS experienced buffer hits or misses during query evaluation:

```sql
EXPLAIN (ANALYZE, BUFFERS, <opt>, ...) <Q>
```



```sql
-- Make sure the extension was added after you will restart the db server
CREATE EXTENSION IF NOT EXISTS pg_buffercache;

-- Experiment with a looooong series of SQL queries below

-- Prepare a large variant of the ternary table to help demonstrate
-- the dynamic buffer behavior:
DROP TABLE IF EXISTS ternary_100k;
CREATE TABLE ternary_100k (a int NOT NULL, b text NOT NULL, c float);
INSERT INTO ternary_100k(a, b, c)
	SELECT  i,
			md5(i::text),
			log(i)
	FROM 	generate_series(1, 100000, 1) AS i;
	
-- NOW RESTART THE POSTGRESQL SERVER TO FLUSH THE BUFFER CACHE


-- Which heap file contains the pages of table ternary_100k and how
-- many blocks / pages does the heap file occupy?
SELECT 	c.relfilenode, c.relpages
FROM 	pg_class AS c
WHERE 	c.relname = 'ternary_100k';
-- Save relfilenode into psql variable :relfilenode (used below)
\gset

-- Check that the buffer cache currently holds no pages of
-- table ternary_100k
SELECT  b.bufferid, b.relblocknumber, b.isdirty, b.usagecount
FROM	pg_buffercache AS b
WHERE b.relfilenode = :relfilenode;


-- Now scan pages of ternary_100k. Expect buffer cache MISSES for
-- all pages:
EXPLAIN (VERBOSE, ANALYZE, BUFFERS)
	SELECT  t.*
	FROM 	ternary_100k AS t;

-- Check buffer cache for pages of ternary_100k: all pages in, not dirty,
-- usagecount = 1:
SELECT  b.bufferid, b.relblocknumber, b.isdirty, b.usagecount
FROM 	pg_buffercache AS b
WHERE 	b.relfilenode = :relfilenode;

-- Re-scan all pages of ternary_100k. Now we see buffer HITS for all pages:
EXPLAIN (VERBOSE, ANALYZE, BUFFERS)
	SELECT  t.*
	FROM 	ternary_100k AS t;
	
-- Re-heck buffer cache for pages of ternary_100k: all pages in, not dirty,
-- usagecount = 2:
SELECT  b.bufferid, b.relblocknumber, b.isdirty, b.usagecount
FROM 	pg_buffercache AS b
WHERE 	b.relfilenode = :relfilenode;

-- Scan all pages of ternary_100k with a < 100: buffer cache HITS for
-- all pages (since no index/ordering on the table):
EXPLAIN (VERBOSE, ANALYZE, BUFFERS)
	SELECT  t.*
	FROM 	ternary_100k AS t
	WHERE 	t.a < 100;

-- Re-heck buffer cache contents for pages of ternary_100k: all pages in,
-- not dirty, usagecount = 3:
SELECT  b.bufferid, b.relblocknumber, b.isdirty, b.usagecount
FROM 	pg_buffercache AS b
WHERE 	b.relfilenode = :relfilenode;

-- Now update a row in ternary_100k. START TRANSACTION such that we can
-- observe things while they are in progress
START TRANSACTION;

-- Update row with a = 10: buffer cache hits for all pages, two pages dirty:
EXPLAIN (VERBOSE, ANALYZE, BUFFERS)
	UPDATE  ternary_100k
	SET 	c = -1
	WHERE 	a = 10; -- affected row on block 0 of ternary_100


-- FOR THE NEXT QUERY YOU NEED TO HAVE EXTENSION ADDED
CREATE EXTENSION pageinspect; -- this is needed to have abitlity to call get_raw_page() function

-- Check page contents of pages for old and new row version,
-- note the updated row slots:
SELECT  t_ctid, lp, lp_off, lp_len, t_xmin, t_xmax,
		(t_infomask::bit(16) & b'0010000000000000')::int::bool 	AS "updated row?",
		(t_infomask2::bit(16) & b'0100000000000000')::int::bool AS "has been HOT updated?"
FROM 	heap_page_items(get_raw_page('ternary_100k', 0)); -- page 0 holds old row version

-- Now commit the UPDATE change
COMMIT;


-- Re-check buffer cache contents for pages of ternary_100k: all pages
-- in, usage_count of pages 0 and 934 higher:
SELECT  b.bufferid, b.relblocknumber, b.isdirty, b.usagecount
FROM 	pg_buffercache AS b
WHERE 	b.relfilenode = :relfilenode; -- AND b.isdirty


-- Re-check buffer cache contents for pages of ternary_100k: all pages
-- in, usage_count of pages 0 and 934 higher:
SELECT  b.bufferid, b.relblocknumber, b.isdirty, b.usagecount
FROM 	pg_buffercache AS b
WHERE 	b.relfilenode = :relfilenode AND b.isdirty;

-- After a forced CHECKPOINT, all buffers are synced with disk image
CHECKPOINT;

SELECT  b.bufferid, b.relblocknumber, b.isdirty, b.usagecount
FROM 	pg_buffercache AS b
WHERE 	b.relfilenode = :relfilenode AND b.isdirty;
```















