# Systems Administration Patterns

Production patterns for Linux/Unix system administration.

## Service Management

### Systemd Service Template

```ini
# /etc/systemd/system/myapp.service
[Unit]
Description=My Application Service
Documentation=https://docs.example.com/myapp
After=network-online.target
Wants=network-online.target
StartLimitIntervalSec=300
StartLimitBurst=5

[Service]
Type=notify
User=appuser
Group=appgroup
WorkingDirectory=/opt/myapp
ExecStart=/opt/myapp/bin/server --config /etc/myapp/config.yaml
ExecReload=/bin/kill -HUP $MAINPID
ExecStop=/bin/kill -TERM $MAINPID

# Restart behavior
Restart=on-failure
RestartSec=5s
TimeoutStartSec=30s
TimeoutStopSec=30s

# Logging
StandardOutput=journal
StandardError=journal
SyslogIdentifier=myapp

# Security hardening
NoNewPrivileges=yes
ProtectSystem=strict
ProtectHome=yes
PrivateTmp=yes
PrivateDevices=yes
ProtectKernelTunables=yes
ProtectKernelModules=yes
ProtectControlGroups=yes
RestrictSUIDSGID=yes
RestrictNamespaces=yes

# Resource limits
LimitNOFILE=65535
LimitNPROC=4096
MemoryMax=2G
CPUQuota=200%

# Allowed paths
ReadWritePaths=/var/lib/myapp /var/log/myapp
ReadOnlyPaths=/etc/myapp

[Install]
WantedBy=multi-user.target
```

### Service Commands

```bash
# Lifecycle
systemctl start myapp
systemctl stop myapp
systemctl restart myapp
systemctl reload myapp

# Status and debugging
systemctl status myapp
systemctl is-active myapp
systemctl is-enabled myapp
systemctl show myapp --property=MainPID

# Enable/disable
systemctl enable myapp
systemctl disable myapp

# Logs
journalctl -u myapp -f
journalctl -u myapp --since "1 hour ago"
journalctl -u myapp -p err
journalctl -u myapp --no-pager -n 100
```

## User Management

### Secure User Creation

```bash
# Create system user (no login shell, no home)
useradd --system --no-create-home --shell /usr/sbin/nologin appuser

# Create regular user with specific UID
useradd --uid 1001 --gid users --create-home --shell /bin/bash username

# Add to supplementary groups
usermod -aG docker,sudo username

# Set password expiry
chage --maxdays 90 --warndays 14 username

# Lock/unlock account
usermod --lock username
usermod --unlock username
```

### SSH Key Management

```bash
# Generate key pair
ssh-keygen -t ed25519 -C "user@host" -f ~/.ssh/id_ed25519

# Copy public key to server
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@server

# Secure permissions
chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_ed25519
chmod 644 ~/.ssh/id_ed25519.pub
chmod 600 ~/.ssh/authorized_keys
chmod 644 ~/.ssh/known_hosts
chmod 600 ~/.ssh/config
```

### SSH Config

```ssh-config
# ~/.ssh/config
Host *
    AddKeysToAgent yes
    IdentitiesOnly yes
    ServerAliveInterval 60
    ServerAliveCountMax 3

Host prod-*
    User deploy
    IdentityFile ~/.ssh/id_ed25519_prod
    ProxyJump bastion

Host bastion
    HostName bastion.example.com
    User admin
    IdentityFile ~/.ssh/id_ed25519_bastion
    ForwardAgent no

Host dev
    HostName dev.example.com
    User developer
    LocalForward 5432 localhost:5432
```

## Security Hardening

### SSH Server Config

```ssh-config
# /etc/ssh/sshd_config
Port 22
Protocol 2

# Authentication
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
AuthenticationMethods publickey
MaxAuthTries 3
LoginGraceTime 30

# Users
AllowUsers deploy admin
DenyUsers root

# Security
X11Forwarding no
AllowTcpForwarding no
AllowAgentForwarding no
PermitTunnel no
ClientAliveInterval 300
ClientAliveCountMax 2

# Logging
LogLevel VERBOSE
```

### Fail2ban Configuration

```ini
# /etc/fail2ban/jail.local
[DEFAULT]
bantime = 3600
findtime = 600
maxretry = 3
banaction = iptables-multiport

[sshd]
enabled = true
port = ssh
filter = sshd
logpath = /var/log/auth.log
maxretry = 3
bantime = 86400

[nginx-http-auth]
enabled = true
filter = nginx-http-auth
port = http,https
logpath = /var/log/nginx/error.log
```

## Backup Patterns

### Rsync Backup Script

