# Production Linux User Activity Monitoring
## Loki + Promtail + Auditd for 150+ Servers

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Grafana](https://img.shields.io/badge/Grafana-12.3-orange)](https://grafana.com/)
[![Loki](https://img.shields.io/badge/Loki-2.9.3-blue)](https://grafana.com/oss/loki/)

Real-time security monitoring solution that captures **ONLY user-initiated activities** across 150+ production Linux servers, eliminating 99% of system noise.

---

## 🎯 Problem Solved

Traditional audit logging generates **thousands of events per minute**, with 95%+ being system noise (browsers, daemons, temp files). This solution:

- ✅ Captures **only real user activities** (sudo, SSH, config changes, deployments)
- ✅ Reduces log volume by **95%** (from 100GB/day to 5-10GB/day)
- ✅ Provides **human-readable logs** instead of cryptic audit format
- ✅ Centralizes monitoring for **150+ servers** in one dashboard
- ✅ Enables **forensic investigation** with `ausearch` on individual servers

---

## 📊 What's Monitored

### ✅ Captured Activities
1. **Sudo Commands** - Every sudo execution by real users
2. **SSH Logins** - Authentication events and sessions
3. **User Management** - useradd, userdel, usermod, group changes
4. **Package Management** - apt, yum, dnf, dpkg, rpm installations
5. **Configuration Changes** - Files modified in /etc, /var, /root/.ssh
6. **Network Configuration** - hosts, hostname, network interfaces
7. **Firewall Changes** - iptables, ufw, firewalld modifications
8. **Cron Jobs** - User and system cron modifications
9. **Systemd Services** - Service file changes
10. **Critical Paths** - /opt, /srv file modifications
11. **Permission Changes** - chmod, chown, chgrp by users
12. **Deployment Activities** - git, docker, kubectl executions
13. **File Operations** - vim, nano, rm, touch, mkdir, cp, mv

### ❌ Filtered Out (System Noise)
- System processes (systemd, cron, dbus, snapd)
- Browser activities (Brave, Chrome, Firefox)
- Monitoring stack itself (Promtail, Loki, Grafana)
- Docker/Container internal operations
- Temporary file operations (/tmp, /var/tmp)
- System users (UID < 1000)
- Unattributed activities (auid=4294967295)

---

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Production Servers (150+)                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │   Auditd     │  │   Auditd     │  │   Auditd     │      │
│  │  (70 rules)  │  │  (70 rules)  │  │  (70 rules)  │ ...  │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘      │
│         │                  │                  │              │
│  ┌──────▼───────┐  ┌──────▼───────┐  ┌──────▼───────┐      │
│  │  Promtail    │  │  Promtail    │  │  Promtail    │      │
│  │  (Filters)   │  │  (Filters)   │  │  (Filters)   │ ...  │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘      │
└─────────┼──────────────────┼──────────────────┼─────────────┘
          │                  │                  │
          └──────────────────┼──────────────────┘
                             │
                    ┌────────▼─────────┐
                    │   Main Server    │
                    │  103.90.87.181   │
                    │                  │
                    │  ┌────────────┐  │
                    │  │    Loki    │  │
                    │  │  Port 3100 │  │
                    │  └─────┬──────┘  │
                    │        │         │
                    │  ┌─────▼──────┐  │
                    │  │  Grafana   │  │
                    │  │  Port 3000 │  │
                    │  └────────────┘  │
                    └──────────────────┘
```

---

## 🚀 Quick Start

### Prerequisites
- Main server with Grafana installed
- Root/sudo access to all production servers
- Network connectivity from servers to main server port 3100

### Step 1: Install Loki on Main Server

```bash
# SSH to main server
ssh root@103.90.87.181

# Download Loki
wget https://github.com/grafana/loki/releases/download/v2.9.3/loki-linux-amd64.zip
unzip loki-linux-amd64.zip
sudo mv loki-linux-amd64 /usr/local/bin/loki
sudo chmod +x /usr/local/bin/loki

# Create directories
sudo mkdir -p /etc/loki
sudo mkdir -p /var/lib/loki/{chunks,rules,tsdb-index,tsdb-cache,compactor}

# Copy loki-config.yml to /etc/loki/loki-config.yml

# Create systemd service
sudo tee /etc/systemd/system/loki.service << 'EOF'
[Unit]
Description=Loki Log Aggregation System
After=network.target

[Service]
Type=simple
User=root
ExecStart=/usr/local/bin/loki -config.file=/etc/loki/loki-config.yml
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF

# Start Loki
sudo systemctl daemon-reload
sudo systemctl enable loki
sudo systemctl start loki
sudo systemctl status loki
```

### Step 2: Configure Grafana

1. Open Grafana: `http://103.90.87.181:3000`
2. Go to: **Configuration → Data Sources → Add data source**
3. Select: **Loki**
4. URL: `http://localhost:3100`
5. Click: **Save & Test**

### Step 3: Import Dashboard

1. Go to: **Dashboards → Import**
2. Upload: `lokifinalprod.json`
3. Select Loki data source
4. Click: **Import**

### Step 4: Deploy to Production Servers

```bash
# On each production server
cd /tmp

# Install auditd
sudo apt update
sudo apt install -y auditd audispd-plugins

# Download Promtail
wget https://github.com/grafana/loki/releases/download/v2.9.3/promtail-linux-amd64.zip
unzip promtail-linux-amd64.zip
sudo mv promtail-linux-amd64 /usr/local/bin/promtail
sudo chmod +x /usr/local/bin/promtail

# Extract deployment package
tar -xzf monitoring-deployment.tar.gz

# Deploy
chmod +x deploy-monitoring.sh
sudo ./deploy-monitoring.sh

# Create Promtail systemd service
sudo tee /etc/systemd/system/promtail.service << 'EOF'
[Unit]
Description=Promtail Log Collector
After=network.target auditd.service

[Service]
Type=simple
User=root
ExecStart=/usr/local/bin/promtail -config.file=/etc/promtail/promtail-config.yml
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF

# Start services
sudo systemctl daemon-reload
sudo systemctl enable promtail
sudo systemctl start promtail

# Verify
sudo systemctl status promtail
sudo systemctl status auditd
sudo auditctl -l | wc -l  # Should show ~70
```

---

## 📁 Repository Files

### Core Configuration Files
- **`audit-rules.rules`** - 70 optimized auditd rules for user activity monitoring
- **`promtail-config.yml`** - Promtail configuration with filtering pipeline
- **`loki-config.yml`** - Loki server configuration (31-day retention)
- **`deploy-monitoring.sh`** - Automated deployment script for production servers

### Grafana Dashboards
- **`lokifinalprod.json`** - Production dashboard with 30 panels (Executive Summary, Security, Operations, Analytics)

### Documentation
- **`README.md`** - This file
- **`DEPLOYMENT-GUIDE.md`** - Detailed deployment instructions
- **`EXECUTIVE-SUMMARY.md`** - Business overview for management
- **`LOG-CAPTURE-REFERENCE.md`** - Technical reference with query examples

### Deployment Package
- **`monitoring-deployment.tar.gz`** - Ready-to-deploy package containing:
  - audit-rules.rules
  - promtail-config.yml
  - deploy-monitoring.sh

---

## 🔍 Grafana Dashboard Features

### 30 Monitoring Panels Organized in 6 Categories:

#### 1. Executive Summary (4 panels)
- Total User Activities (Last 24h)
- Active Servers Count
- Critical Security Events
- SSH Login Attempts

#### 2. Real-Time Activity (1 panel)
- Live Activity Stream (last 50 events)

#### 3. Security Critical (2 panels)
- Sudo Command Executions
- SSH Login Activity (Success/Failed)

#### 4. System Changes (6 panels)
- User Management Actions
- Package Installations
- /etc Configuration Changes
- /var Directory Activity
- ~/.ssh Configuration Changes
- Firewall Modifications

#### 5. Deployment & Operations (3 panels)
- Cron Job Modifications
- Systemd Service Changes
- Deployment Tool Usage (git/docker/kubectl)

#### 6. File Operations (4 panels)
- Permission Changes (chmod/chown)
- File Deletions
- Critical Path Activity (/opt, /srv)
- File Editor Usage

#### 7. Analytics (2 panels)
- Most Active Servers (Top 10)
- Activity Distribution by Type

### Dashboard Features
- **Server Filter Dropdown** - Multi-select filter for specific servers
- **Time Range Selector** - View activity from last 5m to 30 days
- **Auto-refresh** - Real-time updates every 10s/30s/1m
- **Annotations** - Automatic alerts for brute force, user changes, firewall modifications

---

## 📊 Sample Queries

### All User Activities
```logql
{job="audit"}
```

### Sudo Commands Only
```logql
{job="audit"} |= "a0=\"sudo\""
```

### SSH Logins
```logql
{job="auth"} |~ "Accepted"
```

### Package Installations
```logql
{job="audit"} |~ "apt|yum|dnf|dpkg|rpm"
```

### File Modifications in /etc
```logql
{job="audit"} |~ "/etc/"
```

### Specific User Activities
```logql
{job="audit", hostname="prod-server-01"}
```

### Last 1 Hour Activities
```logql
{job="audit"} [1h]
```

---

## 🔧 Troubleshooting

### No logs appearing?
```bash
# Check audit rules
sudo auditctl -l | wc -l  # Should show ~70

# Check auditd status
sudo systemctl status auditd

# Check promtail status
sudo systemctl status promtail

# Check promtail logs
sudo journalctl -u promtail -n 50

# Generate test activity
sudo ls /etc
sudo apt update
```

### Logs not in Grafana?
```bash
# Check Loki is running
curl http://localhost:3100/ready

# Check Loki logs
sudo journalctl -u loki -n 50

# Verify network connectivity from server
curl http://103.90.87.181:3100/ready
```

### Too many logs?
Edit `audit-rules.rules` and add exclusions:
```bash
-a exit,never -F exe=/path/to/noisy/process
```

---

## 📈 Performance & Scalability

### Log Volume
- **Before**: ~100-150 GB/day for 150 servers (raw audit logs)
- **After**: ~5-10 GB/day for 150 servers (filtered EXECVE only)
- **Reduction**: 95% storage savings

### Resource Usage (Per Server)
- **Auditd**: ~10-20 MB RAM, negligible CPU
- **Promtail**: ~50-100 MB RAM, <1% CPU
- **Network**: ~1-5 KB/s per server

### Main Server Requirements
- **CPU**: 4+ cores
- **RAM**: 8+ GB
- **Disk**: 500 GB+ (for 31-day retention)
- **Network**: 1 Gbps

---

## 🔐 Security Considerations

### What's NOT Captured
- **Shell builtins** (cd, echo, export, alias) - Cannot be audited
- **Working directory (cwd)** - Dropped for noise reduction
  - Use `ausearch -a <event_id> -i` on server for forensics
- **IP addresses** - Correlate with auth logs: `{job="auth"} |~ "username"`

### Forensic Investigation
When you need full details for an event:
```bash
# On the production server
sudo ausearch -a <audit_event_id> -i

# Example output includes:
# - Full command with all arguments
# - Working directory (cwd)
# - Process ID (pid)
# - Parent process (ppid)
# - User (auid, uid)
# - Timestamp
```

---

## 🛠️ Maintenance

### Weekly
- Check Loki disk usage: `df -h /var/lib/loki`
- Verify all servers reporting: Check hostname dropdown in dashboard

### Monthly
- Review dashboard for anomalies
- Update audit rules if needed
- Check for Promtail/Loki updates

### Adding New Servers
1. Copy `monitoring-deployment.tar.gz` to new server
2. Run deployment commands from Step 4
3. Verify in Grafana hostname dropdown

---

## 📝 License

MIT License - See LICENSE file for details

---

## 👥 Contributing

Contributions welcome! Please:
1. Fork the repository
2. Create a feature branch
3. Submit a pull request

---

## 📧 Support

For issues or questions:
- Open an issue on GitHub
- Check `DEPLOYMENT-GUIDE.md` for detailed instructions
- Review `LOG-CAPTURE-REFERENCE.md` for query examples

---

## 🙏 Acknowledgments

- [Grafana Loki](https://grafana.com/oss/loki/) - Log aggregation system
- [Promtail](https://grafana.com/docs/loki/latest/clients/promtail/) - Log shipping agent
- [Linux Audit Framework](https://linux.die.net/man/8/auditd) - Kernel-level auditing

---

**Built for production environments requiring comprehensive user activity monitoring with minimal overhead.**
