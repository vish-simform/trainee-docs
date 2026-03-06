[🏠 Home](../README.md) · [Labs](README.md)

# 🔍 Linux Disaster Recovery Lab — Recovering Deleted Nginx Configuration Using File Carving

> **Audience:** DevOps / Cloud Engineers & Trainees | **Difficulty:** Advanced  
> **Time:** 2–3 hours | **Risk Level:** Destructive — lab/disposable environments only  
> **Tip:** This lab pairs well with the [/etc Deletion DR Lab](linux_etc_disaster_recovery.md). Together they cover two complementary recovery approaches — package-manager reconstruction vs forensic file carving.

---

## Table of Contents

1. [Lab Overview](#1-lab-overview)
2. [Lab Environment](#2-lab-environment)
3. [System Architecture Before Failure](#3-system-architecture-before-failure)
4. [Phase 1 — The Configuration Loss](#4-phase-1--the-configuration-loss)
5. [Phase 2 — Why Standard Recovery Methods Failed](#5-phase-2--why-standard-recovery-methods-failed)
6. [Phase 3 — Filesystem Identification & Damage Containment](#6-phase-3--filesystem-identification--damage-containment)
7. [Phase 4 — Installing Forensic Tools on CentOS/RHEL](#7-phase-4--installing-forensic-tools-on-centosrhel)
8. [Phase 5 — Running PhotoRec (File Carving)](#8-phase-5--running-photorec-file-carving)
9. [Phase 6 — Filtering Recovered Files](#9-phase-6--filtering-recovered-files)
10. [Phase 7 — Extracting Configuration Blocks](#10-phase-7--extracting-configuration-blocks)
11. [Phase 8 — Rebuilding Nginx Directory Structure & Configs](#11-phase-8--rebuilding-nginx-directory-structure--configs)
12. [Phase 9 — Restoring Root Configuration (`nginx.conf`)](#12-phase-9--restoring-root-configuration-nginxconf)
13. [Phase 10 — Validation & Service Recovery](#13-phase-10--validation--service-recovery)
14. [Phase 11 — Restoring the Node.js Backend](#14-phase-11--restoring-the-nodejs-backend)
15. [Final System State](#15-final-system-state)
16. [Gotchas & Lessons Learned](#16-gotchas--lessons-learned)
17. [Production-Grade Prevention & Recovery](#17-production-grade-prevention--recovery)
18. [Variables That Change Everything](#18-variables-that-change-everything)
19. [Best Practices — Mapping Lab Lessons to Production](#19-best-practices--mapping-lab-lessons-to-production)
20. [Suggested Extensions](#20-suggested-extensions)
21. [Final Thoughts](#21-final-thoughts)

---

## 1. Lab Overview

This lab simulates a **real-world operational disaster** where critical Nginx configuration files — including upstream definitions, virtual host server blocks, and the root `nginx.conf` — are accidentally deleted on a production-like server.

Unlike the [/etc Deletion Lab](linux_etc_disaster_recovery.md) where a Recovery ISO and package manager were used, this scenario is worse in one specific way: **the Nginx service had already stopped**, meaning the configuration could not be pulled from memory.

Recovery was performed through **filesystem forensics and manual reconstruction**.

### What This Lab Teaches

```
  ┌────────────────────────────────────────────────────────────────┐
  │                    SKILLS DEMONSTRATED                         │
  │                                                                │
  │  ┌──────────────────┐  ┌──────────────────┐  ┌─────────────┐   │
  │  │ Filesystem       │  │ Forensics        │  │ Service     │   │
  │  │ Internals        │  │ & File Carving   │  │ Recovery    │   │
  │  │ • ext4 deletion  │  │ • PhotoRec       │  │ • Nginx     │   │
  │  │ • inode behavior │  │ • grep forensics │  │ • PM2/Node  │   │
  │  │ • remount ro     │  │ • sed/awk        │  │ • systemd   │   │
  │  └──────────────────┘  └──────────────────┘  └─────────────┘   │
  │  ┌──────────────────┐  ┌──────────────────┐  ┌─────────────┐   │
  │  │ Config Reverse   │  │ RHEL/CentOS      │  │ DR Strategy │   │
  │  │ Engineering      │  │ Package Mgmt     │  │ & Prevention│   │
  │  │ • server blocks  │  │ • dnf reinstall  │  │ • etckeeper │   │
  │  │ • upstream pools │  │ • EPEL repos     │  │ • IaC       │   │
  │  │ • vhost rebuild  │  │ • CRB enablement │  │ • backups   │   │
  │  └──────────────────┘  └──────────────────┘  └─────────────┘   │
  └────────────────────────────────────────────────────────────────┘
```

### How This Differs From the `/etc` Lab

| Aspect                | `/etc` Deletion Lab                  | This Lab (Nginx Config)              |
| --------------------- | ------------------------------------ | ------------------------------------ |
| Scope of loss         | Entire system config                 | Service-specific config              |
| OS                    | Ubuntu 24.04                         | CentOS / RHEL-based                  |
| Recovery method       | ISO transplant + package reinstall   | Forensic file carving (PhotoRec)     |
| Package manager       | `apt` (DEB822)                       | `dnf` / `yum` (RPM)                  |
| Key challenge         | System identity + repo format        | Finding config fragments on raw disk |
| Service was running?  | No (reboot killed everything)        | No (Nginx stopped before recovery)   |

---

## 2. Lab Environment

### Environment Used

| Component          | Value                       |
| ------------------ | --------------------------- |
| Provider           | DigitalOcean                |
| OS                 | CentOS / RHEL 9             |
| Filesystem         | ext4                        |
| Web Server         | Nginx (reverse proxy)       |
| Application Layer  | Node.js managed by PM2      |
| Domains            | `test.vishvam.dev`, `dynamic.test.vishvam.dev`  |
| Root Access        | Required                    |

### What Was Running

The server hosted a **reverse-proxy → Node.js** stack:

* Nginx accepting HTTP on port 80
* Multiple Node.js instances on backend ports (`:3000`, `:3001`, `:3002`)
* PM2 process manager handling Node lifecycle
* Custom helper scripts (`start-instance.sh`, `kill-instance.sh`)

### Reproducibility

This lab can be reproduced on any system running:

| Platform          | Notes                                                        |
| ----------------- | ------------------------------------------------------------ |
| CentOS 9 Stream   | Closest match to this lab                                    |
| Rocky Linux 9     | RHEL-compatible, identical package paths                     |
| AlmaLinux 9       | RHEL-compatible                                              |
| Fedora            | `dnf` based, slightly newer packages                         |
| Ubuntu/Debian     | Adjust paths (`/etc/nginx/` structure differs slightly)      |

> **Recommendation:** Take a VM snapshot **before** deleting configs so you can repeat the carving exercise without reprovisioning.

---

## 3. System Architecture Before Failure

### Traffic Flow

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                          │
  │   Internet                                               │
  │      │                                                   │
  │      ▼                                                   │
  │   ┌──────────────────────┐                               │
  │   │  Nginx (port 80)     │                               │
  │   │  • nginx.conf        │                               │
  │   │  • conf.d/*.conf     │                               │
  │   │  • upstreams/*.conf  │                               │
  │   └──────────┬───────────┘                               │
  │              │                                           │
  │              ▼                                           │
  │   ┌──────────────────────┐                               │
  │   │  upstream node_pool  │                               │
  │   │  ┌────────────────┐  │                               │
  │   │  │ :3000 (Node)   │  │                               │
  │   │  │ :3001 (Node)   │  │                               │
  │   │  │ :3002 (Node)   │  │                               │
  │   │  └────────────────┘  │                               │
  │   └──────────────────────┘                               │
  │                                                          │
  │   PM2 manages Node.js process lifecycle                  │
  │   Helper scripts: start-instance.sh, kill-instance.sh    │
  └──────────────────────────────────────────────────────────┘
```

### Nginx Configuration Layout (Before Deletion)

```
  /etc/nginx/
  ├── nginx.conf                           ← root config
  ├── conf.d/
  │   ├── test.conf                        ← test.vishvam.dev vhost
  │   └── dynamic.test.vishvam.dev.conf    ← dynamic subdomain vhost
  └── upstreams/
      └── node_pool.conf                   ← upstream backend pool
```

### Verification Before the Disaster

```bash
# Nginx config was valid
nginx -t
# nginx: the configuration file /etc/nginx/nginx.conf syntax is ok

# Upstream config was in place
cat /etc/nginx/upstreams/node_pool.conf
# upstream node_pool {
#     server 127.0.0.1:3000;
#     server 127.0.0.1:3001;
#     server 127.0.0.1:3002;
# }

# PM2 was managing Node instances
pm2 list
# ┌────┬────────┬─────────┬───────┐
# │ id │ name   │ status  │ port  │
# ├────┼────────┼─────────┼───────┤
# │ 0  │ app    │ online  │ 3000  │
# │ 1  │ app    │ online  │ 3001  │
# │ 2  │ app    │ online  │ 3002  │
# └────┴────────┴─────────┴───────┘
```

---

## 4. Phase 1 — The Configuration Loss

### The Destructive Commands

Critical Nginx files were deleted manually during the lab:

```bash
rm -rf /etc/nginx/upstreams/
```

This removed the upstream definition. Additional config files were also lost (site configs, possibly the root `nginx.conf`).

### Immediate Cascade

```
  ┌──────────────────────────────────────────────────────────────┐
  │  FAILURE CASCADE                                             │
  │                                                              │
  │  1. rm -rf /etc/nginx/upstreams/                             │
  │     └──► upstream node_pool block disappears                 │
  │                                                              │
  │  2. nginx -t → fails (references missing upstream)           │
  │     └──► systemctl restart nginx → fails                     │
  │                                                              │
  │  3. Nginx stops serving traffic                              │
  │     └──► all virtual hosts become unreachable                │
  │                                                              │
  │  4. Backend Node apps still running (PM2)                    │
  │     └──► but no reverse proxy to reach them                  │
  └──────────────────────────────────────────────────────────────┘
```

### Verification

```bash
systemctl restart nginx
# Job for nginx.service failed

systemctl status nginx
# nginx.service - The nginx HTTP and reverse proxy server
#    Active: failed (Result: exit-code)

ss -tulnp | grep 80
# (empty — nothing listening on port 80)
```

---

## 5. Phase 2 — Why Standard Recovery Methods Failed

### Method 1 — `nginx -T` (Memory Dump)

The first instinct when Nginx config is lost is to dump the running configuration from memory:

```bash
nginx -T
```

This works **only if Nginx is still running**. The `-T` flag reads the master process's loaded configuration from memory and prints it to stdout.

```
  ┌──────────────────────────────────────────────────────────┐
  │  nginx -T RECOVERY FLOW                                  │
  │                                                          │
  │  Nginx running?                                          │
  │     │                                                    │
  │     ├── YES → nginx -T dumps full config → save to file  │
  │     │         (includes all includes, upstreams, vhosts) │
  │     │                                                    │
  │     └── NO  → CANNOT USE THIS METHOD ❌                  │
  │               Process is gone, memory is freed           │
  └──────────────────────────────────────────────────────────┘
```

In this lab, Nginx had already **crashed and stopped** because the upstream file it referenced no longer existed. The process exited, the memory was freed.

### Method 2 — `/proc` File Descriptor Recovery

If a process still has a deleted file open, the file descriptor remains in `/proc/<pid>/fd/`. This is a well-known Linux trick:

```bash
lsof | grep nginx.conf
# If Nginx were running, you'd see:
# nginx  1234  root  5r  REG  /etc/nginx/nginx.conf (deleted)
# Then: cat /proc/1234/fd/5 > /tmp/nginx.conf.recovered
```

This also failed — no Nginx process existed.

```bash
ps -ax | grep nginx
# (no results)

lsof | grep -i nginx.conf
# (no results)
```

### Method 3 — Find Backup / Swap Files

Editors sometimes leave behind backup copies:

```bash
find / -name "*nginx.conf~"
# (no results)

find / -name ".nginx.conf.swp"
# (no results)
```

No editor backups existed.

### Method 4 — Package Query (Config Location Only)

On RPM-based systems, you can list which files a package owns:

```bash
rpm -ql nginx
```

This tells you **what files should exist** and their paths — useful for rebuilding the directory structure — but it does **not** recover the custom content of those files.

> **Result:** All standard recovery methods exhausted. The configs existed only as deleted blocks on the ext4 filesystem. **Forensic file carving** was the only remaining option.

---

## 6. Phase 3 — Filesystem Identification & Damage Containment

### Identify the Filesystem

```bash
df -T /etc
```

```
Filesystem     Type  Size  Used Avail Use% Mounted on
/dev/vda1      ext4   25G  5.2G  19G  22% /
```

This matters because recovery techniques differ by filesystem:

| Filesystem | Deletion Behavior                                           | Carving Viability           |
| ---------- | ----------------------------------------------------------- | --------------------------- |
| **ext4**   | Inodes zeroed immediately, data blocks remain until overwritten | Good — if acted on quickly |
| **XFS**    | Similar to ext4, but inode recovery is harder                | Moderate                   |
| **BTRFS**  | Copy-on-write — old data may persist in snapshots            | Use snapshots instead      |
| **ZFS**    | Copy-on-write — snapshots are the correct recovery path      | Use `zfs rollback`         |

### Why ext4 Deletion Is Not Instant Data Loss

```
  ┌──────────────────────────────────────────────────────────────┐
  │  HOW ext4 DELETION WORKS                                     │
  │                                                              │
  │  Before rm:                                                  │
  │  ┌────────┐     ┌──────────────┐     ┌───────────────┐       │
  │  │ dentry │────►│ inode #12345 │────►│ data blocks   │       │
  │  │ "node_ │     │ size: 128B   │     │ upstream {..} │       │
  │  │ pool.  │     │ links: 1     │     │               │       │
  │  │ conf"  │     │ blocks: 1    │     │               │       │
  │  └────────┘     └──────────────┘     └───────────────┘       │
  │                                                              │
  │  After rm:                                                   │
  │  ┌────────┐     ┌──────────────┐     ┌───────────────┐       │
  │  │ dentry │     │ inode #12345 │     │ data blocks   │       │
  │  │ REMOVED│     │ ZEROED       │     │ STILL ON DISK │       │
  │  │        │     │ links: 0     │     │ (until over-  │       │
  │  │        │     │              │     │  written)     │       │
  │  └────────┘     └──────────────┘     └───────────────┘       │
  │                                                              │
  │  The directory entry and inode are gone, but the actual      │
  │  data blocks remain on disk until the filesystem reuses      │
  │  them for new writes.                                        │
  └──────────────────────────────────────────────────────────────┘
```

### Prevent Further Writes — Remount Read-Only

**This is the single most important step after accidental deletion.** Every new write to the filesystem risks overwriting the recoverable data blocks.

```bash
mount -o remount,ro /
```

> **Gotcha:** On a live system, remounting `/` read-only may fail if processes are actively writing (journald, syslog, etc.). In that case:
> * Stop non-essential services first
> * Or boot from a live/rescue environment and mount the disk read-only from there
> * At minimum, avoid installing software or writing files to the same partition

---

## 7. Phase 4 — Installing Forensic Tools on CentOS/RHEL

### The Irony

You need to **install recovery tools**, but installing software **writes to the filesystem** you are trying to recover from.

```
  ⚠️  CATCH-22: Installing PhotoRec writes new data to the disk,
     potentially overwriting the very blocks you need to recover.
```

### Mitigation Strategies

| Strategy                                     | Risk Level |
| -------------------------------------------- | ---------- |
| Install to a different partition/disk         | ✅ Safest   |
| Boot from rescue ISO and install there        | ✅ Safe     |
| Install on the same partition (this lab)      | ⚠️ Risky — some data may be lost |
| Pre-install forensic tools (before disaster)  | ✅ Best — no risk at recovery time |

In this lab, tools were installed on the live system (same partition). This is not ideal but was the only option available at the time.

### Enabling EPEL and CRB Repositories

TestDisk (which includes PhotoRec) is not in CentOS base repos. It requires **EPEL** (Extra Packages for Enterprise Linux) and **CRB** (CodeReady Builder):

```bash
# Enable CRB (CentOS/Rocky/Alma)
dnf config-manager --set-enabled crb

# Install EPEL
dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm

# Install TestDisk (includes PhotoRec)
dnf install testdisk
```

### What Are These Tools?

| Tool       | Purpose                                                           |
| ---------- | ----------------------------------------------------------------- |
| **TestDisk** | Partition recovery, boot sector repair, filesystem analysis     |
| **PhotoRec** | File carving — recovers files by scanning raw disk blocks for known file signatures, independent of filesystem metadata |

> **Why "PhotoRec"?** Originally built for photo recovery, but it recognizes 480+ file formats including plain text, config files, and scripts.

---

## 8. Phase 5 — Running PhotoRec (File Carving)

### How File Carving Works

Unlike filesystem-aware tools (like `extundelete`), PhotoRec does **not** rely on inodes, directory entries, or filesystem metadata. It scans raw disk blocks looking for known file **signatures** (magic bytes).

```
  ┌──────────────────────────────────────────────────────────────┐
  │  FILE CARVING vs FILESYSTEM RECOVERY                         │
  │                                                              │
  │  Filesystem Recovery (extundelete):                          │
  │  "Find inode 12345 → read its block pointers → get data"     │
  │  ⚠️ Fails if inode is zeroed (ext4 does this on delete)      │
  │                                                              │
  │  File Carving (PhotoRec):                                    │
  │  "Scan every block on disk → if block looks like a text      │
  │   file / config file / image → extract it"                   │
  │  ✅ Works even if inodes are destroyed                       │
  │  ❌ Loses filenames and directory structure                  │
  └──────────────────────────────────────────────────────────────┘
```

### Running PhotoRec

```bash
# Create a recovery output directory (ideally on a DIFFERENT disk)
mkdir /root/dump

# Launch PhotoRec
photorec
```

PhotoRec interactive steps:

```
  ┌──────────────────────────────────────────────────────────┐
  │  PhotoRec WORKFLOW                                       │
  │                                                          │
  │  1. Select disk        → /dev/vda                        │
  │  2. Select partition   → /dev/vda1 (ext4)                │
  │  3. Select filesystem  → ext2/ext3/ext4                  │
  │  4. Select scan scope  → "Whole" (or "Free space only")  │
  │  5. Select output dir  → /root/dump                      │
  │  6. Start scan         → wait for completion             │
  │                                                          │
  │  Output:                                                 │
  │  /root/dump/                                             │
  │  ├── recup_dir.1/                                        │
  │  │   ├── f00001234.txt                                   │
  │  │   ├── f00001235.txt                                   │
  │  │   └── ...                                             │
  │  ├── recup_dir.2/                                        │
  │  │   └── ...                                             │
  │  ├── recup_dir.3/                                        │
  │  │   └── ...                                             │
  │  └── (hundreds of recup_dir.* folders)                   │
  └──────────────────────────────────────────────────────────┘
```

### What You Get

| What PhotoRec Recovers | What PhotoRec Loses         |
| ---------------------- | --------------------------- |
| File content           | Original filename           |
| File type (detected)   | Original directory path     |
| Multiple fragments     | File permissions/ownership  |
|                        | Relationship between files  |

Files are named generically: `f12345678.txt`, `f29049232.txt`, etc.

**You now have thousands of recovered text files and need to find your Nginx configs among them.**

---

## 9. Phase 6 — Filtering Recovered Files

This is where the lab became a **forensic investigation**. Thousands of recovered files, no filenames, no directory context — just raw content.

### Strategy 1 — Search for Known Domain Names

If you know what domains were configured, search for them:

```bash
grep -rl "vishvam.dev" /root/dump/
```

This immediately narrows candidates to files containing your specific domain strings.

### Strategy 2 — Search for Nginx Directives

Nginx configs have distinctive syntax. Search for it:

```bash
grep -rlE "server_name|proxy_pass|listen" /root/dump/
```

More specific:

```bash
grep -rlE "server_name .*;|proxy_pass .*;|listen [0-9]+;" /root/dump/
```

### Strategy 3 — Search for Upstream Definitions

The critical missing piece was the upstream pool:

```bash
grep -rl "upstream node_pool" /root/dump/
```

### Strategy 4 — Exclude Known False Positives

Initial runs found matches in log files and other large recovered files. Filter them out:

```bash
find /root/dump/ -type f \
  -not -name "*.log" \
  -size -50k \
  -exec grep -lE "server_name|listen|worker_processes" {} +
```

This restricts results to small text files (configs are typically <50KB) and excludes logs.

### Strategy 5 — Inspect Candidates

For each candidate file, inspect the content:

```bash
head -n 20 ./recup_dir.175/f29049232.txt
head -n 20 ./recup_dir.176/f29122904.txt
head -n 20 ./recup_dir.354/f110863848.txt
```

> **Gotcha:** Some recovered files contain **concatenated fragments** — multiple config files merged into one block, or config mixed with unrelated text. You need to visually inspect and extract the relevant sections.

---

## 10. Phase 7 — Extracting Configuration Blocks

### Aggregate Candidates into a Dump File

Combine all matching content into a single working file:

```bash
grep -rl "vishvam.dev;" /root/dump/ | xargs cat > conf_dump
```

### Extract Server Blocks

Use `sed` to pull complete `server { ... }` blocks:

```bash
sed -n '/server {/,/}/p' conf_dump | head -n 30
```

### Extract Upstream Blocks

```bash
awk '/upstream node_pool \{/,/\}/' conf_dump | sort | uniq
```

### Extract Key Directives (Overview)

Get a quick summary of all proxy targets, ports, and root paths:

```bash
grep -E "proxy_pass|root|listen|upstream" conf_dump | sort | uniq
```

### Deduplicate Full Server Blocks

PhotoRec may recover the same data multiple times (from journal copies, old filesystem snapshots, etc.):

```bash
awk '/server \{/{flag=1; block=""} flag{block = block $0 "\n"} /\}/{flag=0; print block}' conf_dump \
  | awk 'BEGIN{RS=""; ORS="\n\n"} !seen[$0]++'
```

This prints each unique server block exactly once.

> **What you are doing here** is essentially reverse-engineering your own configuration from forensic fragments. This is why configuration management (Git, Ansible) exists — so you never have to do this.

---

## 11. Phase 8 — Rebuilding Nginx Directory Structure & Configs

### Recreate Directory Structure

```bash
mkdir -p /etc/nginx/conf.d
mkdir -p /etc/nginx/upstreams
```

### Restore Upstream Configuration

Based on extracted fragments, recreate the upstream pool:

```bash
cat > /etc/nginx/upstreams/node_pool.conf << 'EOF'
upstream node_pool {
    server 127.0.0.1:3000;
    server 127.0.0.1:3001;
    server 127.0.0.1:3002;
}
EOF
```

> **Gotcha in the lab:** Initially only one backend was written (`server 127.0.0.1:3000;`) using a quick `echo` command. This worked functionally but did not match the original multi-backend pool. Always cross-reference recovered fragments carefully.

### Restore Virtual Host — `dynamic.test.vishvam.dev`

```bash
cat > /etc/nginx/conf.d/dynamic.test.vishvam.dev.conf << 'EOF'
server {
    listen 80;
    server_name dynamic.test.vishvam.dev;

    location / {
        proxy_pass http://node_pool;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
EOF
```

### Restore Virtual Host — `test.vishvam.dev`

```bash
cat > /etc/nginx/conf.d/test.conf << 'EOF'
server {
    listen 80;
    server_name test.vishvam.dev;

    location / {
        proxy_pass http://node_pool;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
EOF
```

### Fix Permissions

```bash
chmod 755 /etc/nginx/conf.d
chmod 755 /etc/nginx/upstreams
chmod 644 /etc/nginx/conf.d/*.conf
chmod 644 /etc/nginx/upstreams/*.conf
```

---

## 12. Phase 9 — Restoring Root Configuration (`nginx.conf`)

### The Problem

If `nginx.conf` itself was lost or corrupted, custom site configs alone are not enough — Nginx has no entry point.

### Recovery via Package Reinstall

On RHEL/CentOS, the root config is owned by the `nginx-core` package. Reinstalling it restores the default `nginx.conf`:

```bash
dnf reinstall nginx-core -y
```

This restores:

| File Restored          | Content                         |
| ---------------------- | ------------------------------- |
| `/etc/nginx/nginx.conf` | Default root configuration     |
| `/etc/nginx/mime.types`  | MIME type mappings             |
| `/etc/nginx/fastcgi_params` | FastCGI parameter defaults |
| `/etc/nginx/scgi_params`    | SCGI parameter defaults    |

### Verify the Include Directives

The default `nginx.conf` must include your custom config directories. Verify:

```bash
grep -E "include|conf\.d" /etc/nginx/nginx.conf
```

Expected (on CentOS/RHEL):

```nginx
include /etc/nginx/conf.d/*.conf;
```

If your upstream directory is **not** included by default, add the include:

```nginx
# Inside the http { } block of nginx.conf:
include /etc/nginx/upstreams/*.conf;
```

> **Gotcha observed in the lab:** The default `nginx.conf` on CentOS/RHEL includes `conf.d/*.conf` but does **not** include a custom `upstreams/` directory. You must add this include manually, or move your upstream definition into `conf.d/`.

### CentOS/RHEL vs Ubuntu — Nginx Config Differences

| Aspect                | CentOS/RHEL                         | Ubuntu/Debian                             |
| --------------------- | ----------------------------------- | ----------------------------------------- |
| Root config           | `/etc/nginx/nginx.conf`             | `/etc/nginx/nginx.conf`                   |
| Site configs          | `/etc/nginx/conf.d/*.conf`          | `/etc/nginx/sites-enabled/*` (symlinks)   |
| Available/enabled     | No split — all configs in `conf.d`  | `sites-available/` + `sites-enabled/`     |
| Default page config   | `conf.d/default.conf`               | `sites-available/default`                 |
| Package reinstall     | `dnf reinstall nginx-core`          | `apt install --reinstall nginx-core`      |
| Force config restore  | `dnf reinstall` (usually sufficient)| Needs `--force-confmiss` dpkg option      |

---

## 13. Phase 10 — Validation & Service Recovery

### Test Configuration Syntax

```bash
nginx -t
```

Expected:

```
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

### Common Failures at This Stage

| Error                                          | Cause                                    | Fix                                       |
| ---------------------------------------------- | ---------------------------------------- | ----------------------------------------- |
| `unknown directive "upstream"`                 | Upstream block outside `http { }`        | Move upstream into `http { }` or included file |
| `host not found in upstream "node_pool"`       | Upstream include missing from `nginx.conf` | Add `include upstreams/*.conf;`         |
| `duplicate upstream "node_pool"`               | Same upstream defined in multiple files  | Remove duplicate                          |
| `bind() to 0.0.0.0:80 failed`                 | Port 80 already in use                   | Check `ss -tulnp \| grep :80`            |
| `[emerg] "server" directive is not allowed here` | Server block outside `http { }`       | Check nesting in `nginx.conf`             |

### Handling Incomplete Configs

During the lab, partially recovered configs caused errors. The strategy was iterative:

```bash
# Test → find error → fix → test again
nginx -t                                  # find the error
nano /etc/nginx/conf.d/problem.conf       # fix or remove
mv conf.d/node-service.conf conf.d/node-service.conf.down  # disable problematic config
nginx -t                                  # test again
```

> **Pro tip:** Renaming a config to `.conf.down` (or any non-`.conf` extension) removes it from the `include *.conf` glob without deleting it. Useful for isolating problems.

### Start Nginx

```bash
systemctl start nginx
systemctl enable nginx
```

### Verify

```bash
systemctl status nginx
# → active (running)

ss -tulnp | grep :80
# → 0.0.0.0:80   LISTEN   nginx: master process
```

---

## 14. Phase 11 — Restoring the Node.js Backend

### Check PM2 Status

```bash
pm2 list
```

If Node processes were stopped or killed during the incident, restart them:

```bash
pm2 start all
# or individually:
pm2 start 0
pm2 start 1
pm2 start 2
```

### Verify Backend Ports

```bash
ss -tulnp | grep -E "3000|3001|3002"
```

Expected — Node apps listening on backend ports.

### End-to-End Verification

```bash
# Direct backend test
curl -s http://127.0.0.1:3000/ | head -5

# Through Nginx reverse proxy
curl -s -H "Host: test.vishvam.dev" http://127.0.0.1/ | head -5

# Full external test (if DNS is configured)
curl -s http://test.vishvam.dev/ | head -5
```

---

## 15. Final System State

### What Was Recovered

| Component                         | Status       | Recovery Method                    |
| --------------------------------- | ------------ | ---------------------------------- |
| `nginx.conf` (root config)        | ✅ Restored   | `dnf reinstall nginx-core`        |
| `upstreams/node_pool.conf`        | ✅ Rebuilt     | PhotoRec + grep forensics        |
| `conf.d/test.conf`                | ✅ Rebuilt     | PhotoRec + manual reconstruction |
| `conf.d/dynamic.test.vishvam.dev.conf` | ✅ Rebuilt | PhotoRec + manual reconstruction|
| Nginx service                     | ✅ Running    | `systemctl start nginx`           |
| PM2 / Node.js backends            | ✅ Running    | `pm2 start all`                   |
| Port 80 serving traffic           | ✅ Confirmed  | `ss -tulnp` + `curl`              |

### What May Still Be Imperfect

| Component                    | Risk                                              |
| ---------------------------- | ------------------------------------------------- |
| Exact upstream pool members  | May have missed backends if fragments were incomplete |
| Proxy headers                | Default headers may differ from original tuning    |
| Rate limiting / caching      | Custom Nginx tuning likely lost                   |
| SSL/TLS config               | Certificates and HTTPS blocks likely lost          |
| Logging customization        | Custom log formats / paths likely lost             |
| Security headers             | CSP, HSTS, X-Frame-Options likely lost             |

---

## 16. Gotchas & Lessons Learned

### 16.1 The PhotoRec Paradox — Installing the Recovery Tool

Installing PhotoRec writes data to the same disk you are recovering from. Every block written is a block that might overwrite your deleted config.

**Best practices:**

| Option                              | Implementation                                             |
| ----------------------------------- | ---------------------------------------------------------- |
| Pre-install forensic tools          | `dnf install testdisk` on every server as part of baseline |
| Boot from rescue/live ISO           | Mount target disk read-only, install tools on rescue env   |
| Recover to external storage         | Point PhotoRec output to a different disk/NFS mount        |
| Use a binary-only install           | Download a static PhotoRec binary — no package manager needed |

### 16.2 Remounting Read-Only May Fail on Live Systems

```bash
mount -o remount,ro /
# mount: / is busy
```

Active processes writing to disk (journald, syslog, cron) prevent read-only remount.

**Workaround:**

```bash
# Stop heavy writers first
systemctl stop rsyslog
systemctl stop systemd-journald

# Try again
mount -o remount,ro /

# Or: accept the risk and proceed without remount
# (less safe, but sometimes the only option on a live system)
```

### 16.3 PhotoRec Recovers EVERYTHING

PhotoRec does not know what you want. It recovers **every recoverable file** on the disk — potentially tens of thousands:

* old log fragments
* cached web pages
* temporary files
* previous versions of configs
* deleted user data

Effective `grep` filtering is essential. Without it, you are searching for a needle in a haystack.

### 16.4 Multiple Versions of the Same Config

PhotoRec may recover the **same file multiple times** — from different points in the filesystem journal or from blocks that were reused. You may find:

* a current version of `nginx.conf`
* an older version with different settings
* a partial fragment

Always compare recovered fragments to identify the most complete and recent version.

### 16.5 The `conf.d/` Include Gotcha (CentOS/RHEL)

CentOS/RHEL Nginx's default `nginx.conf` includes:

```nginx
include /etc/nginx/conf.d/*.conf;
```

But a custom `upstreams/` directory is **not** included by default. If you recreate `upstreams/node_pool.conf` but forget to add the include directive to `nginx.conf`, Nginx will fail with:

```
host not found in upstream "node_pool"
```

### 16.6 The `nginx -T` Window Is Short

Once Nginx stops or is restarted after config deletion, the memory dump opportunity is **gone forever**. The lesson:

```
  ┌──────────────────────────────────────────────────────────────┐
  │  IF YOU ACCIDENTALLY DELETE NGINX CONFIG:                    │
  │                                                              │
  │  1. DO NOT restart Nginx                                     │
  │  2. DO NOT run systemctl reload nginx                        │
  │  3. IMMEDIATELY run: nginx -T > /tmp/nginx_backup.conf       │
  │  4. Then fix the filesystem                                  │
  │                                                              │
  │  If Nginx is still running with the OLD config in memory,    │
  │  -T will dump everything you need.                           │
  └──────────────────────────────────────────────────────────────┘
```

### 16.7 `echo` vs Proper Heredoc for Config Files

During the lab, a quick `echo` was used to write the upstream file:

```bash
echo "server 127.0.0.1:3000;" > /etc/nginx/upstreams/node_pool.conf
```

This is **incorrect** — it writes a bare directive without the `upstream { }` wrapper. Nginx treats it as a stray directive and fails. Always use heredocs or a text editor for multi-line configs:

```bash
cat > /etc/nginx/upstreams/node_pool.conf << 'EOF'
upstream node_pool {
    server 127.0.0.1:3000;
    server 127.0.0.1:3001;
    server 127.0.0.1:3002;
}
EOF
```

---

## 17. Production-Grade Prevention & Recovery

Forensic file carving is a **last resort**. Here is what prevents this scenario entirely.

### 17.1 Version Control for Nginx Configs

```bash
cd /etc/nginx
git init
git add -A
git commit -m "baseline config"
```

Or use `etckeeper` for automatic tracking:

```bash
# Debian/Ubuntu
apt install etckeeper

# CentOS/RHEL
dnf install etckeeper

etckeeper init
etckeeper commit "initial"
```

Every change to `/etc` (including `dnf install` operations) is auto-committed.

**Recovery:**

```bash
cd /etc
git log --oneline
git checkout <last-good-commit> -- nginx/
```

### 17.2 Automated Config Backups

```bash
# Daily Nginx config backup — add to crontab
0 3 * * * tar czf /backup/nginx-$(date +\%F).tar.gz /etc/nginx

# With retention (keep 30 days)
find /backup/ -name "nginx-*.tar.gz" -mtime +30 -delete
```

### 17.3 Infrastructure as Code

In mature environments, Nginx configs are **generated** from templates:

```
  ┌──────────────────────────────────────────────────────────────┐
  │  IaC WORKFLOW                                                │
  │                                                              │
  │  Ansible Playbook (Git)                                      │
  │       │                                                      │
  │       ▼                                                      │
  │  Template: nginx.conf.j2                                     │
  │       │    upstream {{ pool_name }} {                        │
  │       │      {% for backend in backends %}                   │
  │       │      server {{ backend.host }}:{{ backend.port }};   │
  │       │      {% endfor %}                                    │
  │       │    }                                                 │
  │       │                                                      │
  │       ▼                                                      │
  │  Generated: /etc/nginx/upstreams/node_pool.conf              │
  │                                                              │
  │  Recovery: ansible-playbook nginx.yml  ← takes 30 seconds    │
  └──────────────────────────────────────────────────────────────┘
```

### 17.4 `nginx -T` as a Pre-Change Backup

Before any config change, dump the running configuration:

```bash
nginx -T > /root/nginx-backup-$(date +%F-%H%M).conf
```

This captures **everything** — all includes resolved, all upstreams, all server blocks — in a single file.

### 17.5 Immutable Infrastructure

The modern DevOps approach:

```
  ┌──────────────────────────────────────────────┐
  │  Don't repair servers. Replace them.         │
  │                                              │
  │  Server breaks → destroy it                  │
  │  Terraform provisions new instance           │
  │  Ansible applies config                      │
  │  DNS or load balancer switches traffic       │
  │                                              │
  │  Total recovery time: minutes, not hours.    │
  └──────────────────────────────────────────────┘
```

### Recovery Strategy Hierarchy

```
  ┌──────────────────────────────────────────────────────────────┐
  │  RECOVERY PREFERENCE ORDER (fastest → slowest)               │
  │                                                              │
  │  3. Restore from backup tar/borg/restic             ← mins   │
  │  4. Re-run Ansible/Terraform (IaC)                  ← mins   │
  │  5. BTRFS/ZFS snapshot rollback                     ← mins   │
  │  6. Cloud provider snapshot restore                 ← mins   │
  │  7. dnf reinstall + manual config (known configs)   ← hour   │
  │  8. PhotoRec file carving + forensic grep           ← hours  │
  │  9. Rewrite from memory / documentation             ← hours  │
  │                                                              │
  │  ⚠️  This lab used option 8. Options 1–6 are preferred.      │
  └──────────────────────────────────────────────────────────────┘
```

---

## 18. Variables That Change Everything

### 18.1 Filesystem Type

| Filesystem | Impact on File Carving                                            |
| ---------- | ----------------------------------------------------------------- |
| **ext4**   | Inodes zeroed on delete, but data blocks persist. PhotoRec effective if acted quickly. Journal may hold extra copies. |
| **XFS**    | Similar to ext4, but XFS is more aggressive with block reuse. Recovery window is shorter. |
| **BTRFS**  | Copy-on-write means old data persists naturally. **Use `btrfs subvolume snapshot` instead of carving.** |
| **ZFS**    | Same COW principle. **Use `zfs rollback` — no carving needed.** |
| **tmpfs**  | RAM-based. Once deleted, **gone forever**. No carving possible. |

### 18.2 Time Since Deletion

The single biggest factor in ext4 carving success:

| Time Since `rm`    | Estimated Recovery Rate                   |
| ------------------ | ----------------------------------------- |
| Seconds            | ~95%+ (blocks untouched)                  |
| Minutes            | ~80-90% (journal activity may overwrite)  |
| Hours (low I/O)    | ~50-80% (depends on disk activity)        |
| Hours (high I/O)   | ~10-50% (significant block reuse)         |
| Days               | Unreliable — depends entirely on disk use |

**Act fast. Every write reduces recovery chances.**

### 18.3 OS Distribution — Package Management & Paths

| Distro          | Nginx Package    | Config Path              | Reinstall Command                      |
| --------------- | ---------------- | ------------------------ | -------------------------------------- |
| CentOS/RHEL 9   | `nginx-core`    | `/etc/nginx/`            | `dnf reinstall nginx-core`             |
| Ubuntu/Debian   | `nginx-core`    | `/etc/nginx/`            | `apt install --reinstall nginx-core` + `--force-confmiss` |
| Arch Linux      | `nginx`         | `/etc/nginx/`            | `pacman -S nginx`                      |
| Alpine          | `nginx`         | `/etc/nginx/`            | `apk fix nginx`                        |

### 18.4 Nginx vs Other Services

The same PhotoRec + grep approach works for recovering any text-based config:

| Service    | Config Path         | Unique Search Strings                      |
| ---------- | ------------------- | ------------------------------------------ |
| Nginx      | `/etc/nginx/`       | `server_name`, `proxy_pass`, `upstream`    |
| Apache     | `/etc/httpd/`       | `VirtualHost`, `DocumentRoot`, `ProxyPass` |
| HAProxy    | `/etc/haproxy/`     | `frontend`, `backend`, `bind`              |
| Postfix    | `/etc/postfix/`     | `myhostname`, `mydomain`, `relay_host`     |
| MySQL      | `/etc/my.cnf`       | `[mysqld]`, `innodb_buffer_pool_size`      |

---

## 19. Best Practices — Mapping Lab Lessons to Production

### Prevention

| Lab Lesson                               | Production Practice                                             |
| ---------------------------------------- | --------------------------------------------------------------- |
| Deleted configs = total service loss     | **Version-control all configs** (Git / etckeeper)                |
| No backup existed                        | **Automated daily config backups** to remote storage             |
| Service stopped before `-T` could be used | **Pre-change `nginx -T` dump** as standard operating procedure |
| Recovery took hours of grep forensics    | **IaC templates** so configs are regenerated in minutes          |

### Detection

| Lab Lesson                               | Production Practice                                             |
| ---------------------------------------- | --------------------------------------------------------------- |
| Nobody knew configs were deleted         | **File integrity monitoring** (AIDE, Tripwire, OSSEC)           |
| Service failure was the first symptom    | **Health checks** (Prometheus, Nagios, Datadog)                  |
| No audit trail of `rm` command           | **auditd rules** on `/etc/nginx/` directory                     |

### Recovery

| Lab Lesson                               | Production Practice                                             |
| ---------------------------------------- | --------------------------------------------------------------- |
| PhotoRec is chaotic and time-consuming   | **Never rely on forensic carving as a DR plan**                  |
| `dnf reinstall` restores defaults only   | **Custom configs must come from backup or IaC**                  |
| The write-after-delete risk              | **Pre-install forensic tools** on all servers as baseline        |
| Manual grep requires knowing what to look for | **Maintain a config inventory** (what files, what content) |

### Minimum Viable Protection for Any Nginx Server

```bash
# 1. Version control (one-time setup)
cd /etc/nginx && git init && git add -A && git commit -m "baseline"

# 2. Pre-change safety habit
nginx -T > /root/nginx-backup-$(date +%F-%H%M).conf

# 3. Automated daily backup (add to crontab)
echo "0 3 * * * root tar czf /backup/nginx-\$(date +\%F).tar.gz /etc/nginx" \
  >> /etc/cron.d/nginx-backup

# 4. Pre-install forensic tools (in case of emergency)
dnf install testdisk   # CentOS/RHEL
apt install testdisk   # Ubuntu/Debian
```

---

## 20. Suggested Extensions

| Exercise                                           | Skill Practiced                              |
| -------------------------------------------------- | -------------------------------------------- |
| Recover configs using `extundelete` instead         | Filesystem-level (inode-based) recovery      |
| Reconstruct TLS certificates after deletion         | Certificate management, Let's Encrypt reissue |
| Set up `etckeeper` and test git-based rollback      | Automated version control for `/etc`         |
| Perform same lab on BTRFS and use snapshot rollback | COW filesystem recovery                      |
| Automate Nginx config deployment with Ansible       | IaC for web server management                |
| Set up AIDE and trigger on `/etc/nginx/` change     | File integrity monitoring                    |
| Write a script that runs `nginx -T` before every `systemctl reload` | Defensive automation |
| Recover a deleted MySQL `my.cnf` using PhotoRec     | Cross-service forensic recovery              |
| Set up HAProxy and recover its config forensically  | Different config syntax, same technique      |

---

## 21. Final Thoughts

This lab demonstrates a harsh operational reality:

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                          │
  │   Configuration is often more valuable than the          │
  │   application itself.                                    │
  │                                                          │
  │   The Node.js app survived.                              │
  │   The Nginx binaries survived.                           │
  │   The OS survived.                                       │
  │                                                          │
  │   But without configuration files, the entire stack      │
  │   was unreachable — a working engine with no wiring.     │
  │                                                          │
  └──────────────────────────────────────────────────────────┘
```

PhotoRec and forensic grep are powerful tools — they saved this lab. But the hours spent scanning raw disk blocks, filtering thousands of recovered files, extracting config fragments, and manually reconstructing server blocks could have been replaced by a single command:

```bash
# With etckeeper:
cd /etc && git checkout HEAD -- nginx/

# With Ansible:
ansible-playbook nginx.yml

# With backup:
tar xzf /backup/nginx-2026-03-05.tar.gz -C /
```

> **The DevOps lesson:** The best disaster recovery is the one you never have to perform manually. Invest in prevention — version control, automated backups, and Infrastructure as Code — so that recovery is a **30-second operation**, not a **3-hour forensic investigation**.

---

> **Lab performed by:** Vishvam Moliya  
> **Date:** March 2026  
> **Environment:** DigitalOcean CentOS/RHEL 9 — Nginx + Node.js (PM2)  
> **Status:** Educational — not a production runbook

---

[🏠 Back to Home](../README.md) · [Labs](README.md)
