# Log Capture Reference Guide
## Complete Activity Monitoring Specification

**Document Purpose:** Technical reference for all captured user activities with real-world examples.

---

## Dashboard Panel Overview

The monitoring dashboard contains **30 panels** organized into **6 categories**:

1. **Executive Summary** (4 panels) - High-level metrics
2. **Real-Time Activity** (1 panel) - Live command feed
3. **Security Critical** (2 panels) - Sudo & SSH monitoring
4. **System Changes** (6 panels) - Configuration & package changes
5. **Deployment & Operations** (3 panels) - DevOps activities
6. **File Operations** (4 panels) - File manipulation tracking
7. **Analytics** (2 panels) - Activity distribution & rankings

---

## 1. EXECUTIVE SUMMARY PANELS

### Panel 1.1: Total User Activities (Last 24h)
**Purpose:** Count all user-initiated activities in the selected time range

**Query:** `sum(count_over_time({job="audit", hostname=~"$hostname"}[$__range]))`

**What It Captures:**
- Every command execution by real users
- All activities across selected servers
- Aggregated count for the dashboard time range

**Example Output:**
```
2,847 activities
```

**Use Case:** Quick health check - sudden spikes indicate unusual activity

---

### Panel 1.2: Active Servers
**Purpose:** Count servers that have sent logs in the last 30 days

**Query:** `count(count by (hostname) ({job=~"audit|auth", hostname=~"$hostname"}[30d]))`

**What It Captures:**
- Servers with any activity in last 30 days
- Helps identify offline/disconnected servers

**Example Output:**
```
2 servers
```

**Use Case:** Verify all production servers are reporting

---

### Panel 1.3: Critical Security Events
**Purpose:** Count all sudo command executions

**Query:** `sum(count_over_time({job="audit", hostname=~"$hostname"} |~ "a0=\"sudo\""}[$__range]))`

**What It Captures:**
- Every sudo command execution
- Privilege escalation attempts
- Administrative actions

**Example Output:**
```
156 sudo commands
```

**Use Case:** Monitor privileged access frequency

---

### Panel 1.4: SSH Login Attempts
**Purpose:** Count all SSH authentication events

**Query:** `sum(count_over_time({job="auth", hostname=~"$hostname"} |~ "sshd"}[$__range]))`

**What It Captures:**
- Successful SSH logins
- Failed SSH attempts
- Connection closures

**Example Output:**
```
42 SSH events
```

**Use Case:** Detect brute-force attacks or unusual login patterns

---

## 2. REAL-TIME ACTIVITY STREAM

### Panel 2.1: Live Activity Feed
**Purpose:** Real-time stream of all user commands

**Query:** `{job="audit", hostname=~"$hostname"}`

**What It Captures:**
```
type=EXECVE msg=audit(1771401156.785:254131): argc=3 a0="sudo" a1="apt" a2="install" a3="nginx"
type=EXECVE msg=audit(1771401157.123:254132): argc=2 a0="vim" a1="/etc/nginx/nginx.conf"
type=EXECVE msg=audit(1771401158.456:254133): argc=2 a0="systemctl" a1="restart" a2="nginx"
```

**Fields Explained:**
- `argc`: Number of arguments
- `a0`: Command name
- `a1, a2, a3...`: Command arguments
- `msg=audit(timestamp:event_id)`: Unique event identifier

**Use Case:** Watch commands execute in real-time, immediate threat detection

---

## 3. SECURITY CRITICAL ACTIVITIES

### Panel 3.1: Sudo Command Timeline
**Purpose:** Time-series graph of sudo command frequency

**Query:** `sum by (hostname) (count_over_time({job="audit", hostname=~"$hostname"} |~ "a0=\"sudo\""}[5m]))`

**What It Captures:**
```
10:00 AM - 5 sudo commands (server01)
10:05 AM - 12 sudo commands (server01)
10:10 AM - 3 sudo commands (server02)
```

**Example Commands Captured:**
```
sudo apt update
sudo systemctl restart nginx
sudo vim /etc/hosts
sudo useradd newuser
sudo chmod 755 /opt/app
```

**Use Case:** Identify unusual sudo activity spikes

---

### Panel 3.2: SSH Authentication Heatmap
**Purpose:** Visualize SSH login patterns across servers and time

**Query:** `sum by (hostname) (count_over_time({job="auth", hostname=~"$hostname"} |~ "Accepted|Failed"}[1h]))`

