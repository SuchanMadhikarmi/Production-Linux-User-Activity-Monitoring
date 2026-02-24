# Production Deployment Guide - User Activity Monitoring

## Overview
Monitor user activities across 150+ production servers with centralized logging.

**Architecture:**
```
[Production Servers] → [Auditd + Promtail] → [Loki] → [Grafana]
```

---

## Prerequisites

- Main monitoring server with Prometheus + Grafana already installed
- SSH access to all production servers
- Root/sudo access on all servers

---

## PART 1: Setup Main Monitoring Server

### Step 1: Update Loki URL in Template

**IMPORTANT:** Before deploying, update the Loki URL to your main server IP.

```bash
cd /home/suchan/Devops/lokipromtailpr/

# Replace with your main server IP
sed -i 's|http://202.51.0.104:18789|http://YOUR_MAIN_SERVER_IP:3100|g' promtail-config.yml

# Verify the change
grep "url:" promtail-config.yml
```

### Step 2: Install Loki on Main Server

```bash
# Create Loki directory
sudo mkdir -p /opt/loki
cd /opt/loki

# Create Loki configuration
sudo tee loki-config.yml <<'EOF'
auth_enabled: false

server:
  http_listen_port: 3100
  grpc_listen_port: 9096

common:
  path_prefix: /loki
  storage:
    filesystem:
      chunks_directory: /loki/chunks
      rules_directory: /loki/rules
  replication_factor: 1
  ring:
    kvstore:
      store: inmemory

schema_config:
  configs:
    - from: 2024-01-01
      store: tsdb
      object_store: filesystem
      schema: v13
      index:
        prefix: index_
        period: 24h

storage_config:
  tsdb_shipper:
    active_index_directory: /loki/tsdb-index
    cache_location: /loki/tsdb-cache
  filesystem:
    directory: /loki/chunks

limits_config:
  retention_period: 744h
  ingestion_rate_mb: 50
  ingestion_burst_size_mb: 100

compactor:
  working_directory: /loki/compactor
  compaction_interval: 10m
  retention_enabled: true
  retention_delete_delay: 2h
EOF

# Run Loki container
docker run -d \
  --name loki \
  --restart unless-stopped \
  -p 3100:3100 \
  -v $(pwd)/loki-config.yml:/etc/loki/loki-config.yml \
  -v loki-data:/loki \
  grafana/loki:latest \
  -config.file=/etc/loki/loki-config.yml

# Verify Loki is running
curl http://localhost:3100/ready
```

### Step 3: Add Loki to Grafana

1. Open Grafana UI
2. Go to **Configuration → Data Sources**
3. Click **Add data source**
4. Select **Loki**
5. URL: `http://localhost:3100` (or `http://loki:3100` if same Docker network)
6. Click **Save & Test**

### Step 4: Import Dashboard

1. Go to **Dashboards → Import**
2. Upload: `linux-backupfinal.json`
3. Select Loki datasource
4. Click **Import**

---

## PART 2: Deploy to Production Servers

### Step 1: Install Prerequisites on Each Server

**Ubuntu/Debian:**
```bash
sudo apt update
sudo apt install auditd unzip wget -y
```

**RHEL/CentOS:**
```bash
sudo yum install audit unzip wget -y
```

### Step 2: Install Promtail Binary

```bash
# Download Promtail
wget https://github.com/grafana/loki/releases/download/v2.9.3/promtail-linux-amd64.zip

# Extract and install
unzip promtail-linux-amd64.zip
sudo mv promtail-linux-amd64 /usr/local/bin/promtail
sudo chmod +x /usr/local/bin/promtail

# Verify installation
promtail --version
```

### Step 3: Copy Deployment Package

```bash
# From your PC, copy to each server
scp -r /home/suchan/Devops/lokipromtailpr/ user@server01:/tmp/
```

### Step 4: Run Deployment Script

```bash
# SSH to server
ssh user@server01

# Navigate to deployment directory
cd /tmp/lokipromtailpr

# Make script executable
chmod +x deploy-monitoring.sh

# Run deployment
sudo ./deploy-monitoring.sh
```

