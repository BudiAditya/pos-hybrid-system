📁 [FILE: PHASE_6_DEPLOYMENT.md]
📝 [COMMIT: "Add complete Phase 6 Deployment documentation"]
markdown
# Phase 6: Deployment & Production Setup

## Overview
Production deployment untuk seluruh system: VPS configuration, database setup, application deployment, SSL certificates, monitoring, dan maintenance.

## Architecture Diagram

```mermaid
graph TB
    subgraph "Production Environment"
        A[Load Balancer - Nginx]
        B[VPS 1 - API + SignalR]
        C[VPS 2 - API + SignalR]
        D[Database Cluster - PostgreSQL]
        E[Redis Cluster]
        F[RabbitMQ Cluster]
    end
    
    subgraph "External Services"
        G[CDN - Static Assets]
        H[SSL Certificate - Let's Encrypt]
        I[Monitoring - Prometheus + Grafana]
        J[Backup - AWS S3]
    end
    
    subgraph "Client Applications"
        K[Desktop App - Windows]
        L[Mobile App - iOS/Android]
        M[Web Dashboard]
    end
    
    A --> B
    A --> C
    B --> D
    C --> D
    B --> E
    C --> E
    B --> F
    C --> F
    K --> A
    L --> A
    M --> A
Step 1: Production VPS Setup
1.1 VPS Specifications untuk Production
yaml
# Recommended specs untuk 2000+ users:
Primary Server (API + SignalR):
- CPU: 8 vCPU cores
- RAM: 16 GB
- Storage: 200 GB SSD
- OS: CentOS 9 Stream
- Bandwidth: 10 TB/month

Database Server:
- CPU: 4 vCPU cores
- RAM: 32 GB (minimal 16 GB untuk PostgreSQL)
- Storage: 500 GB SSD (with provision for growth)
- OS: CentOS 9 Stream

Redis Server:
- CPU: 4 vCPU cores
- RAM: 8 GB (4 GB allocated to Redis)
- Storage: 100 GB SSD
- OS: CentOS 9 Stream
1.2 Initial Server Hardening
bash
#!/bin/bash
# setup-server.sh - Production server setup script

# Update system
dnf update -y
dnf install -y epel-release

# Install essential packages
dnf install -y \
    curl wget git nano htop \
    firewalld fail2ban \
    chrony ntpstat \
    ufw net-tools \
    telnet tree zip unzip

# Create deployment user
useradd -m -s /bin/bash deployer
usermod -aG wheel deployer
echo "deployer ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers.d/deployer

# SSH hardening
sed -i 's/#Port 22/Port 2222/' /etc/ssh/sshd_config
sed -i 's/#PermitRootLogin yes/PermitRootLogin no/' /etc/ssh/sshd_config
sed -i 's/#PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config
sed -i 's/#PubkeyAuthentication yes/PubkeyAuthentication yes/' /etc/ssh/sshd_config
echo "AllowUsers deployer" >> /etc/ssh/sshd_config

# Firewall configuration
systemctl enable --now firewalld
firewall-cmd --permanent --add-port=2222/tcp    # SSH
firewall-cmd --permanent --add-port=80/tcp      # HTTP
firewall-cmd --permanent --add-port=443/tcp     # HTTPS
firewall-cmd --permanent --add-port=5432/tcp    # PostgreSQL
firewall-cmd --permanent --add-port=6379/tcp    # Redis
firewall-cmd --permanent --add-port=5672/tcp    # RabbitMQ
firewall-cmd --permanent --add-port=9090/tcp    # Prometheus
firewall-cmd --permanent --add-port=3000/tcp    # Grafana
firewall-cmd --reload

# Time synchronization
systemctl enable --now chronyd
chronyc sources

# System limits optimization
echo "* soft nofile 65536" >> /etc/security/limits.conf
echo "* hard nofile 65536" >> /etc/security/limits.conf
echo "* soft nproc 65536" >> /etc/security/limits.conf
echo "* hard nproc 65536" >> /etc/security/limits.conf

# Kernel parameters untuk performance
echo "net.core.somaxconn = 1024" >> /etc/sysctl.conf
echo "net.ipv4.tcp_max_syn_backlog = 2048" >> /etc/sysctl.conf
echo "vm.swappiness = 10" >> /etc/sysctl.conf
sysctl -p

echo "Server setup completed. Please restart SSH service manually."
Step 2: Database Cluster Setup
2.1 PostgreSQL Production Configuration
bash
#!/bin/bash
# setup-postgresql.sh

# Install PostgreSQL 15
dnf install -y postgresql15 postgresql15-server postgresql15-contrib

# Initialize database
postgresql-setup --initdb

# Configure PostgreSQL
cp /var/lib/pgsql/15/data/postgresql.conf /var/lib/pgsql/15/data/postgresql.conf.backup

cat > /var/lib/pgsql/15/data/postgresql.conf << EOF
# Database Configuration
listen_addresses = '*'
port = 5432
max_connections = 500
superuser_reserved_connections = 3

# Memory Configuration (adjust based on RAM)
shared_buffers = 4GB                    # 25% of 16GB RAM
work_mem = 16MB                         # 2% of shared_buffers
maintenance_work_mem = 1GB
effective_cache_size = 12GB             # 75% of RAM

# Checkpoint Configuration
checkpoint_timeout = 15min
checkpoint_completion_target = 0.9
wal_buffers = 16MB

# Performance Configuration
random_page_cost = 1.1
effective_io_concurrency = 200
max_worker_processes = 8
max_parallel_workers_per_gather = 4
max_parallel_workers = 8

# Logging Configuration
log_destination = 'stderr'
logging_collector = on
log_directory = 'pg_log'
log_filename = 'postgresql-%Y-%m-%d_%H%M%S.log'
log_file_mode = 0600
log_rotation_age = 1d
log_rotation_size = 100MB
log_min_duration_statement = 1000       # Log slow queries > 1s

# Replication Configuration (if using replication)
# wal_level = replica
# max_wal_senders = 10
# wal_keep_size = 1GB
EOF

# Configure client authentication
cp /var/lib/pgsql/15/data/pg_hba.conf /var/lib/pgsql/15/data/pg_hba.conf.backup

cat > /var/lib/pgsql/15/data/pg_hba.conf << EOF
# TYPE  DATABASE        USER            ADDRESS                 METHOD
local   all             all                                     peer
host    all             all             127.0.0.1/32            scram-sha-256
host    all             all             ::1/128                 scram-sha-256
host    pos_cloud       pos_user        10.0.0.0/8              scram-sha-256
host    replication     replicator      10.0.0.0/8              scram-sha-256
EOF

# Start and enable PostgreSQL
systemctl enable --now postgresql-15

# Create database and user
sudo -u postgres psql << EOF
CREATE DATABASE pos_cloud;
CREATE USER pos_user WITH PASSWORD 'your_secure_password_here';
GRANT ALL PRIVILEGES ON DATABASE pos_cloud TO pos_user;
ALTER USER pos_user SET search_path = public;

-- Create extensions
\c pos_cloud
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pg_stat_statements";

-- Configure monitoring
ALTER SYSTEM SET shared_preload_libraries = 'pg_stat_statements';
SELECT pg_reload_conf();
EOF

# Configure PgBouncer untuk connection pooling
dnf install -y pgbouncer

cat > /etc/pgbouncer/pgbouncer.ini << EOF
[databases]
pos_cloud = host=localhost port=5432 dbname=pos_cloud

[pgbouncer]
listen_addr = 127.0.0.1
listen_port = 6432
auth_type = scram-sha-256
auth_file = /etc/pgbouncer/userlist.txt
pool_mode = transaction
max_client_conn = 1000
default_pool_size = 25
reserve_pool_size = 5
log_connections = 1
log_disconnections = 1
log_pooler_errors = 1
stats_period = 60
server_reset_query = DISCARD ALL
EOF

# Create auth file
echo "\"pos_user\" \"your_secure_password_here\"" > /etc/pgbouncer/userlist.txt
chown pgbouncer:pgbouncer /etc/pgbouncer/userlist.txt
chmod 600 /etc/pgbouncer/userlist.txt

systemctl enable --now pgbouncer
2.2 Database Backup Strategy
bash
#!/bin/bash
# backup-database.sh

#!/bin/bash
# Database backup script dengan retention policy

BACKUP_DIR="/backup/postgres"
DATE=$(date +%Y%m%d_%H%M%S)
RETENTION_DAYS=7
LOG_FILE="/var/log/postgres-backup.log"

# Create backup directory
mkdir -p $BACKUP_DIR

# Full database backup
echo "$(date): Starting database backup" >> $LOG_FILE

pg_dump -h localhost -U pos_user -d pos_cloud \
  --format=custom \
  --verbose \
  --file=$BACKUP_DIR/pos_cloud_$DATE.dump

if [ $? -eq 0 ]; then
    echo "$(date): Backup completed successfully: pos_cloud_$DATE.dump" >> $LOG_FILE
    
    # Compress backup
    gzip $BACKUP_DIR/pos_cloud_$DATE.dump
    
    # Cleanup old backups
    find $BACKUP_DIR -name "*.gz" -mtime +$RETENTION_DAYS -delete
    
    # Sync to cloud storage (optional)
    # aws s3 sync $BACKUP_DIR s3://your-bucket/postgres-backups/
else
    echo "$(date): Backup failed" >> $LOG_FILE
    exit 1
fi

# Add to crontab untuk daily backup
# 0 2 * * * /usr/local/bin/backup-database.sh
Step 3: Redis Cluster Setup
3.1 Redis Production Configuration
bash
#!/bin/bash
# setup-redis.sh

# Install Redis
dnf install -y redis

# Configure Redis
cp /etc/redis/redis.conf /etc/redis/redis.conf.backup

cat > /etc/redis/redis.conf << EOF
# Basic Configuration
bind 127.0.0.1 ::1
port 6379
protected-mode yes
requirepass your_strong_redis_password_here

# Memory Management
maxmemory 4gb
maxmemory-policy allkeys-lru

# Persistence
save 900 1
save 300 10
save 60 10000
stop-writes-on-bgsave-error yes
rdbcompression yes
rdbchecksum yes
dbfilename dump.rdb
dir /var/lib/redis

# Logging
loglevel notice
logfile /var/log/redis/redis.log

# Performance
timeout 0
tcp-keepalive 300
latency-monitor-threshold 100

# Advanced Configuration
hash-max-ziplist-entries 512
hash-max-ziplist-value 64
list-max-ziplist-size -2
set-max-intset-entries 512
zset-max-ziplist-entries 128
zset-max-ziplist-value 64

# Cluster Configuration (if using cluster)
# cluster-enabled yes
# cluster-config-file nodes.conf
# cluster-node-timeout 15000
EOF

# Create Redis directory and set permissions
mkdir -p /var/lib/redis
chown redis:redis /var/lib/redis
chmod 700 /var/lib/redis

# Enable and start Redis
systemctl enable --now redis
systemctl status redis

# Test Redis connection
redis-cli -a your_strong_redis_password_here ping
Step 4: .NET Application Deployment
4.1 Application Deployment Script
bash
#!/bin/bash
# deploy-app.sh

#!/bin/bash
# .NET Application Deployment Script

APP_NAME="pos-cloud"
DEPLOY_USER="deployer"
DEPLOY_DIR="/var/www/$APP_NAME"
BACKUP_DIR="/backup/apps/$APP_NAME"
DATE=$(date +%Y%m%d_%H%M%S)

# Colors for output
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m'

log() {
    echo -e "${GREEN}[$(date +'%Y-%m-%d %H:%M:%S')]${NC} $1"
}

error() {
    echo -e "${RED}[$(date +'%Y-%m-%d %H:%M:%S')] ERROR:${NC} $1"
}

warn() {
    echo -e "${YELLOW}[$(date +'%Y-%m-%d %H:%M:%S')] WARN:${NC} $1"
}

# Check if running as deployer user
if [ "$(whoami)" != "$DEPLOY_USER" ]; then
    error "This script must be run as $DEPLOY_USER"
    exit 1
fi

# Create backup of current version
log "Creating backup of current version..."
if [ -d "$DEPLOY_DIR" ]; then
    mkdir -p $BACKUP_DIR
    tar -czf $BACKUP_DIR/$APP_NAME-$DATE.tar.gz -C $(dirname $DEPLOY_DIR) $(basename $DEPLOY_DIR)
    log "Backup created: $BACKUP_DIR/$APP_NAME-$DATE.tar.gz"
fi

# Stop application services
log "Stopping application services..."
sudo systemctl stop pos-api.service
sudo systemctl stop pos-signalr.service

# Create deployment directory
log "Creating deployment directory..."
sudo mkdir -p $DEPLOY_DIR
sudo chown $DEPLOY_USER:$DEPLOY_USER $DEPLOY_DIR

# Extract new version (assuming artifact is passed as parameter)
if [ $# -eq 0 ]; then
    error "No deployment artifact specified"
    exit 1
fi

ARTIFACT=$1
log "Extracting artifact: $ARTIFACT"
tar -xzf $ARTIFACT -C $DEPLOY_DIR

# Set permissions
log "Setting permissions..."
sudo chmod +x $DEPLOY_DIR/POSCloud.API
sudo chmod +x $DEPLOY_DIR/POSCloud.SignalR

# Create appsettings.Production.json
log "Creating production configuration..."
cat > $DEPLOY_DIR/appsettings.Production.json << EOF
{
  "ConnectionStrings": {
    "DefaultConnection": "Host=localhost;Port=5432;Database=pos_cloud;Username=pos_user;Password=your_secure_password_here;Pooling=true;Maximum Pool Size=100;"
  },
  "Jwt": {
    "Secret": "your-super-secret-jwt-key-minimum-32-characters-long",
    "Issuer": "pos-cloud.com",
    "Audience": "pos-cloud.com",
    "ExpiryMinutes": 1440
  },
  "Redis": {
    "ConnectionString": "localhost:6379,password=your_strong_redis_password_here,abortConnect=false",
    "InstanceName": "POSCloud"
  },
  "RabbitMQ": {
    "HostName": "localhost",
    "UserName": "posapp",
    "Password": "your_rabbitmq_password",
    "VirtualHost": "/"
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning",
      "Microsoft.EntityFrameworkCore.Database.Command": "Warning"
    },
    "File": {
      "Path": "/var/log/pos-cloud/app.log",
      "FileSizeLimitBytes": 10485760,
      "RetainedFileCountLimit": 5
    }
  },
  "AllowedHosts": "*",
  "Cors": {
    "AllowedOrigins": [
      "https://your-domain.com",
      "capacitor://localhost",
      "ionic://localhost"
    ]
  }
}
EOF

# Create log directory
sudo mkdir -p /var/log/pos-cloud
sudo chown $DEPLOY_USER:$DEPLOY_USER /var/log/pos-cloud

# Database migrations
log "Running database migrations..."
cd $DEPLOY_DIR
export ASPNETCORE_ENVIRONMENT=Production
export DOTNET_ROOT=/usr/lib64/dotnet

./POSCloud.API --migrate

# Start application services
log "Starting application services..."
sudo systemctl daemon-reload
sudo systemctl start pos-api.service
sudo systemctl start pos-signalr.service

# Wait for services to start
sleep 10

# Check service status
if sudo systemctl is-active --quiet pos-api.service; then
    log "POS API service started successfully"
else
    error "POS API service failed to start"
    sudo systemctl status pos-api.service
    exit 1
fi

if sudo systemctl is-active --quiet pos-signalr.service; then
    log "POS SignalR service started successfully"
else
    error "POS SignalR service failed to start"
    sudo systemctl status pos-signalr.service
    exit 1
fi

# Health check
log "Performing health check..."
API_RESPONSE=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:5000/health)
if [ "$API_RESPONSE" = "200" ]; then
    log "Health check passed"
else
    error "Health check failed: HTTP $API_RESPONSE"
    exit 1
fi

log "Deployment completed successfully!"
4.2 Systemd Service Files
ini
; /etc/systemd/system/pos-api.service
[Unit]
Description=POS Cloud API
After=network.target postgresql-15.service redis.service rabbitmq-server.service
Requires=postgresql-15.service redis.service

[Service]
Type=exec
User=deployer
Group=deployer
WorkingDirectory=/var/www/pos-cloud
ExecStart=/var/www/pos-cloud/POSCloud.API
Restart=always
RestartSec=10
KillSignal=SIGINT
Environment=ASPNETCORE_ENVIRONMENT=Production
Environment=DOTNET_ROOT=/usr/lib64/dotnet
Environment=ASPNETCORE_URLS=http://localhost:5000
StandardOutput=journal
StandardError=journal
SyslogIdentifier=pos-api

# Security settings
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=strict
ProtectHome=true
ReadWritePaths=/var/www/pos-cloud /var/log/pos-cloud

[Install]
WantedBy=multi-user.target

; /etc/systemd/system/pos-signalr.service
[Unit]
Description=POS SignalR Hub
After=network.target pos-api.service redis.service
Requires=redis.service

[Service]
Type=exec
User=deployer
Group=deployer
WorkingDirectory=/var/www/pos-cloud
ExecStart=/var/www/pos-cloud/POSCloud.SignalR
Restart=always
RestartSec=10
KillSignal=SIGINT
Environment=ASPNETCORE_ENVIRONMENT=Production
Environment=DOTNET_ROOT=/usr/lib64/dotnet
Environment=ASPNETCORE_URLS=http://localhost:5001
StandardOutput=journal
StandardError=journal
SyslogIdentifier=pos-signalr

# Security settings
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=strict
ProtectHome=true
ReadWritePaths=/var/www/pos-cloud /var/log/pos-cloud

[Install]
WantedBy=multi-user.target
Step 5: Nginx Reverse Proxy dengan SSL
5.1 Nginx Configuration
nginx
# /etc/nginx/nginx.conf
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log notice;
pid /run/nginx.pid;

events {
    worker_connections 1024;
    use epoll;
    multi_accept on;
}

http {
    # Basic Settings
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 2048;
    server_tokens off;

    # MIME Types
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    # Logging Format
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for" '
                    'rt=$request_time uct="$upstream_connect_time" '
                    'uht="$upstream_header_time" urt="$upstream_response_time"';

    access_log /var/log/nginx/access.log main;

    # Gzip Compression
    gzip on;
    gzip_vary on;
    gzip_min_length 1024;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_types
        text/plain
        text/css
        text/xml
        text/javascript
        application/json
        application/javascript
        application/xml+rss
        application/atom+xml
        image/svg+xml;

    # Upstream Servers
    upstream pos_api {
        server 127.0.0.1:5000;
        keepalive 32;
    }

    upstream pos_signalr {
        server 127.0.0.1:5001;
        keepalive 32;
    }

    # Rate Limiting
    limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;
    limit_req_zone $binary_remote_addr zone=signalr:10m rate=100r/s;

    # Include server configurations
    include /etc/nginx/conf.d/*.conf;
}
5.2 Server Block Configuration
nginx
# /etc/nginx/conf.d/pos-cloud.conf
# HTTP to HTTPS redirect
server {
    listen 80;
    listen [::]:80;
    server_name your-domain.com www.your-domain.com;
    
    # Security headers
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    
    # Redirect to HTTPS
    return 301 https://$server_name$request_uri;
}

# HTTPS Server
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name your-domain.com www.your-domain.com;

    # SSL Configuration
    ssl_certificate /etc/letsencrypt/live/your-domain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/your-domain.com/privkey.pem;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers off;

    # Security Headers
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;

    # API Routes
    location /api/ {
        limit_req zone=api burst=20 nodelay;
        
        proxy_pass http://pos_api;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection keep-alive;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
        
        # Timeout settings
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
        
        # Buffer settings
        proxy_buffering on;
        proxy_buffer_size 4k;
        proxy_buffers 8 4k;
        
        # Cache static responses
        location ~* /api/(products|stores) {
            proxy_cache api_cache;
            proxy_cache_key "$scheme$request_method$host$request_uri";
            proxy_cache_valid 200 302 5m;
            proxy_cache_valid 404 1m;
            add_header X-Cache-Status $upstream_cache_status;
        }
    }

    # SignalR WebSocket Routes
    location /poshub/ {
        limit_req zone=signalr burst=50 nodelay;
        
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

    # Health Check Endpoint
    location /health {
        access_log off;
        proxy_pass http://pos_api/health;
        proxy_set_header Host $host;
    }

    # Static Files (for future web dashboard)
    location / {
        root /var/www/pos-static;
        index index.html;
        try_files $uri $uri/ /index.html;
        
        # Cache static assets
        location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff2)$ {
            expires 1y;
            add_header Cache-Control "public, immutable";
        }
    }

    # Deny access to sensitive files
    location ~ /\. {
        deny all;
        access_log off;
        log_not_found off;
    }

    location ~ /\.ht {
        deny all;
    }
}

# Cache zone definition
proxy_cache_path /var/cache/nginx levels=1:2 keys_zone=api_cache:10m max_size=100m 
                 inactive=60m use_temp_path=off;
5.3 SSL Certificate dengan Let's Encrypt
bash
#!/bin/bash
# setup-ssl.sh

#!/bin/bash
# SSL Certificate Setup dengan Certbot

# Install Certbot
dnf install -y certbot python3-certbot-nginx

# Stop Nginx temporarily
systemctl stop nginx

# Obtain SSL certificate
certbot certonly --standalone \
    --non-interactive \
    --agree-tos \
    --email admin@your-domain.com \
    --domains your-domain.com,www.your-domain.com \
    --pre-hook "systemctl stop nginx" \
    --post-hook "systemctl start nginx"

# Setup auto-renewal
echo "0 12 * * * root /usr/bin/certbot renew --quiet --pre-hook 'systemctl stop nginx' --post-hook 'systemctl start nginx'" >> /etc/crontab

# Test renewal
certbot renew --dry-run

# Start Nginx
systemctl start nginx
systemctl enable nginx
Step 6: Monitoring dan Alerting
6.1 Prometheus Configuration
yaml
# /etc/prometheus/prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files:
  - "alert_rules.yml"

scrape_configs:
  # Prometheus itself
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  # .NET Applications
  - job_name: 'pos-api'
    static_configs:
      - targets: ['localhost:5000']
    metrics_path: '/metrics'
    scrape_interval: 10s

  - job_name: 'pos-signalr'
    static_configs:
      - targets: ['localhost:5001']
    metrics_path: '/metrics'
    scrape_interval: 10s

  # PostgreSQL
  - job_name: 'postgresql'
    static_configs:
      - targets: ['localhost:9187']
    scrape_interval: 30s

  # Redis
  - job_name: 'redis'
    static_configs:
      - targets: ['localhost:9121']
    scrape_interval: 30s

  # Node Exporter (system metrics)
  - job_name: 'node'
    static_configs:
      - targets: ['localhost:9100']
    scrape_interval: 30s

  # Nginx
  - job_name: 'nginx'
    static_configs:
      - targets: ['localhost:9113']
    scrape_interval: 30s

alerting:
  alertmanagers:
    - static_configs:
        - targets:
          - localhost:9093
6.2 Grafana Dashboard Setup
bash
#!/bin/bash
# setup-monitoring.sh

#!/bin/bash
# Monitoring Stack Setup (Prometheus + Grafana)

# Install Prometheus
wget https://github.com/prometheus/prometheus/releases/download/v2.40.0/prometheus-2.40.0.linux-amd64.tar.gz
tar xvf prometheus-*.tar.gz
cd prometheus-*/
sudo cp prometheus promtool /usr/local/bin/
sudo cp -r consoles/ console_libraries/ /etc/prometheus/
sudo mkdir -p /var/lib/prometheus
sudo useradd --no-create-home --shell /bin/false prometheus
sudo chown -R prometheus:prometheus /etc/prometheus /var/lib/prometheus

# Create Prometheus service
cat > /etc/systemd/system/prometheus.service << EOF
[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/prometheus \
    --config.file /etc/prometheus/prometheus.yml \
    --storage.tsdb.path /var/lib/prometheus/ \
    --web.console.templates=/etc/prometheus/consoles \
    --web.console.libraries=/etc/prometheus/console_libraries

[Install]
WantedBy=multi-user.target
EOF

# Install Node Exporter
wget https://github.com/prometheus/node_exporter/releases/download/v1.5.0/node_exporter-1.5.0.linux-amd64.tar.gz
tar xvf node_exporter-*.tar.gz
cd node_exporter-*/
sudo cp node_exporter /usr/local/bin/
sudo useradd --no-create-home --shell /bin/false node_exporter

cat > /etc/systemd/system/node_exporter.service << EOF
[Unit]
Description=Node Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=multi-user.target
EOF

# Install Grafana
cat > /etc/yum.repos.d/grafana.repo << EOF
[grafana]
name=grafana
baseurl=https://packages.grafana.com/oss/rpm
repo_gpgcheck=1
enabled=1
gpgcheck=1
gpgkey=https://packages.grafana.com/gpg.key
sslverify=1
sslcacert=/etc/pki/tls/certs/ca-bundle.crt
EOF

dnf install -y grafana
systemctl daemon-reload
systemctl enable --now grafana-server

# Install PostgreSQL exporter
wget https://github.com/prometheus-community/postgres_exporter/releases/download/v0.11.1/postgres_exporter-0.11.1.linux-amd64.tar.gz
tar xvf postgres_exporter-*.tar.gz
cd postgres_exporter-*/
sudo cp postgres_exporter /usr/local/bin/
sudo useradd --no-create-home --shell /bin/false postgres_exporter

cat > /etc/systemd/system/postgres_exporter.service << EOF
[Unit]
Description=PostgreSQL Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=postgres_exporter
Group=postgres_exporter
Type=simple
Environment="DATA_SOURCE_NAME=postgresql://pos_user:your_password@localhost:5432/pos_cloud?sslmode=disable"
ExecStart=/usr/local/bin/postgres_exporter

[Install]
WantedBy=multi-user.target
EOF

# Start all services
systemctl daemon-reload
systemctl enable --now prometheus
systemctl enable --now node_exporter
systemctl enable --now postgres_exporter

# Configure firewall
firewall-cmd --permanent --add-port=9090/tcp  # Prometheus
firewall-cmd --permanent --add-port=3000/tcp  # Grafana
firewall-cmd --permanent --add-port=9100/tcp  # Node Exporter
firewall-cmd --reload

echo "Monitoring stack installed successfully!"
echo "Grafana: http://your-domain.com:3000 (admin/admin)"
echo "Prometheus: http://your-domain.com:9090"
Step 7: Backup dan Disaster Recovery
7.1 Comprehensive Backup Strategy
bash
#!/bin/bash
# comprehensive-backup.sh

#!/bin/bash
# Comprehensive Backup Script untuk seluruh system

BACKUP_DIR="/backup"
DATE=$(date +%Y%m%d)
RETENTION_DAYS=30
LOG_FILE="/var/log/backup.log"

# Colors for output
GREEN='\033[0;32m'
RED='\033[0;31m'
NC='\033[0m'

log() {
    echo -e "${GREEN}[$(date +'%Y-%m-%d %H:%M:%S')]${NC} $1" | tee -a $LOG_FILE
}

error() {
    echo -e "${RED}[$(date +'%Y-%m-%d %H:%M:%S')] ERROR:${NC} $1" | tee -a $LOG_FILE
}

# Create backup directory
mkdir -p $BACKUP_DIR/$DATE

log "Starting comprehensive backup..."

# 1. Database Backup
log "Backing up PostgreSQL database..."
pg_dump -h localhost -U pos_user -d pos_cloud \
    --format=custom \
    --verbose \
    --file=$BACKUP_DIR/$DATE/pos_cloud.dump

if [ $? -eq 0 ]; then
    log "Database backup completed"
else
    error "Database backup failed"
    exit 1
fi

# 2. Application Configuration Backup
log "Backing up application configurations..."
tar -czf $BACKUP_DIR/$DATE/app-configs.tar.gz \
    /etc/nginx/ \
    /etc/systemd/system/pos-*.service \
    /var/www/pos-cloud/appsettings.Production.json

# 3. Redis Backup
log "Backing up Redis data..."
redis-cli -a your_redis_password SAVE
cp /var/lib/redis/dump.rdb $BACKUP_DIR/$DATE/redis-dump.rdb

# 4. SSL Certificates Backup
log "Backing up SSL certificates..."
tar -czf $BACKUP_DIR/$DATE/ssl-certs.tar.gz /etc/letsencrypt/

# 5. Monitoring Data Backup (optional)
log "Backing up monitoring configuration..."
tar -czf $BACKUP_DIR/$DATE/monitoring.tar.gz \
    /etc/prometheus/ \
    /etc/grafana/ \
    /var/lib/grafana/

# 6. Log Files Backup (rotate logs)
log "Rotating and backing up log files..."
tar -czf $BACKUP_DIR/$DATE/logs.tar.gz /var/log/nginx/ /var/log/pos-cloud/

# Compress entire backup
log "Compressing backup..."
tar -czf $BACKUP_DIR/pos-backup-$DATE.tar.gz -C $BACKUP_DIR $DATE

# Upload to cloud storage (AWS S3 example)
log "Uploading to cloud storage..."
aws s3 cp $BACKUP_DIR/pos-backup-$DATE.tar.gz s3://your-backup-bucket/pos-system/

if [ $? -eq 0 ]; then
    log "Cloud upload completed"
else
    error "Cloud upload failed"
fi

# Cleanup local backup
rm -rf $BACKUP_DIR/$DATE

# Cleanup old backups
find $BACKUP_DIR -name "pos-backup-*.tar.gz" -mtime +$RETENTION_DAYS -delete

log "Backup completed successfully!"
7.2 Disaster Recovery Plan
bash
#!/bin/bash
# disaster-recovery.sh

#!/bin/bash
# Disaster Recovery Script

RESTORE_DATE=${1:-$(date +%Y%m%d)}
BACKUP_FILE="pos-backup-$RESTORE_DATE.tar.gz"
S3_BUCKET="s3://your-backup-bucket/pos-system"

log() {
    echo "[$(date +'%Y-%m-%d %H:%M:%S')] $1"
}

# Download backup from S3
log "Downloading backup: $BACKUP_FILE"
aws s3 cp $S3_BUCKET/$BACKUP_FILE /tmp/

if [ $? -ne 0 ]; then
    log "ERROR: Backup file not found"
    exit 1
fi

# Extract backup
log "Extracting backup..."
tar -xzf /tmp/$BACKUP_FILE -C /tmp/

# Stop services
log "Stopping services..."
systemctl stop nginx
systemctl stop pos-api
systemctl stop pos-signalr
systemctl stop redis
systemctl stop postgresql-15

# Restore PostgreSQL
log "Restoring database..."
sudo -u postgres psql -c "DROP DATABASE IF EXISTS pos_cloud;"
sudo -u postgres psql -c "CREATE DATABASE pos_cloud;"
pg_restore -h localhost -U pos_user -d pos_cloud /tmp/$RESTORE_DATE/pos_cloud.dump

# Restore Redis
log "Restoring Redis..."
systemctl stop redis
cp /tmp/$RESTORE_DATE/redis-dump.rdb /var/lib/redis/dump.rdb
chown redis:redis /var/lib/redis/dump.rdb
systemctl start redis

# Restore configurations
log "Restoring configurations..."
tar -xzf /tmp/$RESTORE_DATE/app-configs.tar.gz -C /
tar -xzf /tmp/$RESTORE_DATE/ssl-certs.tar.gz -C /

# Restart services
log "Starting services..."
systemctl start postgresql-15
systemctl start redis
systemctl start pos-api
systemctl start pos-signalr
systemctl start nginx

# Wait for services to stabilize
sleep 30

# Health check
log "Performing health checks..."
if systemctl is-active --quiet nginx && \
   systemctl is-active --quiet pos-api && \
   systemctl is-active --quiet pos-signalr; then
    log "Recovery completed successfully!"
else
    log "ERROR: Some services failed to start"
    exit 1
fi

# Cleanup
rm -rf /tmp/$RESTORE_DATE
rm -f /tmp/$BACKUP_FILE

log "Disaster recovery completed!"
Step 8: Security Hardening
8.1 Security Audit Script
bash
#!/bin/bash
# security-audit.sh

#!/bin/bash
# Security Audit Script untuk Production Server

log() {
    echo "[$(date +'%Y-%m-%d %H:%M:%S')] $1"
}

# 1. System Updates Check
log "Checking for system updates..."
dnf check-update

# 2. Open Ports Audit
log "Checking open ports..."
netstat -tulpn | grep LISTEN

# 3. User Account Audit
log "Checking user accounts..."
awk -F: '($3 >= 1000) {print $1}' /etc/passwd

# 4. SSH Security Check
log "Checking SSH configuration..."
sshd -T | grep -E "(permitrootlogin|passwordauthentication|permitemptypasswords)"

# 5. File Permissions Check
log "Checking critical file permissions..."
ls -l /etc/passwd /etc/shadow /etc/group
ls -l /var/www/pos-cloud/

# 6. Service Security Check
log "Checking service configurations..."
systemctl list-units --type=service --state=running

# 7. Firewall Status
log "Checking firewall status..."
firewall-cmd --list-all

# 8. Log Audit
log "Checking security logs..."
grep -i "failed\|error\|denied" /var/log/secure | tail -20

# 9. Database Security Check
log "Checking PostgreSQL configuration..."
sudo -u postgres psql -c "SELECT usename, client_addr, query_start, query FROM pg_stat_activity;"

# 10. SSL Certificate Check
log "Checking SSL certificates..."
openssl x509 -in /etc/letsencrypt/live/your-domain.com/cert.pem -text -noout | grep -A2 Validity

log "Security audit completed!"
Step 9: Performance Optimization
9.1 Performance Tuning Script
bash
#!/bin/bash
# performance-tuning.sh

#!/bin/bash
# Performance Tuning Script

# Database Performance Tuning
sudo -u postgres psql -d pos_cloud << EOF
-- Create indexes for performance
CREATE INDEX IF NOT EXISTS idx_products_store_active ON products(store_id, is_active);
CREATE INDEX IF NOT EXISTS idx_transactions_store_date ON transactions(store_id, transaction_date);
CREATE INDEX IF NOT EXISTS idx_transaction_items_product ON transaction_items(product_id);

-- Update statistics
ANALYZE;

-- Check query performance
SELECT schemaname, tablename, attname, correlation 
FROM pg_stats 
WHERE tablename IN ('products', 'transactions', 'transaction_items')
ORDER BY correlation DESC;
EOF

# Redis Performance Optimization
redis-cli -a your_redis_password << EOF
CONFIG SET maxmemory-policy allkeys-lru
CONFIG SET save "900 1 300 10 60 10000"
INFO memory
EOF

# System Performance Tuning
echo "vm.swappiness=10" >> /etc/sysctl.conf
echo "net.core.somaxconn=65536" >> /etc/sysctl.conf
sysctl -p

log "Performance tuning completed!"
Step 10: Continuous Deployment Pipeline
10.1 GitHub Actions Workflow
yaml
# .github/workflows/deploy.yml
name: Deploy to Production

on:
  push:
    branches: [ main ]
  release:
    types: [ published ]

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Setup .NET
      uses: actions/setup-dotnet@v3
      with:
        dotnet-version: '8.0.x'
        
    - name: Build API
      run: |
        cd src/POSCloud.API
        dotnet publish -c Release -o ./publish
        
    - name: Build SignalR
      run: |
        cd src/POSCloud.SignalR
        dotnet publish -c Release -o ./publish
        
    - name: Create deployment package
      run: |
        mkdir deploy
        cp -r src/POSCloud.API/publish/* deploy/
        cp -r src/POSCloud.SignalR/publish/* deploy/
        tar -czf pos-cloud-${{ github.sha }}.tar.gz deploy/
        
    - name: Deploy to production
      uses: appleboy/ssh-action@v0.1.7
      with:
        host: ${{ secrets.PRODUCTION_HOST }}
        username: ${{ secrets.PRODUCTION_USER }}
        key: ${{ secrets.PRODUCTION_SSH_KEY }}
        script: |
          cd /tmp
          wget https://github.com/${{ github.repository }}/releases/download/${{ github.ref_name }}/pos-cloud-${{ github.sha }}.tar.gz
          sudo -u deployer /usr/local/bin/deploy-app.sh pos-cloud-${{ github.sha }}.tar.gz
          
    - name: Run health checks
      uses: appleboy/ssh-action@v0.1.7
      with:
        host: ${{ secrets.PRODUCTION_HOST }}
        username: ${{ secrets.PRODUCTION_USER }}
        key: ${{ secrets.PRODUCTION_SSH_KEY }}
        script: |
          curl -f https://your-domain.com/health || exit 1
          curl -f https://your-domain.com/api/products || exit 1
Final Checklist Sebelum Go-Live
Pre-Production Checklist
Semua services running dengan baik

SSL certificates valid dan terkonfigurasi

Database backups working

Monitoring alerts configured

Load testing completed

Security audit passed

Disaster recovery tested

Documentation updated

Post-Deployment Monitoring
Real-time metrics monitoring

Error rate monitoring

Performance metrics tracking

User activity monitoring

Resource utilization monitoring

Production environment sekarang ready untuk go-live! 🚀

Semua components telah di-deploy dan dikonfigurasi untuk handling 2000+ users dengan performance optimal.