**What It Captures:**
```
Feb 18 10:00 - server01: 3 logins
Feb 18 11:00 - server01: 1 login, server02: 5 logins
Feb 18 12:00 - server02: 2 failed attempts
```

**Example Log Entries:**
```
Accepted publickey for admin from 192.168.1.100 port 54321 ssh2
Failed password for root from 203.0.113.45 port 22 ssh2
Connection closed by 192.168.1.100 port 54321
```

**Use Case:** Detect brute-force attacks, unusual login times

---

## 4. SYSTEM CHANGES

### Panel 4.1: Sudo Command Logs
**Purpose:** Detailed log of all sudo executions

**Query:** `{job="audit", hostname=~"$hostname"} |~ "a0=\"sudo\""`

**Example Captures:**
```
✓ sudo apt install nginx
✓ sudo systemctl restart apache2
✓ sudo vim /etc/ssh/sshd_config
✓ sudo useradd developer
✓ sudo chmod 600 /root/.ssh/authorized_keys
✓ sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT
```

**Use Case:** Audit trail of all privileged operations

---

### Panel 4.2: Authentication Logs (Success & Failed)
**Purpose:** SSH login attempts with source IPs

**Query:** `{job="auth", hostname=~"$hostname"} |~ "Accepted|Failed"`

**Example Captures:**
```
✓ Accepted publickey for admin from 192.168.1.100 port 54321
✗ Failed password for root from 203.0.113.45 port 22
✓ Accepted password for developer from 10.0.0.50 port 45678
✗ Failed publickey for backup from 172.16.0.10 port 33221
```

**Fields Captured:**
- Authentication method (publickey/password)
- Username
- Source IP address
- Source port
- Success/failure status

**Use Case:** Investigate unauthorized access attempts, track user logins

---

### Panel 4.3: Package Installations Timeline
**Purpose:** Track software installation activities

**Query:** `sum by (hostname) (count_over_time({job="audit", hostname=~"$hostname"} |~ "apt|apt-get|yum|dnf|dpkg|rpm"}[5m]))`

**Example Captures:**
```
✓ apt update
✓ apt install nginx
✓ apt-get install mysql-server
✓ yum install httpd
✓ dnf install postgresql
✓ dpkg -i custom-package.deb
✓ rpm -ivh custom-app.rpm
```

**Use Case:** Track software changes, detect unauthorized installations

---

### Panel 4.4: Firewall Modifications
**Purpose:** Monitor firewall rule changes

**Query:** `sum(count_over_time({job="audit", hostname=~"$hostname"} |~ "iptables|ufw|firewalld"}[$__range]))`

**Example Captures:**
```
✓ iptables -A INPUT -p tcp --dport 443 -j ACCEPT
✓ iptables -D INPUT -p tcp --dport 8080 -j ACCEPT
✓ ufw allow 22/tcp
✓ ufw deny from 203.0.113.0/24
✓ firewall-cmd --add-port=3306/tcp --permanent
```

**Use Case:** Security compliance, detect unauthorized firewall changes

---

### Panel 4.5: Configuration File Changes (/etc)
**Purpose:** Track modifications to system configuration files

**Query:** `{job="audit", hostname=~"$hostname"} |~ "/etc/"`

**Example Captures:**
```
✓ vim /etc/nginx/nginx.conf
✓ nano /etc/hosts
✓ vim /etc/ssh/sshd_config
✓ vim /etc/systemd/system/myapp.service
✓ vim /etc/crontab
✓ vim /etc/network/interfaces
```

**Use Case:** Track configuration changes, troubleshoot misconfigurations

---

### Panel 4.6: /var Directory Activity
**Purpose:** Monitor /var directory operations (logs, databases, caches)

**Query:** `{job="audit", hostname=~"$hostname"} |~ "/var/"`

**Example Captures:**
```
✓ rm /var/log/old-logs.tar.gz
✓ touch /var/www/html/index.html
✓ cp backup.sql /var/lib/mysql/
✓ mkdir /var/cache/myapp
```

**Use Case:** Track log manipulation, database operations

---

### Panel 4.7: ~/.ssh Configuration Changes
**Purpose:** Monitor SSH key and config modifications

**Query:** `{job="audit", hostname=~"$hostname"} |~ ".ssh"`

**Example Captures:**
```
✓ vim /root/.ssh/authorized_keys
✓ chmod 600 /root/.ssh/id_rsa
✓ cp id_rsa.pub /root/.ssh/
✓ vim /root/.ssh/config
```

**Use Case:** Detect unauthorized SSH key additions, security audit

---

