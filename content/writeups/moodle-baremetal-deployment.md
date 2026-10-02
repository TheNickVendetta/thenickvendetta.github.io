---
title: "Production Deployment: Moodle LMS on Bare-Metal Dell PowerEdge R420"
date: 2026-07-30
draft: false
tags: ["System Administration", "Linux", "Bare-Metal", "Moodle", "Apache", "MariaDB", "Performance Tuning", "Network Security"]
summary: "End-to-end bare-metal provisioning, LAMP stack architecture, performance tuning, and automated disaster recovery workflows for an enterprise Moodle LMS deployment."
---

## Executive Summary

During an institutional ICT engineering attachment, I spearheaded the complete deployment lifecycle of a dedicated, campus-wide Learning Management System (LMS) powered by Moodle. The deployment aimed to centralize academic resources across multiple faculties while replacing legacy, fragmented services.

Rather than relying on virtualized resource slices, the system was architected and deployed directly on bare-metal enterprise rack hardware—a **Dell PowerEdge R420**. The entire implementation was executed strictly via headless Linux CLI and secured using industry-standard system hardening practices, least-privilege access controls, and automated operational pipelines.

---

## 1. Hardware Provisioning & Physical Telemetry

### Enterprise Server Baseline
The platform was provisioned on dedicated rack-mounted enterprise hardware:
* **Server Chassis:** Dell PowerEdge R420 (Dual-socket Intel Xeon architecture, Redundant Power Supplies)
* **Storage Subsystem:** Dual 10,000 RPM enterprise SAS drives (Toshiba AL13SEB300, 300GB each) driven by an integrated PERC hardware RAID controller
* **Memory Architecture:** 16GB DDR3 ECC Registered Memory

```text
+--------------------------------------------------------------+
|                     Dell PowerEdge R420                      |
|  [Dual Intel Xeon]   [16GB ECC RAM]   [Redundant PSUs]       |
+--------------------------------------------------------------+
         |                                     |
         v                                     v
+------------------+                 +--------------------+
|  PERC Controller |                 |    iDRAC Out-of-   |
|   (SAS RAID 1)   |                 |   Band Management  |
+------------------+                 +--------------------+
         |
         v
+--------------------------------------------------------------+
| Logical Volume Manager (LVM)                                 |
|  ├── /dev/sda2: /boot (2 GB)                                 |
|  └── /dev/sda3: / (100 GB Root System)                       |
+--------------------------------------------------------------+
```

### Pre-Installation Firmware & Drive Health Triage
Prior to deploying the OS, out-of-band management was established via **iDRAC** to verify hardware event logs (SEL), power supply redundancy, and thermal thresholds. 

Physical drive integrity was systematically audited via the `smartctl` utility (`smartmontools` suite) over the SAS transport interface:

```bash
# Deep SAS SMART health inspection
sudo smartctl -a /dev/sda
```
* **SMART Health Status:** `OK` (0 uncorrected read/write errors)
* **Operating Temperature:** Stable at 35°C
* **Background Diagnostic:** Extended self-test completed with zero defect list growth.

Volume partitioning was structured using Linux **Logical Volume Manager (LVM)**, separating the root filesystem (`/`) and boot boundaries to facilitate dynamic block expansion without downtime.

---

## 2. Base Operating System Hardening & Networking

The server runs a minimal footprint of **Ubuntu Server LTS**, stripping all graphical desktop environments to dedicate compute cycles exclusively to database transactions and web worker threads.

### Administrative Access Hardening
Remote administration was transitioned exclusively to SSH key-pair authentication, disallowing brute-force password surfaces entirely:

```bash
# Generate high-entropy Ed25519 key on admin workstation
ssh-keygen -t ed25519 -C "admin-infra"

# Install public key and restrict daemon
ssh-copy-id -i ~/.ssh/id_ed25519.pub admin@10.20.10.147
```

In `/etc/ssh/sshd_config`, interactive password authentication and root login were explicitly terminated:
```text
PermitRootLogin no
PasswordAuthentication no
X11Forwarding no
MaxAuthTries 3
```

### Network Topology & Static Addressing
Network addressing was configured via Netplan (`/etc/netplan/00-installer-config.yaml`) with static assignments to ensure deterministic DNS resolution and firewall pass-through consistency:

```yaml
network:
  version: 2
  ethernets:
    eno1:
      addresses:
        - 10.20.10.147/24
      routes:
        - to: default
          via: 10.20.10.254
      nameservers:
        addresses:
          - 8.8.8.8
          - 1.1.1.1
```

