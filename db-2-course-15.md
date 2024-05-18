# SQL Snippets



> [2024/05/18] | [00:20:00]
> DB2 — Chapter #06 — Video #19 — Memory latency, spatial and temporal locality

---

### Chapter 6: Buffering and Caching

-- **PostgreSQL** --



#### Memory: Fast, But Tiny vs. Slow, But Large

* Recall the enormous latency gaps between accesses to the (L2) CPU cache, RAM, and secondary storage (SSD/HDD)
* Facts: faster memory is significantly smaller. We will not be able to build cache-only systems.
  * The lion share of data will live in slow memory.
  * Only selected data fragments may reside in fast memory. Which fragments shall we choose?




#### Spatial Locality

In a DBMS (and most computing processes), memory accesses are not random but exhibit patterns of spatial and/or temporal locality.

Spatial locality: last memory access at address `m`, next access will be at address `m + delta m` (`delta m` small).

- Oftern, delta m equals machine word size: backward / forward scan of memory, i.e., iteration over an array.
- Block I/O does access and read data at m and its vicinity: |block access| << |memory accesses|



#### Temporal Locality

Temporal locality: last memory access at `m` at time `t`, next access at `m` will be at time `t + delta t` (`delta t` small).

Memory that is relevant now, will probably be relevant in the near future -> DBMS tracks frequency and recency of memory usage. Uses both to decide whether to hold a page in fast memory.