## 5. DEPLOYMENT & OPERATIONS

### Panel 5.1: Deployment Activity (Git/Docker)
**Purpose:** Track deployment tool usage

**Query:** 
- Git: `sum(count_over_time({job="audit", hostname=~"$hostname"} |~ "a0=\"git\""}[5m]))`
- Docker: `sum(count_over_time({job="audit", hostname=~"$hostname"} |~ "a0=\"docker\""}[5m]))`

**Example Captures:**
```
✓ git clone https://github.com/company/app.git
✓ git pull origin main
✓ git checkout production
✓ docker build -t myapp:latest .
✓ docker run -d -p 8080:8080 myapp:latest
✓ docker stop container_id
✓ docker-compose up -d
✓ kubectl apply -f deployment.yaml
```

**Use Case:** Track deployments, correlate with incidents

---

### Panel 5.2: Systemd Service Operations
**Purpose:** Monitor service start/stop/restart operations

**Query:** `{job="audit", hostname=~"$hostname"} |~ "systemd|systemctl"`

**Example Captures:**
```
✓ systemctl start nginx
✓ systemctl stop apache2
✓ systemctl restart mysql
✓ systemctl enable docker
✓ systemctl disable firewalld
✓ systemctl daemon-reload
```

**Use Case:** Track service changes, troubleshoot outages

---

### Panel 5.3: Cron Job Changes
**Purpose:** Monitor scheduled task modifications

**Query:** `{job="audit", hostname=~"$hostname"} |~ "crontab"`

**Example Captures:**
```
✓ crontab -e (editing user crontab)
✓ crontab -l (listing crontab)
✓ vim /etc/crontab
✓ vim /etc/cron.d/backup
✓ vim /etc/cron.daily/cleanup
```

**Use Case:** Track automation changes, detect malicious cron jobs

---

## 6. FILE OPERATIONS

### Panel 6.1: Permission & Ownership Changes
**Purpose:** Track chmod/chown operations

**Query:** `{job="audit", hostname=~"$hostname"} |~ "a0=\"(chmod|chown|chgrp)\""`

**Example Captures:**
```
✓ chmod 755 /opt/app/script.sh
✓ chmod 600 /root/.ssh/id_rsa
✓ chown www-data:www-data /var/www/html
✓ chown -R mysql:mysql /var/lib/mysql
✓ chgrp developers /opt/project
```

**Use Case:** Security audit, troubleshoot permission issues

---

### Panel 6.2: Text Editor Activity (vim/vi/nano)
**Purpose:** Track file editing sessions

**Query:** `{job="audit", hostname=~"$hostname"} |~ "vim|nano|vi"`

**Example Captures:**
```
✓ vim /etc/nginx/nginx.conf
✓ nano /etc/hosts
✓ vi /opt/app/config.yaml
✓ vim /home/user/script.sh
```

**Use Case:** Track configuration edits, identify who changed what

---

### Panel 6.3: Critical Path Operations (/opt, /srv)
**Purpose:** Monitor application directory changes

**Query:** `{job="audit", hostname=~"$hostname"} |~ "/opt/|/srv/"`

**Example Captures:**
```
✓ cp app.jar /opt/myapp/
✓ rm /opt/old-version/app.jar
✓ mkdir /opt/newapp
✓ vim /opt/myapp/config.properties
✓ chmod +x /opt/scripts/deploy.sh
```

**Use Case:** Track application deployments, detect unauthorized changes

---

### Panel 6.4: File Operations (touch/cp/mv/rm/mkdir)
**Purpose:** Track file creation, copying, moving, deletion

**Query:** `{job="audit", hostname=~"$hostname"} |~ "touch|cp|mv|rm|mkdir"`

**Example Captures:**
```
✓ touch /tmp/test.txt
✓ cp backup.sql /var/backups/
✓ mv old-config.yaml old-config.yaml.bak
✓ rm /tmp/old-file.txt
✓ mkdir /opt/newproject
✓ rm -rf /tmp/cache/*
```

**Use Case:** Track file lifecycle, investigate deletions

---

## 7. ANALYTICS

### Panel 7.1: Most Active Servers (Top 10)
**Purpose:** Rank servers by activity volume

**Query:** `topk(10, sum by (hostname) (count_over_time({job="audit", hostname=~"$hostname"}[$__range])))`

**Example Output:**
```
1. prod-web-01: 1,247 activities
2. prod-db-01: 892 activities
3. prod-app-01: 654 activities
4. staging-web-01: 423 activities
5. prod-api-01: 312 activities
```

