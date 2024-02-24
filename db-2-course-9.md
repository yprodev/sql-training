# SQL Snippets



> [2024/02/24] | [00:23:00]
> DB2 — Chapter 03 — Video #10 — Column storage in MonetDB, NSM vs. DSM, positional BAT "joins"

---

### Chapter 3: Wide Table Storage

-- **MonetDB** --

```sql
-- Create a three-column (i.e., wide) table and populate it with 1000 rows
DROP TABLE IF EXISTS ternary;
CREATE TABLE ternary (
	a int NOT NULL,
    b text NOT NULL,
    c float
);

INSERT INTO ternary(a, b, c)
	SELECT 	value 			AS a,
			md5(value)	 	AS b,
			log10(value) 	AS c
	FROM 	generate_series(1, 1001);

-- Probe query Q2 (retrieve all three columns a, b, c)
SELECT 	t.*
FROM 	ternary AS t;

-- Show the MAL plan for probe query Q2:
EXPLAIN
	SELECT 	t.*
	FROM 	ternary AS t;
	

```



#### N-ary vs Decomposed Storage Model (NSM/DSM)

MonetDB follows the Decomposed Storage Model (DSM) and represents n-ary tables using full vertical fragmentation



#### Row vs Column storage

Both types of DBMS exhibit strengths / weaknesses for different classes of workloads (OLTP vs OLAP)

OLTP - online transaction processing

OLAP - online analytical processing



```sql
-- Talking MAL to MonetDB (mclient -d<database> -lmsql)

-- Excerpt of the MAL query plan for probe query Q2

-- initialize MonetDB's SQL sub-system
sql 				:= sql.mvc();
-- access row visibility BAT (ternary) + three column BATs (a,b,c)
ternary	:bat[:oid] 	:= sql.tid(sql, "sys", "ternary");

c0 		:bat[:dbl] 	:= sql.bind(sql, "sys", "ternary", "c", 0:int);
c 					:= algebra.projection(ternary, c0);

b0 		:bat[:str] 	:= sql.bind(sql, "sys", "ternary", "b", 0:int);
b 					:= algebra.projection(ternary, b0);

a0 		:bat[:int] 	:= sql.bind(sql, "sys", "ternary", "a", 0:int);
a 					:= algebra.projection(ternary, a0);

-- prints three-column table (+ leading void column)
io.print(a,b,c);
```

















 













