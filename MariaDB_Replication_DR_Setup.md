# MariaDB Replication — Production DR Setup

**Master:** `10.10.1.10` *(replace with your master private IP)*  
**Slave (DR):** `10.10.1.20` *(replace with your slave private IP)*  
**MariaDB Version:** 10.11.x  
**OS:** Ubuntu 24.04 LTS (both nodes)  
**Sync Method:** mysqldump with GTID (no master downtime required)  
**Scope:** All databases  

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Key Advantage — No Downtime Required](#2-key-advantage--no-downtime-required)
3. [Pre-flight Checklist](#3-pre-flight-checklist)
4. [Master Configuration](#4-master-configuration)
5. [Dump and Restore](#5-dump-and-restore)
6. [Slave Configuration](#6-slave-configuration)
7. [Start and Verify Replication](#7-start-and-verify-replication)
8. [Test Commands](#8-test-commands)
9. [Ongoing Monitoring](#9-ongoing-monitoring)
10. [Failover — Promote Slave to Master](#10-failover--promote-slave-to-master)
11. [Security Hardening](#11-security-hardening)
12. [Troubleshooting — Real Scenarios](#12-troubleshooting--real-scenarios)
13. [Known Constraints](#13-known-constraints)

---

## 1. Architecture Overview

```
┌─────────────────────────────────┐         ┌─────────────────────────────────┐
│         MASTER (Primary)        │         │        SLAVE (DR / Standby)     │
│         10.10.1.10              │         │         10.10.1.20              │
│                                 │         │                                 │
│  server-id = 1                  │─ bin ──▶│  server-id = 2                  │
│  binlog: ROW format             │   log   │  read_only = ON                 │
│  GTID: ON                       │         │  log_slave_updates = ON         │
│  innodb_buffer_pool = 2G        │         │  GTID: slave_pos                │
└─────────────────────────────────┘         └─────────────────────────────────┘
              │                                           │
              └──────────────── Private Network ──────────┘
                         (always use private IPs for replication)
```

**Why `log_slave_updates = ON` on the slave:** When the slave is promoted to master, it must have its own complete binary log history so that any future slave can replicate from it without re-initialising.

---

## 2. Key Advantage — No Downtime Required

> **With GTID and binary logging enabled on the master, you never need to stop or pause the master to set up replication.**

This is one of the most important benefits of this setup. Here is why it works:

- `mariadb-dump --gtid --master-data=2` embeds the exact GTID position at the moment the dump was taken inside the dump file itself
- After restoring the dump on the slave, you use `BINLOG_GTID_POS()` to find that exact position and set it with `SET GLOBAL gtid_slave_pos`
- MariaDB then automatically applies all writes that happened on the master **after** the dump was taken, catching up to real-time

This means you can:
- Take the dump while the application is fully live and writing to the master
- Restore it on the slave at any time
- Start replication and watch `Seconds_Behind_Master` count down to `0`

No maintenance window. No stopping writes. No coordinating with application teams.

---

## 3. Pre-flight Checklist

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

---

## 4. Master Configuration

### 4.1 Edit `/etc/mysql/mariadb.conf.d/50-server.cnf`

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

### 4.2 Create log directory and restart

```bash
sudo mkdir -m 2750 /var/log/mysql
sudo chown mysql:mysql /var/log/mysql

sudo systemctl restart mariadb
sudo systemctl status mariadb
```

### 4.3 Verify binary logging is active

```sql
SHOW VARIABLES LIKE 'log_bin';
-- Expected: log_bin | ON

SHOW MASTER STATUS;
-- Expected: shows a binlog file and position
```

### 4.4 Create the replication user

```sql
-- Replace 10.10.1.20 with your actual slave private IP

CREATE USER 'replicator'@'10.10.1.20'
  IDENTIFIED BY 'ReplStr0ng!Pass#2024';

GRANT REPLICATION SLAVE ON *.* TO 'replicator'@'10.10.1.20';

FLUSH PRIVILEGES;

-- Confirm
SELECT user, host FROM mysql.user WHERE user = 'replicator';
```

> **Password:** Change `ReplStr0ng!Pass#2024` to a strong unique value. Store it in a secrets manager — never in plain text on disk.

---

## 5. Dump and Restore

### 5.1 Why mysqldump instead of a VM snapshot

Using `mariadb-dump` with `--gtid` and `--master-data=2` is the recommended production approach because:

- **No master downtime** — the dump runs while the application is live
- **GTID position is embedded** in the dump file, so the slave knows exactly where to start replication
- Works regardless of cloud provider, VM size, or disk type
- Can be repeated at any time without coordination

### 5.2 Take the dump on the master

The application can keep running during this step.

```bash
mariadb-dump \
  --all-databases \
  --single-transaction \
  --master-data=2 \
  --gtid \
  --flush-logs \
  --routines \
  --triggers \
  --events \
  --hex-blob \
  -u root -p \
  | gzip > /tmp/master_full_$(date +%F).sql.gz
```

`--single-transaction` avoids table locks on InnoDB. `--master-data=2` embeds the binlog position as a comment. `--gtid` embeds the GTID position used to start replication.

### 5.3 Transfer dump to the slave

```bash
scp /tmp/master_full_*.sql.gz root@10.10.1.20:/tmp/
```

### 5.4 Restore on the slave

If your databases contain stored functions or procedures, set `log_bin_trust_function_creators` first — otherwise the restore will fail with error 1418 (see [Troubleshooting](#12-troubleshooting--real-scenarios)).

```bash
# Allow stored function restore
mariadb -u root -p -e "SET GLOBAL log_bin_trust_function_creators=1;"

# Restore
zcat /tmp/master_full_*.sql.gz | mariadb -u root -p

# Disable after restore
mariadb -u root -p -e "SET GLOBAL log_bin_trust_function_creators=0;"
```

### 5.5 Find the GTID position from the dump

After the restore, find the exact GTID position embedded in the dump. First check the binlog file and position from the dump header:

```bash
zcat /tmp/master_full_*.sql.gz | grep -m3 "MASTER_LOG_FILE\|MASTER_LOG_POS\|CHANGE MASTER"
# Output example:
# -- CHANGE MASTER TO MASTER_LOG_FILE='mysql-bin.000002', MASTER_LOG_POS=342;
```

Then convert that to a GTID position on the **master**:

```sql
SELECT BINLOG_GTID_POS('mysql-bin.000002', 342);
-- Returns: 0-1-1323   (use this value in the next section)
```

---

## 6. Slave Configuration

SSH into the slave and apply the following.

### 6.1 Edit `/etc/mysql/mariadb.conf.d/50-server.cnf`

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
# NOTE: super_read_only is NOT a valid config file option in MariaDB 10.11
# Set it dynamically after startup: SET GLOBAL super_read_only = ON;

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

### 6.2 Create log directory and restart

```bash
sudo mkdir -m 2750 /var/log/mysql
sudo chown mysql:mysql /var/log/mysql

sudo systemctl restart mariadb
sudo systemctl status mariadb
```

---

## 7. Start and Verify Replication

### 7.1 Set the GTID position and configure the master connection

```sql
-- On SLAVE
STOP SLAVE;
RESET SLAVE ALL;

-- Paste the value returned by BINLOG_GTID_POS() from Section 5.5
SET GLOBAL gtid_slave_pos = '0-1-1323';

CHANGE MASTER TO
  MASTER_HOST          = '10.10.1.10',
  MASTER_USER          = 'replicator',
  MASTER_PASSWORD      = 'ReplStr0ng!Pass#2024',
  MASTER_PORT          = 3306,
  MASTER_USE_GTID      = slave_pos,
  MASTER_CONNECT_RETRY = 10;

START SLAVE;
```

### 7.2 Verify slave status immediately

```sql
SHOW SLAVE STATUS\G
```

All of these must be true before proceeding:

| Field | Expected Value |
|---|---|
| `Slave_IO_Running` | `Yes` |
| `Slave_SQL_Running` | `Yes` |
| `Seconds_Behind_Master` | counting down toward `0` |
| `Last_IO_Error` | *(empty)* |
| `Last_SQL_Error` | *(empty)* |
| `Using_Gtid` | `Slave_Pos` |
| `Master_Host` | your master private IP |

`Seconds_Behind_Master` will initially show a large number representing all the writes since the dump was taken. This is normal — it counts down to `0` as the slave catches up. As long as both `Running` fields show `Yes`, replication is healthy.

---

## 8. Test Commands

### 8.1 GTID position match

```sql
-- On MASTER
SELECT @@gtid_binlog_pos;

-- On SLAVE
SELECT @@gtid_slave_pos;
-- Domain-server-sequence should match or be very close
```

### 8.2 Live write replication test

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

-- Cleanup on MASTER
DROP DATABASE replication_test;

-- Confirm dropped on SLAVE
SHOW DATABASES;
```

### 8.3 Confirm slave is read-only (this must fail)

```sql
-- On SLAVE
INSERT INTO mysql.user (User) VALUES ('test_should_fail');
-- Expected: ERROR 1290 (HY000): server is running with --read-only option
```

### 8.4 Watch lag in real time

```bash
# On SLAVE — refreshes every 5 seconds
watch -n 5 "mariadb -u root -p'<password>' -e 'SHOW SLAVE STATUS\G' 2>/dev/null \
  | grep -E 'Slave_IO_Running|Slave_SQL_Running|Seconds_Behind|Last_IO_Error|Last_SQL_Error|Using_Gtid'"
```

---

## 9. Ongoing Monitoring

### 9.1 Replication lag alert (cron)

Add to `/etc/cron.d/mariadb-replication-check` on the slave:

```bash
*/5 * * * * root mariadb -u root -p'<password>' -NBe \
  "SELECT IF(Seconds_Behind_Master > 60, 'LAG_ALERT', 'OK') \
   FROM information_schema.PROCESSLIST LIMIT 1;" 2>/dev/null \
  | grep LAG_ALERT && echo "MariaDB replication lag exceeded 60s on slave" \
  | mail -s "REPLICATION LAG ALERT" ops@yourcompany.com
```

### 9.2 Check binary log disk usage on master

```bash
du -sh /var/log/mysql/
mariadb -u root -p -e "SHOW BINARY LOGS;"
```

### 9.3 Slave process check

```bash
systemctl is-active mariadb
mariadb -u root -p -e "SHOW SLAVE STATUS\G" | grep -E "Running|Behind|Error"
```

---

## 10. Failover — Promote Slave to Master

Use this procedure when the master is down and you need to cut over to DR. Follow steps in strict order.

### Step 1 — Stop slave IO and let SQL thread catch up

```sql
-- On SLAVE
STOP SLAVE IO_THREAD;

-- Run repeatedly until confirmed
SHOW SLAVE STATUS\G
-- Wait for: Seconds_Behind_Master = 0
-- Wait for: Slave_SQL_Running_State = Slave has read all relay log
```

### Step 2 — Stop slave and remove replication config

```sql
STOP SLAVE;
RESET SLAVE ALL;
```

### Step 3 — Disable read-only

```sql
SET GLOBAL read_only = OFF;
```

Remove `read_only = ON` from `50-server.cnf` on the new master.

### Step 4 — Verify binary logging is active

```sql
SHOW MASTER STATUS;
-- Must show a binlog file and GTID position

SHOW VARIABLES LIKE 'log_bin';
-- Must show ON
```

### Step 5 — Repoint applications

Update your application DB connection string or DNS record to the new master IP:

```env
DB_HOST=10.10.1.20
```

Restart application containers or services after the change.

### Step 6 — Restart MariaDB to lock in config

```bash
sudo systemctl restart mariadb
```

### Step 7 — When you bring up a new slave later

Use the promoted master as the new source. Take a fresh dump, restore to the new slave, and repeat from Section 5. The replication user and binlog config are already in place.

---

## 11. Security Hardening

### 11.1 Firewall — allow slave IP on port 3306 (master)

If using UFW:

```bash
sudo ufw allow from 10.10.1.20 to any port 3306 proto tcp
sudo ufw deny 3306
sudo ufw reload
sudo ufw status
```

If using iptables (insert before DROP rule):

```bash
sudo iptables -A INPUT -p tcp -s 10.10.1.20 --dport 3306 -j ACCEPT
sudo iptables-save > /etc/iptables/rules.v4
```

### 11.2 Slave — deny port 3306 publicly

```bash
sudo iptables -A INPUT -p tcp --dport 3306 -j DROP
sudo iptables-save > /etc/iptables/rules.v4
```

### 11.3 Confirm replication user scope

```sql
SELECT user, host, password FROM mysql.user WHERE user = 'replicator';
-- host must show only the slave private IP
```

### 11.4 Confirm MariaDB process owner

```bash
ps aux | grep mariadbd
# Expected: mysql   ...  /usr/sbin/mariadbd
```

### 11.5 Lock down root account

```sql
-- On both master and slave
SELECT user, host FROM mysql.user WHERE user = 'root';

DELETE FROM mysql.user
  WHERE user = 'root'
  AND host NOT IN ('localhost', '127.0.0.1', '::1');

FLUSH PRIVILEGES;
```

---

## 12. Troubleshooting — Real Scenarios

This section documents actual issues encountered during setup and how they were resolved.

---

### Issue 1 — `super_read_only=ON` crashes MariaDB on startup

**Symptom:**

```
[ERROR] /usr/sbin/mariadbd: unknown variable 'super_read_only=ON'
mariadb.service: Failed with result 'exit-code'
```

**Root cause:** `super_read_only` is a MySQL variable. It is **not a valid config file option in MariaDB 10.11**. Placing it in `50-server.cnf` causes the server to refuse to start.

**Fix:** Remove `super_read_only = ON` from `50-server.cnf`. Use only `read_only = ON` in the config file. If you want to block SUPER users from writing, set it dynamically after the server is running:

```sql
SET GLOBAL super_read_only = ON;
```

---

### Issue 2 — Error 1418 during restore (stored functions blocked)

**Symptom:**

```
ERROR 1418 (HY000): This function has none of DETERMINISTIC, NO SQL,
or READS SQL DATA in its declaration and binary logging is enabled
```

**Root cause:** MariaDB blocks restoration of stored functions that lack a determinism declaration when binary logging is enabled. This is a safety measure to protect replication integrity.

**Fix:** Set `log_bin_trust_function_creators = 1` before the restore and disable it immediately after:

```bash
mariadb -u root -p -e "SET GLOBAL log_bin_trust_function_creators=1;"
zcat /tmp/master_full_*.sql.gz | mariadb -u root -p
mariadb -u root -p -e "SET GLOBAL log_bin_trust_function_creators=0;"
```

This is safe on a DR slave because no application writes functions to it directly.

---

### Issue 3 — Error 1032 on slave (row not found)

**Symptom:**

```
Last_SQL_Error: Could not execute Delete_rows_v1 event on table db.tablename;
Can't find record in 'tablename', Error_code: 1032; handler error HA_ERR_KEY_NOT_FOUND
Slave_SQL_Running: No
```

**Root cause:** Replication was started without first restoring the dump to the slave. The master's binary log replayed DELETE and UPDATE transactions against rows that did not exist on the empty slave.

**Wrong approach:** Skipping the error with `SET GLOBAL SQL_SLAVE_SKIP_COUNTER`. This does not fix data consistency and leaves the slave unreliable as a DR target.

**Correct fix:**

1. Stop and reset the slave:

```sql
STOP SLAVE;
RESET SLAVE ALL;
```

2. Drop all application databases on the slave (do not drop system databases):

```sql
SHOW DATABASES;
DROP DATABASE your_app_db;
-- repeat for each application database
```

3. Restore from the dump as described in Section 5.
4. Set the GTID position and restart replication as described in Section 7.

---

### Issue 4 — `@@gtid_slave_pos` and `@@gtid_binlog_pos` both empty after restore

**Symptom:** After restoring the dump, both GTID variables return empty on the slave. Replication starts from `mysql-bin.000001` (the beginning of binlog history) instead of the dump point, causing immediate 1032 errors.

**Root cause:** The GTID position embedded in the dump was never explicitly set on the slave via `SET GLOBAL gtid_slave_pos` before running `CHANGE MASTER TO`.

**Fix:** Find the binlog file and position from the dump header:

```bash
zcat /tmp/master_full_*.sql.gz | grep -m3 "MASTER_LOG_FILE\|MASTER_LOG_POS\|CHANGE MASTER"
# Example output:
# -- CHANGE MASTER TO MASTER_LOG_FILE='mysql-bin.000002', MASTER_LOG_POS=342;
```

Convert to GTID on the master:

```sql
SELECT BINLOG_GTID_POS('mysql-bin.000002', 342);
-- Returns: 0-1-1323
```

Then on the slave:

```sql
STOP SLAVE;
RESET SLAVE ALL;

SET GLOBAL gtid_slave_pos = '0-1-1323';

CHANGE MASTER TO
  MASTER_HOST     = '10.10.1.10',
  MASTER_USER     = 'replicator',
  MASTER_PASSWORD = 'ReplStr0ng!Pass#2024',
  MASTER_PORT     = 3306,
  MASTER_USE_GTID = slave_pos;

START SLAVE;
```

---

## 13. Known Constraints

| Item | Detail |
|---|---|
| No swap on master | Binary logging adds ~50-200 MB overhead. Monitor `free -m` after enabling. If available RAM drops below 300 MB consistently, add a 4 GB swapfile or resize the server. |
| Public IP replication | If servers are not on the same private network, replication traffic crosses the public internet. Always use private IPs for `MASTER_HOST` where possible. Confirm with `ip addr show eth1`. |
| Binlog disk on master | Binlogs write to `/var/log/mysql` on the master's local disk. If the disk fills, replication stops. Watch with `df -h /var/log/mysql`. Budget 2-5 GB with `expire_logs_days = 7` depending on write volume. |
| GTID strict mode | With `gtid_strict_mode = ON`, any out-of-order GTID hard-stops the slave. This is intentional for data integrity. If you hit GTID errors post-failover, diagnose before skipping. |
| Replication window | The slave can be offline for up to `expire_logs_days` (default 7 days) before the master purges the binary logs it needs. After that a full re-sync is required. Increase to 14 or 30 days for a DR slave if disk space allows. |

---

*Last updated: 2026-02-28 | MariaDB 10.11.x | Ubuntu 24.04 LTS*
