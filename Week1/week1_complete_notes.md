# WEEK 1 — COMPLETE STUDY NOTES
### InfraThrone CoreOps Track — Full Resource with Labs, Practice & Interview Prep

---

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     WEEK 1 CURRICULUM MAP                                  │
│                                                                             │
│  Day 1 → Linux Fundamentals & File System                                  │
│  Day 2 → Users, Permissions & Process Management                           │
│  Day 3 → Networking Fundamentals — OSI Model & TCP/IP                     │
│  Day 4 → DNS, HTTP/HTTPS, Load Balancers & Firewalls                      │
│  Day 5 → Shell Scripting & Automation                                      │
│  Day 6 → Git & Version Control                                             │
│  Day 7 → Docker & Containers — Introduction                               │
│                                                                             │
│  Goal: Build rock-solid fundamentals for every DevOps topic that follows.  │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

# DAY 1 — LINUX FUNDAMENTALS & FILE SYSTEM

---

## SECTION 1.1: WHY LINUX FOR DEVOPS?

Linux powers over 96% of the world's servers, all major cloud platforms, and every Kubernetes node.
As a DevOps engineer you will live in the terminal — building pipelines, debugging outages,
managing infrastructure, and writing automation. Linux is not optional. It is the foundation.

```
┌─────────────────────────────────────────────────────────────────────────┐
│               WHERE LINUX RUNS IN A TYPICAL DEVOPS STACK               │
│                                                                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                 │
│  │  EC2 / GCE   │  │  Kubernetes  │  │  CI/CD Agent │                 │
│  │  Instances   │  │  Nodes       │  │  (Jenkins /  │                 │
│  │  (Ubuntu /   │  │  (Amazon     │  │   GitLab /   │                 │
│  │   Amazon     │  │   Linux /    │  │   GitHub     │                 │
│  │   Linux)     │  │   Ubuntu)    │  │   Actions)   │                 │
│  └──────────────┘  └──────────────┘  └──────────────┘                 │
│                                                                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                 │
│  │  Docker      │  │  Terraform   │  │  Ansible     │                 │
│  │  Containers  │  │  Runner      │  │  Control     │                 │
│  │  (Alpine /   │  │  (Linux VM)  │  │  Node        │                 │
│  │   Debian)    │  │              │  │              │                 │
│  └──────────────┘  └──────────────┘  └──────────────┘                 │
│                                                                         │
│  ALL of the above run on Linux. Master it once, use it everywhere.     │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## SECTION 1.2: LINUX FILE SYSTEM HIERARCHY (FHS)

The Linux File System Hierarchy Standard defines WHERE everything lives.
This is not random — every directory has a specific, intentional purpose.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    LINUX DIRECTORY TREE                                │
│                                                                         │
│  /  (root — the top of everything)                                     │
│  ├── bin/      Essential user binaries (ls, cp, mv, cat, bash)         │
│  ├── sbin/     System binaries (root-only: fdisk, iptables, mount)     │
│  ├── etc/      Configuration files (nginx.conf, /etc/hosts, cron)     │
│  ├── home/     User home directories (/home/ubuntu, /home/devops)      │
│  ├── root/     Root user's home directory                              │
│  ├── var/      Variable data — logs, spool, caches                     │
│  │   ├── log/  System logs (syslog, auth.log, nginx/access.log)       │
│  │   └── www/  Web server files                                        │
│  ├── tmp/      Temporary files — cleared on reboot                     │
│  ├── usr/      User programs and libraries                             │
│  │   ├── bin/  Non-essential user binaries                             │
│  │   └── lib/  Libraries for /usr/bin programs                        │
│  ├── lib/      Essential shared libraries (libc, kernel modules)       │
│  ├── proc/     Virtual FS — kernel & process info (/proc/cpuinfo)     │
│  ├── sys/      Virtual FS — hardware/device info                      │
│  ├── dev/      Device files (/dev/sda, /dev/null, /dev/tty)           │
│  ├── mnt/      Temporary mount points                                  │
│  ├── opt/      Optional/third-party software                           │
│  └── boot/     Bootloader files (vmlinuz, initrd, grub)               │
└─────────────────────────────────────────────────────────────────────────┘
```

### Key Directories You'll Use Daily as a DevOps Engineer

| Directory | What You'll Find There | Why It Matters |
|-----------|------------------------|----------------|
| `/etc/` | All config files | nginx, ssh, cron, hosts, resolv.conf |
| `/var/log/` | All log files | Debugging outages — your first stop |
| `/proc/` | Live kernel stats | CPU, memory, running processes |
| `/tmp/` | Temp files | Scripts drop artifacts here |
| `/home/` | User data | Where your .bashrc, .ssh/keys live |
| `/usr/local/bin/` | Custom tools | Where you install helm, kubectl, etc. |

---

## SECTION 1.3: ESSENTIAL LINUX COMMANDS — COMPLETE REFERENCE

### Navigation Commands

```bash
# Print working directory — where am I right now?
pwd
# Output: /home/ubuntu

# List directory contents
ls              # basic list
ls -l           # long format (permissions, owner, size, date)
ls -la          # long format including hidden files (starting with .)
ls -lh          # human-readable file sizes (K, M, G)
ls -lt          # sorted by modification time (newest first)
ls -lR          # recursive listing

# Change directory
cd /var/log             # absolute path
cd logs                 # relative path
cd ..                   # go up one level
cd ~                    # go to your home directory
cd -                    # go to previous directory (toggle)

# Create directories
mkdir mydir             # create single directory
mkdir -p a/b/c          # create nested directories (parents too)
mkdir -p /opt/myapp/{logs,config,data}  # create multiple subdirs at once
```

### File Operations

```bash
# Create files
touch file.txt          # create empty file or update timestamp
echo "hello" > file.txt # create file with content (overwrites)
echo "world" >> file.txt # append to file

# Copy files and directories
cp file.txt backup.txt          # copy file
cp -r /source/dir /dest/dir     # copy directory recursively
cp -rp /source/dir /dest/dir    # copy preserving permissions & timestamps

# Move and rename
mv file.txt newname.txt         # rename
mv file.txt /tmp/               # move to directory
mv /tmp/file.txt /opt/myapp/    # move to different location

# Delete
rm file.txt                     # delete file
rm -f file.txt                  # force delete (no prompt)
rm -r mydir/                    # delete directory recursively
rm -rf mydir/                   # force delete directory (DANGEROUS — no undo)

# Create symbolic links
ln -s /opt/myapp/config.yml /etc/myapp.yml   # symlink (like a shortcut)
ln /file1 /file2                              # hard link
```

### Viewing Files

```bash
# Display file content
cat file.txt            # print entire file
cat -n file.txt         # print with line numbers
head file.txt           # first 10 lines
head -n 50 file.txt     # first 50 lines
tail file.txt           # last 10 lines
tail -n 100 file.txt    # last 100 lines
tail -f /var/log/syslog # follow — stream new lines in real-time (ESSENTIAL for logs)
tail -F /var/log/syslog # follow even if file is rotated

# Pager — view large files
less /var/log/syslog    # scroll up/down (q to quit, /pattern to search)
more /etc/nginx/nginx.conf  # simple pager (space to scroll, q to quit)
```

### Searching & Text Processing

```bash
# Find files
find / -name "nginx.conf"                   # find by name
find /var/log -name "*.log"                 # find all .log files
find /home -type f -newer /tmp/ref.txt      # files newer than ref
find / -size +100M                          # files larger than 100MB
find /etc -type f -mtime -7                 # modified in last 7 days

# Search inside files (grep)
grep "error" /var/log/syslog                # find lines with "error"
grep -i "error" /var/log/syslog             # case-insensitive
grep -r "database_url" /etc/               # recursive search
grep -n "error" app.log                     # show line numbers
grep -c "error" app.log                     # count matching lines
grep -v "INFO" app.log                      # lines NOT matching (invert)
grep -A 3 "FATAL" app.log                   # 3 lines AFTER match (context)
grep -B 3 "FATAL" app.log                   # 3 lines BEFORE match
grep -E "error|warn|fatal" app.log          # extended regex (OR)

# Word count
wc -l file.txt          # count lines
wc -w file.txt          # count words
wc -c file.txt          # count bytes

# Sort and deduplicate
sort file.txt           # sort alphabetically
sort -n numbers.txt     # sort numerically
sort -r file.txt        # reverse sort
sort -u file.txt        # sort and remove duplicates
uniq file.txt           # remove adjacent duplicate lines (use with sort)
sort file.txt | uniq -c # count occurrences of each line

# Cut, awk, sed — text extraction
cut -d: -f1 /etc/passwd     # cut field 1 using : as delimiter
awk '{print $1}' file.txt   # print first column
awk -F: '{print $1}' /etc/passwd  # with custom delimiter
sed 's/old/new/g' file.txt  # substitute all occurrences
sed -n '10,20p' file.txt    # print lines 10-20
sed '/^#/d' config.txt      # delete comment lines

# Pipe — chain commands together
cat /var/log/syslog | grep "error" | tail -20
ps aux | grep nginx | grep -v grep
```

---

## SECTION 1.4: FILE SYSTEM NAVIGATION LAB

### Lab 1.4.1 — Explore Your System

```bash
# Step 1: Understand where you are
pwd
whoami
id

# Step 2: Explore key directories
ls -la /etc/ | head -20
ls -la /var/log/
ls -la /proc/ | head -20

# Step 3: Check disk usage
df -h           # disk free — shows all mounted filesystems
df -hT          # also shows filesystem type
du -sh /var/*   # disk usage of each item in /var
du -sh /var/log/*.log  # size of each log file

# Step 4: Find the largest files on the system
du -ah / --max-depth=3 2>/dev/null | sort -rh | head -20

# Step 5: Check what's mounted
mount | column -t
cat /proc/mounts
```

### Lab 1.4.2 — Build a Project Directory Structure

```bash
# Create a realistic application directory structure
mkdir -p /opt/myapp/{bin,config,logs,data,tmp,scripts}
mkdir -p /opt/myapp/data/{backups,uploads}
mkdir -p /opt/myapp/config/{nginx,app,certs}

# Verify the structure
find /opt/myapp -type d | sort

# Create some config files
cat > /opt/myapp/config/app/app.env << 'EOF'
APP_NAME=myapp
APP_ENV=production
APP_PORT=8080
DB_HOST=localhost
DB_PORT=5432
DB_NAME=myapp_db
LOG_LEVEL=INFO
EOF

# Create a README
cat > /opt/myapp/README.md << 'EOF'
# MyApp

Production application deployment.

## Directory Structure
- bin/      → application binaries
- config/   → configuration files
- logs/     → application logs
- data/     → persistent data
- scripts/  → operational scripts
EOF

# Verify files
cat /opt/myapp/config/app/app.env
ls -la /opt/myapp/
```

---

# DAY 2 — USERS, PERMISSIONS & PROCESS MANAGEMENT

---

## SECTION 2.1: LINUX USERS AND GROUPS

### Understanding the Permission Model

Linux uses a Discretionary Access Control (DAC) model. Every file has:
- An **owner** (a user)
- An **owning group** (a group)
- **Permissions** for owner, group, and everyone else (others)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    LINUX FILE PERMISSION ANATOMY                       │
│                                                                         │
│   -  r w x   r w -   r - -                                            │
│   │  │ │ │   │ │ │   │ │ │                                            │
│   │  └─┴─┘   └─┴─┘   └─┴─┘                                            │
│   │  Owner   Group   Others                                            │
│   │                                                                     │
│   └── File type:                                                       │
│        -  = regular file                                               │
│        d  = directory                                                  │
│        l  = symbolic link                                              │
│        b  = block device (/dev/sda)                                   │
│        c  = character device (/dev/tty)                               │
│        p  = named pipe                                                 │
│        s  = socket                                                     │
│                                                                         │
│   r = read    (4)                                                      │
│   w = write   (2)                                                      │
│   x = execute (1)                                                      │
│                                                                         │
│   Example: rwxr-xr-- = 754                                            │
│   Owner: rwx = 4+2+1 = 7 (full)                                       │
│   Group: r-x = 4+0+1 = 5 (read+execute)                              │
│   Others: r-- = 4+0+0 = 4 (read only)                                │
└─────────────────────────────────────────────────────────────────────────┘
```

### Permission Octal Reference Table

| Octal | Binary | Permissions | Meaning |
|-------|--------|-------------|---------|
| 0 | 000 | `---` | No permissions |
| 1 | 001 | `--x` | Execute only |
| 2 | 010 | `-w-` | Write only |
| 3 | 011 | `-wx` | Write + Execute |
| 4 | 100 | `r--` | Read only |
| 5 | 101 | `r-x` | Read + Execute |
| 6 | 110 | `rw-` | Read + Write |
| 7 | 111 | `rwx` | Full permissions |

### Common Permission Patterns

| Octal | Pattern | Use Case |
|-------|---------|----------|
| `644` | `-rw-r--r--` | Config files — owner edits, others read |
| `600` | `-rw-------` | SSH private keys, secrets |
| `755` | `-rwxr-xr-x` | Scripts and executables |
| `700` | `-rwx------` | Private scripts |
| `664` | `-rw-rw-r--` | Shared group files |
| `777` | `-rwxrwxrwx` | NEVER use in production (insecure) |
| `400` | `-r--------` | Read-only secrets (e.g., SSL certs) |

---

## SECTION 2.2: MANAGING PERMISSIONS AND OWNERSHIP

```bash
# Change permissions
chmod 644 config.yml         # octal notation
chmod u+x script.sh          # symbolic: add execute for owner
chmod go-w sensitive.key     # symbolic: remove write for group and others
chmod a+r public.html        # symbolic: add read for all
chmod -R 755 /opt/myapp/bin/ # recursive

# Change ownership
chown ubuntu file.txt             # change owner
chown ubuntu:devops file.txt      # change owner and group
chown -R ubuntu:devops /opt/myapp # recursive ownership change

# Change group
chgrp devops file.txt

# View permissions
ls -la /etc/nginx/nginx.conf
stat /etc/nginx/nginx.conf        # detailed file metadata

# Special permissions
chmod u+s /usr/bin/passwd  # setuid — run as file owner (not caller)
chmod g+s /opt/shared/     # setgid — new files inherit group
chmod +t /tmp/             # sticky bit — only owner can delete their files
```

### User and Group Management

```bash
# View users
cat /etc/passwd             # all users (username:x:uid:gid:comment:home:shell)
getent passwd ubuntu        # get specific user
id ubuntu                   # show user's UID, GID, groups
id                          # current user

