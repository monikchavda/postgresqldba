Perfect. 👍 Now let's analyze **your actual PostgreSQL 18 memory settings**, because this is where PostgreSQL starts becoming much clearer.

## 🧠 Your Current Memory Configuration

You have:

| Setting                |     Your value | Actual size | What it means                       |
| ---------------------- | -------------: | ----------: | ------------------------------------ |
| `shared_buffers`       |  16,384 × 8 KB |  **128 MB** | PostgreSQL shared buffer cache      |
| `wal_buffers`          |     512 × 8 KB |    **4 MB** | Memory for WAL records              |
| `work_mem`             |       4,096 KB |    **4 MB** | Per-operation working memory        |
| `maintenance_work_mem` |      65,536 KB |   **64 MB** | Memory for maintenance operations   |
| `effective_cache_size` | 524,288 × 8 KB |    **4 GB** | Planner estimate of available cache |

So your configuration is effectively:

```text
shared_buffers          = 128 MB
wal_buffers             =   4 MB
work_mem                =   4 MB
maintenance_work_mem    =  64 MB
effective_cache_size    =   4 GB
```

---

# 1. `shared_buffers = 128 MB`

This is the most important one to understand first.

Your output:

```text
shared_buffers | 16384 | 8kB
```

PostgreSQL's internal page size is:

```text
8 KB
```

Therefore:

```text
16384 × 8 KB
= 131,072 KB
= 128 MB
```

So:

```text
PostgreSQL Shared Buffers
┌───────────────────────────┐
│                           │
│        128 MB             │
│                           │
│  PostgreSQL data pages    │
│  PostgreSQL index pages   │
│                           │
└───────────────────────────┘
```

### Oracle comparison

Think approximately:

```text
Oracle
SGA
 └── Database Buffer Cache

PostgreSQL
Shared Memory
 └── shared_buffers
```

But there is an important difference:

**PostgreSQL also relies heavily on the Linux filesystem cache.**

So don't think:

```text
PostgreSQL → shared_buffers → disk
```

as the whole story.

A simplified model is:

```text
               PostgreSQL
                    |
             shared_buffers
                    |
             Linux page cache
                    |
                  Disk
```

---

# 2. `wal_buffers = 4 MB`

Your setting:

```text
512 × 8 KB
```

Therefore:

```text
512 × 8 KB = 4 MB
```

Think of it as:

```text
Transaction
     |
     v
WAL records
     |
     v
+----------------+
|  WAL Buffers   |
|     4 MB       |
+----------------+
     |
     v
   pg_wal
```

### Oracle comparison

Very roughly:

```text
Oracle:
Redo Buffer → LGWR → Redo Logs

PostgreSQL:
WAL Buffers → WAL writing → pg_wal
```

Again, this is a conceptual comparison, not an exact implementation mapping.

---

# 3. `work_mem = 4 MB`

This one is **very important for a PostgreSQL DBA**.

You have:

```text
work_mem = 4 MB
```

But don't make this mistake:

> ❌ "PostgreSQL has 4 MB available for queries."

That's incorrect.

It is better to think:

> **Up to approximately 4 MB can be used for an individual query operation before PostgreSQL may need to spill to temporary files.**

For example:

```text
Query
 |
 +-- Sort       → up to work_mem
 |
 +-- Hash Join  → up to work_mem
 |
 +-- Aggregate  → may use work_mem
```

And one query can have multiple operations.

For example:

```text
Query
 |
 +-- Sort
 |     └── work_mem
 |
 +-- Hash Join
 |     └── work_mem
 |
 +-- Aggregate
       └── work_mem
```

And there may be many concurrent sessions.

Therefore, this is dangerous thinking:

```text
100 connections × 4 MB = maximum memory
```

because actual memory usage can be more complicated.

We'll study this deeply during **Day 21 — PostgreSQL Memory & Performance**.

---

# 4. `maintenance_work_mem = 64 MB`

You have:

```text
maintenance_work_mem = 64 MB
```

This is primarily relevant to maintenance operations such as:

```text
VACUUM
CREATE INDEX
ALTER TABLE operations
```

Conceptually:

```text
Normal query
      |
      v
  work_mem

Maintenance
      |
      v
maintenance_work_mem
```

For example, when you create an index:

```sql
CREATE INDEX idx_employee_name
ON employees(name);
```

