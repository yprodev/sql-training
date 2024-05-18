# SQL Snippets



> [2024/05/18] | [00:29:00]
> DB2 — Chapter #06 — Video #20 — PostgreSQL buffer cache, (un-)pinned pages, dirty bit

---

### Chapter 6: Buffering and Caching

-- **PostgreSQL** --



#### The Buffer Cache

The DBMS sets aside a dedicated section of RAM - the buffer cache (or simply buffer) - to temporarily hold pages.

- All DBMS page accesses are performed using the buffer -> cat track page usage
- `|buffer| << |RAM|`. In PostgreSQL, see config variable `shared_buffers` (defafults to 128MB). Good practice: buffer size ~= 25% of RAM.

```sql
-- Check the size of RAM set aside to hold the PostgreSQL buffer
show shared_buffers;

-- Cannot set buffer size from the psql client. Instead edit configuration
-- file postgresql.confg and restart the server.
set shared_buffers="128KB"; -- ERROR: parameter "shared_buffers" cannot be changed without restarting the server

-- Enable a PostgreSQL extension that lets us peak inside the buffer:
CREATE EXTENSION IF NOT EXISTS pg_buffercache;

-- How many pages can the current buffer hold overall?
SELECT  COUNT(*)
FROM 	pg_buffercache;
```



#### Buffer Cache Interface (API)

Any database transaction properly "brackets" page accesses using `ReadBuffer()` and `ReleaseBuffer()` calls:

```c
<b, m> <- ReadBuffer(table, block);
/* now may access 8 KB page stating at address m */
...
if (page at m has been written to)
    MarkBufferDirty(b);
...
ReleaseBuffer(b);
/* accesses to address m illegal from here on */
```

Proper bracketing enables the DBMS to perform bookkeeping of buffer contents.



#### Buffer Page Bookkeeping

Each page in the buffer is associated with meta data that reflects is current utility for the DBMS

```sql
buffer pages 		buffer descriptor

| b - 1 | 			ref_count(b) >= 0 		how often has this page been requested via ReadBuffer()?
---------
|   b   | 			usage_count(b) >= 0 	how many r/w operations have been performed on this page?
---------
| b + 1 | 			dirty?(b) 				does memory on this page differ from its disk image?

```

`ref_count(b)` also commonly known as the pin count of b.



#### Reading a Buffer Page: Hit vs. Miss

```c
ReadBuffer(table, block):
	if (a buffer page b already contains block of table)
        ref_count(b) <- ref_count(b) + 1;

 		return <b, address of b's page>; 					/* hit: no I/O */
    
    else 													/* miss: I/O needed */
        v <- free buffer page; 								/* 1 */
		if (there is no such free v)
            v <- FindVictimBufferPage(); 					/* 2 */
			if dirty?(v)
                write page in v to disk block
        read requested block from disk into page of v;
		ref_count(v) <- 1;
		dirty?(v) <- false;

		return <v, address of v's page>;

```



#### Clean vs. Dirty Buffer Pages

- Read-only transactions leave buffer pages clean. Clean victim pages may simply be overwritten when replaced.

- Marking buffer page b dirty (i.e. written to/altered):

```c
MarkBufferDirty(b):
	dirty?(b) <- true;
```

- In regular intervals, the DBMS writes dirty buffer pages back (**checkpointing**) to match memory and disk content's.
  - Checkpointing may lead to heavy I/O traffic.

PostgreSQL: see config variable `checkpoint_timeout` (default: `5min`). SQL command `CHECKPOINT` forces immediate checkpointing.



#### Releasing a Buffer Page

- Release buffer page b. If `ref_count(b) > 0`, `b` is called pinned. If `ref_count(b) = 0`, `b` is unpinned

```c
ReleaseBuffer(b):
	ref_count(b) <- ref_count(b) - 1; 			/* no I/O */
```

- `ReleaseBuffer()` does not write the page of `b` back to disk, even if `b` is unpinned and dirty. Quiz: Why?
- Any pinned buffer page is in active use by some transaction and thus may never b chosen as a victim for replacement.











