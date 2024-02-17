# SQL Snippets



> [2024/02/17] | [00:31:00]
> DB2 — Chapter 02 — Video #08 — UNIX system call mmap(), fixed/variable-width BAT payloads

---

### Unary Table Storage

-- **MonetDB** --



#### A Main-Memory DBMS

All BATs are processed as in-memory arrays of fixed-width elements (atoms).

- Transient BATs exist in RAM only.
- Persistent BATs live on disk and are `mmap(2)`ed into RAM



#### Map Files into Memory

The contents of file `fd` are mapped `:1 into contiguous memory. No conversion or transformation takes place - compare this to PostgresSQL's row storage (later).

OS implements virtual memory: can map even huge files.



#### Peeking into a MonetDB BAT

Use MAL builtin function `bat.info()` to collect details about the BAT for column `unary(a)` of 100 32-bit `int`s

```sql
-- Collect information about BAT a, in particular find out where its
-- persistent representation is found in the file system
(i1, i2) := bat.info(a);
io.print(i1, i2);
```





