PostgreSQL may use maintenance memory for the operation.

---

# 5. `effective_cache_size = 4 GB`

This is a **very important distinction**.

You have:

```text
effective_cache_size = 4 GB
```

But this does **NOT** mean PostgreSQL allocated 4 GB of RAM.

🚨 This is probably the most important point about today's output.

`effective_cache_size` is essentially a **planner estimate** of how much cache may be available for PostgreSQL data.

It does not allocate memory.

Compare:

```text
shared_buffers
       ↓
Actually allocated memory

effective_cache_size
       ↓
Planner estimate
       ↓
No memory allocation
```

So:

```text
shared_buffers = 128 MB
```

doesn't conflict with:

```text
effective_cache_size = 4 GB
```

They serve completely different purposes.

---

# 6. Your Memory Architecture

Based on your actual configuration:

```text
                 PostgreSQL 18
                      |
       +--------------+--------------+
       |              |              |
       v              v              v
 shared_buffers   WAL buffers    Per-operation
    128 MB           4 MB         work_mem
                                    4 MB
       |
       |
       v
 PostgreSQL data/index pages
```

And separately:

```text
effective_cache_size = 4 GB
           |
           v
     Query Planner
           |
           v
 "I estimate that this much
  cache may be available."
```

---

# 7. 🔥 Very Important: Don't Tune Yet

Your values are likely **installation/default/lab-oriented values**.

Do **not** immediately say:

> "128 MB is too low; let's increase it."

That's not how a production DBA should work.

First we need to know:

```text
RAM
CPU
Storage
Workload
Connections
Database size
Query patterns
```

Then:

```text
Measure
   ↓
Analyze
   ↓
Tune
   ↓
Measure again
```

For example, check your server RAM:

```bash
free -h
```

And CPU:

```bash
nproc
```

Also:

```bash
lscpu | grep -E 'CPU\(s\)|Model name'
```

---

# 🧪 One More Important Lab

Let's actually see how PostgreSQL uses a table page.

Run:

```sql
CREATE DATABASE dba_lab;
```

Then:

```sql
\c dba_lab
```

Create a table:

```sql
CREATE TABLE employees (
    employee_id integer,
    employee_name varchar(100),
    salary numeric(12,2)
);
```

Insert some rows:

```sql
INSERT INTO employees
SELECT
    g,
    'Employee-' || g,
    50000 + g
FROM generate_series(1,10000) g;
```

Now:

```sql
SELECT count(*) FROM employees;
```

You should get:

```text
10000
```

Now check the table size:

```sql
SELECT pg_size_pretty(pg_total_relation_size('employees'));
```

---

# 🔎 Now See the Query Plan

Run:

```sql
EXPLAIN
SELECT *
FROM employees
WHERE employee_id = 5000;
```

You'll probably get a sequential scan because we haven't created an index:

```text
Seq Scan on employees
```

This is going to lead us into one of the most important PostgreSQL DBA concepts:

```text
SQL
 ↓
Parser
 ↓
Planner
 ↓
Execution Plan
 ↓
Seq Scan / Index Scan
 ↓
Buffers
 ↓
Disk / Cache
```

---

# 🎯 Your Day 2 Mental Model

At this point, I want you to remember these five things:

### `shared_buffers`

**Actual PostgreSQL buffer memory**

```text
128 MB
```

### `wal_buffers`

**WAL-related shared memory**

```text
4 MB
```

### `work_mem`

**Per-operation query working memory**

```text
4 MB
```

### `maintenance_work_mem`

**Memory for maintenance operations**

```text
64 MB
```

### `effective_cache_size`

**Planner estimate, NOT allocated RAM**

```text
4 GB
```

---

## 🚀 Next step

Run these commands and paste the output:

```bash
free -h
```

```bash
nproc
```

```bash
sudo -iu postgres psql -c "SHOW max_connections;"
```

And in PostgreSQL:

```sql
SELECT pg_size_pretty(pg_database_size(current_database()));
```

Then we'll use **your actual server's RAM + CPU + PostgreSQL settings** to understand whether these values make sense.

After that we'll do the really interesting part of Day 2:

**SQL → Parser → Planner → Executor → shared_buffers → WAL → Disk**, including a practical `EXPLAIN` experiment. 🐘🔧
