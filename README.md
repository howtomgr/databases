# Database Installation Guide

Comprehensive installation and configuration guide for popular database management systems including MySQL/MariaDB, PostgreSQL, MongoDB, and more. Complete with security hardening and performance optimization for production environments.

## Table of Contents
1. [Prerequisites](#prerequisites)
2. [Supported Operating Systems](#supported-operating-systems)
3. [Installation](#installation)
4. [Configuration](#configuration)
5. [Service Management](#service-management)
6. [Troubleshooting](#troubleshooting)
7. [Security Considerations](#security-considerations)
8. [Performance Tuning](#performance-tuning)
9. [Backup and Restore](#backup-and-restore)
10. [System Requirements](#system-requirements)
11. [Support](#support)
12. [Contributing](#contributing)
13. [License](#license)
14. [Acknowledgments](#acknowledgments)
15. [Version History](#version-history)
16. [Appendices](#appendices)

## 1. Prerequisites

- Linux system (any modern distribution)
- Root or sudo access
- 4GB RAM minimum, 8GB+ recommended for production
- SSD storage recommended for database files
- Network connectivity for replication setups (if applicable)

## MySQL/MariaDB Installation

### Ubuntu/Debian
```bash
# Update system
sudo apt update

# Install MariaDB (recommended over MySQL)
sudo apt install -y mariadb-server mariadb-client

# Or install MySQL
sudo apt install -y mysql-server mysql-client

# Secure installation
sudo mysql_secure_installation

# Enable and start service
sudo systemctl enable --now mariadb  # or mysql

# Verify installation
mysql --version
sudo systemctl status mariadb
```

### RHEL/CentOS/Rocky Linux/AlmaLinux
```bash
# Install MariaDB from official repository
sudo tee /etc/yum.repos.d/MariaDB.repo > /dev/null <<EOF
[mariadb]
name = MariaDB
baseurl = https://mirror.its.dal.ca/mariadb/yum/10.11/rhel\$releasever-\$basearch
module_hotfixes = 1
gpgkey = https://mirror.its.dal.ca/mariadb/yum/RPM-GPG-KEY-MariaDB
gpgcheck = 1
EOF

sudo yum install -y MariaDB-server MariaDB-client MariaDB-backup

# Enable and start service
sudo systemctl enable --now mariadb

# Secure installation
sudo mysql_secure_installation

# Configure firewall
sudo firewall-cmd --permanent --add-service=mysql
sudo firewall-cmd --reload
```

### MariaDB Production Configuration
```bash
# Create optimized configuration
sudo tee /etc/mysql/mariadb.conf.d/50-server.cnf > /dev/null <<EOF
[mysqld]
# Connection and thread handling
max_connections = 500
thread_cache_size = 100
table_open_cache = 4096
table_definition_cache = 2048

# InnoDB settings
innodb_buffer_pool_size = 4G  # 70-80% of RAM
innodb_log_file_size = 1G
innodb_log_buffer_size = 64M
innodb_file_per_table = 1
innodb_flush_log_at_trx_commit = 2
innodb_flush_method = O_DIRECT
innodb_io_capacity = 2000
innodb_io_capacity_max = 4000
innodb_read_io_threads = 8
innodb_write_io_threads = 8
innodb_open_files = 8192

# Query cache (for read-heavy workloads)
query_cache_type = 1
query_cache_size = 256M
query_cache_limit = 2M

# Temporary tables
tmp_table_size = 64M
max_heap_table_size = 64M

# Binary logging (for replication)
log_bin = mysql-bin
binlog_format = ROW
sync_binlog = 1
expire_logs_days = 7
binlog_cache_size = 1M

# Slow query log
slow_query_log = 1
slow_query_log_file = /var/log/mysql/slow.log
long_query_time = 2
log_queries_not_using_indexes = 1

# Security
bind-address = 127.0.0.1  # Change for network access
skip_name_resolve = 1
local_infile = 0

# SSL configuration
ssl_cert = /etc/mysql/ssl/server-cert.pem
ssl_key = /etc/mysql/ssl/server-key.pem
ssl_ca = /etc/mysql/ssl/ca-cert.pem
require_secure_transport = ON

# Character set
character_set_server = utf8mb4
collation_server = utf8mb4_unicode_ci
EOF

sudo systemctl restart mariadb
```

### MySQL/MariaDB Security Hardening
```bash
# Create dedicated database user with limited privileges
mysql -u root -p <<EOF
-- Create application database
CREATE DATABASE myapp CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

-- Create application user
CREATE USER 'appuser'@'localhost' IDENTIFIED BY 'secure_app_password_2024';
CREATE USER 'appuser'@'192.168.1.%' IDENTIFIED BY 'secure_app_password_2024';

-- Grant minimal privileges
GRANT SELECT, INSERT, UPDATE, DELETE ON myapp.* TO 'appuser'@'localhost';
GRANT SELECT, INSERT, UPDATE, DELETE ON myapp.* TO 'appuser'@'192.168.1.%';

-- Create read-only user for backups
CREATE USER 'backup'@'localhost' IDENTIFIED BY 'backup_password_2024';
GRANT SELECT, LOCK TABLES, SHOW VIEW, EVENT, TRIGGER ON *.* TO 'backup'@'localhost';

-- Create replication user
CREATE USER 'replication'@'%' IDENTIFIED BY 'replication_password_2024';
GRANT REPLICATION SLAVE ON *.* TO 'replication'@'%';

-- Remove default users and databases
DROP DATABASE IF EXISTS test;
DELETE FROM mysql.user WHERE User='';
DELETE FROM mysql.user WHERE User='root' AND Host NOT IN ('localhost', '127.0.0.1', '::1');

-- Secure privileges
FLUSH PRIVILEGES;
EOF

# Generate SSL certificates for MySQL
sudo mkdir -p /etc/mysql/ssl
cd /etc/mysql/ssl

# Create CA certificate
sudo openssl genrsa -out ca-key.pem 4096
sudo openssl req -new -x509 -nodes -days 3650 -key ca-key.pem -out ca-cert.pem -subj "/C=US/ST=State/L=City/O=Organization/CN=MySQL-CA"

# Create server certificate
sudo openssl req -newkey rsa:4096 -days 365 -nodes -keyout server-key.pem -out server-req.pem -subj "/C=US/ST=State/L=City/O=Organization/CN=mysql.example.com"
sudo openssl x509 -req -days 365 -set_serial 01 -in server-req.pem -out server-cert.pem -CA ca-cert.pem -CAkey ca-key.pem

# Create client certificate
sudo openssl req -newkey rsa:4096 -days 365 -nodes -keyout client-key.pem -out client-req.pem -subj "/C=US/ST=State/L=City/O=Organization/CN=mysql-client"
sudo openssl x509 -req -days 365 -set_serial 02 -in client-req.pem -out client-cert.pem -CA ca-cert.pem -CAkey ca-key.pem

# Set permissions
sudo chown mysql:mysql /etc/mysql/ssl/*
sudo chmod 600 /etc/mysql/ssl/*key.pem
sudo chmod 644 /etc/mysql/ssl/*.pem

sudo systemctl restart mariadb
```

## PostgreSQL Installation

### Ubuntu/Debian PostgreSQL Setup
```bash
# Install PostgreSQL official repository
sudo apt install -y wget ca-certificates
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -
echo "deb http://apt.postgresql.org/pub/repos/apt/ $(lsb_release -cs)-pgdg main" | sudo tee /etc/apt/sources.list.d/pgdg.list

# Update and install PostgreSQL 16
sudo apt update
sudo apt install -y postgresql-16 postgresql-client-16 postgresql-contrib-16

# Enable and start service
sudo systemctl enable --now postgresql

# Configure PostgreSQL
sudo -u postgres psql <<EOF
-- Create application database
CREATE DATABASE myapp WITH ENCODING='UTF8' LC_COLLATE='en_US.UTF-8' LC_CTYPE='en_US.UTF-8' TEMPLATE=template0;

-- Create application user
CREATE USER appuser WITH ENCRYPTED PASSWORD 'secure_app_password_2024';
GRANT ALL PRIVILEGES ON DATABASE myapp TO appuser;

-- Create read-only user
CREATE USER readonly WITH ENCRYPTED PASSWORD 'readonly_password_2024';
GRANT CONNECT ON DATABASE myapp TO readonly;
\c myapp
GRANT USAGE ON SCHEMA public TO readonly;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO readonly;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO readonly;

-- Security settings
ALTER SYSTEM SET password_encryption = 'scram-sha-256';
SELECT pg_reload_conf();
EOF
```

### RHEL/CentOS/Rocky Linux PostgreSQL
```bash
# Install PostgreSQL repository
sudo yum install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-9-x86_64/pgdg-redhat-repo-latest.noarch.rpm

# Install PostgreSQL 16
sudo yum install -y postgresql16-server postgresql16 postgresql16-contrib postgresql16-devel

# Initialize database
sudo /usr/pgsql-16/bin/postgresql-16-setup initdb

# Enable and start service
sudo systemctl enable --now postgresql-16

# Configure firewall
sudo firewall-cmd --permanent --add-port=5432/tcp
sudo firewall-cmd --reload
```

### PostgreSQL Production Configuration
```bash
# Configure PostgreSQL for production
sudo tee /var/lib/pgsql/16/data/postgresql.conf > /dev/null <<EOF
# PostgreSQL 16 Production Configuration

# Connection settings
listen_addresses = 'localhost'  # Change to '*' for network access
port = 5432
max_connections = 200
shared_buffers = 2GB  # 25% of RAM
effective_cache_size = 8GB  # 75% of RAM

# Memory settings
work_mem = 16MB
maintenance_work_mem = 512MB
dynamic_shared_memory_type = posix

# WAL settings
wal_level = replica
wal_buffers = 64MB
checkpoint_completion_target = 0.9
max_wal_size = 4GB
min_wal_size = 1GB
checkpoint_timeout = 15min

# Query planner
random_page_cost = 1.1  # For SSD
effective_io_concurrency = 200
max_worker_processes = 8
max_parallel_workers_per_gather = 4
max_parallel_workers = 8
max_parallel_maintenance_workers = 4

# Logging
log_destination = 'stderr'
logging_collector = on
log_directory = 'log'
log_filename = 'postgresql-%Y-%m-%d_%H%M%S.log'
log_rotation_age = 1d
log_rotation_size = 100MB
log_min_duration_statement = 1000  # Log slow queries
log_line_prefix = '%t [%p]: [%l-1] user=%u,db=%d,app=%a,client=%h '
log_statement = 'ddl'  # Log DDL statements
log_lock_waits = on

# SSL configuration
ssl = on
ssl_cert_file = '/var/lib/pgsql/16/data/ssl/server.crt'
ssl_key_file = '/var/lib/pgsql/16/data/ssl/server.key'
ssl_ca_file = '/var/lib/pgsql/16/data/ssl/ca.crt'
ssl_min_protocol_version = 'TLSv1.2'
ssl_prefer_server_ciphers = on

# Security
password_encryption = scram-sha-256
krb_server_keyfile = ''
db_user_namespace = off
row_security = on

# Autovacuum
autovacuum = on
autovacuum_max_workers = 4
autovacuum_naptime = 1min
autovacuum_vacuum_threshold = 50
autovacuum_analyze_threshold = 50
autovacuum_vacuum_scale_factor = 0.1
autovacuum_analyze_scale_factor = 0.05

# Background writer
bgwriter_delay = 200ms
bgwriter_lru_maxpages = 100
bgwriter_lru_multiplier = 2.0
bgwriter_flush_after = 512kB

# Checkpointer
checkpoint_flush_after = 256kB

# Statistics
track_activities = on
track_counts = on
track_io_timing = on
track_functions = all
stats_temp_directory = 'pg_stat_tmp'
EOF

# Configure client authentication
sudo tee /var/lib/pgsql/16/data/pg_hba.conf > /dev/null <<EOF
# PostgreSQL Client Authentication Configuration

# TYPE  DATABASE        USER            ADDRESS                 METHOD

# Local connections
local   all             postgres                                peer
local   all             all                                     scram-sha-256

# IPv4 local connections
host    all             all             127.0.0.1/32            scram-sha-256

# IPv6 local connections  
host    all             all             ::1/128                 scram-sha-256

# Network connections (if needed)
hostssl myapp           appuser         192.168.1.0/24          scram-sha-256
hostssl myapp           readonly        192.168.1.0/24          scram-sha-256

# Replication connections
hostssl replication     replication     192.168.1.0/24          scram-sha-256

# Deny all other connections
host    all             all             0.0.0.0/0               reject
EOF

# Generate SSL certificates
sudo mkdir -p /var/lib/pgsql/16/data/ssl
cd /var/lib/pgsql/16/data/ssl

sudo openssl genrsa -out ca.key 4096
sudo openssl req -new -x509 -days 3650 -key ca.key -out ca.crt -subj "/C=US/ST=State/L=City/O=Organization/CN=PostgreSQL-CA"

sudo openssl genrsa -out server.key 4096
sudo openssl req -new -key server.key -out server.csr -subj "/C=US/ST=State/L=City/O=Organization/CN=postgres.example.com"
sudo openssl x509 -req -days 365 -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out server.crt

sudo chown postgres:postgres /var/lib/pgsql/16/data/ssl/*
sudo chmod 600 /var/lib/pgsql/16/data/ssl/*.key
sudo chmod 644 /var/lib/pgsql/16/data/ssl/*.crt

sudo systemctl restart postgresql-16
```

### PostgreSQL Security Hardening
```bash
# Advanced security configuration
sudo -u postgres psql <<EOF
-- Enable row-level security
ALTER SYSTEM SET row_security = on;

-- Configure logging for security
ALTER SYSTEM SET log_statement = 'all';
ALTER SYSTEM SET log_connections = on;
ALTER SYSTEM SET log_disconnections = on;
ALTER SYSTEM SET log_lock_waits = on;
ALTER SYSTEM SET log_temp_files = 0;

-- Password policies
ALTER SYSTEM SET password_encryption = 'scram-sha-256';

-- Create roles with specific privileges
CREATE ROLE app_read;
GRANT CONNECT ON DATABASE myapp TO app_read;
GRANT USAGE ON SCHEMA public TO app_read;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO app_read;

CREATE ROLE app_write;
GRANT app_read TO app_write;
GRANT INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO app_write;

-- Create application-specific user
CREATE USER myapp_user WITH PASSWORD 'secure_password_2024';
GRANT app_write TO myapp_user;

-- Security functions
CREATE OR REPLACE FUNCTION audit_trigger_function()
RETURNS TRIGGER AS \$\$
BEGIN
    INSERT INTO audit_log (table_name, operation, old_values, new_values, user_name, timestamp)
    VALUES (TG_TABLE_NAME, TG_OP, row_to_json(OLD), row_to_json(NEW), current_user, now());
    RETURN COALESCE(NEW, OLD);
END;
\$\$ LANGUAGE plpgsql;

-- Reload configuration
SELECT pg_reload_conf();
EOF

# Configure connection limits
sudo tee -a /var/lib/pgsql/16/data/postgresql.conf > /dev/null <<EOF

# Connection limiting per user/database
# ALTER USER myapp_user CONNECTION LIMIT 50;
# ALTER DATABASE myapp CONNECTION LIMIT 100;
EOF
```

## MongoDB Installation

### Ubuntu/Debian MongoDB Setup
```bash
# Import MongoDB public GPG key
wget -qO - https://www.mongodb.org/static/pgp/server-7.0.asc | sudo apt-key add -

# Add MongoDB repository
echo "deb [ arch=amd64,arm64 ] https://repo.mongodb.org/apt/ubuntu $(lsb_release -cs)/mongodb-org/7.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-7.0.list

# Update and install MongoDB
sudo apt update
sudo apt install -y mongodb-org

# Enable and start service
sudo systemctl enable --now mongod

# Verify installation
mongosh --eval 'db.runCommand("connectionStatus")'
```

### RHEL/CentOS/Rocky Linux MongoDB
```bash
# Add MongoDB repository
sudo tee /etc/yum.repos.d/mongodb-org-7.0.repo > /dev/null <<EOF
[mongodb-org-7.0]
name=MongoDB Repository
baseurl=https://repo.mongodb.org/yum/redhat/\$releasever/mongodb-org/7.0/\$basearch/
gpgcheck=1
enabled=1
gpgkey=https://www.mongodb.org/static/pgp/server-7.0.asc
EOF

# Install MongoDB
sudo yum install -y mongodb-org

# Enable and start service
sudo systemctl enable --now mongod

# Configure firewall
sudo firewall-cmd --permanent --add-port=27017/tcp
sudo firewall-cmd --reload
```

### MongoDB Production Configuration
```bash
# Create secure MongoDB configuration
sudo tee /etc/mongod.conf > /dev/null <<EOF
# MongoDB Production Configuration

storage:
  dbPath: /var/lib/mongo
  journal:
    enabled: true
  wiredTiger:
    engineConfig:
      cacheSizeGB: 4  # 50% of RAM
      journalCompressor: snappy
      directoryForIndexes: false
    collectionConfig:
      blockCompressor: snappy
    indexConfig:
      prefixCompression: true

systemLog:
  destination: file
  logAppend: true
  path: /var/log/mongodb/mongod.log
  quiet: false
  logRotate: reopen
  component:
    accessControl:
      verbosity: 1
    command:
      verbosity: 1

net:
  port: 27017
  bindIp: 127.0.0.1  # Change for network access
  maxIncomingConnections: 1000
  compression:
    compressors: snappy,zstd
  ssl:
    mode: requireSSL
    PEMKeyFile: /etc/ssl/mongodb/mongodb.pem
    CAFile: /etc/ssl/mongodb/ca.pem
    allowInvalidHostnames: false
    allowInvalidCertificates: false

security:
  authorization: enabled
  keyFile: /etc/mongodb/mongodb-keyfile
  clusterAuthMode: x509
  javascriptEnabled: false

operationProfiling:
  mode: slowOp
  slowOpThresholdMs: 100
  slowOpSampleRate: 0.02

replication:
  replSetName: rs0
  enableMajorityReadConcern: true

sharding:
  clusterRole: shardsvr  # or configsvr for config servers

processManagement:
  fork: true
  pidFilePath: /var/run/mongodb/mongod.pid
  timeZoneInfo: /usr/share/zoneinfo

setParameter:
  authenticationMechanisms: SCRAM-SHA-1,SCRAM-SHA-256
  scramIterationCount: 15000
  failIndexKeyTooLong: false
  notablescan: 1  # Disable table scans in production
EOF

# Create MongoDB keyfile for replica set authentication
sudo openssl rand -base64 756 | sudo tee /etc/mongodb/mongodb-keyfile
sudo chmod 600 /etc/mongodb/mongodb-keyfile
sudo chown mongod:mongod /etc/mongodb/mongodb-keyfile

# Generate SSL certificates for MongoDB
sudo mkdir -p /etc/ssl/mongodb
cd /etc/ssl/mongodb

sudo openssl genrsa -out ca.key 4096
sudo openssl req -new -x509 -days 3650 -key ca.key -out ca.pem -subj "/C=US/ST=State/L=City/O=Organization/CN=MongoDB-CA"

sudo openssl genrsa -out mongodb.key 4096
sudo openssl req -new -key mongodb.key -out mongodb.csr -subj "/C=US/ST=State/L=City/O=Organization/CN=mongodb.example.com"
sudo openssl x509 -req -days 365 -in mongodb.csr -CA ca.pem -CAkey ca.key -CAcreateserial -out mongodb.crt

# Combine certificate and key for MongoDB
sudo cat mongodb.crt mongodb.key | sudo tee mongodb.pem
sudo chmod 600 /etc/ssl/mongodb/*.key /etc/ssl/mongodb/*.pem
sudo chown mongod:mongod /etc/ssl/mongodb/*

sudo systemctl restart mongod
```

### MongoDB Security Setup
```bash
# Initialize MongoDB security
mongosh --ssl --sslPEMKeyFile /etc/ssl/mongodb/mongodb.pem --sslCAFile /etc/ssl/mongodb/ca.pem <<EOF
// Create admin user
use admin
db.createUser({
  user: "admin",
  pwd: "secure_admin_password_2024",
  roles: [
    { role: "userAdminAnyDatabase", db: "admin" },
    { role: "readWriteAnyDatabase", db: "admin" },
    { role: "dbAdminAnyDatabase", db: "admin" },
    { role: "clusterAdmin", db: "admin" }
  ]
})

// Create application user
use myapp
db.createUser({
  user: "appuser",
  pwd: "secure_app_password_2024",
  roles: [
    { role: "readWrite", db: "myapp" }
  ]
})

// Create read-only user
db.createUser({
  user: "readonly",
  pwd: "readonly_password_2024",
  roles: [
    { role: "read", db: "myapp" }
  ]
})

// Create backup user
use admin
db.createUser({
  user: "backup",
  pwd: "backup_password_2024",
  roles: [
    { role: "backup", db: "admin" },
    { role: "restore", db: "admin" }
  ]
})

// Initialize replica set (if using replication)
rs.initiate({
  _id: "rs0",
  members: [
    { _id: 0, host: "mongodb1.example.com:27017", priority: 2 },
    { _id: 1, host: "mongodb2.example.com:27017", priority: 1 },
    { _id: 2, host: "mongodb3.example.com:27017", priority: 1, arbiterOnly: true }
  ]
})
EOF
```

## Redis Installation and Configuration

### Redis Setup (All Distributions)
```bash
# Ubuntu/Debian
sudo apt install -y redis-server redis-tools

# RHEL/CentOS
sudo yum install -y redis

# Fedora
sudo dnf install -y redis

# Arch Linux
sudo pacman -S redis

# Configure Redis for production
sudo tee /etc/redis/redis.conf > /dev/null <<EOF
# Redis Production Configuration

# Network
bind 127.0.0.1 ::1  # Change for network access
port 6379
tcp-backlog 511
timeout 300
tcp-keepalive 300

# Security
requirepass redis_secure_password_2024
rename-command FLUSHDB "FLUSHDB_9a8b7c6d5e4f3g2h1"
rename-command FLUSHALL "FLUSHALL_h1g2f3e4d5c6b7a8"
rename-command DEBUG "DEBUG_8a7b6c5d4e3f2g1h"
rename-command CONFIG "CONFIG_1h2g3f4e5d6c7b8a"

# SSL/TLS (Redis 6.0+)
tls-port 6380
port 0  # Disable non-TLS port
tls-cert-file /etc/redis/tls/redis.crt
tls-key-file /etc/redis/tls/redis.key
tls-ca-cert-file /etc/redis/tls/ca.crt
tls-protocols "TLSv1.2 TLSv1.3"
tls-prefer-server-ciphers yes

# Memory management
maxmemory 2gb
maxmemory-policy allkeys-lru
maxmemory-samples 5

# Persistence
save 900 1
save 300 10
save 60 10000
rdbcompression yes
rdbchecksum yes
dbfilename redis.rdb
dir /var/lib/redis/

# AOF (Append Only File)
appendonly yes
appendfilename "appendonly.aof"
appendfsync everysec
no-appendfsync-on-rewrite no
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb

# Logging
loglevel notice
logfile /var/log/redis/redis-server.log
syslog-enabled yes
syslog-ident redis

# Slow log
slowlog-log-slower-than 10000
slowlog-max-len 128

# Latency monitoring
latency-monitor-threshold 100

# Client output buffer limits
client-output-buffer-limit normal 0 0 0
client-output-buffer-limit replica 256mb 64mb 60
client-output-buffer-limit pubsub 32mb 8mb 60

# Advanced configuration
hz 10
dynamic-hz yes
aof-rewrite-incremental-fsync yes
rdb-save-incremental-fsync yes

# Lua scripting
lua-time-limit 5000
EOF

# Generate Redis TLS certificates
sudo mkdir -p /etc/redis/tls
cd /etc/redis/tls

sudo openssl genrsa -out ca.key 4096
sudo openssl req -new -x509 -days 3650 -key ca.key -out ca.crt -subj "/C=US/ST=State/L=City/O=Organization/CN=Redis-CA"

sudo openssl genrsa -out redis.key 4096
sudo openssl req -new -key redis.key -out redis.csr -subj "/C=US/ST=State/L=City/O=Organization/CN=redis.example.com"
sudo openssl x509 -req -days 365 -in redis.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out redis.crt

sudo chown redis:redis /etc/redis/tls/*
sudo chmod 600 /etc/redis/tls/*.key
sudo chmod 644 /etc/redis/tls/*.crt

sudo systemctl restart redis-server
```

## Database Monitoring and Maintenance

### Comprehensive Database Monitoring
```bash
sudo tee /usr/local/bin/database-monitor.sh > /dev/null <<'EOF'
#!/bin/bash
MONITOR_LOG="/var/log/database-monitor.log"

log_message() {
    echo "$(date '+%Y-%m-%d %H:%M:%S') - $1" | tee -a ${MONITOR_LOG}
}

# MySQL/MariaDB monitoring
if command -v mysql >/dev/null 2>&1 && systemctl is-active mariadb >/dev/null 2>&1; then
    log_message "=== MySQL/MariaDB Monitoring ==="
    
    # Connection count
    MYSQL_CONNECTIONS=$(mysql -e "SHOW STATUS LIKE 'Threads_connected';" | tail -1 | awk '{print $2}')
    MYSQL_MAX_CONNECTIONS=$(mysql -e "SHOW VARIABLES LIKE 'max_connections';" | tail -1 | awk '{print $2}')
    log_message "MySQL connections: ${MYSQL_CONNECTIONS}/${MYSQL_MAX_CONNECTIONS}"
    
    # Query performance
    SLOW_QUERIES=$(mysql -e "SHOW STATUS LIKE 'Slow_queries';" | tail -1 | awk '{print $2}')
    log_message "MySQL slow queries: ${SLOW_QUERIES}"
    
    # Buffer pool hit ratio
    BUFFER_HIT_RATIO=$(mysql -e "
      SELECT ROUND((1 - (Innodb_buffer_pool_reads / Innodb_buffer_pool_read_requests)) * 100, 2) as hit_ratio
      FROM INFORMATION_SCHEMA.GLOBAL_STATUS 
      WHERE VARIABLE_NAME IN ('Innodb_buffer_pool_reads', 'Innodb_buffer_pool_read_requests');" | tail -1)
    log_message "InnoDB buffer pool hit ratio: ${BUFFER_HIT_RATIO}%"
fi

# PostgreSQL monitoring
if command -v psql >/dev/null 2>&1 && systemctl is-active postgresql-16 >/dev/null 2>&1; then
    log_message "=== PostgreSQL Monitoring ==="
    
    # Connection count
    PG_CONNECTIONS=$(sudo -u postgres psql -t -c "SELECT count(*) FROM pg_stat_activity;")
    PG_MAX_CONNECTIONS=$(sudo -u postgres psql -t -c "SHOW max_connections;")
    log_message "PostgreSQL connections: ${PG_CONNECTIONS}/${PG_MAX_CONNECTIONS}"
    
    # Database size
    PG_DB_SIZE=$(sudo -u postgres psql -t -c "SELECT pg_size_pretty(pg_database_size('myapp'));")
    log_message "PostgreSQL database size: ${PG_DB_SIZE}"
    
    # Cache hit ratio
    PG_CACHE_HIT=$(sudo -u postgres psql -t -c "
      SELECT round(sum(blks_hit)*100.0/sum(blks_hit+blks_read), 2) 
      FROM pg_stat_database WHERE datname='myapp';")
    log_message "PostgreSQL cache hit ratio: ${PG_CACHE_HIT}%"
    
    # Long running queries
    LONG_QUERIES=$(sudo -u postgres psql -t -c "
      SELECT count(*) FROM pg_stat_activity 
      WHERE state = 'active' AND now() - query_start > interval '5 minutes';")
    log_message "PostgreSQL long-running queries: ${LONG_QUERIES}"
fi

# MongoDB monitoring
if command -v mongosh >/dev/null 2>&1 && systemctl is-active mongod >/dev/null 2>&1; then
    log_message "=== MongoDB Monitoring ==="
    
    # Connection count
    MONGO_CONNECTIONS=$(mongosh --quiet --eval "db.serverStatus().connections.current")
    MONGO_MAX_CONNECTIONS=$(mongosh --quiet --eval "db.serverStatus().connections.available")
    log_message "MongoDB connections: ${MONGO_CONNECTIONS}/${MONGO_MAX_CONNECTIONS}"
    
    # Database statistics
    MONGO_DB_SIZE=$(mongosh myapp --quiet --eval "Math.round(db.stats().dataSize / 1024 / 1024) + ' MB'")
    log_message "MongoDB database size: ${MONGO_DB_SIZE}"
    
    # OpLog status (for replica sets)
    if mongosh admin --quiet --eval "rs.status().ok" 2>/dev/null | grep -q "1"; then
        OPLOG_SIZE=$(mongosh local --quiet --eval "Math.round(db.oplog.rs.stats().maxSize / 1024 / 1024) + ' MB'")
        log_message "MongoDB OpLog size: ${OPLOG_SIZE}"
    fi
fi

# Redis monitoring
if command -v redis-cli >/dev/null 2>&1 && systemctl is-active redis >/dev/null 2>&1; then
    log_message "=== Redis Monitoring ==="
    
    # Memory usage
    REDIS_MEMORY=$(redis-cli info memory | grep used_memory_human: | cut -d: -f2)
    REDIS_MAX_MEMORY=$(redis-cli config get maxmemory | tail -1)
    log_message "Redis memory usage: ${REDIS_MEMORY} / ${REDIS_MAX_MEMORY}"
    
    # Connected clients
    REDIS_CLIENTS=$(redis-cli info clients | grep connected_clients: | cut -d: -f2)
    log_message "Redis connected clients: ${REDIS_CLIENTS}"
    
    # Hit ratio
    REDIS_HITS=$(redis-cli info stats | grep keyspace_hits: | cut -d: -f2)
    REDIS_MISSES=$(redis-cli info stats | grep keyspace_misses: | cut -d: -f2)
    if [ ${REDIS_MISSES} -gt 0 ]; then
        REDIS_HIT_RATIO=$(echo "scale=2; ${REDIS_HITS} / (${REDIS_HITS} + ${REDIS_MISSES}) * 100" | bc)
        log_message "Redis hit ratio: ${REDIS_HIT_RATIO}%"
    fi
fi

log_message "Database monitoring completed"
EOF

sudo chmod +x /usr/local/bin/database-monitor.sh

# Schedule monitoring every 5 minutes
echo "*/5 * * * * root /usr/local/bin/database-monitor.sh" | sudo tee -a /etc/crontab
```

### Database Backup Automation
```bash
sudo tee /usr/local/bin/database-backup.sh > /dev/null <<'EOF'
#!/bin/bash
BACKUP_DIR="/backup/databases"
DATE=$(date +%Y%m%d_%H%M%S)

mkdir -p ${BACKUP_DIR}/{mysql,postgresql,mongodb,redis}

# MySQL/MariaDB backup
if command -v mysql >/dev/null 2>&1 && systemctl is-active mariadb >/dev/null 2>&1; then
    echo "Backing up MySQL/MariaDB..."
    
    # Full backup with all databases
    mysqldump --all-databases --single-transaction --routines --triggers --events \
      --master-data=2 --flush-logs --delete-master-logs \
      > ${BACKUP_DIR}/mysql/full-backup-${DATE}.sql
    
    # Compress backup
    gzip ${BACKUP_DIR}/mysql/full-backup-${DATE}.sql
    
    # Individual database backup
    mysqldump --single-transaction --routines --triggers myapp \
      > ${BACKUP_DIR}/mysql/myapp-backup-${DATE}.sql
    gzip ${BACKUP_DIR}/mysql/myapp-backup-${DATE}.sql
fi

# PostgreSQL backup
if command -v pg_dump >/dev/null 2>&1 && systemctl is-active postgresql-16 >/dev/null 2>&1; then
    echo "Backing up PostgreSQL..."
    
    # Full cluster backup
    sudo -u postgres pg_dumpall > ${BACKUP_DIR}/postgresql/cluster-backup-${DATE}.sql
    
    # Individual database backup
    sudo -u postgres pg_dump -Fc myapp > ${BACKUP_DIR}/postgresql/myapp-backup-${DATE}.dump
    
    # Compress SQL backup
    gzip ${BACKUP_DIR}/postgresql/cluster-backup-${DATE}.sql
fi

# MongoDB backup
if command -v mongodump >/dev/null 2>&1 && systemctl is-active mongod >/dev/null 2>&1; then
    echo "Backing up MongoDB..."
    
    # Full backup
    mongodump --host localhost:27017 --ssl \
      --sslPEMKeyFile /etc/ssl/mongodb/mongodb.pem \
      --sslCAFile /etc/ssl/mongodb/ca.pem \
      --out ${BACKUP_DIR}/mongodb/full-backup-${DATE}
    
    # Individual database backup
    mongodump --host localhost:27017 --ssl \
      --sslPEMKeyFile /etc/ssl/mongodb/mongodb.pem \
      --sslCAFile /etc/ssl/mongodb/ca.pem \
      --db myapp --out ${BACKUP_DIR}/mongodb/myapp-backup-${DATE}
    
    # Compress backups
    tar -czf ${BACKUP_DIR}/mongodb/full-backup-${DATE}.tar.gz -C ${BACKUP_DIR}/mongodb full-backup-${DATE}
    tar -czf ${BACKUP_DIR}/mongodb/myapp-backup-${DATE}.tar.gz -C ${BACKUP_DIR}/mongodb myapp-backup-${DATE}
    
    # Remove uncompressed directories
    rm -rf ${BACKUP_DIR}/mongodb/full-backup-${DATE} ${BACKUP_DIR}/mongodb/myapp-backup-${DATE}
fi

# Redis backup
if command -v redis-cli >/dev/null 2>&1 && systemctl is-active redis >/dev/null 2>&1; then
    echo "Backing up Redis..."
    
    # Trigger background save
    redis-cli BGSAVE
    
    # Wait for save to complete
    while [ "$(redis-cli LASTSAVE)" = "$(redis-cli LASTSAVE)" ]; do
        sleep 1
    done
    
    # Copy RDB file
    cp /var/lib/redis/dump.rdb ${BACKUP_DIR}/redis/redis-backup-${DATE}.rdb
    gzip ${BACKUP_DIR}/redis/redis-backup-${DATE}.rdb
fi

# Upload to cloud storage
aws s3 cp ${BACKUP_DIR}/ s3://database-backups/ --recursive
az storage blob upload-batch --source ${BACKUP_DIR} --destination database-backups
gsutil cp -r ${BACKUP_DIR}/* gs://database-backups/

# Keep only last 14 days of backups
find ${BACKUP_DIR} -name "*backup*" -type f -mtime +14 -delete

# Verify backup integrity
echo "Verifying backup integrity..."
for backup in ${BACKUP_DIR}/*/*.gz; do
    if gzip -t "$backup" 2>/dev/null; then
        echo "✓ $(basename $backup) - OK"
    else
        echo "✗ $(basename $backup) - CORRUPTED"
    fi
done

echo "Database backup completed: ${DATE}"
EOF

sudo chmod +x /usr/local/bin/database-backup.sh

# Schedule daily backups
echo "0 1 * * * root /usr/local/bin/database-backup.sh" | sudo tee -a /etc/crontab
```

## High Availability and Replication

### MySQL/MariaDB Master-Slave Replication
```bash
# Configure master server
sudo tee -a /etc/mysql/mariadb.conf.d/replication.cnf > /dev/null <<EOF
[mysqld]
# Replication settings
server-id = 1
log_bin = mysql-bin
binlog_format = ROW
binlog_do_db = myapp
sync_binlog = 1
relay-log = mysql-relay-bin
relay-log-recovery = 1

# GTID replication (recommended)
gtid_mode = ON
enforce_gtid_consistency = ON
log_slave_updates = ON
EOF

mysql -u root -p <<EOF
-- Create replication user
CREATE USER 'replication'@'%' IDENTIFIED BY 'replication_password_2024';
GRANT REPLICATION SLAVE ON *.* TO 'replication'@'%';
FLUSH PRIVILEGES;

-- Get master status
SHOW MASTER STATUS;
EOF

# Configure slave server (server-id = 2)
# On slave server:
mysql -u root -p <<EOF
CHANGE MASTER TO
  MASTER_HOST='mysql-master.example.com',
  MASTER_USER='replication',
  MASTER_PASSWORD='replication_password_2024',
  MASTER_AUTO_POSITION=1;

START SLAVE;
SHOW SLAVE STATUS\G
EOF
```

### PostgreSQL Streaming Replication
```bash
# Configure master server
sudo tee -a /var/lib/pgsql/16/data/postgresql.conf > /dev/null <<EOF
# Replication settings
wal_level = replica
max_wal_senders = 10
max_replication_slots = 10
wal_keep_size = 1GB
hot_standby = on
archive_mode = on
archive_command = 'cp %p /var/lib/pgsql/16/archive/%f'
EOF

# Configure replication access
sudo tee -a /var/lib/pgsql/16/data/pg_hba.conf > /dev/null <<EOF
# Replication connections
hostssl replication replication 192.168.1.0/24 scram-sha-256
EOF

# Create replication user
sudo -u postgres psql <<EOF
CREATE USER replication WITH REPLICATION ENCRYPTED PASSWORD 'replication_password_2024';
EOF

# Create archive directory
sudo mkdir -p /var/lib/pgsql/16/archive
sudo chown postgres:postgres /var/lib/pgsql/16/archive

sudo systemctl restart postgresql-16

# Setup slave server
# On slave server, create base backup:
sudo -u postgres pg_basebackup -h master.example.com -D /var/lib/pgsql/16/data -U replication -W -v -P -R
```

### MongoDB Replica Set Configuration
```bash
# Initialize replica set (run on primary node)
mongosh admin <<EOF
rs.initiate({
  _id: "rs0",
  version: 1,
  protocolVersion: 1,
  members: [
    { 
      _id: 0, 
      host: "mongodb1.example.com:27017",
      priority: 2,
      votes: 1
    },
    { 
      _id: 1, 
      host: "mongodb2.example.com:27017",
      priority: 1,
      votes: 1
    },
    { 
      _id: 2, 
      host: "mongodb3.example.com:27017",
      priority: 1,
      votes: 1,
      arbiterOnly: true
    }
  ],
  settings: {
    chainingAllowed: false,
    heartbeatIntervalMillis: 2000,
    heartbeatTimeoutSecs: 10,
    electionTimeoutMillis: 10000,
    catchUpTimeoutMillis: -1,
    getLastErrorModes: {
      majority: { 
        tags: { 
          dc: 1 
        } 
      }
    }
  }
})

// Check replica set status
rs.status()

// Configure read preferences
rs.conf()
EOF

# Configure MongoDB sharding (for large deployments)
# Config server initialization:
mongosh admin <<EOF
rs.initiate({
  _id: "configReplSet",
  configsvr: true,
  members: [
    { _id: 0, host: "config1.example.com:27019" },
    { _id: 1, host: "config2.example.com:27019" },
    { _id: 2, host: "config3.example.com:27019" }
  ]
})
EOF
```

### Redis Sentinel High Availability
```bash
# Configure Redis Sentinel for HA
sudo tee /etc/redis/sentinel.conf > /dev/null <<EOF
# Redis Sentinel Configuration
port 26379
sentinel deny-scripts-reconfig yes

# Monitor Redis master
sentinel monitor mymaster redis-master.example.com 6379 2
sentinel auth-pass mymaster redis_secure_password_2024
sentinel down-after-milliseconds mymaster 5000
sentinel parallel-syncs mymaster 1
sentinel failover-timeout mymaster 10000

# Notification scripts
sentinel notification-script mymaster /etc/redis/notify.sh
sentinel client-reconfig-script mymaster /etc/redis/reconfig.sh

# Security
requirepass sentinel_password_2024
EOF

# Create notification script
sudo tee /etc/redis/notify.sh > /dev/null <<'EOF'
#!/bin/bash
echo "$(date): Redis failover event: $*" >> /var/log/redis/sentinel.log
# Add alerting logic here (email, Slack, etc.)
EOF

# Create reconfiguration script
sudo tee /etc/redis/reconfig.sh > /dev/null <<'EOF'
#!/bin/bash
echo "$(date): Redis master changed to: $6:$7" >> /var/log/redis/sentinel.log
# Update application configuration, restart services, etc.
EOF

sudo chmod +x /etc/redis/{notify,reconfig}.sh
sudo systemctl enable --now redis-sentinel
```

## Performance Optimization

### Database Performance Tuning
```bash
sudo tee /usr/local/bin/database-performance-tune.sh > /dev/null <<'EOF'
#!/bin/bash

tune_mysql() {
    echo "Tuning MySQL/MariaDB performance..."
    
    # Calculate optimal buffer pool size (70% of RAM)
    TOTAL_RAM=$(free -b | awk 'NR==2{print $2}')
    BUFFER_POOL_SIZE=$((TOTAL_RAM * 70 / 100))
    
    mysql -u root -p <<EOF
-- Performance tuning
SET GLOBAL innodb_buffer_pool_size = ${BUFFER_POOL_SIZE};
SET GLOBAL query_cache_size = $((TOTAL_RAM * 5 / 100));
SET GLOBAL thread_cache_size = 100;
SET GLOBAL table_open_cache = 4096;
SET GLOBAL innodb_io_capacity = 2000;

-- Show current configuration
SHOW VARIABLES LIKE 'innodb_buffer_pool_size';
SHOW VARIABLES LIKE 'query_cache_size';
EOF
}

tune_postgresql() {
    echo "Tuning PostgreSQL performance..."
    
    # Use pg_tune recommendations
    TOTAL_RAM_MB=$(($(free -m | awk 'NR==2{print $2}')))
    SHARED_BUFFERS=$((TOTAL_RAM_MB / 4))
    EFFECTIVE_CACHE=$((TOTAL_RAM_MB * 3 / 4))
    
    sudo -u postgres psql <<EOF
-- Performance tuning
ALTER SYSTEM SET shared_buffers = '${SHARED_BUFFERS}MB';
ALTER SYSTEM SET effective_cache_size = '${EFFECTIVE_CACHE}MB';
ALTER SYSTEM SET work_mem = '16MB';
ALTER SYSTEM SET maintenance_work_mem = '256MB';
ALTER SYSTEM SET random_page_cost = 1.1;
ALTER SYSTEM SET effective_io_concurrency = 200;

-- Reload configuration
SELECT pg_reload_conf();

-- Show current settings
SHOW shared_buffers;
SHOW effective_cache_size;
EOF
}

tune_mongodb() {
    echo "Tuning MongoDB performance..."
    
    # Calculate WiredTiger cache size (50% of RAM)
    TOTAL_RAM_GB=$(($(free -g | awk 'NR==2{print $2}')))
    CACHE_SIZE_GB=$((TOTAL_RAM_GB / 2))
    
    mongosh admin <<EOF
// Performance tuning
db.adminCommand({
  "setParameter": 1,
  "wiredTigerEngineRuntimeConfig": "cache_size=${CACHE_SIZE_GB}GB"
})

// Show current cache usage
db.serverStatus().wiredTiger.cache
EOF
}

tune_redis() {
    echo "Tuning Redis performance..."
    
    # Calculate maxmemory (50% of RAM for dedicated Redis server)
    TOTAL_RAM_BYTES=$(free -b | awk 'NR==2{print $2}')
    MAX_MEMORY_BYTES=$((TOTAL_RAM_BYTES / 2))
    
    redis-cli CONFIG SET maxmemory ${MAX_MEMORY_BYTES}
    redis-cli CONFIG SET maxmemory-policy allkeys-lru
    redis-cli CONFIG REWRITE
    
    echo "Redis maxmemory set to $(redis-cli CONFIG GET maxmemory | tail -1) bytes"
}

# Check which databases are running and tune them
if systemctl is-active mariadb >/dev/null 2>&1; then
    tune_mysql
fi

if systemctl is-active postgresql-16 >/dev/null 2>&1; then
    tune_postgresql
fi

if systemctl is-active mongod >/dev/null 2>&1; then
    tune_mongodb
fi

if systemctl is-active redis >/dev/null 2>&1; then
    tune_redis
fi

echo "Database performance tuning completed"
EOF

sudo chmod +x /usr/local/bin/database-performance-tune.sh
```

## Verification and Testing

### Database Health Checks
```bash
sudo tee /usr/local/bin/database-health-check.sh > /dev/null <<'EOF'
#!/bin/bash
HEALTH_LOG="/var/log/database-health.log"

log_message() {
    echo "$(date '+%Y-%m-%d %H:%M:%S') - $1" | tee -a ${HEALTH_LOG}
}

# MySQL/MariaDB health check
if command -v mysql >/dev/null 2>&1; then
    if systemctl is-active mariadb >/dev/null 2>&1; then
        log_message "✓ MariaDB service is running"
        
        # Test connectivity
        if mysql -e "SELECT 1;" >/dev/null 2>&1; then
            log_message "✓ MariaDB connectivity test passed"
        else
            log_message "✗ MariaDB connectivity test failed"
        fi
        
        # Check for errors in log
        ERROR_COUNT=$(tail -100 /var/log/mysql/error.log | grep -i error | wc -l)
        log_message "ℹ MariaDB errors in last 100 log lines: ${ERROR_COUNT}"
    else
        log_message "✗ MariaDB service is not running"
    fi
fi

# PostgreSQL health check
if command -v psql >/dev/null 2>&1; then
    if systemctl is-active postgresql-16 >/dev/null 2>&1; then
        log_message "✓ PostgreSQL service is running"
        
        # Test connectivity
        if sudo -u postgres psql -c "SELECT version();" >/dev/null 2>&1; then
            log_message "✓ PostgreSQL connectivity test passed"
        else
            log_message "✗ PostgreSQL connectivity test failed"
        fi
        
        # Check replication lag (if slave)
        if sudo -u postgres psql -t -c "SELECT pg_is_in_recovery();" | grep -q "t"; then
            LAG=$(sudo -u postgres psql -t -c "SELECT EXTRACT(EPOCH FROM (now() - pg_last_xact_replay_timestamp()))::int;")
            log_message "ℹ PostgreSQL replication lag: ${LAG} seconds"
        fi
    else
        log_message "✗ PostgreSQL service is not running"
    fi
fi

# MongoDB health check
if command -v mongosh >/dev/null 2>&1; then
    if systemctl is-active mongod >/dev/null 2>&1; then
        log_message "✓ MongoDB service is running"
        
        # Test connectivity
        if mongosh --quiet --eval "db.adminCommand('ping').ok" 2>/dev/null | grep -q "1"; then
            log_message "✓ MongoDB connectivity test passed"
        else
            log_message "✗ MongoDB connectivity test failed"
        fi
        
        # Check replica set status
        if mongosh admin --quiet --eval "rs.status().ok" 2>/dev/null | grep -q "1"; then
            PRIMARY_COUNT=$(mongosh admin --quiet --eval "rs.status().members.filter(m => m.stateStr === 'PRIMARY').length")
            log_message "ℹ MongoDB replica set has ${PRIMARY_COUNT} primary node(s)"
        fi
    else
        log_message "✗ MongoDB service is not running"
    fi
fi

# Redis health check
if command -v redis-cli >/dev/null 2>&1; then
    if systemctl is-active redis >/dev/null 2>&1; then
        log_message "✓ Redis service is running"
        
        # Test connectivity
        if redis-cli ping | grep -q "PONG"; then
            log_message "✓ Redis connectivity test passed"
        else
            log_message "✗ Redis connectivity test failed"
        fi
        
        # Check memory usage
        REDIS_MEMORY_PERCENT=$(redis-cli info memory | grep used_memory_rss_human: | cut -d: -f2)
        log_message "ℹ Redis memory usage: ${REDIS_MEMORY_PERCENT}"
    else
        log_message "✗ Redis service is not running"
    fi
fi

log_message "Database health check completed"
EOF

sudo chmod +x /usr/local/bin/database-health-check.sh

# Schedule health checks every 10 minutes
echo "*/10 * * * * root /usr/local/bin/database-health-check.sh" | sudo tee -a /etc/crontab
```

## 6. Troubleshooting (Cross-Platform)

### Common Database Issues
```bash
# MySQL/MariaDB troubleshooting
# Check service status
sudo systemctl status mariadb

# Check error logs
sudo tail -f /var/log/mysql/error.log

# Test connectivity
mysql -u root -p -e "SELECT version();"

# Check process list
mysql -u root -p -e "SHOW FULL PROCESSLIST;"

# Check locks
mysql -u root -p -e "SHOW ENGINE INNODB STATUS\G" | grep -A 20 "LATEST DETECTED DEADLOCK"

# Repair tables
mysqlcheck --all-databases --repair -u root -p

# PostgreSQL troubleshooting
# Check service status
sudo systemctl status postgresql-16

# Check logs
sudo tail -f /var/lib/pgsql/16/data/log/postgresql-*.log

# Test connectivity
sudo -u postgres psql -c "SELECT version();"

# Check active connections
sudo -u postgres psql -c "SELECT count(*) FROM pg_stat_activity WHERE state = 'active';"

# Check locks
sudo -u postgres psql -c "SELECT * FROM pg_locks WHERE NOT granted;"

# Vacuum and analyze
sudo -u postgres vacuumdb --all --analyze --verbose

# MongoDB troubleshooting
# Check service status
sudo systemctl status mongod

# Check logs
sudo tail -f /var/log/mongodb/mongod.log

# Test connectivity
mongosh --eval "db.adminCommand('ping')"

# Check replica set status
mongosh admin --eval "rs.status()"

# Check database profiler
mongosh myapp --eval "db.getProfilingStatus()"

# Repair database
mongosh myapp --eval "db.repairDatabase()"

# Redis troubleshooting
# Check service status
sudo systemctl status redis

# Check logs
sudo tail -f /var/log/redis/redis-server.log

# Test connectivity
redis-cli ping

# Check memory stats
redis-cli info memory

# Check slow log
redis-cli slowlog get 10

# Monitor commands
redis-cli monitor
```

### Advanced Database Debugging
```bash
# MySQL/MariaDB debugging
# Enable general log
mysql -u root -p -e "SET GLOBAL general_log = 'ON';"
mysql -u root -p -e "SET GLOBAL log_output = 'FILE';"

# Performance schema
mysql -u root -p -e "SELECT * FROM performance_schema.file_summary_by_event_name WHERE event_name LIKE 'wait/io/file%' ORDER BY sum_timer_wait DESC LIMIT 10;"

# PostgreSQL debugging
# Enable query logging
sudo -u postgres psql -c "ALTER SYSTEM SET log_statement = 'all';"
sudo -u postgres psql -c "SELECT pg_reload_conf();"

# Check query performance
sudo -u postgres psql -c "SELECT query, calls, total_time, mean_time FROM pg_stat_statements ORDER BY total_time DESC LIMIT 10;"

# MongoDB debugging
# Enable profiler
mongosh myapp --eval "db.setProfilingLevel(2, { slowms: 100 })"

# Check slow operations
mongosh myapp --eval "db.system.profile.find().sort({ts:-1}).limit(5).pretty()"

# Redis debugging
# Enable slow log
redis-cli CONFIG SET slowlog-log-slower-than 10000

# Check slow operations
redis-cli SLOWLOG GET 10

# Monitor memory usage
redis-cli --latency-history -i 1
```

## Additional Resources

- [MySQL Documentation](https://dev.mysql.com/doc/)
- [MariaDB Documentation](https://mariadb.com/docs/)
- [PostgreSQL Documentation](https://www.postgresql.org/docs/)
- [MongoDB Documentation](https://docs.mongodb.com/)
- [Redis Documentation](https://redis.io/documentation)
- [Database Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Database_Security_Cheat_Sheet.html)

---

**Note:** This guide is part of the [HowToMgr](https://howtomgr.github.io) collection.