# Create users
useradd -m -s /bin/bash -c "DevOps Engineer" devops_user
useradd -m -G sudo,docker devops_user    # add to groups at creation
passwd devops_user                        # set password

# Modify users
usermod -aG docker ubuntu       # add ubuntu user to docker group (-a = append!)
usermod -s /bin/bash devops_user  # change shell
usermod -L devops_user          # lock account
usermod -U devops_user          # unlock account

# Delete users
userdel devops_user             # delete user (keep home dir)
userdel -r devops_user          # delete user AND home dir

# Groups
cat /etc/group                  # all groups
groupadd devops                 # create group
groupdel devops                 # delete group
groups ubuntu                   # list groups for a user

# Switch users
su - devops_user                # switch to user (full login shell)
sudo -u devops_user command     # run command as different user

# sudo configuration
visudo                          # ALWAYS use visudo to edit sudoers
cat /etc/sudoers.d/             # drop-in sudoers files

# Typical sudoers entry:
# ubuntu  ALL=(ALL:ALL) NOPASSWD:ALL
# devops_user  ALL=(ALL) NOPASSWD:/usr/bin/systemctl, /usr/bin/docker
```

---

## SECTION 2.3: PROCESS MANAGEMENT

### Understanding Linux Processes

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    LINUX PROCESS LIFECYCLE                             │
│                                                                         │
│   CREATED (fork)                                                       │
│       │                                                                 │
│       ▼                                                                 │
│   RUNNING ─────── CPU available ──────► RUNNING (on CPU)              │
│       │                                                                 │
│       ├── Waiting for I/O ──────────► BLOCKED (sleeping)             │
│       │       │                              │                         │
│       │       └──────── I/O done ────────────┘                        │
│       │                                                                 │
│       └── TERMINATED ──────────────► ZOMBIE (waiting for parent)      │
│                                              │                         │
│                                    Parent calls wait() → cleaned up    │
│                                                                         │
│   Every process has:                                                   │
│   • PID  — Process ID (unique)                                        │
│   • PPID — Parent Process ID                                          │
│   • UID  — User who owns it                                           │
│   • Priority (nice value: -20 highest to +19 lowest)                  │
└─────────────────────────────────────────────────────────────────────────┘
```

### Process Commands

```bash
# View processes
ps aux                          # all processes, detailed
ps aux | grep nginx             # find nginx processes
ps -ef                          # full format listing
ps -ef --forest                 # show process tree

# Interactive process viewer
top                             # real-time view (press q to quit)
htop                            # better top (install: apt install htop)
# htop keys: F6=sort, F9=kill, F3=search, F4=filter, F5=tree

# Process tree
pstree                          # tree view
pstree -p                       # show PIDs too

# Find process PID
pgrep nginx                     # get PID by name
pidof nginx                     # same
pgrep -la nginx                 # PID and command line

# Kill processes
kill 1234                       # send SIGTERM (15) — graceful shutdown
kill -9 1234                    # send SIGKILL — force kill (no cleanup)
kill -HUP 1234                  # SIGHUP — reload config (many services)
killall nginx                   # kill all processes named nginx
pkill -f "python app.py"        # kill by matching command line

# Signal reference
# SIGTERM (15) — polite kill, process can clean up
# SIGKILL (9)  — immediate kill, cannot be caught or ignored
# SIGHUP  (1)  — hang up, often reloads config
# SIGINT  (2)  — interrupt (Ctrl+C)
# SIGUSR1/2    — user-defined signals (often used by apps for log rotation)

# Background jobs
command &               # run in background
jobs                    # list background jobs
fg %1                   # bring job 1 to foreground
bg %1                   # send job 1 to background
nohup command &         # run immune to hangup (survives terminal close)
disown -h %1            # disown a background job

# Screen / tmux for persistent sessions
screen -S mysession     # start named screen session
screen -ls              # list sessions
screen -r mysession     # reattach
# Ctrl+A D              # detach from screen

tmux new -s mysession   # start tmux session
tmux ls                 # list sessions
tmux attach -t mysession # reattach
# Ctrl+B D              # detach from tmux
```

### Monitoring System Resources

```bash
# CPU and memory
free -h                 # memory usage (human readable)
vmstat 1 5              # virtual memory stats every 1s, 5 times
mpstat 1                # CPU stats per core
iostat -x 1             # I/O stats
sar -u 1 5              # system activity reporter

# Disk
df -h                   # disk free (filesystem level)
du -sh /var/log/        # disk usage (directory level)
lsblk                   # list block devices
fdisk -l                # partition table
iostat -d 1             # disk I/O statistics

# Network
ss -tlnp                # socket statistics (listening TCP + process)
ss -tunp                # TCP and UDP + process names
netstat -tlnp           # older alternative (install: net-tools)
ifconfig                # interface config (older)
ip addr                 # modern: show IP addresses
ip route                # routing table

# Load average
uptime                  # uptime and load averages
# Load avg: 1.5 3.2 4.1 = last 1min / 5min / 15min
# On a 4-core system: load of 4.0 = 100% utilization

# Watch a command repeatedly
watch -n 2 'ps aux | grep nginx'   # repeat every 2 seconds
```

---

## SECTION 2.4: SYSTEMD — SERVICE MANAGEMENT

