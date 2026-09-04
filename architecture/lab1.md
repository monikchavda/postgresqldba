🐘 Day 2 — Lab Analysis

Your output:

```text
PID    PPID USER     CMD
55133      1 postgres /usr/pgsql-18/bin/postgres -D /data
55134  55133 postgres postgres: logger
55135  55133 postgres postgres: io worker 0
55136  55133 postgres postgres: io worker 2
55137  55133 postgres postgres: io worker 1
55138  55133 postgres postgres: checkpointer
55139  55133 postgres postgres: background writer
55141  55133 postgres postgres: walwriter
55142  55133 postgres postgres: autovacuum launcher
55143  55133 postgres postgres: logical replication launcher
```

The most important relationship is:

```text
                     PID 55133
                  PostgreSQL Main
                       Process
                         |
          +--------------+--------------+
          |              |              |
          v              v              v
       Logger       IO Workers       Checkpointer
                                      |
                                      v
                              Background Writer
                                      |
                                      v
                                  WAL Writer
                                      |
                                      v
                               Autovacuum
```

# 1. PID 55133 — Main PostgreSQL Process

```text
55133 postgres /usr/pgsql-18/bin/postgres -D /data
```

This is the main PostgreSQL server process.

The important part is:

```text
-D /data
```

That tells PostgreSQL:

> "My PostgreSQL data directory is /data."

This is worth verifying from PostgreSQL itself:

```bash
sudo -iu postgres psql -c "SHOW data_directory;"
```

You should get:

```text
 data_directory
----------------
 /data
```

### DBA lesson

Don't assume PGDATA based on the installation documentation.

Always verify:

```sql
SHOW data_directory;
```

or from Linux:

```bash
ps -ef | grep '[p]ostgres'
```

---

# 2. PPID = 1

Notice:

```text
55133       1 postgres
```

The parent PID of PostgreSQL is:

```text
1
```

On Rocky Linux 9, PID 1 is normally systemd.

So the process hierarchy is:

```text
systemd
   |
   +---- postgres (55133)
             |
             +---- logger
             +---- io worker
             +---- io worker
             +---- io worker
             +---- checkpointer
             +---- background writer
             +---- walwriter
             +---- autovacuum launcher
             +---- logical replication launcher
```

This is exactly the kind of relationship you should learn to recognize as a DBA.

---

# 3. PID 55134 — Logger

```text
55134 postgres: logger
```

This is PostgreSQL's logging process.

Depending on your logging configuration, PostgreSQL can write server logs through this logging architecture.

Later we'll investigate:

```sql
SHOW logging_collector;
SHOW log_directory;
SHOW log_filename;
SHOW log_min_duration_statement;
```

Don't change them yet.

---

# 4. PID 55135–55137 — IO Workers

This is particularly interesting in your PostgreSQL 18 installation:

```text
55135 postgres: io worker 0
55136 postgres: io worker 2
55137 postgres: io worker 1
```

You have three I/O worker processes.

This is a PostgreSQL 18 feature worth paying attention to.

Conceptually:

```text
                 PostgreSQL
                     |
              I/O subsystem
                     |
        +------------+------------+
        |            |            |
        v            v            v
    IO Worker 0  IO Worker 1  IO Worker 2
```

These workers participate in PostgreSQL's asynchronous I/O architecture.

### Important DBA point

Don't confuse:

```text
io worker
```

with:

```text
client backend
```

An I/O worker is not a user connection.

It's part of PostgreSQL's internal I/O processing.

---

# 5. PID 55138 — Checkpointer

```text
55138 postgres: checkpointer
```

This is one of the most important processes.

Remember:

```text
SQL
 ↓
shared buffers
 ↓
dirty pages
 ↓
checkpoint/write processes
 ↓
disk
```

The checkpointer manages checkpoint processing.

A checkpoint establishes a known point from which crash recovery can proceed.

Later we'll inspect:

```sql
SELECT *
FROM pg_stat_checkpointer;
```

PostgreSQL 18 provides useful checkpoint statistics here.

---

# 6. PID 55139 — Background Writer

```text
55139 postgres: background writer
```

The background writer helps write dirty buffers from shared memory toward storage.

Conceptually:

```text
                shared_buffers
                     |
                dirty buffers
                     |
                     v
             Background Writer
                     |
                     v
                    Disk
```

But remember:

> Background writer ≠ checkpointer.

They have different responsibilities.

We'll explore this deeply when we reach checkpoints and I/O.

---

# 7. PID 55141 — WAL Writer

```text
55141 postgres: walwriter
```

This process deals with writing WAL information.

Think:

```text
Transaction
     |
     v
 WAL records
     |
     v
 WAL buffers
     |
     v
 WAL writer
     |
     v
 /data/pg_wal/
```

This is conceptually similar to the role an Oracle DBA associates with LGWR, but it is not an exact implementation equivalent.

Oracle:

```text
LGWR → REDO
```

PostgreSQL:

```text
WAL writer → WAL
```

---

# 8. PID 55142 — Autovacuum Launcher

```text
55142 postgres: autovacuum launcher
```

🚨 Very important PostgreSQL DBA process.

PostgreSQL uses MVCC.

For an UPDATE, PostgreSQL generally creates a new row version rather than simply overwriting the existing version in place.

Conceptually:

Before:

```text
Row version A
```

UPDATE

After:

```text
Row version A  → old version
Row version B  → new version
```

Eventually old row versions become removable.

VACUUM helps clean them up.

Autovacuum automatically initiates maintenance.

Later you'll learn:

```text
Autovacuum
    |
    +-- VACUUM
    +-- ANALYZE
    +-- Dead tuple cleanup
    +-- Transaction ID freezing
```

This will become one of the most important production DBA topics.

---

# 9. PID 55143 — Logical Replication Launcher

```text
55143 postgres: logical replication launcher
```

This process manages logical replication workers when logical replication is configured/used.

Conceptually:

```text
PostgreSQL
     |
     v
Logical Replication Launcher
     |
     +---- Apply Worker
     +---- Other logical workers
```

We'll study logical replication later.

For now, remember:

> Logical replication is different from physical streaming replication.

---

# 10. Your Architecture Diagram

Based specifically on your server, we can draw:

```text
                         systemd
                            |
                            v
                    PostgreSQL PID 55133
                     /usr/pgsql-18/bin/
                            |
        +-------------------+-------------------+
        |                   |                   |
        v                   v                   v
      Logger            I/O Workers        Checkpointer
      55134             55135-55137          55138
                                                |
                                                v
                                         Background Writer
                                              55139
                                                |
                                                v
                                           WAL Writer
                                              55141
                                                |
                         +----------------------+----------------+
                         |                                       |
                         v                                       v
                 Autovacuum Launcher                 Logical Replication
                       55142                            Launcher 55143
```

This is now your actual PostgreSQL 18 architecture, rather than a textbook example.
