📁 [FILE: PHASE_1_VPS_SETUP.md]
📝 [COMMIT: "Add complete Phase 1 VPS setup documentation"]
markdown
# Phase 1: VPS Setup & Infrastructure

## Prerequisites

### VPS Specifications (Recommended untuk 2000+ Users)
```yaml
Minimum Requirements:
- OS: CentOS 9 Stream
- CPU: 4 vCPU cores (8 recommended)
- RAM: 8 GB (16 GB recommended) 
- Storage: 50 GB SSD (100 GB recommended)
- Bandwidth: 5 TB/month (10 TB recommended)

Network Requirements:
- Open ports: 22 (SSH), 80 (HTTP), 443 (HTTPS), 5432 (PostgreSQL)
- Static IP address
- Domain name (optional tapi recommended)
Software Requirements
SSH client (PuTTY atau terminal)

Root access ke VPS

Basic Linux command line knowledge

Step 1: Initial Server Setup
1.1 Login dan Update System
bash
# Login sebagai root via SSH
ssh root@your-server-ip

# Update system packages
dnf update -y
dnf install -y epel-release
dnf groupinstall -y "Development Tools"

# Install essential packages
dnf install -y \
    curl wget git nano htop \
    firewalld fail2ban \
    chrony ntpstat \
    ufw net-tools
1.2 Create Non-Root User untuk Security
bash
# Create new user
adduser posadmin
usermod -aG wheel posadmin

# Set password
passwd posadmin

# Configure sudo access
echo "posadmin ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers.d/posadmin

# Switch to new user
su - posadmin
1.3 SSH Hardening
bash
# Backup original config
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.backup

# Edit SSH config
sudo nano /etc/ssh/sshd_config

# Apply these changes:
# Port 2222                          # Change default port
# PermitRootLogin no                # Disable root login
# PasswordAuthentication no         # Use key-based auth only
# PubkeyAuthentication yes          # Enable public key auth
# AllowUsers posadmin               # Only allow specific user

# Restart SSH service
sudo systemctl restart sshd

# TEST dari new terminal sebelum menutup current session!
ssh -p 2222 posadmin@your-server-ip
1.4 Firewall Configuration
bash
# Start and enable firewall
sudo systemctl enable --now firewalld

# Open necessary ports
sudo firewall-cmd --permanent --add-port=2222/tcp    # SSH
sudo firewall-cmd --permanent --add-port=80/tcp      # HTTP
sudo firewall-cmd --permanent --add-port=443/tcp     # HTTPS
sudo firewall-cmd --permanent --add-port=5432/tcp    # PostgreSQL

# Additional ports untuk development
sudo firewall-cmd --permanent --add-port=5000-5001/tcp # .NET API
sudo firewall-cmd --permanent --add-port=8080/tcp      # Additional services

# Reload firewall
sudo firewall-cmd --reload

# Check status
sudo firewall-cmd --list-all
Step 2: PostgreSQL Installation & Configuration
2.1 Install PostgreSQL 15
bash
# Install PostgreSQL
sudo dnf install -y postgresql15 postgresql15-server postgresql15-contrib

# Initialize database
sudo postgresql-setup --initdb

# Start and enable service
sudo systemctl enable --now postgresql-15

# Check status
sudo systemctl status postgresql-15
2.2 PostgreSQL Security Configuration
bash
# Edit PostgreSQL config
sudo nano /var/lib/pgsql/15/data/postgresql.conf

# Important settings:
# listen_addresses = 'localhost,192.168.1.100'  # Your server IP
# port = 5432
# max_connections = 500                         # Increase for multiple stores
# shared_buffers = 2GB                          # Adjust based on RAM
# work_mem = 16MB

# Edit client authentication
sudo nano /var/lib/pgsql/15/data/pg_hba.conf

# Add secure authentication:
# host    pos_cloud     pos_user     192.168.1.0/24      scram-sha-256
# host    all           all          127.0.0.1/32        scram-sha-256

# Restart PostgreSQL
sudo systemctl restart postgresql-15
2.3 Create Database dan User
bash
# Switch to postgres user
sudo -i -u postgres

# Create database and user
createuser --pwprompt pos_user
createdb -O pos_user pos_cloud

# Exit postgres user
exit

# Test connection
psql -h localhost -U pos_user -d pos_cloud -W
2.4 Performance Tuning untuk 2000+ Users
bash
# Edit postgresql.conf untuk optimization
sudo nano /var/lib/pgsql/15/data/postgresql.conf

# Performance settings:
# max_connections = 500
# shared_buffers = 4GB                    # 25% of total RAM
# effective_cache_size = 12GB             # 75% of total RAM
# work_mem = 32MB                         # 2% of shared_buffers
# maintenance_work_mem = 1GB
# random_page_cost = 1.1
# checkpoint_completion_target = 0.9
Step 3: .NET 8 Installation
3.1 Install .NET 8 SDK dan Runtime
bash
# Add Microsoft package repository
sudo rpm -Uvh https://packages.microsoft.com/config/centos/9/packages-microsoft-prod.rpm

# Install .NET SDK dan Runtime
sudo dnf install -y dotnet-sdk-8.0 aspnetcore-runtime-8.0

# Verify installation
dotnet --version
dotnet --list-sdks
dotnet --list-runtimes
3.2 Create Systemd Service untuk .NET Apps
bash
# Create application directory
sudo mkdir -p /var/www/pos-api
sudo mkdir -p /var/www/pos-signalr

# Set ownership
sudo chown -R posadmin:posadmin /var/www/pos-*

# Create service file untuk API
sudo nano /etc/systemd/system/pos-api.service
Isi pos-api.service:

ini
[Unit]
Description=POS Cloud API
After=network.target postgresql-15.service

[Service]
Type=exec
User=posadmin
Group=posadmin
WorkingDirectory=/var/www/pos-api
ExecStart=/usr/bin/dotnet POSCloud.API.dll
Restart=always
RestartSec=10
KillSignal=SIGINT
Environment=ASPNETCORE_ENVIRONMENT=Production
Environment=ASPNETCORE_URLS=http://localhost:5000

[Install]
WantedBy=multi-user.target
Isi pos-signalr.service:

ini
[Unit]
Description=POS SignalR Hub
After=network.target pos-api.service redis.service

[Service]
Type=exec
User=posadmin
Group=posadmin
WorkingDirectory=/var/www/pos-signalr
ExecStart=/usr/bin/dotnet POSCloud.SignalR.dll
Restart=always
RestartSec=10
KillSignal=SIGINT
Environment=ASPNETCORE_ENVIRONMENT=Production
Environment=ASPNETCORE_URLS=http://localhost:5001

[Install]
WantedBy=multi-user.target
Step 4: Redis Installation untuk SignalR Scale-out
4.1 Install dan Configure Redis
bash
# Install Redis
sudo dnf install -y redis

# Configure Redis
sudo nano /etc/redis/redis.conf

# Important settings:
# bind 127.0.0.1 ::1                    # Only local connections
# requirepass your-strong-password-here # Set strong password
# maxmemory 2gb                         # Adjust based on RAM
# maxmemory-policy allkeys-lru

# Start and enable Redis
sudo systemctl enable --now redis
sudo systemctl status redis

# Test Redis connection
redis-cli
127.0.0.1:6379> ping
Step 5: RabbitMQ Installation untuk Message Queue
5.1 Install Erlang dan RabbitMQ
bash
# Install Erlang
sudo dnf install -y erlang

# Install RabbitMQ
sudo dnf install -y rabbitmq-server

# Start and enable RabbitMQ
sudo systemctl enable --now rabbitmq-server
sudo systemctl status rabbitmq-server
5.2 RabbitMQ Configuration
bash
# Enable management plugin
sudo rabbitmq-plugins enable rabbitmq_management

# Create admin user
sudo rabbitmqctl add_user admin your-strong-password
sudo rabbitmqctl set_user_tags admin administrator
sudo rabbitmqctl set_permissions -p / admin ".*" ".*" ".*"

# Create application user
sudo rabbitmqctl add_user posapp posapp-password
sudo rabbitmqctl set_permissions -p / posapp ".*" ".*" ".*"

# Restart RabbitMQ
sudo systemctl restart rabbitmq-server
Step 6: Nginx Reverse Proxy Setup
6.1 Install dan Configure Nginx
bash
# Install Nginx
sudo dnf install -y nginx

# Create Nginx config
sudo nano /etc/nginx/conf.d/pos-cloud.conf
Isi pos-cloud.conf:

nginx
upstream pos_api {
    server localhost:5000;
}

upstream pos_signalr {
    server localhost:5001;
}

server {
    listen 80;
    server_name your-domain.com;  # Ganti dengan domain Anda
    
    # API routes
    location /api/ {
        proxy_pass http://pos_api;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection keep-alive;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
        
        # Timeout settings untuk long-running requests
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }
    
    # SignalR WebSocket routes
    location /poshub/ {
        proxy_pass http://pos_signalr;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
        
        # WebSocket specific timeouts
        proxy_connect_timeout 7d;
        proxy_send_timeout 7d;
        proxy_read_timeout 7d;
    }
    
    # Static files (future use)
    location / {
        root /var/www/pos-static;
        index index.html;
        try_files $uri $uri/ /index.html;
    }
}
6.2 Start Nginx
bash
# Test Nginx configuration
sudo nginx -t

# Start and enable Nginx
sudo systemctl enable --now nginx
sudo systemctl status nginx

# Check if ports are listening
sudo netstat -tulpn | grep :80
Step 7: SSL Certificate dengan Let's Encrypt
7.1 Install Certbot
bash
# Install Certbot
sudo dnf install -y certbot python3-certbot-nginx

# Get SSL certificate
sudo certbot --nginx -d your-domain.com

# Test auto-renewal
sudo certbot renew --dry-run
Step 8: Monitoring dan Maintenance
8.1 Setup Basic Monitoring
bash
# Install monitoring tools
sudo dnf install -y htop iotop nethogs

# Create log rotation
sudo nano /etc/logrotate.d/pos-apps

# Add log rotation config
/var/www/pos-api/logs/*.log /var/www/pos-signalr/logs/*.log {
    daily
    missingok
    rotate 14
    compress
    delaycompress
    notifempty
    create 644 posadmin posadmin
}
8.2 Create Backup Script
bash
# Create backup directory
sudo mkdir -p /backup/postgres
sudo chown posadmin:posadmin /backup/postgres

# Create backup script
sudo nano /usr/local/bin/backup-pos.sh
Isi backup-pos.sh:

bash
#!/bin/bash
# POS System Backup Script

DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="/backup/postgres"
LOG_FILE="/var/log/pos-backup.log"

# Backup PostgreSQL
pg_dump -h localhost -U pos_user -d pos_cloud > $BACKUP_DIR/pos_cloud_$DATE.sql

# Compress backup
gzip $BACKUP_DIR/pos_cloud_$DATE.sql

# Cleanup old backups (keep 7 days)
find $BACKUP_DIR -name "*.gz" -mtime +7 -delete

# Log backup operation
echo "$(date): Backup completed - pos_cloud_$DATE.sql.gz" >> $LOG_FILE
bash
# Make script executable
sudo chmod +x /usr/local/bin/backup-pos.sh

# Add to crontab untuk daily backup
sudo crontab -e
# Add line: 0 2 * * * /usr/local/bin/backup-pos.sh
Step 9: Final Testing
9.1 Test All Services
bash
# Test PostgreSQL
psql -h localhost -U pos_user -d pos_cloud -c "SELECT version();"

# Test Redis
redis-cli -a your-password ping

# Test RabbitMQ
sudo rabbitmqctl list_users

# Test .NET installation
dotnet --info

# Test Nginx
curl -I http://localhost
9.2 Security Checklist
SSH port changed dari 22

Root login disabled

Firewall active dengan minimal ports

All services menggunakan non-root users

Strong passwords untuk semua services

SSL certificate installed

Backup system working

Troubleshooting Common Issues
Port Already in Use
bash
# Find process using port
sudo netstat -tulpn | grep :5000

# Kill process jika necessary
sudo kill -9 <PID>
Service Failed to Start
bash
# Check service status
sudo systemctl status pos-api

# Check logs
sudo journalctl -u pos-api -f
Database Connection Issues
bash
# Test connection locally
psql -h localhost -U pos_user -d pos_cloud

# Check PostgreSQL logs
sudo tail -f /var/lib/pgsql/15/data/log/postgresql-*.log
Next Steps
Setelah Phase 1 complete, lanjut ke:

Phase 2: Backend API Development

Phase 3: SignalR Real-time Implementation

Phase 4: Desktop Application

Phase 5: Mobile Application

Server sekarang ready untuk application deployment! 🚀


