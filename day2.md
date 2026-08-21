# Day 2 — PostgreSQL 18 Architecture Deep Dive (Hands-On Lab)

## Objective

Stop reading about PostgreSQL internals and **watch them happen**.

This lab has two halves:

1. **Observe** — identify every PostgreSQL process from Linux, then from inside PostgreSQL, and see where shared memory lives.
2. **Trace** — run a single transaction and follow it through shared buffers → WAL buffers → WAL writer → checkpointer → background writer → MVCC → autovacuum.

## Environment

| Item | Value |
|---|---|
| OS | Rocky Linux 9 |
| PostgreSQL | 18 |
| PGDATA | `/pgdata/18/data` |
| Binaries | `/usr/pgsql-18/bin/` |
| Service | `postgresql-18` |

Carried over from Day 1. Confirm before starting:

```bash
systemctl status postgresql-18
sudo -iu postgres psql -c "SHOW data_directory;"
```

Expected: `/pgdata/18/data`

## Prerequisites

The lab uses three contrib extensions. Install the contrib package:

```bash
dnf install -y postgresql18-contrib
```

---

# Part 1 — Observe the Processes from Linux

## Step 1 — List Every PostgreSQL Process

```bash
ps -ef | grep '[p]ostgres'
```

Sample output (yours will differ):

```
postgres  1421     1  /usr/pgsql-18/bin/postgres -D /pgdata/18/data/
postgres  1423  1421  postgres: io worker 0
postgres  1424  1421  postgres: io worker 1
postgres  1425  1421  postgres: io worker 2
postgres  1426  1421  postgres: checkpointer
postgres  1427  1421  postgres: background writer
postgres  1429  1421  postgres: walwriter
postgres  1430  1421  postgres: autovacuum launcher
postgres  1431  1421  postgres: logical replication launcher
```

Two things to notice immediately:

- **Every process has PPID `1421`** — that is the postmaster, the main process. Everything else is its child.
- **PostgreSQL is a process-per-connection system**, not a thread-per-connection system. This is why each connection has real cost, and why connection poolers exist.

## Step 2 — See the Hierarchy as a Tree

```bash
pstree -p $(pgrep -f 'postgres -D /pgdata/18/data')
```

This is the picture the theory diagram was describing:

```
postgres(1421)─┬─postgres(1423)   io worker 0
               ├─postgres(1426)   checkpointer
               ├─postgres(1427)   background writer
               ├─postgres(1429)   walwriter
               ├─postgres(1430)   autovacuum launcher
               └─postgres(1431)   logical replication launcher
```

## Step 3 — Find the Postmaster PID the Reliable Way

PostgreSQL writes its own PID into PGDATA:

```bash
head -1 /pgdata/18/data/postmaster.pid
```

The whole file is worth reading:

```bash
cat /pgdata/18/data/postmaster.pid
```

```
1421                       <- postmaster PID
/pgdata/18/data            <- data directory
1732800000                 <- start time (epoch)
5432                       <- port
/var/run/postgresql        <- socket directory
*                          <- listen address
  5432001         0        <- shared memory key and ID
ready                      <- status
```

> **DBA note:** a stale `postmaster.pid` left behind after a crash is a common cause of "server won't start." Never delete it while a live `postgres` process is running.

## Step 4 — Confirm PGDATA from the Process Itself

Don't trust config files — ask the running process:

```bash
sudo ls -l /proc/$(head -1 /pgdata/18/data/postmaster.pid)/cwd
```

Expected:

```
... /proc/1421/cwd -> /pgdata/18/data
```

## Step 5 — Map Each Process to Its Job

| Process | Responsibility |
|---|---|
| **postmaster** | Parent process. Listens on port 5432, forks a backend per connection, starts and restarts auxiliary processes. Does no query work itself. |
| **client backend** | One per connection. Parses, plans, and executes your SQL. Reads and writes shared buffers. |
| **walwriter** | Flushes WAL buffers to disk on `wal_writer_delay`, so committing backends often find the work already done. |
| **checkpointer** | Writes *all* dirty shared buffers to disk at checkpoints and records a restart point in WAL. |
| **background writer** | Trickles out dirty buffers continuously so backends rarely have to write a page themselves to find a free buffer. |
| **autovacuum launcher** | Wakes on `autovacuum_naptime` and spawns autovacuum workers against tables with enough dead tuples. |
| **logical replication launcher** | Manages logical replication apply workers. Idle unless you use logical replication. |
| **io worker** | New in PG 18 — performs asynchronous I/O on behalf of backends. Count is controlled by `io_workers`. |

