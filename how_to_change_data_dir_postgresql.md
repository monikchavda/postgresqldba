Absolutely. These are two **very important Linux/PostgreSQL DBA concepts**, especially when you move PostgreSQL from its default location to `/data`. Let's understand them from the ground up. 👍

# 1. What is a systemd Override?

Think of the PostgreSQL service configuration as having a **main/default file**.

On your Rocky Linux server, PostgreSQL has:

```text
/usr/lib/systemd/system/postgresql-18.service
```

Inside it, PostgreSQL originally says:

```ini
[Service]
Environment=PGDATA=/var/lib/pgsql/18/data/
```

That means:

> "When systemd starts PostgreSQL, use `/var/lib/pgsql/18/data/` as the database cluster."

But you wanted:

```text
/data
```

### ❌ Don't modify the original file

You could technically edit:

```text
/usr/lib/systemd/system/postgresql-18.service
```

but **you shouldn't**.

Why?

Because `/usr/lib/systemd/system/` contains files installed by RPM packages.

When PostgreSQL is upgraded, the package manager may replace that file.

So your change could disappear.

---

# 2. What is an Override?

Instead of changing the original configuration, systemd allows you to create a **drop-in override**.

You created:

```text
/etc/systemd/system/postgresql-18.service.d/override.conf
```

with:

```ini
[Service]
Environment=PGDATA=/data
```

Think of it like:

```text
Original configuration
        │
        │
        ▼
postgresql-18.service
PGDATA=/var/lib/pgsql/18/data/
        │
        │
        +──────────────┐
                       │
                 Override
                       │
                       ▼
              PGDATA=/data
```

The override tells systemd:

> "Keep the original service configuration, but change this particular setting."

---

# 3. Why is it called a Drop-in?

The directory:

```text
/etc/systemd/system/postgresql-18.service.d/
```

is called a **drop-in directory**.

Inside it you can have:

```text
override.conf
```

or:

```text
database.conf
resources.conf
security.conf
```

For example:

```text
/etc/systemd/system/
└── postgresql-18.service.d/
    ├── override.conf
    ├── resources.conf
    └── security.conf
```

Systemd combines these with the original service configuration.

---

# 4. Why `/etc` overrides `/usr/lib`

Linux generally separates:

```text
/usr/lib
```

and

```text
/etc
```

Conceptually:

| Location                   | Purpose                      |
| --------------------------- | ----------------------------- |
| `/usr/lib/systemd/system/` | Package/vendor configuration |
| `/etc/systemd/system/`     | Administrator configuration  |

So:

```text
/usr/lib/systemd/system/postgresql-18.service
```

is basically:

> PostgreSQL vendor's default configuration.

While:

```text
/etc/systemd/system/postgresql-18.service.d/override.conf
```

is:

> My server-specific configuration.

This is why overrides are preferred. 👍

---

# 5. What does `systemctl daemon-reload` do?

After creating:

```text
override.conf
```

systemd doesn't automatically reread everything.

You run:

```bash
systemctl daemon-reload
```

This means:

> "Systemd, reread your service configuration files."

It does **not** restart PostgreSQL.

That's an important distinction.

```text
daemon-reload
      │
      ▼
Reload systemd configuration
      │
      X
      │
      └── Does NOT restart PostgreSQL
```

Then:

```bash
systemctl restart postgresql-18
```

actually restarts PostgreSQL.

---

# 6. Why did `systemctl cat` show both configurations?

When you eventually ran:

```bash
systemctl cat postgresql-18
```

you saw:

```text
# /usr/lib/systemd/system/postgresql-18.service
```

and at the bottom:

```text
# /etc/systemd/system/postgresql-18.service.d/override.conf
[Service]
Environment=PGDATA=/data
```

That's exactly what we wanted.

It means:

```text
Vendor configuration
       +
Administrator override
       =
Effective PostgreSQL service configuration
```

---

# 7. Now let's understand SELinux 🔐

SELinux is a completely different concept.

SELinux stands for:

> **Security-Enhanced Linux**

Rocky Linux/RHEL commonly has SELinux enabled.

You can check:

```bash
getenforce
```

Possible results:

```text
Enforcing
```

```text
Permissive
```

```text
Disabled
```

---

# 8. Normal Linux permissions vs SELinux

This is extremely important for a DBA.

Suppose:

```text
/data
```

has:

```text
owner = postgres
group = postgres
permissions = 700
```

You might think:

> "Postgres owns the directory, so PostgreSQL can access it."

Not necessarily.

Linux has **two security layers** here.

### Layer 1 — Traditional permissions

```text
-rwx
owner
group
others
```

For example:

```text
postgres postgres 700 /data
```

### Layer 2 — SELinux

SELinux can say:

> "Even though postgres owns this directory, this type of file is not allowed for PostgreSQL."

