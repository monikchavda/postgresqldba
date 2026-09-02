# 🐘 PostgreSQL 18 Installation — Rocky Linux 9

Clean PGDG RPM installation procedure for a Rocky Linux 9 + PostgreSQL 18 lab environment. PostgreSQL is installed from the official PostgreSQL Yum/PGDG repository, not compiled from source.

## Table of Contents

- [Step 1 — Check Rocky Linux version](#step-1--check-rocky-linux-version)
- [Step 2 — Update the server](#step-2--update-the-server)
- [Step 3 — Install the PostgreSQL PGDG repository](#step-3--install-the-postgresql-pgdg-repository)
- [Step 4 — Disable Rocky's built-in PostgreSQL module](#step-4--disable-rockys-built-in-postgresql-module)
- [Step 5 — Install PostgreSQL 18](#step-5--install-postgresql-18)
- [Step 6 — Verify the PostgreSQL version](#step-6--verify-the-postgresql-version)
- [Step 7 — Initialize the PostgreSQL cluster](#step-7--initialize-the-postgresql-cluster)
- [Step 8 — Check PostgreSQL service](#step-8--check-postgresql-service)
- [Step 9 — Start PostgreSQL](#step-9--start-postgresql)
- [Step 10 — Enable PostgreSQL at boot](#step-10--enable-postgresql-at-boot)
- [Step 11 — Connect to PostgreSQL](#step-11--connect-to-postgresql)
- [Step 12 — Check PostgreSQL version from SQL](#step-12--check-postgresql-version-from-sql)
- [Step 13 — Check the data directory](#step-13--check-the-data-directory)
- [Step 14 — Check configuration files](#step-14--check-configuration-files)
- [Step 15 — Check PostgreSQL processes](#step-15--check-postgresql-processes)
- [Step 16 — Check PostgreSQL using pg_ctl](#step-16--check-postgresql-using-pg_ctl)
- [Step 17 — Check listening port](#step-17--check-listening-port)
- [Step 18 — Create a Test Database](#step-18--create-a-test-database)
- [Step 19 — Basic Installation Verification](#step-19--basic-installation-verification)
- [Complete Installation Sequence](#-complete-installation-sequence)
- [DBA Learning Point](#-dba-learning-point)

---

## Step 1 — Check Rocky Linux version

```bash
cat /etc/os-release
```

You should see something similar to:

```
NAME="Rocky Linux"
VERSION="9.x"
```

Also check architecture:

```bash
uname -m
```

Expected:

```
x86_64
```

## Step 2 — Update the server

```bash
dnf update -y
```

> For a production server, follow your organization's patching/change-management process rather than blindly updating everything.

## Step 3 — Install the PostgreSQL PGDG repository

Install the PostgreSQL repository RPM:

```bash
dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-9-x86_64/pgdg-redhat-repo-latest.noarch.rpm
```

Verify:

```bash
dnf repolist | grep pgdg
```

You should see PostgreSQL PGDG repositories.

## Step 4 — Disable Rocky's built-in PostgreSQL module

Rocky Linux 9 provides an AppStream PostgreSQL module. Since we're using the PGDG repository, disable the built-in module:

```bash
dnf -qy module disable postgresql
```

Verify:

```bash
dnf module list postgresql
```

The built-in module should show as disabled.

## Step 5 — Install PostgreSQL 18

Install the PostgreSQL 18 server and client packages:

```bash
dnf install -y postgresql18-server postgresql18
```

This installs:

- PostgreSQL Server
- PostgreSQL Client
- `psql`
- `pg_ctl`
- `initdb`
- `pg_dump`
- `pg_restore`
- `createdb`
- `createuser`

## Step 6 — Verify the PostgreSQL version

```bash
/usr/pgsql-18/bin/psql --version
```

You should get:

```
psql (PostgreSQL) 18.x
```

Also check the packages:

```bash
rpm -qa | grep postgresql18
```

## Step 7 — Initialize the PostgreSQL cluster

Before PostgreSQL can start for the first time, initialize the database cluster:

```bash
/usr/pgsql-18/bin/postgresql-18-setup initdb
```

You should see something similar to:

```
Initializing database ... OK
```

The default data directory will be:

```
/var/lib/pgsql/18/data/
```

Verify:

```bash
ls -lah /var/lib/pgsql/18/data/
```

You should see:

```
PG_VERSION
base/
global/
pg_wal/
pg_xact/
postgresql.conf
pg_hba.conf
```

## Step 8 — Check PostgreSQL service

The PostgreSQL 18 service is:

```
postgresql-18
```

Check:

```bash
systemctl status postgresql-18
```

Initially, it may show:

```
inactive (dead)
```

That's okay.

## Step 9 — Start PostgreSQL

```bash
systemctl start postgresql-18
```

Check:

```bash
systemctl status postgresql-18
```

You want:

```
Active: active (running)
```

## Step 10 — Enable PostgreSQL at boot

```bash
systemctl enable postgresql-18
```

Verify:

```bash
systemctl is-enabled postgresql-18
```

Expected:

```
enabled
```

## Step 11 — Connect to PostgreSQL

PostgreSQL creates an operating-system user called `postgres`. Switch to that user:

```bash
sudo -iu postgres
```

Then:

```bash
psql
```

You should get something similar to:

```
psql (18.x)
Type "help" for help.

postgres=#
```

## Step 12 — Check PostgreSQL version from SQL

Inside `psql`:

```sql
SELECT version();
```

You can also run:

```sql
SHOW server_version;
```

## Step 13 — Check the data directory

Inside `psql`:

```sql
SHOW data_directory;
```

Initially you should get:

```
/var/lib/pgsql/18/data
```

This is the default `PGDATA`.

## Step 14 — Check configuration files

**PostgreSQL configuration**

```sql
SHOW config_file;
```

Expected:

```
/var/lib/pgsql/18/data/postgresql.conf
```

**HBA configuration**

```sql
SHOW hba_file;
```

Expected:

```
/var/lib/pgsql/18/data/pg_hba.conf
```

**Port**

```sql
SHOW port;
```

Expected:

```
5432
```

**Listening address**

```sql
SHOW listen_addresses;
```

By default you will typically see:

```
localhost
```

## Step 15 — Check PostgreSQL processes

Exit `psql`:

```
\q
```

Then:

```bash
ps -ef | grep '[p]ostgres'
```

You should see several PostgreSQL processes, for example:

```
postgres  xxxx ... postgres -D /var/lib/pgsql/18/data
postgres  xxxx ... postgres: checkpointer
postgres  xxxx ... postgres: background writer
postgres  xxxx ... postgres: walwriter
postgres  xxxx ... postgres: autovacuum launcher
postgres  xxxx ... postgres: logical replication launcher
```

This is an important point for the Day-2 architecture lab.

## Step 16 — Check PostgreSQL using pg_ctl

```bash
sudo -u postgres /usr/pgsql-18/bin/pg_ctl \
-D /var/lib/pgsql/18/data status
```

Expected:

```
pg_ctl: server is running
```

## Step 17 — Check listening port

From Linux:

```bash
ss -lntp | grep 5432
```

You should see PostgreSQL listening on port `5432`.

You can also check from PostgreSQL:

```sql
SHOW port;
```

## Step 18 — Create a Test Database

Connect:

```bash
sudo -iu postgres psql
```

Create a database:

```sql
CREATE DATABASE dba_lab;
```

Check:

```
\l
```

Connect:

```
\c dba_lab
```

Create a test table:

```sql
CREATE TABLE employee
(
    employee_id SERIAL PRIMARY KEY,
    employee_name VARCHAR(100),
    department VARCHAR(50)
);
```

Insert data:

```sql
INSERT INTO employee
(employee_name, department)
VALUES
('Rajesh', 'DBA'),
('Amit', 'Application'),
('Neha', 'Finance');
```

Check:

```sql
SELECT * FROM employee;
```

## Step 19 — Basic Installation Verification

Run these commands:

```bash
systemctl status postgresql-18
/usr/pgsql-18/bin/psql --version
sudo -iu postgres psql -c "SELECT version();"
sudo -iu postgres psql -c "SHOW data_directory;"
sudo -iu postgres psql -c "SHOW config_file;"
sudo -iu postgres psql -c "SHOW hba_file;"
ss -lntp | grep 5432
```

---

## 📋 Complete Installation Sequence

Short version for lab notes — the complete sequence:

```bash
# 1. Update
dnf update -y

# 2. Install PGDG repository
dnf install -y \
https://download.postgresql.org/pub/repos/yum/reporpms/EL-9-x86_64/pgdg-redhat-repo-latest.noarch.rpm

# 3. Disable built-in PostgreSQL module
dnf -qy module disable postgresql

# 4. Install PostgreSQL 18
dnf install -y postgresql18-server postgresql18

# 5. Verify version
/usr/pgsql-18/bin/psql --version

# 6. Initialize database cluster
/usr/pgsql-18/bin/postgresql-18-setup initdb

# 7. Start PostgreSQL
systemctl start postgresql-18

# 8. Enable at boot
systemctl enable postgresql-18

# 9. Check status
systemctl status postgresql-18

# 10. Connect
sudo -iu postgres psql
```

Then inside PostgreSQL, verify:

```sql
SELECT version();
SHOW data_directory;
SHOW config_file;
SHOW hba_file;
SHOW port;
```

---

## 🧠 DBA Learning Point

Installation flow to remember for the course:

```
Rocky Linux 9
      │
      ▼
PGDG Repository
      │
      ▼
PostgreSQL 18 RPM
      │
      ▼
postgresql-18-setup initdb
      │
      ▼
PGDATA
/var/lib/pgsql/18/data
      │
      ▼
systemd
postgresql-18.service
      │
      ▼
PostgreSQL Server
      │
      ├── postgres
      ├── checkpointer
      ├── background writer
      ├── WAL writer
      ├── autovacuum
      └── replication launcher
```

> **Note:** In a follow-on lab, `PGDATA` was changed from `/var/lib/pgsql/18/data` to `/pgdata/18/data` using a systemd drop-in override, and verified with:
>
> ```bash
> sudo -iu postgres psql -c "SHOW data_directory;"
> ```
>
> which returned:
>
> ```
> /pgdata/18/data
> ```
>
> Treat installation + `PGDATA` migration as two distinct, sequential labs.