> **Note on version drift:** if you have used PostgreSQL 14 or earlier, you may be looking for the **stats collector**. It was removed in PostgreSQL 15 — cumulative statistics now live in shared memory. Don't go hunting for a process that no longer exists.

## Step 6 — Check the PG 18 Asynchronous I/O Settings

```bash
sudo -iu postgres psql -c "SHOW io_method;" -c "SHOW io_workers;"
```

If `io_method` is `worker`, the `io worker` processes you saw in Step 1 are explained. If it is `sync`, you will not see them at all — that is fine, and your instance simply uses the older synchronous path.

---

# Part 2 — Observe the Same Processes from Inside PostgreSQL

Connect:

```bash
sudo -iu postgres psql
```

## Step 7 — List Processes via pg_stat_activity

```sql
SELECT pid, backend_type, state, wait_event_type, wait_event
FROM pg_stat_activity
ORDER BY backend_type, pid;
```

Compare this list against your `ps -ef` output. Same PIDs, different vantage point.

Two things to understand:

- The **postmaster does not appear here.** It is not a backend and collects no statistics.
- Your own session appears as `client backend` with `state = active` — it is the row running the query.

Count them by type:

```sql
SELECT backend_type, count(*)
FROM pg_stat_activity
GROUP BY backend_type
ORDER BY 2 DESC;
```

## Step 8 — Inspect Shared Memory Settings

```sql
SELECT name, setting, unit
FROM pg_settings
WHERE name IN (
  'shared_buffers',
  'wal_buffers',
  'work_mem',
  'maintenance_work_mem',
  'effective_cache_size',
  'max_connections'
)
ORDER BY name;
```

The critical distinction:

```
Shared memory (one copy, all backends)
    ├── shared_buffers        <- the page cache
    └── wal_buffers           <- staging area for WAL records

Local memory (per backend, multiplied by connections)
    ├── work_mem              <- per sort / hash node
    └── maintenance_work_mem  <- per VACUUM / CREATE INDEX
```

`work_mem` is the setting that surprises people: it is allocated *per operation, per backend*, not per connection. A query with three sorts can use three times `work_mem`.

## Step 9 — See the Shared Memory Layout

```sql
SELECT name,
       pg_size_pretty(allocated_size) AS size
FROM pg_shmem_allocations
ORDER BY allocated_size DESC
LIMIT 15;
```

`Buffer Blocks` should dominate — that is `shared_buffers`. You will also see `XLOG Ctl` (WAL buffers and control data), `Lock Manager`, `Proc Array`, and others.

From the Linux side:

```bash
ipcs -m
```

You will find only one small System V segment. Since version 9.3, PostgreSQL allocates the bulk of its shared memory as an anonymous `mmap` region — the tiny SysV segment exists mainly as a startup interlock so two postmasters can't attach to the same data directory.

---

# Part 3 — Look Inside the Shared Buffers

## Step 10 — Set Up the Lab Database

```sql
CREATE DATABASE day2lab;
\c day2lab
CREATE EXTENSION pg_buffercache;
CREATE EXTENSION pageinspect;
CREATE EXTENSION pg_walinspect;
```

Create a table to trace:

```sql
CREATE TABLE accounts (
  id      int PRIMARY KEY,
  owner   text,
  balance numeric
);
```

## Step 11 — Confirm the Table Is Not Yet in Memory

Every relation on disk has a **filenode** number. That is how you find its pages in the buffer cache.

```sql
SELECT pg_relation_filenode('accounts') AS filenode;
```

Now look for its buffers:

```sql
SELECT count(*)
FROM pg_buffercache
WHERE relfilenumber = pg_relation_filenode('accounts');
```

Expected: `0` — or a very small number. Nothing has read the table yet.

> **Version note:** the column is `relfilenumber` in PostgreSQL 16 and later. On 15 and earlier it is `relfilenode`. Run `\d pg_buffercache` if you are unsure.

## Step 12 — Load a Page and Watch It Appear

```sql
INSERT INTO accounts VALUES (1, 'monik', 1000);

SELECT bufferid, relblocknumber, isdirty, usagecount
FROM pg_buffercache
WHERE relfilenumber = pg_relation_filenode('accounts');
```

Now there is a buffer, and critically:

```
 isdirty
---------
 t
```

