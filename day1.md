PostgreSQL 18 — Step-by-Step Data Directory Migration
Objective

Move the PostgreSQL 18 data directory from:

/var/lib/pgsql/18/data

to:

/pgdata/18/data

Environment:

OS         : Rocky Linux 9
PostgreSQL : 18
User       : postgres
Service    : postgresql-18
Step 1 — Stop PostgreSQL

Stop the PostgreSQL service:

systemctl stop postgresql-18

Verify:

systemctl status postgresql-18

Expected:

Active: inactive (dead)

Verify that no PostgreSQL process is running:

ps -ef | grep '[p]ostgres'

There should be no output.

Step 2 — Check the Original Data Directory

The original PGDATA is:

/var/lib/pgsql/18/data

Check:

ls -lah /var/lib/pgsql/18/data

Important files/directories:

PG_VERSION
base/
global/
pg_wal/
pg_xact/
postgresql.conf
pg_hba.conf
Step 3 — Create the New Data Directory

Create the new directory:

mkdir -p /pgdata/18/data

Verify:

ls -ld /pgdata/18/data
Step 4 — Install rsync

We used rsync because PostgreSQL data contains important permissions, ownership, links and extended attributes.

Install:

dnf install -y rsync
Step 5 — Copy the PostgreSQL Data

Copy the complete PostgreSQL cluster:

rsync -aHAX --info=progress2 \
/var/lib/pgsql/18/data/ \
/pgdata/18/data/
Important

Notice the / after the source:

/var/lib/pgsql/18/data/

This copies the contents of data into the new directory.

Step 6 — Verify the Copy

Compare directory sizes:

du -sh /var/lib/pgsql/18/data
du -sh /pgdata/18/data

Check the new directory:

ls -lah /pgdata/18/data

You should see:

PG_VERSION
base/
global/
pg_wal/
pg_xact/
postgresql.conf
pg_hba.conf
Step 7 — Change Ownership

PostgreSQL must own the new directory.

chown -R postgres:postgres /pgdata/18/data

Set permissions:

chmod 700 /pgdata/18/data

Verify:

ls -ld /pgdata/18/data

Expected:

drwx------ postgres postgres ... /pgdata/18/data
Step 8 — Check SELinux

Check SELinux status:

getenforce

If the result is:

Enforcing

install the required utility:

dnf install -y policycoreutils-python-utils
Step 9 — Configure SELinux Context

Tell SELinux that /pgdata/18/data is a PostgreSQL data directory:

semanage fcontext -a -t postgresql_db_t "/pgdata/18/data(/.*)?"

Apply the context:

restorecon -Rv /pgdata/18/data

Verify:

ls -Zd /pgdata/18/data

The output should contain:

postgresql_db_t
If SELinux is disabled

If:

getenforce

returns:

Disabled

skip this step.

Step 10 — Understand the Original systemd Configuration

The PostgreSQL service file is:

/usr/lib/systemd/system/postgresql-18.service

Check it:

systemctl cat postgresql-18

The original configuration contains:

Environment=PGDATA=/var/lib/pgsql/18/data/

and:

ExecStartPre=/usr/pgsql-18/bin/postgresql-18-check-db-dir ${PGDATA}
ExecStart=/usr/pgsql-18/bin/postgres -D ${PGDATA}
Important

We do not modify:

/usr/lib/systemd/system/postgresql-18.service

because it is the package-managed service file.

Instead, we create a systemd drop-in override.

Step 11 — Create systemd Override Directory

Create:

mkdir -p /etc/systemd/system/postgresql-18.service.d
Step 12 — Create override.conf

We created the file directly instead of using systemctl edit because the editor was giving:

Editing ".../override.conf" canceled: temporary file is empty.

Create the file:

cat > /etc/systemd/system/postgresql-18.service.d/override.conf <<'EOF'
[Service]
Environment=PGDATA=/pgdata/18/data/
EOF
Step 13 — Verify override.conf

Run:

cat /etc/systemd/system/postgresql-18.service.d/override.conf

Expected:

