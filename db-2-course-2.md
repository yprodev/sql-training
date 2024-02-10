# SQL Snippets



> [2024/02/10] | [00:25:00]
> DB2 — Chapter 02 — Video #03 — Heap files, row identifiers (RIDs)

---

### Unary Table Storage

#### Heap File === OS File

Most DBMSs store heap files in regular files or the operating system's file system (alternative: raw storage).

- Files held in a DBMS-controlled directory. In PostgreSQL

```sql
SHOW data_directory;
```

- DBMS enjoys OS FS services (e.g., backup, authorization)



Query PostgreSQL's system catalog for all current databases maintained by the DBMS

```sql
SELECT 	oid, datname
FROM 	pg_database;
```



From the system catalog, retrieve the names of all tables as well as the heap file names (column relfilenode) that hold the table data

```sql
SELECT 	relfilenode, relname
FROM 	pg_class
ORDER BY relname DESC;
```



Create and populate a second `unary` table holding textual data that we will able to identify when we peek inside the table's heap file

```sql
DROP TABLE IF EXISTS "unary'";
CREATE TABLE "unary'" (a text);

INSERT INTO "unary'" VALUES ('Yoda'), ('Han Solo'), ('Leia'), ('Luke');

TABLE "unary'";

SELECT 	relfilenode, relname
FROM 	pg_class
WHERE 	relname = 'unary''';
```

[NOTE] You may use terminal utility command `hexdumpt` to see the internals of the DBMS file

```bash
hexdump -C <name of the table file>
```



#### Row IDs and Heap File Locations

Heap files do not support value-based access. We can still directly locate a row via its row identifier (RID):

* RIDs are unique within a table. Even if two rows `r1`, `r2` agree on all column values (in a key-less table), we still have `RID(r1) != RID(r2)`
* `RID(r)` encodes the location of row `r` in its table's heap file. No sequential scan is required to access `r`.
* If `r` is updated, `RID(r)` remains stable.

RIDs do not replace the relational key concept.



Now remove all rows from our original table unary and re-populate it with 1000 rows of integer data

```sql
TRUNCATE unary;

INSERT INTO unary(a)
SELECT 	i
FROM 	generate_series(1, 1000, 1) AS i;

SELECT 	relfilenode, relname
FROM 	pg_class
WHERE 	relname = 'unary';
```



#### RIDs in PostgreSQL

RIDs are considered DBMS-internal and thus withheld from users. PostgreSQL externalizes RIDs via pseudo-column `ctid`

Access all rows and all columns in a table `unary` AND ALSO output the row IDs (RIDs, pseudo-column `ctid`) of all rows

```sql
-- Access all rows and all columns in a table `unary` AND ALSO output the row IDs (RIDs, pseudo-column `ctid`) of all rows
SELECT 	u.ctid, u.*
FROM 	unary AS u;
```



#### File Storage on Disk-Based Secondary Memory

A PostgreSQL RID is a pair (<page number>, <row slot>)

- Page number `p` identifies a contiguous block of bytes in the file
- Page size `B` is system-dependent and configurable. Typical values are in range 4-64 kB. PostgreSQL default: 8 kB.



#### Block I/O Disk-Based Secondary Memory

- Heap files are read and written in units or 8 kB pages. Likewise, heap files grow / shrink by entire pages.
- This page-based access to heap files reflects the OS's mode of performing disk input / output page-by-page. Terminology: DB page === block OS
- Any disk I/O operation will read / write at least one block (of 8 kB). Disk I/O NEVER moves individual bytes.



```sql
SHOW data_directory;

-- get the path to the data_directory and run command
pg_controldata '<path to data directory>'; -- run it in bash

```

[NOTE] Find `frink` terminal tool to count bytes in terminal.

[NOTE] `frink` Alan Eliasen copyright

 

































---