**The row is in memory and not yet on disk.** The INSERT modified a page in `shared_buffers` and marked it dirty. Nothing wrote your table file. This is the single most important idea in PostgreSQL's write path.

Get the cache overview:

```sql
SELECT * FROM pg_buffercache_summary();
SELECT * FROM pg_buffercache_usage_counts();
```

`usagecount` is the clock-sweep counter — it increments on access and decrements as the sweep passes. Buffers that reach zero become eviction candidates.

---

# Part 4 — Trace One Transaction End to End

This is the core of the lab. You will follow a single transaction through every stage.

## Step 13 — Capture the WAL Position Before

```sql
SELECT pg_current_wal_lsn() AS lsn_before \gset
SELECT :'lsn_before' AS lsn_before;
```

An **LSN** (Log Sequence Number) like `0/1A2B3C8` is a byte offset into the WAL stream. It is PostgreSQL's clock for durability.

Which physical file is that?

```sql
SELECT pg_walfile_name(:'lsn_before');
```

```bash
ls -lh /pgdata/18/data/pg_wal/
```

The file is 16 MB and was pre-allocated. WAL files are recycled, not created on demand.

## Step 14 — Open a Transaction and Claim an XID

```sql
BEGIN;

SELECT pg_current_xact_id() AS my_xid;
```

Note the number. This is your **transaction ID** — the identity MVCC uses to decide which rows you can see and which rows you created.

> Transaction IDs are assigned lazily. A read-only transaction never gets one. `pg_current_xact_id()` forces the assignment so we can watch it; use `pg_current_xact_id_if_assigned()` when you don't want that side effect.

## Step 15 — Do the Work

```sql
INSERT INTO accounts VALUES (2, 'chavda', 5000);
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
```

Before committing, in **this same session**, look at the row versions:

```sql
SELECT ctid, xmin, xmax, id, owner, balance FROM accounts;
```

You will see **three** rows for two logical records:

```
 ctid  | xmin | xmax | id | owner  | balance
-------+------+------+----+--------+---------
 (0,2) |  842 |    0 |  2 | chavda |    5000
 (0,3) |  842 |    0 |  1 | monik  |     900
```

The original `id = 1` row at `(0,1)` is still there on the page — it is just no longer visible to you, because your UPDATE stamped its `xmax` with your XID and wrote a *new* version at `(0,3)`.

**PostgreSQL never updates a row in place.** An UPDATE is a DELETE plus an INSERT. This one fact explains table bloat, why VACUUM exists, and why HOT updates matter.

## Step 16 — Prove Isolation with a Second Session

Leave the transaction **open**. In a second terminal:

```bash
sudo -iu postgres psql -d day2lab
```

```sql
SELECT ctid, xmin, xmax, id, owner, balance FROM accounts;
```

The second session sees the **old** `id = 1` row with balance `1000`, and does not see `id = 2` at all. Same table, same instant, two different views of reality. That is MVCC: readers never block writers, writers never block readers.

Check what the other session is doing:

```sql
SELECT pid, state, xact_start, query
FROM pg_stat_activity
WHERE datname = 'day2lab' AND backend_type = 'client backend';
```

The first session shows `state = idle in transaction` — the state every DBA learns to watch for, because an open transaction holds back the vacuum horizon.

## Step 17 — Commit and Watch the WAL Grow

Back in the **first** session:

```sql
COMMIT;

SELECT pg_current_wal_lsn() AS lsn_after \gset
SELECT pg_size_pretty(
  pg_wal_lsn_diff(:'lsn_after', :'lsn_before')
) AS wal_generated;
```

You just measured, in bytes, exactly how much WAL your transaction produced.

**What happened at COMMIT, in order:**

```
1. WAL records were written into WAL buffers   (shared memory)
2. COMMIT flushed WAL buffers to pg_wal/       (fsync to disk)
3. Only then did COMMIT return success
4. The data pages are STILL DIRTY in shared_buffers
```

This is **write-ahead logging**: the log goes to disk before the data does. If the server lost power right now, your data pages would be lost — but the WAL would be replayed at startup and the transaction would survive. Durability comes from the WAL, not from the table file.

## Step 18 — Read the Actual WAL Records

This is where the transaction becomes visible as physical evidence:

```sql
SELECT start_lsn, resource_manager, record_type, record_length
FROM pg_get_wal_records_info(:'lsn_before', :'lsn_after')
ORDER BY start_lsn;
```

You will see records like:

