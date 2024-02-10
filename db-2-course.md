# SQL Snippets



> [2024/02/10] | [00:21:00]
> DB2 — Chapter 02 — Video #02 — Populating tables (generate_series()), sequential scan



---

### Unary Table Storage

Retrieve all rows (in some arbitrary order) and all columns of table `unary`. For now, we assume that unary has a single column of type `int`.

```sql
SELECT	u.* 			-- * === access all columns of row u
FROM 	unary AS u
```

Create and populate table `unary` as follows

```sql
CREATE TABLE unary (a int);

INSERT INTO unary(a)
	SELECT i
	FROM generate_series(1, 100, 1) AS i;
--- 					 ^   ^   ^
---					start / end / step of sequence
```



Generate a single-column table of timestamp values `i`, starting now with step width 1 minute

```sql
SELECT  i
FROM 	generate_series('now'::timestamp,
						'now'::timestamp + '1 hour',
						'1 min') AS i;
```



Create and populate single-column table unary:

```sql
DROP TABLE IF EXISTS unary;
CREATE TABLE unary (a int);
```



Table `unary` will hold 100 rows of integers (recall: there is NO guaranteed row order in a relational table)

```sql
INSERT INTO unary(a)
	SELECT i
	FROM generate_series(1, 100, 1) AS i;
```

Reproduce all rows and all (here: the single) columns of table `unary` 

```Sql
SELECT 	u.*
FROM 	unary AS u;

-- the simpler way of doing the same: getting all the rows from table
TABLE unary;
```



Now use the DBMX x-ray! PostgreSQL explains the plan it will use to evaluate the query

```sql
EXPLAIN VERBOSE
	SELECT u.*
	FROM unary AS u;
```

Show the query evaluation plan for SQL query

1. `EXPLAIN <opt> <Q>`
2. `EXPLAIN (<opt>, <opt>, ...) <Q>`

`<opt>` controls level of detail and mode of explanation

1. `VERBOSE` higher level of detail
2. `ANALYZE` evaluate the query, then produce explanation
3. `FORMAT {TEXT|JSON|XML}` output format (default: `TEXT`)

Without `ANALYZE`, `<Q>` is not evaluated. Output is based on DBMS's best guess of how the plan will perform.

```sql
EXPLAIN (VERBOSE, ANALYZE)
	SELECT u.*
	FROM unary AS u;
```



#### Sequential Scan (Seq Scan)

Seq Scan: Sequentially scan the entire heap file of table `unary`, read row in some order, emit all rows.

Seq Scan returns rows in arbitrary order (not: insertion order) that may change from execution to execution. Meets bag semantics of the tabular data model.



#### Heap Files

The rows of a table are stored in its heap file, a plain row container that can grow / shrink dynamically.

* Row insertion / deletion simple to implement and efficient, no complex file structure to maintain.
* Supports sequential scan across entire file.
* No support for finding rows by column value (no associative row access). If we need value-based row access, additional data maps (indexes) need to be created and maintained.



#### Heap Files and Sequential Scan

The DBMS may reorganize (e.g. compact or 'vacuum') a table's heap file at any time. No guaranteed row order.























---





