# PostgreSQL Architecture — Deep Dive (Day 2 DBA Training)

This is the concept every other Day 2 topic (backup, replication, tuning, security) sits on top of. Once you understand **how PostgreSQL is put together internally**, WAL, vacuum, checkpoints, and replication all start to make sense as consequences of this design, not as isolated features to memorize.

---

# 1. The Big Picture

PostgreSQL uses a **process-based, client-server architecture** (not thread-based, unlike MySQL/Oracle in some configurations).

```text
                     ┌───────────────────────┐
                     │        Client          │
                     │ (psql, app, pgAdmin)   │
                     └───────────┬────────────┘
                                 │ TCP/IP or Unix socket
                                 ▼
                     ┌───────────────────────┐
                     │      Postmaster         │
                     │  (main listener process)│
                     └───────────┬────────────┘
                                 │ fork()
                                 ▼
                     ┌───────────────────────┐
                     │   Backend Process       │
                     │ (one per client conn.)  │
                     └───────────┬────────────┘
                                 │
                                 ▼
                     ┌───────────────────────┐
                     │   Shared Memory         │
                     │ (shared_buffers, WAL)   │
                     └───────────┬────────────┘
                                 │
                                 ▼
                     ┌───────────────────────┐
                     │   Data Files on Disk    │
                     │      (PGDATA)           │
                     └───────────────────────┘
```

Key idea: **every client connection gets its own OS process**, not a thread. This is why connection pooling (PgBouncer, Pgpool-II) matters so much for PostgreSQL — each connection is relatively expensive.

---

# 2. The Postmaster

The **postmaster** is the first process that starts when you run:

```bash
systemctl start postgresql-18
```

Its jobs:

- Listen on the configured port (default `5432`)
- Validate incoming connections (authentication via `pg_hba.conf`)
- `fork()` a new **backend process** for every accepted connection
- Start and supervise all the background processes
- Detect crashes and initiate recovery if a backend dies unexpectedly

```text
postmaster
   │
   ├── fork() ──► backend process (client 1)
   ├── fork() ──► backend process (client 2)
   ├── fork() ──► backend process (client 3)
   │
   ├── checkpointer
   ├── background writer (bgwriter)
   ├── WAL writer
   ├── autovacuum launcher
   ├── stats collector / cumulative stats
   ├── logical replication launcher
   └── archiver (if archive_mode = on)
```

You can see it in `ps aux`:

```bash
ps aux | grep postgres
```

```text
postgres  ... postgres -D /data
postgres  ... postgres: checkpointer
postgres  ... postgres: background writer
postgres  ... postgres: walwriter
postgres  ... postgres: autovacuum launcher
postgres  ... postgres: logical replication launcher
postgres  ... postgres: myapp myuser 10.0.0.5(5432) idle
```

That last line — one process per connected client — is the backend process.

---

# 3. Backend Processes (per-connection)

Each backend process:

- Handles **one client connection**, start to finish
- Parses, plans, and executes SQL for that session
- Has its own **local memory** (work_mem, temp buffers)
- Reads/writes data through **shared memory**, not directly talking to other backends

```text
Client Query
     │
     ▼
Parser  ──►  Rewriter (views/rules)  ──►  Planner/Optimizer  ──►  Executor
                                                                     │
                                                                     ▼
                                                          Shared Buffers / Disk
```

If a backend crashes (e.g., segfault), the postmaster treats it as a potential corruption event and **restarts the entire server**, forcing all other backends to reconnect — this is a deliberate safety measure, not a bug.

---

# 4. Memory Architecture

PostgreSQL memory is split into two categories:

```text
┌─────────────────────────────────────────────┐
│              Shared Memory                    │
│  (allocated once, shared by ALL processes)    │
│                                               │
│   shared_buffers                              │
│   WAL buffers                                 │
│   commit log (CLOG / pg_xact)                 │
│   lock tables                                 │
└─────────────────────────────────────────────┘

┌─────────────────────────────────────────────┐
│         Local Memory (per backend)            │
│                                               │
│   work_mem        (sorts, hash joins)         │
│   maintenance_work_mem (VACUUM, CREATE INDEX) │
│   temp_buffers    (temp tables)                │
└─────────────────────────────────────────────┘
```

### Key memory parameters

| Parameter | Purpose | Typical setting |
|---|---|---|
| `shared_buffers` | PostgreSQL's own page cache | 25% of RAM |
| `work_mem` | Per-operation memory (sort/hash), per backend | 4MB–64MB, tune carefully |
| `maintenance_work_mem` | VACUUM, CREATE INDEX, ALTER TABLE | 256MB–1GB |
| `effective_cache_size` | Planner's *estimate* of OS+PG cache available (not an allocation) | 50–75% of RAM |
| `wal_buffers` | Buffer for WAL records before flush | usually auto (-1) |