Modern Linux systems use `systemd` as the init system and service manager.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    SYSTEMD ARCHITECTURE                                │
│                                                                         │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │  systemd (PID 1) — the first process, parent of everything       │  │
│  │                                                                  │  │
│  │  Manages:                                                        │  │
│  │  • Services (nginx.service, docker.service, postgresql.service)  │  │
│  │  • Targets (multi-user.target, graphical.target)                │  │
│  │  • Timers (cron replacement)                                     │  │
│  │  • Sockets, Mounts, Swap                                        │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                                                         │
│  Unit files location:                                                  │
│  /lib/systemd/system/        — package-installed services              │
│  /etc/systemd/system/        — custom/override services (your files)   │
│  /etc/systemd/system/*.d/    — drop-in override directories            │
└─────────────────────────────────────────────────────────────────────────┘
```

```bash
# Service control
systemctl start nginx           # start service
systemctl stop nginx            # stop service
systemctl restart nginx         # stop then start
systemctl reload nginx          # reload config without stopping
systemctl status nginx          # detailed status
systemctl enable nginx          # start on boot
systemctl disable nginx         # don't start on boot
systemctl is-active nginx       # check if running
systemctl is-enabled nginx      # check if enabled on boot

# View all services
systemctl list-units --type=service
systemctl list-units --type=service --state=running
systemctl list-units --type=service --state=failed

# Logs
journalctl -u nginx                     # all logs for nginx
journalctl -u nginx -f                  # follow live
journalctl -u nginx --since "1 hour ago"
journalctl -u nginx --since "2024-01-01" --until "2024-01-02"
journalctl -u nginx -n 100              # last 100 lines
journalctl --disk-usage                  # how much space logs use
journalctl --vacuum-size=500M           # clean logs older than limit
```

### Writing a Custom Systemd Service

```bash
# Create a service unit file
cat > /etc/systemd/system/myapp.service << 'EOF'
[Unit]
Description=My Application Service
Documentation=https://myapp.example.com/docs
After=network.target postgresql.service
Wants=postgresql.service

[Service]
Type=simple
User=ubuntu
Group=ubuntu
WorkingDirectory=/opt/myapp
EnvironmentFile=/opt/myapp/config/app/app.env
ExecStart=/opt/myapp/bin/myapp --config /opt/myapp/config/app/app.env
ExecReload=/bin/kill -HUP $MAINPID
Restart=always
RestartSec=5
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
EOF

# Reload systemd, enable and start
systemctl daemon-reload
systemctl enable --now myapp
systemctl status myapp
journalctl -u myapp -f
```

---

## SECTION 2.5: DAY 2 LABS

### Lab 2.5.1 — Permission Hardening

```bash
# Step 1: Create a service user (no shell, no home dir)
useradd -r -s /sbin/nologin -c "App Service Account" appuser

# Step 2: Create app directories with correct permissions
mkdir -p /opt/secureapp/{bin,config,logs,data}
chown -R appuser:appuser /opt/secureapp
chmod 750 /opt/secureapp
chmod 640 /opt/secureapp/config

# Step 3: Create a secret config file
cat > /opt/secureapp/config/secrets.env << 'EOF'
DB_PASSWORD=supersecretpassword
API_KEY=abc123xyz
JWT_SECRET=myjwtsecret
EOF

# Step 4: Lock down the secrets file
chown appuser:appuser /opt/secureapp/config/secrets.env
chmod 600 /opt/secureapp/config/secrets.env

# Step 5: Verify permissions
ls -la /opt/secureapp/
ls -la /opt/secureapp/config/

# Step 6: Try to read as another user (should fail)
sudo -u ubuntu cat /opt/secureapp/config/secrets.env
# Expected: Permission denied

# Step 7: Try to read as appuser (should work)
sudo -u appuser cat /opt/secureapp/config/secrets.env
```

### Lab 2.5.2 — Process Monitoring & Investigation

```bash
# Step 1: Find what processes are listening on ports
ss -tlnp

# Step 2: Find the process using port 80
ss -tlnp | grep :80
lsof -i :80

# Step 3: Check process resource usage
ps aux --sort=-%cpu | head -10    # top CPU consumers
ps aux --sort=-%mem | head -10    # top memory consumers

# Step 4: Simulate a busy process and monitor it
# Run a CPU-heavy loop in background
yes > /dev/null &
BG_PID=$!
echo "Started background process: $BG_PID"

# Watch it in top (it should show high CPU)
top -p $BG_PID

# Kill it gracefully
kill $BG_PID
echo "Process killed"

# Step 5: Check system load
uptime
cat /proc/loadavg
```

---

# DAY 3 — NETWORKING FUNDAMENTALS: OSI MODEL & TCP/IP

---

## SECTION 3.1: WHY NETWORKING IS YOUR #1 OUTAGE SKILL

In DevOps, the most common production outages are networking-related:
- Service can't reach database → TCP/firewall issue
- Users getting timeouts → DNS or load balancer issue
- Deploy works locally, fails in cloud → Security group / VPC misconfiguration
- Microservice A can't talk to B → Service mesh / network policy issue

```
┌─────────────────────────────────────────────────────────────────────────┐
│              TOP 5 REAL OUTAGE TYPES — ALL NETWORKING                 │
│                                                                         │
│  1. DNS resolution failure  → service unreachable by name              │
│  2. Security group blocks   → TCP connection refused / timeout         │
│  3. TLS cert expired        → HTTPS connections fail                  │
│  4. Routing misconfiguration → packets going to wrong place            │
│  5. Port not listening      → process crashed or wrong config          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## SECTION 3.2: THE OSI MODEL — DEEP DIVE

The OSI (Open Systems Interconnection) model is a conceptual framework describing how data
travels from one application to another across a network. It has 7 layers.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    THE OSI MODEL — 7 LAYERS                           │
│                                                                         │
│  Layer 7 — APPLICATION                                                 │
│  ─────────────────────────────────────────────────────────────────    │
│  What apps use to communicate. Protocols: HTTP, HTTPS, FTP, SMTP,     │
│  DNS, SSH, SNMP, POP3, IMAP. This is where your web app lives.        │
│                                                                         │
│  Layer 6 — PRESENTATION                                                │
│  ─────────────────────────────────────────────────────────────────    │
│  Data translation, encryption/decryption, compression.                 │
│  SSL/TLS lives here. Converts app data to network format.             │
│  JPEG, MPEG, ASCII encoding. Also called the "syntax layer".          │
│                                                                         │
│  Layer 5 — SESSION                                                     │
│  ─────────────────────────────────────────────────────────────────    │
│  Establishes, maintains, and terminates sessions between apps.        │
│  Handles authentication, reconnection. NetBIOS, RPC, NFS.             │
│                                                                         │
│  Layer 4 — TRANSPORT                                                   │
│  ─────────────────────────────────────────────────────────────────    │
│  End-to-end communication. Protocols: TCP (reliable), UDP (fast).     │
│  Handles: segmentation, flow control, error detection, port numbers.   │
│  PORT numbers live here (80, 443, 22, 5432, 6379)                    │
│                                                                         │
│  Layer 3 — NETWORK                                                     │
│  ─────────────────────────────────────────────────────────────────    │
│  Routing between networks. Protocol: IP (IPv4, IPv6).                 │
│  IP addresses live here. Routers operate at this layer.               │
│  Handles: logical addressing, path selection                           │
│                                                                         │
│  Layer 2 — DATA LINK                                                   │
│  ─────────────────────────────────────────────────────────────────    │
│  Node-to-node delivery within same network segment.                   │
│  MAC addresses live here. Ethernet, Wi-Fi, ARP, VLANs, Switches.     │
│  Handles: framing, MAC addressing, error detection (CRC)              │
│                                                                         │
│  Layer 1 — PHYSICAL                                                    │
│  ─────────────────────────────────────────────────────────────────    │
│  Actual physical transmission. Cables, fiber, radio waves, voltage.   │
│  Bits (0s and 1s) on a wire. Hubs, repeaters, network cables.        │
└─────────────────────────────────────────────────────────────────────────┘
```

### The OSI Model Mnemonic

```
Please  →  Physical    (Layer 1)
Do      →  Data Link   (Layer 2)
Not     →  Network     (Layer 3)
Throw   →  Transport   (Layer 4)
Sausage →  Session     (Layer 5)
Pizza   →  Presentation(Layer 6)
Away    →  Application (Layer 7)
```

### OSI in Action — What Happens When You Type google.com

```
┌─────────────────────────────────────────────────────────────────────────┐
│         REQUEST JOURNEY: Browser → google.com                         │
│                                                                         │
│  YOUR MACHINE (Encapsulation — wrapping layers around data)            │
│  ─────────────────────────────────────────────────────────             │
│                                                                         │
│  L7: Browser creates HTTP GET request                                  │
│      GET / HTTP/1.1\r\nHost: google.com\r\n...                        │
│                          │                                             │
│                          ▼                                             │
│  L6: TLS encrypts the HTTP data                                        │
│      [Encrypted payload]                                               │
│                          │                                             │
│                          ▼                                             │
│  L5: TCP session established (SYN-SYN/ACK-ACK handshake)              │
│                          │                                             │
│                          ▼                                             │
│  L4: TCP wraps data in segment                                         │
│      [Source port: 54321] [Dest port: 443] [Seq/Ack] [Data]          │
│                          │                                             │
│                          ▼                                             │
│  L3: IP wraps segment in packet                                        │
│      [Source IP: 192.168.1.5] [Dest IP: 142.250.80.46] [Segment]    │
│                          │                                             │
│                          ▼                                             │
│  L2: Ethernet wraps packet in frame                                    │
│      [Src MAC: aa:bb:cc] [Dst MAC: router MAC] [Packet] [CRC]        │
│                          │                                             │
│                          ▼                                             │
│  L1: Bits transmitted on wire/Wi-Fi                                    │
│      010110101010001100110101...                                        │
│                                                                         │
│  GOOGLE'S SERVER (Decapsulation — unwrapping)                         │
│  ──────────────────────────────────────────────                        │
│  L1 → L2 → L3 → L4 → L5 → L6 → L7                                   │
│  Bits → Frame → Packet → Segment → Session → Decrypt → HTTP request  │
└─────────────────────────────────────────────────────────────────────────┘
```

### DevOps Relevance — Which Layer Is the Problem?

| Symptom | Likely Layer | Tool to Diagnose |
|---------|-------------|-----------------|
| Physical cable unplugged | L1 | `ip link`, check cable |
| ARP not resolving | L2 | `arping`, `arp -n` |
| Can't ping across networks | L3 | `ping`, `traceroute`, `ip route` |
| Connection refused on port | L4 | `telnet`, `nc`, `ss -tlnp` |
| TLS cert error | L6 | `openssl s_client`, `curl -v` |
| HTTP 404/500 errors | L7 | `curl`, app logs |

---

## SECTION 3.3: TCP/IP MODEL

The TCP/IP model is the practical implementation — 4 layers that map to the OSI model:

```
┌─────────────────────────────────────────────────────────────────────────┐
│         TCP/IP MODEL vs OSI MODEL                                      │
│                                                                         │
│  TCP/IP          OSI                What Lives Here                    │
│  ─────────────   ─────────────────  ─────────────────────────────────  │
│  Application  ←→ Application (7)    HTTP, HTTPS, DNS, SSH, FTP       │
│               ←→ Presentation (6)   TLS/SSL, encoding                 │
│               ←→ Session (5)        sessions, RPC                     │
│  Transport    ←→ Transport (4)      TCP, UDP                          │
│  Internet     ←→ Network (3)       IP, ICMP, ARP (sort of)           │
│  Link         ←→ Data Link (2)     Ethernet, Wi-Fi                   │
│               ←→ Physical (1)      cables, signals                   │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## SECTION 3.4: TCP — THE RELIABLE PROTOCOL

TCP (Transmission Control Protocol) guarantees:
- **Ordered delivery** — packets arrive in the correct sequence
- **Error detection** — corrupted packets are retransmitted
- **Flow control** — sender won't overwhelm receiver
- **Congestion control** — backs off when network is saturated

### TCP Three-Way Handshake

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    TCP 3-WAY HANDSHAKE                                 │
│                                                                         │
│  Client                                Server                          │
│    │                                     │                             │
│    │──── SYN (seq=x) ──────────────────►│                             │
│    │    "I want to connect"              │                             │
│    │                                     │                             │
│    │◄─── SYN-ACK (seq=y, ack=x+1) ──────│                             │
│    │    "OK, I'm ready, acknowledge me"  │                             │
│    │                                     │                             │
│    │──── ACK (ack=y+1) ────────────────►│                             │
│    │    "Got it, connection established" │                             │
│    │                                     │                             │
│    │════════ DATA TRANSFER ══════════════│                             │
│    │                                     │                             │
│    │──── FIN ──────────────────────────►│  (closing connection)       │
│    │◄─── ACK ───────────────────────────│                             │
│    │◄─── FIN ───────────────────────────│                             │
│    │──── ACK ──────────────────────────►│                             │
│                                                                         │
│  WHY THIS MATTERS FOR DEVOPS:                                          │
│  • "Connection refused" = server not listening on that port            │
│  • "Connection timeout" = firewall/security-group blocking SYN         │
│  • Half-open connections = SYN flood attack or server overload         │
└─────────────────────────────────────────────────────────────────────────┘
```

### TCP Connection States

```bash
# View TCP connection states
ss -s      # summary of socket states
ss -tan    # all TCP sockets with state

# Common states:
# LISTEN      → server is waiting for connections
# ESTABLISHED → active connection
# TIME_WAIT   → connection closed, waiting for delayed packets
# CLOSE_WAIT  → remote closed, we haven't yet
# SYN_SENT    → we sent SYN, waiting for SYN-ACK
# SYN_RECV    → received SYN, waiting for ACK

# Too many TIME_WAIT can indicate:
# → High connection rate (need connection pooling or keepalive)
```

---

## SECTION 3.5: UDP — THE FAST PROTOCOL

UDP (User Datagram Protocol):
- **No handshake** — just send and forget
- **No ordering** — packets may arrive out of order
- **No retransmission** — if lost, it's gone
- **Very low overhead** — fast

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    TCP vs UDP COMPARISON                               │
│                                                                         │
│  Feature          TCP                    UDP                           │
│  ─────────────    ─────────────────────  ─────────────────────────     │
│  Connection       Required (3-way HS)   None                          │
│  Reliability      Guaranteed            Not guaranteed                 │
│  Ordering         Guaranteed            Not guaranteed                 │
│  Speed            Slower (overhead)     Faster                        │
│  Error detection  Yes + retransmit      Yes, no retransmit            │
│  Use cases        HTTP, SSH, DB, Email  DNS, video, gaming, DHCP      │
│                                                                         │
│  DEVOPS KEY FACT:                                                      │
│  • Your app APIs use TCP                                              │
│  • DNS uses UDP (port 53) — but falls back to TCP for large queries   │
│  • Monitoring (Prometheus) uses TCP; syslog can use UDP               │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## SECTION 3.6: IP ADDRESSING — SUBNETS & CIDR

### IPv4 Address Structure

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    IPv4 ADDRESS ANATOMY                                │
│                                                                         │
│  192   .  168   .    1   .   50                                       │
│  ─────────────────────────────────                                     │
│  8 bits   8 bits   8 bits  8 bits  = 32 bits total                    │
│                                                                         │
│  With subnet mask /24 (255.255.255.0):                                │
│                                                                         │
│  Network portion: 192.168.1   (first 24 bits)                         │
│  Host portion:           .50  (last 8 bits)                           │
│                                                                         │
│  Network address:  192.168.1.0   (all host bits = 0)                  │
│  Broadcast address: 192.168.1.255 (all host bits = 1)                 │
│  Usable hosts:     192.168.1.1 to 192.168.1.254 = 254 hosts          │
└─────────────────────────────────────────────────────────────────────────┘
```

### CIDR Notation — Quick Reference

| CIDR | Subnet Mask | Total IPs | Usable Hosts | Common Use |
|------|-------------|-----------|--------------|------------|
| /8 | 255.0.0.0 | 16,777,216 | ~16.7M | Class A (10.0.0.0/8) |
| /16 | 255.255.0.0 | 65,536 | 65,534 | VPC CIDR block |
| /24 | 255.255.255.0 | 256 | 254 | Typical subnet |
| /28 | 255.255.255.240 | 16 | 14 | Small subnet |
| /30 | 255.255.255.252 | 4 | 2 | Point-to-point link |
| /32 | 255.255.255.255 | 1 | 1 | Single host (security group rule) |

### Private IP Ranges (RFC 1918)

```
┌─────────────────────────────────────────────────────────────────────────┐
│               PRIVATE (NON-ROUTABLE) IP RANGES                        │
│                                                                         │
│  10.0.0.0/8         →  10.0.0.0  –  10.255.255.255  (16M hosts)      │
│  172.16.0.0/12      →  172.16.0.0 – 172.31.255.255  (1M hosts)       │
│  192.168.0.0/16     →  192.168.0.0 – 192.168.255.255 (65K hosts)     │
│                                                                         │
│  These are used inside:                                                │
│  • Your home network (192.168.x.x)                                    │
│  • AWS VPCs (10.0.0.0/16 is standard)                                 │
│  • Kubernetes pod networks (172.16.0.0/16 or 10.244.0.0/16)          │
│  • Docker bridge network (172.17.0.0/16)                              │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## SECTION 3.7: NETWORKING COMMANDS LAB

### Lab 3.7.1 — Network Investigation Toolkit

```bash
# ─── LAYER 3: IP and Routing ───────────────────────────────────────────

# View your IP addresses
ip addr show
ip addr show eth0          # specific interface

# View routing table
ip route show
ip route get 8.8.8.8       # which interface/gateway to reach 8.8.8.8

# Test connectivity
ping -c 4 8.8.8.8                  # ping Google DNS (4 packets)
ping -c 4 google.com               # ping by hostname (tests DNS too)
ping -I eth0 8.8.8.8               # ping via specific interface

# Traceroute — see each hop
traceroute 8.8.8.8                 # trace path (UDP)
traceroute -T 8.8.8.8             # trace using TCP
mtr 8.8.8.8                        # live traceroute (great for intermittent issues)

# ─── LAYER 4: TCP/UDP Ports ────────────────────────────────────────────

# What's listening on what port?
ss -tlnp                           # TCP listeners with process names
ss -ulnp                           # UDP listeners
ss -tlnp | grep :80                # filter for port 80

# Test if a port is open (from client side)
telnet 10.0.0.1 5432               # test PostgreSQL port (Ctrl+] then quit)
nc -zv 10.0.0.1 5432               # netcat — cleaner test
nc -zv 10.0.0.1 80 443 8080        # test multiple ports

# Test TCP connection with timeout
timeout 3 bash -c "cat < /dev/null > /dev/tcp/10.0.0.1/5432"
echo $?   # 0 = success, 1 = failed

# ─── LAYER 7: DNS ──────────────────────────────────────────────────────

# Basic DNS lookup
nslookup google.com                # query default DNS server
dig google.com                     # detailed DNS query
dig google.com A                   # only A records
dig google.com MX                  # mail records
dig +short google.com              # just the IP
dig @8.8.8.8 google.com           # query specific DNS server

# Reverse DNS lookup
dig -x 8.8.8.8                    # what hostname is 8.8.8.8?

# Check /etc/hosts and DNS config
cat /etc/hosts
cat /etc/resolv.conf               # DNS servers
cat /etc/nsswitch.conf             # name resolution order

# ─── FULL DIAGNOSTIC SEQUENCE ──────────────────────────────────────────

# "I can't reach service X" — systematic diagnosis:

# 1. Am I on the network?
ip addr show
ping -c 2 192.168.1.1    # ping gateway

# 2. Can I reach the internet?
ping -c 2 8.8.8.8        # ping by IP (bypass DNS)

# 3. Does DNS work?
dig +short google.com

# 4. Can I reach the service by IP?
nc -zv 10.0.0.5 5432

# 5. Can I reach the service by hostname?
nc -zv postgres.prod.internal 5432

# 6. What does the service log say?
journalctl -u myapp -n 50
```

---

# DAY 4 — DNS, HTTP/HTTPS, LOAD BALANCERS & FIREWALLS

---

## SECTION 4.1: DNS — THE INTERNET'S PHONEBOOK

DNS (Domain Name System) translates human-readable hostnames to IP addresses.

### How DNS Resolution Works

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    DNS RESOLUTION FLOW                                 │
│                                                                         │
│  Browser types: www.example.com                                        │
│                          │                                             │
│                          ▼                                             │
│  1. Check LOCAL CACHE (OS DNS cache)                                   │
│     → Found? Use it. Not found? Continue.                              │
│                          │                                             │
│                          ▼                                             │
│  2. Check /etc/hosts                                                   │
│     → Found? Use it. Not found? Continue.                              │
│                          │                                             │
│                          ▼                                             │
│  3. Ask RECURSIVE RESOLVER (your ISP or 8.8.8.8)                      │
│     → It has cache? Use it. Not found? Continue.                       │
│                          │                                             │
│                          ▼                                             │
│  4. Resolver asks ROOT NAMESERVER (13 root servers)                    │
│     → "Who handles .com domains?"                                      │
│     → Returns: ".com TLD nameserver addresses"                         │
│                          │                                             │
│                          ▼                                             │
│  5. Resolver asks .com TLD NAMESERVER                                  │
│     → "Who handles example.com?"                                       │
│     → Returns: "example.com authoritative nameserver"                  │
│                          │                                             │
│                          ▼                                             │
│  6. Resolver asks AUTHORITATIVE NAMESERVER for example.com             │
│     → "What's the IP for www.example.com?"                            │
│     → Returns: "93.184.216.34 (TTL: 3600)"                            │
│                          │                                             │
│                          ▼                                             │
│  7. Resolver caches answer, returns to browser                         │
│  8. Browser connects to 93.184.216.34:443                              │
└─────────────────────────────────────────────────────────────────────────┘
```

### DNS Record Types

| Record | Purpose | Example |
|--------|---------|---------|
| `A` | Hostname → IPv4 address | `api.example.com → 93.184.1.1` |
| `AAAA` | Hostname → IPv6 address | `api.example.com → 2001:db8::1` |
| `CNAME` | Hostname → Hostname (alias) | `www.example.com → example.com` |
| `MX` | Mail server for domain | `example.com → mail.example.com` |
| `TXT` | Text data (SPF, DKIM, verification) | `"v=spf1 include:... ~all"` |
| `NS` | Authoritative nameservers for domain | `ns1.route53.com` |
| `PTR` | IP → Hostname (reverse DNS) | `1.1.184.93.in-addr.arpa → api.example.com` |
| `SOA` | Start of authority — zone metadata | Primary NS, admin email, serial |
| `SRV` | Service location (host + port) | `_http._tcp.example.com → 10 0 80 web.example.com` |

### DNS Caching and TTL

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    TTL — TIME TO LIVE                                  │
│                                                                         │
│  TTL is the cache lifetime of a DNS record (in seconds).               │
│                                                                         │
│  TTL = 300   → cached for 5 minutes                                    │
│  TTL = 3600  → cached for 1 hour                                       │
│  TTL = 86400 → cached for 24 hours                                     │
│                                                                         │
│  DEVOPS CRITICAL:                                                      │
│  Before a planned IP change / migration:                               │
│  • Lower TTL to 60-300 BEFORE the change (24-48 hours ahead)          │
│  • Make the IP change                                                  │
│  • Old cached records expire within 60-300 seconds                    │
│  • Users switch over quickly                                           │
│                                                                         │
│  If you change IP without lowering TTL first:                          │
│  • Users cached with old IP for up to 24 hours                        │
│  • Outage appears to affect "some users" randomly                      │
│  • Very hard to debug!                                                 │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## SECTION 4.2: HTTP/HTTPS — THE WEB PROTOCOL

### HTTP Request/Response Cycle

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    HTTP REQUEST / RESPONSE                             │
│                                                                         │
│  CLIENT REQUEST:                                                       │
│  ─────────────────────────────────────────────────────────            │
│  GET /api/users HTTP/1.1                ← Method + Path + Version     │
│  Host: api.example.com                  ← Required header             │
│  Authorization: Bearer eyJhbGci...      ← Auth token                  │
│  Content-Type: application/json         ← Body format                 │
│  User-Agent: curl/7.81.0                ← Client info                 │
│  Accept: application/json               ← Acceptable response format  │
│  [blank line]                                                          │
│  {"key": "value"}                       ← Body (for POST/PUT)         │
│                                                                         │
│  SERVER RESPONSE:                                                      │
│  ─────────────────────────────────────────────────────────            │
│  HTTP/1.1 200 OK                        ← Version + Status code       │
│  Content-Type: application/json         ← Body format                 │
│  Content-Length: 1234                   ← Body size                   │
│  X-Request-ID: abc123                   ← Correlation ID              │
│  Cache-Control: max-age=300             ← Caching directives          │
│  [blank line]                                                          │
│  {"users": [...]}                       ← Response body               │
└─────────────────────────────────────────────────────────────────────────┘
```

### HTTP Methods

| Method | Purpose | Idempotent? | Has Body? |
|--------|---------|-------------|-----------|
| GET | Retrieve resource | Yes | No |
| POST | Create resource | No | Yes |
| PUT | Replace resource | Yes | Yes |
| PATCH | Partially update | No | Yes |
| DELETE | Delete resource | Yes | No |
| HEAD | Like GET but no body | Yes | No |
| OPTIONS | Get allowed methods | Yes | No |

### HTTP Status Codes — Complete Reference

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    HTTP STATUS CODES                                   │
│                                                                         │
│  2xx — SUCCESS                                                         │
│  200 OK            → standard success                                  │
│  201 Created       → resource created (POST/PUT success)              │
│  204 No Content    → success, no body to return (DELETE)              │
│                                                                         │
│  3xx — REDIRECTION                                                     │
│  301 Moved Permanently  → bookmark the new URL                        │
│  302 Found              → temporary redirect                          │
│  304 Not Modified       → use your cached version                     │
│                                                                         │
│  4xx — CLIENT ERROR (your request is wrong)                           │
│  400 Bad Request        → malformed request syntax                    │
│  401 Unauthorized       → not authenticated                           │
│  403 Forbidden          → authenticated but not authorized            │
│  404 Not Found          → resource doesn't exist                      │
│  405 Method Not Allowed → wrong HTTP method                          │
│  408 Request Timeout    → client too slow                             │
│  409 Conflict           → state conflict (duplicate creation)         │
│  429 Too Many Requests  → rate limited                                │
│                                                                         │
│  5xx — SERVER ERROR (server is failing)                               │
│  500 Internal Server Error  → unhandled exception in app              │
│  502 Bad Gateway           → upstream service returned invalid response│
│  503 Service Unavailable   → app is down or overloaded               │
│  504 Gateway Timeout       → upstream service too slow               │
│                                                                         │
│  DEVOPS KEY:                                                           │
│  502/503/504 = your app/backend is the problem, not the client        │
│  401/403     = auth/RBAC misconfiguration                             │
│  503 spike   = likely need to scale or fix a crash loop               │
└─────────────────────────────────────────────────────────────────────────┘
```

### HTTPS and TLS

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    TLS HANDSHAKE (simplified)                          │
│                                                                         │
│  Client                                Server                          │
│    │                                     │                             │
│    │── ClientHello (TLS versions, ciphers)──►│                        │
│    │                                     │                             │
│    │◄─ ServerHello (chosen cipher) ──────│                             │
│    │◄─ Certificate (public key + chain) ─│                             │
│    │◄─ ServerHelloDone ──────────────────│                             │
│    │                                     │                             │
│    │  Client verifies cert:              │                             │
│    │  • Is it signed by trusted CA?      │                             │
│    │  • Is it expired?                   │                             │
│    │  • Does hostname match?             │                             │
│    │                                     │                             │
│    │── ClientKeyExchange (session key)──►│                             │
│    │── ChangeCipherSpec ────────────────►│                             │
│    │── Finished (encrypted) ────────────►│                             │
│    │                                     │                             │
│    │◄─ ChangeCipherSpec ─────────────────│                             │
│    │◄─ Finished (encrypted) ─────────────│                             │
│    │                                     │                             │
│    │══════════ ENCRYPTED DATA ═══════════│                             │
│                                                                         │
│  DEVOPS CERT COMMANDS:                                                 │
│  openssl x509 -in cert.pem -text -noout     # view cert details       │
│  openssl s_client -connect host:443         # test TLS connection      │
│  openssl verify -CAfile ca.pem cert.pem     # verify cert chain       │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## SECTION 4.3: LOAD BALANCERS

Load balancers distribute traffic across multiple backend servers.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    LOAD BALANCER ARCHITECTURE                          │
│                                                                         │
│  Internet / Clients                                                    │
│         │                                                               │
│         ▼                                                               │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │              LOAD BALANCER (Layer 4 or Layer 7)                 │   │
│  │                                                                 │   │
│  │  Health checks: GET /health every 10s                          │   │
│  │  Algorithm: Round-Robin / Least-Connections / IP-Hash          │   │
│  └─────────────────┬──────────────────┬──────────────┬────────────┘   │
│                    │                  │              │                 │
│                    ▼                  ▼              ▼                 │
│           ┌────────────┐    ┌────────────┐  ┌────────────┐            │
│           │  Server 1  │    │  Server 2  │  │  Server 3  │            │
│           │ 10.0.1.1   │    │ 10.0.1.2   │  │ 10.0.1.3   │            │
│           │ :8080      │    │ :8080      │  │ :8080      │            │
│           └────────────┘    └────────────┘  └────────────┘            │
└─────────────────────────────────────────────────────────────────────────┘
```

### Load Balancer Types

| Type | Layer | What It Sees | Use Case |
|------|-------|-------------|----------|
| **L4 (Network LB)** | Transport | IP + Port only | TCP/UDP raw traffic, no HTTP awareness |
| **L7 (Application LB)** | Application | Full HTTP request (headers, URL, cookies) | HTTP routing, auth, SSL termination |

### Load Balancing Algorithms

| Algorithm | How It Works | Best For |
|-----------|-------------|----------|
| Round Robin | Each request goes to next server | Equal capacity servers |
| Weighted RR | Servers with higher weight get more traffic | Different capacity servers |
| Least Connections | Send to server with fewest active connections | Long-lived connections |
| IP Hash | Same client always goes to same server | Session persistence |
| Random | Random server selection | Simple high-performance scenarios |

### Health Checks

```bash
# Nginx load balancer config example
upstream myapp_backend {
    least_conn;                     # algorithm
    server 10.0.1.1:8080 weight=3; # weight
    server 10.0.1.2:8080;
    server 10.0.1.3:8080 backup;   # only used if others fail
}

server {
    listen 80;
    location / {
        proxy_pass http://myapp_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_connect_timeout 5s;
        proxy_read_timeout 30s;
    }
    
    # Health check endpoint
    location /lb-health {
        access_log off;
        return 200 "healthy\n";
    }
}
```

---

## SECTION 4.4: FIREWALLS — IPTABLES & UFW

### IPTables Concepts

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    IPTABLES PACKET FLOW                                │
│                                                                         │
│  Incoming packet                                                       │
│       │                                                                 │
│       ▼                                                                 │
│  PREROUTING (nat)  → DNAT (change destination)                        │
│       │                                                                 │
│       ├──────────────────────────────────────────────                  │
│       │                                                                 │
│  LOCAL PROCESS?   YES → INPUT chain → Local app                       │
│       │                                                                 │
│       NO → FORWARD chain → Outgoing interface                         │
│                                                                         │
│  Outgoing packet:                                                       │
│  Local app → OUTPUT chain → POSTROUTING (nat) → SNAT → Wire          │
│                                                                         │
│  Three default tables:                                                 │
│  • filter  → INPUT, FORWARD, OUTPUT (most common)                     │
│  • nat     → PREROUTING, POSTROUTING (address translation)            │
│  • mangle  → packet modification                                       │
│                                                                         │
│  Default policies: ACCEPT or DROP                                      │
└─────────────────────────────────────────────────────────────────────────┘
```

```bash
# View current rules
iptables -L -v -n              # list all rules
iptables -L -v -n --line-numbers  # with line numbers

# Allow SSH (port 22)
iptables -A INPUT -p tcp --dport 22 -j ACCEPT

# Allow established connections
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# Allow HTTP/HTTPS
iptables -A INPUT -p tcp --dport 80 -j ACCEPT
iptables -A INPUT -p tcp --dport 443 -j ACCEPT

# Drop everything else (default deny)
iptables -P INPUT DROP
iptables -P FORWARD DROP
iptables -P OUTPUT ACCEPT

# Allow from specific IP only
iptables -A INPUT -s 10.0.0.5 -p tcp --dport 5432 -j ACCEPT

# Save rules
iptables-save > /etc/iptables/rules.v4

# UFW (Uncomplicated Firewall) — easier interface
ufw enable
ufw allow ssh                   # allow SSH
ufw allow 80/tcp                # allow HTTP
ufw allow 443/tcp               # allow HTTPS
ufw allow from 10.0.0.0/24 to any port 5432  # restrict postgres
ufw deny 23                     # deny telnet
ufw status verbose              # view rules
ufw delete allow 80/tcp         # remove a rule
```

---

# DAY 5 — SHELL SCRIPTING & AUTOMATION

---

## SECTION 5.1: BASH SCRIPTING FUNDAMENTALS

Shell scripting is one of the most powerful tools in a DevOps engineer's toolkit.
Every piece of automation — deployment scripts, health checks, cron jobs, CI steps —
starts with bash.

### Script Structure

```bash
#!/bin/bash
# ─── Script Header ────────────────────────────────────────────────────
# Name:        deploy.sh
# Description: Deploy application to production
# Author:      DevOps Team
# Created:     2024-01-01
# Usage:       ./deploy.sh <environment> <version>
# ──────────────────────────────────────────────────────────────────────

# Exit on any error, undefined variable, or pipe failure (BEST PRACTICE)
set -euo pipefail

# Debug mode (uncomment to trace every command)
# set -x

# ─── Constants ────────────────────────────────────────────────────────
readonly APP_NAME="myapp"
readonly DEPLOY_DIR="/opt/myapp"
readonly LOG_FILE="/var/log/deploy.log"

# ─── Functions ────────────────────────────────────────────────────────
log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*" | tee -a "$LOG_FILE"
}

error() {
    echo "[ERROR] $*" >&2
    exit 1
}

# ─── Main Logic ───────────────────────────────────────────────────────
main() {
    log "Starting deployment"
    # ... your logic here
}

main "$@"
```

### Variables

```bash
# Variable assignment (NO spaces around =)
NAME="devops"
COUNT=42
PI=3.14

# Using variables
echo "Hello, $NAME"
echo "Count is: ${COUNT}"

# Command substitution
TODAY=$(date +%Y-%m-%d)
HOSTNAME=$(hostname)
FILES=$(ls /etc/*.conf | wc -l)

# Arithmetic
NUM=$((10 + 5))
NUM=$((NUM * 2))
echo $((2 ** 8))     # 256

# String operations
STR="Hello World"
echo ${#STR}                    # length: 11
echo ${STR,,}                   # lowercase: hello world
echo ${STR^^}                   # uppercase: HELLO WORLD
echo ${STR:6}                   # substring from index 6: World
echo ${STR:6:3}                 # substring 3 chars from index 6: Wor
echo ${STR/World/DevOps}        # replace first: Hello DevOps
echo ${STR//l/L}                # replace all: HeLLo WorLd

# Default values
DB_HOST="${DB_HOST:-localhost}"           # use default if not set
DB_PORT="${DB_PORT:-5432}"
APP_ENV="${APP_ENV:?'APP_ENV must be set!'}"  # error if not set

# Arrays
SERVERS=("web1" "web2" "web3")
echo ${SERVERS[0]}              # web1
echo ${SERVERS[@]}              # all elements
echo ${#SERVERS[@]}             # array length
SERVERS+=("web4")               # append

for server in "${SERVERS[@]}"; do
    echo "Processing: $server"
done
```

### Conditionals

```bash
# If-else
if [[ $COUNT -gt 10 ]]; then
    echo "Count is greater than 10"
elif [[ $COUNT -eq 10 ]]; then
    echo "Count is exactly 10"
else
    echo "Count is less than 10"
fi

# String comparisons
if [[ "$ENV" == "production" ]]; then
    echo "Running in production"
fi

if [[ -z "$TOKEN" ]]; then     # empty string
    error "TOKEN is not set"
fi

if [[ -n "$TOKEN" ]]; then     # non-empty string
    echo "Token is set"
fi

# File tests
if [[ -f "/etc/nginx/nginx.conf" ]]; then
    echo "Nginx config exists"
fi

if [[ -d "/opt/myapp" ]]; then
    echo "App directory exists"
fi

if [[ -r "$FILE" ]]; then echo "File is readable"; fi
if [[ -w "$FILE" ]]; then echo "File is writable"; fi
if [[ -x "$FILE" ]]; then echo "File is executable"; fi
if [[ -s "$FILE" ]]; then echo "File is not empty"; fi
if [[ ! -f "$FILE" ]]; then echo "File does not exist"; fi

# Compound conditions
if [[ -f "$FILE" && -r "$FILE" ]]; then
    echo "File exists and is readable"
fi

if [[ "$ENV" == "prod" || "$ENV" == "production" ]]; then
    echo "This is production"
fi

# Exit code check
if ! systemctl is-active --quiet nginx; then
    error "Nginx is not running!"
fi

# Case statement
case "$ENV" in
    dev|development)
        echo "Development environment"
        LOG_LEVEL="DEBUG"
        ;;
    staging)
        echo "Staging environment"
        LOG_LEVEL="INFO"
        ;;
    prod|production)
        echo "Production environment"
        LOG_LEVEL="WARN"
        ;;
    *)
        error "Unknown environment: $ENV"
        ;;
esac
```

### Loops

```bash
# For loop — list
for item in one two three; do
    echo "Item: $item"
done

# For loop — range
for i in {1..10}; do
    echo "Number: $i"
done

# For loop — C style
for ((i=0; i<5; i++)); do
    echo "Index: $i"
done

# For loop — array
SERVERS=("web1" "web2" "web3")
for server in "${SERVERS[@]}"; do
    echo "Pinging $server..."
    ping -c 1 "$server" &>/dev/null && echo "  UP" || echo "  DOWN"
done

# For loop — command output
for file in $(find /etc -name "*.conf" -type f); do
    echo "Config: $file"
done

# While loop
COUNT=0
while [[ $COUNT -lt 5 ]]; do
    echo "Count: $COUNT"
    COUNT=$((COUNT + 1))
done

# Read file line by line
while IFS= read -r line; do
    echo "Line: $line"
done < /etc/hosts

# Until loop
TRIES=0
until ping -c 1 myserver &>/dev/null; do
    TRIES=$((TRIES + 1))
    if [[ $TRIES -ge 10 ]]; then
        error "Server unreachable after $TRIES attempts"
    fi
    echo "Attempt $TRIES — waiting..."
    sleep 5
done
echo "Server is up!"
```

### Functions

```bash
# Function definition
check_service() {
    local service_name="$1"         # local variable
    local expected_port="${2:-80}"  # local with default
    
    if systemctl is-active --quiet "$service_name"; then
        echo "✓ $service_name is running"
        return 0
    else
        echo "✗ $service_name is NOT running"
        return 1
    fi
}

# Call function
check_service nginx 80
check_service postgresql 5432

# Capture return value
if check_service nginx; then
    echo "Proceeding with deployment"
else
    error "Nginx must be running first"
fi

# Function with error handling
wait_for_port() {
    local host="$1"
    local port="$2"
    local timeout="${3:-30}"
    local interval=2
    local elapsed=0
    
    echo "Waiting for $host:$port to be available..."
    
    while [[ $elapsed -lt $timeout ]]; do
        if nc -z "$host" "$port" 2>/dev/null; then
            echo "$host:$port is available (${elapsed}s)"
            return 0
        fi
        sleep $interval
        elapsed=$((elapsed + interval))
    done
    
    echo "Timeout waiting for $host:$port after ${timeout}s"
    return 1
}

wait_for_port localhost 5432 60
```

---

## SECTION 5.2: REAL-WORLD BASH SCRIPTS

### Script 1: Health Check Script

```bash
#!/bin/bash
# health_check.sh — Check all critical services

set -euo pipefail

SERVICES=("nginx" "postgresql" "redis")
FAILED=()

for service in "${SERVICES[@]}"; do
    if systemctl is-active --quiet "$service"; then
        echo "[OK]   $service"
    else
        echo "[FAIL] $service"
        FAILED+=("$service")
    fi
done

if [[ ${#FAILED[@]} -gt 0 ]]; then
    echo ""
    echo "FAILED SERVICES: ${FAILED[*]}"
    exit 1
fi

echo ""
echo "All services healthy"
exit 0
```

### Script 2: Log Analyzer

```bash
#!/bin/bash
# log_analyzer.sh — Analyze nginx access log

LOG_FILE="${1:-/var/log/nginx/access.log}"
THRESHOLD="${2:-100}"

if [[ ! -f "$LOG_FILE" ]]; then
    echo "Log file not found: $LOG_FILE"
    exit 1
fi

echo "=== Log Analysis: $LOG_FILE ==="
echo ""
echo "Top 10 IPs:"
awk '{print $1}' "$LOG_FILE" | sort | uniq -c | sort -rn | head -10

echo ""
echo "Status Code Distribution:"
awk '{print $9}' "$LOG_FILE" | sort | uniq -c | sort -rn

echo ""
echo "Top Requested URLs:"
awk '{print $7}' "$LOG_FILE" | sort | uniq -c | sort -rn | head -10

echo ""
echo "IPs exceeding $THRESHOLD requests:"
awk '{print $1}' "$LOG_FILE" | sort | uniq -c | sort -rn | \
    awk -v threshold="$THRESHOLD" '$1 > threshold {print $0}'
```

### Script 3: Deployment Script

```bash
#!/bin/bash
# deploy.sh — Production deployment script

set -euo pipefail

APP_NAME="myapp"
DEPLOY_DIR="/opt/myapp"
BACKUP_DIR="/opt/backups"
APP_USER="appuser"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
LOG_FILE="/var/log/deploy_${TIMESTAMP}.log"

log() { echo "[$(date '+%H:%M:%S')] $*" | tee -a "$LOG_FILE"; }
error() { log "ERROR: $*"; exit 1; }

# Validate input
[[ $# -lt 1 ]] && error "Usage: $0 <version>"
VERSION="$1"

log "Starting deployment of $APP_NAME v$VERSION"

# Pre-flight checks
log "Running pre-flight checks..."
[[ -d "$DEPLOY_DIR" ]] || error "Deploy directory missing: $DEPLOY_DIR"
systemctl is-active --quiet nginx || error "Nginx is not running"

# Backup current version
log "Backing up current deployment..."
mkdir -p "$BACKUP_DIR"
tar -czf "$BACKUP_DIR/${APP_NAME}_${TIMESTAMP}.tar.gz" "$DEPLOY_DIR"
log "Backup created: $BACKUP_DIR/${APP_NAME}_${TIMESTAMP}.tar.gz"

# Deploy new version
log "Deploying version $VERSION..."
# ... your deploy steps here ...

# Run health check
log "Running health check..."
sleep 5
if ! curl -sf http://localhost/health &>/dev/null; then
    log "Health check FAILED — rolling back!"
    tar -xzf "$BACKUP_DIR/${APP_NAME}_${TIMESTAMP}.tar.gz" -C / 
    systemctl restart "$APP_NAME"
    error "Deployment failed, rolled back to previous version"
fi

log "Deployment of $APP_NAME v$VERSION completed successfully!"
```

---

## SECTION 5.3: CRON JOBS

```bash
# View crontab
crontab -l              # list jobs for current user
crontab -e              # edit crontab (opens in $EDITOR)
crontab -r              # REMOVE all cron jobs (dangerous!)
sudo crontab -l -u ubuntu  # list another user's crontab

# Crontab format:
# ┌──────────── minute (0-59)
# │  ┌─────────── hour (0-23)
# │  │  ┌──────────── day of month (1-31)
# │  │  │  ┌─────────── month (1-12)
# │  │  │  │  ┌──────────── day of week (0=Sun, 6=Sat)
# │  │  │  │  │
# *  *  *  *  *  command

# Examples:
0 2 * * *       /opt/scripts/backup.sh        # every day at 2:00 AM
*/5 * * * *     /opt/scripts/health_check.sh  # every 5 minutes
0 9 * * 1       /opt/scripts/weekly_report.sh # every Monday at 9 AM
0 0 1 * *       /opt/scripts/monthly.sh       # 1st of month midnight
@reboot         /opt/scripts/startup.sh       # at system boot
@daily          /opt/scripts/cleanup.sh       # once a day (midnight)

