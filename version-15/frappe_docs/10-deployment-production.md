# Deployment & Production - Complete Guide

> **Comprehensive reference for production deployment, performance optimization, monitoring, and scaling strategies**

## Table of Contents

- [Deployment Architecture Overview](#deployment-architecture-overview)
- [Production Setup](#production-setup)
- [Performance Optimization](#performance-optimization)
- [Monitoring and Logging](#monitoring-and-logging)
- [Scaling Strategies](#scaling-strategies)
- [Backup and Recovery](#backup-and-recovery)
- [CI/CD Pipelines](#cicd-pipelines)
- [Security Hardening](#security-hardening)
- [Load Testing](#load-testing)
- [Troubleshooting](#troubleshooting)
- [Maintenance Procedures](#maintenance-procedures)
- [Cost Optimization](#cost-optimization)

## Deployment Architecture Overview

### Multi-Tier Architecture

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Load Balancer │    │   CDN/Cache     │    │   Monitoring    │
│   (HAProxy/AWS) │    │   (CloudFlare)  │    │   (DataDog)     │
└─────────────────┘    └─────────────────┘    └─────────────────┘
         │                       │                       │
         ▼                       ▼                       ▼
┌─────────────────────────────────────────────────────────────────┐
│                        Web Tier                                │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐           │
│  │   Nginx 1   │  │   Nginx 2   │  │   Nginx 3   │           │
│  │(Static Files│  │(Static Files│  │(Static Files│           │
│  │ & Reverse   │  │ & Reverse   │  │ & Reverse   │           │
│  │   Proxy)    │  │   Proxy)    │  │   Proxy)    │           │
│  └─────────────┘  └─────────────┘  └─────────────┘           │
└─────────────────────────────────────────────────────────────────┘
         │                       │                       │
         ▼                       ▼                       ▼
┌─────────────────────────────────────────────────────────────────┐
│                     Application Tier                           │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐           │
│  │ Gunicorn 1  │  │ Gunicorn 2  │  │ Gunicorn 3  │           │
│  │(Web Workers)│  │(Web Workers)│  │(Web Workers)│           │
│  └─────────────┘  └─────────────┘  └─────────────┘           │
│                                                               │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐           │
│  │   RQ 1      │  │   RQ 2      │  │   RQ 3      │           │
│  │(Background  │  │(Background  │  │(Background  │           │
│  │   Jobs)     │  │   Jobs)     │  │   Jobs)     │           │
│  └─────────────┘  └─────────────┘  └─────────────┘           │
│                                                               │
│  ┌─────────────┐                  ┌─────────────┐           │
│  │ Scheduler   │                  │  Socketio   │           │
│  │(Cron Jobs)  │                  │(Real-time)  │           │
│  └─────────────┘                  └─────────────┘           │
└─────────────────────────────────────────────────────────────────┘
         │                       │                       │
         ▼                       ▼                       ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Data Tier                                 │
│  ┌─────────────────┐           ┌─────────────────┐            │
│  │  MariaDB Master │◄─────────►│  MariaDB Slave  │            │
│  │   (Read/Write)  │           │   (Read Only)   │            │
│  └─────────────────┘           └─────────────────┘            │
│                                                               │
│  ┌─────────────────┐           ┌─────────────────┐            │
│  │  Redis Master   │◄─────────►│   Redis Slave   │            │
│  │   (Cache/Queue) │           │    (Backup)     │            │
│  └─────────────────┘           └─────────────────┘            │
│                                                               │
│  ┌─────────────────┐           ┌─────────────────┐            │
│  │  File Storage   │           │  Backup Storage │            │
│  │   (NFS/S3)      │           │   (S3/Glacier)  │            │
│  └─────────────────┘           └─────────────────┘            │
└─────────────────────────────────────────────────────────────────┘
```

### Deployment Patterns

```python
# deployment/architecture.py

class DeploymentPatterns:
    """Standard deployment patterns for Frappe applications"""
    
    @staticmethod
    def single_server_pattern():
        """Single server deployment for small applications"""
        return {
            "description": "All components on one server",
            "use_case": "Development, small production, demos",
            "components": {
                "nginx": "Reverse proxy and static files",
                "gunicorn": "4-8 workers",
                "mariadb": "Single instance",
                "redis": "Single instance",
                "supervisor": "Process management"
            },
            "limitations": [
                "Single point of failure",
                "Limited scalability",
                "Resource contention"
            ],
            "server_specs": {
                "cpu": "4+ cores",
                "memory": "8+ GB RAM", 
                "storage": "SSD 100+ GB",
                "bandwidth": "1 Gbps"
            }
        }
    
    @staticmethod
    def multi_server_pattern():
        """Multi-server deployment for scalability"""
        return {
            "description": "Separated tiers for better performance",
            "use_case": "Medium to large production deployments",
            "architecture": {
                "web_tier": {
                    "servers": 2,
                    "components": ["nginx", "gunicorn"],
                    "purpose": "Handle web requests and static files"
                },
                "app_tier": {
                    "servers": 2,
                    "components": ["gunicorn", "rq_workers", "scheduler"],
                    "purpose": "Application logic and background processing"
                },
                "data_tier": {
                    "servers": 2,
                    "components": ["mariadb_master", "mariadb_slave", "redis"],
                    "purpose": "Data storage and caching"
                }
            },
            "benefits": [
                "Horizontal scaling",
                "Fault tolerance",
                "Performance isolation",
                "Resource optimization"
            ]
        }
    
    @staticmethod
    def microservices_pattern():
        """Microservices deployment with containers"""
        return {
            "description": "Containerized services with orchestration",
            "use_case": "Large-scale, cloud-native deployments",
            "technology_stack": {
                "containers": "Docker",
                "orchestration": "Kubernetes",
                "service_mesh": "Istio",
                "api_gateway": "Kong/Ambassador"
            },
            "services": [
                "frappe-web-service",
                "frappe-app-service", 
                "frappe-worker-service",
                "frappe-scheduler-service",
                "frappe-socketio-service"
            ],
            "advantages": [
                "Independent scaling",
                "Technology flexibility",
                "Deployment isolation",
                "Cloud portability"
            ]
        }

def choose_deployment_pattern(requirements):
    """Choose deployment pattern based on requirements"""
    
    patterns = DeploymentPatterns()
    
    if requirements.get("users") < 100 and requirements.get("budget") == "low":
        return patterns.single_server_pattern()
    elif requirements.get("users") < 1000:
        return patterns.multi_server_pattern()
    else:
        return patterns.microservices_pattern()
```

## Production Setup

### Server Configuration

```bash
#!/bin/bash
# production_setup.sh

# System Update and Essential Packages
setup_system() {
    echo "Setting up production system..."
    
    # Update system packages
    apt-get update && apt-get upgrade -y
    
    # Install essential packages
    apt-get install -y \
        curl \
        wget \
        git \
        vim \
        htop \
        unzip \
        software-properties-common \
        apt-transport-https \
        ca-certificates \
        gnupg \
        lsb-release
    
    # Install Python 3.8+ and pip
    apt-get install -y \
        python3.8 \
        python3.8-dev \
        python3-pip \
        python3-setuptools \
        python3-venv
    
    # Install Node.js 16+
    curl -fsSL https://deb.nodesource.com/setup_16.x | bash -
    apt-get install -y nodejs
    
    # Install yarn
    npm install -g yarn
    
    # Install Redis
    apt-get install -y redis-server
    systemctl enable redis-server
    systemctl start redis-server
    
    # Configure Redis for production
    configure_redis_production
    
    echo "System setup completed!"
}

# MariaDB Installation and Configuration
setup_mariadb() {
    echo "Setting up MariaDB..."
    
    # Install MariaDB
    apt-get install -y \
        mariadb-server \
        mariadb-client \
        libmysqlclient-dev
    
    # Secure MariaDB installation
    mysql_secure_installation
    
    # Production MariaDB configuration
    cat > /etc/mysql/conf.d/frappe.cnf << EOF
[mysqld]
# Basic Settings
bind-address = 0.0.0.0
port = 3306
datadir = /var/lib/mysql
socket = /var/run/mysqld/mysqld.sock

# Performance Settings
innodb_buffer_pool_size = 4G
innodb_log_file_size = 256M
innodb_flush_method = O_DIRECT
innodb_file_per_table = ON

# Connection Settings
max_connections = 200
max_connect_errors = 10000
wait_timeout = 300
interactive_timeout = 300

# Query Cache (disable in MySQL 5.7+)
query_cache_size = 0
query_cache_type = 0

# Binary Logging
log-bin = mysql-bin
binlog_format = ROW
expire_logs_days = 7

# Slow Query Log
slow_query_log = 1
slow_query_log_file = /var/log/mysql/mysql-slow.log
long_query_time = 2

# Character Set
character-set-server = utf8mb4
collation-server = utf8mb4_unicode_ci

# Security
skip-name-resolve
sql_mode = STRICT_TRANS_TABLES,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION
EOF
    
    systemctl restart mysql
    systemctl enable mysql
    
    echo "MariaDB setup completed!"
}

# Nginx Installation and Configuration
setup_nginx() {
    echo "Setting up Nginx..."
    
    apt-get install -y nginx
    
    # Create production Nginx configuration
    cat > /etc/nginx/sites-available/frappe << 'EOF'
upstream frappe-server {
    server 127.0.0.1:8000 fail_timeout=0;
    server 127.0.0.1:8001 fail_timeout=0;
    server 127.0.0.1:8002 fail_timeout=0;
}

upstream socketio-server {
    server 127.0.0.1:9000 fail_timeout=0;
}

# Rate limiting
limit_req_zone $binary_remote_addr zone=login:10m rate=5r/m;

server {
    listen 80;
    server_name your-domain.com;
    
    # Redirect to HTTPS
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name your-domain.com;
    
    # SSL Configuration
    ssl_certificate /path/to/ssl/cert.pem;
    ssl_certificate_key /path/to/ssl/key.pem;
    ssl_session_timeout 1d;
    ssl_session_cache shared:MozTLS:10m;
    ssl_session_tickets off;
    
    # Modern SSL configuration
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers off;
    
    # HSTS
    add_header Strict-Transport-Security "max-age=63072000" always;
    
    # Security Headers
    add_header X-Frame-Options DENY;
    add_header X-Content-Type-Options nosniff;
    add_header X-XSS-Protection "1; mode=block";
    add_header Referrer-Policy "strict-origin-when-cross-origin";
    
    root /home/frappe/frappe-bench/sites;
    
    # File upload size
    client_max_body_size 100m;
    
    # Gzip Configuration
    gzip on;
    gzip_vary on;
    gzip_min_length 1024;
    gzip_types
        text/plain
        text/css
        text/xml
        text/javascript
        application/javascript
        application/xml+rss
        application/json;
    
    # Static files
    location /assets {
        alias /home/frappe/frappe-bench/sites/assets;
        expires 1y;
        add_header Cache-Control "public, immutable";
    }
    
    location /files {
        alias /home/frappe/frappe-bench/sites;
        expires 1y;
        add_header Cache-Control "public";
    }
    
    # Socket.io
    location /socket.io {
        proxy_pass http://socketio-server;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
    
    # Rate limiting for login
    location /api/method/login {
        limit_req zone=login burst=5 nodelay;
        proxy_pass http://frappe-server;
        include proxy_params;
    }
    
    # API endpoints
    location /api {
        proxy_pass http://frappe-server;
        include proxy_params;
    }
    
    # Main application
    location / {
        try_files $uri @frappe;
    }
    
    location @frappe {
        proxy_pass http://frappe-server;
        include proxy_params;
    }
}
EOF
    
    # Create proxy_params
    cat > /etc/nginx/proxy_params << 'EOF'
proxy_set_header Host $http_host;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;
proxy_connect_timeout 300s;
proxy_send_timeout 300s;
proxy_read_timeout 300s;
proxy_buffer_size 128k;
proxy_buffers 4 256k;
proxy_busy_buffers_size 256k;
EOF
    
    # Enable site and restart nginx
    ln -sf /etc/nginx/sites-available/frappe /etc/nginx/sites-enabled/
    rm -f /etc/nginx/sites-enabled/default
    nginx -t && systemctl restart nginx
    systemctl enable nginx
    
    echo "Nginx setup completed!"
}

# Configure Redis for production
configure_redis_production() {
    cat > /etc/redis/redis.conf << 'EOF'
# Network
bind 127.0.0.1
port 6379
timeout 300
tcp-keepalive 60

# Memory
maxmemory 2gb
maxmemory-policy allkeys-lru

# Persistence
save 900 1
save 300 10  
save 60 10000
rdbcompression yes
rdbchecksum yes
dbfilename dump.rdb
dir /var/lib/redis

# Logging
loglevel notice
logfile /var/log/redis/redis-server.log

# Security
requirepass your_secure_redis_password

# Performance
tcp-backlog 511
databases 16
EOF
    
    systemctl restart redis-server
}

# Supervisor Configuration
setup_supervisor() {
    echo "Setting up Supervisor..."
    
    apt-get install -y supervisor
    
    # Frappe web workers
    cat > /etc/supervisor/conf.d/frappe-web.conf << EOF
[group:frappe-web]
programs=frappe-web-1,frappe-web-2,frappe-web-3

[program:frappe-web-1]
command=/home/frappe/frappe-bench/env/bin/gunicorn -b 127.0.0.1:8000 -w 4 -t 120 frappe.app:application --preload
directory=/home/frappe/frappe-bench
user=frappe
autostart=true
autorestart=true
redirect_stderr=true
stdout_logfile=/home/frappe/frappe-bench/logs/web-1.log
stdout_logfile_maxbytes=50MB
stdout_logfile_backups=10

[program:frappe-web-2]
command=/home/frappe/frappe-bench/env/bin/gunicorn -b 127.0.0.1:8001 -w 4 -t 120 frappe.app:application --preload
directory=/home/frappe/frappe-bench
user=frappe
autostart=true
autorestart=true
redirect_stderr=true
stdout_logfile=/home/frappe/frappe-bench/logs/web-2.log

[program:frappe-web-3]
command=/home/frappe/frappe-bench/env/bin/gunicorn -b 127.0.0.1:8002 -w 4 -t 120 frappe.app:application --preload
directory=/home/frappe/frappe-bench
user=frappe
autostart=true
autorestart=true
redirect_stderr=true
stdout_logfile=/home/frappe/frappe-bench/logs/web-3.log
EOF

    # Background workers
    cat > /etc/supervisor/conf.d/frappe-workers.conf << EOF
[group:frappe-workers]
programs=frappe-worker-short,frappe-worker-long,frappe-worker-default

[program:frappe-worker-short]
command=/home/frappe/frappe-bench/env/bin/python -m frappe.utils.bench worker --queue short
directory=/home/frappe/frappe-bench
user=frappe
autostart=true
autorestart=true
redirect_stderr=true
stdout_logfile=/home/frappe/frappe-bench/logs/worker-short.log
stdout_logfile_maxbytes=50MB
stdout_logfile_backups=10
numprocs=2

[program:frappe-worker-long]
command=/home/frappe/frappe-bench/env/bin/python -m frappe.utils.bench worker --queue long
directory=/home/frappe/frappe-bench
user=frappe
autostart=true
autorestart=true
redirect_stderr=true
stdout_logfile=/home/frappe/frappe-bench/logs/worker-long.log
numprocs=1

[program:frappe-worker-default]
command=/home/frappe/frappe-bench/env/bin/python -m frappe.utils.bench worker --queue default
directory=/home/frappe/frappe-bench
user=frappe
autostart=true
autorestart=true
redirect_stderr=true
stdout_logfile=/home/frappe/frappe-bench/logs/worker-default.log
numprocs=3
EOF

    # Scheduler
    cat > /etc/supervisor/conf.d/frappe-scheduler.conf << EOF
[program:frappe-scheduler]
command=/home/frappe/frappe-bench/env/bin/python -m frappe.utils.bench schedule
directory=/home/frappe/frappe-bench
user=frappe
autostart=true
autorestart=true
redirect_stderr=true
stdout_logfile=/home/frappe/frappe-bench/logs/scheduler.log
stdout_logfile_maxbytes=50MB
stdout_logfile_backups=10
EOF

    # Socket.io
    cat > /etc/supervisor/conf.d/frappe-socketio.conf << EOF
[program:frappe-socketio]
command=/home/frappe/frappe-bench/env/bin/python -m frappe.utils.bench node-socketio
directory=/home/frappe/frappe-bench
user=frappe
autostart=true
autorestart=true
redirect_stderr=true
stdout_logfile=/home/frappe/frappe-bench/logs/socketio.log
stdout_logfile_maxbytes=50MB
stdout_logfile_backups=10
EOF
    
    systemctl enable supervisor
    supervisorctl reread
    supervisorctl update
    supervisorctl start all
    
    echo "Supervisor setup completed!"
}

# SSL Certificate Setup with Let's Encrypt
setup_ssl() {
    echo "Setting up SSL certificates..."
    
    # Install Certbot
    apt-get install -y certbot python3-certbot-nginx
    
    # Get SSL certificate
    certbot --nginx -d your-domain.com --non-interactive --agree-tos --email admin@your-domain.com
    
    # Setup auto-renewal
    crontab -l | { cat; echo "0 12 * * * /usr/bin/certbot renew --quiet"; } | crontab -
    
    echo "SSL setup completed!"
}

# Main setup function
main() {
    setup_system
    setup_mariadb
    setup_nginx
    configure_redis_production
    setup_supervisor
    setup_ssl
    
    echo "Production setup completed successfully!"
    echo "Remember to:"
    echo "1. Configure firewall (UFW)"
    echo "2. Set up monitoring"
    echo "3. Configure backups"
    echo "4. Update DNS records"
    echo "5. Test the deployment"
}

# Run main function
main "$@"
```

### System Hardening

```bash
#!/bin/bash
# security_hardening.sh

# System Security Hardening
harden_system() {
    echo "Hardening system security..."
    
    # Update system
    apt-get update && apt-get upgrade -y
    
    # Configure UFW firewall
    ufw --force reset
    ufw default deny incoming
    ufw default allow outgoing
    ufw allow ssh
    ufw allow 80/tcp
    ufw allow 443/tcp
    ufw --force enable
    
    # Disable root login
    sed -i 's/PermitRootLogin yes/PermitRootLogin no/' /etc/ssh/sshd_config
    sed -i 's/#PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config
    systemctl restart ssh
    
    # Install fail2ban
    apt-get install -y fail2ban
    
    cat > /etc/fail2ban/jail.local << 'EOF'
[DEFAULT]
bantime = 3600
findtime = 600
maxretry = 5

[sshd]
enabled = true

[nginx-http-auth]
enabled = true

[nginx-noscript]
enabled = true

[nginx-badbots]
enabled = true

[nginx-noproxy]
enabled = true
EOF
    
    systemctl enable fail2ban
    systemctl restart fail2ban
    
    # Secure shared memory
    echo 'tmpfs /run/shm tmpfs defaults,noexec,nosuid 0 0' >> /etc/fstab
    
    # Set proper file permissions
    chmod 700 /home/frappe
    chmod 644 /etc/passwd
    chmod 644 /etc/group
    chmod 600 /etc/shadow
    
    echo "System hardening completed!"
}

# Application Security
secure_application() {
    echo "Securing Frappe application..."
    
    # Set proper ownership
    chown -R frappe:frappe /home/frappe/frappe-bench
    
    # Secure configuration files
    chmod 600 /home/frappe/frappe-bench/sites/*/site_config.json
    
    # Secure private files
    chmod 700 /home/frappe/frappe-bench/sites/*/private
    
    # Create security configuration
    cat > /home/frappe/frappe-bench/sites/common_site_config.json << 'EOF'
{
    "db_host": "127.0.0.1",
    "db_port": 3306,
    "redis_cache": "redis://localhost:6379",
    "redis_queue": "redis://localhost:6379",
    "redis_socketio": "redis://localhost:6379",
    
    "auto_update": false,
    "update_bench_on_update": false,
    "restart_supervisor_on_update": false,
    "restart_systemd_on_update": false,
    
    "serve_default_site": false,
    "rebase_on_pull": false,
    "shallow_clone": true,
    
    "background_workers": 1,
    "async_redis": false,
    
    "mail_server": "smtp.gmail.com",
    "mail_port": 587,
    "use_tls": true,
    
    "encryption_key": "generate_32_character_key_here",
    "host_name": "https://your-domain.com",
    
    "developer_mode": false,
    "allow_tests": false,
    "server_script_enabled": false,
    
    "monitor": true,
    "enable_telemetry": false,
    
    "limits": {
        "max_file_size": 10485760,
        "max_requests_per_hour": 1000
    }
}
EOF
    
    chown frappe:frappe /home/frappe/frappe-bench/sites/common_site_config.json
    chmod 600 /home/frappe/frappe-bench/sites/common_site_config.json
    
    echo "Application security completed!"
}

# Database Security
secure_database() {
    echo "Securing database..."
    
    # Create dedicated database user
    mysql -u root -p << 'EOF'
CREATE USER 'frappe_prod'@'localhost' IDENTIFIED BY 'secure_password_here';
GRANT ALL PRIVILEGES ON `frappe_prod`.* TO 'frappe_prod'@'localhost';
FLUSH PRIVILEGES;
EOF
    
    # Secure MySQL configuration
    cat >> /etc/mysql/conf.d/security.cnf << 'EOF'
[mysqld]
# Security settings
skip-show-database
local-infile=0
skip-symbolic-links
secure-file-priv=/var/lib/mysql-files/

# Logging
general_log = ON
general_log_file = /var/log/mysql/mysql.log

# Connection limits
max_user_connections = 50
max_connections = 200
EOF
    
    systemctl restart mysql
    
    echo "Database security completed!"
}

# Log Security
setup_log_rotation() {
    echo "Setting up log rotation..."
    
    cat > /etc/logrotate.d/frappe << 'EOF'
/home/frappe/frappe-bench/logs/*.log {
    daily
    missingok
    rotate 30
    compress
    notifempty
    create 644 frappe frappe
    postrotate
        /usr/bin/supervisorctl restart all
    endscript
}
EOF
    
    echo "Log rotation setup completed!"
}

# Main hardening function
main() {
    harden_system
    secure_application
    secure_database
    setup_log_rotation
    
    echo "Security hardening completed!"
    echo "Remember to:"
    echo "1. Change default passwords"
    echo "2. Set up intrusion detection"
    echo "3. Configure log monitoring"
    echo "4. Regular security updates"
}

main "$@"
```

## Performance Optimization

### Database Optimization

```python
# performance/database_optimization.py

class DatabaseOptimization:
    """Database performance optimization strategies"""
    
    def __init__(self):
        self.connection = frappe.db
        
    def optimize_mariadb_config(self):
        """Generate optimized MariaDB configuration"""
        
        # Calculate optimal settings based on available memory
        total_memory_gb = self.get_server_memory_gb()
        
        config = {
            # InnoDB settings
            "innodb_buffer_pool_size": f"{int(total_memory_gb * 0.7)}G",
            "innodb_log_file_size": "512M",
            "innodb_log_buffer_size": "64M",
            "innodb_flush_log_at_trx_commit": 2,
            "innodb_flush_method": "O_DIRECT",
            "innodb_file_per_table": "ON",
            "innodb_open_files": 4000,
            
            # Connection settings
            "max_connections": 300,
            "max_connect_errors": 10000,
            "max_allowed_packet": "256M",
            "wait_timeout": 600,
            "interactive_timeout": 600,
            
            # Query cache (disable for MySQL 5.7+)
            "query_cache_size": 0,
            "query_cache_type": 0,
            
            # MyISAM settings
            "key_buffer_size": "32M",
            "myisam_sort_buffer_size": "128M",
            
            # General settings
            "table_open_cache": 4000,
            "sort_buffer_size": "4M",
            "read_buffer_size": "2M",
            "read_rnd_buffer_size": "8M",
            "bulk_insert_buffer_size": "64M",
            
            # Binary logging
            "log_bin": "mysql-bin",
            "binlog_format": "ROW",
            "expire_logs_days": 7,
            "sync_binlog": 0,
            
            # Performance monitoring
            "slow_query_log": "ON",
            "slow_query_log_file": "/var/log/mysql/slow.log",
            "long_query_time": 1,
            "log_queries_not_using_indexes": "ON",
            
            # Character set
            "character_set_server": "utf8mb4",
            "collation_server": "utf8mb4_unicode_ci"
        }
        
        return self.generate_mysql_config(config)
        
    def get_server_memory_gb(self):
        """Get server memory in GB"""
        import psutil
        return psutil.virtual_memory().total / (1024**3)
        
    def generate_mysql_config(self, config):
        """Generate MySQL configuration file content"""
        
        config_content = "[mysqld]\n"
        for key, value in config.items():
            config_content += f"{key} = {value}\n"
            
        return config_content
        
    def optimize_queries(self):
        """Optimize common slow queries"""
        
        optimizations = [
            # Add indexes for common queries
            "ALTER TABLE `tabLibrary Book` ADD INDEX idx_status_category (status, category)",
            "ALTER TABLE `tabLibrary Issue` ADD INDEX idx_member_return_date (member, return_date)",
            "ALTER TABLE `tabLibrary Member` ADD INDEX idx_email (email)",
            "ALTER TABLE `tabUser` ADD INDEX idx_enabled_user_type (enabled, user_type)",
            
            # Add composite indexes for complex queries
            "ALTER TABLE `tabLibrary Book` ADD INDEX idx_category_year_status (category, publication_year, status)",
            "ALTER TABLE `tabLibrary Issue` ADD INDEX idx_issue_due_status (issue_date, due_date, return_date)",
            
            # Add full-text indexes for search
            "ALTER TABLE `tabLibrary Book` ADD FULLTEXT idx_search (book_name, author, description)",
            
            # Optimize table structure
            "OPTIMIZE TABLE `tabLibrary Book`",
            "OPTIMIZE TABLE `tabLibrary Issue`",
            "OPTIMIZE TABLE `tabLibrary Member`",
            
            # Update table statistics
            "ANALYZE TABLE `tabLibrary Book`",
            "ANALYZE TABLE `tabLibrary Issue`",
            "ANALYZE TABLE `tabLibrary Member`"
        ]
        
        results = []
        for query in optimizations:
            try:
                result = frappe.db.sql(query)
                results.append({"query": query, "status": "success", "result": result})
            except Exception as e:
                results.append({"query": query, "status": "error", "error": str(e)})
                
        return results
        
    def analyze_slow_queries(self):
        """Analyze slow query log"""
        
        slow_queries = frappe.db.sql("""
            SELECT 
                sql_text,
                exec_count,
                total_latency,
                mean_latency,
                rows_examined_avg,
                rows_sent_avg
            FROM performance_schema.statements_summary_by_digest
            WHERE avg_timer_wait > 1000000000  -- 1 second
            ORDER BY avg_timer_wait DESC
            LIMIT 20
        """, as_dict=True)
        
        return slow_queries
        
    def partition_large_tables(self):
        """Implement table partitioning for large tables"""
        
        # Partition Library Issue table by date
        partition_sql = """
        ALTER TABLE `tabLibrary Issue` 
        PARTITION BY RANGE (YEAR(issue_date)) (
            PARTITION p2020 VALUES LESS THAN (2021),
            PARTITION p2021 VALUES LESS THAN (2022),
            PARTITION p2022 VALUES LESS THAN (2023),
            PARTITION p2023 VALUES LESS THAN (2024),
            PARTITION p2024 VALUES LESS THAN (2025),
            PARTITION p_future VALUES LESS THAN MAXVALUE
        )
        """
        
        try:
            frappe.db.sql(partition_sql)
            return {"status": "success", "message": "Partitioning applied successfully"}
        except Exception as e:
            return {"status": "error", "message": str(e)}

class CacheOptimization:
    """Caching strategies for performance"""
    
    def __init__(self):
        self.redis_client = frappe.cache()
        
    def implement_multi_level_caching(self):
        """Implement multi-level caching strategy"""
        
        caching_layers = {
            "browser_cache": {
                "static_assets": "1 year",
                "api_responses": "5 minutes",
                "user_preferences": "1 hour"
            },
            "cdn_cache": {
                "static_files": "1 year", 
                "images": "1 month",
                "api_responses": "1 hour"
            },
            "redis_cache": {
                "user_sessions": "24 hours",
                "query_results": "15 minutes",
                "computed_data": "1 hour"
            },
            "application_cache": {
                "doctypes": "restart",
                "translations": "1 day",
                "system_settings": "1 hour"
            }
        }
        
        return caching_layers
        
    def optimize_redis_config(self):
        """Optimize Redis configuration for Frappe"""
        
        redis_config = """
        # Memory optimization
        maxmemory 4gb
        maxmemory-policy allkeys-lru
        
        # Persistence (for production)
        save 900 1
        save 300 10
        save 60 10000
        
        # Network optimization
        tcp-keepalive 60
        timeout 300
        
        # Performance
        hash-max-ziplist-entries 512
        hash-max-ziplist-value 64
        list-max-ziplist-size -2
        set-max-intset-entries 512
        zset-max-ziplist-entries 128
        zset-max-ziplist-value 64
        
        # Logging
        loglevel notice
        logfile /var/log/redis/redis-server.log
        
        # Security
        requirepass your_secure_password
        rename-command FLUSHDB ""
        rename-command FLUSHALL ""
        rename-command CONFIG "CONFIG_b835e3b8f5c941f7"
        """
        
        return redis_config
        
    def cache_warming_strategy(self):
        """Implement cache warming for better performance"""
        
        def warm_common_caches():
            # Warm user permissions cache
            users = frappe.get_all("User", filters={"enabled": 1}, limit=100)
            for user in users:
                frappe.get_roles(user.name)
                
            # Warm DocType metadata cache
            common_doctypes = [
                "Library Book", "Library Member", "Library Issue",
                "User", "Role", "Custom Field"
            ]
            for doctype in common_doctypes:
                frappe.get_meta(doctype)
                
            # Warm system settings
            frappe.get_single("System Settings")
            
            # Warm translation cache
            frappe.get_all("Translation", limit=1000)
            
        return warm_common_caches

# Application Performance Monitoring
class PerformanceMonitor:
    """Monitor application performance"""
    
    def __init__(self):
        self.metrics = {}
        
    def collect_performance_metrics(self):
        """Collect comprehensive performance metrics"""
        
        metrics = {
            "database": self.get_database_metrics(),
            "redis": self.get_redis_metrics(),
            "system": self.get_system_metrics(),
            "application": self.get_application_metrics()
        }
        
        return metrics
        
    def get_database_metrics(self):
        """Get database performance metrics"""
        
        db_metrics = frappe.db.sql("""
            SELECT
                'queries_per_second' as metric,
                ROUND(Queries / Uptime, 2) as value
            FROM information_schema.GLOBAL_STATUS
            WHERE VARIABLE_NAME IN ('QUERIES', 'UPTIME')
            
            UNION ALL
            
            SELECT
                'slow_queries' as metric,
                VARIABLE_VALUE as value
            FROM information_schema.GLOBAL_STATUS
            WHERE VARIABLE_NAME = 'SLOW_QUERIES'
            
            UNION ALL
            
            SELECT
                'connections' as metric,
                VARIABLE_VALUE as value
            FROM information_schema.GLOBAL_STATUS
            WHERE VARIABLE_NAME = 'THREADS_CONNECTED'
        """, as_dict=True)
        
        return {metric["metric"]: metric["value"] for metric in db_metrics}
        
    def get_redis_metrics(self):
        """Get Redis performance metrics"""
        
        redis_info = frappe.cache().connection_pool.connection().info()
        
        return {
            "used_memory": redis_info.get("used_memory_human"),
            "connected_clients": redis_info.get("connected_clients"),
            "total_commands_processed": redis_info.get("total_commands_processed"),
            "keyspace_hits": redis_info.get("keyspace_hits"),
            "keyspace_misses": redis_info.get("keyspace_misses")
        }
        
    def get_system_metrics(self):
        """Get system performance metrics"""
        
        import psutil
        
        return {
            "cpu_percent": psutil.cpu_percent(interval=1),
            "memory_percent": psutil.virtual_memory().percent,
            "disk_usage": psutil.disk_usage('/').percent,
            "load_average": psutil.getloadavg()
        }
        
    def get_application_metrics(self):
        """Get application-specific metrics"""
        
        return {
            "active_users": self.get_active_users_count(),
            "background_jobs": self.get_background_jobs_count(),
            "error_rate": self.get_error_rate(),
            "response_time": self.get_average_response_time()
        }
        
    def get_active_users_count(self):
        """Get count of active users"""
        
        from datetime import datetime, timedelta
        
        active_threshold = datetime.now() - timedelta(minutes=30)
        
        return frappe.db.count("Sessions", {
            "last_updated": [">=", active_threshold]
        })
        
    def get_background_jobs_count(self):
        """Get background jobs statistics"""
        
        from rq import Queue
        import redis
        
        redis_conn = redis.Redis.from_url(frappe.conf.redis_queue)
        
        queues = ["default", "short", "long"]
        job_stats = {}
        
        for queue_name in queues:
            queue = Queue(queue_name, connection=redis_conn)
            job_stats[queue_name] = {
                "waiting": len(queue),
                "failed": len(queue.failed_job_registry),
                "finished": len(queue.finished_job_registry)
            }
            
        return job_stats
        
    def get_error_rate(self):
        """Calculate error rate from logs"""
        
        from datetime import datetime, timedelta
        
        last_hour = datetime.now() - timedelta(hours=1)
        
        total_requests = frappe.db.count("Activity Log", {
            "creation": [">=", last_hour],
            "status": ["in", ["Success", "Error"]]
        })
        
        error_requests = frappe.db.count("Activity Log", {
            "creation": [">=", last_hour],
            "status": "Error"
        })
        
        if total_requests > 0:
            return round((error_requests / total_requests) * 100, 2)
        return 0
        
    def get_average_response_time(self):
        """Calculate average response time"""
        
        # This would typically come from monitoring tools
        # For demo, return a placeholder
        return 250  # milliseconds
```

## Monitoring and Logging

### Comprehensive Monitoring Setup

Based on analysis of `frappe/monitor.py:20-80`, here's a comprehensive monitoring implementation:

```python
# monitoring/monitoring_stack.py

class MonitoringStack:
    """Complete monitoring stack for Frappe applications"""
    
    def __init__(self):
        self.monitoring_config = self.get_monitoring_config()
        
    def setup_application_monitoring(self):
        """Set up application-level monitoring"""
        
        # Enable Frappe's built-in monitoring
        monitoring_config = {
            "monitor": True,
            "monitor_redis_key": "monitor-transactions",
            "monitor_max_entries": 1000000,
            "enable_telemetry": False,
            
            # Performance thresholds
            "slow_query_threshold": 1.0,
            "response_time_threshold": 5.0,
            "memory_usage_threshold": 80,
            
            # Alerting configuration
            "alert_channels": {
                "email": ["admin@company.com", "devops@company.com"],
                "slack": "https://hooks.slack.com/services/...",
                "pagerduty": "service_key_here"
            }
        }
        
        return monitoring_config
        
    def setup_prometheus_metrics(self):
        """Set up Prometheus metrics collection"""
        
        prometheus_config = """
# Prometheus configuration for Frappe
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files:
  - "frappe_rules.yml"

alerting:
  alertmanagers:
    - static_configs:
        - targets:
          - alertmanager:9093

scrape_configs:
  # Frappe application metrics
  - job_name: 'frappe-web'
    static_configs:
      - targets: ['localhost:8000', 'localhost:8001', 'localhost:8002']
    metrics_path: /api/method/frappe.monitor.get_metrics
    scrape_interval: 30s
    
  # System metrics
  - job_name: 'node-exporter'
    static_configs:
      - targets: ['localhost:9100']
      
  # Database metrics
  - job_name: 'mysql-exporter'
    static_configs:
      - targets: ['localhost:9104']
      
  # Redis metrics
  - job_name: 'redis-exporter'
    static_configs:
      - targets: ['localhost:9121']
      
  # Nginx metrics
  - job_name: 'nginx-exporter'
    static_configs:
      - targets: ['localhost:9113']
"""
        
        return prometheus_config
        
    def setup_grafana_dashboards(self):
        """Create Grafana dashboards for monitoring"""
        
        dashboard_config = {
            "frappe_application_dashboard": {
                "title": "Frappe Application Metrics",
                "panels": [
                    {
                        "title": "Request Rate",
                        "type": "graph",
                        "targets": ["rate(http_requests_total[5m])"]
                    },
                    {
                        "title": "Response Time",
                        "type": "graph", 
                        "targets": ["histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))"]
                    },
                    {
                        "title": "Error Rate",
                        "type": "singlestat",
                        "targets": ["rate(http_requests_total{status=~\"5..\"}[5m])"]
                    },
                    {
                        "title": "Active Users",
                        "type": "singlestat",
                        "targets": ["frappe_active_users"]
                    },
                    {
                        "title": "Background Jobs",
                        "type": "graph",
                        "targets": ["frappe_background_jobs_waiting", "frappe_background_jobs_failed"]
                    },
                    {
                        "title": "Database Queries",
                        "type": "graph",
                        "targets": ["rate(frappe_db_queries_total[5m])"]
                    }
                ]
            },
            
            "system_dashboard": {
                "title": "System Metrics",
                "panels": [
                    {
                        "title": "CPU Usage",
                        "type": "graph",
                        "targets": ["100 - (avg(rate(node_cpu_seconds_total{mode=\"idle\"}[5m])) * 100)"]
                    },
                    {
                        "title": "Memory Usage",
                        "type": "graph",
                        "targets": ["(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100"]
                    },
                    {
                        "title": "Disk I/O",
                        "type": "graph",
                        "targets": ["rate(node_disk_io_time_seconds_total[5m])"]
                    },
                    {
                        "title": "Network Traffic",
                        "type": "graph",
                        "targets": ["rate(node_network_receive_bytes_total[5m])", "rate(node_network_transmit_bytes_total[5m])"]
                    }
                ]
            }
        }
        
        return dashboard_config
        
    def setup_alerting_rules(self):
        """Define alerting rules for critical metrics"""
        
        alert_rules = """
groups:
  - name: frappe_alerts
    rules:
      - alert: HighResponseTime
        expr: histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m])) > 5
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "High response time detected"
          description: "95th percentile response time is {{ $value }}s"
          
      - alert: HighErrorRate  
        expr: rate(http_requests_total{status=~"5.."}[5m]) > 0.1
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High error rate detected"
          description: "Error rate is {{ $value }} requests/sec"
          
      - alert: DatabaseConnectionsHigh
        expr: mysql_global_status_threads_connected > 200
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "High database connections"
          description: "Database has {{ $value }} active connections"
          
      - alert: RedisMemoryHigh
        expr: (redis_memory_used_bytes / redis_memory_max_bytes) * 100 > 90
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Redis memory usage high"
          description: "Redis memory usage is {{ $value }}%"
          
      - alert: DiskSpaceLow
        expr: (node_filesystem_free_bytes / node_filesystem_size_bytes) * 100 < 10
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Low disk space"
          description: "Disk space is {{ $value }}% full"
          
      - alert: BackgroundJobsStuck
        expr: frappe_background_jobs_waiting > 100
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Background jobs accumulating"
          description: "{{ $value }} background jobs waiting"
"""
        
        return alert_rules

class LoggingConfiguration:
    """Centralized logging configuration"""
    
    def setup_structured_logging(self):
        """Set up structured logging with JSON format"""
        
        logging_config = {
            "version": 1,
            "disable_existing_loggers": False,
            "formatters": {
                "json": {
                    "format": '{"timestamp": "%(asctime)s", "level": "%(levelname)s", "logger": "%(name)s", "message": "%(message)s", "module": "%(module)s", "function": "%(funcName)s", "line": %(lineno)d}',
                    "datefmt": "%Y-%m-%d %H:%M:%S"
                },
                "detailed": {
                    "format": "%(asctime)s - %(name)s - %(levelname)s - %(module)s:%(funcName)s:%(lineno)d - %(message)s"
                }
            },
            "handlers": {
                "console": {
                    "class": "logging.StreamHandler",
                    "level": "INFO",
                    "formatter": "json",
                    "stream": "ext://sys.stdout"
                },
                "file": {
                    "class": "logging.handlers.RotatingFileHandler",
                    "level": "DEBUG",
                    "formatter": "json",
                    "filename": "/home/frappe/frappe-bench/logs/application.log",
                    "maxBytes": 100000000,
                    "backupCount": 10
                },
                "error_file": {
                    "class": "logging.handlers.RotatingFileHandler",
                    "level": "ERROR",
                    "formatter": "detailed",
                    "filename": "/home/frappe/frappe-bench/logs/error.log",
                    "maxBytes": 100000000,
                    "backupCount": 10
                }
            },
            "loggers": {
                "frappe": {
                    "level": "INFO",
                    "handlers": ["console", "file", "error_file"],
                    "propagate": False
                },
                "frappe.database": {
                    "level": "WARNING",
                    "handlers": ["file"],
                    "propagate": False
                },
                "frappe.auth": {
                    "level": "INFO",
                    "handlers": ["file", "error_file"],
                    "propagate": False
                }
            },
            "root": {
                "level": "INFO",
                "handlers": ["console", "file"]
            }
        }
        
        return logging_config
        
    def setup_log_aggregation(self):
        """Set up log aggregation with ELK stack"""
        
        logstash_config = """
input {
  file {
    path => "/home/frappe/frappe-bench/logs/*.log"
    start_position => "beginning"
    codec => json
    tags => ["frappe"]
  }
  
  file {
    path => "/var/log/nginx/access.log"
    start_position => "beginning" 
    codec => plain
    tags => ["nginx", "access"]
  }
  
  file {
    path => "/var/log/nginx/error.log"
    start_position => "beginning"
    codec => plain
    tags => ["nginx", "error"]
  }
}

filter {
  if "frappe" in [tags] {
    mutate {
      add_field => { "service" => "frappe" }
    }
  }
  
  if "nginx" in [tags] {
    if "access" in [tags] {
      grok {
        match => { 
          "message" => "%{NGINXACCESS}"
        }
      }
    }
    
    mutate {
      add_field => { "service" => "nginx" }
    }
  }
  
  date {
    match => [ "timestamp", "yyyy-MM-dd HH:mm:ss" ]
    target => "@timestamp"
  }
}

output {
  elasticsearch {
    hosts => ["elasticsearch:9200"]
    index => "frappe-logs-%{+YYYY.MM.dd}"
  }
}
"""
        
        return logstash_config
        
    def create_log_analysis_queries(self):
        """Create common log analysis queries"""
        
        queries = {
            "error_analysis": """
            SELECT 
                DATE(timestamp) as date,
                level,
                COUNT(*) as count,
                module,
                message
            FROM logs 
            WHERE level IN ('ERROR', 'CRITICAL')
            AND timestamp >= NOW() - INTERVAL 24 HOUR
            GROUP BY date, level, module, message
            ORDER BY count DESC
            """,
            
            "performance_analysis": """
            SELECT
                DATE(timestamp) as date,
                AVG(response_time) as avg_response_time,
                MAX(response_time) as max_response_time,
                COUNT(*) as total_requests
            FROM access_logs
            WHERE timestamp >= NOW() - INTERVAL 7 DAY
            GROUP BY date
            ORDER BY date
            """,
            
            "user_activity": """
            SELECT
                user_id,
                COUNT(*) as activity_count,
                MAX(timestamp) as last_activity
            FROM activity_logs
            WHERE timestamp >= NOW() - INTERVAL 1 DAY
            GROUP BY user_id
            ORDER BY activity_count DESC
            LIMIT 20
            """
        }
        
        return queries
```

## Scaling Strategies

### Horizontal Scaling

```python
# scaling/horizontal_scaling.py

class HorizontalScaling:
    """Horizontal scaling strategies for Frappe applications"""
    
    def __init__(self):
        self.scaling_metrics = self.define_scaling_metrics()
        
    def define_scaling_metrics(self):
        """Define metrics that trigger scaling decisions"""
        
        return {
            "scale_up_triggers": {
                "cpu_threshold": 70,
                "memory_threshold": 80,
                "response_time_threshold": 3.0,
                "queue_length_threshold": 100,
                "error_rate_threshold": 5.0
            },
            "scale_down_triggers": {
                "cpu_threshold": 30,
                "memory_threshold": 40,
                "response_time_threshold": 1.0,
                "queue_length_threshold": 10,
                "min_instances": 2
            }
        }
        
    def kubernetes_deployment(self):
        """Kubernetes deployment configuration for scaling"""
        
        k8s_config = """
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frappe-web
  labels:
    app: frappe-web
spec:
  replicas: 3
  selector:
    matchLabels:
      app: frappe-web
  template:
    metadata:
      labels:
        app: frappe-web
    spec:
      containers:
      - name: frappe-web
        image: frappe/frappe:latest
        ports:
        - containerPort: 8000
        env:
        - name: FRAPPE_SITE_NAME
          value: "production.local"
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
        livenessProbe:
          httpGet:
            path: /api/method/ping
            port: 8000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /api/method/ping
            port: 8000
          initialDelaySeconds: 5
          periodSeconds: 5

---
apiVersion: v1
kind: Service
metadata:
  name: frappe-web-service
spec:
  selector:
    app: frappe-web
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8000
  type: LoadBalancer

---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: frappe-web-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: frappe-web
  minReplicas: 3
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
      - type: Percent
        value: 100
        periodSeconds: 15
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 50
        periodSeconds: 60

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frappe-worker
spec:
  replicas: 5
  selector:
    matchLabels:
      app: frappe-worker
  template:
    metadata:
      labels:
        app: frappe-worker
    spec:
      containers:
      - name: frappe-worker
        image: frappe/frappe:latest
        command: ["python", "-m", "frappe.utils.bench", "worker"]
        env:
        - name: FRAPPE_SITE_NAME
          value: "production.local"
        - name: WORKER_TYPE
          value: "default"
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
"""
        
        return k8s_config
        
    def load_balancer_configuration(self):
        """Load balancer configuration for multiple instances"""
        
        haproxy_config = """
global
    log 127.0.0.1:514 local0
    chroot /var/lib/haproxy
    stats socket /run/haproxy/admin.sock mode 660 level admin
    stats timeout 30s
    user haproxy
    group haproxy
    daemon

defaults
    mode http
    log global
    option httplog
    option dontlognull
    option log-health-checks
    option forwardfor
    option http-server-close
    timeout connect 5000
    timeout client  50000
    timeout server  50000
    errorfile 400 /etc/haproxy/errors/400.http
    errorfile 403 /etc/haproxy/errors/403.http
    errorfile 408 /etc/haproxy/errors/408.http
    errorfile 500 /etc/haproxy/errors/500.http
    errorfile 502 /etc/haproxy/errors/502.http
    errorfile 503 /etc/haproxy/errors/503.http
    errorfile 504 /etc/haproxy/errors/504.http

# Statistics interface
frontend stats
    bind *:8404
    stats enable
    stats uri /stats
    stats refresh 30s
    stats admin if TRUE

# Frontend for web traffic
frontend web_frontend
    bind *:80
    bind *:443 ssl crt /etc/ssl/certs/frappe.pem
    redirect scheme https if !{ ssl_fc }
    
    # Rate limiting
    stick-table type ip size 100k expire 30s store http_req_rate(10s),http_err_rate(10s)
    http-request track-sc0 src
    http-request reject if { sc_http_req_rate(0) gt 20 }
    
    # Route to backend
    default_backend web_servers

# Backend web servers
backend web_servers
    balance roundrobin
    option httpchk GET /api/method/ping
    http-check expect status 200
    
    # Server health checks
    default-server inter 3s fall 3 rise 2
    
    server web1 10.0.1.10:8000 check
    server web2 10.0.1.11:8000 check
    server web3 10.0.1.12:8000 check
    server web4 10.0.1.13:8000 check backup

# Frontend for Socket.IO
frontend socketio_frontend
    bind *:9000
    default_backend socketio_servers

# Backend Socket.IO servers
backend socketio_servers
    balance source
    option httpchk GET /socket.io/
    
    server socketio1 10.0.1.10:9000 check
    server socketio2 10.0.1.11:9000 check
"""
        
        return haproxy_config
        
    def database_scaling_strategies(self):
        """Database scaling strategies"""
        
        strategies = {
            "read_replicas": {
                "description": "Use read replicas for read-heavy workloads",
                "implementation": """
                # Master-slave configuration
                [mysql-master]
                bind-address = 0.0.0.0
                server-id = 1
                log-bin = mysql-bin
                binlog-do-db = frappe_production
                
                [mysql-slave]
                server-id = 2
                relay-log = mysql-relay
                replicate-do-db = frappe_production
                """,
                "frappe_config": {
                    "db_host": "mysql-master.local",
                    "db_read_host": "mysql-slave.local",
                    "enable_read_replica": True
                }
            },
            
            "connection_pooling": {
                "description": "Use connection pooling to manage database connections",
                "implementation": """
                # ProxySQL configuration
                mysql_servers = (
                    { address="mysql-master.local", port=3306, hostgroup=0, weight=1000 },
                    { address="mysql-slave1.local", port=3306, hostgroup=1, weight=900 },
                    { address="mysql-slave2.local", port=3306, hostgroup=1, weight=900 }
                )
                
                mysql_query_rules = (
                    { match_pattern="^SELECT.*", destination_hostgroup=1, apply=1 },
                    { match_pattern="^INSERT|UPDATE|DELETE.*", destination_hostgroup=0, apply=1 }
                )
                """,
                "benefits": [
                    "Connection reuse",
                    "Load distribution", 
                    "Failover handling",
                    "Query routing"
                ]
            },
            
            "database_sharding": {
                "description": "Shard databases by site or tenant",
                "implementation": """
                # Multi-database configuration
                {
                    "db_host": "mysql-cluster.local",
                    "db_sharding": {
                        "enabled": true,
                        "strategy": "site_based",
                        "shards": {
                            "shard1": {
                                "sites": ["site1.local", "site2.local"],
                                "host": "mysql-shard1.local"
                            },
                            "shard2": {
                                "sites": ["site3.local", "site4.local"], 
                                "host": "mysql-shard2.local"
                            }
                        }
                    }
                }
                """
            }
        }
        
        return strategies
        
    def auto_scaling_implementation(self):
        """Implement auto-scaling based on metrics"""
        
        auto_scaling_script = """
#!/usr/bin/env python3
import time
import requests
import subprocess
import json
from datetime import datetime, timedelta

class FrappeAutoScaler:
    def __init__(self, config):
        self.config = config
        self.current_instances = self.get_current_instances()
        
    def get_metrics(self):
        # Get metrics from monitoring endpoint
        response = requests.get(f"{self.config['monitoring_url']}/metrics")
        return response.json()
        
    def get_current_instances(self):
        # Get current number of running instances
        result = subprocess.run(['kubectl', 'get', 'deployment', 'frappe-web', '-o', 'json'], 
                               capture_output=True, text=True)
        deployment = json.loads(result.stdout)
        return deployment['spec']['replicas']
        
    def scale_up(self, target_instances):
        subprocess.run(['kubectl', 'scale', 'deployment', 'frappe-web', 
                       f'--replicas={target_instances}'])
        print(f"Scaled up to {target_instances} instances")
        
    def scale_down(self, target_instances):
        if target_instances >= self.config['min_instances']:
            subprocess.run(['kubectl', 'scale', 'deployment', 'frappe-web',
                           f'--replicas={target_instances}'])
            print(f"Scaled down to {target_instances} instances")
        
    def should_scale_up(self, metrics):
        return (
            metrics['cpu_usage'] > self.config['scale_up_cpu_threshold'] or
            metrics['memory_usage'] > self.config['scale_up_memory_threshold'] or
            metrics['response_time'] > self.config['scale_up_response_time_threshold']
        )
        
    def should_scale_down(self, metrics):
        return (
            metrics['cpu_usage'] < self.config['scale_down_cpu_threshold'] and
            metrics['memory_usage'] < self.config['scale_down_memory_threshold'] and
            metrics['response_time'] < self.config['scale_down_response_time_threshold'] and
            self.current_instances > self.config['min_instances']
        )
        
    def run(self):
        while True:
            try:
                metrics = self.get_metrics()
                
                if self.should_scale_up(metrics):
                    new_instances = min(self.current_instances + 2, self.config['max_instances'])
                    self.scale_up(new_instances)
                    self.current_instances = new_instances
                    
                elif self.should_scale_down(metrics):
                    new_instances = max(self.current_instances - 1, self.config['min_instances'])
                    self.scale_down(new_instances)
                    self.current_instances = new_instances
                    
                time.sleep(60)  # Check every minute
                
            except Exception as e:
                print(f"Auto-scaling error: {e}")
                time.sleep(60)

if __name__ == "__main__":
    config = {
        'monitoring_url': 'http://prometheus:9090',
        'min_instances': 3,
        'max_instances': 20,
        'scale_up_cpu_threshold': 70,
        'scale_up_memory_threshold': 80,
        'scale_up_response_time_threshold': 3.0,
        'scale_down_cpu_threshold': 30,
        'scale_down_memory_threshold': 40,
        'scale_down_response_time_threshold': 1.0
    }
    
    scaler = FrappeAutoScaler(config)
    scaler.run()
"""
        
        return auto_scaling_script

class VerticalScaling:
    """Vertical scaling strategies for resource optimization"""
    
    def optimize_resource_allocation(self):
        """Optimize resource allocation for different components"""
        
        resource_profiles = {
            "web_servers": {
                "cpu_cores": 4,
                "memory_gb": 8,
                "disk_gb": 50,
                "worker_processes": "auto",
                "worker_connections": 1024,
                "optimization": "I/O intensive workload"
            },
            
            "background_workers": {
                "cpu_cores": 2, 
                "memory_gb": 4,
                "disk_gb": 20,
                "concurrent_jobs": 4,
                "queue_size": 1000,
                "optimization": "CPU intensive workload"
            },
            
            "database_server": {
                "cpu_cores": 8,
                "memory_gb": 32,
                "disk_gb": 500,
                "disk_type": "NVMe SSD",
                "innodb_buffer_pool": "24GB",
                "optimization": "Memory and I/O intensive"
            },
            
            "cache_server": {
                "cpu_cores": 2,
                "memory_gb": 16,
                "disk_gb": 20,
                "max_memory": "14GB",
                "optimization": "Memory intensive"
            }
        }
        
        return resource_profiles
        
    def performance_tuning_guide(self):
        """Comprehensive performance tuning guide"""
        
        tuning_guide = {
            "operating_system": {
                "kernel_parameters": {
                    "net.core.somaxconn": 65535,
                    "net.ipv4.tcp_max_syn_backlog": 65535,
                    "net.ipv4.tcp_keepalive_time": 300,
                    "vm.swappiness": 10,
                    "fs.file-max": 2097152
                },
                
                "limits_conf": {
                    "soft_nofile": 65535,
                    "hard_nofile": 65535,
                    "soft_nproc": 65535,
                    "hard_nproc": 65535
                }
            },
            
            "application_level": {
                "gunicorn_config": {
                    "workers": "2 * CPU_CORES + 1",
                    "worker_class": "gevent",
                    "worker_connections": 1000,
                    "max_requests": 1000,
                    "max_requests_jitter": 100,
                    "timeout": 120,
                    "keepalive": 5
                },
                
                "python_optimizations": {
                    "garbage_collection": "gc.set_threshold(700, 10, 10)",
                    "bytecode_optimization": "-O flag",
                    "memory_profiling": "Use memory_profiler for optimization"
                }
            },
            
            "database_tuning": {
                "connection_optimization": {
                    "max_connections": "CPU_CORES * 25",
                    "wait_timeout": 300,
                    "interactive_timeout": 300
                },
                
                "query_optimization": {
                    "query_cache": "Disable for MySQL 5.7+",
                    "slow_query_log": "Enable with 1s threshold",
                    "index_optimization": "Regular ANALYZE TABLE"
                }
            }
        }
        
        return tuning_guide
```

## Best Practices

### Production Deployment Checklist

```python
# deployment/production_checklist.py

class ProductionDeploymentChecklist:
    """Comprehensive production deployment checklist"""
    
    def __init__(self):
        self.checklist = self.create_checklist()
        
    def create_checklist(self):
        """Create detailed production deployment checklist"""
        
        return {
            "pre_deployment": [
                {
                    "category": "Security",
                    "tasks": [
                        "Enable firewall (UFW) with appropriate rules",
                        "Disable root SSH access",
                        "Configure SSH key-based authentication",
                        "Install and configure fail2ban",
                        "Set up SSL certificates",
                        "Configure security headers in Nginx",
                        "Change default passwords",
                        "Enable audit logging"
                    ]
                },
                {
                    "category": "Performance",
                    "tasks": [
                        "Optimize database configuration",
                        "Configure Redis for production",
                        "Set up connection pooling",
                        "Configure Nginx for performance",
                        "Enable gzip compression",
                        "Set up CDN for static assets",
                        "Configure caching strategies"
                    ]
                },
                {
                    "category": "Monitoring", 
                    "tasks": [
                        "Set up application monitoring",
                        "Configure system monitoring",
                        "Set up log aggregation",
                        "Configure alerting rules",
                        "Set up health checks",
                        "Configure performance monitoring",
                        "Set up error tracking"
                    ]
                },
                {
                    "category": "Backup",
                    "tasks": [
                        "Configure automated database backups",
                        "Set up file system backups",
                        "Configure off-site backup storage",
                        "Test backup restoration",
                        "Set up backup monitoring",
                        "Document backup procedures"
                    ]
                }
            ],
            
            "deployment": [
                {
                    "category": "Application Deployment",
                    "tasks": [
                        "Deploy application code",
                        "Run database migrations",
                        "Update configurations",
                        "Install/update dependencies",
                        "Build static assets",
                        "Update supervisor configurations",
                        "Restart services",
                        "Verify deployment"
                    ]
                }
            ],
            
            "post_deployment": [
                {
                    "category": "Verification",
                    "tasks": [
                        "Test application functionality",
                        "Verify all services are running",
                        "Check application logs for errors",
                        "Validate performance metrics",
                        "Test backup procedures",
                        "Verify monitoring and alerting",
                        "Test disaster recovery procedures",
                        "Update documentation"
                    ]
                }
            ]
        }
        
    def validate_deployment(self):
        """Validate production deployment"""
        
        validation_tests = {
            "system_health": [
                "Check system resources (CPU, memory, disk)",
                "Verify network connectivity",
                "Check service status",
                "Validate file permissions"
            ],
            
            "application_health": [
                "Test web application response",
                "Verify database connectivity",
                "Check background job processing",
                "Test API endpoints",
                "Verify file uploads/downloads"
            ],
            
            "security_validation": [
                "Test SSL certificate",
                "Verify firewall rules", 
                "Check authentication systems",
                "Validate access controls",
                "Test rate limiting"
            ],
            
            "performance_validation": [
                "Run load tests",
                "Check response times",
                "Validate caching",
                "Test database performance",
                "Verify CDN functionality"
            ]
        }
        
        return validation_tests
        
    def create_runbook(self):
        """Create operational runbook"""
        
        runbook = {
            "daily_operations": [
                "Check system health dashboard",
                "Review application logs for errors", 
                "Monitor performance metrics",
                "Check backup status",
                "Review security alerts"
            ],
            
            "weekly_operations": [
                "Update system packages",
                "Review and rotate logs",
                "Test backup restoration",
                "Review performance trends",
                "Update documentation"
            ],
            
            "monthly_operations": [
                "Security audit and updates",
                "Performance optimization review",
                "Capacity planning review",
                "Disaster recovery testing",
                "Update monitoring and alerting rules"
            ],
            
            "incident_response": [
                "Application downtime response",
                "Database issues response",
                "Security incident response",
                "Performance degradation response",
                "Data corruption response"
            ]
        }
        
        return runbook

def generate_deployment_report():
    """Generate deployment status report"""
    
    report = {
        "deployment_info": {
            "timestamp": frappe.utils.now(),
            "environment": "production",
            "version": frappe.__version__,
            "deployed_by": frappe.session.user
        },
        
        "system_info": {
            "os": "Ubuntu 20.04 LTS",
            "python": "3.8.10",
            "node": "16.14.0",
            "database": "MariaDB 10.6",
            "redis": "6.2.7"
        },
        
        "services_status": {
            "nginx": "active",
            "gunicorn": "active (3 workers)",
            "redis": "active", 
            "mysql": "active",
            "supervisor": "active",
            "background_workers": "active (5 workers)"
        },
        
        "performance_metrics": {
            "response_time": "< 500ms",
            "cpu_usage": "< 70%",
            "memory_usage": "< 80%",
            "disk_usage": "< 70%"
        },
        
        "security_status": {
            "ssl_certificate": "valid",
            "firewall": "enabled",
            "fail2ban": "active",
            "security_headers": "configured"
        }
    }
    
    return report
```

---

**Next Steps**: Continue with [Troubleshooting Guide](11-troubleshooting-guide.md) to learn about common issues, debugging techniques, error resolution, and maintenance procedures for Frappe applications.