```bash
#!/usr/bin/env bash
set -euo pipefail

# Configuration
SOURCE="/data"
DEST="/backup"
REMOTE="backup@backup-server:/backups"
RETENTION_DAYS=30
LOG_FILE="/var/log/backup.log"

log() {
    echo "[$(date +'%Y-%m-%d %H:%M:%S')] $*" | tee -a "$LOG_FILE"
}

# Local backup with hard links (incremental)
backup_local() {
    local today=$(date +%Y-%m-%d)
    local latest="$DEST/latest"
    local target="$DEST/$today"
    
    if [[ -d "$latest" ]]; then
        rsync -avz --delete --link-dest="$latest" "$SOURCE/" "$target/"
    else
        rsync -avz --delete "$SOURCE/" "$target/"
    fi
    
    ln -snf "$target" "$latest"
    log "Local backup completed: $target"
}

# Remote sync
backup_remote() {
    rsync -avz --delete -e "ssh -i /root/.ssh/backup_key" \
        "$DEST/latest/" "$REMOTE/$(hostname)/"
    log "Remote sync completed"
}

# Cleanup old backups
cleanup() {
    find "$DEST" -maxdepth 1 -type d -mtime +$RETENTION_DAYS -exec rm -rf {} \;
    log "Cleanup completed (removed backups older than $RETENTION_DAYS days)"
}

main() {
    log "Starting backup..."
    backup_local
    backup_remote
    cleanup
    log "Backup completed successfully"
}

main "$@"
```

### Database Backup

```bash
#!/usr/bin/env bash
set -euo pipefail

BACKUP_DIR="/backup/postgres"
RETENTION_DAYS=7
DATE=$(date +%Y%m%d_%H%M%S)

# PostgreSQL
pg_dump -Fc -f "$BACKUP_DIR/db_$DATE.dump" mydb
pg_dumpall --globals-only -f "$BACKUP_DIR/globals_$DATE.sql"

# MySQL
mysqldump --single-transaction --routines --triggers \
    mydb > "$BACKUP_DIR/mysql_$DATE.sql"

# Compress
gzip "$BACKUP_DIR"/*_$DATE.*

# Cleanup
find "$BACKUP_DIR" -name "*.gz" -mtime +$RETENTION_DAYS -delete
```

## Monitoring

### Health Check Script

```bash
#!/usr/bin/env bash
set -euo pipefail

check_disk() {
    local threshold=90
    df -h | awk 'NR>1 {gsub(/%/,"",$5); if($5 > '$threshold') print "WARN: "$6" at "$5"%"}'
}

check_memory() {
    local threshold=90
    local used=$(free | awk '/Mem:/ {printf "%.0f", $3/$2 * 100}')
    [[ $used -gt $threshold ]] && echo "WARN: Memory at ${used}%"
}

check_load() {
    local threshold=$(nproc)
    local load=$(uptime | awk -F'load average:' '{print $2}' | cut -d, -f1 | tr -d ' ')
    (( $(echo "$load > $threshold" | bc -l) )) && echo "WARN: Load at $load"
}

check_services() {
    local services=("nginx" "postgresql" "redis")
    for svc in "${services[@]}"; do
        systemctl is-active --quiet "$svc" || echo "WARN: $svc is not running"
    done
}

main() {
    check_disk
    check_memory
    check_load
    check_services
}

main
```

### Log Rotation

```conf
# /etc/logrotate.d/myapp
/var/log/myapp/*.log {
    daily
    rotate 14
    compress
    delaycompress
    missingok
    notifempty
    create 0640 appuser appgroup
    sharedscripts
    postrotate
        systemctl reload myapp > /dev/null 2>&1 || true
    endscript
}
```

## Performance Tuning

### Kernel Parameters

```bash
# /etc/sysctl.d/99-performance.conf

# Network
net.core.somaxconn = 65535
net.core.netdev_max_backlog = 65535
net.ipv4.tcp_max_syn_backlog = 65535
net.ipv4.ip_local_port_range = 1024 65535
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_fin_timeout = 15
net.ipv4.tcp_keepalive_time = 300
net.ipv4.tcp_keepalive_probes = 5
net.ipv4.tcp_keepalive_intvl = 15

# Memory
vm.swappiness = 10
vm.dirty_ratio = 60
vm.dirty_background_ratio = 2
vm.overcommit_memory = 1

# File descriptors
fs.file-max = 2097152
fs.nr_open = 2097152

# Apply: sysctl -p /etc/sysctl.d/99-performance.conf
```

### Resource Limits

```bash
# /etc/security/limits.d/99-limits.conf
* soft nofile 65535
* hard nofile 65535
* soft nproc 65535
* hard nproc 65535
appuser soft memlock unlimited
appuser hard memlock unlimited
```

## Troubleshooting Checklist

| Issue | Commands |
|-------|----------|
| High CPU | `top -c`, `ps aux --sort=-%cpu`, `strace -p PID` |
| High Memory | `free -h`, `ps aux --sort=-%mem`, `smem` |
| Disk Full | `df -h`, `du -sh /*`, `lsof +D /path` |
| Network Issues | `ss -tulpn`, `netstat -i`, `tcpdump` |
| Service Down | `systemctl status`, `journalctl -u`, `dmesg` |
| Slow I/O | `iostat -x 1`, `iotop`, `dstat` |
| Permission Denied | `ls -la`, `namei -l /path`, `getfacl` |
| DNS Issues | `dig`, `nslookup`, `cat /etc/resolv.conf` |