# Redirect cron output to log
0 2 * * * /opt/scripts/backup.sh >> /var/log/backup.log 2>&1

# System-wide cron directories
ls /etc/cron.daily/     # scripts run daily
ls /etc/cron.hourly/    # scripts run hourly
ls /etc/cron.weekly/    # scripts run weekly
ls /etc/cron.monthly/   # scripts run monthly
```

---

# DAY 6 — GIT & VERSION CONTROL

---

## SECTION 6.1: GIT FUNDAMENTALS

Git is a distributed version control system. Understanding Git is non-negotiable for DevOps —
it is the foundation of every CI/CD pipeline.

### Git Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    GIT STORAGE AREAS                                   │
│                                                                         │
│  ┌─────────────┐   git add    ┌─────────────┐  git commit ┌─────────┐ │
│  │  Working    │ ──────────► │   Staging   │ ──────────► │  Local  │ │
│  │  Directory  │             │   Area      │             │  Repo   │ │
│  │             │ ◄────────── │  (Index)    │ ◄────────── │  (.git) │ │
│  │  (your      │  git restore│             │  git reset  │         │ │
│  │   files)    │             │             │             │         │ │
│  └─────────────┘             └─────────────┘             └────┬────┘ │
│                                                                │       │
│                                                          git push      │
│                                                                │       │
│                                                          ┌─────▼────┐  │
│                                                          │  Remote  │  │
│                                                          │  Repo    │  │
│                                                          │ (GitHub) │  │
│                                                          └──────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
```