`work_mem` is **per sort/hash operation, per connection** — a single complex query can use several multiples of it. Setting it too high with many concurrent connections is a classic way to exhaust server RAM.

Check current values:

```sql
SHOW shared_buffers;
SHOW work_mem;
SHOW effective_cache_size;
```

---

# 5. Shared Buffers — PostgreSQL's Own Cache

PostgreSQL does **not** read/write disk directly for every query. It maintains its own buffer cache in shared memory.

```text
Disk Page
    │
    ▼
Shared Buffers (in RAM)
    │
    ▼
Backend reads/modifies page in memory
    │
    ▼
Page marked "dirty"
    │
    ▼
Background writer / checkpointer flushes it to disk later
```

This is why PostgreSQL performance depends on **both**:
- `shared_buffers` (PostgreSQL's cache)
- OS page cache (Linux's own disk cache)

`effective_cache_size` tells the query planner roughly how much total caching (both layers combined) is available — it does **not** allocate any memory itself, it only influences planning decisions (e.g., whether an index scan is likely to hit cache).

---

# 6. Write-Ahead Log (WAL) — The Heart of Durability

This is the single most important architectural concept for a DBA to internalize.

**Rule:** PostgreSQL never modifies a data file on disk *before* writing a record of that change to the WAL first.

```text
Transaction commits
        │
        ▼
Change written to WAL buffer (in memory)
        │
        ▼
WAL flushed to disk (pg_wal/ segment file)  ◄── this is the durability point
        │
        ▼
Client receives "COMMIT" confirmation
        │
        ▼
(later, asynchronously)
Actual data page flushed to disk by
checkpointer / background writer
```

Why this matters:

- If the server crashes **after** WAL flush but **before** the data page is written, PostgreSQL **replays the WAL** on startup ("crash recovery") and reconstructs the change.
- If WAL was **not** flushed, the transaction is considered not committed — no data loss, no corruption.

```text
pg_wal/
├── 000000010000000000000001
├── 000000010000000000000002
└── 000000010000000000000003
```

Each file is (by default) 16MB and contains a sequential, append-only log of every change. This is also the exact mechanism that powers:

- **Crash recovery**
- **Point-in-time recovery (PITR)**
- **Streaming replication** (standbys just replay the same WAL stream)

Check WAL location and settings:

```sql
SHOW wal_level;
SELECT pg_current_wal_lsn();
```

---

# 7. Background Processes — What Each One Actually Does

```text
┌────────────────────┬───────────────────────────────────────────────┐
│ Process              │ Job                                            │
├────────────────────┼───────────────────────────────────────────────┤
│ checkpointer         │ Periodically flushes ALL dirty buffers to disk │
│                      │ and creates a "checkpoint" — a recovery        │
│                      │ starting point                                 │
├────────────────────┼───────────────────────────────────────────────┤
│ background writer    │ Continuously trickles dirty pages to disk      │
│ (bgwriter)           │ between checkpoints, smoothing I/O spikes      │
├────────────────────┼───────────────────────────────────────────────┤
│ WAL writer           │ Flushes WAL buffers to disk periodically,      │
│                      │ even without a commit (reduces flush latency)  │
├────────────────────┼───────────────────────────────────────────────┤
│ autovacuum launcher  │ Spawns autovacuum worker processes to reclaim  │
│ + workers            │ dead tuples and update planner statistics      │
├────────────────────┼───────────────────────────────────────────────┤
│ stats collector /    │ Tracks table/index usage stats for the         │
│ cumulative stats     │ planner and for pg_stat_* views                │
├────────────────────┼───────────────────────────────────────────────┤
│ archiver              │ Copies completed WAL segments to an archive   │
│ (if enabled)          │ location — required for PITR                  │
├────────────────────┼───────────────────────────────────────────────┤
│ logical replication  │ Manages logical replication subscriptions      │
│ launcher              │                                                │
└────────────────────┴───────────────────────────────────────────────┘
```

**Checkpoint vs bgwriter — the distinction that confuses most people:**

```text
bgwriter:      small, continuous trickle of dirty pages → smooths I/O
checkpointer:  periodic, guarantees ALL dirty pages up to a point
               are on disk → defines a crash-recovery starting line
```

A checkpoint essentially says: *"Everything before this point in the WAL is guaranteed to be safely on disk — recovery only needs to replay WAL from here forward."*

---

# 8. MVCC — Multi-Version Concurrency Control

PostgreSQL never locks rows for reads. Instead, it keeps **multiple versions of a row** and each transaction sees a consistent snapshot.

```text
UPDATE creates a NEW row version, doesn't overwrite the old one

Row (old version)   xmin=100  xmax=105   ← invisible to txns after 105
Row (new version)   xmin=105  xmax=NULL  ← current visible version
```

- `xmin` = transaction ID that created this row version
- `xmax` = transaction ID that deleted/updated it away (NULL if still current)

This is **why VACUUM exists**: old row versions ("dead tuples") pile up and must be reclaimed, or the table bloats and `xid` wraparound risk increases.

```text
Transaction A (long-running, reads old snapshot)
        │
        ▼
   sees row version xmin=100
        │
Transaction B updates the row
        │
        ▼
   new row version xmin=105 created
   old version NOT deleted yet
        │
        ▼
   VACUUM later reclaims the old version
   once no transaction can still see it
```

---

# 9. Storage Layout on Disk

```text
PGDATA (/data)
├── base/              → one subdirectory per database, contains table/index files
├── global/            → cluster-wide tables (pg_database, etc.)
├── pg_wal/            → Write-Ahead Log segments
├── pg_xact/           → transaction commit status (was pg_clog)
├── pg_stat/           → statistics files
├── pg_tblspc/         → symlinks to tablespaces outside PGDATA
├── postgresql.conf    → main config
├── pg_hba.conf         → client authentication rules
└── PG_VERSION          → data format version
```

Within `base/`, each table is stored as one or more **1GB-segmented files**, broken into **8KB pages**. Each page holds **tuples** (row versions) plus a small header.

```text
Table file
├── Page 0  (8KB) → tuple headers + tuple data + free space
├── Page 1  (8KB)
├── Page 2  (8KB)
└── ...
```

Large field values (e.g., a long text column) that don't fit in a page get moved to a **TOAST table** (The Oversized-Attribute Storage Technique) — PostgreSQL's way of handling values bigger than a page.

---

# 10. Query Lifecycle, End to End

```text
1. Client sends SQL over the connection
2. Backend process: Parser builds a parse tree
3. Rewriter applies any rules/views
4. Planner/Optimizer generates the cheapest execution plan
   (uses pg_statistic, cost parameters, effective_cache_size)
5. Executor runs the plan:
      - reads pages via shared_buffers (cache hit) or disk (cache miss)
      - applies MVCC visibility rules to filter row versions
6. Results streamed back to client
7. On COMMIT:
      - WAL record written and flushed to pg_wal/
      - transaction marked committed in pg_xact
      - client receives commit acknowledgment
```

Inspect any plan with:

```sql
EXPLAIN ANALYZE SELECT * FROM orders WHERE customer_id = 42;
```

---

# 11. Putting It All Together

```text
                         ┌─────────────┐
                         │   Client     │
                         └──────┬───────┘
                                │
                         ┌──────▼───────┐
                         │  Postmaster   │  (auth, forks backends)
                         └──────┬───────┘
                                │
                         ┌──────▼───────┐
                         │   Backend     │  (parse/plan/execute)
                         └──────┬───────┘
                                │
             ┌──────────────────┼──────────────────┐
             ▼                                     ▼
     ┌───────────────┐                   ┌──────────────────┐
     │ Shared Buffers │                   │   WAL Buffers      │
     │ (data cache)   │                   │  (durability log)  │
     └───────┬────────┘                   └─────────┬──────────┘
             │                                       │
             ▼                                       ▼
     ┌───────────────┐                   ┌──────────────────┐
     │  base/ files    │                   │   pg_wal/ segments │
     └───────────────┘                   └──────────────────┘
             ▲                                       ▲
             │                                       │
   flushed by checkpointer / bgwriter        flushed by WAL writer
```

Background processes (checkpointer, bgwriter, walwriter, autovacuum) run continuously in the background, keeping this whole system durable, cache-warm, and free of bloat — without ever blocking client backends from doing their work.

---

# 12. Commands to Explore Architecture Live

```bash
# See all running PostgreSQL processes
ps aux | grep postgres
```

```sql
-- Current WAL position
SELECT pg_current_wal_lsn();

-- Buffer cache hit ratio (should be > 99% ideally)
SELECT sum(heap_blks_hit) / nullif(sum(heap_blks_hit) + sum(heap_blks_read), 0)
FROM pg_statio_user_tables;

-- Active backends and what they're doing
SELECT pid, usename, state, query FROM pg_stat_activity;

-- Checkpoint activity
SELECT * FROM pg_stat_bgwriter;

-- Data directory location
SHOW data_directory;
```

---

## ⭐ Key Concept to Remember

```text
Client → Backend Process → Shared Buffers ↔ WAL Buffers → Disk
                                    │
                          Background processes keep it
                          durable (WAL/checkpoint),
                          fast (bgwriter/cache),
                          and clean (autovacuum)
```

Everything else in Day 2 training — backups, replication, tuning, monitoring — is really just **operating on one of these pieces**:

- **Backup/PITR** → captures WAL + base backups
- **Replication** → ships the WAL stream to standbys
- **Performance tuning** → tunes shared_buffers, work_mem, checkpoints
- **Vacuum/autovacuum** → manages MVCC bloat
- **Monitoring** → watches `pg_stat_*` views for all of the above

Understanding this architecture first is what makes the rest of DBA work click into place instead of feeling like disconnected commands to memorize.
