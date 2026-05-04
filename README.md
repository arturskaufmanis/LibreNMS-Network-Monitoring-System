# LibreNMS Network Monitoring System — Homelab Deployment

## Project Overview

This project documents the full deployment of **LibreNMS** on a Rocky Linux 9.7 homelab node, providing live network monitoring, device availability mapping, and SNMP-based infrastructure visibility across a mixed Linux/Windows environment.

LibreNMS is an open-source, agentless network management system (NMS) that uses SNMP to discover and monitor network devices, servers, and infrastructure. It provides real-time topology maps, interface traffic graphs, CPU/memory health monitoring, and alerting.

---

## Lab Environment

### Network Topology

```
ISP
 │
enp1s0 (Topton N150)
 │
br0 (transparent bridge — Suricata IDS in promiscuous mode)
 │
enp2s0 (Topton N150) ──── WAN port (EE Smart Hub 7HC25H)
                                  │
                     ┌────────────┴────────────┐
                     │                         │
               enp3s0 (Topton)         TAROX PC (ethernet)
               management loop         Windows Server 2025

Traffic flow: Internet → enp1s0 → br0 → enp2s0 → Router WAN → Router LAN → enp3s0 → internal network
```

### Hardware Inventory

| Device | Role | IP Address |
|--------|------|------------|
| Topton N150 Mini PC | Network TAP / IDS / LibreNMS host | 192.168.1.217 |
| TAROX PC (NUC7i5BNH) | Windows Server 2025 DC / Splunk SIEM | 192.168.1.10 |
| EE Smart Hub 7HC25H | WiFi Router / Gateway / DHCP | 192.168.1.1 |

### Topton N150 Specification

| Spec | Detail |
|------|--------|
| CPU | Intel N150 (quad-core) |
| RAM | 32 GB DDR5 4800 MHz |
| Storage | 1 TB NVMe |
| OS | Rocky Linux 9.7 (Blue Onyx) |
| NICs | 4x Ethernet (enp1s0, enp2s0, enp3s0, enp4s0) |
| Bridge | br0 (transparent — WAN-side and LAN-side ports) |
| Management | Cockpit — https://192.168.1.217:9090 |

---

## Software Stack

| Component | Version | Purpose |
|-----------|---------|---------|
| LibreNMS | Latest (daily channel) | NMS platform |
| MariaDB | 10.5.29 | Database backend |
| Nginx | 1.20.1 | Web server |
| PHP | 8.2.30 (Remi) | Application runtime |
| net-snmp | 5.9.1 | SNMP daemon |
| python3-PyMySQL | 0.10.1 | Python DB connector for poller |
| python3-dotenv | 0.19.2 | Environment file loader for poller |

---

## Prerequisites

### Rocky Linux 9 with the following repositories enabled

```bash
sudo dnf install -y epel-release
sudo dnf install -y https://rpms.remirepo.net/enterprise/remi-release-9.rpm
sudo dnf module reset php -y
sudo dnf module enable php:remi-8.2 -y
```

---

## Installation Steps

### Step 1 — Install Dependencies

```bash
sudo dnf install -y git curl unzip
sudo dnf install -y nginx mariadb-server
sudo dnf install -y php php-fpm php-cli php-mysqlnd php-gd php-mbstring php-xml php-zip php-bcmath php-snmp php-curl php-opcache
sudo dnf install -y net-snmp net-snmp-utils fping mtr rrdtool ImageMagick python3
sudo dnf install -y python3-PyMySQL python3-dotenv
```

### Step 2 — Configure MariaDB

```bash
sudo systemctl start mariadb
sudo systemctl enable mariadb
sudo mysql_secure_installation
```

When prompted:
- Switch to unix_socket authentication: **Y**
- Change root password: **n**
- Remove anonymous users: **Y**
- Disallow root login remotely: **Y**
- Remove test database: **Y**
- Reload privilege tables: **Y**

Create the LibreNMS database:

```bash
sudo mysql
```