### Essential Git Commands

```bash
# ─── Setup ────────────────────────────────────────────────────────────
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
git config --global core.editor "vim"
git config --global init.defaultBranch main
git config --list           # view all config

# ─── Start a project ──────────────────────────────────────────────────
git init                    # initialize new repo
git init myproject          # create and initialize in new directory
git clone https://github.com/user/repo.git  # clone remote repo
git clone --depth=1 https://...             # shallow clone (faster CI)

# ─── Daily workflow ───────────────────────────────────────────────────
git status                  # what changed?
git diff                    # show unstaged changes (working vs staging)
git diff --staged           # show staged changes (staging vs last commit)
git add file.txt            # stage specific file
git add .                   # stage all changes
git add -p                  # interactive staging (choose hunks)
git commit -m "feat: add user authentication"
git commit --amend          # modify last commit message/content (ONLY if not pushed)

# ─── Branching ────────────────────────────────────────────────────────
git branch                  # list local branches
git branch -a               # list all branches (including remote)
git branch -r               # list remote branches
git branch feature/auth     # create branch
git checkout feature/auth   # switch to branch
git checkout -b feature/auth  # create AND switch (shorthand)
git switch feature/auth     # modern way to switch branches
git switch -c feature/auth  # create and switch (modern)

# Merge and rebase
git merge feature/auth      # merge branch into current
git merge --no-ff feature/auth  # merge preserving branch history
git rebase main             # reapply commits on top of main (cleaner history)
git rebase -i HEAD~3        # interactive rebase — squash/edit last 3 commits

# Delete branches
git branch -d feature/auth  # delete branch (safe — won't delete unmerged)
git branch -D feature/auth  # force delete
git push origin --delete feature/auth  # delete remote branch

# ─── Remote operations ────────────────────────────────────────────────
git remote -v               # list remotes
git remote add origin https://github.com/user/repo.git
git remote add upstream https://github.com/original/repo.git  # for forks
git fetch origin            # fetch all remote changes (don't merge)
git pull origin main        # fetch + merge
git pull --rebase origin main  # fetch + rebase (cleaner)
git push origin main        # push local commits to remote
git push -u origin feature/auth  # push and set upstream tracking

# ─── History ──────────────────────────────────────────────────────────
git log                     # full commit log
git log --oneline           # compact log
git log --oneline --graph   # visual branch graph
git log --oneline -20       # last 20 commits
git log --author="John"     # filter by author
git log --since="2 weeks ago"
git show abc1234            # show specific commit
git blame file.txt          # who changed each line and when

# ─── Undoing Changes ──────────────────────────────────────────────────
git restore file.txt            # discard working dir changes (IRREVERSIBLE)
git restore --staged file.txt   # unstage file (keep working changes)
git reset HEAD~1                # undo last commit, keep changes staged
git reset --soft HEAD~1         # undo commit, keep changes staged
git reset --hard HEAD~1         # undo commit AND discard changes (DANGEROUS)
git revert abc1234              # create new commit that undoes abc1234 (SAFE for pushed commits)

# ─── Stash ────────────────────────────────────────────────────────────
git stash                       # save current changes and clean working dir
git stash push -m "WIP: auth feature"  # named stash
git stash list                  # list all stashes
git stash pop                   # apply top stash and remove it
git stash apply stash@{1}       # apply specific stash
git stash drop stash@{0}        # delete a stash
git stash branch feature/wip    # create branch from stash

# ─── Tags ─────────────────────────────────────────────────────────────
git tag v1.0.0                  # create lightweight tag
git tag -a v1.0.0 -m "Release 1.0.0"  # annotated tag (preferred)
git tag -l                      # list tags
git push origin v1.0.0          # push specific tag
git push origin --tags          # push all tags
```

---

## SECTION 6.2: BRANCHING STRATEGIES