**Expected Output:**
```
=== Deploying User Activity Monitoring ===
Server: server01
✓ Promtail restarted
✓ Deployment complete for server01
✓ Audit rules loaded: 70
✓ Check Grafana: {job="audit", hostname="server01"}
```

### Step 5: Create Promtail Systemd Service

```bash
sudo tee /etc/systemd/system/promtail.service <<'EOF'
[Unit]
Description=Promtail Log Shipper
After=network.target auditd.service

[Service]
Type=simple
User=root
ExecStart=/usr/local/bin/promtail -config.file=/etc/promtail/promtail-config.yml
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF

# Reload systemd
sudo systemctl daemon-reload

# Enable and start Promtail
sudo systemctl enable promtail
sudo systemctl start promtail

# Check status
sudo systemctl status promtail
```

### Step 6: Verify Deployment

```bash
# Check audit rules loaded
sudo auditctl -l | wc -l
# Should show: 70

# Check Promtail is running
sudo systemctl status promtail
# Should show: active (running)

# Check Promtail logs
sudo journalctl -u promtail -f
# Should show: "Clients configured" and no errors
```

---

## PART 3: Verify in Grafana

1. Open Grafana dashboard
2. Go to **Explore**
3. Select **Loki** datasource
4. Query: `{job="audit", hostname="server01"}`
5. Should see logs within 30 seconds

**Dashboard URL:** `http://YOUR_GRAFANA_URL/d/linux-user-activity-monitor-v1`

---

## Deployment Checklist

### Main Server:
- [ ] Loki URL updated in `promtail-config.yml`
- [ ] Loki installed and running on port 3100
- [ ] Loki datasource added to Grafana
- [ ] Dashboard imported successfully

### Each Production Server:
- [ ] Auditd installed
- [ ] Promtail binary installed at `/usr/local/bin/promtail`
- [ ] Deployment script executed successfully
- [ ] Promtail systemd service created and running
- [ ] 70 audit rules loaded
- [ ] Logs visible in Grafana

---

## Troubleshooting

### No logs in Grafana?

```bash
# On production server:
# 1. Check audit rules
sudo auditctl -l | wc -l

# 2. Check Promtail status
sudo systemctl status promtail

# 3. Check Promtail logs
sudo journalctl -u promtail -n 50

# 4. Test Loki connectivity
curl -v http://YOUR_MAIN_SERVER_IP:3100/ready

# 5. Generate test activity
sudo ls /etc
```

### Promtail not starting?

```bash
# Check config syntax
/usr/local/bin/promtail -config.file=/etc/promtail/promtail-config.yml -dry-run

# Check permissions
ls -l /etc/promtail/promtail-config.yml
ls -l /var/log/audit/audit.log
```

### Too many logs?

Edit `/etc/audit/rules.d/audit-rules.rules` and add exclusions:
```bash
-a exit,never -F exe=/path/to/noisy/process
```

Then reload:
```bash
sudo augenrules --load
sudo systemctl restart auditd
```

---

## Maintenance

### Update Configuration

```bash
# On production server:
cd /tmp/lokipromtailpr

# Edit files as needed
vim promtail-config.yml
vim audit-rules.rules

# Re-run deployment
sudo ./deploy-monitoring.sh

# Restart services
sudo systemctl restart promtail
```

### Check Disk Usage

```bash
# On main server:
docker exec loki du -sh /loki/*
```

### Backup Configuration

```bash
# Backup from production server:
scp user@server01:/etc/promtail/promtail-config.yml ./backup/
scp user@server01:/etc/audit/rules.d/audit-rules.rules ./backup/
```

---

## Files Included

- `audit-rules.rules` - 70 audit rules for user activity monitoring
- `promtail-config.yml` - Promtail configuration with filters
- `deploy-monitoring.sh` - Automated deployment script
- `linux-backupfinal.json` - Grafana dashboard with 18 panels
- `DEPLOYMENT-GUIDE.md` - This file

---

## Support

For issues or questions:
1. Check Promtail logs: `sudo journalctl -u promtail -f`
2. Check audit logs: `sudo tail -f /var/log/audit/audit.log`
3. Test with: `sudo ls /etc` and verify in Grafana

---

**Deployment Package Location:** `/home/suchan/Devops/lokipromtailpr/`

**Ready to deploy to 150+ production servers!**