```sql
CREATE DATABASE librenms CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'librenms'@'localhost' IDENTIFIED BY 'YourStrongPassword!';
GRANT ALL PRIVILEGES ON librenms.* TO 'librenms'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

### Step 3 — Create LibreNMS System User and Clone Repository

```bash
sudo useradd librenms -d /opt/librenms -M -r -s /usr/bin/bash
sudo git clone https://github.com/librenms/librenms.git /opt/librenms
sudo chown -R librenms:librenms /opt/librenms
sudo chmod 771 /opt/librenms
```

Install PHP dependencies:

```bash
cd /opt/librenms
sudo -u librenms ./scripts/composer_wrapper.php install --no-dev
```

### Step 4 — Fix Directory Permissions and SELinux

```bash
sudo mkdir -p /opt/librenms/rrd /opt/librenms/logs /opt/librenms/bootstrap/cache /opt/librenms/storage
sudo chown -R librenms:librenms /opt/librenms
sudo setfacl -d -m g::rwx /opt/librenms/rrd /opt/librenms/logs /opt/librenms/bootstrap/cache /opt/librenms/storage
sudo setfacl -R -m g::rwx /opt/librenms/rrd /opt/librenms/logs /opt/librenms/bootstrap/cache /opt/librenms/storage

sudo semanage fcontext -a -t httpd_sys_rw_content_t '/opt/librenms/logs(/.*)?'
sudo semanage fcontext -a -t httpd_sys_rw_content_t '/opt/librenms/bootstrap/cache(/.*)?'
sudo semanage fcontext -a -t httpd_sys_rw_content_t '/opt/librenms/storage(/.*)?'
sudo semanage fcontext -a -t httpd_sys_rw_content_t '/opt/librenms/rrd(/.*)?'
sudo restorecon -RFvv /opt/librenms
```

### Step 5 — Configure PHP-FPM

```bash
sudo cp /etc/php-fpm.d/www.conf /etc/php-fpm.d/www.conf.bak
sudo sed -i 's/^user = apache/user = librenms/' /etc/php-fpm.d/www.conf
sudo sed -i 's/^group = apache/group = librenms/' /etc/php-fpm.d/www.conf
sudo sed -i 's/^listen = .*/listen = \/run\/php-fpm\/librenms.sock/' /etc/php-fpm.d/www.conf
sudo sed -i 's/^;listen.owner/listen.owner/' /etc/php-fpm.d/www.conf
sudo sed -i 's/^;listen.group/listen.group/' /etc/php-fpm.d/www.conf
sudo sed -i 's/^;listen.mode/listen.mode/' /etc/php-fpm.d/www.conf
sudo systemctl enable php-fpm
sudo systemctl start php-fpm
```

### Step 6 — Configure Nginx

```bash
sudo nano /etc/nginx/conf.d/librenms.conf
```

Paste the following:

```nginx
server {
    listen 80;
    server_name 192.168.1.217;
    root /opt/librenms/html;
    index index.php;

    charset utf-8;
    gzip on;
    gzip_types text/css application/javascript text/javascript application/x-javascript image/svg+xml text/plain text/xsd text/xsl text/xml image/x-icon;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ [^/]\.php(/|$) {
        fastcgi_pass unix:/run/php-fpm/librenms.sock;
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        include fastcgi.conf;
    }

    location ~ /\.(?!well-known).* {
        deny all;
    }
}
```

```bash
sudo nginx -t
sudo systemctl enable nginx
sudo systemctl start nginx
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --reload
```

### Step 7 — Configure SNMP on Topton

```bash
sudo tee /etc/snmp/snmpd.conf << 'EOF'
agentAddress udp:161
rocommunity public 192.168.1.0/24
syslocation Homelab - Topton N150
syscontact SecAdmin
EOF

sudo systemctl enable snmpd
sudo systemctl start snmpd
sudo firewall-cmd --permanent --add-service=snmp
sudo firewall-cmd --reload
```

### Step 8 — Set Up Cron Jobs and Scheduler

```bash
sudo cp /opt/librenms/dist/librenms.cron /etc/cron.d/librenms
sudo cp /opt/librenms/dist/librenms-scheduler.service /etc/systemd/system/
sudo cp /opt/librenms/dist/librenms-scheduler.timer /etc/systemd/system/
sudo systemctl enable librenms-scheduler.timer
sudo systemctl start librenms-scheduler.timer
sudo cp /opt/librenms/misc/librenms.logrotate /etc/logrotate.d/librenms
```

### Step 9 — Web Installer

Open a browser and navigate to `http://192.168.1.217`

