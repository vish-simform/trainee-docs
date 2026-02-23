[🏠 Home](../README.md) · [Linux](README.md)

# 🐧 Linux Administration — Comprehensive DevOps/Cloud Handbook

> **Audience:** DevOps / Cloud Engineers & Trainees | **Focus:** Architecture, Mental Models, Why it Matters  
> **Source:** Complete Linux Training Course to Get Your Dream IT Job 2026  
> **Tip:** Sections are grouped by domain for logical revision flow. Use the TOC to jump around.

---

## Table of Contents

1. [Linux Boot Process & System Architecture](#1-linux-boot-process--system-architecture)
2. [CLI Essentials & Productivity](#2-cli-essentials--productivity)
3. [File Operations & Text Processing](#3-file-operations--text-processing)
4. [Vi/Vim & sed — The Editor & Stream Editor](#4-vivim--sed--the-editor--stream-editor)
5. [User & Group Administration](#5-user--group-administration)
6. [File Permissions & Special Permissions](#6-file-permissions--special-permissions)
7. [Process Management & Job Control](#7-process-management--job-control)
8. [Task Scheduling — crontab & at](#8-task-scheduling--crontab--at)
9. [System Monitoring, Performance & Logs](#9-system-monitoring-performance--logs)
10. [Shell Scripting & Environment Variables](#10-shell-scripting--environment-variables)
11. [Networking Fundamentals & Commands](#11-networking-fundamentals--commands)
12. [Remote Access & File Transfer — SSH, SCP, rsync](#12-remote-access--file-transfer--ssh-scp-rsync)
13. [Network Configuration & DNS](#13-network-configuration--dns)
14. [Package Management](#14-package-management)
15. [Storage & Disk Management — fdisk, LVM, NFS](#15-storage--disk-management--fdisk-lvm-nfs)
16. [Essential Services Stack — NTP, HTTP, Mail, Proxy, DHCP](#16-essential-services-stack--ntp-http-mail-proxy-dhcp)
17. [Directory Services & Authentication — LDAP, IDM, WinBIND](#17-directory-services--authentication--ldap-idm-winbind)
18. [Security, Firewall & OS Hardening](#18-security-firewall--os-hardening)
19. [Containerization & Configuration Management — Docker, Ansible](#19-containerization--configuration-management--docker-ansible)
20. [Monitoring & High Availability — Nagios, Cockpit, HA Clusters](#20-monitoring--high-availability--nagios-cockpit-ha-clusters)
21. [X vs Y — Head-to-Head Comparisons](#21-x-vs-y--head-to-head-comparisons)
22. [Viva / Trainee Review — Q&A Bank](#22-viva--trainee-review--qa-bank)
23. [DevOps/Cloud Mapping Table + Quick Revision Card](#23-devopscloud-mapping-table--quick-revision-card)

---

## 1. Linux Boot Process & System Architecture

### 1.1 Why Understand the Boot Process?

As a DevOps/Cloud engineer, you **will** encounter:
- VMs stuck in boot loops → need to know WHERE in the chain it fails
- Kernel panics → is it hardware (POST)? bootloader (GRUB)? kernel? init?
- Custom AMIs/images → you control what kernel, initramfs, and init system is used
- Recovery scenarios → GRUB rescue, single-user mode, root password recovery

### 1.2 The Boot Chain — Mental Model

```
  ┌─────────────────────────────────────────────────────────────────┐
  │                    LINUX BOOT SEQUENCE                          │
  │                                                                 │
  │  ┌─────────┐   ┌────────┐   ┌──────────┐   ┌────────────────┐   │
  │  │  Power  │──►│  BIOS/ │──►│  Boot    │──►│    Kernel      │   │
  │  │  On     │   │  UEFI  │   │  Loader  │   │    Loading     │   │
  │  └─────────┘   └────────┘   │  (GRUB2) │   └───────┬────────┘   │
  │                  │          └──────────┘           │            │
  │            ┌─────┴─────┐                    ┌───────▼────────┐  │
  │            │   POST    │                    │   initramfs    │  │
  │            │ (Power-On │                    │  (temp root    │  │
  │            │  Self     │                    │   filesystem)  │  │
  │            │  Test)    │                    └───────┬────────┘  │
  │            └───────────┘                           │            │
  │                                             ┌──────▼─────────┐  │
  │                                             │   Init System  │  │
  │                                             │   (systemd)    │  │
  │                                             │   PID 1        │  │
  │                                             └──────┬─────────┘  │
  │                                                    │            │
  │                                             ┌──────▼─────────┐  │
  │                                             │  Target/       │  │
  │                                             │  Runlevel      │  │
  │                                             │  (multi-user   │  │
  │                                             │   or graphical)│  │
  │                                             └────────────────┘  │
  └─────────────────────────────────────────────────────────────────┘
```

### 1.3 Each Stage in Detail

| Stage | Component | What Happens | Failure Symptom |
|-------|-----------|-------------|-----------------|
| **1** | **POST** | Hardware check — CPU, RAM, peripherals | Beep codes, no display |
| **2** | **BIOS/UEFI** | Finds boot device (disk, USB, network PXE) | "No bootable device" |
| **3** | **CMOS** | Stores BIOS settings (boot order, clock) — battery-backed | Clock resets, settings lost |
| **4** | **MBR/GPT** | First 512 bytes (MBR) or GPT partition table — points to bootloader | "Missing operating system" |
| **5** | **GRUB2** | Loads kernel + initramfs into RAM, presents boot menu | GRUB rescue shell |
| **6** | **Kernel** | Decompresses, initializes hardware, mounts initramfs | Kernel panic |
| **7** | **initramfs** | Temporary root filesystem — loads drivers needed to mount real root | "Unable to mount root fs" |
| **8** | **systemd (PID 1)** | Takes over — starts services, mounts filesystems, reaches target | Stuck at boot, service failures |

### 1.4 BIOS vs UEFI

```
  BIOS (Legacy)                          UEFI (Modern)
  ┌───────────────────────┐               ┌──────────────────────────┐
  │ • 1975-era firmware   │               │ • Modern firmware        │
  │ • MBR partition table │               │ • GPT partition table    │
  │ • Max 2 TB disk       │               │ • Max 9.4 ZB disk        │
  │ • 4 primary partitions│               │ • 128 partitions         │
  │ • 16-bit real mode    │               │ • 32/64-bit              │
  │ • No Secure Boot      │               │ • Secure Boot ✅         │
  │ • Slow boot           │               │ • Fast boot              │
  │ • No network stack    │               │ • Built-in network (PXE) │
  └───────────────────────┘               └──────────────────────────┘
```

> **DevOps Note:** Cloud VMs typically use UEFI. Understanding Secure Boot matters for hardened images and compliance (HIPAA, SOC2).

### 1.5 GRUB2 — The Bootloader

GRUB2 (GRand Unified Bootloader) is the default bootloader for most Linux distros.

**Key files:**
| File/Path | Purpose |
|-----------|---------|
| `/boot/grub2/grub.cfg` | Generated config — **never edit directly** |
| `/etc/default/grub` | User settings (timeout, kernel params) |
| `/etc/grub.d/` | Scripts that generate `grub.cfg` |
| `grub2-mkconfig -o /boot/grub2/grub.cfg` | Regenerate GRUB config |

**Common GRUB operations:**
```bash
# View current default kernel
grub2-editenv list

# Set default kernel
grub2-set-default 0

# Regenerate GRUB config after changes
grub2-mkconfig -o /boot/grub2/grub.cfg
```

### 1.6 Recovering Root Password (GRUB Recovery)

This is a **critical sysadmin skill** — you will be asked about this in interviews.

```
  Step-by-step:
  ┌──────────────────────────────────────────────────────┐
  │ 1. Reboot → interrupt GRUB menu (press 'e')          │
  │ 2. Find the line starting with 'linux' or 'linux16'  │
  │ 3. Append: rd.break                                  │
  │ 4. Ctrl+X to boot                                    │
  │ 5. You land in initramfs emergency shell             │
  │                                                      │
  │    switch_root:/# mount -o remount,rw /sysroot       │
  │    switch_root:/# chroot /sysroot                    │
  │    sh-4.4# passwd root                               │
  │    sh-4.4# touch /.autorelabel    ← SELinux fix      │
  │    sh-4.4# exit                                      │
  │    switch_root:/# exit                               │
  │                                                      │
  │ System reboots with new root password                │
  └──────────────────────────────────────────────────────┘
```

> **Why `touch /.autorelabel`?** SELinux labels the shadow file. Changing the password outside normal context breaks the label. This forces a relabel on next boot.

### 1.7 systemd — The Init System (PID 1)

systemd replaced SysVinit and is now the standard init system on RHEL, CentOS, Ubuntu, Debian, etc.

#### Mental Model — systemd Architecture

```
  ┌────────────────────────────────────────────────────────────┐
  │                      systemd (PID 1)                       │
  │                                                            │
  │  Manages:                                                  │
  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────────┐   │
  │  │ Services │ │ Mounts   │ │ Timers   │ │ Sockets      │   │
  │  │ (.service│ │ (.mount) │ │ (.timer) │ │ (.socket)    │   │
  │  │  units)  │ │          │ │          │ │              │   │
  │  └──────────┘ └──────────┘ └──────────┘ └──────────────┘   │
  │  ┌──────────┐ ┌──────────┐ ┌──────────────────────────┐    │
  │  │ Targets  │ │ Devices  │ │ Paths, Slices, Scopes... │    │
  │  │ (.target)│ │ (.device)│ │                          │    │
  │  └──────────┘ └──────────┘ └──────────────────────────┘    │
  │                                                            │
  │  Targets (≈ Runlevels):                                    │
  │  ┌────────────────────────────────────────────────┐        │
  │  │ poweroff.target     = runlevel 0 (halt)        │        │
  │  │ rescue.target       = runlevel 1 (single-user) │        │
  │  │ multi-user.target   = runlevel 3 (no GUI)      │        │
  │  │ graphical.target    = runlevel 5 (GUI)         │        │
  │  │ reboot.target       = runlevel 6 (reboot)      │        │
  │  └────────────────────────────────────────────────┘        │
  └────────────────────────────────────────────────────────────┘
```

#### systemctl — The Command to Rule Them All

```bash
# Service lifecycle
systemctl start httpd          # Start a service
systemctl stop httpd           # Stop a service
systemctl restart httpd        # Restart (stop + start)
systemctl reload httpd         # Reload config without stopping
systemctl status httpd         # Check status + recent logs

# Boot persistence
systemctl enable httpd         # Start on boot
systemctl disable httpd        # Don't start on boot
systemctl enable --now httpd   # Enable AND start immediately

# Inspection
systemctl is-active httpd      # Returns "active" or "inactive"
systemctl is-enabled httpd     # Returns "enabled" or "disabled"
systemctl list-units --type=service          # All loaded services
systemctl list-unit-files --type=service     # All installed services

# Targets (runlevels)
systemctl get-default                         # Current default target
systemctl set-default multi-user.target       # Set to CLI-only boot
systemctl isolate rescue.target               # Switch NOW to rescue mode
```

> **DevOps Relevance:**  
> - You'll write `.service` unit files for custom daemons (e.g., Node.js app, Go binary)  
> - CI/CD agents, monitoring agents, custom exporters — all managed via systemctl  
> - `systemctl enable` = "survive reboot" = **critical for production services**  
> - Understanding targets helps you build minimal VM/container images  

---

## 2. CLI Essentials & Productivity

### 2.1 Why CLI Mastery Matters for DevOps

GUIs don't scale. You can't SSH into 500 servers and click buttons. Every DevOps tool (Terraform, Ansible, Docker, K8s) is CLI-first. **Speed in the terminal = speed in your career.**

### 2.2 Navigation & Filesystem Hierarchy

#### The Linux Filesystem — Mental Model

```
  /                          ← Root of everything
  ├── bin/                   ← Essential user binaries (ls, cp, cat)
  ├── sbin/                  ← System binaries (fdisk, iptables) — root only
  ├── etc/                   ← Configuration files (THE most important dir for sysadmins)
  ├── home/                  ← User home directories
  ├── root/                  ← Root user's home (NOT /home/root)
  ├── var/                   ← Variable data (logs, mail, spool, databases)
  │   ├── log/               ← System logs (messages, secure, dmesg)
  │   ├── spool/             ← Print/mail queues
  │   └── tmp/               ← Persistent temp files
  ├── tmp/                   ← Temporary files (cleared on reboot)
  ├── usr/                   ← User programs & data (secondary hierarchy)
  │   ├── bin/               ← Non-essential user binaries
  │   ├── sbin/              ← Non-essential system binaries
  │   ├── local/             ← Locally installed software
  │   │   └── bin/           ← Custom scripts/tools go here ✅
  │   └── share/             ← Shared data (man pages, docs)
  ├── boot/                  ← Kernel, GRUB, initramfs
  ├── dev/                   ← Device files (disks, terminals, null)
  ├── proc/                  ← Virtual filesystem — running process info
  ├── sys/                   ← Virtual filesystem — kernel/hardware info
  ├── opt/                   ← Optional/third-party software
  ├── mnt/                   ← Temporary mount point
  ├── media/                 ← Removable media auto-mount
  └── srv/                   ← Service data (web, FTP)
```

> **Pro Tip:** `/usr/local/bin` is where you put custom CLI tools so they're available system-wide without modifying system paths.

#### Essential Navigation

```bash
pwd                    # Print working directory
cd /path/to/dir        # Change directory (absolute)
cd relative/path       # Change directory (relative)
cd ~                   # Go to home directory
cd -                   # Go to previous directory (toggle)
cd ..                  # Go up one level

ls -la                 # List all files (including hidden) with details
ls -lhS               # List sorted by size, human-readable
ls -lt                 # List sorted by modification time
```

### 2.3 Tab Completion & Shortcuts

| Shortcut | Action | Why It Matters |
|----------|--------|---------------|
| `Tab` | Auto-complete file/command names | 2x Tab shows all matches |
| `Ctrl+A` | Move cursor to beginning of line | Quick edit |
| `Ctrl+E` | Move cursor to end of line | Quick edit |
| `Ctrl+U` | Cut everything before cursor | Clear mistypes |
| `Ctrl+K` | Cut everything after cursor | Trim commands |
| `Ctrl+W` | Cut previous word | Delete last argument |
| `Ctrl+Y` | Paste cut text | Undo cut |
| `Ctrl+R` | Reverse search history | Find old commands |
| `Ctrl+L` | Clear screen (same as `clear`) | Clean terminal |
| `Ctrl+C` | Kill current process | Abort |
| `Ctrl+Z` | Suspend to background | Pause and resume later |
| `Ctrl+D` | Exit shell / EOF | Logout |
| `!!` | Repeat last command | `sudo !!` = rerun as root |
| `!$` | Last argument of previous command | Quick reuse |

### 2.4 Pipes & Redirection — The Unix Philosophy

**Unix Philosophy:** Each tool does ONE thing well. Combine them with pipes.

```
  ┌─────────────────────────────────────────────────────────┐
  │   STDIN (0)  ──►  ┌─────────┐  ──►  STDOUT (1)          │
  │   (keyboard)      │ COMMAND │       (screen)            │
  │                   └────┬────┘                           │
  │                        │                                │
  │                        ▼                                │
  │                   STDERR (2)                            │
  │                   (screen — error output)               │
  └─────────────────────────────────────────────────────────┘
```

#### Redirection Operators

```bash
# Output redirection
command > file          # Redirect STDOUT to file (overwrite)
command >> file         # Redirect STDOUT to file (append)
command 2> file         # Redirect STDERR to file
command 2>&1            # Redirect STDERR to same destination as STDOUT
command > file 2>&1     # Both STDOUT and STDERR to file
command &> file         # Shorthand: both STDOUT and STDERR to file (Bash 4+)

# Input redirection
command < file          # Use file as STDIN

# Pipe — connect STDOUT of one command to STDIN of next
command1 | command2     # Pipe

# Special destinations
command > /dev/null 2>&1    # Discard ALL output (silent execution)
```

> **Real-world crontab example:**  
> `0 * * * * /opt/scripts/backup.sh >> /var/log/backup.log 2>&1`  
> This logs both success and error output to the same file — essential for debugging cron jobs.

#### Pipe Examples (Building Power)

```bash
# Count how many processes are running
ps aux | wc -l

# Find the top 5 memory-consuming processes
ps aux --sort=-%mem | head -6

# Find all unique IPs in an access log
awk '{print $1}' /var/log/httpd/access_log | sort | uniq -c | sort -rn | head 10

# Search for error lines and show context
grep -C 3 "ERROR" /var/log/application.log | tail -50
```

### 2.5 Terminal Multiplexing — tmux

tmux lets you run multiple terminal sessions inside one SSH connection. If your SSH drops, tmux sessions persist.

```
  ┌────────────────────────────────────────────────────────────┐
  │  SSH Connection                                            │
  │  ┌──────────────── tmux session ────────────────────────┐  │
  │  │                                                      │  │
  │  │  ┌─────────────────────┬────────────────────────┐    │  │
  │  │  │  Window 0: htop     │  Window 1: vim config  │    │  │
  │  │  │                     │                        │    │  │
  │  │  │  Pane 0  │ Pane 1   │                        │    │  │
  │  │  │  (logs)  │ (shell)  │                        │    │  │
  │  │  └─────────────────────┴────────────────────────┘    │  │
  │  └──────────────────────────────────────────────────────┘  │
  │                                                            │
  │  Even if SSH disconnects, the tmux session SURVIVES        │
  └────────────────────────────────────────────────────────────┘
```

```bash
tmux                        # Start new session
tmux new -s mysession       # Start named session
tmux ls                     # List sessions
tmux attach -t mysession    # Re-attach to session
tmux kill-session -t name   # Kill a session

# Inside tmux (prefix = Ctrl+B):
Ctrl+B, c                   # New window
Ctrl+B, n / p               # Next / previous window
Ctrl+B, %                   # Split pane vertically
Ctrl+B, "                   # Split pane horizontally
Ctrl+B, d                   # Detach (session keeps running)
Ctrl+B, x                   # Kill current pane
```

> **DevOps Use:** Running long deployments, tailing logs while editing configs, keeping sessions alive on jump hosts.

### 2.6 The `script` Command

Records your terminal session (input + output) to a file. Great for auditing and documentation.

```bash
script session_log.txt       # Start recording
# ... do your work ...
exit                          # Stop recording

scriptreplay                  # Replay a recorded session (with timing)
```

---

## 3. File Operations & Text Processing

### 3.1 File Maintenance Commands

| Command | Purpose | Key Flags | Example |
|---------|---------|-----------|---------|
| `cp` | Copy files/directories | `-r` (recursive), `-p` (preserve), `-i` (interactive) | `cp -rp /src /dest` |
| `mv` | Move/rename | `-i` (interactive), `-f` (force) | `mv old.txt new.txt` |
| `rm` | Remove files/dirs | `-r` (recursive), `-f` (force), `-i` (interactive) | `rm -rf /tmp/build/` |
| `mkdir` | Create directory | `-p` (parent dirs too) | `mkdir -p /opt/app/config` |
| `rmdir` | Remove empty dir | — | `rmdir /tmp/empty/` |
| `touch` | Create file / update timestamp | — | `touch newfile.txt` |
| `ln` | Create links | `-s` (symbolic/soft link) | `ln -s /etc/nginx/nginx.conf ~/nginx.conf` |

#### Hard Link vs Symbolic (Soft) Link

```
  Hard Link:                              Symbolic Link:
  ┌──────────┐     ┌──────────┐           ┌──────────┐     ┌──────────┐
  │ fileA    ├────►│  inode   │           │ linkB    ├────►│ fileA    │
  │          │     │  (data)  │           │ (symlink)│     │          │
  └──────────┘     │          │           └──────────┘     └──────────┘
  ┌──────────┐     │          │
  │ fileB    ├────►│          │           • Points to the PATH (name)
  │(hard link)│    └──────────┘           • Breaks if original is deleted
  └──────────┘                            • Can cross filesystems
                                          • Can link to directories
  • Both point to same inode (data)
  • Deleting one doesn't affect the other
  • Cannot cross filesystems
  • Cannot link to directories
```

> **DevOps Use:** Symlinks are everywhere — `/etc/nginx/sites-enabled/` → `sites-available/`, custom binaries in `/usr/local/bin/`, Docker volume mounts follow symlinks.

### 3.2 File Display Commands

| Command | Purpose | When to Use |
|---------|---------|-------------|
| `cat` | Display entire file | Small files only |
| `less` | Page through file (scrollable) | **Default go-to for reading files** |
| `more` | Page through file (forward only) | Legacy — use `less` instead |
| `head` | Show first N lines | `head -20 file` — quick peek |
| `tail` | Show last N lines | `tail -20 file` — check end of logs |
| `tail -f` | **Follow file** (live updates) | `tail -f /var/log/syslog` — **THE log watching command** |

> **Pro Tip:** `tail -f` is your best friend in production. `tail -f /var/log/messages | grep ERROR` — live error monitoring.

### 3.3 Text Processing Toolkit — The Power Tools

This is where Linux CLI becomes a **data processing pipeline**. These commands are used daily by DevOps engineers.

#### cut — Extract Columns

```bash
# Extract usernames from /etc/passwd (field 1, delimiter ':')
cut -d: -f1 /etc/passwd

# Extract columns 1 and 3 (username and UID)
cut -d: -f1,3 /etc/passwd

# Extract characters 1-8
cut -c1-8 /etc/passwd
```

#### awk — Pattern Scanning & Processing

awk is a **mini programming language** for text processing. It's column-aware by default.

```bash
# Print first column (default delimiter = whitespace)
awk '{print $1}' file.txt

# Print first and third columns
awk '{print $1, $3}' file.txt

# Custom delimiter (colon)
awk -F: '{print $1, $3}' /etc/passwd

# Filter: print lines where column 3 > 1000 (UIDs > 1000 = regular users)
awk -F: '$3 > 1000 {print $1, $3}' /etc/passwd

# Sum a column
awk '{sum += $5} END {print "Total:", sum}' data.txt

# Count lines matching a pattern
awk '/ERROR/ {count++} END {print count}' logfile
```

> **awk vs cut:** Use `cut` for simple column extraction. Use `awk` when you need conditions, calculations, or formatting.

#### grep / egrep — Search Text

```bash
grep "pattern" file              # Basic search
grep -i "pattern" file           # Case-insensitive
grep -r "pattern" /dir/          # Recursive search in directory
grep -n "pattern" file           # Show line numbers
grep -c "pattern" file           # Count matches
grep -v "pattern" file           # Invert (show non-matching lines)
grep -l "pattern" /dir/*         # List filenames with matches
grep -C 3 "pattern" file         # Show 3 lines of context (before + after)
grep -A 5 "ERROR" file           # Show 5 lines AFTER match
grep -B 2 "ERROR" file           # Show 2 lines BEFORE match

# Extended grep (regex)
egrep "error|warning|critical" /var/log/messages    # OR pattern
grep -E "^root" /etc/passwd                         # Lines starting with "root"
grep -E "[0-9]{1,3}\.[0-9]{1,3}" file               # IP-like patterns
```

> **DevOps Essential:** `grep -r "DB_PASSWORD" /opt/app/` — find hardcoded secrets. `grep -c "200" access.log` — count successful requests.

#### sed — Stream Editor

sed modifies text **in a stream** (or in-place). Think of it as "find and replace for the command line."

```bash
# Replace first occurrence per line
sed 's/old/new/' file

# Replace ALL occurrences per line (global)
sed 's/old/new/g' file

# Replace in-place (modifies the file!)
sed -i 's/old/new/g' file

# Delete lines matching pattern
sed '/pattern/d' file

# Delete line 5
sed '5d' file

# Delete lines 5 through 10
sed '5,10d' file

# Print only lines matching pattern
sed -n '/pattern/p' file

# Insert line before match
sed '/pattern/i\New line above' file

# Append line after match
sed '/pattern/a\New line below' file

# Multiple operations
sed -e 's/foo/bar/g' -e 's/baz/qux/g' file
```

> **DevOps Power Move:** `sed -i 's/LISTEN_PORT=80/LISTEN_PORT=8080/g' /etc/app/config` — change config values in scripts without opening an editor. Used extensively in Dockerfiles, CI/CD pipelines, and Ansible.

#### sort, uniq, wc, diff, cmp

```bash
# sort
sort file                    # Sort alphabetically
sort -n file                 # Sort numerically
sort -r file                 # Reverse sort
sort -t: -k3 -n /etc/passwd  # Sort by 3rd field (UID), numerically
sort -u file                 # Sort and remove duplicates

# uniq (requires sorted input!)
sort file | uniq             # Remove duplicate lines
sort file | uniq -c          # Count occurrences
sort file | uniq -d          # Show only duplicates

# wc (word count)
wc file                     # Lines, words, characters
wc -l file                  # Line count only
wc -w file                  # Word count only
cat /etc/passwd | wc -l      # How many users?

# diff — compare files
diff file1 file2             # Show differences
diff -u file1 file2          # Unified format (patch-style) ← most readable
diff -y file1 file2          # Side-by-side

# cmp — binary comparison
cmp file1 file2              # Reports first difference byte
```

> **The Classic Pipeline:** `sort | uniq -c | sort -rn | head` — the "frequency analysis" pattern. Used for top IPs, top error messages, most common requests.

### 3.4 Archiving & Compression

```bash
# tar (Tape Archive) — bundles files, does NOT compress by default
tar -cvf archive.tar /dir/       # Create archive (verbose)
tar -xvf archive.tar             # Extract archive
tar -tvf archive.tar             # List contents (don't extract)

# tar + gzip (most common combo)
tar -czvf archive.tar.gz /dir/   # Create compressed archive
tar -xzvf archive.tar.gz         # Extract compressed archive

# tar + bzip2 (better compression, slower)
tar -cjvf archive.tar.bz2 /dir/
tar -xjvf archive.tar.bz2

# gzip / gunzip (single file compression)
gzip file                        # Compress → file.gz (original deleted)
gunzip file.gz                   # Decompress
gzip -k file                     # Keep original

# Other tools
split -b 100M bigfile.tar.gz chunk_   # Split large file into 100MB chunks
truncate -s 0 /var/log/big.log        # Empty a file without deleting it
```

**Mnemonic for tar flags:**

```
  c = Create       x = eXtract      t = lisT
  z = gZip         j = bzip2 (J)    v = Verbose
  f = File (always last, followed by filename)

  "Create Zipped Verbose File" = czvf
  "eXtract Zipped Verbose File" = xzvf
```

---

## 4. Vi/Vim & sed — The Editor & Stream Editor

### 4.1 Why Learn Vim?

- It's on **every** Linux server — even minimal installs have `vi`
- When you SSH into a production server at 3 AM, Vim is all you've got
- It's the default editor for `crontab -e`, `visudo`, git commit messages
- Speed: once learned, it's faster than any GUI editor for quick edits

### 4.2 Vim's Modal Architecture — Mental Model

```
  ┌─────────────────────────────────────────────────────────┐
  │                     VIM MODES                           │
  │                                                         │
  │            ┌──────────────┐                             │
  │      i,a,o │              │ Esc                         │
  │     ┌──────┤  NORMAL      ├──────┐                      │
  │     │      │  (navigate,  │      │                      │
  │     ▼      │   delete,    │      ▼                      │
  │  ┌─────────┤   yank)      ├──────────┐                  │
  │  │ INSERT  │              │ VISUAL   │                  │
  │  │ (type   └──────┬───────┘ (select  │                  │
  │  │  text)         │        text)     │                  │
  │  └────────────────│        └─────────┘                  │
  │                   │ :                                   │
  │            ┌──────▼───────┐                             │
  │            │ COMMAND-LINE │                             │
  │            │ (:w :q :s    │                             │
  │            │  /search)    │                             │
  │            └──────────────┘                             │
  │                                                         │
  │  Golden Rule: When in doubt, press Esc                  │
  └─────────────────────────────────────────────────────────┘
```

### 4.3 Essential Vim Commands

#### Mode Switching

| Key | Action |
|-----|--------|
| `i` | Insert before cursor |
| `a` | Insert after cursor |
| `o` | Insert new line below |
| `O` | Insert new line above |
| `Esc` | Return to Normal mode |
| `v` | Visual mode (character select) |
| `V` | Visual line mode |
| `Ctrl+V` | Visual block mode (column select) |

#### Navigation (Normal Mode)

| Key | Action |
|-----|--------|
| `h j k l` | Left, Down, Up, Right |
| `w / b` | Next word / Previous word |
| `0 / $` | Beginning / End of line |
| `gg / G` | First line / Last line |
| `:<number>` | Go to line number |
| `Ctrl+F / Ctrl+B` | Page forward / Page back |
| `%` | Jump to matching bracket |

#### Editing (Normal Mode)

| Key | Action |
|-----|--------|
| `dd` | Delete (cut) entire line |
| `yy` | Yank (copy) entire line |
| `p` | Paste below / `P` paste above |
| `u` | Undo |
| `Ctrl+R` | Redo |
| `x` | Delete character |
| `dw` | Delete word |
| `cw` | Change word (delete + insert mode) |
| `.` | Repeat last action |
| `>>` / `<<` | Indent / Unindent |

#### Command-Line Mode

```vim
:w                    " Save
:q                    " Quit
:wq  or  :x  or  ZZ  " Save and quit
:q!                   " Quit without saving
:w !sudo tee %        " Save as root (when you forgot sudo)

" Search and replace
/pattern              " Search forward
?pattern              " Search backward
n / N                 " Next / Previous match
:%s/old/new/g         " Replace all in file
:%s/old/new/gc        " Replace all with confirmation

" Useful
:set number           " Show line numbers
:set nonumber         " Hide line numbers
:set paste            " Paste mode (no auto-indent)
:syntax on            " Enable syntax highlighting
```

> **Survival Kit:** If you remember nothing else: `i` to type, `Esc` to stop typing, `:wq` to save and quit, `:q!` to abandon changes.

### 4.4 sed Revisited — Stream Editor in Depth

sed was covered in Section 3.3. Here's the **mental model** for when to use `sed` vs `vim`:

```
  ┌──────────────────────────────────────────────────────────┐
  │  Interactive editing (you're sitting at a terminal)?     │
  │  ──► Use Vim                                             │
  │                                                          │
  │  Automated editing (in a script, Dockerfile, pipeline)?  │
  │  ──► Use sed                                             │
  │                                                          │
  │  sed is the "scriptable vim" — non-interactive editing   │
  └──────────────────────────────────────────────────────────┘
```

**Advanced sed patterns for DevOps:**
```bash
# Comment out a line in a config file
sed -i '/^ServerName/s/^/#/' /etc/httpd/conf/httpd.conf

# Uncomment a line
sed -i 's/^#ServerName/ServerName/' /etc/httpd/conf/httpd.conf

# Replace only in a specific line range
sed -i '10,20s/old/new/g' file

# Add a line after a match (useful for config injection)
sed -i '/\[mysqld\]/a max_connections=500' /etc/my.cnf

# Extract text between two patterns
sed -n '/BEGIN/,/END/p' file

# Delete blank lines
sed -i '/^$/d' file
```

---

## 5. User & Group Administration

### 5.1 Why This Matters for DevOps

- **Principle of Least Privilege:** Services run as dedicated users (not root!)
- **Container Security:** Dockerfiles should specify `USER` — understanding UID/GID matters
- **Automation:** Provisioning scripts create service accounts automatically
- **Audit & Compliance:** Who can access what? `sudo` logs, `last`, `who`

### 5.2 User Architecture — Mental Model

```
  ┌────────────────────────────────────────────────────────────────┐
  │                    LINUX IDENTITY MODEL                        │
  │                                                                │
  │   User                          Group                          │
  │   ┌──────────────┐              ┌──────────────────┐           │
  │   │ username     │              │ groupname        │           │
  │   │ UID (number) │──── has ────►│ GID (number)     │           │
  │   │ home dir     │  primary     │ member list      │           │
  │   │ login shell  │  group       └──────────────────┘           │
  │   │ password     │                    ▲                        │
  │   └──────────────┘                    │                        │
  │                              can be in multiple                │
  │                              supplementary groups              │
  │                                                                │
  │   Key Files:                                                   │
  │   /etc/passwd  ← user info (everyone can read)                 │
  │   /etc/shadow  ← hashed passwords (root only)                  │
  │   /etc/group   ← group memberships                             │
  │   /etc/gshadow ← group passwords (rarely used)                 │
  │                                                                │
  │   UID Ranges:                                                  │
  │   0        = root                                              │
  │   1-999    = system/service accounts                           │
  │   1000+    = regular users                                     │
  └────────────────────────────────────────────────────────────────┘
```

#### /etc/passwd Format

```
  username : x : UID : GID : Comment : HomeDir : Shell
  ────────   ─   ───   ───   ───────   ───────   ─────
  root     : x : 0   : 0   : root    : /root   : /bin/bash
  nginx    : x : 998 : 996 : Nginx   : /var/lib/nginx : /sbin/nologin
  vishvam  : x : 1000: 1000: Vishvam : /home/vishvam  : /bin/bash
                 │
                 └── 'x' means password is in /etc/shadow
```

> **Note:** `/sbin/nologin` as shell = service account that **cannot** log in interactively. Essential for security!

### 5.3 User Management Commands

```bash
# Create a user
useradd username                        # Basic (uses defaults from /etc/default/useradd)
useradd -m -s /bin/bash -G wheel john   # Create with home dir, bash shell, add to wheel group
useradd -r -s /sbin/nologin svc-app     # System/service account (no home, no login)

# Set / change password
passwd username                         # Set password interactively
echo "username:password" | chpasswd     # Non-interactive (scripts)

# Modify existing user
usermod -aG wheel username              # Add to supplementary group (KEEP -a or it replaces!)
usermod -L username                     # Lock account
usermod -U username                     # Unlock account
usermod -s /sbin/nologin username       # Change shell (disable login)
usermod -d /new/home -m username        # Move home directory

# Delete user
userdel username                        # Delete user (keep home dir)
userdel -r username                     # Delete user + home directory + mail spool
```

> ⚠️ **Critical:** `usermod -aG group user` — the `-a` (append) flag is **essential**. Without it, the user is REMOVED from all other supplementary groups. This is a classic gotcha.

### 5.4 Group Management Commands

```bash
# Create group
groupadd developers

# Create with specific GID
groupadd -g 2000 devops

# Delete group
groupdel developers

# Add user to group
usermod -aG developers john

# View group members
getent group developers

# View user's groups
groups username
id username
```

### 5.5 Password Aging & Policies — chage

Password policies are a compliance requirement (SOC2, HIPAA, PCI-DSS).

```bash
# View password aging info
chage -l username

# Set maximum password age (90 days)
chage -M 90 username

# Set minimum password age (7 days — prevent rapid cycling)
chage -m 7 username

# Set warning days before expiry
chage -W 14 username

# Force password change on next login
chage -d 0 username

# Set account expiration date
chage -E 2026-12-31 username

# Set all at once
chage -M 90 -m 7 -W 14 username
```

#### Password Aging — Visual Flow

```
  ┌───────────────────────────────────────────────────────────────┐
  │  Password Lifecycle                                           │
  │                                                               │
  │  Password    Minimum     <── Can't change ──>  Maximum        │
  │  Changed     Age (m)     <── Active period ──> Age (M)        │
  │  ──┬──────────┬─────────────────────────────────┬───────      │
  │    │          │                                 │             │
  │    │          │              ┌─── Warning ──────┤             │
  │    │          │              │    Period (W)     │            │
  │    │          │              │    "Change your   │            │
  │    │          │              │     password!"    │            │
  │    ▼          ▼              ▼                   ▼            │
  │  Day 0      Day 7         Day 76             Day 90           │
  │  (set)      (can change)  (warnings start)   (EXPIRED)        │
  │                                                               │
  │  After expiry → Inactive period (I) → Account LOCKED          │
  └───────────────────────────────────────────────────────────────┘
```

### 5.6 su vs sudo — Privilege Escalation

```
  ┌──────────────────────────────────────────────────────────────┐
  │                                                              │
  │  su (Switch User)              sudo (Super User DO)          │
  │  ─────────────────             ─────────────────────         │
  │  • Switches to another         • Runs ONE command as         │
  │    user entirely                 another user (default: root)│
  │  • Requires TARGET user's      • Requires YOUR password      │
  │    password                    • Configured in /etc/sudoers  │
  │  • No audit trail of           • Full audit trail in         │
  │    individual commands           /var/log/secure             │
  │  • All or nothing              • Granular: allow specific    │
  │                                  commands only               │
  │                                                              │
  │  su -                          sudo command                  │
  │  (login shell, loads           sudo -i                       │
  │   target's environment)        (interactive root shell)      │
  │                                                              │
  │  DevOps verdict: ALWAYS use sudo. Never share root password. │
  └──────────────────────────────────────────────────────────────┘
```

#### /etc/sudoers — Anatomy

```bash
# ALWAYS edit with visudo (validates syntax)
visudo

# Format: WHO   WHERE    AS_WHOM    WHAT
  root    ALL=(ALL)       ALL        # root can do anything
  %wheel  ALL=(ALL)       ALL        # wheel group members = sudo access
  john    ALL=(ALL)       NOPASSWD: /usr/bin/systemctl restart httpd
  #                                  ^ Allow john to restart httpd without password
```

> **DevOps Pattern:** CI/CD agents need specific sudo commands (restart services, read logs). Use `NOPASSWD` for **specific commands only**, never `ALL`.

### 5.7 Monitoring Users

```bash
# Who is logged in RIGHT NOW
who                    # Logged-in users with terminal and login time
w                      # who + what they're doing (load average, idle time, process)

# Login history
last                   # Show recent logins (from /var/log/wtmp)
last username          # History for specific user
last reboot            # Reboot history
lastb                  # Failed login attempts (from /var/log/btmp) — needs root
lastlog                # Last login time for all users

# User identity
id                     # Current user's UID, GID, groups
id username            # Specific user's info
whoami                 # Just the username

# Communicate with users
users                  # List currently logged-in usernames
wall "System going down for maintenance in 10 minutes"    # Broadcast to all users
write username pts/0   # Send message to specific user on specific terminal
```

> **Security Use:** `last | head -20` — check who's been logging in. `lastb | head` — check for brute-force attempts. `w` — see if someone's running suspicious commands.

### 5.8 System Utility Commands

| Command | Purpose | Example |
|---------|---------|---------|
| `hostname` | Show/set hostname | `hostname` |
| `hostnamectl` | Persistent hostname change (systemd) | `hostnamectl set-hostname web-prod-01` |
| `uname -a` | Kernel version & system info | `uname -r` for kernel only |
| `arch` | CPU architecture | `x86_64`, `aarch64` |
| `dmidecode` | Hardware info from BIOS/DMI table | `dmidecode -t system` for vendor/model |
| `uptime` | How long system has been running + load avg | `uptime` |
| `date` | Current date/time | `date +%Y-%m-%d` for formatted output |
| `cal` | Calendar | `cal 2026` |
| `bc` | Calculator | `echo "5.2 * 3.1" \| bc` |
| `which` | Find binary location | `which python3` → `/usr/bin/python3` |
| `timedatectl` | Manage timezone and NTP sync | `timedatectl set-timezone Asia/Kolkata` |

#### hostnamectl — Setting Hostname Properly

```bash
# View current hostname details
hostnamectl

# Set hostname (persists across reboots)
hostnamectl set-hostname web-prod-01.corp.com

# Types of hostnames:
# static    → stored in /etc/hostname (permanent)
# transient → assigned by DHCP (temporary)
# pretty    → free-form UTF-8 name for display
```

> **DevOps Convention:** Hostnames should follow a pattern: `<role>-<env>-<number>.<domain>` → `web-prod-01.corp.com`, `db-staging-02.internal`

---

<!-- END OF BATCH 1 — Sections 1-5 -->

## 6. File Permissions & Special Permissions

### 6.1 Why Permissions Matter for DevOps

- **Security Posture:** A misconfigured permission (`chmod 777`) is a vulnerability waiting to be exploited
- **Container Security:** Files copied into Docker images retain their permissions — wrong perms = broken or insecure containers
- **CI/CD Pipelines:** Build scripts need execute permission, config files must be readable by the service user
- **Compliance:** SOC2, HIPAA, PCI-DSS all require strict access controls — auditors WILL check file permissions

### 6.2 The Permission Model — Mental Model

```
  ┌──────────────────────────────────────────────────────────────┐
  │              LINUX PERMISSION TRIPLET                        │
  │                                                              │
  │   -rwxr-xr--  1  vishvam  developers  4096  Feb 23  file.sh  │
  │   ││││││││││                                                 │
  │   │├┤├┤├┤                                                    │
  │   │ │ │ │                                                    │
  │   │ │ │ └── Others (o)     : r-- = read only                 │
  │   │ │ └──── Group (g)      : r-x = read + execute            │
  │   │ └────── Owner/User (u) : rwx = read + write + execute    │
  │   └──────── File Type      : - = file, d = dir, l = link     │
  │                                                              │
  │   Permission Values:                                         │
  │   ┌─────┬────────┬───────┬─────────────────────────────┐     │
  │   │ Sym │ Octal  │ File  │ Directory                   │     │
  │   ├─────┼────────┼───────┼─────────────────────────────┤     │
  │   │  r  │   4    │ Read  │ List contents (ls)          │     │
  │   │  w  │   2    │ Write │ Create/delete files inside  │     │
  │   │  x  │   1    │ Exec  │ Enter directory (cd)        │     │
  │   └─────┴────────┴───────┴─────────────────────────────┘     │
  │                                                              │
  │   Octal = sum of each: rwx = 4+2+1 = 7                       │
  │                         r-x = 4+0+1 = 5                      │
  │                         r-- = 4+0+0 = 4                      │
  │                                                              │
  │   So: rwxr-xr-- = 754                                        │
  └──────────────────────────────────────────────────────────────┘
```

### 6.3 chmod — Changing Permissions

```bash
# Symbolic mode
chmod u+x script.sh            # Add execute for owner
chmod g-w file.txt             # Remove write for group
chmod o=r file.txt             # Set others to read-only exactly
chmod a+r file.txt             # Add read for all (a = all)
chmod u+rwx,g+rx,o+r file     # Combine multiple

# Octal mode (most common in practice)
chmod 755 script.sh            # rwxr-xr-x  (scripts, binaries)
chmod 644 config.yml           # rw-r--r--  (config files)
chmod 600 id_rsa               # rw-------  (private keys, secrets)
chmod 700 .ssh/                # rwx------  (SSH directory)
chmod 750 /opt/app/            # rwxr-x---  (app dir, group readable)

# Recursive
chmod -R 755 /opt/app/         # Apply to directory + all contents
```

#### Common Permission Patterns for DevOps

| Octal | Symbolic | Use Case |
|-------|----------|----------|
| `755` | `rwxr-xr-x` | Scripts, binaries, public directories |
| `644` | `rw-r--r--` | Config files, HTML, general readable files |
| `600` | `rw-------` | SSH private keys, secrets, passwords |
| `700` | `rwx------` | `.ssh/` directory, private scripts |
| `750` | `rwxr-x---` | Application directories (owner + group) |
| `440` | `r--r-----` | Sudoers files, sensitive read-only configs |
| `777` | `rwxrwxrwx` | ⚠️ **NEVER in production** — security hole |

### 6.4 chown & chgrp — Changing Ownership

```bash
# Change owner
chown nginx /var/www/html/index.html

# Change owner and group
chown nginx:webdev /var/www/html/

# Recursive ownership change
chown -R appuser:appgroup /opt/application/

# Change group only
chgrp developers project/
```

> **DevOps Pattern:** After deploying application files, always `chown` them to the service user: `chown -R node:node /opt/app/` — otherwise the app can't read its own files.

### 6.5 umask — Default Permission Mask

umask defines what permissions are **removed** from newly created files and directories.

```
  ┌─────────────────────────────────────────────────────────────┐
  │  Default permissions (before umask):                        │
  │    Files:       666 (rw-rw-rw-)                             │
  │    Directories: 777 (rwxrwxrwx)                             │
  │                                                             │
  │  umask SUBTRACTS permissions:                               │
  │                                                             │
  │    umask 022:                                               │
  │      Files:       666 - 022 = 644 (rw-r--r--)               │
  │      Directories: 777 - 022 = 755 (rwxr-xr-x)               │
  │                                                             │
  │    umask 077:                                               │
  │      Files:       666 - 077 = 600 (rw-------)               │
  │      Directories: 777 - 077 = 700 (rwx------)               │
  │                                                             │
  │  Common umask values:                                       │
  │    022 → Standard (group/others can read)                   │
  │    027 → Restrictive (others get nothing)                   │
  │    077 → Very restrictive (only owner)                      │
  └─────────────────────────────────────────────────────────────┘

# Check current umask
umask

# Set umask (session only)
umask 027

# Persistent: add to ~/.bashrc or /etc/profile
echo "umask 027" >> ~/.bashrc
```

### 6.6 Special Permissions — setuid, setgid, Sticky Bit

These are the **4th octal digit** (the one most people forget). They solve specific security and collaboration problems.

#### Mental Model

```
  ┌────────────────────────────────────────────────────────────────┐
  │              SPECIAL PERMISSIONS (4th octal digit)             │
  │                                                                │
  │  ┌──────────────┐  ┌──────────────┐  ┌────────────────────┐    │
  │  │  SUID (4)    │  │  SGID (2)    │  │  Sticky Bit (1)    │    │
  │  │              │  │              │  │                    │    │
  │  │  Set User ID │  │  Set Group ID│  │  Restrict Delete   │    │
  │  │              │  │              │  │                    │    │
  │  │  On FILE:    │  │  On FILE:    │  │  On DIRECTORY:     │    │
  │  │  Runs as the │  │  Runs as the │  │  Only owner can    │    │
  │  │  FILE OWNER  │  │  FILE GROUP  │  │  delete their own  │    │
  │  │  (not caller)│  │  (not caller)│  │  files (even if    │    │
  │  │              │  │              │  │  dir is writable   │    │
  │  │  Example:    │  │  On DIR:     │  │  by all)           │    │
  │  │  /usr/bin/   │  │  New files   │  │                    │    │
  │  │  passwd      │  │  inherit the │  │  Example:          │    │
  │  │  (runs as    │  │  directory's │  │  /tmp              │    │
  │  │   root!)     │  │  group       │  │  (everyone writes, │    │
  │  │              │  │              │  │  only owner deletes│    │
  │  │  -rwsr-xr-x  │  │  -rwxr-sr-x  │  │  drwxrwxrwt        │    │
  │  │     ^        │  │        ^     │  │           ^        │    │
  │  └──────────────┘  └──────────────┘  └────────────────────┘    │
  │                                                                │
  │  Octal:  chmod 4755 file  (SUID + 755)                         │
  │          chmod 2755 dir   (SGID + 755)                         │
  │          chmod 1777 dir   (Sticky + 777)                       │
  └────────────────────────────────────────────────────────────────┘
```

#### Practical Examples

```bash
# SUID — passwd command (needs to write to /etc/shadow as root)
ls -la /usr/bin/passwd
# -rwsr-xr-x 1 root root ... /usr/bin/passwd
#    ^ 's' in user execute = SUID

# SGID on directory — team collaboration
mkdir /opt/shared-project
chgrp developers /opt/shared-project
chmod 2775 /opt/shared-project
# Now: every file created inside inherits "developers" group

# Sticky bit — /tmp protection
ls -ld /tmp
# drwxrwxrwt ... /tmp
#           ^ 't' in others execute = sticky bit
# User A can't delete User B's temp files

# Find all SUID files (security audit!)
find / -perm -4000 -type f 2>/dev/null

# Find all SGID files
find / -perm -2000 -type f 2>/dev/null
```

> **Security Note:** `find / -perm -4000` is a standard security audit command. Any unexpected SUID binary is a potential privilege escalation vector. Run it regularly.

> **DevOps Use of SGID:** Set SGID on shared project directories so all CI/CD artifacts, deployment files, and logs inherit the correct group — no more "permission denied" when different users deploy.

---

## 7. Process Management & Job Control

### 7.1 Why Process Management Matters for DevOps

- **Troubleshooting:** A runaway process consuming 100% CPU? You need to find it and kill it — fast
- **Service Health:** Is nginx running? How much memory is the Java app consuming?
- **Capacity Planning:** Understanding CPU/memory/IO per process → right-sizing VMs and containers
- **Container Internals:** Docker containers are just isolated processes — understanding Linux processes = understanding containers

### 7.2 Process Architecture — Mental Model

```
  ┌────────────────────────────────────────────────────────────┐
  │                    PROCESS TREE                            │
  │                                                            │
  │    systemd (PID 1) ← The ancestor of all processes         │
  │    ├── sshd                                                │
  │    │   └── sshd (user session)                             │
  │    │       └── bash (login shell)                          │
  │    │           ├── vim config.yml                          │
  │    │           └── tail -f /var/log/messages               │
  │    ├── httpd (master)                                      │
  │    │   ├── httpd (worker)                                  │
  │    │   ├── httpd (worker)                                  │
  │    │   └── httpd (worker)                                  │
  │    ├── crond                                               │
  │    └── dockerd                                             │
  │        └── containerd                                      │
  │            ├── nginx (container PID 1)                     │
  │            └── node app.js (container PID 1)               │
  │                                                            │
  │  Every process has:                                        │
  │  • PID  — unique process ID                                │
  │  • PPID — parent process ID                                │
  │  • UID  — user who owns it                                 │
  │  • State — Running, Sleeping, Stopped, Zombie              │
  │  • Priority — nice value (-20 to 19)                       │
  └────────────────────────────────────────────────────────────┘
```

### 7.3 Process States

```
  ┌──────────────────────────────────────────────────────┐
  │              PROCESS STATE DIAGRAM                   │
  │                                                      │
  │           ┌──────────┐                               │
  │   fork()──►│ Created  │                              │
  │           └────┬─────┘                               │
  │                │                                     │
  │           ┌────▼─────┐      ┌────────────┐           │
  │           │ Running  │◄─────┤ Ready      │           │
  │           │ (R)      ├─────►│ (runnable) │           │
  │           └────┬─────┘      └────────────┘           │
  │                │                   ▲                 │
  │           ┌────▼─────┐             │                 │
  │           │ Sleeping │─────────────┘                 │
  │           │ S = inter│ (wakes up on I/O, signal)     │
  │           │ D = unint│ (waiting for disk — CANNOT be │
  │           └────┬─────┘  killed! Common cause of      │
  │                │        "unkillable" processes)      │
  │           ┌────▼─────┐                               │
  │           │ Stopped  │  (Ctrl+Z, SIGSTOP)            │
  │           │ (T)      │                               │
  │           └────┬─────┘                               │
  │                │                                     │
  │           ┌────▼─────┐                               │
  │           │ Zombie   │  (finished but parent hasn't  │
  │           │ (Z)      │   called wait() to collect    │
  │           └──────────┘   exit status)                │
  └──────────────────────────────────────────────────────┘
```

> **Why Zombies matter:** Zombie processes don't consume CPU/memory, but they occupy a PID slot. Too many zombies (from a poorly written parent process) can exhaust the PID table. Fix: fix the parent process or kill the parent.

### 7.4 Viewing Processes — ps and top

#### ps — Snapshot of Processes

```bash
# Most common usage patterns
ps aux                          # ALL processes, user-oriented format (BSD-style)
ps -ef                          # ALL processes, full format (System V-style)
ps aux --sort=-%mem | head      # Top memory consumers
ps aux --sort=-%cpu | head      # Top CPU consumers
ps -ef --forest                 # Process tree (parent-child hierarchy)
ps -u username                  # Processes for a specific user
ps -p 1234                      # Info about specific PID

# Useful combinations
ps aux | grep nginx             # Find nginx processes
ps aux | grep -v grep | grep nginx   # Same, but exclude the grep itself
pgrep -la nginx                 # Cleaner way: find PID + command
```

#### ps aux Output — Reading It

```
  USER   PID  %CPU %MEM   VSZ   RSS  TTY  STAT  START  TIME  COMMAND
  root     1   0.0  0.1 169372 13256  ?    Ss    Feb20  0:12  /usr/lib/systemd/systemd
  nginx  1234  2.3  1.5 456000 62000  ?    S     10:30  5:42  nginx: worker process
  │      │     │    │    │      │     │    │
  │      │     │    │    │      │     │    └── S=sleeping, s=session leader
  │      │     │    │    │      │     └── ? = no terminal (daemon)
  │      │     │    │    │      └── Resident Set Size (actual RAM used)
  │      │     │    │    └── Virtual memory size
  │      │     │    └── % of total RAM
  │      │     └── % of CPU
  │      └── Process ID
  └── Owner
```

#### top / htop — Real-time Process Monitor

```bash
top                            # Real-time process monitor
htop                           # Enhanced version (install: yum install htop)
```

**Inside `top` — key commands:**

| Key | Action |
|-----|--------|
| `M` | Sort by memory |
| `P` | Sort by CPU |
| `k` | Kill a process (enter PID) |
| `r` | Renice a process |
| `1` | Show individual CPU cores |
| `c` | Show full command line |
| `f` | Select fields to display |
| `q` | Quit |

**Understanding top header:**

```
  top - 14:30:25 up 3 days, 2:15, 2 users, load average: 0.52, 0.73, 0.81
  │                                                       │     │     │
  │                                         1 min avg ────┘     │     │
  │                                         5 min avg ──────────┘     │
  │                                        15 min avg ────────────────┘
  │
  │  Load Average Rule of Thumb:
  │  • Value = number of processes waiting for CPU
  │  • If load avg ≤ number of CPU cores → system is fine
  │  • If load avg > number of CPU cores → system is overloaded
  │  • Example: 4-core server with load avg 3.5 → OK
  │             4-core server with load avg 8.0 → OVERLOADED
  │
  │  Check cores: nproc  or  lscpu | grep "^CPU(s):"
```

### 7.5 Process Signals — kill, killall, pkill

Signals are how you communicate with processes. `kill` doesn't always mean "terminate."

```
  ┌─────────────────────────────────────────────────────────┐
  │  ESSENTIAL SIGNALS                                      │
  │                                                         │
  │  Signal   Number   Action          When to Use          │
  │  ──────   ──────   ──────          ───────────          │
  │  SIGHUP     1      Reload config   Graceful reload      │
  │  SIGINT     2      Interrupt       Same as Ctrl+C       │
  │  SIGQUIT    3      Quit + coredump Debugging            │
  │  SIGKILL    9      FORCE KILL      Last resort only!    │
  │  SIGTERM   15      Graceful stop   DEFAULT — try first! │
  │  SIGSTOP   19      Pause           Can't be caught      │
  │  SIGCONT   18      Resume          Unpause              │
  │  SIGUSR1   10      User-defined    App-specific (e.g.   │
  │  SIGUSR2   12      User-defined    log rotation)        │
  └─────────────────────────────────────────────────────────┘
```

```bash
# Always try SIGTERM first (graceful)
kill 1234                     # Sends SIGTERM (15) by default
kill -15 1234                 # Explicit SIGTERM

# Reload config without restarting
kill -HUP 1234                # SIGHUP — nginx, Apache reload config
kill -1 1234                  # Same thing

# Force kill (ONLY when SIGTERM doesn't work)
kill -9 1234                  # SIGKILL — cannot be caught or ignored
# ⚠️ Data loss risk! Process can't clean up. Last resort only.

# Kill by name
killall nginx                 # Kill all processes named "nginx"
pkill -f "python app.py"     # Kill by full command match (regex)

# Kill all processes of a user
pkill -u username

# List all signals
kill -l
```

> **DevOps Best Practice:** `kill PID` → wait 5 seconds → `kill -9 PID` only if still running. SIGTERM lets the process flush buffers, close connections, write state. SIGKILL is an instant death — no cleanup.

### 7.6 Job Control — bg, fg, nohup, &

Jobs are processes started from your shell session. Understanding foreground/background is essential for managing tasks over SSH.

```
  ┌───────────────────────────────────────────────────────────┐
  │               JOB CONTROL FLOW                            │
  │                                                           │
  │  $ ./long_task.sh              ← Runs in FOREGROUND       │
  │  ^Z (Ctrl+Z)                   ← SUSPEND (SIGSTOP)        │
  │  [1]+  Stopped  ./long_task.sh                            │
  │                                                           │
  │  $ bg %1                       ← Resume in BACKGROUND     │
  │  [1]+ ./long_task.sh &                                    │
  │                                                           │
  │  $ fg %1                       ← Bring back to FOREGROUND │
  │                                                           │
  │  Alternative: start in background directly:               │
  │  $ ./long_task.sh &            ← & = run in background    │
  │                                                           │
  │  $ jobs                        ← List all jobs            │
  │  [1]+  Running   ./long_task.sh &                         │
  │  [2]-  Stopped   vim config.yml                           │
  └───────────────────────────────────────────────────────────┘
```

```bash
# Run command in background
./deploy.sh &

# List background jobs
jobs

# Bring job to foreground
fg %1                         # By job number
fg                            # Bring most recent job

# Send to background
bg %1

# nohup — keep running after logout (immune to SIGHUP)
nohup ./long-process.sh &
# Output goes to nohup.out by default

# Better: redirect output explicitly
nohup ./long-process.sh > /var/log/process.log 2>&1 &

# disown — detach a running job from the shell
./server.sh &
disown %1                     # Now it won't die when you logout
```

#### nohup vs tmux vs systemd — When to Use Which?

```
  ┌────────────────────────────────────────────────────────────┐
  │  Quick one-off task that must survive logout?              │
  │  ──► nohup ./script.sh &                                   │
  │                                                            │
  │  Interactive session that must survive SSH disconnect?     │
  │  ──► tmux / screen                                         │
  │                                                            │
  │  Production service that must survive reboots?             │
  │  ──► systemd service unit (.service file)                  │
  │                                                            │
  │  Rule: If you're using nohup for a production service,     │
  │  you're doing it wrong. Write a systemd unit file.         │
  └────────────────────────────────────────────────────────────┘
```

### 7.7 Process Priority — nice and renice

Every process has a **nice value** that affects CPU scheduling priority.

```
  ┌───────────────────────────────────────────────────────────┐
  │  Nice Value Range: -20 (highest priority)                 │
  │                     to                                    │
  │                    +19 (lowest priority)                  │
  │                                                           │
  │  Default: 0                                               │
  │  Only root can set negative values (higher priority)      │
  │                                                           │
  │  -20 ◄──── More CPU time ── 0 ── Less CPU time ────► +19  |
  │  (aggresive)           (default)            (nice/polite) |
  └───────────────────────────────────────────────────────────┘
```

```bash
# Start a process with lower priority
nice -n 10 ./cpu-intensive-backup.sh

# Start with higher priority (root only)
nice -n -5 ./critical-task.sh

# Change priority of running process
renice 10 -p 1234              # Set PID 1234 to nice value 10
renice -5 -u nginx             # Set all nginx user processes to -5

# View nice values
ps -eo pid,ni,comm | head       # NI column = nice value
top                              # NI column visible
```

> **DevOps Use:** Running a heavy backup during production hours? `nice -n 19 ./backup.sh` — give it lowest priority so it doesn't starve your web server of CPU.

---

## 8. Task Scheduling — crontab & at

### 8.1 Why Scheduling Matters for DevOps

- **Automated Backups:** Database dumps, config backups, log rotation
- **Health Checks:** Periodic scripts that verify services are running
- **Certificate Renewal:** Let's Encrypt auto-renewal via cron
- **Log Cleanup:** Prevent disks from filling up with old logs
- **Monitoring Alerts:** Scripts that check disk space, memory, etc.
- **Deployment Windows:** Scheduled deployments during off-peak hours

### 8.2 cron Architecture — Mental Model

```
  ┌────────────────────────────────────────────────────────────┐
  │                    CRON SYSTEM                             │
  │                                                            │
  │  crond (daemon)  ← Runs continuously, checks every minute  │
  │    │                                                       │
  │    ├── User Crontabs (/var/spool/cron/<username>)          │
  │    │   └── Edited with: crontab -e                         │
  │    │                                                       │
  │    ├── System Crontab (/etc/crontab)                       │
  │    │   └── Has extra USER field                            │
  │    │                                                       │
  │    ├── Cron Directories:                                   │
  │    │   ├── /etc/cron.d/         ← Drop-in cron files       │
  │    │   ├── /etc/cron.hourly/    ← Scripts run every hour   │
  │    │   ├── /etc/cron.daily/     ← Scripts run daily        │
  │    │   ├── /etc/cron.weekly/    ← Scripts run weekly       │
  │    │   └── /etc/cron.monthly/   ← Scripts run monthly      │
  │    │                                                       │
  │    └── Access Control:                                     │
  │        ├── /etc/cron.allow  ← Whitelist (if exists, ONLY   │
  │        │                     these users can use cron)     │
  │        └── /etc/cron.deny   ← Blacklist                    │
  └────────────────────────────────────────────────────────────┘
```

### 8.3 Crontab Syntax — The 5-Field Format

```
  ┌───────────── minute (0-59)
  │ ┌─────────── hour (0-23)
  │ │ ┌───────── day of month (1-31)
  │ │ │ ┌─────── month (1-12)
  │ │ │ │ ┌───── day of week (0-7, 0 and 7 = Sunday)
  │ │ │ │ │
  * * * * *  command_to_execute

  Special Characters:
  *     = every (wildcard)
  ,     = list (1,3,5 = 1 and 3 and 5)
  -     = range (1-5 = 1 through 5)
  /     = step (*/5 = every 5th)
```

#### Examples — From Basic to Advanced

```bash
# Every minute
* * * * * /opt/scripts/check-health.sh

# Every 5 minutes
*/5 * * * * /opt/scripts/monitor.sh

# Every day at 2:30 AM
30 2 * * * /opt/scripts/backup.sh

# Every Monday at 9 AM
0 9 * * 1 /opt/scripts/weekly-report.sh

# First day of every month at midnight
0 0 1 * * /opt/scripts/monthly-cleanup.sh

# Every weekday (Mon-Fri) at 6 PM
0 18 * * 1-5 /opt/scripts/deploy-staging.sh

# Every 15 minutes during business hours (9 AM - 5 PM)
*/15 9-17 * * * /opt/scripts/sync.sh

# Twice a day (8 AM and 8 PM)
0 8,20 * * * /opt/scripts/report.sh
```

#### Special Strings (Shortcuts)

```bash
@reboot     /opt/scripts/startup.sh       # Run once at boot
@hourly     /opt/scripts/check.sh         # Same as: 0 * * * *
@daily      /opt/scripts/backup.sh        # Same as: 0 0 * * *
@weekly     /opt/scripts/cleanup.sh       # Same as: 0 0 * * 0
@monthly    /opt/scripts/report.sh        # Same as: 0 0 1 * *
@yearly     /opt/scripts/audit.sh         # Same as: 0 0 1 1 *
```

### 8.4 Crontab Management

```bash
crontab -e                   # Edit YOUR crontab
crontab -l                   # List YOUR crontab
crontab -r                   # Remove ALL YOUR cron jobs (careful!)
crontab -u username -e       # Edit another user's crontab (root only)
crontab -u username -l       # List another user's crontab
```

### 8.5 Cron Output & Logging — The #1 Debugging Skill

**By default, cron emails output to the user.** If no mail system is configured, output is SILENTLY LOST. This is the #1 reason cron jobs "don't work."

```bash
# ❌ BAD: Output goes nowhere (if no mail configured)
* * * * * /opt/scripts/backup.sh

# ✅ GOOD: Redirect both STDOUT and STDERR to a log file
* * * * * /opt/scripts/backup.sh >> /var/log/backup.log 2>&1

# ✅ GOOD: Discard output (if you truly don't care)
* * * * * /opt/scripts/cleanup.sh > /dev/null 2>&1

# ✅ Log with timestamp
* * * * * echo "$(date): Starting backup" >> /var/log/backup.log && /opt/scripts/backup.sh >> /var/log/backup.log 2>&1
```

> **Cron Gotcha #2:** Cron runs with a **minimal environment**. Your `$PATH` is NOT the same as your login shell. Always use **full paths** in cron commands: `/usr/bin/python3` not `python3`, `/usr/local/bin/node` not `node`.

### 8.6 at — One-Time Scheduled Tasks

Unlike cron (recurring), `at` runs a command **once** at a specified time.

```bash
# Schedule a one-time task
at 3:00 AM
> /opt/scripts/migration.sh
> Ctrl+D                        # Submit

# Schedule for a specific date
at 10:00 PM Feb 28
> /opt/scripts/deploy-prod.sh
> Ctrl+D

# Schedule relative to now
at now + 30 minutes
> echo "Reminder: check deployment" | mail -s "Deploy Check" admin@corp.com
> Ctrl+D

# Using echo/pipe (non-interactive — great for scripts)
echo "/opt/scripts/cutover.sh" | at 2:00 AM tomorrow

# List pending at jobs
atq

# Remove a scheduled job
atrm <job_number>

# View job details
at -c <job_number>
```

> **DevOps Use:** `at` is perfect for one-off maintenance: "restart the service at 2 AM tonight" or "run the migration in 30 minutes." For recurring tasks, always use cron.

### 8.7 cron vs at — When to Use Which

```
  ┌───────────────────────────────────────────────────────────┐
  │  Recurring task? (daily, hourly, weekly)                  │
  │  ──► cron                                                 │
  │                                                           │
  │  One-time task? (run once at a specific time)             │
  │  ──► at                                                   │
  │                                                           │
  │  Need it to survive reboot?                               │
  │  ──► cron (@reboot) or systemd timer                      │
  │                                                           │
  │  Modern alternative to cron?                              │
  │  ──► systemd timers (.timer units) — more features,       │
  │      better logging, dependency management                │
  └───────────────────────────────────────────────────────────┘
```

---

## 9. System Monitoring, Performance & Logs

### 9.1 Why Monitoring Matters for DevOps

This is arguably the **most important skill set** for a DevOps engineer. When production is down, you need to answer:
- Is it **CPU** bound? → `top`, `mpstat`
- Is it **memory** bound? → `free`, `vmstat`
- Is it **disk I/O** bound? → `iostat`, `iotop`
- Is it **network** bound? → `netstat`, `ss`
- Is it an **application** error? → `/var/log/`, `journalctl`

### 9.2 System Monitoring Commands — Mental Model

```
  ┌────────────────────────────────────────────────────────────────┐
  │              MONITORING TOOLKIT BY DOMAIN                      │
  │                                                                │
  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────┐    │
  │  │  CPU     │  │  Memory  │  │  Disk    │  │  Network     │    │
  │  │          │  │          │  │          │  │              │    │
  │  │  top     │  │  free    │  │  df      │  │  ss          │    │
  │  │  htop    │  │  vmstat  │  │  du      │  │  netstat     │    │
  │  │  mpstat  │  │  top     │  │  iostat  │  │  ip a        │    │
  │  │  uptime  │  │          │  │  iotop   │  │  ping        │    │
  │  │  sar     │  │          │  │  lsblk   │  │  traceroute  │    │
  │  └──────────┘  └──────────┘  └──────────┘  └──────────────┘    │
  │                                                                │
  │  ┌──────────────────┐  ┌────────────────────────────────────┐  │
  │  │  Logs            │  │  All-in-one                        │  │
  │  │                  │  │                                    │  │
  │  │  journalctl      │  │  dmesg  (kernel messages)          │  │
  │  │  /var/log/       │  │  sar    (historical — sysstat)     │  │
  │  │  tail -f         │  │  vmstat (CPU, memory, IO, swap)    │  │
  │  └──────────────────┘  └────────────────────────────────────┘  │
  └────────────────────────────────────────────────────────────────┘
```

### 9.3 Memory Monitoring — free

```bash
free -h                          # Human-readable (MB/GB)
```

**Reading `free` output:**

```
              total    used    free    shared  buff/cache  available
Mem:          15Gi     6.2Gi   1.1Gi   245Mi   8.3Gi       8.8Gi
Swap:         2.0Gi    0.0Gi   2.0Gi

  ┌──────────────────────────────────────────────────────────────┐
  │  Key Insight:                                                │
  │                                                              │
  │  "free" memory looks LOW? Don't panic!                       │
  │                                                              │
  │  Linux uses free RAM for disk cache (buff/cache).            │
  │  This is GOOD — unused RAM is wasted RAM.                    │
  │  Cache is released when apps need it.                        │
  │                                                              │
  │  The number that MATTERS is "available" — this is what       │
  │  your applications can actually use.                         │
  │                                                              │
  │  Rule of thumb:                                              │
  │  • available < 20% of total → investigate                    │
  │  • swap is being used significantly → add RAM or find leak   │
  └──────────────────────────────────────────────────────────────┘
```

### 9.4 Disk Monitoring — df and du

```bash
# df — disk filesystem usage (how full are the partitions?)
df -h                            # Human-readable
df -hT                           # Include filesystem type
df -i                            # Inode usage (can run out even with space left!)

# du — disk usage (what's eating the space?)
du -sh /var/log/                 # Total size of a directory
du -sh /var/log/*                # Size of each item inside
du -sh /* 2>/dev/null | sort -rh | head   # Top 10 largest dirs from root
du -ah /opt/ | sort -rh | head 20         # Top 20 largest files/dirs
```

> **Production Alert Pattern:** `df -h | awk '$5+0 > 80 {print "ALERT: " $6 " is " $5 " full"}'` — one-liner to check for partitions over 80%.

### 9.5 I/O Monitoring — iostat

```bash
# Install if missing
yum install sysstat

# CPU and disk I/O stats
iostat                           # Snapshot
iostat 1                         # Refresh every 1 second (live monitoring)
iostat -x 1                      # Extended stats (includes await, %util)
```

**Key metrics in `iostat -x`:**

| Metric | What It Means | When to Worry |
|--------|--------------|---------------|
| `%util` | How busy the disk is | > 80% sustained = bottleneck |
| `await` | Average time (ms) for I/O requests | > 20ms for SSD, > 50ms for HDD |
| `r/s`, `w/s` | Reads/writes per second | Depends on workload baseline |
| `rkB/s`, `wkB/s` | Read/write throughput | Compare to disk spec |

### 9.6 Network — ss and netstat

```bash
# ss — modern replacement for netstat (faster, more info)
ss -tulnp                       # TCP/UDP, listening, numeric, process info
ss -s                            # Socket statistics summary
ss -t state established         # All established TCP connections
ss -tlnp | grep :80             # What's listening on port 80?

# netstat — legacy but still widely used
netstat -tulnp                   # Same as ss -tulnp
netstat -an                      # All connections, numeric
netstat -rn                      # Routing table (same as ip route)
```

**Reading `ss -tulnp`:**

```
  State    Recv-Q  Send-Q  Local Address:Port   Peer Address:Port  Process
  LISTEN   0       128     0.0.0.0:22           0.0.0.0:*          sshd
  LISTEN   0       511     0.0.0.0:80           0.0.0.0:*          nginx
  LISTEN   0       128     127.0.0.1:3000       0.0.0.0:*          node
                           │         │
                           │         └── Port number
                           └── 0.0.0.0 = all interfaces
                               127.0.0.1 = localhost only ← secure
```

> **DevOps Essential:** `ss -tulnp` is the first command you run to check "is my service running and listening?" — before checking firewall, DNS, or load balancer.

### 9.7 System Logs — /var/log/ and journalctl

#### Log File Locations

```
  /var/log/
  ├── messages        ← General system messages (catch-all)
  ├── secure          ← Authentication logs (SSH, sudo, su)
  ├── boot.log        ← Boot process messages
  ├── cron            ← Cron job execution logs
  ├── dmesg           ← Kernel ring buffer (hardware, drivers)
  ├── maillog         ← Mail server logs
  ├── httpd/          ← Apache logs
  │   ├── access_log
  │   └── error_log
  ├── nginx/          ← Nginx logs
  │   ├── access.log
  │   └── error.log
  ├── audit/          ← SELinux audit logs
  │   └── audit.log
  ├── wtmp            ← Login records (read with: last)
  ├── btmp            ← Failed login records (read with: lastb)
  └── lastlog         ← Last login info (read with: lastlog)
```

#### journalctl — systemd's Log Viewer

```bash
# View all logs
journalctl

# Follow new entries in real-time (like tail -f)
journalctl -f

# Logs for a specific service
journalctl -u nginx.service
journalctl -u nginx -f              # Follow nginx logs live

# Logs since last boot
journalctl -b

# Logs from previous boot (debugging what happened before crash)
journalctl -b -1

# Filter by time
journalctl --since "2026-02-23 10:00" --until "2026-02-23 12:00"
journalctl --since "1 hour ago"

# Filter by priority (0=emerg to 7=debug)
journalctl -p err                    # Only errors and above
journalctl -p warning -u httpd       # Warnings+ for httpd

# Disk usage by journal
journalctl --disk-usage

# Clean old logs (keep last 500MB)
journalctl --vacuum-size=500M
```

> **journalctl vs /var/log/messages:** journalctl queries the systemd journal (binary, structured, indexed = fast searches). `/var/log/messages` is the traditional syslog (plain text). In modern RHEL/CentOS, both exist. journalctl is preferred for service-level debugging.

#### dmesg — Kernel Messages

```bash
dmesg                              # All kernel messages
dmesg -T                           # Human-readable timestamps
dmesg | tail -20                   # Last 20 kernel messages
dmesg | grep -i error              # Kernel errors
dmesg | grep -i "usb\|disk\|eth"   # Hardware-related messages
```

> **When to use dmesg:** Hardware issues — disk failures, NIC problems, USB devices, OOM (Out of Memory) killer events. If a process gets killed mysteriously, check `dmesg | grep -i "oom\|killed"`.

### 9.8 rsyslog — Centralized Logging

rsyslog is the logging daemon that writes to `/var/log/` files and can forward logs to remote servers.

```
  ┌──────────────────────────────────────────────────────────────┐
  │                    RSYSLOG ARCHITECTURE                      │
  │                                                              │
  │  ┌──────────┐  ┌──────────┐  ┌──────────┐                    │
  │  │ Server A │  │ Server B │  │ Server C │                    │
  │  │ rsyslog  │  │ rsyslog  │  │ rsyslog  │                    │
  │  └────┬─────┘  └────┬─────┘  └────┬─────┘                    │
  │       │              │              │                        │
  │       └──────────────┼──────────────┘                        │
  │                      │                                       │
  │                      ▼                                       │
  │            ┌──────────────────┐                              │
  │            │  Central Syslog  │                              │
  │            │  Server (rsyslog │                              │
  │            │  or ELK/Splunk)  │                              │
  │            └──────────────────┘                              │
  │                                                              │
  │  Config: /etc/rsyslog.conf                                   │
  │  Format: facility.priority   action                          │
  │  Example: *.info             /var/log/messages               │
  │           authpriv.*         /var/log/secure                 │
  │           *.* @@10.0.1.50:514   ← Forward to remote (TCP)    |
  │           *.* @10.0.1.50:514    ← Forward to remote (UDP)    |
  └──────────────────────────────────────────────────────────────┘
```

> **DevOps Mapping:** rsyslog forwarding to a central server is the on-prem equivalent of CloudWatch Logs (AWS), Azure Monitor Logs, or sending to an ELK/Splunk stack. Understanding syslog = understanding log pipelines.

---

## 10. Shell Scripting & Environment Variables

### 10.1 Why Shell Scripting Matters for DevOps

- **Automation First:** If you do it twice, script it
- **Glue Language:** Shell connects all your tools — Docker, kubectl, terraform, aws-cli
- **CI/CD Pipelines:** GitHub Actions, Jenkins, GitLab CI — all run shell commands
- **Infrastructure Provisioning:** User data scripts, cloud-init, bootstrap scripts
- **Quick Prototyping:** Before writing Python/Go, prove the concept in Bash

### 10.2 Environment Variables — Mental Model

```
  ┌────────────────────────────────────────────────────────────┐
  │             ENVIRONMENT VARIABLE HIERARCHY                 │
  │                                                            │
  │  System-wide (all users):                                  │
  │  ┌──────────────────────────────────────────────────────┐  │
  │  │ /etc/environment      ← Simple KEY=VALUE pairs       │  │
  │  │ /etc/profile           ← Login shell (system-wide)   │  │
  │  │ /etc/profile.d/*.sh    ← Drop-in scripts ✅ preferred│  │
  │  │ /etc/bashrc            ← Non-login shells (system)   │  │
  │  └──────────────────────────────────────────────────────┘  │
  │                                                            │
  │  User-specific:                                            │
  │  ┌──────────────────────────────────────────────────────┐  │
  │  │ ~/.bash_profile        ← Login shell (user-specific) │  │
  │  │ ~/.bashrc              ← Non-login shells (user)     │  │
  │  │ ~/.bash_logout         ← Runs on logout              │  │
  │  └──────────────────────────────────────────────────────┘  │
  │                                                            │
  │  Load Order (login shell):                                 │
  │  /etc/profile → /etc/profile.d/*.sh → ~/.bash_profile      │
  │  → ~/.bashrc → /etc/bashrc                                 │
  │                                                            │
  │  Load Order (non-login shell, e.g., opening new terminal): │
  │  ~/.bashrc → /etc/bashrc                                   │
  └────────────────────────────────────────────────────────────┘
```

#### Common Environment Variables

| Variable | Purpose | Example |
|----------|---------|---------|
| `$PATH` | Directories searched for commands | `/usr/local/bin:/usr/bin:/bin` |
| `$HOME` | Current user's home directory | `/home/vishvam` |
| `$USER` | Current username | `vishvam` |
| `$SHELL` | Current shell | `/bin/bash` |
| `$HOSTNAME` | System hostname | `web-prod-01` |
| `$PWD` | Current working directory | `/opt/app` |
| `$EDITOR` | Default text editor | `vim` |
| `$LANG` | System locale | `en_US.UTF-8` |
| `$?` | Exit code of last command | `0` = success |
| `$$` | PID of current shell | `12345` |
| `$!` | PID of last background process | `12346` |

```bash
# View all environment variables
env
printenv

# View a specific variable
echo $PATH
printenv PATH

# Set variable (current session only)
export MY_VAR="hello"

# Set permanently for user
echo 'export MY_VAR="hello"' >> ~/.bashrc
source ~/.bashrc               # Reload without logging out

# Set permanently for all users
echo 'export COMPANY_ENV="production"' > /etc/profile.d/company.sh

# Unset a variable
unset MY_VAR

# Add to PATH
export PATH="$PATH:/opt/mytools/bin"
```

> **DevOps Relevance:** Environment variables are THE mechanism for injecting configuration into applications: `DATABASE_URL`, `API_KEY`, `NODE_ENV`. Docker, Kubernetes, CI/CD — all use env vars for config. This is the 12-Factor App methodology.

### 10.3 Bash Scripting Fundamentals

#### Script Structure

```bash
#!/bin/bash
# ^^^ Shebang — tells the OS which interpreter to use

# Script description: What this script does
# Author: Vishvam
# Date: 2026-02-23

set -euo pipefail
# set -e    → Exit immediately on any error
# set -u    → Treat unset variables as errors
# set -o pipefail → Pipe fails if ANY command in pipe fails
# ^^^ ALWAYS use this in production scripts

# Variables
APP_NAME="myapp"
LOG_FILE="/var/log/${APP_NAME}.log"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)

# Functions
log() {
    echo "[$(date +'%Y-%m-%d %H:%M:%S')] $*" | tee -a "$LOG_FILE"
}

# Main logic
log "Starting deployment of ${APP_NAME}"
# ... your code here ...
log "Deployment complete"
```

#### Variables & Quoting

```bash
# Variable assignment (NO SPACES around =)
name="Vishvam"           # ✅ Correct
name = "Vishvam"         # ❌ Error! Bash thinks 'name' is a command

# Variable usage
echo "$name"             # ✅ Always double-quote variables
echo "${name}_suffix"    # ✅ Use braces for clarity/concatenation

# Command substitution
today=$(date +%Y-%m-%d)          # Modern syntax ✅
today=`date +%Y-%m-%d`           # Legacy syntax (works but avoid)

# Arithmetic
count=$((count + 1))             # Integer arithmetic
result=$(echo "5.5 * 3.2" | bc)  # Floating point (needs bc)
```

#### Conditionals

```bash
# if-elif-else
if [[ -f "/etc/nginx/nginx.conf" ]]; then
    echo "Nginx config exists"
elif [[ -f "/etc/httpd/conf/httpd.conf" ]]; then
    echo "Apache config exists"
else
    echo "No web server config found"
fi

# Common test operators
[[ -f file ]]        # File exists and is regular file
[[ -d dir ]]         # Directory exists
[[ -e path ]]        # Path exists (file or dir)
[[ -r file ]]        # File is readable
[[ -w file ]]        # File is writable
[[ -x file ]]        # File is executable
[[ -s file ]]        # File exists and is not empty
[[ -z "$var" ]]      # Variable is empty/unset
[[ -n "$var" ]]      # Variable is NOT empty

# String comparison
[[ "$a" == "$b" ]]   # Equal
[[ "$a" != "$b" ]]   # Not equal
[[ "$a" =~ regex ]]  # Regex match

# Numeric comparison
[[ $a -eq $b ]]      # Equal
[[ $a -ne $b ]]      # Not equal
[[ $a -gt $b ]]      # Greater than
[[ $a -lt $b ]]      # Less than
[[ $a -ge $b ]]      # Greater or equal
[[ $a -le $b ]]      # Less or equal

# Logical operators
[[ cond1 && cond2 ]]  # AND
[[ cond1 || cond2 ]]  # OR
[[ ! condition ]]     # NOT
```

#### Loops

```bash
# for loop — iterate over list
for server in web01 web02 web03; do
    echo "Deploying to $server"
    ssh "$server" "/opt/deploy.sh"
done

# for loop — iterate over files
for file in /var/log/*.log; do
    echo "Processing: $file"
    gzip "$file"
done

# for loop — C-style
for ((i=1; i<=10; i++)); do
    echo "Iteration $i"
done

# for loop — iterate over command output
for user in $(cat /tmp/userlist.txt); do
    useradd "$user"
done

# while loop
counter=0
while [[ $counter -lt 10 ]]; do
    echo "Count: $counter"
    ((counter++))
done

# while loop — read file line by line (BEST PRACTICE)
while IFS= read -r line; do
    echo "Processing: $line"
done < /tmp/input.txt

# until loop (opposite of while)
until ping -c 1 google.com &>/dev/null; do
    echo "Waiting for network..."
    sleep 5
done
echo "Network is up!"
```

#### Functions

```bash
# Define a function
check_disk_space() {
    local threshold=${1:-80}         # Default parameter value = 80
    local usage=$(df -h / | awk 'NR==2 {print $5}' | tr -d '%')

    if [[ $usage -gt $threshold ]]; then
        echo "WARNING: Disk usage is ${usage}% (threshold: ${threshold}%)"
        return 1                      # Non-zero = failure
    else
        echo "OK: Disk usage is ${usage}%"
        return 0                      # Zero = success
    fi
}

# Call the function
check_disk_space 90
if [[ $? -ne 0 ]]; then
    # Send alert...
    echo "Sending alert..."
fi

# Or more concisely:
if ! check_disk_space 90; then
    echo "Disk space critical!"
fi
```

#### Input Handling & Arguments

```bash
#!/bin/bash
# Script: deploy.sh
# Usage: ./deploy.sh <environment> <version>

# Positional arguments
echo "Script name: $0"
echo "First argument: $1"
echo "Second argument: $2"
echo "All arguments: $@"
echo "Number of arguments: $#"

# Validation
if [[ $# -lt 2 ]]; then
    echo "Usage: $0 <environment> <version>"
    echo "Example: $0 production v2.1.0"
    exit 1
fi

ENVIRONMENT="$1"
VERSION="$2"

# Read user input
read -p "Are you sure you want to deploy $VERSION to $ENVIRONMENT? (y/n): " confirm
if [[ "$confirm" != "y" ]]; then
    echo "Deployment cancelled."
    exit 0
fi
```

#### Exit Codes & Error Handling

```bash
# Every command returns an exit code
# 0 = success, non-zero = failure

# Check exit code
if command; then
    echo "Success"
else
    echo "Failed with exit code: $?"
fi

# set -e: Exit on first error (add at top of script)
set -e

# Trap errors for cleanup
cleanup() {
    echo "Cleaning up temporary files..."
    rm -f /tmp/deploy_*.tmp
}
trap cleanup EXIT              # Run cleanup on exit (success or failure)
trap cleanup ERR               # Run cleanup on error

# Custom exit codes
exit 0    # Success
exit 1    # General error
exit 2    # Misuse of shell command
exit 126  # Command not executable
exit 127  # Command not found
exit 128  # Invalid exit argument
```

### 10.4 Real-World Script Patterns for DevOps

#### Pattern 1: Health Check Script

```bash
#!/bin/bash
set -euo pipefail

SERVICES=("nginx" "docker" "sshd")
FAILED=()

for svc in "${SERVICES[@]}"; do
    if ! systemctl is-active --quiet "$svc"; then
        FAILED+=("$svc")
        echo "[FAIL] $svc is not running"
    else
        echo "[OK]   $svc is running"
    fi
done

if [[ ${#FAILED[@]} -gt 0 ]]; then
    echo "ALERT: ${#FAILED[@]} service(s) down: ${FAILED[*]}"
    exit 1
fi
echo "All services healthy."
```

#### Pattern 2: Log Rotation with Cleanup

```bash
#!/bin/bash
# Rotate and compress logs older than 7 days, delete older than 30

LOG_DIR="/var/log/myapp"
find "$LOG_DIR" -name "*.log" -mtime +7 -exec gzip {} \;
find "$LOG_DIR" -name "*.log.gz" -mtime +30 -delete
echo "$(date): Log rotation complete" >> /var/log/myapp/rotation.log
```

#### Pattern 3: Deployment Script Skeleton

```bash
#!/bin/bash
set -euo pipefail

APP_NAME="myapp"
DEPLOY_DIR="/opt/${APP_NAME}"
BACKUP_DIR="/opt/backups/${APP_NAME}"
VERSION="${1:?Usage: $0 <version>}"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)

log() { echo "[$(date +'%H:%M:%S')] $*"; }

log "Starting deployment of ${APP_NAME} v${VERSION}"

# Backup current version
log "Backing up current version..."
mkdir -p "$BACKUP_DIR"
cp -rp "$DEPLOY_DIR" "${BACKUP_DIR}/${TIMESTAMP}"

# Deploy new version
log "Deploying v${VERSION}..."
cd "$DEPLOY_DIR"
git fetch --all
git checkout "v${VERSION}"

# Restart service
log "Restarting service..."
sudo systemctl restart "$APP_NAME"

# Verify
sleep 3
if systemctl is-active --quiet "$APP_NAME"; then
    log "✅ Deployment successful — ${APP_NAME} v${VERSION} is running"
else
    log "❌ Deployment failed — rolling back..."
    cp -rp "${BACKUP_DIR}/${TIMESTAMP}"/* "$DEPLOY_DIR/"
    sudo systemctl restart "$APP_NAME"
    log "Rolled back to previous version"
    exit 1
fi
```

> **Shell Script Golden Rules:**
> 1. Always start with `set -euo pipefail`
> 2. Always quote your variables: `"$var"` not `$var`
> 3. Use `[[ ]]` not `[ ]` for conditionals
> 4. Use `$()` not backticks for command substitution
> 5. Use `local` for function variables
> 6. Always validate inputs
> 7. Always handle errors and provide rollback
> 8. Log everything with timestamps

---

<!-- END OF BATCH 2 — Sections 6-10 -->

## 11. Networking Fundamentals & Commands

### 11.1 Why Networking is THE Core DevOps Skill

Every single thing in modern infrastructure is networked. Containers talk over networks. Kubernetes IS a network orchestrator. Cloud = someone else's network. If you can't debug networking, you can't debug production.

### 11.2 The TCP/IP Model — Mental Model

```
  ┌──────────────────────────────────────────────────────────────┐
  │           TCP/IP MODEL (What Linux Actually Uses)            │
  │                                                              │
  │  Layer 4: APPLICATION     HTTP, SSH, DNS, FTP, SMTP          │
  │           ─────────────────────────────────────              │
  │  Layer 3: TRANSPORT       TCP (reliable) / UDP (fast)        │
  │           ─────────────────────────────────────              │
  │  Layer 2: INTERNET        IP addressing, routing             │
  │           ─────────────────────────────────────              │
  │  Layer 1: NETWORK ACCESS  Ethernet, WiFi, ARP                │
  │                                                              │
  │  How data flows DOWN (sending):                              │
  │  ┌──────┐ → ┌──────┐ → ┌──────┐ → ┌──────┐ → 🔌 Wire         │
  │  │ HTTP │   │ TCP  │   │  IP  │   │ ETH  │                   │
  │  │ Data │   │+Port │   │+IP   │   │+MAC  │                   │
  │  └──────┘   └──────┘   └──────┘   └──────┘                   │
  │  Each layer WRAPS the previous (encapsulation)               │
  └──────────────────────────────────────────────────────────────┘
```

### 11.3 TCP vs UDP — When to Use Which

```
  ┌──────────────────────────┬──────────────────────────────┐
  │        TCP               │          UDP                 │
  │  (Transmission Control)  │  (User Datagram Protocol)    │
  ├──────────────────────────┼──────────────────────────────┤
  │  Connection-oriented     │  Connectionless              │
  │  3-way handshake         │  Fire and forget             │
  │  Reliable delivery       │  No guarantee                │
  │  Ordered packets         │  May arrive out of order     │
  │  Slower (overhead)       │  Faster (minimal overhead)   │
  │  Flow & congestion ctrl  │  No flow control             │
  │                          │                              │
  │  Used by:                │  Used by:                    │
  │  HTTP/S, SSH, FTP,       │  DNS (queries), DHCP,        │
  │  SMTP, databases         │  NTP, streaming, VoIP,       │
  │                          │  SNMP, syslog                │
  ├──────────────────────────┴──────────────────────────────┤
  │  DevOps Rule: If data integrity matters → TCP           │
  │               If speed matters & loss is OK → UDP       │
  └─────────────────────────────────────────────────────────┘
```

### 11.4 IP Addressing Quick Reference

```
  ┌──────────────────────────────────────────────────────────┐
  │  IPv4: 32-bit → 4 octets → 192.168.1.100                 │
  │  IPv6: 128-bit → 8 groups → 2001:0db8::1                 │
  │                                                          │
  │  Private IP Ranges (RFC 1918) — NOT routable on internet │
  │  ┌────────────────────────────────────────────────────┐  │
  │  │  10.0.0.0    - 10.255.255.255    (10.0.0.0/8)      │  │
  │  │  172.16.0.0  - 172.31.255.255   (172.16.0.0/12)    │  │
  │  │  192.168.0.0 - 192.168.255.255 (192.168.0.0/16)    │  │
  │  └────────────────────────────────────────────────────┘  │
  │                                                          │
  │  Special Addresses:                                      │
  │  127.0.0.1    = localhost (loopback)                     │
  │  0.0.0.0      = all interfaces / any address             │
  │  255.255.255.255 = broadcast                             │
  │  169.254.x.x  = APIPA (no DHCP response — self-assigned) │
  └──────────────────────────────────────────────────────────┘
```

> **DevOps Note:** Cloud VPCs use private IP ranges. Choosing the right CIDR avoids overlaps when peering VPCs or connecting on-prem to cloud via VPN. Plan your IP space early.

### 11.5 Essential Network Commands

#### ping — Connectivity Test

```bash
ping google.com                  # Continuous ping (Ctrl+C to stop)
ping -c 4 google.com             # Send exactly 4 packets
ping -c 4 -W 2 10.0.1.5          # 4 packets, 2-second timeout
```

> **What ping tells you:** Packet loss % and round-trip time (RTT). 0% loss + low RTT = healthy. High loss = network issue. "Destination host unreachable" = routing problem. "Request timed out" = firewall or host down.

#### ip — The Modern Network Swiss Army Knife

`ip` replaces `ifconfig`, `route`, and `arp` — all deprecated.

```bash
# View interfaces and IP addresses
ip addr show                     # Full details (or: ip a)
ip -4 addr show                  # IPv4 only
ip addr show eth0                # Specific interface

# View routing table
ip route show                    # All routes (or: ip r)
ip route get 8.8.8.8             # How would I reach this IP?

# View ARP cache (MAC ↔ IP mappings)
ip neigh show                    # Neighbour table (or: ip n)

# Add/remove IP (temporary — lost on reboot)
ip addr add 10.0.1.100/24 dev eth0
ip addr del 10.0.1.100/24 dev eth0

# Bring interface up/down
ip link set eth0 up
ip link set eth0 down

# Add a static route
ip route add 10.10.0.0/16 via 10.0.1.1 dev eth0
ip route del 10.10.0.0/16
```

#### traceroute — Path Discovery

```bash
traceroute google.com              # Show every hop to destination
traceroute -n google.com           # Numeric only (skip DNS lookups — faster)
traceroute -T google.com           # Use TCP instead of UDP (bypasses some firewalls)

# tracepath — simpler alternative (no root needed)
tracepath google.com
```

> **Reading traceroute:** Each line = one router hop. `* * *` = that router doesn't respond (firewall) — doesn't necessarily mean a problem. High latency jump between hops = congestion at that point.

#### curl & wget — HTTP/Download Tools

```bash
# curl — transfer data (HTTP, FTP, etc.)
curl https://api.example.com/health          # GET request
curl -I https://example.com                  # Headers only (HEAD)
curl -o file.tar.gz https://example.com/f    # Download to file
curl -O https://example.com/file.tar.gz      # Download, keep original name
curl -X POST -H "Content-Type: application/json" \
     -d '{"key":"value"}' https://api.com/endpoint   # POST with JSON
curl -u user:pass https://api.com/protected  # Basic auth
curl -k https://self-signed.example.com      # Skip SSL verification (testing only!)
curl -w "%{http_code}" -s -o /dev/null URL   # Get just the HTTP status code

# wget — download files (simpler, supports recursive)
wget https://example.com/file.tar.gz                  # Download
wget -O custom-name.tar.gz https://example.com/file   # Custom filename
wget -q URL                                           # Quiet mode
wget -r -l 2 https://example.com/docs/                # Recursive download (depth 2)
wget -c https://example.com/large-file.iso             # Resume interrupted download
```

> **curl vs wget:** Use `curl` for API interactions, testing endpoints, CI/CD scripts. Use `wget` for downloading files, mirroring sites, resumable downloads.

### 11.6 Network Ports — What DevOps Engineers Must Know

| Port | Service | Protocol | Why You Care |
|------|---------|----------|-------------|
| 22 | SSH | TCP | Remote server access — **THE** DevOps port |
| 80 | HTTP | TCP | Web traffic (unencrypted) |
| 443 | HTTPS | TCP | Web traffic (encrypted) — always use this |
| 53 | DNS | TCP/UDP | Name resolution — debugging starts here |
| 25 | SMTP | TCP | Email sending |
| 110/143 | POP3/IMAP | TCP | Email receiving |
| 3306 | MySQL | TCP | Database |
| 5432 | PostgreSQL | TCP | Database |
| 6379 | Redis | TCP | Cache/queue |
| 27017 | MongoDB | TCP | NoSQL database |
| 8080 | Alt HTTP | TCP | App servers, proxies, dev servers |
| 9090 | Prometheus | TCP | Monitoring |
| 3000 | Grafana/Dev | TCP | Dashboards, Node.js dev |
| 2379 | etcd | TCP | Kubernetes state store |
| 6443 | K8s API | TCP | Kubernetes API server |
| 10250 | Kubelet | TCP | Kubernetes node agent |

> **Security Rule:** Only expose the ports you need. Every open port is an attack surface. Use firewalls (`firewalld`/`iptables`) to allow only required ports.

---

## 12. Remote Access & File Transfer — SSH, SCP, rsync

### 12.1 SSH — Secure Shell (The Foundation of Remote Access)

SSH is how you access remote servers. Period. Every DevOps engineer uses it hundreds of times a day.

#### SSH Architecture — Mental Model

```
  ┌────────────────────────────────────────────────────────────┐
  │                  SSH CONNECTION FLOW                       │
  │                                                            │
  │  Local Machine                    Remote Server            │
  │  ┌──────────────┐                ┌──────────────────┐      │
  │  │ SSH Client   │───── TCP:22 ──►│ SSH Daemon (sshd)│      │
  │  │              │                │                  │      │
  │  │ ~/.ssh/      │                │ ~/.ssh/          │      │
  │  │  id_rsa      │ Private Key    │  authorized_keys │      │
  │  │  id_rsa.pub  │ Public Key ───►│  (your pub key)  │      │
  │  │  known_hosts │ Server FP      │                  │      │
  │  │  config      │ SSH config     │ /etc/ssh/sshd_   │      │
  │  └──────────────┘                │  config          │      │
  │                                  └──────────────────┘      │
  │                                                            │
  │  Authentication Methods (in order of preference):          │
  │  1. 🔑 Key-based (most secure, recommended)                │
  │  2. 🔒 Password (convenient, less secure)                  │
  │  3. 🎫 Certificate-based (enterprise scale)                │
  └────────────────────────────────────────────────────────────┘
```

#### SSH Key-Based Authentication — Step by Step

```bash
# Step 1: Generate key pair on LOCAL machine
ssh-keygen -t ed25519 -C "vishvam@corp.com"
# Or RSA: ssh-keygen -t rsa -b 4096 -C "vishvam@corp.com"
# Saves to: ~/.ssh/id_ed25519 (private) and ~/.ssh/id_ed25519.pub (public)

# Step 2: Copy public key to REMOTE server
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@remote-server
# This appends your pub key to remote's ~/.ssh/authorized_keys

# Step 3: Connect (no password needed!)
ssh user@remote-server

# Step 4: Harden — disable password auth on server
sudo sed -i 's/^#PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config
sudo systemctl restart sshd
```

> **Key Type Choice:** Use `ed25519` (modern, fast, short keys). Fall back to `rsa -b 4096` only if the remote system doesn't support ed25519.

#### SSH Commands & Tricks

```bash
# Basic connection
ssh user@hostname                    # Connect
ssh -p 2222 user@hostname            # Non-standard port
ssh -i ~/.ssh/custom_key user@host   # Specific key file

# Execute command remotely (don't open shell)
ssh user@server "uptime && df -h"
ssh user@server "sudo systemctl restart nginx"

# SSH tunneling / port forwarding
# Local: access remote service through your localhost
ssh -L 8080:localhost:80 user@server
# Now: http://localhost:8080 → server's port 80

# Remote: expose your local service on remote server
ssh -R 9090:localhost:3000 user@server
# Now: server:9090 → your localhost:3000

# Dynamic SOCKS proxy
ssh -D 1080 user@server
# Configure browser to use SOCKS5 proxy on localhost:1080

# Jump host / bastion (ProxyJump)
ssh -J jumphost user@internal-server
# Connects: you → jumphost → internal-server

# Keep connection alive
ssh -o ServerAliveInterval=60 user@server

# Agent forwarding (use your local keys on remote)
ssh -A user@bastion
# Now on bastion, you can SSH to other servers using YOUR keys
```

#### ~/.ssh/config — The Productivity Booster

Instead of typing long SSH commands, configure shortcuts:

```
# ~/.ssh/config

Host web-prod
    HostName 10.0.1.50
    User deploy
    Port 22
    IdentityFile ~/.ssh/id_ed25519

Host bastion
    HostName bastion.corp.com
    User vishvam
    IdentityFile ~/.ssh/id_ed25519

Host internal-*
    ProxyJump bastion
    User admin
    IdentityFile ~/.ssh/id_ed25519

Host db-staging
    HostName 10.0.2.30
    User dbadmin
    LocalForward 5432 localhost:5432
    # Now: psql -h localhost connects to remote DB
```

```bash
# Now just type:
ssh web-prod                    # Instead of: ssh -i ~/.ssh/id_ed25519 deploy@10.0.1.50
ssh internal-app01              # Automatically jumps through bastion
```

> **DevOps Best Practice:** Every engineer should have an SSH config. It eliminates errors, speeds up access, and documents your infrastructure implicitly.

### 12.2 SCP — Secure Copy

SCP copies files over SSH. Simple and reliable.

```bash
# Copy file TO remote server
scp file.txt user@server:/remote/path/
scp -P 2222 file.txt user@server:/path/     # Non-standard port

# Copy file FROM remote server
scp user@server:/remote/file.txt /local/path/

# Copy entire directory (recursive)
scp -r /local/dir/ user@server:/remote/path/

# Copy with specific key
scp -i ~/.ssh/id_ed25519 file.txt user@server:/path/

# Copy between two remote servers (from your machine)
scp user@server1:/file.txt user@server2:/path/

# Preserve timestamps and permissions
scp -rp /local/dir/ user@server:/path/
```

### 12.3 rsync — The Superior File Sync Tool

rsync is scp's **big brother**. It transfers only the **differences**, making it dramatically faster for repeated syncs.

#### rsync vs scp — Why rsync Wins

```
  ┌──────────────────────────────────────────────────────────┐
  │  scp:   Copies EVERYTHING every time (full transfer)     │
  │  rsync: Copies ONLY what changed (delta transfer)        │
  │                                                          │
  │  First sync of 10GB directory:                           │
  │    scp:   transfers 10 GB                                │
  │    rsync: transfers 10 GB (same — no prior data)         │
  │                                                          │
  │  Second sync (5 files changed, 50 MB total):             │
  │    scp:   transfers 10 GB again 😱                       │
  │    rsync: transfers 50 MB only ✅                        │
  │                                                          │
  │  rsync also supports:                                    │
  │  • Resuming interrupted transfers (--partial)            │
  │  • Excluding files (--exclude)                           │
  │  • Compression during transfer (-z)                      │
  │  • Dry run (--dry-run, -n)                               │
  │  • Deleting files on destination that don't exist        │
  │    on source (--delete) — true mirror                    │
  └──────────────────────────────────────────────────────────┘
```

```bash
# Basic local sync
rsync -avh /source/ /destination/
# -a = archive (recursive, preserves perms, timestamps, symlinks, etc.)
# -v = verbose
# -h = human-readable sizes

# Sync to remote server (over SSH)
rsync -avh /local/dir/ user@server:/remote/dir/

# Sync from remote server
rsync -avh user@server:/remote/dir/ /local/dir/

# Dry run — see what WOULD be transferred (no changes)
rsync -avhn /source/ /destination/

# With compression (good for slow links)
rsync -avhz /source/ user@server:/dest/

# Delete files on destination that don't exist on source (mirror)
rsync -avh --delete /source/ /destination/
# ⚠️ Dangerous! Always dry-run first: rsync -avhn --delete

# Exclude patterns
rsync -avh --exclude='*.log' --exclude='node_modules/' /source/ /dest/

# Resume interrupted transfer
rsync -avh --partial --progress /source/ user@server:/dest/

# Limit bandwidth (in KB/s)
rsync -avh --bwlimit=5000 /source/ user@server:/dest/
```

> **Trailing Slash Matters!**
> - `rsync /source/ /dest/` → copies **contents** of source INTO dest
> - `rsync /source /dest/` → copies the **directory itself** INTO dest (creates /dest/source/)
> - This is the #1 rsync gotcha. Always think about the trailing slash.

> **DevOps Uses:** Deployment sync, backup scripts, log shipping, config distribution across servers. rsync + cron = poor man's continuous backup.

### 12.4 FTP — File Transfer Protocol (Legacy)

FTP is **unencrypted** and considered insecure. Know it exists, but prefer SFTP/SCP/rsync.

```bash
# SFTP (SSH-based, encrypted) — preferred
sftp user@server
sftp> put localfile.txt                # Upload
sftp> get remotefile.txt               # Download
sftp> ls                               # List remote files
sftp> lcd /local/path                  # Change local directory
sftp> bye                              # Exit

# FTP (legacy — don't use for sensitive data)
ftp server
ftp> put file
ftp> get file
ftp> bye
```

> **DevOps Verdict:** Never use plain FTP. Use SFTP (over SSH), SCP, or rsync. If a legacy system requires FTP, use FTPS (FTP over TLS) at minimum.

---

## 13. Network Configuration & DNS

### 13.1 Network Configuration Tools

Linux offers multiple ways to configure networking. Understanding the hierarchy is key.

```
  ┌───────────────────────────────────────────────────────────────┐
  │          NETWORK CONFIGURATION HIERARCHY                      │
  │                                                               │
  │  ┌──────────────────────────────────────────────────────┐     │
  │  │  NetworkManager (nmcli, nmtui, nm-connection-editor) │     │
  │  │  • Default on RHEL 7+, CentOS, Fedora, Ubuntu Desktop│     │
  │  │  • Manages connections dynamically                   │     │
  │  │  • Persists config to /etc/NetworkManager/           │     │
  │  └────────────────────────┬─────────────────────────────┘     │
  │                           │ controls                          │
  │  ┌────────────────────────▼────────────────────────────────┐  │
  │  │  Config Files                                           │  │
  │  │  RHEL/CentOS:                                           │  │
  │  │    /etc/sysconfig/network-scripts/ifcfg-eth0            │  │
  │  │  RHEL 9+/Fedora:                                        │  │
  │  │    /etc/NetworkManager/system-connections/*.nmconnection│  │
  │  │  Ubuntu:                                                │  │
  │  │    /etc/netplan/*.yaml                                  │  │
  │  └─────────────────────────────────────────────────────────┘  │
  │                                                               │
  │  ┌──────────────────────────────────────────────────────┐     │
  │  │  ip command (temporary — lost on reboot)             │     │
  │  │  For quick testing only, not persistent config       │     │
  │  └──────────────────────────────────────────────────────┘     │
  └───────────────────────────────────────────────────────────────┘
```

#### nmcli — NetworkManager CLI

```bash
# View all connections
nmcli connection show
nmcli con show

# View device status
nmcli device status

# View details of a connection
nmcli con show "Wired connection 1"

# Set static IP
nmcli con mod "eth0" \
  ipv4.addresses 10.0.1.100/24 \
  ipv4.gateway 10.0.1.1 \
  ipv4.dns "8.8.8.8,8.8.4.4" \
  ipv4.method manual

# Switch to DHCP
nmcli con mod "eth0" ipv4.method auto

# Bring connection up/down
nmcli con up "eth0"
nmcli con down "eth0"

# Add a new connection
nmcli con add type ethernet con-name "prod-net" ifname eth1 \
  ipv4.addresses 10.0.2.50/24 \
  ipv4.gateway 10.0.2.1 \
  ipv4.method manual

# Reload after config file changes
nmcli con reload
```

#### nmtui — Text User Interface

```bash
nmtui                          # Launch interactive TUI
# Menu options:
# 1. Edit a connection    → modify IP, DNS, gateway
# 2. Activate a connection → up/down interfaces
# 3. Set system hostname
```

> **nmcli vs nmtui:** Use `nmtui` for quick interactive changes (like when you're at the console). Use `nmcli` in scripts and automation — it's fully scriptable.

### 13.2 DNS Configuration

#### Client-Side DNS Resolution

```
  ┌────────────────────────────────────────────────────────────┐
  │          DNS RESOLUTION ORDER (on the client)              │
  │                                                            │
  │  1. /etc/hosts         ← Static mappings (checked first!)  │
  │  2. DNS Server         ← Configured in /etc/resolv.conf    │
  │                                                            │
  │  Order controlled by: /etc/nsswitch.conf                   │
  │    hosts: files dns    ← "files" (hosts) first, then dns   │
  └────────────────────────────────────────────────────────────┘
```

```bash
# /etc/hosts — static hostname mapping
# Format: IP   FQDN   alias
127.0.0.1   localhost
10.0.1.50   web-prod-01.corp.com   web-prod-01
10.0.1.51   db-prod-01.corp.com    db-prod-01

# /etc/resolv.conf — DNS server config (often auto-generated by NetworkManager)
nameserver 8.8.8.8
nameserver 8.8.4.4
search corp.com             # Append this domain to unqualified names
# So: ping web01 → tries web01.corp.com
```

> **DevOps Note:** In containers and Kubernetes, `/etc/resolv.conf` is auto-generated. Understanding it helps debug DNS failures in pods.

#### DNS Lookup Tools — nslookup and dig

```bash
# nslookup — basic DNS query
nslookup google.com                      # Forward lookup
nslookup 8.8.8.8                         # Reverse lookup
nslookup -type=MX google.com             # MX records
nslookup -type=NS google.com             # Nameserver records
nslookup google.com 1.1.1.1              # Query specific DNS server

# dig — advanced DNS query (DevOps preferred)
dig google.com                           # Full query with details
dig google.com +short                    # Just the answer
dig google.com MX                        # MX records
dig google.com NS                        # Nameserver records
dig google.com @1.1.1.1                  # Query specific DNS server
dig -x 8.8.8.8                           # Reverse lookup
dig google.com +trace                    # Trace full resolution path
dig google.com ANY                       # All record types
```

> **dig vs nslookup:** `dig` gives more detail, supports `+trace` for debugging resolution chains, and is scriptable. `nslookup` is simpler for quick checks. DevOps preference: `dig`.

### 13.3 Local DNS Server — BIND

BIND (Berkeley Internet Name Domain) is the most widely used DNS server software.

```
  ┌────────────────────────────────────────────────────────────┐
  │                  BIND DNS SERVER                           │
  │                                                            │
  │  Config: /etc/named.conf                                   │
  │  Zone Files: /var/named/                                   │
  │  Service: named (or named-chroot for security)             │
  │                                                            │
  │  DNS Server Roles:                                         │
  │  ┌──────────────────┐  ┌────────────────────────────────┐  │
  │  │  Authoritative   │  │  Recursive/Caching             │  │
  │  │  • Owns zone data│  │  • Resolves on behalf of client│  │
  │  │  • Answers for   │  │  • Caches responses            │  │
  │  │    your domains  │  │  • Forwards to upstream        │  │
  │  │  • Primary/      │  │  • Like: 8.8.8.8, 1.1.1.1      │  │
  │  │    Secondary     │  │                                │  │
  │  └──────────────────┘  └────────────────────────────────┘  │
  │                                                            │
  │  Zone File Anatomy (Forward Zone):                         │
  │  ┌──────────────────────────────────────────────────────┐  │
  │  │  $TTL 86400                                          │  │
  │  │  @   IN  SOA  ns1.corp.com. admin.corp.com. (        │  │
  │  │              2026022301  ; Serial                    │  │
  │  │              3600        ; Refresh                   │  │
  │  │              900         ; Retry                     │  │
  │  │              604800      ; Expire                    │  │
  │  │              86400 )     ; Min TTL                   │  │
  │  │  @   IN  NS   ns1.corp.com.                          │  │
  │  │  @   IN  A    10.0.1.10                              │  │
  │  │  ns1 IN  A    10.0.1.10                              │  │
  │  │  web IN  A    10.0.1.50                              │  │
  │  │  db  IN  A    10.0.1.51                              │  │
  │  │  www IN  CNAME web.corp.com.                         │  │
  │  └──────────────────────────────────────────────────────┘  │
  └────────────────────────────────────────────────────────────┘
```

```bash
# Install BIND
yum install bind bind-utils

# Key commands
systemctl start named
systemctl enable named
named-checkconf                        # Validate config syntax
named-checkzone corp.com /var/named/corp.com.zone   # Validate zone file
```

> **DevOps Relevance:** You probably won't run BIND in production (use cloud DNS: Route 53, Cloud DNS, Azure DNS). But understanding zone files, record types, SOA, and TTL is essential for managing DNS anywhere — cloud included. TTL too high = slow failover. TTL too low = more DNS queries = more cost.

---

## 14. Package Management

### 14.1 Why Package Management Matters

- **Consistent environments:** Same packages, same versions across all servers
- **Security patching:** Automated updates close vulnerabilities
- **Dependency management:** Package managers resolve dependencies automatically
- **Rollback capability:** Undo bad updates
- **Compliance:** Auditors want to know what's installed and what version

### 14.2 Package Manager Landscape — Mental Model

```
  ┌──────────────────────────────────────────────────────────────┐
  │              LINUX PACKAGE MANAGERS                          │
  │                                                              │
  │  RPM-based (RHEL, CentOS, Fedora, Amazon Linux):             │
  │  ┌──────────────────────────────────────────────────────┐    │
  │  │  rpm          ← Low-level: install .rpm files        │    │
  │  │  yum          ← High-level: resolves dependencies    │    │
  │  │  dnf          ← Modern replacement for yum (RHEL 8+) │    │
  │  └──────────────────────────────────────────────────────┘    │
  │                                                              │
  │  DEB-based (Ubuntu, Debian):                                 │
  │  ┌──────────────────────────────────────────────────────┐    │
  │  │  dpkg         ← Low-level: install .deb files        │    │
  │  │  apt / apt-get← High-level: resolves dependencies    │    │
  │  └──────────────────────────────────────────────────────┘    │
  │                                                              │
  │  Universal:                                                  │
  │  ┌──────────────────────────────────────────────────────┐    │
  │  │  snap         ← Canonical's universal packages       │    │
  │  │  flatpak      ← Desktop app packaging                │    │
  │  └──────────────────────────────────────────────────────┘    │
  └──────────────────────────────────────────────────────────────┘
```

### 14.3 yum / dnf — The High-Level Managers

yum and dnf are functionally equivalent for daily use. dnf is the modern successor.

```bash
# Search for a package
yum search nginx
dnf search nginx

# Get package info
yum info nginx

# Install a package
yum install nginx -y                # -y = auto-confirm
dnf install nginx -y

# Remove a package
yum remove nginx -y

# Update a specific package
yum update openssl -y

# Update ALL packages (system upgrade)
yum update -y

# List installed packages
yum list installed
yum list installed | grep nginx

# Check what package provides a file
yum provides /usr/bin/dig             # → bind-utils
yum whatprovides */traceroute

# View package dependencies
yum deplist nginx

# Clean cached data
yum clean all

# View transaction history
yum history
yum history info <ID>                 # Details of a specific transaction

# ROLLBACK an update!
yum history undo <ID> -y              # Undo a specific transaction
```

> **`yum history undo` is a lifesaver.** Updated a package that broke production? `yum history` → find the transaction ID → `yum history undo <ID>`. Instant rollback.

### 14.4 rpm — Low-Level Package Tool

```bash
# Install an .rpm file (does NOT resolve dependencies)
rpm -ivh package.rpm               # Install, verbose, hash progress
rpm -Uvh package.rpm               # Upgrade (or install if not present)

# Query installed packages
rpm -qa                             # List ALL installed packages
rpm -qa | grep ssh                  # Find SSH-related packages
rpm -qi nginx                       # Info about installed package
rpm -ql nginx                       # List files installed by package
rpm -qf /usr/sbin/nginx             # What package owns this file?

# Verify package integrity
rpm -V nginx                        # Check if files have been modified

# Remove package
rpm -e nginx
```

### 14.5 which — Finding Binaries

```bash
which nginx                  # /usr/sbin/nginx — shows binary location
which python3                # /usr/bin/python3
which kubectl                # Find if/where kubectl is installed

whereis nginx                # Shows binary, source, and man page locations
type nginx                   # Shows how the shell resolves the command
```

### 14.6 System Update Strategy for DevOps

```
  ┌────────────────────────────────────────────────────────────┐
  │           UPDATE STRATEGY — BEST PRACTICES                 │
  │                                                            │
  │  1. NEVER run `yum update -y` blindly on production        │
  │                                                            │
  │  2. Test updates on staging/dev FIRST                      │
  │     staging$ yum update -y                                 │
  │     → run tests → verify → only then push to prod          │
  │                                                            │
  │  3. Security updates only (targeted):                      │
  │     yum update --security -y                               │
  │                                                            │
  │  4. Lock critical package versions:                        │
  │     yum install yum-plugin-versionlock                     │
  │     yum versionlock nginx-1.24.*                           │
  │                                                            │
  │  5. Always snapshot/backup before major updates            │
  │                                                            │
  │  6. Automate with:                                         │
  │     • yum-cron (auto-apply security patches)               │
  │     • Ansible playbooks for controlled rollouts            │
  │     • Cloud: AMI/image-based updates (immutable infra)     │
  └────────────────────────────────────────────────────────────┘
```

> **Immutable Infrastructure Pattern (DevOps):** Instead of updating servers in-place, build new AMIs/images with updates → deploy new instances → destroy old ones. Never patch a running production server. This is the cloud-native approach.

---

## 15. Storage & Disk Management — fdisk, LVM, NFS

### 15.1 Why Storage Matters for DevOps

- **Disk full = production down:** The #1 outage cause after "DNS issue"
- **LVM = flexible volumes:** Resize without downtime — essential for growing databases
- **NFS = shared storage:** Multiple servers accessing the same files
- **Cloud mapping:** LVM concepts map directly to EBS volumes, Azure Disks, Persistent Volumes in K8s

### 15.2 Storage Architecture — Mental Model

```
  ┌────────────────────────────────────────────────────────────────┐
  │               LINUX STORAGE STACK                              │
  │                                                                │
  │  Physical Disks (/dev/sda, /dev/nvme0n1)                       │
  │       │                                                        │
  │       ▼                                                        │
  │  ┌──────────────────────────────────────────────┐              │
  │  │  Partition Table (MBR or GPT)                │              │
  │  │  /dev/sda1, /dev/sda2, ...                   │              │
  │  └──────────────────────┬───────────────────────┘              │
  │                         │                                      │
  │           ┌─────────────┴──────────────┐                       │
  │           │                            │                       │
  │     Direct Use                    LVM (Recommended)            │
  │     ┌──────────┐           ┌──────────────────────────┐        │
  │     │ mkfs.xfs │           │ PV → VG → LV             │        │
  │     │ /dev/sda1│           │ Physical  Volume  Logical│        │
  │     └────┬─────┘           │ Volume    Group   Volume │        │
  │          │                 └──────────┬───────────────┘        │
  │          │                            │                        │
  │          ▼                            ▼                        │
  │  ┌───────────────┐          ┌───────────────┐                  │
  │  │ Filesystem    │          │ Filesystem    │                  │
  │  │ (xfs, ext4)   │          │ (xfs, ext4)   │                  │
  │  └───────┬───────┘          └───────┬───────┘                  │
  │          │                          │                          │
  │          ▼                          ▼                          │
  │  ┌───────────────┐          ┌───────────────┐                  │
  │  │ Mount Point   │          │ Mount Point   │                  │
  │  │ /data         │          │ /var/lib/mysql│                  │
  │  └───────────────┘          └───────────────┘                  │
  └────────────────────────────────────────────────────────────────┘
```

### 15.3 Disk Information Commands

```bash
# List block devices (best overview)
lsblk                               # Tree view of disks, partitions, mount points
lsblk -f                            # Include filesystem type and UUIDs

# Disk space usage (filesystem level)
df -hT                               # Usage + filesystem type

# Disk hardware info
fdisk -l                             # List all disks and partitions (root required)
blkid                                # Show UUIDs and filesystem types

# Check disk health (SMART)
smartctl -a /dev/sda                  # Requires smartmontools
```

### 15.4 Partitioning — fdisk and gdisk

```bash
# fdisk — MBR partitions (disks ≤ 2TB)
fdisk /dev/sdb                       # Interactive partition editor
# Commands inside fdisk:
#   n = new partition
#   p = print partition table
#   d = delete partition
#   t = change partition type
#   w = write changes and exit
#   q = quit without saving

# gdisk — GPT partitions (disks > 2TB, modern systems)
gdisk /dev/sdb                       # Same interactive flow

# parted — supports both MBR and GPT, scriptable
parted /dev/sdb mklabel gpt
parted /dev/sdb mkpart primary xfs 0% 100%
```

#### MBR vs GPT

| Feature | MBR | GPT |
|---------|-----|-----|
| Max Disk Size | 2 TB | 9.4 ZB (zetabytes) |
| Max Partitions | 4 primary (or 3 primary + extended) | 128 |
| Boot Mode | BIOS | UEFI |
| Redundancy | No backup | Backup partition table at end of disk |
| Modern Use | Legacy systems only | **Default for everything new** |

### 15.5 Filesystems — mkfs

```bash
# Create filesystem
mkfs.xfs /dev/sdb1                   # XFS (default for RHEL/CentOS)
mkfs.ext4 /dev/sdb1                  # ext4 (default for Ubuntu/Debian)

# Mount filesystem
mkdir /data
mount /dev/sdb1 /data

# Make mount persistent (survives reboot)
# Add to /etc/fstab:
echo "/dev/sdb1  /data  xfs  defaults  0 0" >> /etc/fstab
# Or better, use UUID:
echo "UUID=$(blkid -s UUID -o value /dev/sdb1)  /data  xfs  defaults  0 0" >> /etc/fstab

# Verify fstab (dry run — catches errors before reboot!)
mount -a                              # Mount all entries in fstab

# Unmount
umount /data
```

#### XFS vs ext4

| Feature | XFS | ext4 |
|---------|-----|------|
| Default on | RHEL, CentOS, Amazon Linux | Ubuntu, Debian |
| Max Volume | 500 TB (practical) | 1 EB |
| Online Resize | Grow only (no shrink) | Grow and shrink |
| Performance | Better for large files, parallel I/O | Better for small files |
| Defragmentation | `xfs_fsr` | `e4defrag` |
| Repair | `xfs_repair` | `fsck.ext4` |
| **DevOps Choice** | Production servers (RHEL ecosystem) | Ubuntu/Debian servers |

### 15.6 LVM — Logical Volume Manager

LVM is the **most important storage concept** for Linux servers. It adds a layer of abstraction between physical disks and filesystems, enabling **flexibility**.

#### Why LVM? — The Problem It Solves

```
  ❌ Without LVM:
  /dev/sda1 (50GB) mounted on /data
  Database grows to 60GB → PANIC!
  You can't resize the partition without:
  • Unmounting (downtime!)
  • Backup → delete → recreate → restore

  ✅ With LVM:
  /dev/mapper/data-lv (50GB) mounted on /data
  Database grows to 60GB → No problem!
  • Add another disk, extend the LV online
  • Zero downtime, zero data loss
```

#### LVM Architecture — Mental Model

```
  ┌──────────────────────────────────────────────────────────────┐
  │                    LVM LAYERS                                │
  │                                                              │
  │  Physical Disks                                              │
  │  ┌──────┐ ┌──────┐ ┌──────┐                                  │
  │  │/dev/ │ │/dev/ │ │/dev/ │                                  │
  │  │ sdb  │ │ sdc  │ │ sdd  │                                  │
  │  └──┬───┘ └──┬───┘ └──┬───┘                                  │
  │     │        │        │                                      │
  │     ▼        ▼        ▼                                      │
  │  ┌──────┐ ┌──────┐ ┌──────┐    PV = Physical Volume          │
  │  │ PV   │ │ PV   │ │ PV   │    (pvcreate — mark disks        │
  │  │ sdb  │ │ sdc  │ │ sdd  │     for LVM use)                 │
  │  └──┬───┘ └──┬───┘ └──┬───┘                                  │
  │     └────────┼────────┘                                      │
  │              ▼                                               │
  │  ┌──────────────────────────┐  VG = Volume Group             │
  │  │  VG: data-vg             │  (vgcreate — pool PVs          │
  │  │  Total: 150 GB           │   together as one big pool)    │
  │  └────────────┬─────────────┘                                │
  │        ┌──────┴──────┐                                       │
  │        ▼             ▼                                       │
  │  ┌──────────┐ ┌──────────┐     LV = Logical Volume           │
  │  │ LV: db   │ │ LV: logs │     (lvcreate — allocate          │
  │  │ 80 GB    │ │ 40 GB    │      from VG pool)                │
  │  └────┬─────┘ └────┬─────┘                                   │
  │       │             │                                        │
  │       ▼             ▼                                        │
  │  /var/lib/mysql  /var/log       ← Mount points               │
  │                                                              │
  │  Free space in VG: 30 GB (can extend either LV anytime!)     │
  └──────────────────────────────────────────────────────────────┘
```

#### LVM Commands — Full Lifecycle

```bash
# ── STEP 1: Create Physical Volumes ──
pvcreate /dev/sdb /dev/sdc
pvs                              # List PVs
pvdisplay                        # Detailed PV info

# ── STEP 2: Create Volume Group ──
vgcreate data-vg /dev/sdb /dev/sdc
vgs                              # List VGs
vgdisplay                        # Detailed VG info

# ── STEP 3: Create Logical Volumes ──
lvcreate -n db-lv -L 80G data-vg        # Fixed size
lvcreate -n logs-lv -l 100%FREE data-vg  # Use ALL remaining space
lvs                              # List LVs
lvdisplay                        # Detailed LV info

# ── STEP 4: Create Filesystem & Mount ──
mkfs.xfs /dev/data-vg/db-lv
mkdir /var/lib/mysql
mount /dev/data-vg/db-lv /var/lib/mysql

# Add to /etc/fstab for persistence
echo "/dev/data-vg/db-lv  /var/lib/mysql  xfs  defaults  0 0" >> /etc/fstab

# ── EXTENDING (The Killer Feature) ──
# Scenario: need more space for database

# Option A: Extend using free space in VG
lvextend -L +20G /dev/data-vg/db-lv     # Add 20G
xfs_growfs /var/lib/mysql                # Grow XFS filesystem (online!)
# Or for ext4: resize2fs /dev/data-vg/db-lv

# Option B: Add a new disk to the VG first
pvcreate /dev/sdd                         # New disk
vgextend data-vg /dev/sdd                 # Add to VG pool
lvextend -L +50G /dev/data-vg/db-lv      # Now extend LV
xfs_growfs /var/lib/mysql                 # Grow filesystem

# One-liner: extend and resize in one command
lvextend -r -L +20G /dev/data-vg/db-lv   # -r = auto-resize filesystem
```

> **DevOps Mapping:**
> - LVM Physical Volume ≈ EBS Volume (AWS) / Managed Disk (Azure)
> - LVM Volume Group ≈ Storage Pool
> - LVM Logical Volume ≈ Partition you actually use
> - `lvextend` ≈ Resizing an EBS volume in AWS console + `xfs_growfs`

### 15.7 dd — Disk Copy/Imaging Tool

```bash
# Create a disk image (backup entire disk)
dd if=/dev/sda of=/backup/sda.img bs=4M status=progress

# Restore from image
dd if=/backup/sda.img of=/dev/sda bs=4M status=progress

# Create a bootable USB
dd if=rhel-9.iso of=/dev/sdb bs=4M status=progress

# Wipe a disk (fill with zeros)
dd if=/dev/zero of=/dev/sdb bs=4M status=progress

# Create a swap file (1GB)
dd if=/dev/zero of=/swapfile bs=1M count=1024
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile
```

> ⚠️ **dd is called "disk destroyer" for a reason.** Double-check `if=` (input) and `of=` (output). Reversing them DESTROYS your data instantly. There is no confirmation prompt.

### 15.8 NFS — Network File System

NFS allows sharing directories across the network. Multiple servers access the same files.

#### NFS Architecture

```
  ┌──────────────────────────────────────────────────────────┐
  │                   NFS ARCHITECTURE                       │
  │                                                          │
  │  NFS Server                     NFS Clients              │
  │  ┌──────────────────┐           ┌──────────────┐         │
  │  │ /shared/data     │◄─ mount ──│ /mnt/data    │         │
  │  │                  │           │ (mount point)│         │
  │  │ /etc/exports:    │           └──────────────┘         │
  │  │ /shared/data     │           ┌──────────────┐         │
  │  │ 10.0.1.0/24(rw,  │◄─ mount ──│ /mnt/data    │         │
  │  │ sync,no_root_    │           │              │         │
  │  │ squash)          │           └──────────────┘         │
  │  └──────────────────┘                                    │
  │                                                          │
  │  Key options in /etc/exports:                            │
  │  rw           = read-write access                        │
  │  ro           = read-only access                         │
  │  sync         = write to disk before responding (safe)   │
  │  async        = respond before write completes (fast)    │
  │  no_root_squash = allow root on client to be root on     │
  │                   server (⚠️ security risk)              │
  │  root_squash  = map remote root to nobody (default/safe) │
  └──────────────────────────────────────────────────────────┘
```

#### NFS Setup Commands

```bash
# ── ON THE SERVER ──
# Install NFS
yum install nfs-utils -y

# Create shared directory
mkdir -p /shared/data
chown nobody:nobody /shared/data

# Configure exports
echo "/shared/data 10.0.1.0/24(rw,sync,no_root_squash)" >> /etc/exports

# Export and start service
exportfs -rav                       # Apply export changes
systemctl start nfs-server
systemctl enable nfs-server

# ── ON THE CLIENT ──
# Install NFS utilities
yum install nfs-utils -y

# Check what the server exports
showmount -e 10.0.1.10

# Mount
mkdir /mnt/data
mount 10.0.1.10:/shared/data /mnt/data

# Persistent mount (add to /etc/fstab)
echo "10.0.1.10:/shared/data  /mnt/data  nfs  defaults  0 0" >> /etc/fstab
```

> **DevOps Mapping:**
> - NFS ≈ Amazon EFS, Azure Files (NFS tier), GCP Filestore
> - Used for shared storage in Kubernetes (NFS-based Persistent Volumes)
> - Common for shared web content, shared logs, CI/CD artifact storage

---

<!-- END OF BATCH 3 — Sections 11-15 -->

## 16. Essential Services Stack — NTP, HTTP, Mail, Proxy, DHCP

### 16.1 Why Know the Services Stack?

As a DevOps/Cloud engineer, you don't just deploy YOUR application — you manage the **ecosystem** it depends on. Time sync, web servers, proxies, and DHCP are foundational services that, when broken, take everything else down with them.

### 16.2 Time Synchronization — NTP & chronyd

#### Why Time Sync is CRITICAL

```
  ┌──────────────────────────────────────────────────────────┐
  │  WHY TIME MATTERS — REAL FAILURES FROM TIME DRIFT:       │
  │                                                          │
  │  • Kerberos auth fails if clock skew > 5 minutes         │
  │  • TLS/SSL certificates rejected (not yet valid/expired) │
  │  • Log correlation impossible (which event came first?)  │
  │  • Distributed databases corrupt (Cassandra, CockroachDB)│
  │  • Cron jobs fire at wrong times                         │
  │  • MFA/TOTP codes rejected (time-based)                  │
  │  • AWS API calls rejected (signature timestamp mismatch) │
  │                                                          │
  │  In cloud: VMs can drift after suspend/resume, live      │
  │  migration, or snapshot restore.                         │
  └──────────────────────────────────────────────────────────┘
```

#### chronyd vs ntpd

```
  ┌────────────────────────┬──────────────────────────────────┐
  │       chronyd          │           ntpd                   │
  │  (Modern — RHEL 7+)    │     (Legacy)                     │
  ├────────────────────────┼──────────────────────────────────┤
  │  Faster sync           │  Slower convergence              │
  │  Handles intermittent  │  Needs constant network          │
  │  network (VMs, laptops)│                                  │
  │  Better for VMs        │  Better for dedicated NTP servers│
  │  Lower memory          │  Higher memory                   │
  │  Default on RHEL 7+    │  Default on RHEL 6 and older     │
  ├────────────────────────┴──────────────────────────────────┤
  │  DevOps verdict: Use chronyd. It's the modern default.    │
  └───────────────────────────────────────────────────────────┘
```

```bash
# Check current time sync status
timedatectl

# Check chrony status
chronyc tracking                  # Detailed sync info
chronyc sources -v                # NTP sources with details

# Force immediate sync
chronyc makestep

# Config file
cat /etc/chrony.conf
# Key lines:
# server 0.rhel.pool.ntp.org iburst
# server time.google.com iburst

# Set timezone
timedatectl set-timezone Asia/Kolkata
timedatectl list-timezones | grep Asia

# Enable NTP sync
timedatectl set-ntp true

# Service management
systemctl status chronyd
systemctl enable --now chronyd
```

> **DevOps Best Practice:** In cloud, use the provider's NTP server: AWS → `169.254.169.123`, GCP → `metadata.google.internal`, Azure → built-in Hyper-V time sync. These are the most accurate sources for your VMs.

### 16.3 Web Servers — Apache (httpd) & Nginx

These are the two workhorses of the web. Every DevOps engineer must understand both.

#### Apache vs Nginx — Architecture Comparison

```
  ┌──────────────────────────────────────────────────────────────┐
  │           APACHE (httpd)                                     │
  │                                                              │
  │  Process/Thread-based Model:                                 │
  │  ┌──────────┐                                                │
  │  │  Master  │                                                │
  │  │  Process │                                                │
  │  └────┬─────┘                                                │
  │       ├── Worker Process/Thread 1 ← handles 1 connection     │
  │       ├── Worker Process/Thread 2 ← handles 1 connection     │
  │       ├── Worker Process/Thread 3 ← handles 1 connection     │
  │       └── ...                                                │
  │                                                              │
  │  • Each connection = 1 thread/process                        │
  │  • 10,000 connections = 10,000 threads (C10K problem)        │
  │  • More memory per connection                                │
  │  • .htaccess per-directory config (flexible but slower)      │
  │  • mod_php runs PHP inside Apache (simple)                   │
  ├──────────────────────────────────────────────────────────────┤
  │           NGINX                                              │
  │                                                              │
  │  Event-driven, Async Model:                                  │
  │  ┌──────────┐                                                │
  │  │  Master  │                                                │
  │  │  Process │                                                │
  │  └────┬─────┘                                                │
  │       ├── Worker 1 ← handles THOUSANDS of connections        │
  │       ├── Worker 2 ← handles THOUSANDS of connections        │
  │       └── Worker N (typically = CPU cores)                   │
  │                                                              │
  │  • Event loop: one thread handles many connections           │
  │  • 10,000 connections = same few workers (no C10K problem)   │
  │  • Minimal memory per connection                             │
  │  • No .htaccess (all config centralized = faster)            │
  │  • Reverse proxy + load balancer built-in                    │
  └──────────────────────────────────────────────────────────────┘
```

| Feature | Apache | Nginx |
|---------|--------|-------|
| Architecture | Process/Thread per connection | Event-driven async |
| Static content | Good | **Excellent** |
| Dynamic content | mod_php (built-in) | Proxies to PHP-FPM, upstream |
| Reverse proxy | mod_proxy (add-on) | **Native, core feature** |
| Load balancing | mod_proxy_balancer | **Native, advanced** |
| Config style | .htaccess (per-dir) | Centralized (faster) |
| Memory usage | Higher | **Lower** |
| High concurrency | Struggles | **Excels** |
| **DevOps Use** | Legacy apps, WordPress | **Reverse proxy, LB, modern apps** |

#### Key Commands & Config

```bash
# ── APACHE ──
yum install httpd -y
systemctl enable --now httpd
# Config: /etc/httpd/conf/httpd.conf
# DocumentRoot: /var/www/html/
# Logs: /var/log/httpd/access_log, error_log
apachectl configtest                   # Validate config

# ── NGINX ──
yum install nginx -y
systemctl enable --now nginx
# Config: /etc/nginx/nginx.conf
# Sites: /etc/nginx/conf.d/*.conf
# DocumentRoot: /usr/share/nginx/html/
# Logs: /var/log/nginx/access.log, error.log
nginx -t                                # Validate config
nginx -s reload                         # Graceful reload
```

#### Nginx as Reverse Proxy (Most Common DevOps Use)

```
  ┌─────────────────────────────────────────────────────────────┐
  │         NGINX REVERSE PROXY PATTERN                         │
  │                                                             │
  │  Client ──► :443 ┌──────────────┐ ──► :3000 ┌────────────┐  │
  │  (HTTPS)         │  Nginx       │           │  Node.js   │  │
  │                  │  (TLS        │           │  App       │  │
  │                  │  termination,|           │  (no TLS,  │  │
  │                  │  load        │           │  internal) │  │
  │                  │  balancing)  │           └────────────┘  │
  │                  │              │ ──► :8080 ┌────────────┐  │
  │                  │              │           │  Python    │  │
  │                  └──────────────┘           │  App       │  │
  │                                             └────────────┘  │
  └─────────────────────────────────────────────────────────────┘
```

```nginx
# /etc/nginx/conf.d/app.conf
server {
    listen 80;
    server_name app.corp.com;
    
    # Redirect HTTP to HTTPS
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name app.corp.com;
    
    ssl_certificate /etc/ssl/certs/app.crt;
    ssl_certificate_key /etc/ssl/private/app.key;
    
    location / {
        proxy_pass http://localhost:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

> **DevOps Pattern:** Nginx sits in front of your application, handles TLS termination, serves static files, and load-balances across backend instances. Your app runs on localhost:3000 — no public exposure. This is the standard production pattern.

### 16.4 Mail Transfer Agent (MTA) — Overview

You rarely configure mail servers as a DevOps engineer, but you need to know the concepts for alert emails, relay configuration, and troubleshooting.

```
  ┌──────────────────────────────────────────────────────────┐
  │              EMAIL FLOW                                  │
  │                                                          │
  │  Sender                                   Recipient      │
  │  ┌──────┐    ┌──────┐    ┌──────┐    ┌──────┐            │
  │  │ MUA  │───►│ MTA  │───►│ MTA  │───►│ MDA  │──► 📬      │
  │  │(mail │    │(Send │    │(Recv │    │(Mail │  Mailbox   │
  │  │client)│   │server)│   │server)│   │Deliv.)│           │
  │  └──────┘    └──────┘    └──────┘    └──────┘            │
  │  Outlook     Postfix     Postfix     Dovecot             │
  │  Thunderbird sendmail    Exchange                        │
  │              SMTP:25     SMTP:25     IMAP:143/POP3:110   │
  │                                                          │
  │  Linux MTAs:                                             │
  │  • sendmail  ← legacy, complex config                    │
  │  • Postfix   ← modern, easier, secure ✅                 │
  │  • Exim      ← common on Debian                          │
  └──────────────────────────────────────────────────────────┘
```

> **DevOps Use:** You typically configure a local MTA to relay alert emails (from cron, monitoring, etc.) through a cloud email service (SES, SendGrid, Mailgun) — not run a full mail server.

### 16.5 Squid Proxy Server

```
  ┌──────────────────────────────────────────────────────────┐
  │           PROXY SERVER TYPES                             │
  │                                                          │
  │  Forward Proxy:                                          │
  │  Client ──► Proxy ──► Internet                           │
  │  • Client knows about the proxy                          │
  │  • Proxy hides the client from the server                │
  │  • Use: caching, filtering, access control               │
  │  • Example: Squid                                        │
  │                                                          │
  │  Reverse Proxy:                                          │
  │  Internet ──► Proxy ──► Backend Servers                  │
  │  • Client doesn't know about backend servers             │
  │  • Proxy hides servers from the client                   │
  │  • Use: load balancing, TLS termination, caching         │
  │  • Example: Nginx, HAProxy                               │
  │                                                          │
  │  Transparent Proxy:                                      │
  │  Client ──► [Proxy] ──► Internet                         │
  │  • Client doesn't know the proxy exists                  │
  │  • Network intercepts traffic automatically              │
  │  • Use: corporate filtering, ISP caching                 │
  └──────────────────────────────────────────────────────────┘
```

```bash
# Install Squid
yum install squid -y
systemctl enable --now squid

# Config: /etc/squid/squid.conf
# Default port: 3128
# Key directives:
#   acl localnet src 10.0.0.0/8
#   http_access allow localnet
#   http_port 3128
#   cache_dir ufs /var/spool/squid 100 16 256
```

> **DevOps Relevance:** Forward proxies are used in air-gapped environments (servers can't reach the internet directly) and for caching package downloads (yum/apt proxy saves bandwidth when updating 100+ servers).

### 16.6 DHCP Server

```
  ┌──────────────────────────────────────────────────────────┐
  │               DHCP — DORA Process                        │
  │                                                          │
  │  Client                            DHCP Server           │
  │  (no IP yet)                                             │
  │                                                          │
  │  ──── D: DISCOVER (broadcast) ────►                      │
  │  ◄─── O: OFFER (here's an IP) ─────                      │
  │  ──── R: REQUEST (I'll take it) ───►                     │
  │  ◄─── A: ACKNOWLEDGE (it's yours) ──                     │
  │                                                          │
  │  Config: /etc/dhcp/dhcpd.conf                            │
  │  Service: dhcpd                                          │
  │                                                          │
  │  Key concepts:                                           │
  │  • Scope/Range: Pool of IPs to assign                    │
  │  • Lease time: How long client keeps the IP              │
  │  • Reservation: Fixed IP for specific MAC                │
  │  • Options: Gateway, DNS, NTP servers                    │
  └──────────────────────────────────────────────────────────┘
```

> **DevOps Note:** In cloud, DHCP is handled automatically by the VPC/VNET. You don't run DHCP servers. But understanding DORA and leases helps debug on-prem networking and VPN issues.

---

## 17. Directory Services & Authentication — LDAP, IDM, WinBIND

### 17.1 Why Directory Services Matter

In any organization beyond a handful of servers, you need **centralized identity**. Without it:
- Users have different passwords on every server
- No single place to disable a departed employee's access
- No group-based access control
- Compliance nightmare (who has access to what?)

### 17.2 The Directory Services Landscape — Mental Model

```
  ┌────────────────────────────────────────────────────────────────┐
  │         DIRECTORY SERVICES ECOSYSTEM                           │
  │                                                                │
  │  ┌──────────────────────────────────────────────────────┐      │
  │  │                 LDAP (Protocol)                      │      │
  │  │  Lightweight Directory Access Protocol               │      │
  │  │  • The LANGUAGE all directory services speak         │      │
  │  │  • Like SQL is to databases, LDAP is to directories  │      │
  │  │  • Port 389 (plaintext) / 636 (LDAPS/TLS)            │      │
  │  └───────────────────────┬──────────────────────────────┘      │
  │                          │ implemented by                      │
  │           ┌──────────────┼──────────────────┐                  │
  │           ▼              ▼                  ▼                  │
  │   ┌───────────────┐ ┌───────────┐  ┌──────────────────────┐    │
  │   │  OpenLDAP     │ │  AD DS    │  │  FreeIPA / IdM       │    │
  │   │               │ │ (Windows) │  │  (Red Hat)           │    │
  │   │  • Open source│ │           │  │                      │    │
  │   │  • Just the   │ │  • LDAP + │  │  • LDAP + Kerberos + │    │
  │   │    directory  │ │  Kerberos+│  │    DNS + CA + SUDO   │    │
  │   │  • Manual     │ │  DNS +    │  │  • Full Linux IAM    │    │
  │   │    config     │ │  GPO      │  │  • Web UI            │    │
  │   │  • Lightweight│ │  • Full   │  │  • Best for pure     │    │
  │   │               │ │  Windows  │  │    Linux environments│    │
  │   │               │ │  ecosystem│  │                      │    │
  │   └───────────────┘ └───────────┘  └──────────────────────┘    │
  │                                                                │
  │  ┌──────────────────────────────────────────────────────┐      │
  │  │  WinBIND / SSSD                                      │      │
  │  │  • Bridges: joins Linux to Windows Active Directory  │      │
  │  │  • Linux servers authenticate against Windows AD     │      │
  │  │  • SSSD = modern replacement (caching, offline auth) │      │
  │  └──────────────────────────────────────────────────────┘      │
  └────────────────────────────────────────────────────────────────┘
```

### 17.3 LDAP — The Protocol

```
  LDAP Data Structure (DIT — Directory Information Tree):

  dc=corp,dc=com                            ← Domain Component (root)
  ├── ou=People                             ← Organizational Unit
  │   ├── uid=vishvam,ou=People,dc=corp,dc=com    ← User entry (DN)
  │   └── uid=john,ou=People,dc=corp,dc=com
  ├── ou=Groups
  │   ├── cn=developers,ou=Groups,dc=corp,dc=com  ← Group entry
  │   └── cn=devops,ou=Groups,dc=corp,dc=com
  └── ou=Services
      └── cn=jenkins,ou=Services,dc=corp,dc=com

  Key Terms:
  DN  = Distinguished Name (full path to an entry — like absolute file path)
  RDN = Relative Distinguished Name (just the entry name — like filename)
  dc  = Domain Component
  ou  = Organizational Unit
  cn  = Common Name
  uid = User ID
```

### 17.4 Comparison Matrix

| Feature | OpenLDAP | Active Directory | FreeIPA/IdM |
|---------|----------|-----------------|-------------|
| Protocol | LDAP | LDAP + Kerberos | LDAP + Kerberos |
| OS Focus | Linux | Windows | Linux (RHEL) |
| DNS | No (separate) | Integrated | Integrated |
| Kerberos | No (separate) | Integrated | Integrated |
| Certificate Authority | No | AD CS (separate) | Integrated (Dogtag) |
| Sudo rules | No | No (GPO equivalent) | Yes, centralized |
| Web UI | No (3rd party) | ADUC/ADAC | Yes |
| Complexity | High (manual) | Medium | Low (all-in-one) |
| **Cloud Mapping** | — | Azure AD / Entra ID | — |

### 17.5 Joining Linux to Active Directory

In mixed environments (common in enterprises), Linux servers authenticate against Windows AD.

```
  ┌──────────────────────────────────────────────────────────┐
  │     LINUX + ACTIVE DIRECTORY INTEGRATION                 │
  │                                                          │
  │  ┌────────────┐                 ┌──────────────────┐     │
  │  │ Linux      │  SSSD/WinBIND   │ Windows AD DC    │     │
  │  │ Server     │◄───────────────►│                  │     │
  │  │            │  Kerberos/LDAP  │  Users, Groups,  │     │
  │  │ realm join │                 │  Policies        │     │
  │  └────────────┘                 └──────────────────┘     │
  │                                                          │
  │  Tools:                                                  │
  │  • realmd      ← Discovery & join (simplest)             │
  │  • sssd        ← Auth daemon (caching, offline)          │
  │  • winbind     ← Samba-based (legacy, still works)       │
  │  • adcli       ← CLI for AD operations                   │
  └──────────────────────────────────────────────────────────┘
```

```bash
# Join Linux to AD (using realmd — the easy way)
yum install realmd sssd oddjob samba-common-tools

# Discover the domain
realm discover corp.com

# Join the domain
realm join corp.com -U admin

# Verify
realm list
id admin@corp.com

# Allow specific AD groups to log in
realm permit -g 'Linux Admins'

# SSH as AD user
ssh vishvam@corp.com@linux-server
```

> **DevOps Relevance:** Enterprise environments almost always have AD. Your Linux servers, CI/CD agents, and even Kubernetes nodes may need to authenticate against AD. SSSD with realmd is the modern, clean approach.

---

## 18. Security, Firewall & OS Hardening

### 18.1 Why Security is a DevOps Responsibility

DevSecOps = Security is NOT an afterthought. It's built into every pipeline, every server, every config. A single misconfigured firewall rule or open port can lead to a breach.

### 18.2 firewalld — The Dynamic Firewall

firewalld is the default firewall management tool on RHEL/CentOS 7+. It wraps around `iptables`/`nftables` and adds the concept of **zones**.

#### firewalld Architecture — Mental Model

```
  ┌───────────────────────────────────────────────────────────┐
  │                   FIREWALLD ZONES                         │
  │                                                           │
  │  Each network interface is assigned to a ZONE.            │
  │  Each zone has its own set of allowed services/ports.     │
  │                                                           │
  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
  │  │   public     │  │   internal   │  │    dmz       │     │
  │  │  (default)   │  │              │  │              │     │
  │  │  • ssh       │  │  • ssh       │  │  • ssh       │     │
  │  │  • dhcpv6    │  │  • all from  │  │  • http      │     │
  │  │              │  │    internal  │  │  • https     │     │
  │  │  eth0 ←      │  │  eth1 ←      │  │  eth2 ←      │     │
  │  └──────────────┘  └──────────────┘  └──────────────┘     │
  │                                                           │
  │  Pre-defined zones (least → most permissive):             │
  │  drop → block → public → external → dmz →                 │
  │  work → home → internal → trusted                         │
  └───────────────────────────────────────────────────────────┘
```

#### firewall-cmd — Essential Commands

```bash
# Status
firewall-cmd --state
systemctl status firewalld

# View current zone config
firewall-cmd --get-active-zones
firewall-cmd --get-default-zone
firewall-cmd --list-all                        # All rules for default zone
firewall-cmd --zone=public --list-all          # Specific zone

# Allow a service (permanent + reload)
firewall-cmd --permanent --add-service=http
firewall-cmd --permanent --add-service=https
firewall-cmd --reload                          # Apply permanent changes

# Allow a specific port
firewall-cmd --permanent --add-port=8080/tcp
firewall-cmd --permanent --add-port=3000-3010/tcp    # Port range
firewall-cmd --reload

# Remove a rule
firewall-cmd --permanent --remove-service=http
firewall-cmd --permanent --remove-port=8080/tcp
firewall-cmd --reload

# List available services
firewall-cmd --get-services

# Change default zone
firewall-cmd --set-default-zone=internal

# Add interface to zone
firewall-cmd --zone=internal --change-interface=eth1

# Rich rules (advanced — IP-based filtering)
firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="10.0.1.0/24" service name="ssh" accept'
firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.168.1.100" port port="3306" protocol="tcp" accept'
firewall-cmd --reload
```

> **DevOps Pattern:** Default deny, explicit allow. Only open the ports your application needs. Use rich rules to restrict access by source IP (e.g., only allow DB access from app server IPs).

### 18.3 iptables — The Low-Level Firewall

iptables is the underlying packet filtering framework. firewalld is a frontend to it. Know iptables for debugging and legacy systems.

```
  ┌──────────────────────────────────────────────────────────┐
  │              IPTABLES CHAIN FLOW                         │
  │                                                          │
  │  Incoming Packet                                         │
  │       │                                                  │
  │       ▼                                                  │
  │  ┌───────────┐   Destination    ┌──────────┐             │
  │  │ PREROUTING├──► is this ────► │  INPUT   │──► Local    │
  │  │           │   host?    NO    │          │   Process   │
  │  └───────────┘     │            └──────────┘             │
  │                    │                                     │
  │                    ▼ YES (forwarding)                    │
  │              ┌──────────┐                                │
  │              │ FORWARD  │──► Out another interface       │
  │              └──────────┘                                │
  │                                                          │
  │  Local Process ──► ┌──────────┐ ──► ┌────────────┐       │
  │  (outgoing)        │ OUTPUT   │     │POSTROUTING │──► 🌐 │
  │                    └──────────┘     └────────────┘       │
  │                                                          │
  │  Tables: filter (default), nat, mangle, raw              │
  │  Targets: ACCEPT, DROP, REJECT, LOG, MASQUERADE          │
  └──────────────────────────────────────────────────────────┘
```

```bash
# View current rules
iptables -L -n -v                    # List rules (numeric, verbose)
iptables -L -n --line-numbers        # With line numbers (for deletion)

# Allow incoming SSH
iptables -A INPUT -p tcp --dport 22 -j ACCEPT

# Allow HTTP and HTTPS
iptables -A INPUT -p tcp --dport 80 -j ACCEPT
iptables -A INPUT -p tcp --dport 443 -j ACCEPT

# Allow established connections (IMPORTANT — don't break existing sessions)
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# Drop everything else
iptables -A INPUT -j DROP

# Allow from specific IP
iptables -A INPUT -s 10.0.1.50 -p tcp --dport 3306 -j ACCEPT

# Delete a rule by line number
iptables -D INPUT 3

# Save rules (RHEL/CentOS)
iptables-save > /etc/sysconfig/iptables
# Or: service iptables save

# NAT / Port forwarding
iptables -t nat -A PREROUTING -p tcp --dport 80 -j REDIRECT --to-port 8080
```

> **firewalld vs iptables:** Use `firewalld` for day-to-day management (zones, services = human-friendly). Use `iptables` when you need raw packet-level control or are on a system without firewalld. **Never run both simultaneously.**

### 18.4 SELinux — Security Enhanced Linux

SELinux adds **mandatory access control (MAC)** on top of standard file permissions. Even if a process has file permissions, SELinux can block access based on security context.

```
  ┌───────────────────────────────────────────────────────────┐
  │  Standard Permissions: "Can user X read file Y?"          │
  │  SELinux:              "Can process of TYPE A access      │
  │                         file of TYPE B?"                  │
  │                                                           │
  │  Modes:                                                   │
  │  • Enforcing  — blocks and logs violations ✅ (production)│
  │  • Permissive — logs but doesn't block (debugging)        │
  │  • Disabled   — off entirely (not recommended)            │
  └───────────────────────────────────────────────────────────┘
```

```bash
# Check current mode
getenforce                           # Returns: Enforcing/Permissive/Disabled
sestatus                             # Detailed status

# Temporarily change mode (until reboot)
setenforce 0                         # Set to Permissive (debugging)
setenforce 1                         # Set to Enforcing

# Permanently change: edit /etc/selinux/config
# SELINUX=enforcing

# View SELinux context on files
ls -Z /var/www/html/
# -rw-r--r--. root root unconfined_u:object_r:httpd_sys_content_t:s0 index.html
#                                     ^^^^^^^^^^^^^^^^^^^^^^^^
#                                     SELinux type label

# Fix SELinux context (common fix for "permission denied" with correct unix perms)
restorecon -Rv /var/www/html/

# Allow a boolean (e.g., let httpd connect to network)
getsebool -a | grep httpd                     # List httpd booleans
setsebool -P httpd_can_network_connect on     # -P = persistent

# Check for SELinux denials
ausearch -m avc -ts recent                    # Recent denials
sealert -a /var/log/audit/audit.log           # Human-readable analysis
```

> **DevOps Rule:** Never disable SELinux. If something is blocked, fix the context (`restorecon`) or enable the right boolean (`setsebool`). Disabling SELinux = compliance failure and security risk.

### 18.5 Linux OS Hardening — Checklist

```
  ┌────────────────────────────────────────────────────────────┐
  │         OS HARDENING CHECKLIST FOR DEVOPS                  │
  │                                                            │
  │  Authentication & Access:                                  │
  │  ☐ Disable root SSH login (PermitRootLogin no)             │
  │  ☐ Use SSH key auth only (PasswordAuthentication no)       │
  │  ☐ Configure sudo (don't share root password)              │
  │  ☐ Set password complexity and aging (pam, chage)          │
  │  ☐ Remove/lock unnecessary user accounts                   │
  │  ☐ Set idle session timeout (TMOUT=600 in profile)         │
  │                                                            │
  │  Network:                                                  │
  │  ☐ Enable firewall (firewalld) — default deny              │
  │  ☐ Open only required ports                                │
  │  ☐ Disable unused services (systemctl disable)             │
  │  ☐ Disable IPv6 if not needed                              │
  │  ☐ Change SSH port from 22 (security through obscurity)    │
  │                                                            │
  │  Filesystem:                                               │
  │  ☐ Set nosuid,noexec on /tmp mount                         │
  │  ☐ Enable SELinux (Enforcing mode)                         │
  │  ☐ Set proper file permissions (600 for keys, 644 config)  │
  │  ☐ Find and audit SUID/SGID files                          │
  │  ☐ Configure log rotation                                  │
  │                                                            │
  │  Updates:                                                  │
  │  ☐ Enable automatic security updates (yum-cron)            │
  │  ☐ Regular vulnerability scanning                          │
  │                                                            │
  │  Monitoring:                                               │
  │  ☐ Forward logs to central syslog/SIEM                     │
  │  ☐ Enable auditd for file access tracking                  │
  │  ☐ Monitor failed logins (lastb, /var/log/secure)          │
  │  ☐ Set up intrusion detection (AIDE, OSSEC)                │
  └────────────────────────────────────────────────────────────┘
```

### 18.6 tuned — Performance Tuning Profiles

tuned automatically adjusts system settings based on workload profile.

```bash
# Install and enable
yum install tuned -y
systemctl enable --now tuned

# List available profiles
tuned-adm list

# Current profile
tuned-adm active

# Set a profile
tuned-adm profile throughput-performance     # Servers
tuned-adm profile virtual-guest              # VMs
tuned-adm profile latency-performance        # Low-latency apps

# Recommend a profile
tuned-adm recommend
```

| Profile | Use Case |
|---------|----------|
| `throughput-performance` | Maximum throughput (web servers, databases) |
| `latency-performance` | Minimum latency (real-time apps) |
| `virtual-guest` | **VMs in cloud** (most common for DevOps) |
| `virtual-host` | Hypervisor hosts |
| `balanced` | Default — balance between power and performance |
| `powersave` | Laptops, power-saving |

> **DevOps Tip:** Always set `virtual-guest` on cloud VMs. It optimizes for virtualized workloads — scheduler settings, disk I/O elevator, etc.

---

## 19. Containerization & Configuration Management — Docker, Ansible

### 19.1 Why These Are the DevOps Power Tools

Docker and Ansible are arguably the two most transformative tools in the DevOps ecosystem. Docker changed HOW we package and run software. Ansible changed HOW we configure infrastructure.

### 19.2 Docker & Podman — Container Runtime

#### Mental Model — Containers vs VMs

```
  ┌──────────────────────────────────────────────────────────────┐
  │  Virtual Machines                    Containers              │
  │  ┌──────┐ ┌──────┐ ┌──────┐        ┌──────┐ ┌──────┐         │
  │  │ App1 │ │ App2 │ │ App3 │        │ App1 │ │ App2 │         │
  │  ├──────┤ ├──────┤ ├──────┤        ├──────┤ ├──────┤         │
  │  │ Bins │ │ Bins │ │ Bins │        │ Bins │ │ Bins │         │
  │  │ Libs │ │ Libs │ │ Libs │        │ Libs │ │ Libs │         │
  │  ├──────┤ ├──────┤ ├──────┤        └──┬───┘ └──┬───┘         │
  │  │Guest │ │Guest │ │Guest │           │        │             │
  │  │  OS  │ │  OS  │ │  OS  │        ┌──┴────────┴───┐         │
  │  ├──────┤ ├──────┤ ├──────┤        │ Container     │         │
  │  │      Hypervisor        │        │ Runtime       │         │
  │  ├────────────────────────┤        │(Docker/Podman)│         │
  │  │      Host OS           │        ├───────────────┤         │
  │  ├────────────────────────┤        │   Host OS     │         │
  │  │     Hardware           │        ├───────────────┤         │
  │  └────────────────────────┘        │  Hardware     │         │
  │                                    └───────────────┘         │
  │  • Each VM = full OS (GB)          • Share host kernel       │
  │  • Minutes to boot                 • Seconds to start        │
  │  • Strong isolation                • Process isolation       │
  │  • Heavy resource usage            • Lightweight             │
  └──────────────────────────────────────────────────────────────┘
```

#### Docker vs Podman

| Feature | Docker | Podman |
|---------|--------|--------|
| Daemon | dockerd (runs as root) | **Daemonless** (no root daemon) |
| Root required | Yes (default) | **No — rootless by default** |
| CLI | `docker` | `podman` (drop-in compatible) |
| Security | Root daemon = attack surface | Rootless = more secure |
| Compose | docker-compose | podman-compose |
| Default on | Ubuntu, most distros | **RHEL 8+, CentOS Stream** |
| Kubernetes | Docker → containerd | Podman plays well with K8s |

> **Alias trick:** `alias docker=podman` — Podman is CLI-compatible with Docker. Same commands, more secure.

#### Essential Container Commands

```bash
# Image management
docker pull nginx:latest              # Download image
docker images                         # List local images
docker rmi nginx:latest               # Remove image

# Run containers
docker run nginx                      # Run in foreground
docker run -d nginx                   # Run detached (background)
docker run -d -p 80:80 nginx          # Map host:container port
docker run -d -p 80:80 --name web nginx  # Named container
docker run -d -v /host/data:/data nginx  # Mount volume
docker run -d --restart unless-stopped nginx  # Auto-restart

# Container lifecycle
docker ps                             # Running containers
docker ps -a                          # All containers (including stopped)
docker stop web                       # Graceful stop
docker start web                      # Start stopped container
docker restart web                    # Restart
docker rm web                         # Remove stopped container
docker rm -f web                      # Force remove running container

# Inspect & debug
docker logs web                       # View logs
docker logs -f web                    # Follow logs (tail -f)
docker exec -it web /bin/bash         # Interactive shell inside container
docker inspect web                    # Full JSON details
docker stats                          # Live resource usage

# Build images
docker build -t myapp:v1 .            # Build from Dockerfile in current dir
docker build -t myapp:v1 -f Dockerfile.prod .  # Specific Dockerfile
```

#### Creating a systemd Service for a Container

This maps to the practical of creating a custom service for a Node.js process:

```ini
# /etc/systemd/system/myapp.service
[Unit]
Description=My Node.js Application Container
After=network.target

[Service]
Type=simple
Restart=always
RestartSec=5
ExecStartPre=-/usr/bin/docker rm -f myapp
ExecStart=/usr/bin/docker run --rm --name myapp -p 3000:3000 myapp:latest
ExecStop=/usr/bin/docker stop myapp

[Install]
WantedBy=multi-user.target
```

```bash
systemctl daemon-reload
systemctl enable --now myapp.service
# Now: container auto-starts on boot, restarts on crash
```

> **This is the pattern:** application in container → managed by systemd → fronted by nginx reverse proxy → automatic restart on failure. This is production-grade deployment without Kubernetes.

### 19.3 Ansible — Configuration Management

#### Why Ansible Over Others?

```
  ┌──────────────────────────────────────────────────────────┐
  │        CONFIGURATION MANAGEMENT TOOLS                    │
  │                                                          │
  │  Ansible           Chef              Puppet              │
  │  ┌──────────┐      ┌──────────┐      ┌──────────┐        │
  │  │Agentless │      │ Agent    │      │ Agent    │        │
  │  │SSH-based │      │ Required │      │ Required │        │
  │  │YAML      │      │ Ruby DSL │      │ Ruby DSL │        │
  │  │Push model│      │Pull model│      │Pull model│        │
  │  │Simple ✅ │      │ Complex  │      │ Complex  │        │
  │  └──────────┘      └──────────┘      └──────────┘        │
  │                                                          │
  │  Ansible wins for DevOps because:                        │
  │  • No agent to install on target servers                 │
  │  • Uses SSH (already set up)                             │
  │  • YAML = human-readable, low learning curve             │
  │  • Idempotent: run it 100 times, same result             │
  │  • Massive module library (cloud, containers, network)   │
  │  • Ad-hoc commands for one-off tasks                     │
  └──────────────────────────────────────────────────────────┘
```

#### Ansible Architecture — Mental Model

```
  ┌───────────────────────────────────────────────────────────┐
  │                ANSIBLE ARCHITECTURE                       │
  │                                                           │
  │  Control Node (your laptop/CI server)                     │
  │  ┌──────────────────────────────────────────────────┐     │
  │  │  ansible.cfg      ← Configuration                │     │
  │  │  inventory        ← List of managed hosts        │     │
  │  │  playbook.yml     ← Tasks to execute             │     │
  │  │  roles/           ← Reusable task collections    │     │
  │  └────────────────────────┬─────────────────────────┘     │
  │                           │ SSH                           │
  │            ┌──────────────┼──────────────┐                │
  │            ▼              ▼              ▼                │
  │  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐       │
  │  │  Managed     │ │  Managed     │ │  Managed     │       │
  │  │  Node 1      │ │  Node 2      │ │  Node 3      │       │
  │  │  (web-01)    │ │  (web-02)    │ │  (db-01)     │       │
  │  │              │ │              │ │              │       │
  │  │  No agent!   │ │  No agent!   │ │  No agent!   │       │
  │  │  Just SSH +  │ │  Just SSH +  │ │  Just SSH +  │       │
  │  │  Python      │ │  Python      │ │  Python      │       │
  │  └──────────────┘ └──────────────┘ └──────────────┘       │
  └───────────────────────────────────────────────────────────┘
```

#### Quick Start Example

```yaml
# inventory.ini
[webservers]
web-01 ansible_host=10.0.1.50
web-02 ansible_host=10.0.1.51

[databases]
db-01 ansible_host=10.0.1.60

[all:vars]
ansible_user=deploy
ansible_ssh_private_key_file=~/.ssh/id_ed25519
```

```yaml
# playbook.yml — Deploy web application
---
- name: Configure web servers
  hosts: webservers
  become: yes                        # Use sudo

  tasks:
    - name: Install nginx
      yum:
        name: nginx
        state: present

    - name: Start and enable nginx
      systemd:
        name: nginx
        state: started
        enabled: yes

    - name: Deploy application config
      template:
        src: templates/nginx.conf.j2
        dest: /etc/nginx/conf.d/app.conf
      notify: Reload nginx            # Trigger handler only if changed

    - name: Open firewall for HTTP
      firewalld:
        service: http
        permanent: yes
        state: enabled
        immediate: yes

  handlers:
    - name: Reload nginx
      systemd:
        name: nginx
        state: reloaded
```

```bash
# Run ad-hoc command on all servers
ansible all -m ping                                 # Test connectivity
ansible webservers -m shell -a "uptime"             # Run command
ansible webservers -m yum -a "name=nginx state=latest" -b  # Install package

# Run playbook
ansible-playbook -i inventory.ini playbook.yml

# Dry run (check mode)
ansible-playbook -i inventory.ini playbook.yml --check

# Run with verbose output
ansible-playbook -i inventory.ini playbook.yml -vvv
```

> **DevOps Workflow:** Ansible is your "infrastructure autopilot." Write playbooks → store in Git → run via CI/CD pipeline → infrastructure changes are versioned, reviewed, and repeatable. This is **Infrastructure as Code**.

---

## 20. Monitoring & High Availability — Nagios, Cockpit, HA Clusters

### 20.1 Why Monitoring & HA Matter

```
  "If you can't measure it, you can't improve it."
  "If it's not monitored, it's not in production."
  "Single points of failure are production incidents waiting to happen."
```

### 20.2 Nagios — Resource Monitoring

Nagios is one of the oldest and most widely deployed monitoring systems. While modern alternatives exist (Prometheus, Datadog, Zabbix), Nagios concepts are foundational.

#### Nagios Architecture

```
  ┌───────────────────────────────────────────────────────────┐
  │                  NAGIOS ARCHITECTURE                      │
  │                                                           │
  │  ┌──────────────────────────────────────────────────┐     │
  │  │           Nagios Server (Core)                   │     │
  │  │                                                  │     │
  │  │  • Scheduling engine (runs checks on schedule)   │     │
  │  │  • Notification engine (email, SMS, PagerDuty)   │     │
  │  │  • Web UI (status dashboard)                     │     │
  │  │  • Event handler (auto-remediation)              │     │
  │  └──────────────────┬───────────────────────────────┘     │
  │                     │                                     │
  │        ┌────────────┼────────────┐                        │
  │        ▼            ▼            ▼                        │
  │  ┌──────────┐ ┌──────────┐ ┌──────────┐                   │
  │  │ Host 1   │ │ Host 2   │ │ Host 3   │                   │
  │  │ NRPE     │ │ NRPE     │ │ NRPE     │                   │
  │  │ agent    │ │ agent    │ │ agent    │                   │
  │  │          │ │          │ │          │                   │
  │  │ Checks:  │ │ Checks:  │ │ Checks:  │                   │
  │  │ •CPU     │ │ •Disk    │ │ •MySQL   │                   │
  │  │ •Memory  │ │ •HTTP    │ │ •Repl.   │                   │
  │  │ •Disk    │ │ •Load    │ │ •Backup  │                   │
  │  └──────────┘ └──────────┘ └──────────┘                   │
  │                                                           │
  │  Check Types:                                             │
  │  • Active: Nagios initiates (polls the host)              │
  │  • Passive: Host sends results to Nagios (push)           │
  │                                                           │
  │  States: OK (0) → WARNING (1) → CRITICAL (2) → UNKNOWN (3)│
  └───────────────────────────────────────────────────────────┘
```

#### Nagios vs Modern Monitoring — Comparison

| Feature | Nagios | Prometheus + Grafana | Datadog/New Relic |
|---------|--------|---------------------|-------------------|
| Type | Pull-based (active checks) | Pull-based (scraping) | Agent-based (push) |
| Storage | Flat files | Time-series DB (TSDB) | Cloud (SaaS) |
| Query Language | None | **PromQL** | Proprietary |
| Visualization | Basic web UI | **Grafana (excellent)** | Built-in dashboards |
| Alerting | Email, SMS, plugins | **AlertManager** | Built-in |
| Container Support | Limited | **Native (K8s SD)** | Native |
| Cost | Free + enterprise | Free | $$$$ |
| **DevOps Verdict** | Legacy, but concepts matter | **Industry standard** | Enterprise/SaaS |

> **Key Takeaway:** Learn Nagios concepts (host checks, service checks, thresholds, notification chains) — they apply to EVERY monitoring tool. Then use Prometheus + Grafana in practice.

### 20.3 Cockpit — Web-Based Linux Administration

Cockpit provides a **web UI for managing Linux servers** directly from your browser. Think of it as a lightweight web console.

```bash
# Install
yum install cockpit -y
systemctl enable --now cockpit.socket

# Access
# https://your-server-ip:9090
# Login with any system user (requires sudo for admin tasks)
```

```
  ┌──────────────────────────────────────────────────────────┐
  │             COCKPIT DASHBOARD                            │
  │  https://server:9090                                     │
  │                                                          │
  │  ┌─────────────────────────────────────────────────────┐ │
  │  │  System     │ CPU, Memory, Disk, Network graphs     │ │
  │  │  Logs       │ journalctl viewer with filters        │ │
  │  │  Storage    │ LVM, RAID, NFS — GUI management       │ │
  │  │  Networking │ Interfaces, firewall, bonds           │ │
  │  │  Accounts   │ User management                       │ │
  │  │  Services   │ systemd unit management               │ │
  │  │  Terminal   │ In-browser terminal!                  │ │
  │  │  Updates    │ Apply system updates                  │ │
  │  │  SELinux    │ View/resolve SELinux alerts           │ │
  │  └─────────────────────────────────────────────────────┘ │
  └──────────────────────────────────────────────────────────┘
```

> **DevOps Use:** Cockpit is great for **quick visual inspection** when you need a dashboard without setting up Grafana. Perfect for home labs, small deployments, and training. Not a replacement for proper monitoring in production.

### 20.4 High Availability (HA) Clusters

#### Why HA?

Single server = Single Point of Failure (SPOF). If it goes down, your service goes down. HA eliminates SPOFs by running multiple copies of critical services.

#### HA Architecture — Mental Model

```
  ┌────────────────────────────────────────────────────────────────┐
  │            HIGH AVAILABILITY PATTERNS                          │
  │                                                                │
  │  Pattern 1: Active-Passive (Failover)                          │
  │  ┌───────────────┐        ┌───────────────┐                    │
  │  │  Node A       │        │  Node B       │                    │
  │  │  ACTIVE ✅    │        │  PASSIVE 💤   │                    │
  │  │  (handles all │        │  (standby,    │                    │
  │  │   traffic)    │        │   takes over  │                    │
  │  └───────┬───────┘        │   on failure) │                    │
  │          │                └───────┬───────┘                    │
  │          └───── Heartbeat ────────┘                            │
  │                (monitors health)                               │
  │                                                                │
  │  Pattern 2: Active-Active (Load Balanced)                      │
  │  ┌───────────────┐        ┌───────────────┐                    │
  │  │  Node A       │        │  Node B       │                    │
  │  │  ACTIVE ✅    │        │  ACTIVE ✅    │                    │
  │  │  (handles     │        │  (handles     │                    │
  │  │   50% traffic)│        │   50% traffic)│                    │
  │  └───────┬───────┘        └───────┬───────┘                    │
  │          └──── Load Balancer ─────┘                            │
  │                                                                │
  │  Pattern 3: N+1 Redundancy                                     │
  │  3 nodes running, 1 extra for failover capacity                │
  │  Any single node can fail without service impact               │
  └────────────────────────────────────────────────────────────────┘
```

#### Linux HA Tools

```
  ┌──────────────────────────────────────────────────────────┐
  │  Pacemaker + Corosync — Linux HA Stack                   │
  │                                                          │
  │  Corosync:  Cluster communication layer (heartbeat)      │
  │             "Are all nodes alive?"                       │
  │                                                          │
  │  Pacemaker: Cluster resource manager                     │
  │             "Which node runs which service?"             │
  │             "What to do when a node fails?"              │
  │                                                          │
  │  Resources: Services managed by the cluster              │
  │  • Virtual IP (VIP): floating IP that moves to active    │
  │  • Services: httpd, database, application                │
  │  • Filesystems: shared storage mounts                    │
  │                                                          │
  │  ┌──────────┐                    ┌──────────┐            │
  │  │  Node A  │◄─── Corosync ────► │ Node B   │            │
  │  │          │   (heartbeat)      │          │            │
  │  │ Pacemaker│                    │ Pacemaker│            │
  │  │ httpd ✅ │                    │ httpd 💤 │            │
  │  │ VIP: ✅  │                    │ VIP: 💤  │            │
  │  └──────────┘                    └──────────┘            │
  │       │                                                  │
  │       │  Node A fails? Pacemaker:                        │
  │       │  1. Detects failure via Corosync                 │
  │       │  2. Moves VIP to Node B                          │
  │       │  3. Starts httpd on Node B                       │
  │       │  4. Clients reconnect to same IP (VIP)           │
  └──────────────────────────────────────────────────────────┘
```

```bash
# Install HA packages
yum install pacemaker corosync pcs -y

# Enable cluster services
systemctl enable --now pcsd

# Set password for hacluster user
echo "StrongPassword" | passwd --stdin hacluster

# Authenticate nodes
pcs host auth node1 node2

# Create cluster
pcs cluster setup mycluster node1 node2

# Start cluster
pcs cluster start --all
pcs cluster enable --all

# Check cluster status
pcs status

# Add a Virtual IP resource
pcs resource create VIP ocf:heartbeat:IPaddr2 \
    ip=10.0.1.100 cidr_netmask=24 op monitor interval=30s

# Add a service resource
pcs resource create WebServer systemd:httpd \
    op monitor interval=30s

# Ensure VIP and WebServer run on the same node
pcs constraint colocation add WebServer with VIP INFINITY

# Ensure VIP starts before WebServer
pcs constraint order VIP then WebServer
```

> **DevOps/Cloud Mapping:**
> - Pacemaker/Corosync ≈ AWS Auto Scaling Groups + ELB
> - Virtual IP (VIP) ≈ Elastic IP / Azure Load Balancer frontend IP
> - Active-Active ≈ Multiple instances behind a load balancer
> - In cloud, HA is typically achieved through managed services (RDS Multi-AZ, ELB, ASG) rather than Pacemaker — but the CONCEPTS are identical

### 20.5 OpenVPN — VPN for Secure Access

```
  ┌──────────────────────────────────────────────────────────┐
  │                  VPN ARCHITECTURE                        │
  │                                                          │
  │  Remote User                     Corporate Network       │
  │  ┌──────────┐                    ┌──────────────────┐    │
  │  │ Laptop   │══ encrypted ══════►│ OpenVPN Server   │    │
  │  │ OpenVPN  │   tunnel over      │ 10.8.0.1         │    │
  │  │ Client   │   public internet  │                  │    │
  │  │ 10.8.0.6 │                    │ Routes traffic   │    │
  │  └──────────┘                    │ to internal net  │    │
  │                                  └────────┬─────────┘    │
  │                                           │              │
  │                                  ┌────────▼─────────┐    │
  │                                  │ Internal Servers │    │
  │                                  │ 10.0.1.0/24      │    │
  │                                  └──────────────────┘    │
  │                                                          │
  │  Types:                                                  │
  │  • Site-to-Site: connect two networks                    │
  │  • Client-to-Site: remote users access internal network  │
  │                                                          │
  │  Cloud equivalent:                                       │
  │  • AWS VPN / AWS Client VPN                              │
  │  • Azure VPN Gateway                                     │
  │  • WireGuard (modern, faster alternative to OpenVPN)     │
  └──────────────────────────────────────────────────────────┘
```

> **DevOps Use:** VPN is how you securely access internal infrastructure (databases, admin panels, monitoring) from outside the corporate network. In cloud, you'd use VPC peering, PrivateLink, or a VPN gateway instead of running OpenVPN manually.

---

<!-- END OF BATCH 4 — Sections 16-20 -->

## 21. X vs Y — Head-to-Head Comparisons

> These comparisons are the backbone of trainee reviews and interviews. Know the "when to use which" reasoning, not just the feature list.

### 21.1 systemd vs SysVinit

```
  ┌──────────────────────────┬──────────────────────────────────┐
  │       SysVinit (Legacy)  │       systemd (Modern)           │
  ├──────────────────────────┼──────────────────────────────────┤
  │  Sequential boot         │  Parallel boot (faster)          │
  │  Shell scripts in        │  Unit files (.service, .timer)   │
  │  /etc/init.d/            │  /usr/lib/systemd/system/        │
  │  Runlevels (0-6)         │  Targets (multi-user, graphical) │
  │  service httpd start     │  systemctl start httpd           │
  │  chkconfig httpd on      │  systemctl enable httpd          │
  │  No dependency tracking  │  Auto dependency resolution      │
  │  No process supervision  │  Restarts crashed services       │
  │  /var/log/messages       │  journalctl (structured logs)    │
  │  No cgroup integration   │  cgroup-based resource control   │
  │  No socket activation    │  Socket activation (on-demand)   │
  ├──────────────────────────┴──────────────────────────────────┤
  │  Verdict: systemd is the standard since RHEL 7 / Ubuntu     │
  │  15.04. SysVinit knowledge is needed only for legacy        │
  │  systems and understanding boot concepts.                   │
  └─────────────────────────────────────────────────────────────┘
```

### 21.2 firewalld vs iptables

```
  ┌───────────────────────────┬──────────────────────────────────┐
  │       iptables (Low-level)│      firewalld (High-level)      │
  ├───────────────────────────┼──────────────────────────────────┤
  │  Raw packet rules         │  Zone-based abstractions         │
  │  Chain: INPUT/OUTPUT/FWD  │  Zones: public/internal/dmz      │
  │  Must specify protocol,   │  Add by service name:            │
  │  port, action manually    │  --add-service=http              │
  │  Static (flush + reload)  │  Dynamic (reload without drop)   │
  │  No runtime/permanent     │  --permanent + --reload          │
  │  separation               │  separation                      │
  │  Direct kernel interface  │  Frontend to iptables/nftables   │
  │  More powerful for NAT,   │  Rich rules for advanced cases   │
  │  raw packet manipulation  │                                  │
  │  Legacy (RHEL 6 and older)│  Default (RHEL 7+)               │
  ├───────────────────────────┴──────────────────────────────────┤
  │  Verdict: Use firewalld for daily management. Use iptables   │
  │  only for raw packet-level control or legacy systems.        │
  │  ⚠️ Never run both simultaneously.                           │
  └──────────────────────────────────────────────────────────────┘
```

### 21.3 yum vs dnf

```
  ┌───────────────────────────┬──────────────────────────────────┐
  │        yum (Legacy)       │        dnf (Modern)              │
  ├───────────────────────────┼──────────────────────────────────┤
  │  Default: RHEL 5-7        │  Default: RHEL 8+, Fedora        │
  │  Python 2 based           │  Python 3 based                  │
  │  Slower dependency solver │  libsolv (much faster)           │
  │  Higher memory usage      │  Lower memory usage              │
  │  yum install / remove     │  dnf install / remove (same CLI) │
  │  No modular repos         │  Module streams support          │
  │  Less informative errors  │  Better error messages           │
  │  Plugins: separate pkg    │  Plugins: built-in system        │
  ├───────────────────────────┴──────────────────────────────────┤
  │  Verdict: dnf is the drop-in replacement. On RHEL 8+,        │
  │  `yum` is literally a symlink to `dnf`. Same commands,       │
  │  better engine. Use dnf on anything modern.                  │
  └──────────────────────────────────────────────────────────────┘
```

### 21.4 Apache vs Nginx

```
  ┌───────────────────────────┬──────────────────────────────────┐
  │   Apache (httpd)          │   Nginx                          │
  ├───────────────────────────┼──────────────────────────────────┤
  │  Process/Thread per conn  │  Event-driven async              │
  │  .htaccess (per-dir)      │  Centralized config only         │
  │  mod_php (in-process)     │  Proxies to PHP-FPM (separate)   │
  │  mod_proxy (add-on)       │  Native reverse proxy            │
  │  mod_proxy_balancer       │  Native load balancer            │
  │  Higher memory @ scale    │  Minimal memory per connection   │
  │  Struggles at C10K        │  Handles C10K+ easily            │
  │  Great module ecosystem   │  Lean, focused feature set       │
  │  Best: WordPress, legacy  │  Best: reverse proxy, static,    │
  │  PHP apps, .htaccess need │  microservices, API gateway      │
  ├───────────────────────────┴──────────────────────────────────┤
  │  DevOps reality: Most modern deployments use Nginx as        │
  │  reverse proxy/LB in front of app servers. Apache still      │
  │  dominates shared hosting and WordPress. Know both.          │
  └──────────────────────────────────────────────────────────────┘
```

### 21.5 Docker vs Podman

```
  ┌───────────────────────────┬──────────────────────────────────┐
  │       Docker              │       Podman                     │
  ├───────────────────────────┼──────────────────────────────────┤
  │  dockerd daemon (root)    │  Daemonless (no root daemon)     │
  │  Root by default          │  Rootless by default ✅          │
  │  docker CLI               │  podman CLI (drop-in)            │
  │  Docker Hub default       │  Any OCI registry                │
  │  docker-compose           │  podman-compose / pods           │
  │  Swarm built-in           │  No orchestration (use K8s)      │
  │  Larger attack surface    │  Smaller attack surface          │
  │  Default: most distros    │  Default: RHEL 8+, CentOS Stream │
  │  containerd runtime       │  crun/runc runtime               │
  ├───────────────────────────┴──────────────────────────────────┤
  │  Verdict: Podman is Docker-compatible and more secure.       │
  │  `alias docker=podman` works. Use Podman for RHEL/CentOS,    │
  │  Docker for broad ecosystem compatibility. Both produce      │
  │  OCI-compliant images — images are interchangeable.          │
  └──────────────────────────────────────────────────────────────┘
```

### 21.6 rsync vs scp

```
  ┌───────────────────────────┬──────────────────────────────────┐
  │       scp                 │       rsync                      │
  ├───────────────────────────┼──────────────────────────────────┤
  │  Full copy every time     │  Delta/differential transfer     │
  │  No resume on interrupt   │  Resume interrupted transfers    │
  │  No compression option    │  -z flag for compression         │
  │  Simple syntax            │  Rich options (exclude, include) │
  │  No progress for dirs     │  --progress per file             │
  │  Good for: single files,  │  Good for: large dirs, backups,  │
  │  quick one-off transfers  │  mirrors, repeated syncs         │
  │  Deprecated in OpenSSH    │  Actively maintained             │
  │  9.0 (use sftp instead)   │  Uses SSH by default (-e ssh)    │
  ├───────────────────────────┴──────────────────────────────────┤
  │  Verdict: Use rsync for everything except the simplest       │
  │  one-file transfer. rsync -avz is the gold standard for      │
  │  server-to-server file sync, backups, and deployments.       │
  └──────────────────────────────────────────────────────────────┘
```

### 21.7 vi/vim vs nano

```
  ┌───────────────────────────┬──────────────────────────────────┐
  │       vi / vim            │       nano                       │
  ├───────────────────────────┼──────────────────────────────────┤
  │  Modal editor (modes)     │  Modeless (always inserting)     │
  │  Steep learning curve     │  Instant productivity            │
  │  Extremely powerful       │  Basic editing only              │
  │  Available on EVERY Unix  │  May need installation           │
  │  Regex find/replace       │  Simple find/replace             │
  │  Macros, scripting        │  No macros                       │
  │  Used by: sysadmins,      │  Used by: quick edits,           │
  │  DevOps pros, developers  │  beginners, commit messages      │
  ├───────────────────────────┴──────────────────────────────────┤
  │  Verdict: Learn vi/vim — it's on every server, inside        │
  │  containers, everywhere. nano is fine for quick edits, but   │
  │  vi proficiency is expected of any Linux admin.              │
  └──────────────────────────────────────────────────────────────┘
```

### 21.8 LVM vs Standard Partitions

```
  ┌───────────────────────────┬──────────────────────────────────┐
  │  Standard Partitions      │  LVM                             │
  ├───────────────────────────┼──────────────────────────────────┤
  │  Fixed size at creation   │  Resize on-the-fly               │
  │  Cannot span disks        │  Span multiple disks             │
  │  Delete + recreate to     │  lvextend / lvreduce             │
  │  resize                   │  (online for XFS grow)           │
  │  No snapshots             │  Snapshots for backup/testing    │
  │  Simpler to understand    │  More abstraction layers         │
  │  Good for: /boot, EFI,    │  Good for: /, /var, /home, data  │
  │  simple setups            │  production servers              │
  ├───────────────────────────┴──────────────────────────────────┤
  │  Verdict: Use LVM for all production servers. Standard       │
  │  partitions only for /boot and EFI. LVM's online resize      │
  │  and snapshot capabilities are essential for operations.     │
  └──────────────────────────────────────────────────────────────┘
```

### 21.9 ext4 vs XFS

```
  ┌───────────────────────────┬──────────────────────────────────┐
  │       ext4                │       XFS                        │
  ├───────────────────────────┼──────────────────────────────────┤
  │  Default: Ubuntu, Debian  │  Default: RHEL, CentOS, Amazon   │
  │  Max vol: 1 EB            │  Max vol: 500 TB (practical)     │
  │  Grow AND shrink          │  Grow only (no shrink!)          │
  │  Better for small files   │  Better for large files          │
  │  fsck.ext4 repair         │  xfs_repair                      │
  │  Mature, battle-tested    │  High-performance, parallel I/O  │
  │  Good for: general,       │  Good for: databases, media,     │
  │  containers, OS root      │  large-scale production          │
  ├───────────────────────────┴──────────────────────────────────┤
  │  Verdict: Match your distro default — XFS for RHEL/CentOS,   │
  │  ext4 for Ubuntu/Debian. Both are production-grade. XFS      │
  │  cannot be shrunk — plan capacity carefully with XFS.        │
  └──────────────────────────────────────────────────────────────┘
```

### 21.10 MBR vs GPT

```
  ┌───────────────────────────┬──────────────────────────────────┐
  │       MBR                 │       GPT                        │
  ├───────────────────────────┼──────────────────────────────────┤
  │  Max disk: 2 TB           │  Max disk: 9.4 ZB (zetabytes)    │
  │  Max partitions: 4 primary│  Max partitions: 128             │
  │  Boot mode: BIOS          │  Boot mode: UEFI                 │
  │  No backup of partition   │  Backup table at end of disk     │
  │  table                    │  (self-healing)                  │
  │  Legacy systems           │  All modern systems              │
  ├───────────────────────────┴──────────────────────────────────┤
  │  Verdict: Use GPT for everything new. MBR only for legacy    │
  │  BIOS systems that can't boot UEFI.                          │
  └──────────────────────────────────────────────────────────────┘
```

### 21.11 Postfix vs sendmail

```
  ┌──────────────────────────┬──────────────────────────────────┐
  │       sendmail (Legacy)  │       Postfix (Modern)           │
  ├──────────────────────────┼──────────────────────────────────┤
  │  Oldest MTA (1983)       │  Modern, secure replacement      │
  │  Monolithic binary       │  Modular architecture            │
  │  Complex config syntax   │  Simple key=value config         │
  │  sendmail.cf = cryptic   │  main.cf = human-readable        │
  │  Known security issues   │  Security-focused design         │
  │  Still default on some   │  Default on RHEL/CentOS          │
  │  legacy systems          │                                  │
  ├──────────────────────────┴──────────────────────────────────┤
  │  Verdict: Postfix. No reason to use sendmail on new systems.│
  │  If you see sendmail.cf, it's a legacy system.              │
  └─────────────────────────────────────────────────────────────┘
```

### 21.12 chronyd vs ntpd

```
  ┌──────────────────────────┬──────────────────────────────────┐
  │       ntpd (Legacy)      │       chronyd (Modern)           │
  ├──────────────────────────┼──────────────────────────────────┤
  │  Slower initial sync     │  Faster initial sync             │
  │  Needs constant network  │  Handles intermittent network    │
  │  Higher memory           │  Lower memory footprint          │
  │  Default: RHEL 6         │  Default: RHEL 7+                │
  │  Better for dedicated    │  Better for VMs, laptops,        │
  │  NTP servers             │  cloud instances                 │
  ├──────────────────────────┴──────────────────────────────────┤
  │  Verdict: chronyd for everything. It's the modern default   │
  │  and handles VM time drift better — critical for cloud.     │
  └─────────────────────────────────────────────────────────────┘
```

### 21.13 OpenLDAP vs FreeIPA/IdM vs Active Directory

```
  ┌─────────────────┬────────────────────┬────────────────────┐
  │   OpenLDAP      │   FreeIPA / IdM    │  Active Directory  │
  ├─────────────────┼────────────────────┼────────────────────┤
  │  Just LDAP      │  LDAP + Kerberos + │  LDAP + Kerberos + │
  │  (bare bones)   │  DNS + CA + SUDO   │  DNS + GPO         │
  │  Manual config  │  Web UI, CLI       │  GUI (ADUC/ADAC)   │
  │  Linux only     │  Linux only (RHEL) │  Windows ecosystem │
  │  No Kerberos    │  Kerberos built-in │  Kerberos built-in │
  │  No cert mgmt   │  Dogtag CA built-in│  AD CS (separate)  │
  │  High complexity│  Low complexity    │  Medium complexity │
  │  Best: embedded,│  Best: pure Linux  │  Best: Windows-    │
  │  lightweight    │  environments      │  centric orgs      │
  ├─────────────────┴────────────────────┴────────────────────┤
  │  Mixed env? Use AD + SSSD/realmd to join Linux servers.   │
  │  Pure Linux? Use FreeIPA/IdM. OpenLDAP only if you need   │
  │  absolute minimal footprint.                              │
  └───────────────────────────────────────────────────────────┘
```

### 21.14 Ansible vs Chef vs Puppet

```
  ┌─────────────────┬────────────────────┬────────────────────┐
  │   Ansible       │   Chef             │   Puppet           │
  ├─────────────────┼────────────────────┼────────────────────┤
  │  Agentless (SSH)│  Agent required    │  Agent required    │
  │  Push model     │  Pull model        │  Pull model        │
  │  YAML           │  Ruby DSL          │  Ruby DSL          │
  │  Low learning   │  Steep learning    │  Steep learning    │
  │  curve          │  curve             │  curve             │
  │  Idempotent     │  Idempotent        │  Idempotent        │
  │  Ad-hoc + playb.│  Recipes+Cookbooks │  Manifests+Modules │
  │  Massive module │  Supermarket       │  Forge             │
  │  library        │  (community)       │  (community)       │
  │  Best: DevOps,  │  Best: complex     │  Best: large-scale │
  │  cloud, CI/CD   |  infra at scale    │  enterprise config │
  ├─────────────────┴────────────────────┴────────────────────┤
  │  DevOps verdict: Ansible dominates due to agentless SSH,  │
  │  YAML simplicity, and massive cloud module support. Chef  │
  │  and Puppet are legacy choices for most new projects.     │
  └───────────────────────────────────────────────────────────┘
```

### 21.15 Ubuntu/Debian vs RHEL/CentOS vs Other Server Distributions — Cloud & DevOps Showdown

> Choosing the right server distribution is one of the **first architectural decisions** in any cloud/DevOps environment. This isn't about personal preference — it's about support, ecosystem, security posture, and operational tooling.

#### 21.15.1 Distribution Family Tree — Mental Model

```
  ┌─────────────────────────────────────────────────────────────────────┐
  │              LINUX DISTRIBUTION FAMILIES (SERVER FOCUS)             │
  │                                                                     │
  │  ┌─── Debian Family ─────────────────────────────────────────────┐  │
  │  │  Debian (upstream) ──► Ubuntu Server ──► Ubuntu Pro (paid)    │  │
  │  │                                     └──► Ubuntu LTS (free)    │  │
  │  └───────────────────────────────────────────────────────────────┘  │
  │                                                                     │
  │  ┌─── Red Hat Family ────────────────────────────────────────────┐  │
  │  │  Fedora (upstream) ──► RHEL (paid) ──► CentOS Stream          │  │
  │  │                                    └──► Oracle Linux          │  │
  │  │                                    └──► Rocky Linux           │  │
  │  │                                    └──► AlmaLinux             │  │
  │  └───────────────────────────────────────────────────────────────┘  │
  │                                                                     │
  │  ┌─── SUSE Family ───────────────────────────────────────────────┐  │
  │  │  openSUSE Tumbleweed ──► SLES (paid, enterprise)              │  │
  │  │  openSUSE Leap ────────► (community enterprise-grade)         │  │
  │  └───────────────────────────────────────────────────────────────┘  │
  │                                                                     │
  │  ┌─── Cloud-Native / Specialty ──────────────────────────────────┐  │
  │  │  Amazon Linux 2023 (RHEL/Fedora-based, AWS optimised)         │  │
  │  │  Flatcar Container Linux (immutable, container-only)          │  │
  │  │  Alpine Linux (musl-based, ultra-minimal, container images)   │  │
  │  └───────────────────────────────────────────────────────────────┘  │
  └─────────────────────────────────────────────────────────────────────┘
```

#### 21.15.2 Master Comparison Chart

| Aspect | **Ubuntu Server (Debian)** | **RHEL / CentOS Stream / Rocky / Alma** | **SLES (SUSE)** | **Amazon Linux 2023** | **Alpine Linux** |
|---|---|---|---|---|---|
| **Base family** | Debian | Fedora/RHEL | SUSE | Fedora/RHEL hybrid | Independent (musl + BusyBox) |
| **Package format** | `.deb` | `.rpm` | `.rpm` | `.rpm` | `.apk` |
| **Package manager** | `apt` / `apt-get` | `dnf` (RHEL 8+) / `yum` (7) | `zypper` | `dnf` | `apk` |
| **Init system** | systemd | systemd | systemd | systemd | OpenRC (default) |
| **Default filesystem** | ext4 | XFS | Btrfs | XFS | ext4 |
| **SELinux / AppArmor** | **AppArmor** (default) | **SELinux** (enforcing) | AppArmor | SELinux | None (grsecurity patches optional) |
| **Firewall tool** | `ufw` (iptables frontend) | `firewalld` (nftables) | `firewalld` | `firewalld` | `iptables` (manual) |
| **LTS / Support cycle** | 5 yr LTS (10 yr with Ubuntu Pro) | 10 yr (RHEL), ~5 yr (Stream) | 13 yr (LTSS paid) | ~2 yr cadence, ~3 yr support | Rolling (community) |
| **Release model** | Fixed (LTS every 2 yr) | Fixed (RHEL ~3 yr major), Stream = rolling | Fixed (major ~3-4 yr) | Fixed (~2 yr) | Rolling |
| **Commercial support** | Canonical (Ubuntu Pro) | Red Hat subscription | SUSE subscription | AWS Support (bundled) | None (community) |
| **FIPS 140-2/3 certified** | ✅ (Ubuntu Pro) | ✅ (RHEL) | ✅ (SLES) | ✅ (via RHEL lineage) | ❌ |
| **Kernel live-patching** | Livepatch (Ubuntu Pro) | kpatch (RHEL) | kGraft (SLES) | Kernel Live Patching | ❌ |
| **Cloud marketplace presence** | All clouds (very popular) | All clouds | AWS, Azure, GCP | AWS only | Containers mainly |
| **Container base image size** | ~78 MB (ubuntu:22.04) | ~230 MB (ubi9-minimal ~100 MB) | ~120 MB | ~150 MB | **~7 MB** |
| **Primary cloud bias** | Azure, GCP, general | AWS (RHEL), enterprise | SAP on Azure/AWS | AWS (default free-tier) | Docker Hub default |
| **Best for** | Startups, dev teams, K8s, general cloud | Enterprise, regulated, SAP, govt | SAP, SUSE ecosystem | AWS-native workloads | Container images, edge |

#### 21.15.3 Administration Differences — What Actually Changes Day-to-Day

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │         ADMINISTRATION DIFFERENCES — SIDE BY SIDE                    │
  ├──────────────────────┬────────────────────────┬──────────────────────┤
  │   Task               │  Ubuntu/Debian         │  RHEL/Rocky/Alma     │
  ├──────────────────────┼────────────────────────┼──────────────────────┤
  │  Install a package   │  apt install nginx     │  dnf install nginx   │ 
  │  Remove a package    │  apt remove nginx      │  dnf remove nginx    │
  │  Update all packages │  apt update && upgrade │  dnf update          │
  │  Search packages     │  apt search nginx      │  dnf search nginx    │
  │  List installed      │  dpkg -l               │  rpm -qa             │
  │  Package info        │  dpkg -s nginx         │  rpm -qi nginx       │
  │  File → package      │  dpkg -S /path/file    │  rpm -qf /path/file  │
  │  Add a repo          │  add-apt-repository /  │  dnf config-manager  │
  │                      │  edit sources.list     │  --add-repo URL      │
  │  Repo config file    │  /etc/apt/sources.list │  /etc/yum.repos.d/   │
  │                      │  .d/*.list             │  *.repo              │
  ├──────────────────────┼────────────────────────┼──────────────────────┤
  │  Security framework  │  AppArmor              │  SELinux             │
  │  Check status        │  aa-status             │  getenforce          │
  │  Profiles/Policies   │  /etc/apparmor.d/      │  /etc/selinux/       │
  │  Enforce/Complain    │  aa-enforce / complain │  setenforce 1 / 0    │
  │  Troubleshoot        │  journalctl (denied)   │  ausearch -m avc     │
  ├──────────────────────┼────────────────────────┼──────────────────────┤
  │  Firewall            │  ufw enable            │  systemctl start     │
  │                      │  ufw allow 80/tcp      │    firewalld         │
  │                      │  ufw status            │  firewall-cmd        │
  │                      │                        │   --add-service=http │
  ├──────────────────────┼────────────────────────┼──────────────────────┤
  │  Service management  │  systemctl (same)      │  systemctl (same)    │
  │  Networking          │  netplan → networkd /  │  nmcli /             │
  │                      │  NetworkManager        │  NetworkManager      │
  │  Network config dir  │  /etc/netplan/*.yaml   │  /etc/NetworkManager │
  │                      │                        │  /system-connections/│
  │  Hostname            │  hostnamectl (same)    │  hostnamectl (same)  │
  ├──────────────────────┼────────────────────────┼──────────────────────┤
  │  Default shell       │  bash (dash for sh)    │  bash                │
  │  Sudo group          │  sudo                  │  wheel               │
  │  Root login          │  Disabled by default   │  Enabled by default  │
  │  Log directory       │  /var/log/syslog       │  /var/log/messages   │
  │  Audit logs          │  /var/log/auth.log     │  /var/log/secure     │
  └──────────────────────┴────────────────────────┴──────────────────────┘
```

#### 21.15.4 Security Model — AppArmor vs SELinux Fundamentally

```
  ┌───────────────────────────┬──────────────────────────────────────┐
  │  AppArmor (Ubuntu/SUSE)   │  SELinux (RHEL/CentOS/Fedora)        │
  ├───────────────────────────┼──────────────────────────────────────┤
  │  Path-based enforcement   │  Label-based enforcement (contexts)  │
  │  Per-application profiles │  System-wide mandatory policy        │
  │  Easier to write rules    │  Steeper learning curve              │
  │  Profiles: enforce /      │  Modes: Enforcing / Permissive /     │
  │  complain / disable       │  Disabled                            │
  │  Simpler for containers   │  Deeper container isolation          │
  │  Adequate for most cases  │  Required for compliance (PCI,HIPAA) │
  │  "Good enough" security   │  "Defense in depth" security         │
  ├───────────────────────────┴──────────────────────────────────────┤
  │  Cloud reality: Most CIS benchmarks expect SELinux on RHEL       │
  │  and AppArmor on Ubuntu. NEVER disable either — learn to         │
  │  troubleshoot them instead.                                      │
  └──────────────────────────────────────────────────────────────────┘
```

#### 21.15.5 Cloud & DevOps Adoption — Where Each Distro Dominates

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │                   CLOUD / DEVOPS ADOPTION MAP                        │
  ├──────────────────────────────────────────────────────────────────────┤
  │                                                                      │
  │  AWS:                                                                │
  │    • Amazon Linux 2023 — default free-tier, best integration         │
  │    • Ubuntu — most launched AMI overall                              │
  │    • RHEL — enterprise / govt workloads                              │
  │                                                                      │
  │  Azure:                                                              │
  │    • Ubuntu — #1 Linux on Azure (Canonical partnership)              │
  │    • RHEL — enterprise (Red Hat + Microsoft alliance)                │
  │    • SLES — SAP HANA certified workloads                             │
  │                                                                      │
  │  GCP:                                                                │
  │    • Ubuntu — default in many GCP tutorials/quickstarts              │
  │    • Container-Optimized OS (ChromeOS-based, GKE nodes)              │
  │    • RHEL — enterprise customers                                     │
  │                                                                      │
  │  Kubernetes (nodes):                                                 │
  │    • Ubuntu — most popular K8s node OS (EKS, AKS, GKE)               │
  │    • Flatcar / Bottlerocket — immutable, container-optimised         │
  │    • RHEL CoreOS — OpenShift nodes (Red Hat ecosystem)               │
  │                                                                      │
  │  Container base images:                                              │
  │    • Alpine — smallest (~7 MB), most pulled on Docker Hub            │
  │    • Ubuntu — when you need apt & full userland                      │
  │    • UBI (Red Hat) — free redistribution, RHEL compatible            │
  │    • Distroless (Google) — no shell, no pkg mgr, minimal attack      │
  │      surface — ideal for production microservices                    │
  │                                                                      │
  │  CI/CD runners:                                                      │
  │    • Ubuntu — default for GitHub Actions, GitLab CI, CircleCI        │
  │    • Alpine — used in lightweight pipeline steps                     │
  │                                                                      │
  │  Configuration Management / IaC:                                     │
  │    • Ansible modules work on all, but yum/dnf vs apt                 │
  │      modules differ — playbooks must be distro-aware                 │
  │    • Packer images: choose one family and standardize                │
  │                                                                      │
  └──────────────────────────────────────────────────────────────────────┘
```

#### 21.15.6 Licensing, Cost & Support — The Business Side

| | **Ubuntu LTS** | **Ubuntu Pro** | **RHEL** | **Rocky / Alma** | **SLES** | **Amazon Linux** |
|---|---|---|---|---|---|---|
| **License cost** | Free | Free (personal, 5 nodes) / Paid | Subscription | Free | Subscription | Free (AWS only) |
| **Vendor support** | Community only | Canonical 24/7 | Red Hat 24/7 | Community / CIQ (Rocky) | SUSE 24/7 | AWS Support |
| **Security patches SLA** | Best-effort | USN within 24 hr (critical) | RHSA within 1 business day | Follows RHEL | Within 24 hr | Follows Fedora/RHEL |
| **CVE backporting** | ✅ (main repo) | ✅ (universe too) | ✅ (10 yr) | ✅ (mirrors RHEL) | ✅ | ✅ |
| **Compliance certs** | CIS, DISA-STIG (Pro) | FIPS, FedRAMP, HIPAA | FIPS, CC, DISA-STIG | Community CIS | FIPS, CC, EAL4+ | FedRAMP (via AWS) |
| **Ideal org type** | Startups, SMBs, dev teams | Regulated SMBs | Large enterprise, govt | Cost-conscious enterprise | SAP shops | AWS-native orgs |

#### 21.15.7 Decision Flowchart — Which Server OS to Pick?

```
  ┌──────────────────────────────────────────────────────────────────┐
  │            CHOOSING A SERVER OS — DECISION TREE                  │
  │                                                                  │
  │  Need FIPS / FedRAMP / strict compliance?                        │
  │    ├─ YES → RHEL (gold standard) or Ubuntu Pro                   │
  │    └─ NO ↓                                                       │
  │                                                                  │
  │  Running SAP HANA or SAP workloads?                              │
  │    ├─ YES → SLES (SAP-certified) or RHEL for SAP                 │
  │    └─ NO ↓                                                       │
  │                                                                  │
  │  Exclusively on AWS and want zero license cost?                  │
  │    ├─ YES → Amazon Linux 2023                                    │
  │    └─ NO ↓                                                       │
  │                                                                  │
  │  Building container images (minimal footprint)?                  │
  │    ├─ YES → Alpine (tiny) or Distroless (no shell)               │
  │    └─ NO ↓                                                       │
  │                                                                  │
  │  Enterprise with Red Hat ecosystem (OpenShift, Satellite)?       │
  │    ├─ YES → RHEL                                                 │
  │    └─ NO ↓                                                       │
  │                                                                  │
  │  Want RHEL compatibility without subscription cost?              │
  │    ├─ YES → Rocky Linux or AlmaLinux                             │
  │    └─ NO ↓                                                       │
  │                                                                  │
  │  General cloud workloads, K8s, CI/CD, dev environments?          │
  │    └─ Ubuntu LTS (largest community, most docs, widest support)  │
  │                                                                  │
  └──────────────────────────────────────────────────────────────────┘
```

#### 21.15.8 Ansible Playbook Portability — Practical Impact

> When you manage a **mixed fleet**, every playbook needs distro-awareness. This is the #1 operational pain point of multi-distro environments.

```yaml
# Example: Install Nginx — distro-aware Ansible task
- name: Install Nginx on Debian/Ubuntu
  ansible.builtin.apt:
    name: nginx
    state: present
    update_cache: yes
  when: ansible_os_family == "Debian"

- name: Install Nginx on RHEL/Rocky/Alma
  ansible.builtin.dnf:
    name: nginx
    state: present
  when: ansible_os_family == "RedHat"

- name: Install Nginx on SUSE
  community.general.zypper:
    name: nginx
    state: present
  when: ansible_os_family == "Suse"

# ⚠️ Config paths also differ:
#   Debian:  /etc/nginx/sites-available/ + sites-enabled/
#   RHEL:    /etc/nginx/conf.d/*.conf   (no sites-available)
```

#### 21.15.9 Key Takeaways

```
  ╔══════════════════════════════════════════════════════════════════╗
  ║          SERVER OS SELECTION — KEY TAKEAWAYS                     ║
  ╠══════════════════════════════════════════════════════════════════╣
  ║                                                                  ║
  ║  1. STANDARDIZE: Pick ONE distro family for your org.            ║
  ║     Mixing Debian + RHEL = double the playbooks, double          ║
  ║     the troubleshooting, double the golden image builds.         ║
  ║                                                                  ║
  ║  2. Ubuntu wins on: community size, cloud docs, CI/CD            ║
  ║     defaults, container popularity, developer friendliness.      ║
  ║                                                                  ║
  ║  3. RHEL wins on: enterprise support SLAs, compliance certs,     ║
  ║     10-yr lifecycle, SELinux depth, SAP/Oracle certification.    ║
  ║                                                                  ║
  ║  4. Alpine wins on: container image size. Period.                ║
  ║     Don't use Alpine as a server OS — musl libc causes subtle    ║
  ║     compatibility issues with glibc-compiled software.           ║
  ║                                                                  ║
  ║  5. Amazon Linux wins on: AWS-native integration, free on AWS,   ║
  ║     optimized kernel for EC2. Loses on: portability (AWS-only).  ║
  ║                                                                  ║
  ║  6. Rocky/Alma = "free RHEL" — great for cost-conscious orgs     ║
  ║     that need RHEL binary compatibility without subscription.    ║
  ║                                                                  ║
  ║  7. KNOW BOTH families — interviews & real-world will test you   ║
  ║     on apt vs dnf, AppArmor vs SELinux, ufw vs firewalld,        ║
  ║     netplan vs nmcli. A DevOps engineer is distro-fluent.        ║
  ║                                                                  ║
  ╚══════════════════════════════════════════════════════════════════╝
```

---

## 22. Viva / Trainee Review — Q&A Bank

> 💡 **How to use:** Cover each question, try to answer from memory first, then verify. These cover the most common review/interview topics.

### 22.1 Boot Process & System Architecture

**Q1: Walk through the Linux boot process from power-on to login prompt.**
> BIOS/UEFI → POST → MBR/GPT bootloader → GRUB2 → kernel load (vmlinuz) → initramfs (temp root) → systemd (PID 1) → default.target → login prompt. Key: GRUB2 loads the kernel, kernel mounts initramfs to find the real root filesystem, then hands off to systemd.

**Q2: What is systemd and why did it replace SysVinit?**
> systemd is the modern init system (PID 1). It replaced SysVinit because it boots in parallel (faster), has built-in service supervision (auto-restart), dependency management, socket activation, structured logging (journalctl), and cgroup-based resource control.

**Q3: How do you rescue a system that won't boot?**
> Interrupt GRUB → press `e` → add `rd.break` or `init=/bin/bash` to kernel line → boot → `chroot /sysroot` → fix the issue (reset password, fix fstab, etc.) → `touch /.autorelabel` (SELinux) → reboot.

**Q4: What is the difference between `systemctl enable` and `systemctl start`?**
> `start` runs the service NOW (this session). `enable` creates symlinks so the service starts automatically ON BOOT. Use `enable --now` to do both.

### 22.2 File System & Permissions

**Q5: Explain the Linux permission model — owner, group, others.**
> Every file has owner (u), group (g), others (o) with read (4), write (2), execute (1) bits. Directories need execute to enter (`cd`). `chmod 755` = rwxr-xr-x. `chmod 600` = rw------- (private keys).

**Q6: What are setuid, setgid, and sticky bit? Give real examples.**
> **setuid (4xxx):** File executes as the file OWNER, not the user. Example: `/usr/bin/passwd` runs as root so normal users can change their password. **setgid (2xxx):** On files: executes as file's group. On directories: new files inherit the directory's group (team collaboration). **Sticky bit (1xxx):** On directories: only file owner can delete their files. Example: `/tmp` (1777) — everyone can write, but you can't delete others' files.

**Q7: What's the difference between hard links and soft links?**
> **Hard link:** Same inode, same data blocks, can't cross filesystems, can't link directories, survives original deletion. **Soft link (symlink):** Different inode, points to filename, can cross filesystems, can link directories, breaks if original deleted.

### 22.3 User & Group Administration

**Q8: What files store user and group information?**
> `/etc/passwd` (user accounts — UID, GID, home, shell), `/etc/shadow` (encrypted passwords, aging), `/etc/group` (group memberships), `/etc/gshadow` (group passwords). Never edit these directly — use `useradd`, `usermod`, `passwd`.

**Q9: How do you give a user sudo access?**
> Add to `wheel` group: `usermod -aG wheel username`. Or edit `/etc/sudoers` with `visudo`: `username ALL=(ALL) ALL` or `username ALL=(ALL) NOPASSWD: ALL` (no password prompt).

**Q10: Difference between `useradd` and `adduser`?**
> On RHEL/CentOS: identical (adduser is a symlink). On Debian/Ubuntu: `adduser` is interactive (prompts for details), `useradd` is low-level (no prompts, no home dir by default unless `-m`).

### 22.4 Process Management

**Q11: How do you find and kill a process using a specific port?**
> `lsof -i :8080` or `ss -tlnp | grep 8080` → find PID → `kill <PID>` (graceful SIGTERM) → if stuck, `kill -9 <PID>` (SIGKILL, forceful). Or one-liner: `fuser -k 8080/tcp`.

**Q12: What is a zombie process and how do you handle it?**
> A zombie is a child process that has finished but its parent hasn't read its exit status (`wait()`). Shows as `Z` in `ps`. You can't kill a zombie (it's already dead). Fix: kill the PARENT process, or reboot. Zombies don't consume resources except a PID table entry.

**Q13: Difference between `kill`, `kill -9`, and `kill -15`?**
> `kill` / `kill -15` = SIGTERM (graceful — process can clean up, close connections). `kill -9` = SIGKILL (force — kernel terminates immediately, no cleanup). Always try SIGTERM first. SIGKILL is a last resort.

### 22.5 Networking

**Q14: How do you troubleshoot "server is unreachable"?**
> Systematic approach: 1) `ping server-ip` (is it up?), 2) `traceroute server-ip` (where does it fail?), 3) `ss -tlnp` on server (is the service listening?), 4) `firewall-cmd --list-all` (is the port open?), 5) `curl localhost:port` on server (does the app respond locally?), 6) Check DNS: `dig` / `nslookup`, 7) Check SELinux: `ausearch -m avc`.

**Q15: Explain DNS resolution order on a Linux system.**
> `/etc/nsswitch.conf` defines order (usually `files dns`): 1) `/etc/hosts` (local file first), 2) DNS query to servers in `/etc/resolv.conf`. For DNS query: local cache → recursive resolver → root servers → TLD servers → authoritative server → answer.

**Q16: What's the difference between TCP and UDP?**
> **TCP:** Connection-oriented, reliable (acknowledgments, retransmission), ordered delivery, slower. Used for: HTTP, SSH, databases. **UDP:** Connectionless, unreliable (no acks), unordered, faster. Used for: DNS, NTP, streaming, VoIP.

### 22.6 SSH & Remote Access

**Q17: How do you set up SSH key-based authentication?**
> 1) Generate keypair: `ssh-keygen -t ed25519`, 2) Copy public key: `ssh-copy-id user@server`, 3) Disable password auth: edit `/etc/ssh/sshd_config` → `PasswordAuthentication no`, 4) Restart: `systemctl restart sshd`. Private key stays local, public key goes to server's `~/.ssh/authorized_keys`.

**Q18: How do you harden SSH?**
> Disable root login (`PermitRootLogin no`), key auth only (`PasswordAuthentication no`), change port (optional), allow specific users (`AllowUsers`), use fail2ban (auto-block brute force), set `MaxAuthTries 3`, disable empty passwords.

### 22.7 Storage & LVM

**Q19: Walk through extending a logical volume online.**
> 1) Check free space: `vgs` (free in VG?), 2) If no space, add disk: `pvcreate /dev/sdd` → `vgextend vg-name /dev/sdd`, 3) Extend LV: `lvextend -L +20G /dev/vg-name/lv-name`, 4) Grow filesystem: `xfs_growfs /mount-point` (XFS) or `resize2fs /dev/vg-name/lv-name` (ext4). All done online, no downtime.

**Q20: What is /etc/fstab and what happens if it's wrong?**
> `/etc/fstab` defines permanent mounts (applied at boot via `mount -a`). If an entry is wrong (bad UUID, missing device), the system may fail to boot or drop to emergency shell. Always run `mount -a` after editing to test. Use `nofail` mount option for non-critical mounts.

### 22.8 Package Management

**Q21: How do you roll back a bad update?**
> `yum history` → find transaction ID → `yum history undo <ID>`. This reverts the specific transaction (downgrades packages). For critical systems, take snapshots before updates.

**Q22: What's the difference between rpm and yum/dnf?**
> `rpm` is the low-level tool — installs/queries individual .rpm files, does NOT resolve dependencies. `yum/dnf` is the high-level tool — searches repositories, resolves and installs all dependencies automatically. Use `yum/dnf` for installation, `rpm -qa/-qi/-ql` for querying.

### 22.9 Services & Security

**Q23: How do you open a port in the firewall?**
> `firewall-cmd --permanent --add-port=8080/tcp && firewall-cmd --reload`. Or by service name: `firewall-cmd --permanent --add-service=http && firewall-cmd --reload`. Verify: `firewall-cmd --list-all`.

**Q24: A web server returns "permission denied" but file permissions are correct. What's wrong?**
> Likely SELinux. Check: `getenforce` (if Enforcing), `ausearch -m avc -ts recent` (find denial). Fix: `restorecon -Rv /var/www/html/` (fix context) or enable the right boolean: `setsebool -P httpd_can_network_connect on`.

**Q25: How do you check what ports are open and what's listening?**
> `ss -tlnp` (TCP listening, numeric, with process), `ss -ulnp` (UDP). Or `netstat -tlnp` (legacy). For specific port: `ss -tlnp | grep :22`.

### 22.10 Containers & Configuration Management

**Q26: What is the difference between a Docker image and a container?**
> **Image:** Read-only template (blueprint). Built from Dockerfile. Stored in registries. Immutable. **Container:** Running instance of an image. Writable layer on top. Has its own process, network, filesystem. Multiple containers from same image.

**Q27: Why is Ansible called "agentless"?**
> Ansible connects to managed nodes over SSH (Linux) or WinRM (Windows). No software needs to be installed on target servers — just SSH access and Python. This simplifies deployment and reduces maintenance overhead compared to Chef/Puppet which require agent installation.

**Q28: What does "idempotent" mean in Ansible?**
> Running a playbook multiple times produces the SAME result. If nginx is already installed, `yum: name=nginx state=present` does nothing (no error, no reinstall). This makes playbooks safe to run repeatedly. Status shows "ok" (no change) vs "changed" (action taken).

### 22.11 Monitoring & HA

**Q29: What is a Single Point of Failure (SPOF) and how do you eliminate it?**
> A SPOF is any component whose failure brings down the entire service. Eliminate by: redundant servers (Active-Active or Active-Passive), load balancers, database replication, multiple network paths, redundant power. In cloud: multi-AZ deployments, Auto Scaling Groups, managed HA services (RDS Multi-AZ, ELB).

**Q30: Explain Active-Passive vs Active-Active HA.**
> **Active-Passive:** One node handles traffic, standby takes over on failure. Simple, may waste resources. Uses heartbeat + VIP failover. **Active-Active:** All nodes handle traffic simultaneously via load balancer. Better resource utilization, more complex. Both eliminate SPOF.

---

## 23. DevOps/Cloud Mapping Table + Quick Revision Card

### 23.1 Linux → Cloud Service Mapping

> Every Linux concept maps to a cloud managed service. Understanding both gives you the "why" behind cloud architectures.

| Linux Concept | AWS | Azure | GCP | Why it Matters |
|---------------|-----|-------|-----|----------------|
| **systemd** | EC2 User Data, ASG health checks | VM Extensions | Startup Scripts | Service lifecycle management |
| **crontab** | EventBridge + Lambda | Azure Functions Timer | Cloud Scheduler | Scheduled tasks without servers |
| **firewalld/iptables** | Security Groups, NACLs | NSG (Network Security Groups) | VPC Firewall Rules | Network access control |
| **SELinux** | IAM Policies, Resource Policies | Azure RBAC, Policy | IAM, Org Policies | Mandatory access control |
| **LVM** | EBS Volumes (resize) | Managed Disks | Persistent Disks | Flexible storage |
| **NFS** | EFS (Elastic File System) | Azure Files (NFS) | Filestore | Shared filesystems |
| **RAID** | EBS io2 (built-in redundancy) | Premium SSD (managed) | Local SSD (striped) | Disk redundancy |
| **DNS (BIND)** | Route 53 | Azure DNS | Cloud DNS | Name resolution |
| **DHCP** | VPC DHCP Option Sets | VNET DHCP | VPC (automatic) | IP assignment |
| **Apache/Nginx** | ALB (Application LB) | App Gateway | Cloud Load Balancing | HTTP routing + TLS |
| **Squid Proxy** | NAT Gateway, PrivateLink | Azure Firewall, NAT GW | Cloud NAT | Outbound internet access |
| **NTP (chronyd)** | Amazon Time Sync (169.254.169.123) | Hyper-V Time Sync | Google NTP | Time synchronization |
| **OpenVPN** | Client VPN, Site-to-Site VPN | VPN Gateway | Cloud VPN | Secure remote access |
| **LDAP/AD** | AWS Directory Service, IAM Identity Center | Entra ID (Azure AD) | Cloud Identity | Centralized auth |
| **Pacemaker/HA** | Auto Scaling Groups + ELB | Availability Sets + LB | Instance Groups + LB | High availability |
| **Nagios** | CloudWatch | Azure Monitor | Cloud Monitoring | Infrastructure monitoring |
| **rsync** | S3 Sync, DataSync | AzCopy | gsutil rsync | Data transfer/sync |
| **Docker/Podman** | ECS, EKS, Fargate | AKS, ACI | GKE, Cloud Run | Container orchestration |
| **Ansible** | Systems Manager, CloudFormation | ARM/Bicep, Automation | Deployment Manager | IaC / Config management |
| **journalctl** | CloudWatch Logs | Log Analytics | Cloud Logging | Centralized logging |
| **tuned** | Instance types (optimized) | VM sizes (compute/memory) | Machine types | Performance optimization |
| **/etc/fstab** | EBS auto-attach (user data) | Managed Disk attach | Persistent Disk attach | Boot-time mounts |

### 23.2 Quick Revision Card — Command Cheatsheet

```
  ╔══════════════════════════════════════════════════════════════╗
  ║           🐧 LINUX ADMIN QUICK REVISION CARD                 ║
  ╠══════════════════════════════════════════════════════════════╣
  ║                                                              ║
  ║  ─── SYSTEM INFO ───                                         ║
  ║  hostname / uname -a / cat /etc/os-release                   ║
  ║  uptime / who / w / last / lastb                             ║
  ║  df -hT / lsblk / free -h / lscpu                            ║
  ║                                                              ║
  ║  ─── SERVICE MANAGEMENT ───                                  ║
  ║  systemctl start|stop|restart|status <svc>                   ║
  ║  systemctl enable|disable <svc>                              ║
  ║  systemctl enable --now <svc>     (start + enable)           ║
  ║  systemctl list-units --type=service --state=running         ║
  ║  journalctl -u <svc> -f           (follow logs)              ║
  ║                                                              ║
  ║  ─── USER MANAGEMENT ───                                     ║
  ║  useradd -m -s /bin/bash <user>   (create with home)         ║
  ║  passwd <user>                    (set password)             ║
  ║  usermod -aG wheel <user>         (add sudo access)          ║
  ║  userdel -r <user>                (delete with home)         ║
  ║  id <user> / groups <user>                                   ║
  ║                                                              ║
  ║  ─── FILE PERMISSIONS ───                                    ║
  ║  chmod 755 file    (rwxr-xr-x)                               ║
  ║  chmod 600 file    (rw-------)   [SSH keys]                  ║
  ║  chown user:group file                                       ║
  ║  chmod u+s file    (setuid)                                  ║
  ║  chmod g+s dir     (setgid — inherit group)                  ║
  ║  chmod +t dir      (sticky bit — /tmp)                       ║
  ║                                                              ║
  ║  ─── FINDING THINGS ───                                      ║
  ║  find / -name "*.conf" -type f                               ║
  ║  find / -user root -perm -4000    (find SUID files)          ║
  ║  grep -rn "pattern" /path/                                   ║
  ║  locate filename    (fast, uses updatedb cache)              ║
  ║  which <cmd> / whereis <cmd>                                 ║
  ║                                                              ║
  ║  ─── TEXT PROCESSING ───                                     ║
  ║  cat / head -20 / tail -20 / tail -f (follow)                ║
  ║  grep -i "pattern" file                                      ║
  ║  awk '{print $1,$3}' file   (columns)                        ║
  ║  sed 's/old/new/g' file     (find-replace)                   ║
  ║  cut -d: -f1 /etc/passwd    (split by delimiter)             ║
  ║  sort | uniq -c | sort -rn  (top frequencies)                ║
  ║  wc -l file                 (line count)                     ║
  ║                                                              ║
  ║  ─── PROCESS MANAGEMENT ───                                  ║
  ║  ps aux / ps -ef             (all processes)                 ║
  ║  top / htop                  (live monitoring)               ║
  ║  kill <PID>                  (SIGTERM — graceful)            ║
  ║  kill -9 <PID>               (SIGKILL — force)               ║
  ║  bg / fg / jobs              (job control)                   ║
  ║  nohup cmd &                 (survive logout)                ║
  ║                                                              ║
  ║  ─── NETWORKING ───                                          ║
  ║  ip addr / ip route          (interfaces & routes)           ║
  ║  ss -tlnp                    (listening ports)               ║
  ║  ping / traceroute / mtr     (connectivity)                  ║
  ║  dig / nslookup              (DNS lookup)                    ║
  ║  curl -v http://host:port    (HTTP test)                     ║
  ║  nmcli con show / modify     (NetworkManager)                ║
  ║                                                              ║
  ║  ─── SSH & REMOTE ───                                        ║
  ║  ssh-keygen -t ed25519                                       ║
  ║  ssh-copy-id user@server                                     ║
  ║  scp file user@server:/path/                                 ║
  ║  rsync -avz src/ user@server:/dest/                          ║
  ║                                                              ║
  ║  ─── FIREWALL ───                                            ║
  ║  firewall-cmd --list-all                                     ║
  ║  firewall-cmd --permanent --add-service=http                 ║
  ║  firewall-cmd --permanent --add-port=8080/tcp                ║
  ║  firewall-cmd --reload                                       ║
  ║                                                              ║
  ║  ─── STORAGE / LVM ───                                       ║
  ║  lsblk / df -hT / blkid                                      ║
  ║  pvcreate → vgcreate → lvcreate → mkfs → mount               ║
  ║  lvextend -r -L +20G /dev/vg/lv  (extend + resize)           ║
  ║  /etc/fstab → mount -a           (persistent mounts)         ║
  ║                                                              ║
  ║  ─── PACKAGES ───                                            ║
  ║  yum install / remove / update <pkg>                         ║
  ║  yum search <pkg> / yum provides <file>                      ║
  ║  yum history / yum history undo <ID>  (rollback!)            ║
  ║  rpm -qa / -qi / -ql <pkg>            (query)                ║
  ║                                                              ║
  ║  ─── CONTAINERS ───                                          ║
  ║  docker run -d -p 80:80 --name web nginx                     ║
  ║  docker ps / logs / exec -it <name> bash                     ║
  ║  docker build -t myapp:v1 .                                  ║
  ║  docker stop / rm / rmi                                      ║
  ║                                                              ║
  ║  ─── ANSIBLE ───                                             ║
  ║  ansible all -m ping                                         ║
  ║  ansible-playbook -i inventory playbook.yml                  ║
  ║  ansible-playbook ... --check   (dry run)                    ║
  ║                                                              ║
  ║  ─── SELINUX ───                                             ║
  ║  getenforce / sestatus                                       ║
  ║  setenforce 0|1                  (Permissive/Enforcing)      ║
  ║  restorecon -Rv /path/           (fix context)               ║
  ║  setsebool -P <boolean> on       (enable feature)            ║
  ║  ausearch -m avc -ts recent      (check denials)             ║
  ║                                                              ║
  ║  ─── SCHEDULING ───                                          ║
  ║  crontab -e / -l                 (edit/list cron)            ║
  ║  * * * * * command               (min hr dom mon dow)        ║
  ║  at now + 5 minutes              (one-time task)             ║
  ║                                                              ║
  ║  ─── LOGS ───                                                ║
  ║  journalctl -u <svc> --since "1 hour ago"                    ║
  ║  journalctl -p err               (errors only)               ║
  ║  /var/log/messages, /var/log/secure, /var/log/audit/         ║
  ║                                                              ║
  ║  ─── EMERGENCY ───                                           ║
  ║  GRUB: press 'e' → add rd.break → chroot /sysroot            ║
  ║  Reset root pw: passwd → touch /.autorelabel → reboot        ║
  ║  Disk full: find / -xdev -size +100M | sort                  ║
  ║  Can't SSH: check firewall, sshd status, /var/log/secure     ║
  ║                                                              ║
  ╚══════════════════════════════════════════════════════════════╝
```

### 23.3 The DevOps Engineer's Linux Mindset

```
  ┌──────────────────────────────────────────────────────────────┐
  │                                                              │
  │  "Everything is a file."        — Unix philosophy            │
  │  "Do one thing and do it well." — Unix philosophy            │
  │  "Automate everything you do twice."                         │
  │  "If it's not in Git, it doesn't exist."                     │
  │  "If it's not monitored, it's not in production."            │
  │  "Cattle, not pets." — Treat servers as replaceable.         │
  │  "Immutable infrastructure." — Don't patch, rebuild.         │
  │                                                              │
  │  Your Workflow:                                              │
  │  1. Understand the system (read logs, check status)          │
  │  2. Automate the fix (Ansible playbook, shell script)        │
  │  3. Version it (Git)                                         │
  │  4. Test it (staging environment)                            │
  │  5. Deploy it (CI/CD pipeline)                               │
  │  6. Monitor it (Prometheus/Grafana/CloudWatch)               │
  │  7. Iterate                                                  │
  │                                                              │
  └──────────────────────────────────────────────────────────────┘
```

---

<!-- END OF BATCH 5 — Sections 21-23 -->
<!-- ✅ ALL BATCHES COMPLETE — HANDBOOK FINISHED -->

---

[🏠 Home](../README.md) · [Linux](README.md)
