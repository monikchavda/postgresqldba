# PostgreSQL 18 — Step-by-Step Data Directory Migration

Move the PostgreSQL 18 data directory from `/var/lib/pgsql/18/data` to `/pgdata/18/data`.

## Environment

| Item | Value |
|---|---|
| OS | Rocky Linux 9 |
| PostgreSQL | 18 |
| User | `postgres` |
| Service | `postgresql-18` |

---

## Step 1 — Stop PostgreSQL

```bash
systemctl stop postgresql-18
systemctl status postgresql-18
```

Expected:

```
Active: inactive (dead)
```

Verify that no PostgreSQL process is running:

```bash
ps -ef | grep '[p]ostgres'
```

There should be no output.

---

## Step 2 — Check the Original Data Directory

systemctl cat postgresql-18|grep environment

### The output is :

[Service]
Type=notify

User=postgres
Group=postgres
# Location of database directory
Environment=PGDATA=/var/lib/pgsql/18/data/





The original `PGDATA` is `/var/lib/pgsql/18/data`.

```bash
ls -lah /var/lib/pgsql/18/data
```

Important files and directories:

- `PG_VERSION`
- `base/`
- `global/`
- `pg_wal/`
- `pg_xact/`
- `postgresql.conf`
- `pg_hba.conf`

---

## Step 3 — Create the New Data Directory

```bash
mkdir -p /pgdata/18/data
ls -ld /pgdata/18/data
```

---

## Step 4 — Install rsync

`rsync` is used because PostgreSQL data contains important permissions, ownership, links, and extended attributes.

```bash
dnf install -y rsync
```

---

## Step 5 — Copy the PostgreSQL Data

```bash
rsync -aHAX --info=progress2 \
  /var/lib/pgsql/18/data/ \
  /pgdata/18/data/
```

> **Important:** note the trailing `/` after the source path `/var/lib/pgsql/18/data/`. This copies the *contents* of `data` into the new directory rather than nesting the directory itself.

---

## Step 6 — Verify the Copy

Compare directory sizes:

```bash
du -sh /var/lib/pgsql/18/data
du -sh /pgdata/18/data
```

Check the new directory:

```bash
ls -lah /pgdata/18/data
```

You should see `PG_VERSION`, `base/`, `global/`, `pg_wal/`, `pg_xact/`, `postgresql.conf`, and `pg_hba.conf`.

---

## Step 7 — Change Ownership

PostgreSQL must own the new directory.

```bash
chown -R postgres:postgres /pgdata/18/data
chmod 700 /pgdata/18/data
ls -ld /pgdata/18/data
```

Expected:

```
drwx------ postgres postgres ... /pgdata/18/data
```

---

## Step 8 — Check SELinux

```bash
getenforce
```

If the result is `Enforcing`, install the required utility:

```bash
dnf install -y policycoreutils-python-utils
```

---

## Step 9 — Configure SELinux Context

Tell SELinux that `/pgdata/18/data` is a PostgreSQL data directory:

```bash
semanage fcontext -a -t postgresql_db_t "/pgdata/18/data(/.*)?"
```

Apply the context:

```bash
restorecon -Rv /pgdata/18/data
```

Verify:

```bash
ls -Zd /pgdata/18/data
```

The output should contain `postgresql_db_t`.

> If `getenforce` returns `Disabled`, skip this step.

---

## Step 10 — Understand the Original systemd Configuration

The PostgreSQL service file is `/usr/lib/systemd/system/postgresql-18.service`.

```bash
systemctl cat postgresql-18
```

The original configuration contains:

```ini
Environment=PGDATA=/var/lib/pgsql/18/data/
```

and:

```ini
ExecStartPre=/usr/pgsql-18/bin/postgresql-18-check-db-dir ${PGDATA}
ExecStart=/usr/pgsql-18/bin/postgres -D ${PGDATA}
```

> **Important:** do **not** modify `/usr/lib/systemd/system/postgresql-18.service` — it is the package-managed service file. Create a systemd drop-in override instead.

---

## Step 11 — Create systemd Override Directory

```bash
mkdir -p /etc/systemd/system/postgresql-18.service.d
```

---

## Step 12 — Create override.conf

The file is created directly instead of using `systemctl edit`, which was failing with:

```
Editing ".../override.conf" canceled: temporary file is empty.
```

Create the file:

```bash
cat > /etc/systemd/system/postgresql-18.service.d/override.conf <<'EOF'
[Service]
Environment=PGDATA=/pgdata/18/data/
EOF
```

---

## Step 13 — Verify override.conf

```bash
cat /etc/systemd/system/postgresql-18.service.d/override.conf
```

Expected:

```ini
[Service]
Environment=PGDATA=/pgdata/18/data/
```

Check the file exists:

```bash
ls -l /etc/systemd/system/postgresql-18.service.d/
```

Expected: `override.conf`

---

## Step 14 — Reload systemd