### GitFlow (Feature Branch Workflow)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    GITFLOW BRANCHING MODEL                             │
│                                                                         │
│  main (production-ready code only)                                     │
│  ──────────────────────────────────────────────────────►              │
│       •──────────────────────────────•         •                      │
│       │ v1.0.0                        │ v2.0.0  │ v2.1.0              │
│       │                               │                               │
│  develop (integration branch)         │                               │
│  ─────────────────────────────────────┼───────────────────────────►   │
│       •──────────•──────────────•─────•                               │
│       │          │              │                                      │
│  feature/auth    │    feature/payment                                 │
│  ──────────────►•│              │                                      │
│                  │    feature/  │                                      │
│             feature/ui  reports│                                      │
│             ─────────────────►•│                                      │
│                                │                                      │
│  hotfix/security-patch          │                                      │
│  ──────────────────────────────►•                                     │
│  (from main, merges to main AND develop)                              │
│                                                                        │
│  Branch types:                                                         │
│  main      → always deployable, production code                       │
│  develop   → integration branch, latest completed features            │
│  feature/* → new features (from develop, merge to develop)           │
│  release/* → stabilization for a release (from develop)              │
│  hotfix/*  → urgent production fixes (from main, merge to both)      │
└─────────────────────────────────────────────────────────────────────────┘
```

### Trunk-Based Development (Modern/CD-friendly)

```
┌─────────────────────────────────────────────────────────────────────────┐
│               TRUNK-BASED DEVELOPMENT                                  │
│                                                                         │
│  main / trunk (direct commits or very short-lived branches)            │
│  ──────────────────────────────────────────────────────────►           │
│   • ──•─────────•──────────•──────────•──────────•──────►             │
│       │         │                                                       │
│  feature-A      feature-B                                              │
│  (1-2 days max) (1-2 days max)                                        │
│                                                                         │
│  Rules:                                                                │
│  • Commit to main/trunk at least once daily                           │
│  • Features hidden behind feature flags if not ready                   │
│  • Short-lived branches (< 2 days)                                    │
│  • CI runs on every commit                                             │
│  • No release branches — you release from trunk                       │
│                                                                         │
│  WHY: Avoids "integration hell", enables true continuous delivery      │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## SECTION 6.3: GIT BEST PRACTICES

### Commit Message Convention (Conventional Commits)

```
┌─────────────────────────────────────────────────────────────────────────┐
│              CONVENTIONAL COMMIT FORMAT                                │
│                                                                         │
│  <type>(<scope>): <short description>                                  │
│                                                                         │
│  [optional body]                                                       │
│                                                                         │
│  [optional footer]                                                     │
│                                                                         │
│  Types:                                                                │
│  feat     → new feature                                                │
│  fix      → bug fix                                                    │
│  docs     → documentation only                                         │
│  style    → formatting, no logic change                               │
│  refactor → code restructure, no feature/fix                          │
│  test     → adding tests                                               │
│  chore    → tooling, build, CI changes                                │
│  perf     → performance improvement                                    │
│  ci       → CI/CD pipeline changes                                    │
│                                                                         │
│  Examples:                                                             │
│  feat(auth): add JWT refresh token support                            │
│  fix(api): handle null response from payment service                  │
│  docs(readme): update deployment instructions                         │
│  chore(deps): upgrade axios to 1.4.0                                  │
│  feat!: redesign API (breaking change — ! means breaking)             │
└─────────────────────────────────────────────────────────────────────────┘
```

### .gitignore Best Practices

```bash
# Create a comprehensive .gitignore
cat > .gitignore << 'EOF'
# OS files
.DS_Store
Thumbs.db
.directory

# Secrets and credentials — NEVER commit these!
*.env
.env
.env.local
.env.production
*.key
*.pem
*.p12
secrets/
credentials/

# Logs
*.log
logs/
*.log.*

# Build artifacts
build/
dist/
*.pyc
__pycache__/
node_modules/
.terraform/
*.tfstate
*.tfstate.backup

# IDE files
.idea/
.vscode/
*.swp
*.swo

# Test artifacts
.coverage
htmlcov/
.pytest_cache/
EOF

# Check what would be ignored
git check-ignore -v someFile
git status --ignored

# If you accidentally committed secrets:
# 1. Remove from history
git filter-branch --force --index-filter \
  'git rm --cached --ignore-unmatch path/to/secret.env' \
  --prune-empty --tag-name-filter cat -- --all
# 2. Force push (you must notify team!)
# 3. ROTATE ALL CREDENTIALS — assume they are compromised!
```

---

## SECTION 6.4: GIT LABS

### Lab 6.4.1 — Complete Git Workflow

```bash
# Initialize a new project
mkdir my-devops-project && cd my-devops-project
git init
git config user.email "you@example.com"
git config user.name "Your Name"

# Create initial structure
mkdir -p src tests docs
cat > src/app.py << 'EOF'
def hello():
    return "Hello, DevOps!"
EOF

cat > README.md << 'EOF'
# My DevOps Project
A sample project to practice Git workflows.
EOF

# First commit
git add .
git commit -m "chore: initial project structure"

# Create feature branch
git checkout -b feature/add-greet-function

# Add feature
cat > src/greet.py << 'EOF'
def greet(name):
    if not name:
        raise ValueError("Name cannot be empty")
    return f"Hello, {name}! Welcome to DevOps."
EOF

# Add test
cat > tests/test_greet.py << 'EOF'
from src.greet import greet

def test_greet_basic():
    assert greet("Alice") == "Hello, Alice! Welcome to DevOps."

def test_greet_empty():
    try:
        greet("")
        assert False
    except ValueError:
        pass
EOF

git add .
git commit -m "feat(greet): add greeting function with validation"

# Go back to main and merge
git checkout main
git merge --no-ff feature/add-greet-function -m "Merge feature/add-greet-function"
git log --oneline --graph

# Tag the release
git tag -a v1.0.0 -m "Release 1.0.0 — Initial version with greeting"
git log --oneline
```

### Lab 6.4.2 — Conflict Resolution

```bash
# Create scenario with conflict
git checkout -b branch-A
echo "Version A changes" >> README.md
git add README.md
git commit -m "feat: branch A changes"

git checkout main
git checkout -b branch-B
echo "Version B changes" >> README.md
git add README.md
git commit -m "feat: branch B changes"

# Merge branch-A into main
git checkout main
git merge branch-A

# Try to merge branch-B (will conflict)
git merge branch-B
# ERROR: Merge conflict in README.md

# View the conflict markers
cat README.md
# <<<<<<< HEAD
# Version A changes
# =======
# Version B changes
# >>>>>>> branch-B

# Resolve manually — edit the file to keep what you want
# Remove the conflict markers
# Then:
git add README.md
git commit -m "chore: resolve merge conflict between branch-A and branch-B"

git log --oneline --graph
```

---

# DAY 7 — DOCKER & CONTAINERS INTRODUCTION

---

## SECTION 7.1: WHY CONTAINERS?

The classic DevOps problem: "It works on my machine."

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    THE PROBLEM CONTAINERS SOLVE                        │
│                                                                         │
│  Before Containers:                                                    │
│  ─────────────────────────────────────────────────────────            │
│  Developer's laptop: Python 3.9, Postgres 13, Redis 6, Ubuntu 20     │
│  Staging server:     Python 3.7, Postgres 11, Redis 5, CentOS 7      │
│  Production server:  Python 3.6, Postgres 10, Redis 4, RHEL 7        │
│                                                                         │
│  Result: App works on laptop, breaks on staging, catastrophe on prod   │
│                                                                         │
│  With Containers:                                                      │
│  ─────────────────────────────────────────────────────────            │
│  Container carries:                                                    │
│  • Your code                                                           │
│  • Your runtime (Python 3.9, Node 18, Java 17)                       │
│  • Your dependencies (pip packages, node_modules)                     │
│  • System libraries                                                    │
│  • Config                                                              │
│                                                                         │
│  The container runs identically on:                                    │
│  ✓ Developer laptop                                                    │
│  ✓ CI/CD runner                                                        │
│  ✓ Staging server                                                      │
│  ✓ Production server                                                   │
│  ✓ Any Kubernetes node                                                 │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## SECTION 7.2: CONTAINERS vs VIRTUAL MACHINES

```
┌─────────────────────────────────────────────────────────────────────────┐
│              VMs vs CONTAINERS ARCHITECTURE                            │
│                                                                         │
│  VIRTUAL MACHINE                  CONTAINER                            │
│  ──────────────────────────────   ───────────────────────────────      │
│  ┌────────────────────────────┐   ┌────────────────────────────┐      │
│  │  App A   │  App B  │ App C │   │  App A   │  App B  │ App C │      │
│  ├──────────┼─────────┼───────┤   ├──────────┼─────────┼───────┤      │
│  │ Guest OS │ Guest OS│ Guest │   │ Bins/Libs│Bins/Libs│Bins/  │      │
│  │ (Linux)  │ (Linux) │  OS   │   │          │         │ Libs  │      │
│  ├──────────┴─────────┴───────┤   ├──────────────────────────────┤    │
│  │     Hypervisor             │   │         Container Runtime    │    │
│  │  (VMware / VirtualBox /    │   │      (Docker / containerd)   │    │
│  │   KVM / Hyper-V)           │   ├──────────────────────────────┤    │
│  ├────────────────────────────┤   │         Host OS (Linux)      │    │
│  │      Host OS (Linux)       │   ├──────────────────────────────┤    │
│  ├────────────────────────────┤   │         Hardware             │    │
│  │         Hardware           │   └──────────────────────────────┘    │
│  └────────────────────────────┘                                        │
│                                                                         │
│  VM: Full OS per app (GBs)        Container: Shared OS kernel (MBs)  │
│  Slow startup (minutes)           Fast startup (milliseconds)         │
│  Strong isolation (separate OS)   Process-level isolation              │
│  Good for: different OS needed    Good for: app isolation + portability│
└─────────────────────────────────────────────────────────────────────────┘
```

### How Containers Work — Linux Primitives

Containers are not magic. They use three Linux kernel features:

```
┌─────────────────────────────────────────────────────────────────────────┐
│               LINUX FEATURES THAT POWER CONTAINERS                    │
│                                                                         │
│  1. NAMESPACES — What the container can SEE                            │
│  ─────────────────────────────────────────────────────────            │
│  pid namespace  → container has its own process tree (PID 1 = app)    │
│  net namespace  → container has its own network interfaces             │
│  mnt namespace  → container has its own filesystem view               │
│  uts namespace  → container has its own hostname                       │
│  ipc namespace  → isolated inter-process communication                 │
│  user namespace → UID mapping (container root ≠ host root)           │
│                                                                         │
│  2. CGROUPS (Control Groups) — What the container can USE             │
│  ─────────────────────────────────────────────────────────            │
│  Limit and account for:                                                │
│  • CPU (0.5 cores, 25% of a CPU)                                      │
│  • Memory (512MB hard limit)                                           │
│  • Disk I/O (100MB/s max)                                             │
│  • Network bandwidth                                                   │
│                                                                         │
│  3. UNION FILE SYSTEM (OverlayFS) — Layered filesystem                │
│  ─────────────────────────────────────────────────────────            │
│  Docker images are stacked read-only layers.                           │
│  Container adds a writable layer on top.                               │
│  Layers are shared between containers (efficient storage)              │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## SECTION 7.3: DOCKER — COMPLETE REFERENCE

### Docker Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    DOCKER ARCHITECTURE                                 │
│                                                                         │
│  CLI (docker commands)                                                 │
│       │  REST API                                                       │
│       ▼                                                                 │
│  Docker Daemon (dockerd)                                               │
│       │                                                                 │
│       ├── containerd (container lifecycle management)                  │
│       │       │                                                         │
│       │       └── runc (OCI container runtime — creates containers)   │
│       │                                                                 │
│       ├── Image storage (layers in /var/lib/docker)                   │
│       │                                                                 │
│       └── Network / Volume management                                  │
│                                                                         │
│  Docker Registry (Docker Hub / ECR / GCR / private)                   │
│  → Stores and distributes Docker images                                │
└─────────────────────────────────────────────────────────────────────────┘
```

### Docker Image Layers

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    DOCKER IMAGE LAYERS                                 │
│                                                                         │
│  FROM ubuntu:22.04              ← Layer 1: Base OS                    │
│  RUN apt-get install python3    ← Layer 2: Python                     │
│  COPY requirements.txt .        ← Layer 3: Requirements file          │
│  RUN pip install -r req.txt     ← Layer 4: Python packages            │
│  COPY . /app                    ← Layer 5: Application code           │
│  CMD ["python3", "app.py"]      ← Metadata (not a layer)             │
│                                                                         │
│  Each layer is a diff from the previous.                               │
│  Layers are cached — if Layer 3 doesn't change, Layers 1-3 are reused │
│  This is why COPY requirements.txt comes BEFORE COPY . /app           │
│  (app code changes every build; deps change rarely)                   │
└─────────────────────────────────────────────────────────────────────────┘
```

### Essential Docker Commands

```bash
# ─── Images ───────────────────────────────────────────────────────────
docker images                           # list local images
docker images -a                        # include intermediate layers
docker pull ubuntu:22.04                # download image
docker pull nginx:alpine                # specific tag
docker tag myapp:latest myapp:v1.0.0   # create tag/alias
docker rmi ubuntu:22.04                 # remove image
docker image prune                      # remove dangling images
docker image prune -a                   # remove ALL unused images

# ─── Containers ───────────────────────────────────────────────────────
docker run nginx                            # run in foreground (Ctrl+C stops)
docker run -d nginx                         # detached (background)
docker run -d --name my-nginx nginx         # with name
docker run -d -p 8080:80 nginx              # port mapping host:container
docker run -d -p 8080:80 -p 8443:443 nginx  # multiple ports
docker run -it ubuntu:22.04 bash            # interactive terminal
docker run --rm ubuntu:22.04 echo "hello"   # auto-remove after exit
docker run -e DB_HOST=localhost nginx       # environment variable
docker run --env-file .env nginx            # env from file
docker run -v /host/data:/container/data nginx  # bind mount
docker run -v myvolume:/data nginx          # named volume
docker run --memory="512m" --cpus="0.5" nginx  # resource limits

# List containers
docker ps                               # running containers
docker ps -a                            # all containers (including stopped)
docker ps -q                            # only IDs
docker ps --format "table {{.ID}}\t{{.Image}}\t{{.Status}}"  # custom format

# Container operations
docker stop my-nginx                    # graceful stop (SIGTERM)
docker kill my-nginx                    # force stop (SIGKILL)
docker start my-nginx                   # start stopped container
docker restart my-nginx                 # restart
docker rm my-nginx                      # remove stopped container
docker rm -f my-nginx                   # force remove (even if running)
docker rm $(docker ps -aq)              # remove ALL stopped containers

# Inspect and debug
docker logs my-nginx                    # container logs
docker logs -f my-nginx                 # follow logs
docker logs --tail=100 my-nginx         # last 100 lines
docker exec -it my-nginx bash           # exec into running container
docker exec my-nginx cat /etc/nginx/nginx.conf  # run single command
docker inspect my-nginx                 # full container details (JSON)
docker stats                            # live resource usage
docker stats --no-stream               # one-time stats snapshot
docker top my-nginx                     # processes inside container

# Copy files
docker cp my-nginx:/etc/nginx/nginx.conf ./nginx.conf  # from container
docker cp ./nginx.conf my-nginx:/etc/nginx/nginx.conf  # to container
```

### Writing a Dockerfile

```dockerfile
# ─── GOOD Dockerfile Example ──────────────────────────────────────────

# Use specific version, not 'latest'
FROM python:3.11-slim

# Set metadata
LABEL maintainer="devops@example.com"
LABEL version="1.0.0"

# Set working directory
WORKDIR /app

# Install system dependencies FIRST (changes less often)
RUN apt-get update && apt-get install -y --no-install-recommends \
    curl \
    && rm -rf /var/lib/apt/lists/*

# Copy and install Python dependencies SEPARATELY from app code
# → This layer is cached unless requirements.txt changes
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code last (changes most often)
COPY . .

# Create non-root user for security
RUN useradd -r -s /sbin/nologin appuser && \
    chown -R appuser:appuser /app
USER appuser

# Document the port (does not actually publish it)
EXPOSE 8080

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:8080/health || exit 1

# Default command
CMD ["python", "app.py"]
```

### Docker Build Flags

```bash
# Build image
docker build -t myapp:latest .
docker build -t myapp:v1.0.0 -f Dockerfile.prod .  # custom Dockerfile
docker build --no-cache -t myapp:latest .           # no cache
docker build --build-arg APP_ENV=production .       # build arguments
docker build --target builder .                     # multi-stage target

# Build args in Dockerfile:
# ARG APP_VERSION=1.0.0
# ARG APP_ENV=development
# RUN echo "Building version $APP_VERSION for $APP_ENV"

# Multi-stage build (production best practice)
cat > Dockerfile.multistage << 'EOF'
# Stage 1: Build
FROM node:18 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Stage 2: Production (much smaller image)
FROM nginx:alpine AS production
COPY --from=builder /app/dist /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
EOF
```

---

## SECTION 7.4: DOCKER NETWORKING

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    DOCKER NETWORK TYPES                                │
│                                                                         │
│  bridge (default)                                                      │
│  ─────────────────────────────────────────────────────────            │
│  • Default network for containers                                      │
│  • docker0 bridge interface (172.17.0.0/16)                           │
│  • Containers communicate via container name (in user-defined bridge)  │
│  • Port mapping required to reach from host                            │
│                                                                         │
│  host                                                                  │
│  ─────────────────────────────────────────────────────────            │
│  • Container shares host's network stack                               │
│  • No port mapping needed (container port IS host port)               │
│  • Less isolation, better performance                                  │
│                                                                         │
│  none                                                                  │
│  ─────────────────────────────────────────────────────────            │
│  • No networking at all                                                │
│  • Use for batch processing containers that don't need network        │
│                                                                         │
│  overlay                                                               │
│  ─────────────────────────────────────────────────────────            │
│  • Multi-host networking (Docker Swarm / Kubernetes)                  │
│  • Connects containers across multiple Docker hosts                    │
└─────────────────────────────────────────────────────────────────────────┘
```

```bash
# Network management
docker network ls                               # list networks
docker network create myapp-net                 # create user-defined bridge
docker network create --subnet=172.20.0.0/16 mynet
docker network inspect myapp-net               # network details
docker network connect myapp-net my-nginx      # connect container to network
docker network disconnect myapp-net my-nginx   # disconnect

# Run containers on same network (they can reach each other by name)
docker network create app-net
docker run -d --name db --network app-net postgres:15
docker run -d --name app --network app-net -e DB_HOST=db myapp

# Inside app container, can now do: psql -h db -U postgres
```

### Docker Volumes

```bash
# Volume management
docker volume ls                            # list volumes
docker volume create mydata                 # create named volume
docker volume inspect mydata               # volume details
docker volume rm mydata                     # delete volume
docker volume prune                         # remove unused volumes

# Types of mounts:
# 1. Named volume (managed by Docker)
docker run -v mydata:/data postgres:15

# 2. Bind mount (specific host path)
docker run -v /host/postgres/data:/var/lib/postgresql/data postgres:15

# 3. tmpfs mount (in-memory, not persisted)
docker run --tmpfs /tmp:rw,size=100m myapp

# Volume backup
docker run --rm -v mydata:/data -v $(pwd):/backup \
    alpine tar czf /backup/mydata_backup.tar.gz -C /data .

# Volume restore
docker run --rm -v mydata:/data -v $(pwd):/backup \
    alpine tar xzf /backup/mydata_backup.tar.gz -C /data
```

---

## SECTION 7.5: DOCKER COMPOSE

Docker Compose lets you define and run multi-container applications with a single YAML file.

```yaml
# docker-compose.yml — Complete example

version: '3.8'

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
      args:
        APP_ENV: production
    image: myapp:latest
    container_name: myapp
    restart: unless-stopped
    ports:
      - "8080:8080"
    environment:
      - APP_ENV=production
      - DB_HOST=postgres
      - DB_PORT=5432
      - REDIS_HOST=redis
    env_file:
      - .env
    volumes:
      - app_logs:/app/logs
      - ./config:/app/config:ro   # :ro = read-only
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_started
    networks:
      - app-net
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 30s

  postgres:
    image: postgres:15-alpine
    container_name: postgres
    restart: unless-stopped
    environment:
      POSTGRES_DB: myapp_db
      POSTGRES_USER: myapp_user
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./migrations:/docker-entrypoint-initdb.d:ro
    networks:
      - app-net
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U myapp_user -d myapp_db"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    container_name: redis
    restart: unless-stopped
    command: redis-server --appendonly yes --requirepass ${REDIS_PASSWORD}
    volumes:
      - redis_data:/data
    networks:
      - app-net

  nginx:
    image: nginx:alpine
    container_name: nginx
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
      - app-net

volumes:
  postgres_data:
  redis_data:
  app_logs:

networks:
  app-net:
    driver: bridge
```

```bash
# Docker Compose commands
docker compose up                       # start all services (foreground)
docker compose up -d                    # start in background
docker compose up --build               # rebuild images before starting
docker compose up -d --scale app=3     # scale app to 3 instances
docker compose down                     # stop and remove containers
docker compose down -v                  # also remove volumes
docker compose down --rmi all          # also remove images

docker compose ps                       # list services
docker compose logs                     # all service logs
docker compose logs -f app             # follow logs for 'app' service
docker compose logs --tail=50 app

docker compose exec app bash            # exec into running service
docker compose run --rm app python manage.py migrate  # one-off command

docker compose stop                     # stop without removing
docker compose start                    # start stopped services
docker compose restart app             # restart specific service

docker compose pull                     # pull latest images
docker compose build                   # build/rebuild images
docker compose config                  # validate and view config
```

---

## SECTION 7.6: DOCKER DAY 7 LABS

### Lab 7.6.1 — Build and Run Your First Application

```bash
# Step 1: Create a simple Python web app
mkdir my-docker-app && cd my-docker-app

cat > app.py << 'EOF'
from http.server import HTTPServer, BaseHTTPRequestHandler
import json
import platform
import os

class HealthHandler(BaseHTTPRequestHandler):
    def do_GET(self):
        if self.path == '/health':
            response = {
                "status": "healthy",
                "hostname": platform.node(),
                "environment": os.environ.get("APP_ENV", "unknown"),
                "python_version": platform.python_version()
            }
            self.send_response(200)
            self.send_header('Content-Type', 'application/json')
            self.end_headers()
            self.wfile.write(json.dumps(response).encode())
        else:
            self.send_response(404)
            self.end_headers()
    
    def log_message(self, format, *args):
        print(f"[{self.log_date_time_string()}] {format % args}")

if __name__ == '__main__':
    port = int(os.environ.get('PORT', 8080))
    print(f"Starting server on port {port}")
    HTTPServer(('0.0.0.0', port), HealthHandler).serve_forever()
EOF

# Step 2: Create requirements.txt (empty for this simple example)
touch requirements.txt

# Step 3: Create Dockerfile
cat > Dockerfile << 'EOF'
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

RUN useradd -r -s /sbin/nologin appuser
USER appuser

EXPOSE 8080

HEALTHCHECK --interval=15s --timeout=3s \
    CMD python3 -c "import urllib.request; urllib.request.urlopen('http://localhost:8080/health')" || exit 1

CMD ["python3", "app.py"]
EOF

# Step 4: Create .dockerignore
cat > .dockerignore << 'EOF'
.git
.gitignore
*.md
*.pyc
__pycache__
.env
Dockerfile*
docker-compose*
tests/
EOF

# Step 5: Build the image
docker build -t my-docker-app:v1.0.0 .
docker images my-docker-app

# Step 6: Run the container
docker run -d \
    --name my-app \
    -p 8080:8080 \
    -e APP_ENV=development \
    my-docker-app:v1.0.0

# Step 7: Test it
curl http://localhost:8080/health | python3 -m json.tool

# Step 8: View logs
docker logs my-app -f

# Step 9: Check health
docker inspect --format='{{.State.Health.Status}}' my-app

# Step 10: Clean up
docker stop my-app && docker rm my-app
```

### Lab 7.6.2 — Multi-Container App with Docker Compose

```bash
# Use the plm-onboarding app in BattleOps/Week1/plm-onboarding as reference
# Create a simple multi-service app

mkdir multi-app && cd multi-app

# Create web app
mkdir -p web
cat > web/app.py << 'EOF'
import os
import redis
from http.server import HTTPServer, BaseHTTPRequestHandler
import json

r = redis.Redis(
    host=os.environ.get('REDIS_HOST', 'localhost'),
    port=int(os.environ.get('REDIS_PORT', 6379)),
    decode_responses=True
)

class Handler(BaseHTTPRequestHandler):
    def do_GET(self):
        if self.path == '/health':
            try:
                r.ping()
                count = r.incr('visit_count')
                response = {"status": "healthy", "visits": count}
                code = 200
            except Exception as e:
                response = {"status": "unhealthy", "error": str(e)}
                code = 503
            self.send_response(code)
            self.send_header('Content-Type', 'application/json')
            self.end_headers()
            self.wfile.write(json.dumps(response).encode())
    
    def log_message(self, *args): pass

HTTPServer(('0.0.0.0', 8080), Handler).serve_forever()
EOF

cat > web/requirements.txt << 'EOF'
redis==5.0.1
EOF

cat > web/Dockerfile << 'EOF'
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY app.py .
EXPOSE 8080
CMD ["python3", "app.py"]
EOF

# Create docker-compose.yml
cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  web:
    build: ./web
    ports:
      - "8080:8080"
    environment:
      - REDIS_HOST=redis
    depends_on:
      - redis
    networks:
      - app-net

  redis:
    image: redis:7-alpine
    networks:
      - app-net

networks:
  app-net:
    driver: bridge
EOF

# Launch everything
docker compose up -d --build

# Test
curl http://localhost:8080/health
curl http://localhost:8080/health   # visit count increases
curl http://localhost:8080/health

# Check containers
docker compose ps
docker compose logs web

# Tear down
docker compose down
```

---

# WEEK 1 — INTERVIEW PREP

---

## SECTION 8.1: LINUX INTERVIEW QUESTIONS

**Q: What is the difference between a hard link and a symbolic link?**

> A hard link is a directory entry that points directly to the inode of a file.
> Multiple hard links share the same inode — deleting one doesn't delete the data.
> A symbolic (soft) link is a special file that contains a path to another file.
> Deleting the target breaks the symlink. Hard links can't cross filesystems;
> symlinks can. Hard links don't work on directories; symlinks do.

**Q: What does `set -euo pipefail` do in a bash script?**

> `set -e` — exit immediately if any command fails (non-zero exit code).
> `set -u` — treat unset variables as errors instead of empty strings.
> `set -o pipefail` — if any command in a pipeline fails, the pipeline fails
> (without this, `false | true` would succeed).
> Together they make scripts fail fast and loud, preventing silent errors.

**Q: How do you find the process using port 8080?**

```bash
ss -tlnp | grep :8080
lsof -i :8080
fuser 8080/tcp
```

**Q: What's the difference between `kill -9` and `kill -15`?**

> `kill -15` sends SIGTERM — the process receives the signal and can clean up
> (close connections, flush logs, finish transactions) before exiting.
> `kill -9` sends SIGKILL — the kernel immediately terminates the process with
> no opportunity to clean up. Use SIGTERM first; only escalate to SIGKILL if
> the process doesn't respond.

**Q: How do you find files larger than 100MB?**

```bash
find / -type f -size +100M 2>/dev/null
find / -type f -size +100M -exec ls -lh {} \;
```

**Q: What is sticky bit? When would you use it?**

> The sticky bit on a directory means only the file owner (or root) can delete
> or rename files within it, even if others have write permission to the directory.
> /tmp has sticky bit — users can create files there but can't delete each other's files.
> Set with `chmod +t /path/to/dir` or `chmod 1755 /path/to/dir`.

---

## SECTION 8.2: NETWORKING INTERVIEW QUESTIONS

**Q: Explain the TCP three-way handshake.**

> 1. Client sends SYN (synchronize) with its initial sequence number.
> 2. Server responds with SYN-ACK (acknowledges client's SYN, sends own SYN).
> 3. Client sends ACK (acknowledges server's SYN).
> Connection is established. Allows both sides to synchronize sequence numbers
> for ordered, reliable delivery.

**Q: What's the difference between TCP and UDP? When would you use each?**

> TCP: connection-oriented, reliable, ordered, error-corrected. Slower due to overhead.
> Use for: web (HTTP/S), SSH, databases, email, file transfer — anything where data integrity matters.
> UDP: connectionless, unreliable, no ordering. Faster, lower latency.
> Use for: DNS, video streaming, gaming, VoIP, DHCP — where speed matters more than perfect delivery.

**Q: What happens when you type google.com in a browser?**

> 1. Browser checks local DNS cache.
> 2. OS checks /etc/hosts.
> 3. OS queries recursive DNS resolver (e.g., 8.8.8.8).
> 4. Resolver queries root nameservers → .com TLD nameservers → google.com authoritative NS.
> 5. Authoritative NS returns IP address.
> 6. Browser performs TCP 3-way handshake on port 443.
> 7. TLS handshake — certificates exchanged and verified.
> 8. HTTP request sent inside encrypted TLS session.
> 9. Server responds with HTML.

**Q: What is the difference between 502 and 503?**

> 502 Bad Gateway: The load balancer/proxy received an invalid response from the upstream server.
> The backend is responding but sending garbage. Check app health, look at app logs.
> 503 Service Unavailable: The backend cannot handle requests — it's down, overloaded,
> or health checks are failing. Often means no healthy upstream backends.

**Q: What is CIDR? What is 10.0.0.0/24?**

> CIDR (Classless Inter-Domain Routing) is a method of specifying IP address ranges.
> 10.0.0.0/24 means the first 24 bits are the network address (10.0.0),
> the remaining 8 bits are host addresses. This gives 256 total IPs (10.0.0.0 to 10.0.0.255)
> with 254 usable host addresses (excluding network and broadcast addresses).

---

## SECTION 8.3: GIT INTERVIEW QUESTIONS

**Q: What is the difference between `git merge` and `git rebase`?**

> `git merge` creates a new merge commit that joins two branch histories.
> Preserves complete history. Good for shared branches.
> `git rebase` moves commits from one branch to apply on top of another.
> Creates linear history. Good for local/feature branches before merging.
> Golden rule: Never rebase commits that have been pushed to a shared branch.

**Q: How do you undo the last commit without losing changes?**

```bash
git reset --soft HEAD~1   # undo commit, keep changes staged
git reset HEAD~1          # undo commit, keep changes unstaged
git reset --hard HEAD~1   # undo commit AND discard changes (destructive!)
```

**Q: What is `git stash`? When would you use it?**

> `git stash` temporarily saves uncommitted changes (staged and unstaged) 
> and restores the working directory to a clean state.
> Use it when you need to quickly switch branches but have unfinished work
> you're not ready to commit. `git stash pop` restores the changes.

**Q: How do you find who introduced a bug on a specific line?**

```bash
git blame file.py             # shows who last changed each line and when
git log -p -S "buggy_function" --all  # find commits that added/removed a string
git bisect start              # binary search to find the commit that introduced a bug
```

---

## SECTION 8.4: DOCKER INTERVIEW QUESTIONS

**Q: What is the difference between a Docker image and a Docker container?**

> A Docker image is a read-only template — like a class definition. It contains the
> filesystem, binaries, libraries, and config needed to run an application.
> A Docker container is a running instance of an image — like an object instantiated
> from a class. Containers are ephemeral; images are persistent.

**Q: How do you reduce Docker image size?**

> 1. Use slim/alpine base images (python:3.11-slim vs python:3.11).
> 2. Multi-stage builds — build in one stage, copy only artifacts to final stage.
> 3. Combine RUN commands to reduce layers.
> 4. Use .dockerignore to exclude unnecessary files.
> 5. Install only required packages (--no-install-recommends).
> 6. Remove package manager caches (rm -rf /var/lib/apt/lists/*).

**Q: What is the difference between CMD and ENTRYPOINT?**

> `ENTRYPOINT` defines the executable that runs. It's the "what to run".
> `CMD` provides default arguments to ENTRYPOINT, or the command if no ENTRYPOINT.
> When both are used: ENTRYPOINT is the fixed command, CMD is overridable args.
> `docker run myimage arg1` overrides CMD but not ENTRYPOINT.
> Use ENTRYPOINT when the container has a specific purpose (e.g., a specific binary).
> Use CMD for flexible default behavior.

**Q: Explain Docker networking. How do containers communicate?**

> Docker creates a virtual network (bridge by default). Containers on the same
> user-defined bridge network can reach each other by container name (DNS resolution
> built-in). The host communicates with containers via port mappings (-p host:container).
> The default bridge network doesn't provide DNS — you must use user-defined bridges.

**Q: What is a Docker volume? When would you use a bind mount vs a named volume?**

> Named volumes are managed by Docker (stored in /var/lib/docker/volumes).
> Best for: database data, any data that must persist across container restarts.
> Bind mounts map a host path into the container.
> Best for: development (live code reload), config files, accessing host data.
> Never store important data only inside a container — it's lost when the container is removed.

---

# WEEK 1 — PRACTICE PROBLEMS

---

## SECTION 9.1: LINUX PRACTICE SCENARIOS

### Scenario 1: Disk Space Outage
```
Alert: Disk is 100% full on web server.
The deployment pipeline is failing.
Logs are no longer being written.

Task: Free disk space without deleting application data.
```

```bash
# Step 1: Find which filesystem is full
df -h

# Step 2: Find the largest directories
du -sh /* 2>/dev/null | sort -rh | head -10
du -sh /var/* | sort -rh | head -10

# Step 3: Clean Docker artifacts (often GBs)
docker system df               # see usage
docker system prune            # remove stopped containers, unused networks, dangling images
docker system prune -a --volumes  # aggressive (removes ALL unused images too)

# Step 4: Clean old logs
journalctl --disk-usage
journalctl --vacuum-size=200M   # keep only 200MB of logs

# Step 5: Find and compress old log files
find /var/log -name "*.log" -mtime +7 | xargs gzip

# Step 6: Find and delete core dumps
find / -name "core" -type f 2>/dev/null
find / -name "*.core" -type f 2>/dev/null
```

### Scenario 2: Service Not Starting

```
Alert: myapp service is failing to start.
systemctl status myapp shows "Failed to start"
```

```bash
# Step 1: Get detailed status
systemctl status myapp -l

# Step 2: Read the logs
journalctl -u myapp -n 100 --no-pager
journalctl -u myapp --since "5 minutes ago"

# Step 3: Check if port is already in use
ss -tlnp | grep :8080
# If port is taken, find and kill the process

# Step 4: Check file permissions
ls -la /opt/myapp/
# Service user must own the files

# Step 5: Run as service user manually to see errors
sudo -u appuser /opt/myapp/bin/myapp

# Step 6: Check environment variables
systemctl cat myapp          # view unit file
systemctl show myapp --property=Environment

# Step 7: Check dependencies
systemctl status postgresql redis
```

### Scenario 3: Permission Denied Errors

```
Application logs show: "Permission denied: /opt/myapp/logs/app.log"
```

```bash
# Step 1: Check who owns the log directory
ls -la /opt/myapp/

# Step 2: Check who the service runs as
grep "User=" /etc/systemd/system/myapp.service

# Step 3: Fix ownership
chown -R appuser:appuser /opt/myapp/logs
chmod 755 /opt/myapp/logs
chmod 644 /opt/myapp/logs/*.log 2>/dev/null

# Step 4: Restart service
systemctl restart myapp
journalctl -u myapp -f
```

---

## SECTION 9.2: NETWORKING PRACTICE SCENARIOS

### Scenario 4: Cannot Connect to Database

```
App error: "Connection refused to postgres:5432"
```

```bash
# Step 1: Is postgres running?
systemctl status postgresql
# OR (if in Docker)
docker ps | grep postgres

# Step 2: Is it listening on the right port?
ss -tlnp | grep :5432

# Step 3: Can we reach it by IP?
nc -zv 10.0.1.5 5432
telnet 10.0.1.5 5432

# Step 4: Is there a firewall blocking?
iptables -L -n | grep 5432
ufw status verbose

# Step 5: Check postgres config — is it listening on all interfaces?
cat /etc/postgresql/*/main/postgresql.conf | grep listen_addresses
# Must be: listen_addresses = '*'  (not just localhost)

