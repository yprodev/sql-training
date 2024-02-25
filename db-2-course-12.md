# SQL Snippets



> [2024/02/25] | [00:29:00]
> DB2 — Chapter #04 — Video #13 — Column projection in PostgreSQL, routine slot_get_attr()

---

### Chapter 4: Row Internals

-- **PostgreSQL** --



#### Column Access (Projection)

If `t` denotes a row, column access - denoted using dot notation `t.a` - is the most common operation in SQL query expressions.

A typical SQL query will perform multiple column accesses per row (in `SELECT`, `WHERE`, `GROUP BY`, ... clauses), potentially millions of times during evaluation of a single query.

Even tiny savings in processing effort (here: CPU time) will add up and can lead to substantial benefits.

This is a recurring theme in DBMS implementation. The larger the table cardinalities, the more worthwhile "micro optimizations" become.



#### Column Access: PostgreSQL's `slot_getattr()`

See PostgreSQL source code (a prime example of readable, consistent, well-documented C code)

`src/backend/access/common/heaptuple.c`

`slot_deform_tuple()` does the hard decoding work



```sql
-- Experiment: how much time does PostgreSQL spent in routine
-- slot_deform_tuple() and slot_getattr() when a SQL query is
-- being processed? (Spoiler: A LOT!)

-- Create and populate a larger version of table ternary (10M rows)
-- (evaluation of the INSERT may take some time and disk space)
DROP TABLE IF EXISTS ternary_10M;
CREATE TABLE ternary_10M(
	a int 	NOT NULL,
    b text 	NOT NULL,
    c float
);

INSERT INTO ternary_10M(a, b, c)
	SELECT 	i 												AS a,
			md5(i::text) 									AS b,
			CASE WHEN i % 10 = 0 THEN NULL ELSE log(i) END 	AS c 	-- place NULL in every 10th
	FROM 	generate_series(1, 10000000, 1) as i;

```



```bash
# A UNIX shell command to monitor all PostgreSQL DBMS kernel processes and list their CPU activity

eval (echo top -stats pid,command,cpu -pid (pgrep -d ' -pid ' -f postgres | sed 's/-pid $//'))
```



```sql
-- How is time spent when PostgreSQL performs this query on the 10M table?
EXPLAIN (VERBOSE, ANALYZE)
	SELECT 	t.b, t.c
	FROM 	ternary_10M AS t;
```



```bash
# Use sample command to see which postgres functions will be activated while running a SQL query
sample postgres 10

# You will get the list of processes. Pick the one you got on the previous step, while running EXPLAIN query
# Then run EXPLAIN query once again
```



#### Alternative Layout of Row Payload: Fixed-Width First

Separate fixed- from variable-width payload data at

- fixed-width columns `a`, `c` (types `int`, `double`) + fixed-width pointers `b*` to variable-width columns
- variable-width value for column `b` (type `text`)

Can calculate offsets of fixed-width columns at query compile time, no left-to-right scanning at run time.



```sql
DROP TABLE IF EXISTS ternary_10M;
CREATE TABLE ternary_10M(
	a int 	NOT NULL,
    b text 	NOT NULL,
    c float
);

INSERT INTO ternary_10M(a, b, c)
	SELECT 	i 												AS a,
			md5(i::text) 									AS b,
			CASE WHEN i % 10 = 0 THEN NULL ELSE log(i) END 	AS c 	-- place NULL in every 10th
	FROM 	generate_series(1, 10000000, 1) as i;
	
-- Does the evaluation time for an arbitrary projection (column order c, b, a) differ from the
-- retrieval in storage order (a, b, c)?
EXPLAIN (VERBOSE, ANALYZE)
	SELECT 	t.b, t.c, t.a
	FROM 	ternary_10M AS t;
	
EXPLAIN (VERBOSE, ANALYZE)
	SELECT 	t.* 					-- also OK: t.a, t.b, t.c === t.*
	FROM 	ternary_10M AS t; 		-- t.* is supported by a fast-path in PostgreSQL C code
```







