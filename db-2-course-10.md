# SQL Snippets



> [2024/02/24] | [00:30:00]
> DB2 — Chapter 04 — Video #11 — PostgreSQL row internals, meta/payload data, column padding, "Tetris"

---

### Chapter 4: Row Internals

-- **PostgreSQL** --



```sql
-- Create three-column table ternary and populate it with 1000 rows
-- NB: every 10th row carries a NULL value in column c
DROP TABLE IF EXISTS ternary;
CREATE TABLE ternary (
	a int 		NOT NULL, 	-- 4 byte integer
    b text 		NOT NULL, 	-- variable width
    c float
);

INSERT INTO ternary(a, b, c)
	SELECT 	i 												AS a,
			md5(i::text) 									AS b,
			CASE WHEN i % 10 = 0 THEN NULL ELSE log(i) END 	AS c 	-- place NULL in every 10th
	FROM 	generate_series(1, 1000, 1) as i;
	
ANALYZE ternary;

-- Probe query Q3: Retrieve all rows (in arbitrary order) but
-- retrieve columns a, c only (column b "projected away")
SELECT 	t.a, t.c
FROM 	ternary AS t;



```



#### Internal Layout of a PostgreSQL Row

NULL bitmap is of variable length (1 bit per column), offset `t_hoff` points to first byte of row payload data.

NB: `EXPLAIN`'s `width=w` reports payload bytes only.



#### Padding and Alignment

CPU and memory sybsystem require alignment: value of width `n` bytes is stored at address `a` with `a mod n = 0`

Pad payload such that each column starts at properly aligned offset (PostgreSQL: see table `pg_attribute`).

- Non-aligned data access incur performance penalties (multiple accesses) or even exceptions.



```sql
-- Check PostgreSQL's system catalog for the widths (attlen) of the columns of table ternary and
-- check for memory alignement requirements (attalign)

SELECT 	a.attnum, a.attname, a.attlen, a.attstorage, a.attalign
FROM 	pg_attribute AS a
WHERE 	a.attrelid = 'ternary'::regclass AND a.attnum > 0
ORDER BY a.attnum;
```



#### Column Tetris

Padding may lead to substantial space overhead. If viable, reorder columns to tightly pack rows and avoid padding. 

```sql
CREATE TABLE padded (
	d int2,
    a int8,
    e int2,
    b int8,
    f int2,
    c int8
);
-- column data width 48

CREATE TABLE packed (
	a int8, 		-- int8: 8-byte aligned
    b int8,
    c int8,
    d int2, 		-- int2: 2-byte aligned
    e int2,
    f int2
);
-- column data width 30 (+2) 

```

+2: Rows start at `MAXALIGN` offsets ( `===` 8 on 64-bit CPUs).



```sql
-- "Column Tetris"

-- Create and populate two tables that carry equivalent information.
-- Table packed arranges columns such that column values need NO PADDING
-- on the tables's heap file pages. Do we see any benefit? (You bet!)

DROP TABLE IF EXISTS padded;
DROP TABLE IF EXISTS packed;
CREATE TABLE padded (
	d int2,
    a int8,
    e int2,
    b int8,
    f int2,
    c int8
);
CREATE TABLE packed (
	a int8,
    b int8,
    c int8,
    d int2,
    e int2,
    f int2
);

INSERT INTO 	padded(d,a,e,b,f,c)
	SELECT 	0, i, 0, i, 0, i
	FROM 	generate_series(1, 1000000) AS i;
	
INSERT INTO 	padded(a,b,c,d,e,f)
	SELECT 	i, i, i, 0, 0, 0
	FROM 	generate_series(1, 1000000) AS i;
	
-- Check how many pages are needed to store the rows of tables padded & packed
VACUUM padded;
VACUUM packed;

SELECT 	COUNT(*)
FROM 	pg_freespace('padded');

SELECT 	COUNT(*)
FROM 	pg_freespace('packed');

-- Check how many rows are found on each page of tables padded & packed
SELECT 	lp, lp_off, lp_len, t_hoff, t_ctid
FROM 	heap_page_items(get_raw_page('padded', 0));

SELECT 	lp, lp_off, lp_len, t_hoff, t_ctid
FROM 	heap_page_items(get_raw_page('packed', 0));

-- Can the query processor benefit (see cost output in plan)?
EXPLAIN VERBOSE
	SELECT 	p.*
	FROM 	padded AS p;
	
EXPLAIN VERBOSE
	SELECT 	p.*
	FROM 	packed AS p;
```















