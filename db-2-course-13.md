# SQL Snippets



> [2024/05/06] | [00:19:00]
> DB2 — Chapter #05 — Video #15 — INSERT/UPDATE/DELETE plans in PostgreSQL, how to read plans

---

### Chapter 5: Row Updates

-- **PostgreSQL** --



`INSERT`: evaluate query to construct new rows

`UPDATE`, `DELETE`: query the table and identify the affected rows.

Modify table storage to reflect the row updates (we still assume that the table has no associated index structures)



#### Using EXPLAIN on Q4: INSERT



```sql
-- Q4: Perform row insertions / updates / deletions (no ANALYZE here, don't alter the table)
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


EXPLAIN VERBOSE
	INSERT INTO ternary(a, b, c)
		SELECT 	t.a, 'Han Solo', t.c
		FROM 	ternary AS t;
```

* Seq Scan scans table `ternary` to construct rows to be inserted, feeds 1000 rows into Insert for insertion.
* Width of inserted rows (over-)estimated to be 44 bytes = 4 (int) + 32 (text) + 8 (float) bytes.



---

#### Reading Complex EXPLAIN Outputs

* EXPLAIN uses symbol `->` and indentation to visualize larger, **tree-shaped** query evaluation plains
* Read plans "inside out", root delivers query result.

---

#### Using EXPLAIN on Q4: UPDATE

```sql
EXPLAIN VERBOSE
	UPDATE ternary AS t
	SET 	c = -1
	WHERE 	t.a = 982;
```

* Seq Scan emits complete rows (only `c` was updated)
* Additionally feeds row ID (`ctid`) into Update to identify the affected row(s).

---

#### Using EXPLAIN on Q4: DELETE

```sql
EXPLAIN VERBOSE
	DELETE FROM ternary AS t
	WHERE 	t.a = 982;
```

* Seq Scan returns affected row IDs (of 6 bytes each) only.
* We turn to Filter (makes scan skip non-qualifying rows) later in this course.











