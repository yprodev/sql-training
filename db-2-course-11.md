# SQL Snippets



> [2024/02/25] | [00:15:00]
> DB2 — Chapter #04 — Video #12 — (Non-)storage of NULL in PostgreSQL, NULL bitmaps

---

### Chapter 4: Row Internals

-- **PostgreSQL** --



#### NULL (Non-)Storage

`NULL` values are represented by 0 bits in a NULL bitmap (bitmap is present only if the row indeed contains a `NULL`)



```sql
-- Representation of NULL in rows

-- Check the rows on page 0 of table ternary and how the row's meta data represent NULL
SELECT 	lp, lp_off, lp_len, t_hoff, t_ctid, t_infomask::bit(1) AS "any NULL?", t_bits
FROM 	heap_page_items(get_raw_page('ternary', 0));

-- What's the representation of a row that entirely consists of NULL values?
INSERT INTO padded(a, b, c, d, e, f)
VALUES (NULL, NULL, NULL, NULL, NULL, NULL);

-- Which row is that all-NULL row? (Yield RID (<p>, <slot>))
SELECT 	p.ctid
FROM 	padded AS p
WHERE 	(p.a, p.b, p.c, p.d, p.e, p.f) IS NULL; 	-- === a IS NULL AND b IS NULL AND ...

-- Take a closer look at that row
SELECT 	lp, lp_off, lp_len, t_hoff, t_ctid, t_infomask::bit(1) AS "any NULL?", t_bits
FROM 	heap_page_items(get_raw_page('padded', <p>)) 		-- replace <p> 		p is page number
WHERE 	lp = <slot>; 										-- replace <slot> - slot is a tuple / row number


```









