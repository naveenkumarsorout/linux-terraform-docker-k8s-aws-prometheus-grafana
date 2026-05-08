# SRE & DevOps Master Study Guide
## Linux · Terraform · Docker · AWS · Prometheus · Grafana
### Basic → Advanced | Production Use Cases | Interview Guide | Commands with Outputs & Errors

---

# TABLE OF CONTENTS

1. [LINUX](#linux)
2. [TERRAFORM](#terraform)
3. [DOCKER](#docker)
4. [AWS IMPORTANT SERVICES](#aws)
5. [PROMETHEUS & GRAFANA](#prometheus-grafana)
6. [INTERVIEW QUESTIONS (All Topics)](#interview-questions)

---

# LINUX

## 1. File System & Navigation

### Basic Commands

```bash
# Show current directory
pwd
# Output:
/home/devops

# List files with details
ls -lah
# Output:
total 48K
drwxr-xr-x  5 devops devops 4.0K May  9 10:00 .
drwxr-xr-x 20 root   root   4.0K May  9 09:00 ..
-rw-r--r--  1 devops devops  220 May  9 09:00 .bash_logout
-rw-r--r--  1 devops devops 3.5K May  9 09:00 .bashrc
drwxr-xr-x  2 devops devops 4.0K May  9 10:00 scripts
-rw-r--r--  1 devops devops 1.2K May  9 10:00 deploy.sh

# Navigate directories
cd /var/log
cd ~           # home directory
cd -           # previous directory

# Create directory structure
mkdir -p /app/configs/prod
# Output: (no output on success)

# Find files
find /var/log -name "*.log" -mtime -1
# Output:
/var/log/syslog
/var/log/auth.log
/var/log/nginx/access.log
```

### Production Use Case: Disk Space Investigation
```bash
# Step 1: Check overall disk usage
df -hT
# Output:
Filesystem     Type      Size  Used Avail Use% Mounted on
/dev/sda1      ext4       50G   48G  2.0G  96% /
tmpfs          tmpfs     3.9G  512K  3.9G   1% /dev/shm
/dev/sdb1      xfs       200G   50G  150G  25% /data

# Step 2: Find top disk consumers
du -sh /* 2>/dev/null | sort -rh | head -10
# Output:
22G  /var
15G  /opt
8.5G /home
4.2G /usr

# Step 3: Drill into /var
du -sh /var/* | sort -rh | head -5
# Output:
18G  /var/log
2.1G /var/lib
800M /var/cache

# Step 4: Find large files
find /var/log -type f -size +500M
# Output:
/var/log/app/app.log

# Step 5: Check if log rotation is configured
cat /etc/logrotate.d/app
# Output:
/var/log/app/*.log {
    daily
    missingok
    rotate 7
    compress
    delaycompress
    notifempty
    create 0640 www-data adm
}

# Step 6: Truncate a log safely (never delete live logs)
> /var/log/app/app.log
# or
truncate -s 0 /var/log/app/app.log
```

### Common Errors
```bash
# Error: No space left on device
touch /tmp/test
# touch: cannot touch '/tmp/test': No space left on device

# Fix: Find and clean inodes (sometimes disk space shows free but inodes exhausted)
df -i
# Output:
Filesystem      Inodes  IUsed   IFree IUse% Mounted on
/dev/sda1      3276800 3276800       0  100% /

find / -xdev -printf '%h\n' | sort | uniq -c | sort -k 1 -n | tail -20
# Clean session files or temp files consuming inodes
find /tmp -type f -delete
```

---

## 2. Process Management

### Basic Commands
```bash
# List all processes
ps aux
# Output:
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  0.0  0.1 225700  9012 ?        Ss   May08   0:05 /sbin/init
nginx     1234  0.5  1.2 125700 50012 ?        S    10:00   2:10 nginx: worker process
devops    5678  2.1  5.4 900700 220012 pts/0   Sl   10:05  15:30 java -jar app.jar

# Real-time process monitoring
top
htop  # Better alternative

# Find process by name
pgrep -la nginx
# Output:
1234 nginx: master process /usr/sbin/nginx
1235 nginx: worker process
1236 nginx: worker process

# Kill process
kill -9 1234         # Force kill
kill -15 1234        # Graceful kill (SIGTERM)
killall nginx        # Kill all nginx processes

# Background/Foreground jobs
./long_script.sh &   # Run in background
jobs                 # List background jobs
fg 1                 # Bring job 1 to foreground
bg 1                 # Resume stopped job in background
nohup ./script.sh &  # Run immune to hangup
```

### Production Use Case: High CPU Investigation
```bash
# Step 1: Identify high CPU process
top -bn1 | head -20
# Output:
top - 10:30:00 up 5 days,  2:15,  2 users,  load average: 8.50, 7.20, 5.10
Tasks: 245 total,   3 running, 242 sleeping,   0 stopped,   0 zombie
%Cpu(s): 95.2 us,  2.1 sy,  0.0 ni,  2.0 id,  0.0 wa,  0.5 hi,  0.2 si
  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
 9999 java      20   0 8.2g   5.1g   32m S  390.0 32.8  45:12.34 java

# Step 2: Check what threads are consuming CPU
top -H -p 9999
# Output:
  PID USER      PR  NI    VIRT    RES    SHR S %CPU %MEM     TIME+ COMMAND
10001 java      20   0 8.2g   5.1g   32m R 95.5 32.8   5:12.34 java
10002 java      20   0 8.2g   5.1g   32m R 94.2 32.8   5:10.12 java

# Step 3: Get thread dump (Java)
jstack 9999 > /tmp/threaddump_$(date +%s).txt

# Step 4: Check system load context
uptime
# Output:
10:30:05 up 5 days,  2:15,  2 users,  load average: 8.50, 7.20, 5.10
# Rule: Load > number of CPUs = system is overloaded

nproc
# Output: 4
# load average 8.50 on 4 CPUs = 2x overloaded

# Step 5: Check for runaway processes
sar -u 1 5
# Output:
Average:        CPU     %user     %nice   %system   %iowait    %steal     %idle
Average:        all     90.25      0.00      4.50      3.25      0.50      1.50
```

---

## 3. Memory Management

```bash
# Check memory usage
free -h
# Output:
              total        used        free      shared  buff/cache   available
Mem:           15Gi        12Gi       512Mi       512Mi       2.5Gi       2.5Gi
Swap:           4Gi         2Gi         2Gi

# Detailed memory info
cat /proc/meminfo | head -20
# Output:
MemTotal:       16384000 kB
MemFree:          524288 kB
MemAvailable:    2621440 kB
Buffers:          204800 kB
Cached:          2457600 kB
SwapCached:       102400 kB

# Check memory per process
ps aux --sort=-%mem | head -10
# Output:
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
java      9999  2.5 32.8 8599552 5349632 ?    Sl   08:00 120:12 java -jar app.jar
postgres  1234  0.5  8.5 1048576  1388544 ?   S    08:00  12:10 postgres: main

# Check OOM killer logs
dmesg | grep -i "out of memory"
# Output:
[1234567.890] Out of memory: Kill process 9999 (java) score 850 or sacrifice child
[1234567.891] Killed process 9999 (java) total-vm:8599552kB, anon-rss:5349632kB

# Check swap usage per process
for pid in $(ls /proc | grep -E '^[0-9]+$'); do
  swap=$(grep VmSwap /proc/$pid/status 2>/dev/null | awk '{print $2}')
  if [ ! -z "$swap" ] && [ "$swap" -gt 0 ]; then
    name=$(cat /proc/$pid/comm 2>/dev/null)
    echo "$pid $name ${swap}kB"
  fi
done | sort -k3 -rn | head -10
```

### Production Use Case: Memory Leak Investigation
```bash
# Monitor memory growth of a process over time
watch -n 5 'ps -p 9999 -o pid,rss,vsz,comm'
# Output (refreshes every 5s):
  PID   RSS    VSZ COMMAND
 9999 5349632 8599552 java

# If RSS grows continuously = memory leak

# Check valgrind for C/C++ apps
valgrind --leak-check=full --track-origins=yes ./myapp 2>&1 | tee /tmp/valgrind.log

# For Java: enable GC logging
java -Xmx4g -XX:+PrintGCDetails -XX:+PrintGCDateStamps \
     -Xloggc:/var/log/app/gc.log -jar app.jar
```

---

## 4. Networking

### Basic Commands
```bash
# Check IP addresses
ip addr show
# Output:
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default
    inet 10.0.1.50/24 brd 10.0.1.255 scope global eth0

# Check routing table
ip route show
# Output:
default via 10.0.1.1 dev eth0 proto dhcp src 10.0.1.50 metric 100
10.0.1.0/24 dev eth0 proto kernel scope link src 10.0.1.50

# Check open ports
ss -tlnp
# Output:
State   Recv-Q  Send-Q  Local Address:Port  Peer Address:Port  Process
LISTEN  0       128     0.0.0.0:22         0.0.0.0:*          users:(("sshd",pid=1234))
LISTEN  0       511     0.0.0.0:80         0.0.0.0:*          users:(("nginx",pid=5678))
LISTEN  0       511     0.0.0.0:443        0.0.0.0:*          users:(("nginx",pid=5678))
LISTEN  0       128     127.0.0.1:5432     0.0.0.0:*          users:(("postgres",pid=9012))

# Test connectivity
ping -c 4 8.8.8.8
# Output:
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=118 time=1.23 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=118 time=1.18 ms
--- 8.8.8.8 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss
rtt min/avg/max/mdev = 1.18/1.20/1.23/0.02 ms

# Trace route
traceroute 8.8.8.8
# Output:
 1  10.0.1.1 (10.0.1.1)  0.512 ms  0.489 ms  0.472 ms
 2  203.0.113.1 (203.0.113.1)  1.234 ms  1.201 ms
 3  8.8.8.8 (8.8.8.8)  1.891 ms  1.876 ms  1.862 ms

# DNS lookup
dig google.com +short
# Output:
142.250.80.46

nslookup google.com
# Output:
Server:         8.8.8.8
Address:        8.8.8.8#53
Non-authoritative answer:
Name:   google.com
Address: 142.250.80.46

# Check DNS resolution time
time dig google.com @8.8.8.8 +short
```

### Production Use Case: Network Issue Debugging
```bash
# Connection refused to app port 8080
curl -v http://localhost:8080/health
# Output:
* Trying 127.0.0.1:8080...
* connect to 127.0.0.1 port 8080 failed: Connection refused
curl: (7) Failed to connect to localhost port 8080: Connection refused

# Debug steps:
# 1. Is the service running?
systemctl status myapp
ss -tlnp | grep 8080

# 2. Is firewall blocking?
iptables -L -n | grep 8080
# or for firewalld
firewall-cmd --list-all

# 3. Check if service is listening on wrong interface
ss -tlnp | grep 8080
# Output:
LISTEN  0  128  127.0.0.1:8080  0.0.0.0:*  # Only listening on loopback!
# Fix: Change app config to listen on 0.0.0.0

# Test with tcpdump
tcpdump -i eth0 -n port 80 -w /tmp/capture.pcap
# Then analyze:
tcpdump -r /tmp/capture.pcap -n | head -20

# Check network statistics
netstat -s | grep -E "failed|error|retransmit"
# Output:
    134 failed connection attempts
    2156 resets received
    89 segments retransmitted
```

---

## 5. File Permissions & User Management

```bash
# File permissions
chmod 755 script.sh    # rwxr-xr-x
chmod 644 config.conf  # rw-r--r--
chmod 600 private.key  # rw-------

# Recursive change
chmod -R 755 /var/www/html
chown -R www-data:www-data /var/www/html

# Special permissions
chmod u+s /usr/bin/sudo   # SUID - runs as file owner
chmod g+s /shared/dir     # SGID - files inherit group
chmod +t /tmp             # Sticky bit - only owner can delete

# ACL (Access Control Lists)
setfacl -m u:devops:rwx /opt/app
getfacl /opt/app
# Output:
# file: opt/app
# owner: root
# group: root
user::rwx
user:devops:rwx
group::r-x
mask::rwx
other::r-x

# User management
useradd -m -s /bin/bash -G sudo,docker devops
passwd devops
usermod -aG wheel devops    # Add to group
userdel -r olduser          # Delete user and home

# sudo configuration
visudo
# Add:
devops ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart myapp

# Check who can sudo
grep -r sudo /etc/sudoers.d/
cat /etc/sudoers
```

---

## 6. System Services (systemd)

```bash
# Service management
systemctl start nginx
systemctl stop nginx
systemctl restart nginx
systemctl reload nginx    # Reload config without restart
systemctl enable nginx    # Start on boot
systemctl disable nginx
systemctl status nginx

# Output of systemctl status:
● nginx.service - A high performance web server and a reverse proxy server
   Loaded: loaded (/lib/systemd/system/nginx.service; enabled; vendor preset: enabled)
   Active: active (running) since Fri 2026-05-09 10:00:00 UTC; 2h 30min ago
     Docs: man:nginx(8)
 Main PID: 1234 (nginx)
    Tasks: 5 (limit: 4915)
   Memory: 25.6M
   CGroup: /system.slice/nginx.service
           ├─1234 nginx: master process /usr/sbin/nginx -g daemon on; master_process on;
           └─1235 nginx: worker process

# Create a systemd service
cat > /etc/systemd/system/myapp.service << 'EOF'
[Unit]
Description=My Application
After=network.target postgresql.service
Requires=postgresql.service

[Service]
Type=simple
User=appuser
WorkingDirectory=/opt/myapp
ExecStart=/usr/bin/java -Xmx2g -jar /opt/myapp/app.jar
ExecReload=/bin/kill -HUP $MAINPID
Restart=always
RestartSec=5
StandardOutput=journal
StandardError=journal
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable myapp
systemctl start myapp

# View service logs
journalctl -u nginx -f              # Follow logs
journalctl -u nginx --since today   # Today's logs
journalctl -u nginx -n 100          # Last 100 lines
journalctl -u nginx --since "2026-05-09 10:00" --until "2026-05-09 11:00"
```

---

## 7. Log Analysis

### Production Use Case: Incident Investigation
```bash
# Search for errors in last hour
journalctl --since "1 hour ago" -p err

# Grep patterns in logs
grep -E "ERROR|FATAL|Exception" /var/log/app/app.log | tail -50

# Count errors by type
grep "ERROR" /var/log/app/app.log | awk '{print $5}' | sort | uniq -c | sort -rn
# Output:
    523 NullPointerException
    234 DatabaseConnectionException
     89 TimeoutException
     12 OutOfMemoryError

# Find slow requests in nginx access log
awk '($NF > 1.0) {print $0}' /var/log/nginx/access.log | wc -l
# Count requests taking more than 1 second

# Top 10 IPs hitting the server
awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head -10
# Output:
   5234 192.168.1.100
   3456 10.0.0.50
    891 172.16.0.1

# Real-time log monitoring
tail -f /var/log/nginx/error.log | grep --line-buffered "crit\|alert\|emerg"

# Parse structured JSON logs
cat /var/log/app/app.json | jq 'select(.level == "ERROR") | .message' | sort | uniq -c

# AWK for log analysis
awk '
  /ERROR/ { errors++ }
  /WARN/  { warns++ }
  END { print "Errors:", errors, "Warnings:", warns }
' /var/log/app/app.log
# Output:
Errors: 523 Warnings: 1234
```

---

## 8. Shell Scripting for SRE/DevOps

### Production Script: Health Check
```bash
#!/bin/bash
# health_check.sh - Production health check script

set -euo pipefail

APP_URL="http://localhost:8080/health"
ALERT_EMAIL="oncall@company.com"
LOG_FILE="/var/log/health_check.log"
THRESHOLD_CPU=80
THRESHOLD_MEM=85
THRESHOLD_DISK=90

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOG_FILE"
}

check_http() {
    local url=$1
    local response
    response=$(curl -s -o /dev/null -w "%{http_code}:%{time_total}" --max-time 10 "$url")
    local status_code=$(echo "$response" | cut -d: -f1)
    local time=$(echo "$response" | cut -d: -f2)
    
    if [ "$status_code" -ne 200 ]; then
        log "CRITICAL: $url returned HTTP $status_code"
        return 1
    fi
    
    if (( $(echo "$time > 2.0" | bc -l) )); then
        log "WARNING: $url slow response: ${time}s"
    fi
    log "OK: $url - HTTP $status_code in ${time}s"
}

check_cpu() {
    local cpu_usage
    cpu_usage=$(top -bn1 | grep "Cpu(s)" | awk '{print $2}' | cut -d. -f1)
    if [ "$cpu_usage" -gt "$THRESHOLD_CPU" ]; then
        log "WARNING: CPU usage is ${cpu_usage}%"
        return 1
    fi
    log "OK: CPU usage is ${cpu_usage}%"
}

check_memory() {
    local mem_usage
    mem_usage=$(free | awk '/Mem:/ {printf "%.0f", $3/$2 * 100}')
    if [ "$mem_usage" -gt "$THRESHOLD_MEM" ]; then
        log "CRITICAL: Memory usage is ${mem_usage}%"
        return 1
    fi
    log "OK: Memory usage is ${mem_usage}%"
}

check_disk() {
    local disk_usage
    disk_usage=$(df / | awk 'NR==2 {print $5}' | tr -d '%')
    if [ "$disk_usage" -gt "$THRESHOLD_DISK" ]; then
        log "CRITICAL: Disk usage is ${disk_usage}%"
        return 1
    fi
    log "OK: Disk usage is ${disk_usage}%"
}

main() {
    log "=== Starting health check ==="
    local exit_code=0
    
    check_http "$APP_URL"   || exit_code=1
    check_cpu               || exit_code=1
    check_memory            || exit_code=1
    check_disk              || exit_code=1
    
    if [ $exit_code -ne 0 ]; then
        log "ALERT: Issues detected, sending notification"
        # mail -s "Health Check Failed" "$ALERT_EMAIL" < "$LOG_FILE"
    fi
    
    log "=== Health check complete ==="
    exit $exit_code
}

main "$@"
```

### Production Script: Log Rotation & Cleanup
```bash
#!/bin/bash
# cleanup.sh - Clean old logs and temp files

find /var/log/app -name "*.log" -mtime +30 -exec gzip {} \;
find /var/log/app -name "*.log.gz" -mtime +90 -delete
find /tmp -type f -atime +7 -delete
find /var/tmp -type f -atime +30 -delete

echo "Cleanup complete: $(date)"
```

---

## 9. Performance Tuning

```bash
# System limits
ulimit -a
# Output:
open files                      (-n) 1024
max user processes              (-u) 63454
virtual memory          (kbytes, -v) unlimited

# Increase file descriptor limits
echo "* soft nofile 65536" >> /etc/security/limits.conf
echo "* hard nofile 65536" >> /etc/security/limits.conf

# Kernel parameters
sysctl -a | grep net.ipv4
sysctl net.ipv4.ip_local_port_range
# Output: net.ipv4.ip_local_port_range = 32768 60999

# Production tuning for high-traffic servers
cat >> /etc/sysctl.conf << 'EOF'
# Network performance
net.core.somaxconn = 65535
net.ipv4.tcp_max_syn_backlog = 65535
net.ipv4.ip_local_port_range = 1024 65535
net.ipv4.tcp_fin_timeout = 15
net.ipv4.tcp_keepalive_time = 300
net.core.netdev_max_backlog = 5000

# Memory
vm.swappiness = 10
vm.dirty_ratio = 15
vm.dirty_background_ratio = 5
EOF

sysctl -p  # Apply without reboot
```

---

# TERRAFORM

## 1. Core Concepts & Commands

```bash
# Initialize terraform
terraform init
# Output:
Initializing the backend...
Initializing provider plugins...
- Finding hashicorp/aws versions matching "~> 5.0"...
- Installing hashicorp/aws v5.40.0...
- Installed hashicorp/aws v5.40.0 (signed by HashiCorp)
Terraform has been successfully initialized!

# Plan changes
terraform plan
# Output:
Terraform will perform the following actions:
  # aws_instance.web will be created
  + resource "aws_instance" "web" {
      + ami                          = "ami-0c55b159cbfafe1f0"
      + instance_type                = "t3.medium"
      + tags                         = {
          + "Environment" = "prod"
          + "Name"        = "web-server"
        }
    }
Plan: 1 to add, 0 to change, 0 to destroy.

# Apply changes
terraform apply -auto-approve
# Output:
aws_instance.web: Creating...
aws_instance.web: Still creating... [10s elapsed]
aws_instance.web: Creation complete after 32s [id=i-0abc123def456789]
Apply complete! Resources: 1 added, 0 changed, 0 destroyed.

# Show current state
terraform show

# List resources in state
terraform state list
# Output:
aws_instance.web
aws_security_group.web_sg
aws_vpc.main

# Destroy infrastructure
terraform destroy
terraform destroy -target=aws_instance.web  # Destroy specific resource

# Format code
terraform fmt

# Validate configuration
terraform validate
# Output:
Success! The configuration is valid.
```

---

## 2. Production Project Structure

```
production-infra/
├── environments/
│   ├── prod/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── terraform.tfvars
│   ├── staging/
│   │   ├── main.tf
│   │   └── terraform.tfvars
│   └── dev/
│       ├── main.tf
│       └── terraform.tfvars
├── modules/
│   ├── vpc/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── ec2/
│   ├── rds/
│   └── eks/
└── backend.tf
```

### Backend Configuration (Remote State)
```hcl
# backend.tf
terraform {
  required_version = ">= 1.0"
  
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }

  backend "s3" {
    bucket         = "mycompany-terraform-state"
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-state-lock"
  }
}
```

---

## 3. VPC Module (Production)

```hcl
# modules/vpc/main.tf
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = merge(var.tags, {
    Name = "${var.environment}-vpc"
  })
}

resource "aws_subnet" "public" {
  count             = length(var.public_subnet_cidrs)
  vpc_id            = aws_vpc.main.id
  cidr_block        = var.public_subnet_cidrs[count.index]
  availability_zone = var.availability_zones[count.index]
  map_public_ip_on_launch = true

  tags = merge(var.tags, {
    Name = "${var.environment}-public-subnet-${count.index + 1}"
    "kubernetes.io/role/elb" = "1"
  })
}

resource "aws_subnet" "private" {
  count             = length(var.private_subnet_cidrs)
  vpc_id            = aws_vpc.main.id
  cidr_block        = var.private_subnet_cidrs[count.index]
  availability_zone = var.availability_zones[count.index]

  tags = merge(var.tags, {
    Name = "${var.environment}-private-subnet-${count.index + 1}"
    "kubernetes.io/role/internal-elb" = "1"
  })
}

resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
  tags = merge(var.tags, { Name = "${var.environment}-igw" })
}

resource "aws_eip" "nat" {
  count  = length(var.public_subnet_cidrs)
  domain = "vpc"
}

resource "aws_nat_gateway" "main" {
  count         = length(var.public_subnet_cidrs)
  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = aws_subnet.public[count.index].id

  tags = merge(var.tags, {
    Name = "${var.environment}-nat-gw-${count.index + 1}"
  })
}

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }
  tags = merge(var.tags, { Name = "${var.environment}-public-rt" })
}

resource "aws_route_table" "private" {
  count  = length(var.private_subnet_cidrs)
  vpc_id = aws_vpc.main.id
  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.main[count.index].id
  }
  tags = merge(var.tags, { Name = "${var.environment}-private-rt-${count.index + 1}" })
}

resource "aws_route_table_association" "public" {
  count          = length(var.public_subnet_cidrs)
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

resource "aws_route_table_association" "private" {
  count          = length(var.private_subnet_cidrs)
  subnet_id      = aws_subnet.private[count.index].id
  route_table_id = aws_route_table.private[count.index].id
}
```

---

## 4. EC2 with Auto Scaling (Production)

```hcl
# modules/ec2/main.tf
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]
  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}

resource "aws_launch_template" "app" {
  name_prefix   = "${var.environment}-app-"
  image_id      = data.aws_ami.amazon_linux.id
  instance_type = var.instance_type
  key_name      = var.key_pair_name

  iam_instance_profile {
    name = aws_iam_instance_profile.app.name
  }

  network_interfaces {
    associate_public_ip_address = false
    security_groups             = [aws_security_group.app.id]
  }

  block_device_mappings {
    device_name = "/dev/xvda"
    ebs {
      volume_size           = 50
      volume_type           = "gp3"
      encrypted             = true
      delete_on_termination = true
    }
  }

  user_data = base64encode(templatefile("${path.module}/userdata.sh", {
    environment = var.environment
    app_version = var.app_version
  }))

  tag_specifications {
    resource_type = "instance"
    tags = merge(var.tags, { Name = "${var.environment}-app" })
  }

  lifecycle {
    create_before_destroy = true
  }
}

resource "aws_autoscaling_group" "app" {
  name                = "${var.environment}-app-asg"
  vpc_zone_identifier = var.private_subnet_ids
  min_size            = var.asg_min_size
  max_size            = var.asg_max_size
  desired_capacity    = var.asg_desired_capacity

  launch_template {
    id      = aws_launch_template.app.id
    version = "$Latest"
  }

  target_group_arns         = [aws_lb_target_group.app.arn]
  health_check_type         = "ELB"
  health_check_grace_period = 300

  instance_refresh {
    strategy = "Rolling"
    preferences {
      min_healthy_percentage = 50
    }
  }

  tag {
    key                 = "Name"
    value               = "${var.environment}-app"
    propagate_at_launch = true
  }
}

resource "aws_autoscaling_policy" "scale_up" {
  name                   = "${var.environment}-scale-up"
  autoscaling_group_name = aws_autoscaling_group.app.name
  adjustment_type        = "ChangeInCapacity"
  scaling_adjustment     = 2
  cooldown               = 300
}

resource "aws_cloudwatch_metric_alarm" "high_cpu" {
  alarm_name          = "${var.environment}-high-cpu"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "CPUUtilization"
  namespace           = "AWS/EC2"
  period              = 120
  statistic           = "Average"
  threshold           = 75
  alarm_actions       = [aws_autoscaling_policy.scale_up.arn]

  dimensions = {
    AutoScalingGroupName = aws_autoscaling_group.app.name
  }
}
```

---

## 5. Common Terraform Errors & Fixes

```bash
# Error 1: State lock
# Error: Error acquiring the state lock
# Lock Info: ID: 12345678-abcd-efgh
terraform force-unlock 12345678-abcd-efgh

# Error 2: Provider version conflict
# Error: Failed to query available provider packages
terraform init -upgrade

# Error 3: Resource already exists
# Error: EntityAlreadyExists: Role with name already exists
# Fix: Import existing resource into state
terraform import aws_iam_role.my_role my-existing-role-name

# Error 4: Cycle dependency
# Error: Cycle: aws_instance.a, aws_instance.b
# Fix: Use depends_on explicitly or refactor module dependencies

# Error 5: Backend config changed
# Error: Backend configuration changed
terraform init -reconfigure

# Error 6: State drift
terraform refresh   # Update state from real infrastructure
terraform plan      # See what would change

# Error 7: Module not found
# Error: Module not installed
terraform get       # Download modules
terraform init      # Re-initialize

# Error 8: Invalid value in tfvars
# Error: Invalid value for input variable
# Check terraform.tfvars types match variables.tf declarations

# Useful debug commands
TF_LOG=DEBUG terraform apply 2>&1 | tee /tmp/tf_debug.log
terraform show -json | jq '.values.root_module.resources[].address'
```

---

## 6. Terraform Workspaces

```bash
# Create and manage workspaces
terraform workspace new dev
terraform workspace new staging
terraform workspace new prod

terraform workspace list
# Output:
  default
  dev
* prod
  staging

terraform workspace select dev
terraform workspace show
# Output: dev

# Use workspace in configuration
resource "aws_instance" "web" {
  instance_type = terraform.workspace == "prod" ? "t3.large" : "t3.micro"
  
  tags = {
    Environment = terraform.workspace
  }
}
```

---

# DOCKER

## 1. Core Concepts & Commands

```bash
# Build image
docker build -t myapp:1.0.0 .
# Output:
[+] Building 45.2s (12/12) FINISHED
 => [internal] load build definition from Dockerfile
 => [1/8] FROM python:3.11-slim
 => [2/8] WORKDIR /app
 => [3/8] COPY requirements.txt .
 => [4/8] RUN pip install --no-cache-dir -r requirements.txt
 => [5/8] COPY . .
 => exporting to image
 => => writing image sha256:abc123
 => => naming to docker.io/library/myapp:1.0.0

# Run container
docker run -d \
  --name myapp \
  -p 8080:8080 \
  -e APP_ENV=production \
  -e DB_HOST=postgres \
  -v /data/app:/app/data \
  --restart unless-stopped \
  --memory 512m \
  --cpus 1.0 \
  myapp:1.0.0

# Output:
a1b2c3d4e5f6789012345678901234567890123456789012345678901234567890

# List containers
docker ps
# Output:
CONTAINER ID   IMAGE         COMMAND                  CREATED        STATUS        PORTS                    NAMES
a1b2c3d4e5f6   myapp:1.0.0   "python app.py"          5 min ago      Up 5 minutes  0.0.0.0:8080->8080/tcp   myapp

docker ps -a  # Show all including stopped

# Container logs
docker logs myapp -f --tail 100
# Output:
[2026-05-09 10:00:00] INFO Starting application...
[2026-05-09 10:00:01] INFO Connected to database at postgres:5432
[2026-05-09 10:00:02] INFO Server listening on port 8080

# Execute command in container
docker exec -it myapp bash
docker exec myapp cat /app/config.yaml

# Container resource usage
docker stats
# Output:
CONTAINER ID   NAME     CPU %   MEM USAGE / LIMIT   MEM %   NET I/O          BLOCK I/O
a1b2c3d4e5f6   myapp    2.45%   245MiB / 512MiB     47.85%  1.5GB / 500MB    0B / 100MB

# Inspect container
docker inspect myapp
docker inspect myapp | jq '.[0].NetworkSettings.IPAddress'
# Output: "172.17.0.2"

# Copy files
docker cp myapp:/app/logs/error.log /tmp/error.log
docker cp /tmp/config.yaml myapp:/app/config.yaml
```

---

## 2. Production Dockerfile Best Practices

```dockerfile
# Multi-stage build for production
# Stage 1: Build
FROM maven:3.9-openjdk-17 AS builder
WORKDIR /app
COPY pom.xml .
# Cache dependencies layer
RUN mvn dependency:go-offline -B
COPY src ./src
RUN mvn clean package -DskipTests -B

# Stage 2: Production image
FROM openjdk:17-jre-slim AS production

# Security: Run as non-root user
RUN groupadd -r appuser && useradd -r -g appuser appuser

WORKDIR /app

# Copy only the artifact
COPY --from=builder /app/target/*.jar app.jar

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=60s --retries=3 \
  CMD curl -f http://localhost:8080/actuator/health || exit 1

# Metadata
LABEL maintainer="devops@company.com" \
      version="1.0.0" \
      description="Production Java Application"

# JVM tuning for containers
ENV JAVA_OPTS="-Xms512m -Xmx1g -XX:+UseContainerSupport -XX:MaxRAMPercentage=75"

USER appuser
EXPOSE 8080

ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar app.jar"]
```

---

## 3. Docker Compose (Production)

```yaml
# docker-compose.prod.yml
version: "3.9"

services:
  app:
    image: myapp:${APP_VERSION:-latest}
    restart: unless-stopped
    ports:
      - "8080:8080"
    environment:
      - APP_ENV=production
      - DB_HOST=postgres
      - DB_NAME=${DB_NAME}
      - DB_USER=${DB_USER}
      - DB_PASSWORD=${DB_PASSWORD}
      - REDIS_HOST=redis
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s
    deploy:
      resources:
        limits:
          cpus: "2.0"
          memory: 1G
        reservations:
          cpus: "0.5"
          memory: 512M
    logging:
      driver: "json-file"
      options:
        max-size: "100m"
        max-file: "5"
    networks:
      - app-network

  postgres:
    image: postgres:15-alpine
    restart: unless-stopped
    environment:
      POSTGRES_DB: ${DB_NAME}
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER} -d ${DB_NAME}"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - app-network

  redis:
    image: redis:7-alpine
    restart: unless-stopped
    command: redis-server --appendonly yes --maxmemory 256mb --maxmemory-policy allkeys-lru
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 5
    networks:
      - app-network

  nginx:
    image: nginx:1.25-alpine
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/ssl:/etc/nginx/ssl:ro
    depends_on:
      - app
    networks:
      - app-network

volumes:
  postgres_data:
    driver: local
  redis_data:
    driver: local

networks:
  app-network:
    driver: bridge
```

```bash
# Docker Compose commands
docker compose -f docker-compose.prod.yml up -d
docker compose -f docker-compose.prod.yml down
docker compose -f docker-compose.prod.yml logs -f app
docker compose -f docker-compose.prod.yml ps
docker compose -f docker-compose.prod.yml exec app bash
docker compose -f docker-compose.prod.yml pull  # Update images
```

---

## 4. Docker Networking

```bash
# Network types
docker network ls
# Output:
NETWORK ID     NAME        DRIVER    SCOPE
abc123def456   bridge      bridge    local
def456ghi789   host        host      local
ghi789jkl012   none        null      local

# Create custom network
docker network create \
  --driver bridge \
  --subnet 172.20.0.0/16 \
  --gateway 172.20.0.1 \
  production-net

# Connect container to network
docker network connect production-net myapp

# Inspect network
docker network inspect production-net
# Output:
[{
    "Name": "production-net",
    "Driver": "bridge",
    "IPAM": {
        "Config": [{"Subnet": "172.20.0.0/16", "Gateway": "172.20.0.1"}]
    },
    "Containers": {
        "a1b2c3": {"Name": "myapp", "IPv4Address": "172.20.0.2/16"},
        "d4e5f6": {"Name": "postgres", "IPv4Address": "172.20.0.3/16"}
    }
}]
```

---

## 5. Docker Registry & Image Management

```bash
# Tag and push to ECR
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  123456789.dkr.ecr.us-east-1.amazonaws.com

docker tag myapp:1.0.0 123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:1.0.0
docker push 123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:1.0.0

# Image cleanup
docker image prune -a --filter "until=24h"
docker system prune -a --volumes  # CAUTION: removes everything unused

# Image security scanning
docker scout cves myapp:1.0.0
# Output:
✓ SBOM of image already cached, 245 packages indexed
✗ Detected 3 vulnerable packages with a total of 5 vulnerabilities
  CVE-2023-1234  CRITICAL  libssl  3.0.2 → 3.0.8
```

---

## 6. Common Docker Errors & Fixes

```bash
# Error 1: Container exits immediately
docker run myapp
# Exit code 1 - check logs
docker logs $(docker ps -lq)

# Error 2: Port already in use
docker run -p 8080:8080 myapp
# Error: Bind for 0.0.0.0:8080 failed: port is already allocated
lsof -i :8080
kill -9 $(lsof -t -i:8080)

# Error 3: Out of disk space
docker run myapp
# Error: No space left on device
docker system df
# Output:
TYPE            TOTAL     ACTIVE    SIZE      RECLAIMABLE
Images          50        10        25.5GB    18GB (70%)
Containers      20        5         1.2GB     800MB (66%)
Local Volumes   30        10        15GB      8GB (53%)

docker system prune -a  # Clean unused resources

# Error 4: Cannot pull image
docker pull myapp:latest
# Error: unauthorized: authentication required
docker logout
docker login registry.company.com

# Error 5: Container OOM killed
docker inspect myapp | jq '.[0].State.OOMKilled'
# Output: true
# Fix: Increase memory limit or fix memory leak

# Error 6: Volume permission denied
docker run -v /data:/app/data myapp
# Permission denied
# Fix: Match UID/GID
docker run -u $(id -u):$(id -g) -v /data:/app/data myapp
```

---

# AWS IMPORTANT SERVICES

## 1. EC2 (Elastic Compute Cloud)

### Production Use Cases
```bash
# Launch instance with AWS CLI
aws ec2 run-instances \
  --image-id ami-0c55b159cbfafe1f0 \
  --instance-type t3.medium \
  --key-name prod-keypair \
  --security-group-ids sg-0abc123 \
  --subnet-id subnet-0def456 \
  --iam-instance-profile Name=app-instance-profile \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=prod-app},{Key=Environment,Value=prod}]' \
  --user-data file://userdata.sh
# Output:
{
    "Instances": [{
        "InstanceId": "i-0abc123def456789",
        "InstanceType": "t3.medium",
        "PrivateIpAddress": "10.0.1.50",
        "State": {"Name": "pending"}
    }]
}

# List instances
aws ec2 describe-instances \
  --filters "Name=tag:Environment,Values=prod" "Name=instance-state-name,Values=running" \
  --query 'Reservations[].Instances[].[InstanceId,InstanceType,PrivateIpAddress,Tags[?Key==`Name`].Value|[0]]' \
  --output table

# Stop/Start instances
aws ec2 stop-instances --instance-ids i-0abc123def456789
aws ec2 start-instances --instance-ids i-0abc123def456789

# Create AMI (snapshot of instance)
aws ec2 create-image \
  --instance-id i-0abc123def456789 \
  --name "prod-app-$(date +%Y%m%d)" \
  --description "Production app snapshot"

# Connect via SSM (no SSH needed)
aws ssm start-session --target i-0abc123def456789

# Check instance metadata (from within instance)
curl http://169.254.169.254/latest/meta-data/instance-id
curl http://169.254.169.254/latest/meta-data/local-ipv4
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/
```

---

## 2. ELB / ALB (Application Load Balancer)

```bash
# Create ALB
aws elbv2 create-load-balancer \
  --name prod-alb \
  --subnets subnet-0pub1 subnet-0pub2 \
  --security-groups sg-0alb \
  --type application \
  --scheme internet-facing

# Create target group
aws elbv2 create-target-group \
  --name prod-app-tg \
  --protocol HTTP \
  --port 8080 \
  --vpc-id vpc-0abc123 \
  --health-check-path /health \
  --health-check-interval-seconds 30 \
  --healthy-threshold-count 2 \
  --unhealthy-threshold-count 3

# Register targets
aws elbv2 register-targets \
  --target-group-arn arn:aws:elasticloadbalancing:us-east-1:123456789:targetgroup/prod-app-tg/abc \
  --targets Id=i-0abc123,Port=8080

# Create listener with HTTPS redirect
aws elbv2 create-listener \
  --load-balancer-arn arn:aws:elasticloadbalancing:us-east-1:123456789:loadbalancer/app/prod-alb/abc \
  --protocol HTTPS \
  --port 443 \
  --certificates CertificateArn=arn:aws:acm:us-east-1:123456789:certificate/abc \
  --default-actions Type=forward,TargetGroupArn=arn:...

# Check target health
aws elbv2 describe-target-health \
  --target-group-arn arn:aws:elasticloadbalancing:us-east-1:123456789:targetgroup/prod-app-tg/abc
# Output:
{
    "TargetHealthDescriptions": [{
        "Target": {"Id": "i-0abc123", "Port": 8080},
        "TargetHealth": {"State": "healthy"}
    }, {
        "Target": {"Id": "i-0def456", "Port": 8080},
        "TargetHealth": {
            "State": "unhealthy",
            "Reason": "Target.FailedHealthChecks",
            "Description": "Health checks failed with these codes: [502]"
        }
    }]
}
```

---

## 3. RDS (Relational Database Service)

```bash
# Create RDS instance
aws rds create-db-instance \
  --db-instance-identifier prod-postgres \
  --db-instance-class db.t3.medium \
  --engine postgres \
  --engine-version 15.3 \
  --master-username dbadmin \
  --master-user-password $(aws secretsmanager get-secret-value --secret-id prod/db/password --query SecretString --output text | jq -r .password) \
  --allocated-storage 100 \
  --storage-type gp3 \
  --storage-encrypted \
  --vpc-security-group-ids sg-0db \
  --db-subnet-group-name prod-db-subnet-group \
  --backup-retention-period 7 \
  --preferred-backup-window "03:00-04:00" \
  --preferred-maintenance-window "Mon:04:00-Mon:05:00" \
  --multi-az \
  --deletion-protection \
  --enable-performance-insights \
  --tags Key=Environment,Value=prod

# Create read replica
aws rds create-db-instance-read-replica \
  --db-instance-identifier prod-postgres-read \
  --source-db-instance-identifier prod-postgres \
  --db-instance-class db.t3.medium

# Describe instances
aws rds describe-db-instances \
  --query 'DBInstances[*].[DBInstanceIdentifier,DBInstanceStatus,Endpoint.Address]' \
  --output table

# Take snapshot
aws rds create-db-snapshot \
  --db-instance-identifier prod-postgres \
  --db-snapshot-identifier prod-postgres-$(date +%Y%m%d)

# Restore from snapshot
aws rds restore-db-instance-from-db-snapshot \
  --db-instance-identifier prod-postgres-restored \
  --db-snapshot-identifier prod-postgres-20260509

# Performance Insights query (top waits)
aws pi get-resource-metrics \
  --service-type RDS \
  --identifier db-ABCDEFGHIJKLMNOPQRSTUVWXYZ \
  --metric-queries '[{"Metric": "db.load.avg"}]' \
  --start-time 2026-05-09T09:00:00Z \
  --end-time 2026-05-09T10:00:00Z \
  --period-in-seconds 60
```

---

## 4. S3 (Simple Storage Service)

```bash
# Create bucket
aws s3api create-bucket \
  --bucket mycompany-prod-data \
  --region us-east-1

# Enable versioning
aws s3api put-bucket-versioning \
  --bucket mycompany-prod-data \
  --versioning-configuration Status=Enabled

# Enable encryption
aws s3api put-bucket-encryption \
  --bucket mycompany-prod-data \
  --server-side-encryption-configuration '{
    "Rules": [{"ApplyServerSideEncryptionByDefault": {"SSEAlgorithm": "AES256"}}]
  }'

# Block public access
aws s3api put-public-access-block \
  --bucket mycompany-prod-data \
  --public-access-block-configuration \
    BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true

# Sync files
aws s3 sync /local/data s3://mycompany-prod-data/backup/ --delete
aws s3 sync s3://mycompany-prod-data/backup/ /local/restore/

# Copy with metadata
aws s3 cp file.tar.gz s3://mycompany-prod-data/backups/ \
  --storage-class STANDARD_IA \
  --server-side-encryption AES256

# Lifecycle policy (move to Glacier after 90 days)
aws s3api put-bucket-lifecycle-configuration \
  --bucket mycompany-prod-data \
  --lifecycle-configuration '{
    "Rules": [{
      "Id": "archive-old-logs",
      "Status": "Enabled",
      "Filter": {"Prefix": "logs/"},
      "Transitions": [
        {"Days": 30, "StorageClass": "STANDARD_IA"},
        {"Days": 90, "StorageClass": "GLACIER"}
      ],
      "Expiration": {"Days": 365}
    }]
  }'

# Pre-signed URL for temporary access
aws s3 presign s3://mycompany-prod-data/report.pdf --expires-in 3600
# Output: https://mycompany-prod-data.s3.amazonaws.com/report.pdf?X-Amz-...
```

---

## 5. IAM (Identity and Access Management)

```bash
# Create role for EC2
aws iam create-role \
  --role-name prod-app-role \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"Service": "ec2.amazonaws.com"},
      "Action": "sts:AssumeRole"
    }]
  }'

# Attach policies
aws iam attach-role-policy \
  --role-name prod-app-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore

# Create inline policy (least privilege)
aws iam put-role-policy \
  --role-name prod-app-role \
  --policy-name s3-read-access \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:ListBucket"],
      "Resource": [
        "arn:aws:s3:::mycompany-prod-data",
        "arn:aws:s3:::mycompany-prod-data/*"
      ]
    }]
  }'

# Create instance profile
aws iam create-instance-profile --instance-profile-name prod-app-profile
aws iam add-role-to-instance-profile \
  --instance-profile-name prod-app-profile \
  --role-name prod-app-role

# Check effective permissions
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::123456789:role/prod-app-role \
  --action-names s3:GetObject s3:DeleteObject \
  --resource-arns arn:aws:s3:::mycompany-prod-data/test.txt
# Output:
{
    "EvaluationResults": [
        {"EvalActionName": "s3:GetObject", "EvalDecision": "allowed"},
        {"EvalActionName": "s3:DeleteObject", "EvalDecision": "implicitDeny"}
    ]
}
```

---

## 6. CloudWatch

```bash
# Create metric alarm
aws cloudwatch put-metric-alarm \
  --alarm-name prod-cpu-high \
  --alarm-description "CPU > 80% for 5 minutes" \
  --metric-name CPUUtilization \
  --namespace AWS/EC2 \
  --statistic Average \
  --period 300 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2 \
  --dimensions Name=AutoScalingGroupName,Value=prod-asg \
  --alarm-actions arn:aws:sns:us-east-1:123456789:prod-alerts \
  --ok-actions arn:aws:sns:us-east-1:123456789:prod-alerts

# Get metric statistics
aws cloudwatch get-metric-statistics \
  --namespace AWS/EC2 \
  --metric-name CPUUtilization \
  --dimensions Name=InstanceId,Value=i-0abc123 \
  --start-time 2026-05-09T09:00:00Z \
  --end-time 2026-05-09T10:00:00Z \
  --period 300 \
  --statistics Average Maximum
# Output:
{
    "Datapoints": [
        {"Timestamp": "2026-05-09T09:00:00Z", "Average": 45.2, "Maximum": 89.5},
        {"Timestamp": "2026-05-09T09:05:00Z", "Average": 62.1, "Maximum": 95.2}
    ]
}

# CloudWatch Logs - query with Insights
aws logs start-query \
  --log-group-name /app/production \
  --start-time $(date -d "1 hour ago" +%s) \
  --end-time $(date +%s) \
  --query-string 'fields @timestamp, @message | filter @message like /ERROR/ | stats count(*) as errorCount by bin(5m) | sort @timestamp desc'

# Get query results
aws logs get-query-results --query-id <query-id>

# Create log metric filter (count errors)
aws logs put-metric-filter \
  --log-group-name /app/production \
  --filter-name error-count \
  --filter-pattern "ERROR" \
  --metric-transformations \
    metricName=ErrorCount,metricNamespace=MyApp,metricValue=1,defaultValue=0
```

---

## 7. EKS (Elastic Kubernetes Service)

```bash
# Create EKS cluster (using eksctl)
eksctl create cluster \
  --name prod-cluster \
  --region us-east-1 \
  --nodegroup-name prod-nodes \
  --node-type m5.large \
  --nodes 3 \
  --nodes-min 2 \
  --nodes-max 10 \
  --managed \
  --with-oidc \
  --ssh-access \
  --ssh-public-key prod-keypair

# Update kubeconfig
aws eks update-kubeconfig --region us-east-1 --name prod-cluster

# Basic kubectl commands
kubectl get nodes
# Output:
NAME                          STATUS   ROLES    AGE   VERSION
ip-10-0-1-50.ec2.internal     Ready    <none>   2d    v1.28.5-eks-5e0fdde
ip-10-0-2-75.ec2.internal     Ready    <none>   2d    v1.28.5-eks-5e0fdde
ip-10-0-3-100.ec2.internal    Ready    <none>   2d    v1.28.5-eks-5e0fdde

kubectl get pods --all-namespaces
kubectl describe pod myapp-6d8f9b-xyz -n production
kubectl logs myapp-6d8f9b-xyz -n production -f --tail=100

# Deploy application
kubectl apply -f deployment.yaml
kubectl rollout status deployment/myapp -n production
kubectl rollout history deployment/myapp -n production

# Rollback
kubectl rollout undo deployment/myapp -n production
kubectl rollout undo deployment/myapp --to-revision=3 -n production

# Scale deployment
kubectl scale deployment myapp --replicas=5 -n production

# Port forward for debugging
kubectl port-forward pod/myapp-6d8f9b-xyz 8080:8080 -n production

# Get events (for debugging)
kubectl get events --sort-by='.lastTimestamp' -n production
```

---

## 8. Secrets Manager

```bash
# Create secret
aws secretsmanager create-secret \
  --name prod/app/database \
  --description "Production database credentials" \
  --secret-string '{"username":"dbadmin","password":"SecurePass123!","host":"prod-db.cluster-xyz.us-east-1.rds.amazonaws.com","port":5432,"dbname":"appdb"}'

# Get secret value
aws secretsmanager get-secret-value \
  --secret-id prod/app/database \
  --query SecretString \
  --output text | jq .
# Output:
{
  "username": "dbadmin",
  "password": "SecurePass123!",
  "host": "prod-db.cluster-xyz.us-east-1.rds.amazonaws.com",
  "port": 5432,
  "dbname": "appdb"
}

# Rotate secret
aws secretsmanager rotate-secret \
  --secret-id prod/app/database \
  --rotation-lambda-arn arn:aws:lambda:us-east-1:123456789:function:SecretsManagerRDSRotation

# Use in application (Python)
import boto3, json
client = boto3.client('secretsmanager', region_name='us-east-1')
response = client.get_secret_value(SecretId='prod/app/database')
secret = json.loads(response['SecretString'])
```

---

# PROMETHEUS & GRAFANA

## 1. Prometheus Setup & Configuration

### prometheus.yml (Production)
```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  external_labels:
    cluster: 'prod'
    region: 'us-east-1'

alerting:
  alertmanagers:
    - static_configs:
        - targets: ['alertmanager:9093']

rule_files:
  - "/etc/prometheus/rules/*.yml"

scrape_configs:
  # Prometheus itself
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  # Node Exporter (system metrics)
  - job_name: 'node-exporter'
    static_configs:
      - targets:
          - 'prod-app-1:9100'
          - 'prod-app-2:9100'
          - 'prod-db-1:9100'
    relabel_configs:
      - source_labels: [__address__]
        target_label: instance

  # Application metrics
  - job_name: 'myapp'
    metrics_path: '/actuator/prometheus'
    scrape_interval: 10s
    static_configs:
      - targets: ['myapp:8080']
    metric_relabel_configs:
      - source_labels: [__name__]
        regex: 'jvm_.*|http_.*|process_.*'
        action: keep

  # Kubernetes pods (auto-discovery)
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)
      - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
        action: replace
        regex: ([^:]+)(?::\d+)?;(\d+)
        replacement: $1:$2
        target_label: __address__
      - action: labelmap
        regex: __meta_kubernetes_pod_label_(.+)
      - source_labels: [__meta_kubernetes_namespace]
        action: replace
        target_label: kubernetes_namespace
      - source_labels: [__meta_kubernetes_pod_name]
        action: replace
        target_label: kubernetes_pod_name

  # MySQL Exporter
  - job_name: 'mysql'
    static_configs:
      - targets: ['mysql-exporter:9104']

  # Nginx Exporter
  - job_name: 'nginx'
    static_configs:
      - targets: ['nginx-exporter:9113']

  # Blackbox (endpoint probing)
  - job_name: 'blackbox-http'
    metrics_path: /probe
    params:
      module: [http_2xx]
    static_configs:
      - targets:
          - https://myapp.company.com/health
          - https://api.company.com/v1/status
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: blackbox-exporter:9115
```

---

## 2. Alert Rules (Production)

```yaml
# /etc/prometheus/rules/alerts.yml
groups:
  - name: infrastructure
    interval: 30s
    rules:
      - alert: HighCPUUsage
        expr: 100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High CPU usage on {{ $labels.instance }}"
          description: "CPU usage is {{ $value | printf \"%.2f\" }}% on {{ $labels.instance }}"

      - alert: HighMemoryUsage
        expr: (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100 > 85
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High memory usage on {{ $labels.instance }}"
          description: "Memory usage is {{ $value | printf \"%.2f\" }}% on {{ $labels.instance }}"

      - alert: DiskSpaceLow
        expr: (1 - (node_filesystem_avail_bytes{fstype!="tmpfs"} / node_filesystem_size_bytes{fstype!="tmpfs"})) * 100 > 85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Low disk space on {{ $labels.instance }}"
          description: "Disk usage is {{ $value | printf \"%.2f\" }}% on {{ $labels.instance }}:{{ $labels.mountpoint }}"

      - alert: DiskSpaceCritical
        expr: (1 - (node_filesystem_avail_bytes{fstype!="tmpfs"} / node_filesystem_size_bytes{fstype!="tmpfs"})) * 100 > 95
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "CRITICAL: Disk almost full on {{ $labels.instance }}"

  - name: application
    rules:
      - alert: HighErrorRate
        expr: |
          rate(http_server_requests_seconds_count{status=~"5.."}[5m])
          /
          rate(http_server_requests_seconds_count[5m]) > 0.05
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "High error rate: {{ $labels.application }}"
          description: "Error rate is {{ $value | printf \"%.2%\" }} for {{ $labels.uri }}"

      - alert: HighResponseTime
        expr: histogram_quantile(0.95, rate(http_server_requests_seconds_bucket[5m])) > 2
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High response time: {{ $labels.application }}"
          description: "95th percentile response time is {{ $value | printf \"%.2f\" }}s"

      - alert: ServiceDown
        expr: up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Service down: {{ $labels.job }} on {{ $labels.instance }}"
          description: "{{ $labels.job }} has been down for more than 1 minute"

      - alert: JVMHeapUsageHigh
        expr: |
          jvm_memory_used_bytes{area="heap"} / jvm_memory_max_bytes{area="heap"} * 100 > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "JVM heap usage high on {{ $labels.instance }}"
          description: "JVM heap is {{ $value | printf \"%.2f\" }}% full"

  - name: database
    rules:
      - alert: PostgresDown
        expr: pg_up == 0
        for: 30s
        labels:
          severity: critical
        annotations:
          summary: "PostgreSQL is down on {{ $labels.instance }}"

      - alert: PostgresReplicationLag
        expr: pg_replication_lag_seconds > 30
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "PostgreSQL replication lag: {{ $value }}s"

      - alert: TooManyConnections
        expr: |
          sum by (instance) (pg_stat_activity_count) 
          / pg_settings_max_connections * 100 > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "PostgreSQL connection pool > 80% on {{ $labels.instance }}"
```

---

## 3. PromQL Queries (Production)

```promql
# CPU Usage per instance (%)
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# Memory usage (%)
(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100

# Disk usage (%)
(1 - (node_filesystem_avail_bytes{fstype!="tmpfs"} / node_filesystem_size_bytes{fstype!="tmpfs"})) * 100

# Network I/O (bytes/sec)
rate(node_network_receive_bytes_total{device!="lo"}[5m])
rate(node_network_transmit_bytes_total{device!="lo"}[5m])

# HTTP Request rate (req/sec)
sum(rate(http_server_requests_seconds_count[5m])) by (application, uri)

# HTTP Error rate (%)
sum(rate(http_server_requests_seconds_count{status=~"5.."}[5m])) by (application)
/
sum(rate(http_server_requests_seconds_count[5m])) by (application) * 100

# 95th percentile response time
histogram_quantile(0.95, sum(rate(http_server_requests_seconds_bucket[5m])) by (le, application, uri))

# 99th percentile response time
histogram_quantile(0.99, sum(rate(http_server_requests_seconds_bucket[5m])) by (le, application))

# JVM GC pause time
rate(jvm_gc_pause_seconds_sum[5m]) / rate(jvm_gc_pause_seconds_count[5m])

# JVM heap usage
jvm_memory_used_bytes{area="heap"} / jvm_memory_max_bytes{area="heap"} * 100

# Active database connections
pg_stat_activity_count{datname="appdb"}

# Container CPU usage
rate(container_cpu_usage_seconds_total{image!=""}[5m]) * 100

# Container memory usage
container_memory_usage_bytes{image!=""} / container_spec_memory_limit_bytes{image!=""} * 100

# Uptime
time() - node_boot_time_seconds
```

---

## 4. Alertmanager Configuration

```yaml
# alertmanager.yml
global:
  smtp_smarthost: 'smtp.company.com:587'
  smtp_from: 'alerts@company.com'
  smtp_auth_username: 'alerts@company.com'
  smtp_auth_password: 'smtp-password'
  slack_api_url: 'https://hooks.slack.com/services/T00000000/B00000000/XXXXXXXXXXXX'
  pagerduty_url: 'https://events.pagerduty.com/v2/enqueue'

route:
  group_by: ['alertname', 'cluster', 'service']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  receiver: 'default'
  routes:
    - match:
        severity: critical
      receiver: 'pagerduty-critical'
      continue: true
    - match:
        severity: critical
      receiver: 'slack-critical'
    - match:
        severity: warning
      receiver: 'slack-warning'

receivers:
  - name: 'default'
    slack_configs:
      - channel: '#alerts'
        title: '{{ .GroupLabels.alertname }}'
        text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'

  - name: 'pagerduty-critical'
    pagerduty_configs:
      - routing_key: 'your-pagerduty-routing-key'
        description: '{{ .GroupLabels.alertname }}: {{ .CommonAnnotations.summary }}'
        severity: critical

  - name: 'slack-critical'
    slack_configs:
      - channel: '#critical-alerts'
        color: 'danger'
        title: '🔴 CRITICAL: {{ .GroupLabels.alertname }}'
        text: |
          *Alert:* {{ .CommonAnnotations.summary }}
          *Details:* {{ .CommonAnnotations.description }}
          *Instances:* {{ range .Alerts }}{{ .Labels.instance }} {{ end }}
        send_resolved: true

  - name: 'slack-warning'
    slack_configs:
      - channel: '#alerts'
        color: 'warning'
        title: '⚠️ WARNING: {{ .GroupLabels.alertname }}'
        text: |
          *Summary:* {{ .CommonAnnotations.summary }}
          *Description:* {{ .CommonAnnotations.description }}
        send_resolved: true

inhibit_rules:
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname', 'instance']
```

---

## 5. Grafana Dashboards & Configuration

### Data Source Setup (API)
```bash
# Add Prometheus data source via API
curl -X POST http://admin:admin@localhost:3000/api/datasources \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Prometheus",
    "type": "prometheus",
    "url": "http://prometheus:9090",
    "access": "proxy",
    "isDefault": true,
    "jsonData": {
      "timeInterval": "15s",
      "httpMethod": "POST"
    }
  }'
# Output:
{"id":1,"message":"Datasource added","name":"Prometheus"}

# Import dashboard by ID
curl -X POST http://admin:admin@localhost:3000/api/dashboards/import \
  -H "Content-Type: application/json" \
  -d '{"dashboard":{"id":1860},"overwrite":true,"inputs":[{"name":"DS_PROMETHEUS","type":"datasource","pluginId":"prometheus","value":"Prometheus"}]}'
```

### Production Dashboard Panel Examples
```json
// Panel: Request Rate
{
  "title": "Request Rate",
  "type": "stat",
  "targets": [{
    "expr": "sum(rate(http_server_requests_seconds_count[5m]))",
    "legendFormat": "req/s"
  }],
  "options": {
    "colorMode": "background",
    "graphMode": "area",
    "justifyMode": "auto"
  },
  "fieldConfig": {
    "defaults": {
      "unit": "reqps",
      "thresholds": {
        "steps": [
          {"color": "green", "value": null},
          {"color": "yellow", "value": 1000},
          {"color": "red", "value": 5000}
        ]
      }
    }
  }
}
```

### Grafana Provisioning (as Code)
```yaml
# /etc/grafana/provisioning/datasources/prometheus.yaml
apiVersion: 1
datasources:
  - name: Prometheus
    type: prometheus
    url: http://prometheus:9090
    isDefault: true
    access: proxy
    jsonData:
      timeInterval: "15s"

# /etc/grafana/provisioning/dashboards/all.yaml
apiVersion: 1
providers:
  - name: 'Default'
    folder: 'Production'
    type: file
    options:
      path: /var/lib/grafana/dashboards
      foldersFromFilesStructure: true
```

---

## 6. Exporters Setup

```bash
# Node Exporter (system metrics)
docker run -d \
  --name node-exporter \
  --net host \
  --pid host \
  -v /:/host:ro,rslave \
  prom/node-exporter \
  --path.rootfs=/host

# Verify metrics
curl http://localhost:9100/metrics | grep "node_cpu_seconds_total" | head -5
# Output:
node_cpu_seconds_total{cpu="0",mode="idle"} 8765.43
node_cpu_seconds_total{cpu="0",mode="system"} 123.45
node_cpu_seconds_total{cpu="0",mode="user"} 456.78

# cAdvisor (container metrics)
docker run -d \
  --name cadvisor \
  --volume /:/rootfs:ro \
  --volume /var/run:/var/run:ro \
  --volume /sys:/sys:ro \
  --volume /var/lib/docker/:/var/lib/docker:ro \
  -p 8080:8080 \
  gcr.io/cadvisor/cadvisor

# Blackbox Exporter (probe endpoints)
docker run -d \
  --name blackbox-exporter \
  -p 9115:9115 \
  -v $(pwd)/blackbox.yml:/etc/blackbox_exporter/config.yml \
  prom/blackbox-exporter

# Test probe
curl "http://localhost:9115/probe?target=https://google.com&module=http_2xx"
# Output:
# HELP probe_success Displays whether or not the probe was a success
# TYPE probe_success gauge
probe_success 1
probe_duration_seconds 0.234567
probe_http_status_code 200
```

---

## 7. Full Stack Docker Compose (Monitoring)

```yaml
# monitoring/docker-compose.yml
version: "3.9"

services:
  prometheus:
    image: prom/prometheus:v2.48.0
    container_name: prometheus
    restart: unless-stopped
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - ./rules:/etc/prometheus/rules
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=30d'
      - '--web.console.libraries=/usr/share/prometheus/console_libraries'
      - '--web.console.templates=/usr/share/prometheus/consoles'
      - '--web.enable-lifecycle'
      - '--web.enable-admin-api'
    ports:
      - "9090:9090"
    networks:
      - monitoring

  alertmanager:
    image: prom/alertmanager:v0.26.0
    container_name: alertmanager
    restart: unless-stopped
    volumes:
      - ./alertmanager.yml:/etc/alertmanager/alertmanager.yml
      - alertmanager_data:/alertmanager
    command:
      - '--config.file=/etc/alertmanager/alertmanager.yml'
      - '--storage.path=/alertmanager'
    ports:
      - "9093:9093"
    networks:
      - monitoring

  grafana:
    image: grafana/grafana:10.2.0
    container_name: grafana
    restart: unless-stopped
    environment:
      GF_SECURITY_ADMIN_PASSWORD: ${GRAFANA_PASSWORD:-admin}
      GF_USERS_ALLOW_SIGN_UP: "false"
      GF_SERVER_ROOT_URL: "https://grafana.company.com"
      GF_SMTP_ENABLED: "true"
      GF_SMTP_HOST: "smtp.company.com:587"
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning
      - ./grafana/dashboards:/var/lib/grafana/dashboards
    ports:
      - "3000:3000"
    depends_on:
      - prometheus
    networks:
      - monitoring

  node-exporter:
    image: prom/node-exporter:v1.7.0
    container_name: node-exporter
    restart: unless-stopped
    pid: host
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.sysfs=/host/sys'
      - '--path.rootfs=/rootfs'
    ports:
      - "9100:9100"
    networks:
      - monitoring

  cadvisor:
    image: gcr.io/cadvisor/cadvisor:v0.47.2
    container_name: cadvisor
    restart: unless-stopped
    privileged: true
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
    ports:
      - "8080:8080"
    networks:
      - monitoring

volumes:
  prometheus_data:
  alertmanager_data:
  grafana_data:

networks:
  monitoring:
    driver: bridge
```

---

## 8. Common Prometheus/Grafana Errors & Fixes

```bash
# Error 1: Target scrape failed
# Prometheus UI: Status > Targets shows "connection refused"
# Fix: Check if exporter is running
curl http://prod-app-1:9100/metrics

# Error 2: TSDB storage full
# prometheus.log: "err="no space left on device""
df -h /prometheus
# Fix: Reduce retention or add disk
# prometheus command: --storage.tsdb.retention.size=50GB

# Error 3: High cardinality (too many time series)
# Prometheus UI: Status > TSDB Stats
curl http://localhost:9090/api/v1/status/tsdb | jq '.data.seriesCountByMetricName | to_entries | sort_by(.value) | reverse | .[0:10]'
# Fix: Add metric_relabel_configs to drop high-cardinality labels

# Error 4: Alert never fires (check expression)
curl 'http://localhost:9090/api/v1/query?query=up==0'
# Or use Prometheus UI: Graph tab to test PromQL

# Error 5: Grafana "No data" panel
# Check data source: Explore tab, run raw query
# Check time range alignment
# Check metric name exists: curl http://prometheus:9090/api/v1/label/__name__/values | jq '.data[]' | grep mymetric

# Error 6: Alertmanager not sending
# Check config is valid
amtool check-config /etc/alertmanager/alertmanager.yml
# Test alert routing
amtool alert add alertname=TestAlert severity=critical --alertmanager.url=http://localhost:9093

# Reload Prometheus config without restart
curl -X POST http://localhost:9090/-/reload

# Reload Alertmanager config
curl -X POST http://localhost:9093/-/reload
```

---

# INTERVIEW QUESTIONS

## Linux Interview Questions

### Basic Level
**Q: What is the difference between a process and a thread?**
A: A process is an independent program in execution with its own memory space, file descriptors, and system resources. A thread is a unit of execution within a process, sharing the same memory space with other threads in the same process. Threads are lighter and context-switch faster, but bugs in one thread can corrupt shared memory affecting all threads.

**Q: What does `chmod 755` mean?**
A: `7` (rwx) for owner, `5` (r-x) for group, `5` (r-x) for others. Binary: 7=111, 5=101. The owner can read, write, execute; group and others can only read and execute.

**Q: How do you check what process is using a specific port?**
```bash
ss -tlnp | grep :8080
lsof -i :8080
fuser 8080/tcp
```

**Q: What is the difference between `>` and `>>`?**
A: `>` overwrites the file; `>>` appends to the file.

### Intermediate Level
**Q: How do you troubleshoot high load average?**
A: 
1. Check `uptime` and compare load to CPU count via `nproc`
2. `top` or `htop` to identify CPU/IO-bound processes
3. If high `%wa` (wait) in top → IO bottleneck, check `iostat -x 1`
4. If high `%us` (user) → CPU-bound, identify process with `ps aux --sort=-%cpu`
5. Check for zombie processes: `ps aux | grep Z`

**Q: Explain how `sudo` works internally.**
A: `sudo` is a SUID binary owned by root. When executed, it runs with root's effective UID regardless of who called it. It reads `/etc/sudoers` to check if the calling user is authorized for the requested command. It logs the action to syslog/auth.log. The SUID bit allows this privilege escalation without requiring root password.

**Q: What is inode exhaustion and how do you fix it?**
A: Inodes are data structures storing file metadata (permissions, timestamps, ownership). Each file/directory consumes one inode regardless of size. Inode exhaustion occurs when all inodes are used despite available disk space. Common cause: millions of small files (e.g., mail spools, session files). 
Fix: `df -i` to confirm, `find / -xdev -printf '%h\n' | sort | uniq -c | sort -rn | head` to find directories with most files, then clean.

### Advanced Level
**Q: How does the Linux kernel handle context switching?**
A: The scheduler saves the current process state (registers, program counter, stack pointer) to its PCB (Process Control Block), restores the next process's saved state, updates memory management registers (CR3 for page table), and transfers control. Context switches are triggered by: scheduler timer interrupts, I/O blocking, explicit yield (`sched_yield`), or priority preemption.

**Q: Explain cgroups and how Docker uses them.**
A: Control Groups (cgroups) are a kernel feature that limit, account for, and isolate resource usage (CPU, memory, disk I/O, network) of process groups. Docker uses cgroups v2 to enforce `--memory`, `--cpus`, and `--pids-limit` flags. Each container gets its own cgroup hierarchy. You can inspect: `cat /sys/fs/cgroup/system.slice/docker-<containerid>.scope/memory.max`

---

## Terraform Interview Questions

**Q: What is Terraform state and why is it important?**
A: State is a JSON file mapping your Terraform configuration to real-world resources. Terraform uses it to determine what changes to make (diff between desired and current state), track resource metadata (IDs, attributes), manage dependencies, and enable collaboration. Without state, Terraform can't know what it already created.

**Q: Explain the difference between `terraform plan` and `terraform apply`.**
A: `plan` shows what changes Terraform would make (add/change/destroy) without making them — a dry run. `apply` executes those changes. In production, always run plan first, review output, then apply. CI/CD pipelines often save the plan file: `terraform plan -out=plan.tfplan && terraform apply plan.tfplan`

**Q: What is the purpose of `terraform import`?**
A: It brings existing infrastructure (created outside Terraform) under Terraform management by adding it to the state file. After import, you must write the matching Terraform configuration manually — import only updates state, not HCL files.

**Q: How do you manage sensitive values in Terraform?**
A: 
1. Never hardcode in `.tf` files
2. Use `sensitive = true` on variable/output definitions
3. Pass via environment variables: `TF_VAR_db_password=...`
4. Use AWS Secrets Manager + data source:
```hcl
data "aws_secretsmanager_secret_version" "db" {
  secret_id = "prod/db/password"
}
```
5. Use Vault provider for HashiCorp Vault

**Q: What is the lifecycle meta-argument and when do you use it?**
A: Controls Terraform's resource lifecycle:
- `create_before_destroy`: creates replacement before destroying old resource (zero-downtime)
- `prevent_destroy`: blocks accidental deletion
- `ignore_changes`: ignore drift on specific attributes (e.g., tags changed externally)
- `replace_triggered_by`: force replacement when referenced resource changes

---

## Docker Interview Questions

**Q: What is the difference between `CMD` and `ENTRYPOINT` in Dockerfile?**
A: `ENTRYPOINT` sets the fixed executable that always runs. `CMD` provides default arguments to ENTRYPOINT, or the default command if no ENTRYPOINT is set. `CMD` is overridden when you pass arguments to `docker run`. Best practice: use `ENTRYPOINT` for the main executable and `CMD` for default flags.

**Q: How does Docker networking work? What are the network types?**
A: 
- `bridge`: Default. Containers get their own network namespace, connected via virtual bridge. Containers communicate via container name on same bridge.
- `host`: Container shares host's network namespace. Best performance, no port mapping needed.
- `none`: No network access.
- `overlay`: Multi-host networking for Docker Swarm/clusters.
- `macvlan`: Container gets MAC address, appears as physical device on network.

**Q: What is a Docker layer and how does caching work?**
A: Each Dockerfile instruction creates a read-only layer. Layers are cached by content hash. On rebuild, Docker reuses cached layers until it finds a changed instruction — all subsequent layers are rebuilt. Best practice: put frequently changing instructions (COPY source code) after rarely changing ones (RUN apt-get install).

**Q: How do you reduce Docker image size?**
A:
1. Use minimal base images (alpine, distroless, slim variants)
2. Multi-stage builds — copy only final artifact
3. Combine RUN commands with `&&` (fewer layers)
4. Clean cache in same RUN: `RUN apt-get update && apt-get install -y curl && rm -rf /var/lib/apt/lists/*`
5. Use `.dockerignore` to exclude unnecessary files
6. Don't install debugging tools in production images

---

## AWS Interview Questions

**Q: Explain the difference between Security Groups and NACLs.**
A: Security Groups are stateful (return traffic automatically allowed), operate at the instance level, support only allow rules. NACLs (Network ACLs) are stateless (must explicitly allow both inbound and outbound), operate at the subnet level, support both allow and deny rules, and rules are evaluated in order by number.

**Q: What is the difference between horizontal and vertical scaling in AWS?**
A: Vertical scaling: increase instance size (t3.micro → m5.xlarge). Has downtime, limited ceiling. Horizontal scaling: add more instances behind a load balancer. No downtime, effectively unlimited. AWS Auto Scaling does horizontal scaling. Always design for horizontal scaling in production.

**Q: Explain S3 consistency model.**
A: Since December 2020, S3 provides strong read-after-write consistency for all operations (GET, PUT, LIST, DELETE). Previously, there was eventual consistency for overwrite PUTs and DELETEs. This means after a successful write, all subsequent reads will return the latest version.

**Q: What is the difference between an IAM Role and IAM User?**
A: IAM User: a persistent identity with long-term credentials (access key/password). For humans or legacy apps. IAM Role: a temporary identity assumed by AWS services, EC2 instances, Lambda functions, or federated users. Uses short-lived credentials (STS tokens). Roles are more secure (no static credentials) and are the preferred approach for applications running on AWS.

**Q: How does VPC peering work and what are its limitations?**
A: VPC Peering creates a private network connection between two VPCs allowing traffic to route via AWS backbone (not internet). Limitations: not transitive (A↔B and B↔C doesn't mean A↔C), CIDR ranges can't overlap, cross-region costs money, limited to VPC pairs. For complex multi-VPC networking, use AWS Transit Gateway instead.

---

## Prometheus & Grafana Interview Questions

**Q: Explain the Prometheus data model.**
A: Every metric is a time series identified by metric name + label set (key-value pairs). Time series is stored as (timestamp, float64 value) pairs. Metric types: Counter (monotonically increasing), Gauge (can go up/down), Histogram (distributions with buckets), Summary (client-side percentiles). Labels enable powerful multi-dimensional querying.

**Q: What is the difference between a Counter and a Gauge?**
A: Counter only increases (or resets to 0 on restart). Use for: requests processed, errors, bytes sent. Always use `rate()` or `increase()` to get meaningful values from counters. Gauge represents a value that can go up or down. Use for: current connections, memory usage, temperature. Use directly without `rate()`.

**Q: Explain `rate()` vs `irate()` in PromQL.**
A: `rate(counter[5m])` calculates per-second average over 5 minutes — smoothed, good for dashboards and alerts. `irate(counter[5m])` calculates per-second rate between the last two data points — instant rate, very spiky. Use `rate()` for alerting (avoids false alarms from spikes). Use `irate()` when you want to see instant changes.

**Q: What is a Recording Rule and when do you use it?**
A: Recording rules pre-compute expensive PromQL queries and store results as new time series. Use when: complex queries are slow, the same query is used in multiple dashboards/alerts, you need to downsample data for long-term storage.
```yaml
- record: job:http_request_rate:rate5m
  expr: sum(rate(http_requests_total[5m])) by (job)
```

**Q: How do you handle high cardinality in Prometheus?**
A: High cardinality (too many unique label value combinations) causes memory/storage issues. Solutions:
1. Avoid high-cardinality labels (user IDs, request IDs, email addresses)
2. Use `metric_relabel_configs` to drop or aggregate problematic labels
3. Use recording rules to pre-aggregate
4. Use `count_values()` instead of individual labels
5. Configure `--storage.tsdb.retention.size` to limit storage

**Q: What is Grafana provisioning?**
A: Provisioning lets you configure Grafana as code (no manual UI clicks): data sources, dashboards, alert rules, and plugins are defined in YAML files loaded at startup. Benefits: version control, reproducible deployments, CI/CD integration, no manual configuration drift. Files go in `/etc/grafana/provisioning/`.

---

## SRE-Specific Questions

**Q: What are SLIs, SLOs, and SLAs?**
A:
- **SLI (Service Level Indicator)**: A metric measuring service behavior. Example: request success rate, latency at 99th percentile, availability.
- **SLO (Service Level Objective)**: A target value for an SLI. Example: "99.9% of requests succeed", "p99 latency < 200ms". SLOs are internal targets.
- **SLA (Service Level Agreement)**: A contractual commitment with consequences if SLO is not met. Example: "99.9% uptime or customer gets a refund". SLAs are external commitments.

**Q: What is an error budget?**
A: Error budget = 100% - SLO. If SLO is 99.9%, error budget is 0.1% (about 43 minutes per month of allowed downtime). It balances reliability and innovation: as long as error budget remains, teams can release quickly. When budget is exhausted, focus shifts to reliability over new features.

**Q: How do you approach an incident?**
A: 
1. **Detect**: Alert triggers, monitor dashboards
2. **Triage**: Assess severity and user impact
3. **Communicate**: Notify stakeholders, update status page
4. **Investigate**: Check logs, metrics, recent deployments
5. **Mitigate**: Rollback, failover, disable feature flag — stop the bleeding first
6. **Resolve**: Fix root cause
7. **Post-mortem**: Blameless review within 48-72 hours, document timeline, root cause, action items

**Q: What is the difference between MTTR and MTBF?**
A:
- **MTBF (Mean Time Between Failures)**: Average time between incidents. Higher is better. Measures reliability.
- **MTTR (Mean Time To Recover)**: Average time to restore service after an incident. Lower is better. Measures resilience.
- Also: **MTTD (Mean Time To Detect)** — how quickly you find out about incidents (monitoring quality).

---

## Quick Reference Cheat Sheet

### Linux Commands
| Task | Command |
|------|---------|
| Check disk space | `df -hT` |
| Find large files | `find / -type f -size +1G` |
| Check open ports | `ss -tlnp` |
| Top CPU processes | `ps aux --sort=-%cpu \| head -10` |
| Top memory processes | `ps aux --sort=-%mem \| head -10` |
| Check system load | `uptime` |
| Check failed services | `systemctl --failed` |
| Follow logs | `journalctl -f -u servicename` |
| Check listening ports | `ss -tlnp` |
| Network connections | `ss -s` |

### Terraform Commands
| Task | Command |
|------|---------|
| Initialize | `terraform init` |
| Plan | `terraform plan -out=plan.tfplan` |
| Apply | `terraform apply plan.tfplan` |
| Destroy resource | `terraform destroy -target=resource` |
| Import resource | `terraform import resource.name id` |
| Force unlock state | `terraform force-unlock LOCK_ID` |
| Show state | `terraform state show resource` |
| Move resource | `terraform state mv old new` |

### Docker Commands
| Task | Command |
|------|---------|
| Build image | `docker build -t name:tag .` |
| Run container | `docker run -d --name name image` |
| View logs | `docker logs -f --tail 100 name` |
| Shell in container | `docker exec -it name bash` |
| Resource usage | `docker stats` |
| Remove stopped | `docker container prune` |
| Clean everything | `docker system prune -a` |
| Copy files | `docker cp name:/path /local` |

### AWS CLI Commands
| Task | Command |
|------|---------|
| List EC2 instances | `aws ec2 describe-instances --output table` |
| Get secret | `aws secretsmanager get-secret-value --secret-id name` |
| S3 sync | `aws s3 sync /local s3://bucket/` |
| CloudWatch logs | `aws logs tail /log-group --follow` |
| EKS kubeconfig | `aws eks update-kubeconfig --name cluster` |
| SSM session | `aws ssm start-session --target i-xxx` |

### PromQL Quick Reference
| Metric | Query |
|--------|-------|
| CPU usage % | `100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)` |
| Memory used % | `(1 - node_memory_MemAvailable_bytes/node_memory_MemTotal_bytes) * 100` |
| Disk used % | `(1 - node_filesystem_avail_bytes/node_filesystem_size_bytes) * 100` |
| Error rate % | `rate(errors_total[5m]) / rate(requests_total[5m]) * 100` |
| p95 latency | `histogram_quantile(0.95, rate(duration_bucket[5m]))` |

---

*Document version: 2026-05-09 | Coverage: Linux, Terraform, Docker, AWS, Prometheus, Grafana*
*Audience: Junior SRE/DevOps → Senior SRE/DevOps*