### Firewall Architecture (UFW)
A strict default-deny perimeter policy was enforced using the Uncomplicated Firewall (UFW):

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 2222/tcp comment 'Hardened SSH Port'
sudo ufw allow 80/tcp comment 'HTTP Web Traffic'
sudo ufw allow 443/tcp comment 'HTTPS Encrypted'
sudo ufw enable
```

---

## 3. LAMP Stack Architecture & Performance Tuning

The application stack consists of **Apache 2.4**, **MariaDB 10.x**, and **PHP 8.x**.

```text
   [ Client HTTPS Requests / Web Browsers ]
                      │
                      ▼
+─────────────────────────────────────────────+
|           UFW Perimeter Firewall            |
|       (Allow: 80, 443, 2222 | Deny All)     |
+─────────────────────────────────────────────+
                      │
                      ▼
+─────────────────────────────────────────────+
|               Apache 2.4                    |
|    (VirtualHost /var/www/html/moodle)       |
|    - mod_rewrite (Clean REST URLs)          |
|    - mod_headers (HTTP Security Headers)    |
+─────────────────────────────────────────────+
                      │
                      ▼
+─────────────────────────────────────────────+
|               PHP 8.x Engine                |
|    (Tuned: 512M Memory | 200M Upload Payload)|
+─────────────────────────────────────────────+
         │                           │
         ▼                           ▼