So:

```text
Linux permissions
        +
SELinux policy
        =
Access decision
```

Both need to permit access.

---

# 9. What is an SELinux context?

You ran:

```bash
ls -Zd /data
```

and got:

```text
unconfined_u:object_r:postgresql_db_t:s0 /data
```

This isn't random information.

It's the **SELinux security context**.

It contains several components:

```text
unconfined_u : object_r : postgresql_db_t : s0
     │             │             │            │
     │             │             │            └─ Security level
     │             │             └────────────── SELinux type
     │             └────────────────────────── role
     └──────────────────────────────────────── user
```

For PostgreSQL, the important part is:

```text
postgresql_db_t
```

That tells SELinux:

> "This is PostgreSQL database data."

---

# 10. Why did we need SELinux when moving `/data`?

PostgreSQL's standard directory is:

```text
/var/lib/pgsql/18/data
```

Rocky/RHEL already knows that this location contains PostgreSQL data.

It therefore has the appropriate SELinux labeling.

Conceptually:

```text
/var/lib/pgsql/18/data
             │
             ▼
      postgresql_db_t
             │
             ▼
       PostgreSQL
          ALLOWED
```

But you created:

```text
/data
```

Initially it might have had a generic SELinux type.

For example:

```text
default_t
```

SELinux could then say:

```text
PostgreSQL process
       │
       ▼
      /data
       │
       ▼
   default_t
       │
       ▼
      DENY ❌
```

---

# 11. What did we do to fix that?

We told SELinux:

```bash
semanage fcontext -a -t postgresql_db_t "/data(/.*)?"
```

This means approximately:

> "Assign the SELinux type `postgresql_db_t` to `/data` and everything underneath it."

Then:

```bash
restorecon -Rv /data
```

actually applies the context.

So now:

```text
/data
  │
  ├── base/
  ├── global/
  ├── pg_wal/
  ├── pg_xact/
  └── ...
       │
       ▼
postgresql_db_t
```

And PostgreSQL is allowed to access it under the SELinux policy.

---

# 12. Why `chown` alone isn't enough

This is a common DBA interview question. 😄

Suppose:

```bash
chown -R postgres:postgres /data
chmod 700 /data
```

Traditional permissions say:

```text
postgres → ALLOWED ✅
```

But SELinux might say:

```text
postgresql process → /data → DENIED ❌
```

Therefore:

```text
chown/chmod
      ≠
SELinux permission
```

You need both.

---

# 13. The complete picture

For your PostgreSQL installation, there are actually **three different things** involved:

### ① systemd

Controls:

> **How PostgreSQL is started**

```text
systemd
   │
   └── PGDATA=/data
```

### ② Linux permissions

Controls:

> **Who owns and can access the files**

```text
/data
owner = postgres
group = postgres
mode  = 700
```

### ③ SELinux

Controls:

> **Whether the PostgreSQL process is allowed to access those files**

```text
/data
SELinux type = postgresql_db_t
```

Together:

```text
                    PostgreSQL
                         │
                         ▼
                    systemd
                         │
                  PGDATA=/data
                         │
                         ▼
                       /data
                    ┌────┴────┐
                    │         │
              Linux ACL    SELinux
                    │         │
              postgres      postgresql_db_t
                    │         │
                    └────┬────┘
                         ▼
                    ACCESS ✅
```

---

# 14. DBA commands to remember 🧠

### systemd

```bash
systemctl status postgresql-18
```

```bash
systemctl cat postgresql-18
```

```bash
systemctl show postgresql-18 -p Environment
```

```bash
systemctl daemon-reload
```

```bash
systemctl restart postgresql-18
```

### Linux permissions

```bash
ls -ld /data
```

```bash
chown -R postgres:postgres /data
```

```bash
chmod 700 /data
```

### SELinux

```bash
getenforce
```

```bash
ls -Zd /data
```

```bash
semanage fcontext -l | grep postgresql
```

```bash
restorecon -Rv /data
```

### PostgreSQL

```bash
sudo -u postgres psql
```

```sql
SHOW data_directory;
```

---

## ⭐ The key concept to remember

When you move PostgreSQL's data directory:

```text
Default:
 /var/lib/pgsql/18/data
             │
             │
             ▼
        PostgreSQL
```

to:

```text
Custom:
 /data
   │
   ├── systemd must know PGDATA=/data
   │
   ├── Linux must allow postgres access
   │
   └── SELinux must label it postgresql_db_t
```

So the DBA checklist is:

> **PGDATA → Ownership → Permissions → SELinux → Start → Verify**

This is a very good topic to include in your **PostgreSQL DBA Day 1/Day 2 notes**, because it teaches not just PostgreSQL, but how PostgreSQL actually integrates with a RHEL/Rocky Linux operating system. 🚀
