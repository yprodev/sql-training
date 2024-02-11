# SQL Snippets



> [2024/02/11] | [00:24:00]
> DB2 — Chapter 02 — Video #05 — Free space management (FSM trees)

---

### Unary Table Storage



#### Heap Files: Free Space Management

Row updates and deletions may lead to heap file pages that not 100% filled. New records could fill such "holes".

- DBMS maintains a free space map (FSM) for each heap file, recording the (approximate) number of bytes available on each 8 kB page.

Required FSM operations

1. Given a row of n bytees, which page `p` (in the vicinity) has sufficient free space to hold the row?
2. Free space on page `p` has been reduced / enlarged by `n` bytes. Update the FSM.



PostgreSQL maintains a tree-shaped FSM for each heap file:

```bash
						8
				   /         \
				 8              5
			  /     \ 	     /     \
			 1       8      3       5       # < Maximal space available in subtree
			/ \     / \    / \     / \
		   0   1   0   8  2   3   4   5     # < Space available on page 7
		   0   1   2   3  4   5   6   7     # page
```



- Leaf nodes: space available in heap file page.
- Inner nodes: maximal space found in this file (segment)

PostgreSQL: space measured in 32 byte units (= 1 / 256 of a 8 kB page)



1. Find a page with at least 4 available slots in the vicinity of page #4 (traverses 2^ 3^ 5v 5v 4).
2. Update page #4 to provide 4 available slots (traverses, updates 3 to max(3, 4) = 4, stops when max(4,5) = 5)



```sql
-- Enable a PostgreSQL extension that enables us to peek inside the
-- free space maps (FSMs) maintainted for each heap file:
CREATE EXTENSION IF NOT EXISTS pg_freespacemap;

-- (Re-)create table unary with 1000 rows of integers:
DROP TABLE IF EXISTS unary;
CREATE TABLE unary (a int);
INSERT INTO unary(a)
	SELECT 	i
	FROM 	generate_series(1, 1000, 1) as i;
```

```bash
# Locate the directory and heap file associated with table unary of
# database scratch. The FSM of the heap file has suffix '_fsm'.
show data_directory
```

```sql
SELECT 	oid, datname
FROM	pg_database
WHERE 	datname = 'scratch';
```

```sql
SELECT 	relfilenode, relname
FROM 	pg_class
WHERE 	relname = 'unary';
```

```sql
-- Query the leaf nodes of the current free space map for table unary
-- to check how many bytes are available on the heap file pages:
VACUUM unary;

SELECT  *
FROM 	pg_freespace('unary');
```

```sql
-- Delete a range of rows from table unary and recheck the available
-- space on the heap file pages:
DELETE FROM unary AS u
WHERE u.a BETWEEN 400 AND 500;
```

```sql
-- ... check the values
VACUUM unary;

SELECT  *
FROM 	pg_freespace('unary');
```

```sql
-- Insert a sentinel value into table unary so that we can easily
-- spot where the DBMS will place the new row in the heap file:
INSERT INTO unary(a) VALUES (-1);

TABLE unary;
```

```sql
-- ... check the values
VACUUM unary;

SELECT  *
FROM 	pg_freespace('unary');

-- ...check on which page the value has been placed
-- u.ctid - row id -> RID
SELECT  u.ctid, u.a
FROM 	unary AS u;
```

```sql
-- Perform a full vacuum run on table unary to reorganize the rows in the
-- heap file such that now space is wasted (remove all "holes" on the
-- pages):
VACUUM (VERBOSE, FULL) unary;

SELECT  *
FROM 	pg_freespace('unary');
```

```sql
-- Such a full vacuum run in fact creates a new heap file and marks the
-- old heap file as obsolete (OS file system can remove teh file):
SELECT  relfilenode, relname
FROM 	pg_class
WHERE 	relname = 'unary';
```





















