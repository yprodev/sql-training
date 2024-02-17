# SQL Snippets



> [2024/02/17] | [00:40:00]
> DB2 — Chapter 03 — Video #09 — Row storage in PostgreSQL, heap file page layout

---

### Chapter 3: Wide Table Storage

-- **PostgreSQL** --

```sql
SELECT 	t.* 			-- access all columns of row t
FROM 	ternary AS t
```

Retrieve all rows (in some arbitrary order) and all columns of table `ternary`, a three-column table craeted by this SQL DDL statement

```sql
CREATE TABLE ternary (a int NOT NULL,
                      b text NOT NULL, 	-- variable width
                      c float); 		-- may be NULL
```

```sql
-- Create a three-colun (i.e., wide) table and populate it
-- with 1000 rows
DROP TABLE IF EXISTS ternary;
CREATE TABLE ternary (
	a int NOT NULL,
    b text NOT NULL,
    c float
);

INSERT INTO ternary(a, b, c)
	SELECT 	i,
			md5(i::text), 		-- MD5 hash: 32-character string
			log(i)
	FROM generate_series(1, 1000, 1) AS i;

ANALYZE ternary;

-- Q2: Retrieve all rows (in arbitrary order) and all columns of table ternary
SELECT 	t.*
FROM 	ternary AS t;

-- Equivalent to Q2
TABLE ternary;

-- Sequential scan delivers wider rows now, width: 45 bytes
-- (cf. sequential scan over table unary, width: 4 bytes):
EXPLAIN VERBOSE
SELECT 	t.*
FROM 	ternary AS t;

```



#### PostgreSQL: Row Storage

PostgreSQL implements row storage: all columns of a row `t` are held in contiguous memory (the same heap file page).

Loading one heap file page reads all columns of all contained rows (recall: block I / O), regardless of whether the query uses `t.*` or `t.a` to access the row.



#### The Innards of a Heap File Page

Comments on the previous slide:

- Page header (24 bytes) carries page meta information.
- Special space is unused on regular table heap file pages (but used on index pages - later)
- Page is full if pointers lower and upper meet (row pointers and payload grow towards --> <-- each other)
- Row pointer (or: line pointer, 4 byte) points to row, admits variable-width rows and intra-page row relocation (row updates)
- Internal structure of row payloads addressed later



PostgreSQL comes with extension `pageinspect` that provides an "X-ray for heap file pages":

```sql
-- Enable a PostgreSQL extension that enables us to peek inside the
-- pages of a heap file (page header, row pointers_)
CREATE EXTENSION IF NOT EXISTS pageinspect;

-- inspect page header (first 24 bytes)
SELECT 	*
FROM 	page_header(get_raw_page('ternary', <page>));

-- inspect row pointers
SELECT 	*
FROM 	heap_page_items(get_raw_page('ternary', <page>));

```

```sql
-- Enable a PostgreSQL extension that enables us to peek inside the
-- pages of a heap file (page header, row pointers_)
CREATE EXTENSION IF NOT EXISTS pageinspect;

-- All rows of ternary, together with their RID (RID === (<page>, <i>)),
-- where <i> identifies the row's row / line pointer
SELECT 	t.ctid, t.*
FROM 	ternary AS t;

-- Check the page headers of heap file pages 0 (first page)
-- and 9 (last page of file)
SELECT 	*
FROM 	page_header(get_raw_page('ternary', 0));

SELECT 	*
FROM 	page_header(get_raw_page('ternary', 9));
```



Get the position of the lower pointer (172) and minus the number of bytes the pointer holds (24) and divide the result by 4 (4 bytes) - each line pointer occupies 4 byes, and you will find the number of pointers on the page.

```sql
SELECT (172 - 24) / 4;
```



```sql
-- Enable a PostgreSQL extension that enables us to peek inside the
-- free space maps (FSMs) maintainted for each heap file:
CREATE EXTENSION IF NOT EXISTS pg_freespacemap;

-- Cross-checking our findings on the pages with free space management
-- information
VACUUM 	ternary;
SELECT 	*
FROM 	pg_freespace('ternary');

-- Inspect the row pointers (of heap file page 0)
SELECT 	lp, lp_off, lp_len, t_hoff, t_ctid, t_infomask :: bit(16), t_infomask2
FROM 	heap_page_items(get_raw_page('ternary', 0));
```



```bash
# Cross-check with in on-disk heap file for contents of the 
# third row (lp = 3) on page 0
show data_directory; 	# locate in-filesystem representation of PostgreSQL

```

```sql
SELECT 	oid 						-- identify directory of database 'scratch'
FROM 	pg_database
WHERE 	datname = 'scratch';

SELECT 	relfilenode
FROM 	pg_class
WHERE 	relname = 'ternary';

SELECT 	t.ctid, t.*
FROM 	ternary AS t
LIMIT 	3;

```



















 













