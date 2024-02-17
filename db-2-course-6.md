# SQL Snippets



> [2024/02/17] | [00:09:00]
> DB2 — Chapter 02 — Video #07 — MAL↔︎SQL, algebra.projection()

---

### Unary Table Storage

-- **MonetDB** --

```sql
-- Use MAL to access the BAT (call it a) that holds the values in
-- column of a SQL table unary:
sql := sql.mvc();
a:bat[:int] := sql.bind(sql, "sys", "unary", "a", 0:int);
io.print(a);

```





