# Step 6: Check pg_hba.conf — client authentication
cat /etc/postgresql/*/main/pg_hba.conf
# Must have an entry allowing connection from app server IP

# Step 7: Test with psql
psql -h 10.0.1.5 -U myapp_user -d myapp_db
```

### Scenario 5: DNS Resolution Failing

```
App error: "Could not resolve host: api.internal.example.com"
```

```bash
# Step 1: Verify DNS is configured
cat /etc/resolv.conf

# Step 2: Test DNS resolution
dig api.internal.example.com
nslookup api.internal.example.com

# Step 3: Test with specific DNS server
dig @10.0.0.2 api.internal.example.com  # use internal DNS

# Step 4: Check /etc/hosts
cat /etc/hosts

# Step 5: Check nsswitch.conf (resolution order)
cat /etc/nsswitch.conf
# Should have: hosts: files dns

# Step 6: If using Docker — check container DNS
docker exec myapp cat /etc/resolv.conf
docker exec myapp nslookup api.internal.example.com

# Fix: In docker-compose, set custom DNS
# dns:
#   - 10.0.0.2
```

---

## SECTION 9.3: DOCKER PRACTICE SCENARIOS

### Scenario 6: Container Keeps Restarting

```
docker ps shows: STATUS = Restarting (1) 2 seconds ago
```

```bash
# Step 1: Get logs
docker logs myapp --tail=50
# Look for the error that's causing the crash

# Step 2: Inspect the container
docker inspect myapp | grep -A 10 '"State"'

# Step 3: Check exit code
docker inspect myapp | grep '"ExitCode"'
# 0 = clean exit, 1 = error, 137 = OOM killed, 143 = SIGTERM

# Step 4: Run manually to see error
docker run --rm -it myapp:latest bash
# Then manually run the command to see error message

# Step 5: Common causes:
# - Missing environment variable → check your env_file or -e flags
# - Port already in use → ss -tlnp | grep :8080
# - Config file not found → check volume mounts
# - OOM killed → add memory limits, check for memory leak
# - Missing dependencies → check Dockerfile

# Step 6: Temporarily increase restart delay for debugging
# Add to docker-compose: restart: "no"
```

### Scenario 7: Docker Image Too Large

```
Image size: 2.4 GB
Target: < 200 MB
```

```bash
# Step 1: Analyze layers
docker history myapp:latest
docker image inspect myapp:latest | grep -i size

# Step 2: Use dive tool to inspect layers
docker run --rm -it wagoodman/dive myapp:latest

# Step 3: Optimize Dockerfile
# BEFORE (bad):
# FROM python:3.11          # 1.2GB image
# RUN apt-get update
# RUN apt-get install build-essential
# RUN pip install -r requirements.txt
# COPY . .

# AFTER (good):
# FROM python:3.11-slim     # 130MB base image
# RUN apt-get update && apt-get install -y --no-install-recommends \
#     gcc \
#     && rm -rf /var/lib/apt/lists/*
# COPY requirements.txt .
# RUN pip install --no-cache-dir -r requirements.txt
# COPY . .

# Step 4: Multi-stage build
# FROM python:3.11 AS builder
# RUN pip install --no-cache-dir -r requirements.txt
# 
# FROM python:3.11-slim AS final
# COPY --from=builder /usr/local/lib/python3.11/site-packages /usr/local/lib/python3.11/site-packages
# COPY . .
```

---

# WEEK 1 — QUICK REFERENCE CHEATSHEET

---

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    LINUX COMMANDS CHEATSHEET                           │
│                                                                         │
│  Navigation       Files              Permissions                       │
│  ─────────────    ────────────────   ─────────────                     │
│  pwd              cat / less / more  chmod 755 file                   │
│  ls -la           touch file         chmod u+x file                   │
│  cd /path         cp -r src dst      chown user:grp f                 │
│  mkdir -p a/b/c   mv old new         ls -la                           │
│  find / -name f   rm -rf dir         stat file                        │
│                                                                         │
│  Processes        System Info        Networking                        │
│  ─────────────    ────────────────   ─────────────                     │
│  ps aux           df -h              ip addr                           │
│  top / htop       free -h            ip route                         │
│  kill -9 PID      uptime             ss -tlnp                         │
│  systemctl        du -sh /dir        ping / traceroute                │
│  journalctl -u    lsblk              dig / nslookup                   │
│                                                                         │
│  Search           Text Processing    Package Mgmt                     │
│  ─────────────    ────────────────   ─────────────                     │
│  grep -r          awk / sed          apt update                       │
│  find . -name     cut -d: -f1        apt install                     │
│  locate file      sort | uniq -c     apt remove                      │
│  which cmd        wc -l              dpkg -l | grep X                 │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│                    GIT CHEATSHEET                                      │
│                                                                         │
│  Setup             Daily Workflow     Branching                        │
│  ─────────────     ─────────────────  ─────────────                    │
│  git config        git status         git branch -a                   │
│  git init          git add .          git switch -c feat               │
│  git clone URL     git commit -m ""   git merge feat                  │
│                    git push           git rebase main                  │
│                    git pull           git branch -d feat               │
│                                                                         │
│  Undo             History             Remote                           │
│  ─────────────    ─────────────────   ─────────────                    │
│  git restore f    git log --oneline   git remote -v                   │
│  git reset HEAD~1 git log --graph     git fetch                        │
│  git revert SHA   git show SHA        git pull --rebase               │
│  git stash        git blame file      git push -u origin              │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│                    DOCKER CHEATSHEET                                   │
│                                                                         │
│  Images           Containers          Compose                         │
│  ─────────────    ─────────────────   ─────────────                    │
│  docker pull      docker run -d       docker compose up -d            │
│  docker images    docker ps           docker compose down             │
│  docker build -t  docker stop/rm      docker compose logs -f          │
│  docker rmi       docker logs -f      docker compose exec             │
│  docker push      docker exec -it     docker compose ps               │
│                                                                         │
│  Debug            Cleanup             Networks & Volumes               │
│  ─────────────    ─────────────────   ─────────────                    │
│  docker stats     docker system prune docker network ls               │
│  docker inspect   docker image prune  docker volume ls                │
│  docker top       docker rm $(ps -aq) docker network create           │
│  docker events    docker volume prune docker volume create            │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│                    OSI MODEL CHEATSHEET                                │
│                                                                         │
│  Layer  Name          Protocol          Tool                           │
│  ─────  ────────────  ─────────────     ─────────────                  │
│  7      Application   HTTP/S, DNS, SSH  curl, dig, nslookup            │
│  6      Presentation  TLS/SSL           openssl s_client               │
│  5      Session       Sockets, RPC      ss -s                          │
│  4      Transport     TCP, UDP          ss -tlnp, nc -zv               │
│  3      Network       IP, ICMP          ping, traceroute, ip route     │
│  2      Data Link     Ethernet, ARP     arp -n, ip link                │
│  1      Physical      Cables, RF        ip link show, ethtool          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## FINAL NOTES — WEEK 1 MINDSET

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    WEEK 1 KEY TAKEAWAYS                                │
│                                                                         │
│  1. LINUX IS YOUR HOME                                                 │
│     Every tool you use runs on Linux. Know the filesystem,            │
│     permissions, processes, and systemd cold.                          │
│                                                                         │
│  2. NETWORKING EXPLAINS EVERY OUTAGE                                   │
│     Most production incidents are networking-related.                  │
│     OSI model lets you narrow down where the problem is.              │
│     Learn to diagnose from Layer 1 to Layer 7.                        │
│                                                                         │
│  3. SCRIPTING MULTIPLIES YOUR EFFORT                                   │
│     Manual tasks become automated. Automation becomes pipelines.       │
│     Pipelines become DevOps. Start scripting everything.               │
│                                                                         │
│  4. GIT IS NOT OPTIONAL                                                │
│     Every change to infrastructure, code, and config must go          │
│     through Git. Infrastructure as Code means Git history = audit log. │
│                                                                         │
│  5. CONTAINERS ARE THE UNIT OF DEPLOYMENT                              │
│     Everything you build will run in a container.                     │
│     Understand images, layers, networking, and volumes deeply.         │
│     Docker Compose is the gateway to Kubernetes.                       │
│                                                                         │
│  WHAT'S NEXT (Week 2):                                                 │
│  • AWS Core Services (EC2, VPC, S3, IAM, RDS)                        │
│  • Networking in AWS (Security Groups, NACLs, Route Tables)           │
│  • AWS CLI and automation                                              │
└─────────────────────────────────────────────────────────────────────────┘
```

---

*InfraThrone CoreOps Track — Week 1 Complete Notes*  
*Version: 1.0 | Last Updated: June 2026*
