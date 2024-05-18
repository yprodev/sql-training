# SQL Snippets



> [2024/05/06] | [00:31:00]
> DB2 — Chapter #05 — Video #16 — PostgresQL: row versions, xmin/xmax, timestamps, VACUUM

---

### Chapter 5: Row Updates

-- **PostgreSQL** --



#### How do Row Updates Alter the Table Storage

Let us take a closer look at how plan operator Update alters the target table's heap file pages. We find (this implementation of Update is typical for all DBMS that implement Muti-Version Concurrency Control - MVCC. We discuss MVCC later in this course):

* Rows are not updated in-place. A new version of the row is created - original and updated row co-exists.
* Any database user (query, application) sees exactly one version of any row at any time. Different users may see different row versions.
* A separate `VACCUM` ("garbage collection") step collects and removes old version that cannot be seen by any user.

---

#### Row Version Chains

* Original and updated versions of a row form a chain, linked by the rows' IDs (held in row header field ctid).

---

#### Row Visibility and Timestamps

1. Each row carries two timestamps - `xmin` and `xmax` - that mark its first and last time of existance.
2. Each query / update is executed at some timestamp `T` which defines the rows that are visible to the operation:

Row is visible for any operation with timestamp: `t0 < T < t1 .`If `t1 = infinity` the row has not been updated yet. 

DBMS uses system-wide virtual timestamps (transaction IDs), see PostgreSQL built-in function `txid_current()`.

```sql
-- 1 list rows on page 9 (table occupies 9 heap file pages)
SELECT 	t.ctid, t.*
FROM 	ternary AS t
WHERE 	t.ctid >= '(9,1)';

-- 2 check row header contents before update
CREATE EXTENSION pageinspect; -- this is needed to have abitlity to call get_raw_page() function

SELECT  t_ctid, lp, lp_off, lp_len, t_xmin, t_xmax,
		(t_infomask::bit(16) & b'0010000000000000')::int::bool 	AS "updated row?",
		(t_infomask2::bit(16) & b'0100000000000000')::int::bool 	AS "has been HOT updated?"
FROM 	heap_page_items(get_raw_page('ternary', 9));


-- 3 check current transaction ID (= virtual timestamp)
SELECT txid_current(); -- the value would be a global counter of the transactions

-- 4 update one row
UPDATE 	ternary AS t
SET 	c = -1
WHERE 	t.a = 982;

```



---

#### Impact of Updates Beyond the Row's Page

* Updates on full pages may lead to row relocation across pages.
  * Traversal of longer update chains may lead to I/O-costly "page hopping".
  * Perform `VACCUM` to collect inaccessible old version. From outside page, point to most recent row directly.
* PostgreSQL optimizes for the good-natured case where `pi = pi+1` and indexed row fields have not been changed.
  * Such heap-only tuple (HOT) updated have page-internal impact only, no maintenance outside page required.