[Service]
Environment=PGDATA=/pgdata/18/data/

Check the file:

ls -l /etc/systemd/system/postgresql-18.service.d/

Expected:

override.conf
Step 14 — Reload systemd

After changing the systemd configuration:

systemctl daemon-reload

This makes systemd reload the service configuration.

Step 15 — Verify the Effective PGDATA

This is a very important command:

systemctl show postgresql-18 -p Environment

Our result was:

Environment=PGDATA=/pgdata/18/data/ PG_OOM_ADJUST_FILE=/proc/self/oom_score_adj PG_OOM_ADJUST_VALUE=0

The important part is:

PGDATA=/pgdata/18/data/

This confirms that the effective systemd configuration uses the new directory.

Step 16 — Why systemctl cat Still Shows the Old Path

When you run:

systemctl cat postgresql-18

you may still see:

Environment=PGDATA=/var/lib/pgsql/18/data/

This is the original vendor file:

/usr/lib/systemd/system/postgresql-18.service

That is not a problem.

The override is:

/etc/systemd/system/postgresql-18.service.d/override.conf

with:

[Service]
Environment=PGDATA=/pgdata/18/data/

Conceptually:

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

The effective setting is verified with:

systemctl show postgresql-18 -p Environment
Step 17 — Verify the New PostgreSQL Cluster

Before starting PostgreSQL:

/usr/pgsql-18/bin/pg_ctl \
-D /pgdata/18/data \
status

Because PostgreSQL is stopped, the expected result is:

pg_ctl: no server running

This confirms we're checking the correct directory.

Step 18 — Start PostgreSQL

Start the service:

systemctl start postgresql-18

Check:

systemctl status postgresql-18

Expected:

Active: active (running)
Step 19 — Verify PostgreSQL is Using the New Directory

This is the most important verification.

Run:

sudo -iu postgres psql -c "SHOW data_directory;"

Our successful result was:

 data_directory
-----------------
 /pgdata/18/data
(1 row)

Therefore:

Old:
 /var/lib/pgsql/18/data


New:
 /pgdata/18/data

✅ Migration successful.

Step 20 — Verify the Linux PostgreSQL Process

Run:

ps -ef | grep '[p]ostgres'

The main process should contain:

/usr/pgsql-18/bin/postgres -D /pgdata/18/data/

You can also check:

/usr/pgsql-18/bin/pg_ctl \
-D /pgdata/18/data \
status

Expected:

pg_ctl: server is running (PID: xxxx)
/usr/pgsql-18/bin/postgres "-D" "/pgdata/18/data/"
Step 21 — Test PostgreSQL Restart

Do not consider the migration complete until restart works.

Run:

systemctl restart postgresql-18

Check:

systemctl status postgresql-18

Then:

sudo -iu postgres psql -c "SHOW data_directory;"

Expected:

/pgdata/18/data
Step 22 — Test Rocky Linux Reboot

Once the restart test is successful:

reboot

After the server comes back:

systemctl status postgresql-18

Then:

sudo -iu postgres psql -c "SHOW data_directory;"

Expected:

/pgdata/18/data
Step 23 — Do NOT Delete the Old Directory Immediately

Keep the old directory:

/var/lib/pgsql/18/data

until you've confirmed:

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

Only after successful validation should you consider removing or archiving the old directory.

Final Configuration
PostgreSQL service
postgresql-18
Effective PGDATA
/pgdata/18/data
Systemd override
/etc/systemd/system/postgresql-18.service.d/override.conf

Contents:

[Service]
Environment=PGDATA=/pgdata/18/data/
PostgreSQL binaries
/usr/pgsql-18/bin/
New data directory
/pgdata/18/data/
Old data directory
/var/lib/pgsql/18/data/
Verification command
sudo -iu postgres psql -c "SHOW data_directory;"

Expected:

/pgdata/18/data
🔑 Key DBA Lesson

The important chain to remember is:

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

Never modify the package-managed service file directly when a systemd drop-in override can be used. This keeps your PostgreSQL configuration maintainable across package upgrades.