+──────────────────+       +──────────────────+
|     MariaDB      |       |  /var/moodledata |
| (utf8mb4_unicode)|       | (External Storage|
|   400+ Tables    |       |   Isolated Root) |
+──────────────────+       +──────────────────+
```

### Database Layer Provisioning
To support full multilingual indexing and emoji rendering in modern course communications, MariaDB was configured strictly with `utf8mb4` encoding:

```sql
CREATE DATABASE moodle DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'moodle_admin'@'localhost' IDENTIFIED BY 'Hardened_DB_Pass_2026!';
GRANT ALL PRIVILEGES ON moodle.* TO 'moodle_admin'@'localhost';
FLUSH PRIVILEGES;
```

### PHP Runtime Optimization
Default PHP distribution parameters are notorious for failing under concurrent classroom uploads and background cron executions. The runtime configuration (`php.ini`) was tuned specifically for high-concurrency academic payloads:

| Directive             | Factory Default | Tuned Value | Engineering Rationale                                                        |
| :-------------------- | :-------------- | :---------- | :--------------------------------------------------------------------------- |
| `memory_limit`        | `128M`          | **`512M`**  | Prevents Out-Of-Memory exhaustion during large course backup restorations.   |
| `upload_max_filesize` | `2M`            | **`200M`**  | Accommodates large engineering CAD, ZIP, and slide assignments.              |
| `post_max_size`       | `8M`            | **`200M`**  | Matches POST buffer capacity with multi-file form uploads.                   |
| `max_execution_time`  | `30`            | **`300`**   | Eliminates script timeout aborts during heavy gradebook report calculations. |
| `max_input_vars`      | `1000`          | **`5000`**  | Prevents payload truncation on complex grading forms and large quizzes.      |

---

## 4. Application Deployment & Filesystem Isolation

The stable production branch of Moodle was checked out directly into the document root via Git:

```bash
sudo git clone -b MOODLE_405_STABLE [https://github.com/moodle/moodle.git](https://github.com/moodle/moodle.git) /var/www/html/moodle
```

### Filesystem Security & Path Isolation
In strict compliance with web application security standards, user uploads, dynamic sessions, and caches were isolated entirely outside the exposed web root:
* **Public Web Root:** `/var/www/html/moodle` (Permissions: `755`, owned by `www-data:www-data`)
* **Private Dynamic Data Store:** `/var/moodledata` (Permissions: `770`, strictly owned by `www-data:www-data`)

Placing `moodledata` outside the document root ensures that arbitrary uploaded files (e.g., student assignments or malicious payloads) can never be directly executed via an HTTP request URL.

### Reproducible CLI Installation
Rather than completing the setup via a stateful web browser wizard, deployment was executed via Moodle's native administrative CLI:

```bash
sudo -u www-data /usr/bin/php /var/www/html/moodle/admin/cli/install.php \
  --lang=en \
  --wwwroot=[https://lms.institution.edu.my](https://lms.institution.edu.my) \
  --dataroot=/var/moodledata \
  --dbtype=mariadb \
  --dbhost=localhost \
  --dbname=moodle \
  --dbuser=moodle_admin \
  --dbpass='Hardened_DB_Pass_2026!' \
  --fullname="Institutional LMS Portal" \
  --shortname="LMS" \
  --adminuser=admin \
  --adminpass='Admin_Priv_Pass_2026!' \
  --non-interactive \
  --agree-license
```

---

## 5. Automation, Maintenance & Disaster Recovery

### Asynchronous Task Execution (System Cron)
Moodle requires continuous background task processing for gradebook recalculations, email dispatches, and course backups. This was offloaded to the Linux system daemon rather than web requests:

```bash
# Append to www-data crontab
sudo crontab -u www-data -e

# Run maintenance runner every minute
* * * * * /usr/bin/php /var/www/html/moodle/admin/cli/cron.php >/dev/null 2>&1
```

### Database Backup Pipeline with Retention
To guarantee business continuity, an automated shell script was deployed to perform timestamped database exports with a rolling 14-day purging lifecycle:

```bash
#!/bin/bash
BACKUP_DIR="/var/backups/moodle-db"
TIMESTAMP=$(date +"%F_%H-%M-%S")
mkdir -p "$BACKUP_DIR"

# Generate compressed logical dump
mysqldump -u moodle_admin -p'Hardened_DB_Pass_2026!' moodle | gzip > "$BACKUP_DIR/moodle_db_$TIMESTAMP.sql.gz"

# Enforce 14-day retention cycle
find "$BACKUP_DIR" -type f -name "*.sql.gz" -mtime +14 -delete
```

Log bloat was addressed by creating a dedicated logrotate policy in `/etc/logrotate.d/apache2`, rotating access and error records daily with `gzip` compression.

---

## 6. Functional Verification, Performance Benchmarking & Auditing

### Concurrency Stress Testing (ApacheBench)
To validate infrastructure stability under simultaneous peak login spikes, synthetic load benchmarking was conducted using **ApacheBench (`ab`)** at 100 requests with a concurrency level of 10:

```text
Server Software:        Apache/2.4.66
Server Port:            80
Document Path:          /
Document Length:        1484 bytes

Concurrency Level:      10
Time taken for tests:   0.132 seconds
Complete requests:      100
Failed requests:        0
Total transferred:      173700 bytes
HTML transferred:       148400 bytes
Requests per second:    759.51 [#/sec] (mean)
Time per request:       13.166 [ms] (mean)
Time per request:       1.317 [ms] (mean, across all concurrent requests)
Transfer rate:          1288.35 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0    0   0.2      0       1
Processing:     5   12   3.3     12      20
Waiting:        5   11   3.1     11      20
Total:          6   12   3.3     12      20
```

* **Zero Failed Transactions:** Handled all concurrent requests with 100% completion.
* **Sub-14ms Latency:** Mean latency per request was recorded at **13.16 ms**, comfortably exceeding institutional SLA thresholds.
* **High Transaction Velocity:** Achieved **~760 requests per second** on bare-metal hardware.

### Cross-VLAN Routing & Latency Verification
Layer-3 reachability across distributed campus subnets was confirmed using ICMP echo tests and hop trace diagnostics:

```text
C:\Users\Nick>ping 10.20.10.147
Pinging 10.20.10.147 with 32 bytes of data:
Reply from 10.20.10.147: bytes=32 time=5ms TTL=62
Reply from 10.20.10.147: bytes=32 time=5ms TTL=62
Reply from 10.20.10.147: bytes=32 time=7ms TTL=62
Reply from 10.20.10.147: bytes=32 time=5ms TTL=62

Ping statistics for 10.20.10.147:
    Packets: Sent = 4, Received = 4, Lost = 0 (0% loss),
    Approximate round trip times in milli-seconds:
    Minimum = 5ms, Maximum = 7ms, Average = 5ms
```

### Post-Deployment Verification Checklist

| Operational Component        | Verification Command / Target              | Success Criteria                           | Status     |
| :--------------------------- | :----------------------------------------- | :----------------------------------------- | :--------- |
| **Web Service Connectivity** | `curl -I http://localhost`                 | HTTP 200 OK via Apache runtime listener    | **PASSED** |
| **Automation Persistence**   | `crontab -u www-data -l`                   | CLI cron active on 1-minute schedule       | **PASSED** |
| **Firewall Enforcement**     | `sudo ufw status verbose`                  | Ingress ports restricted to SSH/Web        | **PASSED** |
| **Directory Isolation**      | `/var/moodledata`                          | `www-data` ownership with restricted root  | **PASSED** |
| **Database Integrity**       | `mariadb-check -u root -p --all-databases` | Zero orphaned schemas or table corruptions | **PASSED** |
| **Log Rotation Cycle**       | `/etc/logrotate.d/apache2`                 | Daily rotation enforced with 14-day TTL    | **PASSED** |

---

## Key Takeaways & Operational Impact

1. **Bare-Metal Performance Efficiency:** Moving away from hypervisor emulation eliminated virtualization I/O overhead, providing dedicated access to physical SAS spindles and raw CPU cycles during assessment submission spikes.
2. **Security by Default:** Enforcing strict boundary isolation between web assets and dynamic data stores eliminated primary vector risks associated with unauthenticated arbitrary file uploads.
3. **Reproducibility Through CLI:** Documenting the stack via shell scripts and CLI installation flags ensures that any future hardware refresh or cold-site recovery can be orchestrated within minutes rather than hours.