```bash
systemctl daemon-reload
```

This makes systemd reload the service configuration.

---

## Step 15 — Verify the Effective PGDATA

This is a very important command:

```bash
systemctl show postgresql-18 -p Environment
```

Result:

```
Environment=PGDATA=/pgdata/18/data/ PG_OOM_ADJUST_FILE=/proc/self/oom_score_adj PG_OOM_ADJUST_VALUE=0
```

The important part is `PGDATA=/pgdata/18/data/`, which confirms the effective systemd configuration uses the new directory.

---

## Step 16 — Why `systemctl cat` Still Shows the Old Path

When you run `systemctl cat postgresql-18`, you may still see:

```ini
Environment=PGDATA=/var/lib/pgsql/18/data/
```

This is the original vendor file at `/usr/lib/systemd/system/postgresql-18.service`, and it is not a problem. The override at `/etc/systemd/system/postgresql-18.service.d/override.conf` takes precedence:

```
Vendor service
    |
    | PGDATA=/var/lib/pgsql/18/data
    |
    +----------------------+
                           |
                    systemd override
                           |
                           v
                 PGDATA=/pgdata/18/data
                           |
                           v
                  Effective setting
```

The effective setting is verified with:

```bash
systemctl show postgresql-18 -p Environment
```

---

## Step 17 — Verify the New PostgreSQL Cluster

Before starting PostgreSQL:

```bash
/usr/pgsql-18/bin/pg_ctl \
  -D /pgdata/18/data \
  status
```

Because PostgreSQL is stopped, the expected result is:

```
pg_ctl: no server running
```

This confirms we're checking the correct directory.

---

## Step 18 — Start PostgreSQL

```bash
systemctl start postgresql-18
systemctl status postgresql-18
```

Expected:

```
Active: active (running)
```

---

## Step 19 — Verify PostgreSQL is Using the New Directory

This is the most important verification.

```bash
sudo -iu postgres psql -c "SHOW data_directory;"
```

Result:

```
 data_directory
-----------------
 /pgdata/18/data
(1 row)
```

| | Path |
|---|---|
| Old | `/var/lib/pgsql/18/data` |
| New | `/pgdata/18/data` |

✅ Migration successful.

---

## Step 20 — Verify the Linux PostgreSQL Process

```bash
ps -ef | grep '[p]ostgres'
```

The main process should contain:

```
/usr/pgsql-18/bin/postgres -D /pgdata/18/data/
```

You can also check:

```bash
/usr/pgsql-18/bin/pg_ctl \
  -D /pgdata/18/data \
  status
```

Expected:

```
pg_ctl: server is running (PID: xxxx)
/usr/pgsql-18/bin/postgres "-D" "/pgdata/18/data/"
```

---

## Step 21 — Test PostgreSQL Restart

Do not consider the migration complete until restart works.

```bash
systemctl restart postgresql-18
systemctl status postgresql-18
sudo -iu postgres psql -c "SHOW data_directory;"
```

Expected: `/pgdata/18/data`

---

## Step 22 — Test Rocky Linux Reboot

Once the restart test is successful:

```bash
reboot
```

After the server comes back:

```bash
systemctl status postgresql-18
sudo -iu postgres psql -c "SHOW data_directory;"
```

Expected: `/pgdata/18/data`

---

## Step 23 — Do NOT Delete the Old Directory Immediately

Keep `/var/lib/pgsql/18/data` until you've confirmed the full chain:

```
PostgreSQL starts
        ↓
PostgreSQL restarts
        ↓
Rocky Linux reboots
        ↓
Database is accessible
        ↓
Data is intact
        ↓
Backup works
```

Only after successful validation should you consider removing or archiving the old directory.

---

## Final Configuration

| Item | Value |
|---|---|
| PostgreSQL service | `postgresql-18` |
| Effective PGDATA | `/pgdata/18/data` |
| Systemd override | `/etc/systemd/system/postgresql-18.service.d/override.conf` |
| PostgreSQL binaries | `/usr/pgsql-18/bin/` |
| New data directory | `/pgdata/18/data/` |
| Old data directory | `/var/lib/pgsql/18/data/` |

Override contents:

```ini
[Service]
Environment=PGDATA=/pgdata/18/data/
```

Verification command:

```bash
sudo -iu postgres psql -c "SHOW data_directory;"
```

Expected: `/pgdata/18/data`

---

## 🔑 Key DBA Lesson

The important chain to remember:

```
systemd
   |
   | PGDATA
   ↓
/pgdata/18/data
   |
   | -D ${PGDATA}
   ↓
PostgreSQL 18
   |
   +-- base/
   +-- global/
   +-- pg_wal/
   +-- pg_xact/
   +-- postgresql.conf
   +-- pg_hba.conf
```

Never modify the package-managed service file directly when a systemd drop-in override can be used. This keeps your PostgreSQL configuration maintainable across package upgrades.