```
 resource_manager |  record_type   | record_length
------------------+----------------+---------------
 Heap             | INSERT         |            79
 Heap             | HOT_UPDATE     |           101
 Transaction      | COMMIT         |            34
```

Summarised by type:

```sql
SELECT resource_manager, count, record_size
FROM pg_get_wal_stats(:'lsn_before', :'lsn_after')
WHERE count > 0
ORDER BY record_size DESC;
```

There is your transaction — a heap insert, a heap update, and a commit record, on disk, in order.

## Step 19 — Confirm the Data Is Still Only in Memory

```sql
SELECT bufferid, relblocknumber, isdirty
FROM pg_buffercache
WHERE relfilenumber = pg_relation_filenode('accounts');
```

Still `isdirty = t`. The transaction is committed and durable, but the table file on disk has not been touched. Confirm from Linux:

```bash
sudo ls -l /pgdata/18/data/base/*/$(sudo -iu postgres psql -Atd day2lab -c "SELECT pg_relation_filenode('accounts');")
```

## Step 20 — Force a Checkpoint and Watch the Page Land

Look at checkpoint statistics first:

```sql
SELECT * FROM pg_stat_checkpointer;
```

Note `num_timed` (checkpoints triggered by `checkpoint_timeout`) and `num_requested` (triggered manually or by `max_wal_size`).

Now force one:

```sql
CHECKPOINT;
```

Re-check the buffer:

```sql
SELECT bufferid, relblocknumber, isdirty
FROM pg_buffercache
WHERE relfilenumber = pg_relation_filenode('accounts');
```

```
 isdirty
---------
 f
```

**Clean.** The checkpointer wrote the page to the data file and marked the buffer clean. The page is still cached — checkpointing writes buffers out, it does not evict them.

Confirm the counter moved:

```sql
SELECT num_requested, buffers_written FROM pg_stat_checkpointer;
```

And watch it in the server log:

```bash
tail -20 /pgdata/18/data/log/*.log
```

Since PostgreSQL 15, `log_checkpoints` defaults to `on`, so you will see a `checkpoint starting` / `checkpoint complete` pair with the buffer count and timings.

## Step 21 — Check the Background Writer

```sql
SELECT * FROM pg_stat_bgwriter;
```

| Column | Meaning |
|---|---|
| `buffers_clean` | Buffers written by the background writer |
| `maxwritten_clean` | Times it stopped early because it hit `bgwriter_lru_maxpages` |
| `buffers_alloc` | Buffers allocated |

A high `maxwritten_clean` means the background writer is being throttled and backends are being left to write their own pages — a real tuning signal.

For the fuller picture, PostgreSQL 16+ gives you per-process I/O:

```sql
SELECT backend_type, object, context, reads, writes, extends
FROM pg_stat_io
WHERE reads > 0 OR writes > 0
ORDER BY backend_type;
```

This tells you *which process type* is doing your I/O — checkpointer, background writer, or client backends doing their own writes.

---

# Part 5 — See MVCC on the Raw Page

## Step 22 — Read the Heap Page Directly

```sql
SELECT lp AS line_pointer,
       t_xmin, t_xmax, t_ctid,
       lp_flags
FROM heap_page_items(get_raw_page('accounts', 0));
```

`lp_flags = 1` means the line pointer is normal (live or recently dead); `lp_flags = 0` means unused; `lp_flags = 2` means redirected by a HOT chain.

Your dead row version from Step 15 is still physically on the page, occupying space, with `t_xmax` set to the XID that killed it. It will stay there until VACUUM reclaims it.

## Step 23 — Create Dead Tuples Deliberately

```sql
INSERT INTO accounts
SELECT g, 'user_' || g, g * 100
FROM generate_series(3, 10000) g;

UPDATE accounts SET balance = balance + 1;
DELETE FROM accounts WHERE id % 3 = 0;
```

Measure the damage:

```sql
SELECT n_live_tup, n_dead_tup,
       pg_size_pretty(pg_total_relation_size('accounts')) AS size
FROM pg_stat_user_tables
WHERE relname = 'accounts';
```

Thousands of dead tuples, and the table is much larger than its live contents justify. **This is bloat**, and it is the direct consequence of the no-in-place-update rule from Step 15.

---

# Part 6 — Autovacuum

## Step 24 — Understand the Trigger Formula

```sql
SELECT name, setting
FROM pg_settings
WHERE name IN (
  'autovacuum',
  'autovacuum_naptime',
  'autovacuum_vacuum_threshold',
  'autovacuum_vacuum_scale_factor',
  'autovacuum_vacuum_insert_threshold',
  'autovacuum_max_workers'
)
ORDER BY name;
```