Complete the installer steps:
1. **Pre-install checks** — all should pass green
2. **Configure Database** — Host: `localhost`, User: `librenms`, Password: your password, Database: `librenms`
3. **Build Database** — click Build Database and wait for completion
4. **Create Admin User** — set username, password, and email
5. **Finish Install** — select update channel and theme, click Finish Install

**Important:** If the installer cannot write the `.env` file, edit it manually:

```bash
sudo nano /opt/librenms/.env
```

Ensure the DB lines are **not** commented out (remove any leading `#` from DB_ lines) and that `INSTALL=true` is removed. Then:

```bash
sudo chown librenms:librenms /opt/librenms/.env
sudo chmod 660 /opt/librenms/.env
sudo systemctl restart php-fpm nginx
```

---

## Adding Devices

### Add Topton (Linux host with SNMP)

```bash
sudo -u librenms /opt/librenms/lnms device:add 192.168.1.217 --v2c --community public
```

### Add Windows Server (SNMP setup required first)

On the Windows Server (PowerShell as Administrator):

```powershell
Add-WindowsFeature SNMP-Service
Set-Service -Name SNMP -StartupType Automatic
Start-Service SNMP
Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Services\SNMP\Parameters\ValidCommunities" -Name "public" -Value 4 -Type DWord
Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Services\SNMP\Parameters\PermittedManagers" -Name "1" -Value "192.168.1.217"
Restart-Service SNMP
New-NetFirewallRule -DisplayName "SNMP" -Direction Inbound -Protocol UDP -LocalPort 161 -Action Allow
New-NetFirewallRule -DisplayName "Allow ICMPv4" -Direction Inbound -Protocol ICMPv4 -IcmpType 8 -Action Allow
```

Then add from Topton:

```bash
sudo -u librenms /opt/librenms/lnms device:add 192.168.1.10 --v2c --community public
```

### Add Router (ping only, no SNMP)

```bash
sudo -u librenms /opt/librenms/lnms device:add 192.168.1.1 --force
```

---

## Troubleshooting

### Poller fails with `No module named 'pymysql'`

```bash
sudo dnf install python3-PyMySQL -y
```

### Poller fails with `No module named 'dotenv'`

```bash
sudo dnf install python3-dotenv -y
```

### Device not appearing on availability map

Run discovery and polling manually:

```bash
sudo -u librenms /opt/librenms/lnms device:discover <device_id>
sudo -u librenms /opt/librenms/poller-wrapper.py 16
```

Check device IDs with:

```bash
sudo mysql -e "SELECT device_id, hostname, ip, status FROM librenms.devices;"
```

### Windows SNMP not responding

Verify the PermittedManagers registry key includes the LibreNMS host IP, and restart the SNMP service. Test from Topton:

```bash
snmpwalk -v2c -c public 192.168.1.10 sysDescr
```

---

## Result

After successful deployment, the LibreNMS availability map shows all monitored devices with real-time status:

| Device | IP | Status | SNMP |
|--------|-----|--------|------|
| Topton N150 (Linux) | 192.168.1.217 | Up (green) | Full |
| TAROX Windows Server 2025 | 192.168.1.10 | Up (green) | Full |
| EE Smart Hub Router | 192.168.1.1 | Down (ping blocked) | None |

Each device page provides OS identification, uptime, overall traffic graphs, CPU/memory graphs, interface statistics, and event logs.

---

## Related Projects

- [HomeLab Overview](https://github.com/arturskaufmanis/HomeLab-Overview)
- [Hardware Repair and Upgrade Project](https://arturskaufmanis.github.io/Hardware-repair-and-upgrade-project/)

---

*Author: Arturs Kaufmanis — Homelab cybersecurity and IT portfolio*
