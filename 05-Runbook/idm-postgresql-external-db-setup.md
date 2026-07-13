---
type: runbook
domain:
- forgerock
---



## Install PostgreSQL

Install required PostgreSQL packages

```bash
sudo dnf install -y postgresql postgresql-contrib
sudo dnf install postgresql-server
```

---

## Initialize Database

Initialize PostgreSQL data directory

```bash
sudo postgresql-setup --initdb
```

---

## Enable and Start Service

Enable PostgreSQL service and start it

```bash
sudo systemctl enable postgresql
sudo systemctl start postgresql
sudo systemctl status postgresql
```

---

## Configure Network and Logging

Allow external connections and enable logging

```bash
sudo vi /var/lib/pgsql/data/postgresql.conf
sudo vi /var/lib/pgsql/data/pg_hba.conf
sudo systemctl restart postgresql
```

### postgresql.conf

```text
listen_addresses = '*'

log_destination = 'stderr'
log_directory = 'log'
log_filename = 'postgresql-%a.log'
log_truncate_on_rotation = on
log_rotation_age = 1d
log_rotation_size = 0
logging_collector = on  
log_connections = on  
log_disconnections = on
```

### pg_hba.conf

```text
host    all    all    192.168.100.0/24    md5
```

---

## Access PostgreSQL Shell

Switch to postgres user and open shell

```bash
sudo -u postgres psql
```

---

## Create IDM Database

```sql
CREATE DATABASE idm
  WITH ENCODING 'UTF8'
  LC_COLLATE='en_US.UTF-8'
  LC_CTYPE='en_US.UTF-8'
  TEMPLATE template0;
```

---

## Create IDM User

```sql
CREATE USER idm_user WITH PASSWORD 'flight@123';
```

---

## Grant Database Privileges

```sql
GRANT ALL PRIVILEGES ON DATABASE idm TO idm_user;
```

---

## Create Schema

```sql
CREATE SCHEMA idm AUTHORIZATION idm_user;
```

---

## Set Default Schema

```sql
ALTER ROLE idm_user SET search_path TO idm;
```

---

## Grant Schema Privileges

```sql
GRANT ALL ON SCHEMA idm TO idm_user;
```

---

## Set Default Privileges

```sql
ALTER DEFAULT PRIVILEGES IN SCHEMA idm
GRANT ALL ON TABLES TO idm_user;

ALTER DEFAULT PRIVILEGES IN SCHEMA idm
GRANT ALL ON SEQUENCES TO idm_user;
```

---

## Verify Database Objects

```sql
\l
\dn
\du
\dt
```

---

## Test Connectivity from Host

```bash
psql -h 192.168.100.169 -U idm_user -d idm
```

---

## Test Connectivity from Kubernetes Pod

```bash
kubectl run psql-test --rm -it --image=postgres:13 -- bash
psql -h 192.168.100.169 -U idm_user -d idm
```

---

## Verify PostgreSQL Logs

```bash
tail -f /var/lib/pgsql/data/log/*
```

---