Autovacuum triggers a table when:

```
dead_tuples > autovacuum_vacuum_threshold
              + (autovacuum_vacuum_scale_factor × live_tuples)

            = 50 + (0.2 × live_tuples)     [defaults]
```

The scale factor is the problem in production. On a 100-million-row table, the default means waiting for **20 million dead tuples** before autovacuum even starts. This is why large tables get per-table overrides:

```sql
-- Do not run this now; understand it for later.
-- ALTER TABLE accounts SET (autovacuum_vacuum_scale_factor = 0.01);
```

## Step 25 — Watch Autovacuum Work

Check whether it has already fired:

```sql
SELECT relname, last_vacuum, last_autovacuum,
       vacuum_count, autovacuum_count, n_dead_tup
FROM pg_stat_user_tables
WHERE relname = 'accounts';
```

Catch a worker in the act (run repeatedly during a large workload):

```sql
SELECT pid, backend_type, query, xact_start
FROM pg_stat_activity
WHERE backend_type = 'autovacuum worker';
```

Or force it manually and read the report:

```sql
VACUUM (VERBOSE, ANALYZE) accounts;
```

The verbose output tells you tuples removed, pages scanned, and index vacuum passes.

## Step 26 — Confirm Space Was Reclaimed

```sql
SELECT n_live_tup, n_dead_tup,
       pg_size_pretty(pg_total_relation_size('accounts')) AS size
FROM pg_stat_user_tables
WHERE relname = 'accounts';
```

`n_dead_tup` drops to near zero — but **the table size does not shrink.** Plain VACUUM marks space reusable *within* the file; it does not return it to the operating system. Only `VACUUM FULL` (which takes an ACCESS EXCLUSIVE lock and rewrites the whole table) or a tool like `pg_repack` does that.

That distinction is a genuine production trap. Learn it here rather than at 2 a.m.

---

# Cleanup

```sql
\c postgres
DROP DATABASE day2lab;
```

---

# What You Actually Traced

```
    Your INSERT
         |
         v
 +-------------------+
 |  Client Backend   |  forked by postmaster, one per connection
 +-------------------+
         |
         +--> modifies page in SHARED BUFFERS  --> marked dirty
         |
         +--> writes record to WAL BUFFERS
                     |
                     v
                 COMMIT
                     |
         +-----------+-----------+
         |                       |
         v                       v
  WAL flushed to pg_wal/   Data page STILL dirty
  (durability achieved)    (in memory only)
         |                       |
         |          +------------+------------+
         |          |                         |
         v          v                         v
    WAL WRITER  CHECKPOINTER            BACKGROUND WRITER
    (periodic   (writes ALL dirty       (trickles dirty
     WAL flush)  buffers, bounds         buffers out to
                 recovery time)          keep buffers free)
                     |
                     v
              Data files on disk
                     |
                     v
       Old row versions left behind by MVCC
                     |
                     v
                AUTOVACUUM
             (reclaims dead tuples)
```

## 🔑 Key DBA Lessons from Day 2

1. **The postmaster forks; it does not execute.** Every query runs in a child backend.
2. **Writes go to memory first.** A committed transaction lives in the WAL and in dirty shared buffers, not in the table file.
3. **Durability comes from the WAL, not the data file.** That is what "write-ahead" means.
4. **UPDATE is DELETE + INSERT.** Every update leaves a corpse; MVCC is the reason readers and writers don't block each other, and bloat is the price.
5. **The checkpointer bounds recovery time; the background writer keeps buffers available.** They are different jobs, and `pg_stat_io` will tell you which one is actually doing your writes.
6. **Plain VACUUM reclaims space for reuse, not for the OS.** Size on disk staying flat after a vacuum is correct behaviour, not a failure.

---

## Next: Day 3 — Configuration, Memory Tuning, and the Postgres Config Hierarchy

```
postgresql.conf
      ↓
postgresql.auto.conf   (written by ALTER SYSTEM)
      ↓
Command-line / systemd
      ↓
ALTER DATABASE / ALTER ROLE
      ↓
SET in session
      ↓
Effective setting  (pg_settings.source)
```

Day 3 will cover sizing `shared_buffers`, `work_mem`, and `effective_cache_size` for a real workload, which parameters need a restart versus a reload, and how to find out *why* a setting has the value it has.