**Use Case:** Identify heavily-used servers, capacity planning

---

### Panel 7.2: Activity Distribution by Type
**Purpose:** Pie chart showing activity breakdown

**Queries:**
- Sudo: `sum(count_over_time({job="audit"} |~ "a0=\"sudo\""}[$__range]))`
- SSH: `sum(count_over_time({job="auth"} |~ "sshd"}[$__range]))`
- Packages: `sum(count_over_time({job="audit"} |~ "apt|yum|dnf"}[$__range]))`
- Config: `sum(count_over_time({job="audit"} |~ "/etc/"}[$__range]))`
- Firewall: `sum(count_over_time({job="audit"} |~ "iptables|ufw"}[$__range]))`
- Deployment: `sum(count_over_time({job="audit"} |~ "git|docker|kubectl"}[$__range]))`
- File Ops: `sum(count_over_time({job="audit"} |~ "chmod|chown"}[$__range]))`

**Example Output:**
```
Sudo: 45%
SSH: 20%
Config Changes: 15%
Packages: 10%
Deployment: 5%
File Ops: 3%
Firewall: 2%
```

**Use Case:** Understand activity patterns, optimize monitoring

---

## 8. What Is NOT Captured

### Shell Builtins (Linux Limitation)
These commands don't execute binaries, so they cannot be audited:
```
✗ cd /home/user
✗ echo "test"
✗ export PATH=/usr/bin
✗ alias ll='ls -la'
✗ source ~/.bashrc
```

**Workaround:** Use `history` command on the server for forensics

### System Processes
Filtered out to reduce noise:
```
✗ Browser activity (Chrome, Firefox, Brave)
✗ System daemons (systemd, cron, dbus)
✗ Monitoring tools (Prometheus, node_exporter)
✗ Docker internal operations
✗ Temporary file operations (/tmp, /var/tmp)
```

### Working Directory
The `cwd` (current working directory) is not captured in EXECVE logs to reduce storage.

**Workaround:** Use `ausearch -a <event_id> -i` on the server to see full event details including `cwd`

---

## 9. Forensic Investigation

### When You See Suspicious Activity

**Step 1:** Note the event details from Grafana
```
type=EXECVE msg=audit(1771401156.785:254131): argc=2 a0="rm" a1="/important/file"
```

**Step 2:** SSH to the server
```bash
ssh admin@server01
```

**Step 3:** Get full event details
```bash
sudo ausearch -a 254131 -i
```

**Output:**
```
type=SYSCALL ... auid=admin uid=admin cwd=/home/admin ...
type=EXECVE ... a0="rm" a1="/important/file"
type=PATH ... name="/important/file" ...
```

**Step 4:** Check user's command history
```bash
sudo -u admin history
```

**Step 5:** Check SSH session details
```bash
grep "254131" /var/log/auth.log
```

---

## 10. Query Examples for Grafana Explore

### Find all sudo commands by specific user
```logql
{job="audit"} |~ "auid=john" |~ "a0=\"sudo\""
```

### Find failed SSH attempts from specific IP
```logql
{job="auth"} |~ "Failed" |~ "203.0.113.45"
```

### Find all package installations in last hour
```logql
{job="audit"} |~ "apt install|yum install" [1h]
```

### Find all file deletions
```logql
{job="audit"} |~ "a0=\"rm\""
```

### Find all firewall changes
```logql
{job="audit"} |~ "iptables|ufw|firewalld"
```

---

## 11. Summary

### Total Monitoring Coverage

| Category | Commands Monitored | Example Count/Day |
|----------|-------------------|-------------------|
| Sudo Operations | All sudo executions | 50-100 |
| SSH Access | Login/logout/failed | 20-50 |
| Package Management | apt/yum/dnf/rpm | 5-10 |
| Config Changes | /etc modifications | 10-20 |
| Firewall | iptables/ufw | 1-5 |
| Cron Jobs | crontab edits | 1-3 |
| Services | systemctl operations | 10-20 |
| File Operations | touch/cp/mv/rm/mkdir | 20-40 |
| Permissions | chmod/chown | 5-10 |
| Deployments | git/docker/kubectl | 5-15 |
| Text Editors | vim/nano/vi | 10-20 |
| Critical Paths | /opt, /srv, /var | 5-10 |
| SSH Keys | ~/.ssh changes | 1-2 |

**Total:** ~150-300 meaningful events per server per day

---

**Document Version:** 1.0  
**Last Updated:** February 2026  
**Maintained By:** DevOps Team
