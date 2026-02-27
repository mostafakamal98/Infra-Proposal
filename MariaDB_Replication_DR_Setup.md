# MariaDB Replication — Production DR Setup

**Master:** `143.198.90.234` (ubuntu-s-4vcpu-8gb-sgp1-01, Singapore)  
**Slave (DR):** `217.15.164.242`  
**MariaDB Version:** 10.11.13  
**OS:** Ubuntu 24.04 LTS (both nodes)  
**Sync Method:** DigitalOcean Snapshot  
**Scope:** All databases  

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Pre-flight Checklist](#2-pre-flight-checklist)
3. [Master Configuration](#3-master-configuration)
4. [Snapshot and Restore](#4-snapshot-and-restore)
5. [Slave Configuration (Post-Restore)](#5-slave-configuration-post-restore)
6. [Start and Verify Replication](#6-start-and-verify-replication)
7. [Test Commands](#7-test-commands)
8. [Ongoing Monitoring](#8-ongoing-monitoring)
9. [Failover — Promote Slave to Master](#9-failover--promote-slave-to-master)
10. [Security Hardening](#10-security-hardening)
11. [Known Constraints](#11-known-constraints)

---

## 1. Architecture Overview

```
┌─────────────────────────────────┐         ┌─────────────────────────────────┐
│         MASTER (Primary)        │         │        SLAVE (DR / Standby)     │
│   143.198.90.234                │         │         217.15.164.242          │
│                                 │         │                                 │
│  server-id = 1                  │─ bin ──▶│  server-id = 2                  │
│  binlog: ROW format             │   log   │  read_only = ON                 │
│  GTID: ON                       │         │  super_read_only = ON           │
│  innodb_buffer_pool = 2G        │         │  log_slave_updates = ON         │
│  DB Size: ~5.6G                 │         │  GTID: slave_pos                │
└─────────────────────────────────┘         └─────────────────────────────────┘
              │                                           │
              └──────── DigitalOcean Private Network ─────┘
                         (use private IPs for replication)
```

**Why `log_slave_updates = ON` on the slave:** When the slave is promoted to master, it must have its own complete binary log history so that any future slave can replicate from it.

---

## 2. Pre-flight Checklist

Run these on the master before touching any config:

```bash
# Confirm MariaDB version
mariadb --version

# Check current disk and memory
df -h /var/lib/mysql
free -m
nproc

# Check if binary logging is already on
mariadb -u root -p -e "SHOW VARIABLES LIKE 'log_bin';"

# Check current GTID state
mariadb -u root -p -e "SHOW VARIABLES LIKE 'gtid%';"

# Check existing server-id
mariadb -u root -p -e "SHOW VARIABLES LIKE 'server_id';"
```

**Note on memory:** Master is currently at ~934 MB available with no swap. Binary logging adds modest overhead. Monitor closely after enabling. Slave should have the same 8 GB RAM spec.

---

## 3. Master Configuration

### 3.1 Edit `/etc/mysql/mariadb.conf.d/50-server.cnf`

Replace or update the `[mysqld]` section:

```ini
[server]

[mysqld]

# ── Identity ──────────────────────────────────────────────
server-id                       = 1
bind-address                    = 0.0.0.0
pid-file                        = /run/mysqld/mysqld.pid
basedir                         = /usr

# ── Character Set ─────────────────────────────────────────
character-set-server            = utf8mb4
collation-server                = utf8mb4_general_ci

# ── InnoDB ────────────────────────────────────────────────
innodb_buffer_pool_size         = 2G
innodb_flush_log_at_trx_commit  = 1
innodb_file_per_table           = ON

# ── Binary Logging ────────────────────────────────────────
log_bin                         = /var/log/mysql/mysql-bin.log
log_bin_index                   = /var/log/mysql/mysql-bin.index
binlog_format                   = ROW
binlog_row_image                = FULL
expire_logs_days                = 7
max_binlog_size                 = 256M
sync_binlog                     = 1

# ── GTID ──────────────────────────────────────────────────
gtid_strict_mode                = ON

# ── Replication ───────────────────────────────────────────
# Required: slave logs its own updates so it can be promoted later
log_slave_updates               = ON

# ── Slow Query Log ────────────────────────────────────────
slow_query_log                  = 1
slow_query_log_file             = /var/log/mysql/mariadb-slow.log
long_query_time                 = 2

# ── Connections ───────────────────────────────────────────
max_connections                 = 200
wait_timeout                    = 600
interactive_timeout             = 600

[embedded]
[mariadb]
[mariadb-10.11]
```

### 3.2 Create log directory and restart

```bash
sudo mkdir -m 2750 /var/log/mysql
sudo chown mysql:mysql /var/log/mysql

sudo systemctl restart mariadb
sudo systemctl status mariadb
```

### 3.3 Verify binary logging is active

```sql
SHOW VARIABLES LIKE 'log_bin';
-- Expected: log_bin | ON

SHOW MASTER STATUS;
-- Expected: shows a binlog file and position
```

### 3.4 Create the replication user

```sql
-- Replace 217.15.164.242 with the slave's private IP on DigitalOcean
-- If both droplets are in the same VPC, private IP is strongly preferred

CREATE USER 'replicator'@'217.15.164.242'
  IDENTIFIED BY 'ReplStr0ng!Pass#2024';

GRANT REPLICATION SLAVE ON *.* TO 'replicator'@'217.15.164.242';

FLUSH PRIVILEGES;

-- Confirm
SELECT user, host FROM mysql.user WHERE user = 'replicator';
```

> **Password:** Change `ReplStr0ng!Pass#2024` to a strong unique password. Store it in your secrets manager (e.g., AWS Secrets Manager or a secure vault) — not in plain text.

---

## 4. Snapshot and Restore

### 4.1 Why snapshot instead of mysqldump

At 5.6 GB, a consistent snapshot is faster and guarantees byte-level consistency across all databases, including InnoDB internal state and GTID position embedded in the data directory. No application downtime risk from long-running dump locks.

### 4.2 Steps in DigitalOcean

1. **Power off the master droplet** from the DigitalOcean control panel (or via CLI).

   ```bash
   sudo poweroff
   ```

2. In DigitalOcean: go to the master droplet > **Snapshots** > **Take Snapshot**. Name it clearly, e.g., `mariadb-master-repl-baseline-2026-02-27`.

3. Wait for the snapshot to complete (typically 5–15 min for a 5.6 GB disk).

4. **Power the master back on.**

5. From the snapshot, **create a new droplet** (the slave):
   - Same region (Singapore) for low-latency replication
   - Same size (4 vCPU / 8 GB RAM) or larger
   - Enable **VPC / private networking** — same VPC as master
   - Do **not** add SSH keys from scratch; the snapshot already has your keys

6. The new droplet starts with an identical copy of the master data.

---

## 5. Slave Configuration (Post-Restore)

SSH into the **slave droplet** (217.15.164.242).

### 5.1 Edit `/etc/mysql/mariadb.conf.d/50-server.cnf`

```ini
[server]

[mysqld]

# ── Identity ──────────────────────────────────────────────
# CRITICAL: Must be different from master (master = 1)
server-id                       = 2
bind-address                    = 0.0.0.0
pid-file                        = /run/mysqld/mysqld.pid
basedir                         = /usr

# ── Read Only (DR slave must never accept writes) ─────────
read_only                       = ON
super_read_only                 = ON

# ── Character Set ─────────────────────────────────────────
character-set-server            = utf8mb4
collation-server                = utf8mb4_general_ci

# ── InnoDB ────────────────────────────────────────────────
innodb_buffer_pool_size         = 2G
innodb_flush_log_at_trx_commit  = 1
innodb_file_per_table           = ON

# ── Relay Log ─────────────────────────────────────────────
relay_log                       = /var/log/mysql/mysql-relay.log
relay_log_index                 = /var/log/mysql/mysql-relay.index
relay_log_recovery              = ON

# ── Binary Logging on Slave (needed for future promotion) ─
log_bin                         = /var/log/mysql/mysql-bin.log
log_bin_index                   = /var/log/mysql/mysql-bin.index
log_slave_updates               = ON
binlog_format                   = ROW
binlog_row_image                = FULL
expire_logs_days                = 7
max_binlog_size                 = 256M
sync_binlog                     = 1

# ── GTID ──────────────────────────────────────────────────
gtid_strict_mode                = ON

# ── Slow Query Log ────────────────────────────────────────
slow_query_log                  = 1
slow_query_log_file             = /var/log/mysql/mariadb-slow.log
long_query_time                 = 2

# ── Connections ───────────────────────────────────────────
max_connections                 = 200
wait_timeout                    = 600
interactive_timeout             = 600

[embedded]
[mariadb]
[mariadb-10.11]
```

### 5.2 Create log directory and restart

```bash
sudo mkdir -m 2750 /var/log/mysql
sudo chown mysql:mysql /var/log/mysql

sudo systemctl restart mariadb
sudo systemctl status mariadb
```

### 5.3 Confirm the GTID position from the snapshot

```sql
-- On SLAVE
SELECT @@gtid_binlog_pos;
-- This shows the GTID embedded in the InnoDB snapshot — this is your sync point
```

---

## 6. Start and Verify Replication

### 6.1 Configure the slave to point to the master

```sql
-- On SLAVE
STOP SLAVE;

CHANGE MASTER TO
  MASTER_HOST          = '143.198.90.234',
  MASTER_USER          = 'replicator',
  MASTER_PASSWORD      = 'ReplStr0ng!Pass#2024',
  MASTER_PORT          = 3306,
  MASTER_USE_GTID      = slave_pos,
  MASTER_CONNECT_RETRY = 10;

START SLAVE;
```

> Use the master's **private/VPC IP** for `MASTER_HOST`, not the public IP. This keeps replication traffic off the public internet and avoids egress costs.

### 6.2 Verify slave status immediately

```sql
SHOW SLAVE STATUS\G
```

**All of these must be true before proceeding:**

| Field | Expected Value |
|---|---|
| `Slave_IO_Running` | `Yes` |
| `Slave_SQL_Running` | `Yes` |
| `Seconds_Behind_Master` | `0` (or trending to 0) |
| `Last_IO_Error` | *(empty)* |
| `Last_SQL_Error` | *(empty)* |
| `Using_Gtid` | `Slave_Pos` |
| `Master_Host` | your master private IP |

---

## 7. Test Commands

### 7.1 GTID position match

```sql
-- On MASTER
SELECT @@gtid_binlog_pos;

-- On SLAVE
SELECT @@gtid_slave_pos;
-- Domain-server-sequence should match or be very close
```

### 7.2 Live write replication test

```sql
-- On MASTER
CREATE DATABASE replication_test;
USE replication_test;
CREATE TABLE ping (
  id   INT PRIMARY KEY AUTO_INCREMENT,
  ts   TIMESTAMP DEFAULT NOW(),
  note VARCHAR(100)
);
INSERT INTO ping (note) VALUES ('replication_check_1');
INSERT INTO ping (note) VALUES ('replication_check_2');
```

```sql
-- On SLAVE (within a few seconds)
USE replication_test;
SELECT * FROM ping;
-- Expected: 2 rows matching master inserts
```

```sql
-- Cleanup on MASTER
DROP DATABASE replication_test;

-- Confirm dropped on SLAVE
SHOW DATABASES;
-- replication_test should be gone
```

### 7.3 Confirm slave is read-only (write must fail)

```sql
-- On SLAVE — this must return an error
INSERT INTO mysql.user (User) VALUES ('test_should_fail');
-- Expected: ERROR 1290 (HY000): The MariaDB server is running with the --read-only option
```

### 7.4 Watch lag in real time

```bash
# On SLAVE — refreshes every 5 seconds
watch -n 5 "mariadb -u root -p'<password>' -e 'SHOW SLAVE STATUS\G' 2>/dev/null \
  | grep -E 'Slave_IO_Running|Slave_SQL_Running|Seconds_Behind|Last_IO_Error|Last_SQL_Error|Using_Gtid'"
```

---

## 8. Ongoing Monitoring

### 8.1 Replication lag alert (simple cron)

Add to `/etc/cron.d/mariadb-replication-check` on the slave:

```bash
# Alert if replication lag exceeds 60 seconds
*/5 * * * * root mariadb -u root -p'<password>' -NBe \
  "SELECT IF(Seconds_Behind_Master > 60, 'LAG_ALERT', 'OK') \
   FROM information_schema.PROCESSLIST LIMIT 1;" 2>/dev/null \
  | grep LAG_ALERT && echo "MariaDB replication lag exceeded 60s on slave 217.15.164.242" \
  | mail -s "REPLICATION LAG ALERT" ops@yourcompany.com
```

### 8.2 Check binary log disk usage on master

```bash
# Run periodically on MASTER
du -sh /var/log/mysql/
mariadb -u root -p -e "SHOW BINARY LOGS;"
```

With 5.6 GB data and `expire_logs_days = 7`, budget approximately 2–5 GB for binlogs depending on write volume.

### 8.3 Slave process check

```bash
# On SLAVE
systemctl is-active mariadb
mariadb -u root -p -e "SHOW SLAVE STATUS\G" | grep -E "Running|Behind|Error"
```

---

## 9. Failover — Promote Slave to Master

Use this procedure when the master is down or unreachable and you need to cut over to DR.

### Step 1 — Stop slave IO and let SQL thread catch up

```sql
-- On SLAVE
STOP SLAVE IO_THREAD;

-- Watch until Seconds_Behind_Master = 0
-- Run repeatedly until confirmed
SHOW SLAVE STATUS\G
```

Wait until `Slave_SQL_Running_State` shows `Slave has read all relay log` and `Seconds_Behind_Master = 0`.

### Step 2 — Stop slave completely and reset replication config

```sql
-- On SLAVE
STOP SLAVE;
RESET SLAVE ALL;
```

### Step 3 — Disable read-only

```sql
SET GLOBAL read_only       = OFF;
SET GLOBAL super_read_only = OFF;
```

Remove these two lines from `50-server.cnf` on the new master:

```ini
# read_only         = ON    <-- remove or comment out
# super_read_only   = ON    <-- remove or comment out
```

### Step 4 — Verify binary logging is active on the new master

```sql
SHOW MASTER STATUS;
-- Must show a binlog file and a GTID position
SHOW VARIABLES LIKE 'log_bin';
-- Must show ON
```

### Step 5 — Repoint applications

Update your application's database connection string (or Route 53 / DigitalOcean DNS record) to `217.15.164.242`. If using an `.env` file:

```env
DB_HOST=217.15.164.242
```

Restart application containers/services after the change.

### Step 6 — Restart MariaDB to lock in config

```bash
sudo systemctl restart mariadb
```

### Step 7 — When you bring up a new slave later

When the original master is restored or a new slave is provisioned, repeat the snapshot process using the new master (217.15.164.242) as the source. The replication user and binlog configuration are already in place.

---

## 10. Security Hardening

### 10.1 Firewall — restrict port 3306

```bash
# On MASTER — allow slave private IP only, deny everything else on 3306
sudo ufw allow from 217.15.164.242 to any port 3306 proto tcp
sudo ufw deny 3306
sudo ufw reload
sudo ufw status
```

```bash
# On SLAVE — 3306 should not be exposed publicly at all
sudo ufw deny 3306
sudo ufw reload
```

### 10.2 Replication user is scoped to slave IP only

The `replicator` user created in Step 3.4 is already locked to `217.15.164.242`. Confirm:

```sql
SELECT user, host, password FROM mysql.user WHERE user = 'replicator';
```

### 10.3 Do not run MariaDB as root

Confirm the process owner:

```bash
ps aux | grep mariadbd
# Expected: mysql    ...  /usr/sbin/mariadbd
```

### 10.4 Secure the root account

```sql
-- On both master and slave
-- Ensure root has no remote access
SELECT user, host FROM mysql.user WHERE user = 'root';
-- All root rows should show 'localhost' or '127.0.0.1' only

DELETE FROM mysql.user WHERE user = 'root' AND host NOT IN ('localhost', '127.0.0.1', '::1');
FLUSH PRIVILEGES;
```

---

## 11. Known Constraints

| Item | Detail |
|---|---|
| No swap on master | Binary logging adds ~50–200 MB overhead. Monitor `free -m` after enabling. If memory drops below 300 MB available, add a 4 GB swapfile or resize the droplet. |
| Public IP replication | If DigitalOcean private networking is not enabled between the two droplets, replication traffic crosses the public internet. Use the private IP wherever possible. Confirm: `ip addr show eth1` should show a private RFC1918 address on both. |
| Single-point binlog | Binlogs are written to `/var/log/mysql` on the master's local disk. If the disk fills up, replication will stop. Watch with `df -h /var/log/mysql`. |
| GTID strict mode | With `gtid_strict_mode = ON`, any out-of-order GTID will hard-stop the slave. This is intentional for data integrity. If you hit GTID errors post-failover, diagnose before skipping. |

---

*Last updated: 2026-02-27 | Environment: DigitalOcean Singapore | MariaDB 10.11.13*
