# SQL Snippets



> [2024/02/11] | [00:21:00]
> DB2 — Chapter 02 — Video #06 — MonetDB: BATs, MAL, (v)oid head columns

---

### Unary Table Storage

-- **MonetDB** --

#### Aside: Populating Tables via generate_series()

One way to create and populate table `unary` in MonetDB

```sql
CREATE TABLE unary (a int);

INSERT INTO unary(a)
	SELECT  value 		-- fixed column name
	FROM 	generate_series(1,     101, CASE(1 AS int));
-- 							^       ^        ^
-- 						start  /  end + 1 / step of sequence
```

```sql
-- Create our playground table unary in MonetDB as usual:
DROP TABLE IF EXISTS unary;
CREATE TABLE unary (a int);

-- Populate the table with 100 integer rows (generate_series(s, e))
-- generates values less than e, default step delta is 1
INSERT INTO unary(a)
	SELECT  value
	FROM 	generate_series(1, 101);
	
-- Check table contents:
SELECT  u.*
FROM 	unary AS u;

-- MonetDB's plans for a SQL query are MonetDB Assembly Language (MAL)
-- programs:
EXPLAIN
	SELECT  u.*
	FROM 	unary AS u;

```

Queries are compiled into (mostly) linear MonetDB Assembly Language (MAL) programs. Program === sequence of assignment statements.

The MonetDB kernel implements a MAL virtual machine (VM).

MonetDB implements a single collection type `bat[:t]`, the Binary Association Tables (BATs) of values of type `t`:

1. Head: store sequence base `0@0` only ('virtual oids', `void`)
2. Tail: one ordered column (or vector) of data

```sql
-- [WARNING] Below we're talking MAL (not SQL). Start MonetDB mclient session
-- 			 with language flat '-l msql'

-- Create a new empty BAT and assign it to MAL variable t:
t := bat.new(nil:int);

-- Populate BAT t with integer values and check BAT contents (io.print)
bat.append(t, 42);
bat.append(t, 42);
bat.append(t, 0);
bat.append(t, -1);
bat.append(t, nil:int);
io.print(t);

-- BATs admit positional access to rows (fetch row at offset 3):
v := algebra.fetch(t, 3@0);
io.print(v);

-- Replace integer value in row at offset 4:
ba t.replace(t, 4@0, 2);
io.print(t);

-- Extract positional slice from BAT (from offset 1 to 3). The result
-- is a new BAT assigned to t1:
t1 := algebra.slice(t, 1@0, 3@0);

-- In t, offsets (oids) are counted from 0, in t1 they are counted from 1:
b := bat.getSequenceBase(t);
io.print(b);
b := bat.getSequenceBase(t1);
io.print(b);

-- Deleting a row from a BAT leaves no holes (move last row in hole
-- left by deleted row):
bat.delete(t, 1@0);
io.print(t);


```























