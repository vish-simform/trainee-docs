[🏠 Home](../README.md) · [Labs](README.md)

# 🔥 Linux Disaster Recovery Lab — Rebuilding a System After `/etc` Deletion

> **Audience:** DevOps / Cloud Engineers & Trainees | **Difficulty:** Advanced  
> **Time:** 2–4 hours | **Risk Level:** Destructive — lab/disposable environments only  
> **Tip:** This is a structured walkthrough of a real experiment. Follow phases sequentially. The gotchas are real.

---

## Table of Contents

1. [Lab Overview](#1-lab-overview)
2. [Lab Environment](#2-lab-environment)
3. [Understanding `/etc` — What You Are About to Destroy](#3-understanding-etc--what-you-are-about-to-destroy)
4. [Phase 1 — The Destructive Event](#4-phase-1--the-destructive-event)
5. [Phase 2 — Booting Into Recovery](#5-phase-2--booting-into-recovery)
6. [Phase 3 — The `/etc` Transplant](#6-phase-3--the-etc-transplant)
7. [Phase 4 — Restoring System Identity & Access](#7-phase-4--restoring-system-identity--access)
8. [Phase 5 — Fixing Package Repositories (Ubuntu 24.04 DEB822)](#8-phase-5--fixing-package-repositories-ubuntu-2404-deb822)
9. [Phase 6 — Forced Configuration Restoration](#9-phase-6--forced-configuration-restoration)
10. [Phase 7 — Rebuilding Service Users & Machine Identity](#10-phase-7--rebuilding-service-users--machine-identity)
11. [Phase 8 — Apache Recovery: The MPM Problem](#11-phase-8--apache-recovery-the-mpm-problem)
12. [Phase 9 — Network Port Verification & Final Validation](#12-phase-9--network-port-verification--final-validation)
13. [Post-Recovery State Assessment](#13-post-recovery-state-assessment)
14. [Gotchas & Lessons Learned](#14-gotchas--lessons-learned)
15. [Production-Grade Recovery Alternatives](#15-production-grade-recovery-alternatives)
16. [Variables That Change Everything](#16-variables-that-change-everything)
17. [Best Practices — Mapping Lab Lessons to Production](#17-best-practices--mapping-lab-lessons-to-production)
18. [Suggested Extensions](#18-suggested-extensions)
19. [Final Thoughts](#19-final-thoughts)

---

## 1. Lab Overview

This lab simulates a **catastrophic configuration loss** on a Linux server by intentionally deleting `/etc`, then manually reconstructing the system from a recovery environment.

### What This Lab Teaches

```
  ┌────────────────────────────────────────────────────────────────┐
  │                    SKILLS DEMONSTRATED                         │
  │                                                                │
  │  ┌──────────────────┐  ┌──────────────────┐  ┌─────────────┐   │
  │  │ Boot & Recovery  │  │ Package Manager  │  │ Service     │   │
  │  │ • Recovery ISO   │  │ • apt internals  │  │ Recovery    │   │
  │  │ • chroot         │  │ • DEB822 format  │  │ • Apache    │   │
  │  │ • fs mounting    │  │ • dpkg options   │  │ • SSH       │   │
  │  │ • GRUB concepts  │  │ • repo structure │  │ • systemd   │   │
  │  └──────────────────┘  └──────────────────┘  └─────────────┘   │
  │  ┌──────────────────┐  ┌──────────────────┐  ┌─────────────┐   │
  │  │ Identity & Auth  │  │ OS Internals     │  │ DR Strategy │   │
  │  │ • passwd/shadow  │  │ • machine-id     │  │ • snapshots │   │
  │  │ • permissions    │  │ • system users   │  │ • IaC       │   │
  │  │ • host keys      │  │ • symlink deps   │  │ • backups   │   │
  │  └──────────────────┘  └──────────────────┘  └─────────────┘   │
  └────────────────────────────────────────────────────────────────┘
```

### The Goal

The goal is **not simply to fix the server**. It is to understand **what actually breaks in Linux when `/etc` disappears**, and why every production system needs a strategy that makes manual recovery unnecessary.

---

## 2. Lab Environment

### Environment Used

| Component          | Value                     |
| ------------------ | ------------------------- |
| Provider           | DigitalOcean              |
| OS                 | Ubuntu 24.04 LTS (Noble)  |
| Access Method      | Recovery ISO + Console    |
| Filesystem         | ext4                      |
| Installed Services | Apache2, Nginx, OpenSSH   |
| Root Access        | Required                  |

### Alternative Local Setups

This lab is fully reproducible on:

| Platform        | Notes                                                   |
| --------------- | ------------------------------------------------------- |
| VirtualBox      | Take a snapshot before deletion — instant rollback       |
| VMware          | Snapshot support built-in                               |
| QEMU/KVM        | Use `virsh snapshot-create` before the experiment        |
| Multipass       | Quick Ubuntu VMs — `multipass launch noble`              |
| WSL2            | Partial — no systemd recovery, limited boot simulation   |

> **Recommendation:** Use a hypervisor with snapshot support. Take a snapshot **before** deletion so you can repeat the lab without reprovisioning.

---

## 3. Understanding `/etc` — What You Are About to Destroy

The `/etc` directory is the **configuration brain** of a Linux system. Every daemon, every identity file, every network definition lives here.

### Critical Contents

```
  /etc/
  ├── passwd              ← user accounts (UID/GID mapping)
  ├── shadow              ← password hashes (restricted)
  ├── group               ← group membership
  ├── gshadow             ← group password hashes
  ├── hostname            ← machine name
  ├── hosts               ← local DNS overrides
  ├── fstab               ← filesystem mount table
  ├── resolv.conf         ← DNS resolver config
  ├── machine-id          ← unique system identifier
  ├── ssh/
  │   ├── sshd_config     ← SSH daemon configuration
  │   └── ssh_host_*      ← host keypairs (fingerprint identity)
  ├── apache2/
  │   ├── apache2.conf    ← main Apache config
  │   ├── mods-enabled/   ← symlinks to active modules
  │   ├── sites-enabled/  ← symlinks to active vhosts
  │   └── envvars         ← runtime env variables
  ├── nginx/
  │   ├── nginx.conf      ← main Nginx config
  │   └── sites-enabled/  ← symlinks to active server blocks
  ├── apt/
  │   ├── sources.list    ← (legacy) repo definitions
  │   └── sources.list.d/
  │       └── ubuntu.sources  ← (DEB822) modern repo definitions
  ├── ssl/
  │   └── private/        ← TLS private keys
  └── systemd/            ← systemd overrides and drop-ins
```

### What Survives Deletion

Deleting `/etc` does **not** destroy:

| Survives                | Reason                                        |
| ----------------------- | --------------------------------------------- |
| Kernel                  | Lives in `/boot`                              |
| Installed binaries      | Live in `/usr/bin`, `/usr/sbin`               |
| Package database        | Lives in `/var/lib/dpkg` (Debian/Ubuntu)       |
| User home directories   | Live in `/home`                               |
| Application data        | Typically in `/var`, `/opt`, `/srv`            |
| Logs                    | Live in `/var/log`                             |

This is why the system **boots** but cannot **function**. The engine exists — it just has no wiring diagram.

---

## 4. Phase 1 — The Destructive Event

> ⚠️ **NEVER run this on a production system.**

```bash
rm -rf /etc/
```

### Immediate Effects

The moment `/etc` is gone, the running system still operates (processes are in memory), but:

| What Broke                 | Why                                           |
| -------------------------- | --------------------------------------------- |
| User database              | `/etc/passwd`, `/etc/shadow` gone              |
| Group definitions          | `/etc/group`, `/etc/gshadow` gone              |
| Hostname                   | `/etc/hostname` gone — resets to default       |
| DNS resolution             | `/etc/resolv.conf` gone                        |
| Package manager repos      | `/etc/apt/` tree gone                          |
| SSH daemon config          | `/etc/ssh/sshd_config` + host keys gone        |
| Apache config              | `/etc/apache2/` tree gone                      |
| Nginx config               | `/etc/nginx/` tree gone                        |
| Filesystem mount table     | `/etc/fstab` gone                              |
| System identity            | `/etc/machine-id` gone                         |
| TLS certificates           | `/etc/ssl/` gone                               |

### After Reboot

```bash
systemctl reboot
```

Post-reboot symptoms:

```
  ┌──────────────────────────────────────────────────────────────┐
  │  SYMPTOM                     │  ROOT CAUSE                   │
  ├──────────────────────────────┼───────────────────────────────┤
  │  SSH connection refused      │  sshd_config missing          │
  │  SSH fingerprint mismatch    │  host keys regenerated/gone   │
  │  Cannot login as root        │  passwd/shadow missing        │
  │  Apache won't start          │  apache2.conf missing         │
  │  Nginx won't start           │  nginx.conf missing           │
  │  hostname shows "localhost"  │  /etc/hostname missing        │
  │  apt update fails            │  sources config missing       │
  │  systemd journal errors      │  machine-id missing           │
  └──────────────────────────────┴───────────────────────────────┘
```

The system is now a **partially functional kernel with no configuration layer** — alive, but confused.

---

## 5. Phase 2 — Booting Into Recovery

Since SSH and local login are broken, recovery requires **out-of-band access**:

| Provider       | Recovery Method                              |
| -------------- | -------------------------------------------- |
| DigitalOcean   | Recovery ISO via Droplet Console             |
| AWS EC2        | Detach EBS → attach to rescue instance       |
| GCP            | Serial console + rescue disk                 |
| Azure          | Serial console + repair VM                   |
| Local VM       | Boot from Ubuntu Live ISO                    |

### DigitalOcean Recovery Workflow

```
  ┌─────────────────────────────────────────────────────────┐
  │  1. Droplet → Power Off                                 │
  │  2. Droplet → Recovery → Boot from Recovery ISO         │
  │  3. Access via Droplet Console (web-based VNC)          │
  │  4. Recovery ISO provides:                              │
  │     • Live Linux environment                            │
  │     • recovery.sh — DigitalOcean recovery helper        │
  │     • Options for mounting, chroot, key reset           │
  └─────────────────────────────────────────────────────────┘
```

The `recovery.sh` script on DigitalOcean provides interactive options:

| Option | Function                                |
| ------ | --------------------------------------- |
| 1      | Mount filesystem                        |
| 6      | chroot into mounted system              |
| 7      | Reset SSH host keys                     |

---

## 6. Phase 3 — The `/etc` Transplant

### Locate and Mount the Root Partition

```bash
lsblk
```

Typical output on a DigitalOcean droplet:

```
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
vda    254:0    0   25G  0 disk
└─vda1 254:1    0   25G  0 part
```

Mount it:

```bash
mount /dev/vda1 /mnt
```

Verify:

```bash
ls /mnt
# Expected: bin  boot  dev  home  lib  opt  proc  root  srv  tmp  usr  var
# Notice: NO /etc directory
```

### Copy `/etc` from Recovery ISO

Since `/etc` was completely destroyed, the fastest initial fix is transplanting the recovery environment's configuration:

```bash
cp -r /etc /mnt/
```

### What This Gives You

| ✅ Restored                         | ⚠️ Problem Introduced                            |
| ----------------------------------- | -----------------------------------------------  |
| Default `passwd`, `shadow`, `group` | Users belong to the **ISO**, not your server     |
| Package manager base config         | Repo URLs may be wrong or incomplete             |
| Default system configs              | Service-specific configs (Apache, etc.) missing  |
| A bootable `/etc` skeleton          | `machine-id` is the **ISO's identity**           |

This is a **Frankenstein transplant** — the system gains a pulse but has someone else's identity.

> **Why not reinstall the OS?** The purpose of this lab is to understand what breaks and how to fix it. In production, you would almost certainly redeploy. But understanding this process teaches you what every config file actually does.

---

## 7. Phase 4 — Restoring System Identity & Access

### Enter the System via chroot

Bind-mount kernel virtual filesystems, then chroot:

```bash
mount --bind /dev  /mnt/dev
mount --bind /proc /mnt/proc
mount --bind /sys  /mnt/sys

chroot /mnt
```

You are now operating **inside your broken system's filesystem** as if it were running.

> **What is chroot?** It changes the apparent root directory for the current process. Combined with bind mounts for `/dev`, `/proc`, and `/sys`, it gives you a near-complete system environment without actually booting it.

### Reset Root Password

The transplanted `/etc/shadow` contains the recovery ISO's root hash, not yours:

```bash
passwd root
```

### Verify Root Shell

Ensure `/etc/passwd` has a valid root entry:

```bash
grep ^root /etc/passwd
```

Expected:

```
root:x:0:0:root:/root:/bin/bash
```

If the shell is `/usr/sbin/nologin` or missing, fix it:

```bash
usermod -s /bin/bash root
```

### Fix Critical File Permissions

Linux enforces strict permissions on authentication files. Incorrect permissions = login failures.

```bash
chmod 644 /etc/passwd
chmod 640 /etc/shadow
chmod 644 /etc/group
chmod 644 /etc/hosts
```

**Why these specific permissions?**

| File           | Permission | Reason                                                   |
| -------------- | ---------- | -------------------------------------------------------- |
| `/etc/passwd`  | `644`      | Readable by all (programs resolve UIDs) — no hashes here |
| `/etc/shadow`  | `640`      | Only root + shadow group — contains password hashes      |
| `/etc/group`   | `644`      | Readable by all — group membership lookups               |
| `/etc/hosts`   | `644`      | Readable by all — local DNS resolution                   |

> **Gotcha:** If `/etc/shadow` is world-readable (`644`), `sshd` and `login` may refuse to authenticate. If it is `600` instead of `640`, the `shadow` group cannot read it — some PAM modules may fail. `640` with `root:shadow` ownership is the standard.

### Reset SSH Host Keys (DigitalOcean)

The transplanted host keys belong to the recovery ISO. SSH clients that previously connected to this server will reject it:

```
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
@    WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!     @
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
```

On DigitalOcean, use the recovery script:

```bash
# Exit chroot first
exit

# Run recovery helper
bash /root/recovery.sh
# Select Option 7: Reset SSH host keys
```

On other environments, regenerate manually inside chroot:

```bash
rm -f /etc/ssh/ssh_host_*
dpkg-reconfigure openssh-server
# Or manually:
ssh-keygen -A
```

> **Production note:** After host key regeneration, update your `~/.ssh/known_hosts` on the client side:
> ```bash
> ssh-keygen -R <server-ip>
> ```

---

## 8. Phase 5 — Fixing Package Repositories (Ubuntu 24.04 DEB822)

This phase was one of the most time-consuming parts of the lab. Ubuntu 24.04 (Noble) introduced a significant change to repository configuration.

### The Old Way (Pre-24.04)

```
/etc/apt/sources.list          ← one-line-per-repo format
```

Example:

```
deb http://archive.ubuntu.com/ubuntu/ noble main restricted
deb http://archive.ubuntu.com/ubuntu/ noble-updates main restricted
```

### The New Way (24.04+ / DEB822 Format)

```
/etc/apt/sources.list.d/ubuntu.sources    ← structured multi-line format
```

Example:

```
Types: deb
URIs: http://archive.ubuntu.com/ubuntu/
Suites: noble noble-updates noble-security
Components: main universe restricted multiverse
Signed-By: /usr/share/keyrings/ubuntu-archive-keyring.gpg
```

### The Conflict — What Actually Happened

The transplanted `/etc` came from the recovery ISO, which may have had a minimal or mismatched repository setup. The sequence of issues:

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  TIMELINE OF REPOSITORY STRUGGLES                                │
  │                                                                  │
  │  1. apt update → fails (no valid sources)                        │
  │  2. Created old-style sources.list manually                      │
  │     → apt update works, but packages still missing               │
  │  3. "Unable to locate package apache2-common"                    │
  │     → apache2-common is gone in Noble (merged into apache2-bin)  │
  │  4. Moved sources.list → sources.list.bak                        │
  │     → Force apt to use DEB822 ubuntu.sources only                │
  │  5. Edited ubuntu.sources to include universe component          │
  │     → apt update now sees full package set                       │
  │  6. add-apt-repository universe → enables universe component     │
  └──────────────────────────────────────────────────────────────────┘
```

### The Fix

**Step 1 — Disable conflicting old-style sources (if present):**

```bash
sudo mv /etc/apt/sources.list /etc/apt/sources.list.bak
```

**Step 2 — Ensure correct DEB822 configuration:**

Edit `/etc/apt/sources.list.d/ubuntu.sources`:

```
Types: deb
URIs: http://archive.ubuntu.com/ubuntu/
Suites: noble noble-updates noble-security
Components: main universe restricted multiverse
Signed-By: /usr/share/keyrings/ubuntu-archive-keyring.gpg
```

**Step 3 — Enable universe (if missing):**

```bash
add-apt-repository universe
apt update
```

### Package Name Gotcha — `apache2-common`

On Ubuntu 24.04 (Noble), `apache2-common` is a **transitional/legacy package**. The actual contents were reorganized:

| Old Package (Pre-Noble) | New Package (Noble)     | Contains                        |
| ----------------------- | ----------------------- | ------------------------------- |
| `apache2-common`        | `apache2-bin`           | Apache binaries                 |
|                         | `apache2-data`          | Default pages, icons, error docs |

Trying `apt install apache2-common` results in:

```
E: Unable to locate package apache2-common
```

The correct install command for Noble:

```bash
apt install apache2 apache2-bin apache2-data apache2-utils
```

> **Lesson:** Package names change between releases. Always verify with `apt-cache search <name>` or `apt-cache policy <name>` before assuming a package exists.

---

## 9. Phase 6 — Forced Configuration Restoration

### The Problem

Standard `apt install --reinstall` **refuses to recreate missing config files** if the package was previously installed. `dpkg` assumes the user intentionally removed the config.

```bash
apt install --reinstall apache2
# Result: binaries reinstalled, but /etc/apache2/apache2.conf still missing
```

### The Fix — dpkg Force Options

```bash
apt-get -o Dpkg::Options::="--force-confmiss" \
        -o Dpkg::Options::="--force-confnew" \
        install --reinstall apache2 apache2-bin apache2-data apache2-utils
```

| Option              | Effect                                              |
| ------------------- | --------------------------------------------------- |
| `--force-confmiss`  | Recreate config files that are missing               |
| `--force-confnew`   | Replace existing configs with package defaults       |

### Verify Config Restoration

```bash
ls -l /etc/apache2/
```

Expected structure after forced reinstall:

```
/etc/apache2/
├── apache2.conf
├── conf-available/
├── conf-enabled/
├── envvars
├── magic
├── mods-available/
├── mods-enabled/
├── ports.conf
├── sites-available/
│   ├── 000-default.conf
│   └── default-ssl.conf
└── sites-enabled/
    └── 000-default.conf → ../sites-available/000-default.conf
```

### Detour — dpkg Dependency Errors

During the lab, `dpkg` threw errors related to `ssl-cert` and the `/etc/ssl/private` directory:

```bash
# Fix: recreate the ssl-cert group that dpkg expects
groupadd --system ssl-cert
dpkg-statoverride --remove /etc/ssl/private   # if stale override exists
dpkg --configure -a                           # fix half-configured packages
apt install -f                                # resolve broken dependencies
```

> **Why this happened:** The transplanted `/etc` did not include the `ssl-cert` system group. When `dpkg` tried to set ownership on `/etc/ssl/private`, it failed because the target group did not exist. This is a classic example of cascading dependency failures.

---

## 10. Phase 7 — Rebuilding Service Users & Machine Identity

### Missing Service Users

The transplanted `/etc/passwd` came from the recovery ISO and contains **none of the original server's service accounts**.

Services that create dedicated system users during installation:

| Service    | User        | Group       | Home Dir        | Shell                  |
| ---------- | ----------- | ----------- | --------------- | ---------------------- |
| Apache2    | `www-data`  | `www-data`  | `/var/www`      | `/usr/sbin/nologin`    |
| Nginx      | `www-data`  | `www-data`  | `/var/www`      | `/usr/sbin/nologin`    |
| MySQL      | `mysql`     | `mysql`     | `/var/lib/mysql` | `/bin/false`           |
| PostgreSQL | `postgres`  | `postgres`  | `/var/lib/postgresql` | `/bin/bash`       |
| OpenSSH    | `sshd`      | `nogroup`   | `/run/sshd`     | `/usr/sbin/nologin`    |

Apache2 requires `www-data` to run worker processes. Without it:

```
[error] (22)Invalid argument: AH00013: Pre-configuration failed
```

### Recreate www-data

```bash
groupadd www-data
useradd -r -g www-data -d /var/www -s /usr/sbin/nologin www-data
```

Verify:

```bash
id www-data
# uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

> **Note:** The UID/GID may not match the original values. If application data in `/var/www` has hardcoded ownership, you may need to `chown -R www-data:www-data /var/www` after recreation.

### Regenerate Machine Identity

The transplanted `/etc/machine-id` belongs to the recovery ISO. This ID is used by:

| Consumer       | Impact of Wrong ID                              |
| -------------- | ----------------------------------------------- |
| `systemd`      | Journal logs mixed with wrong identity          |
| `journald`     | Persistent journal directory mismatch           |
| `D-Bus`        | IPC bus identity collision                      |
| `cloud-init`   | Cloud provider may misidentify the instance     |
| DHCP client    | May receive wrong lease                         |

Fix:

```bash
rm -f /etc/machine-id
systemd-machine-id-setup
```

Verify:

```bash
cat /etc/machine-id
# Should show a new 32-character hex string
```

---

## 11. Phase 8 — Apache Recovery: The MPM Problem

### The Error

After reinstalling Apache and recreating `www-data`, the service still fails:

```bash
apachectl -t
```

```
AH00534: apache2: Configuration error: No MPM loaded.
```

### Why This Happens

Apache uses a **Multi-Processing Module (MPM)** to handle concurrent connections. The MPM is loaded via **symlinks** in `/etc/apache2/mods-enabled/`:

```
  /etc/apache2/
  ├── mods-available/         ← all module .load and .conf files
  │   ├── mpm_event.load
  │   ├── mpm_prefork.load
  │   ├── mpm_worker.load
  │   ├── authz_core.load
  │   └── ...
  └── mods-enabled/           ← symlinks to active modules
      ├── mpm_event.load → ../mods-available/mpm_event.load
      ├── authz_core.load → ../mods-available/authz_core.load
      └── ...
```

The forced reinstall restored the `mods-available/` directory but the `mods-enabled/` symlinks were empty or incomplete.

### The Fix

```bash
# Enable the event MPM (modern, recommended for most workloads)
a2enmod mpm_event

# Enable core authorization modules
a2enmod authz_core
a2enmod authz_host

# Verify (optional — enable other common defaults)
a2enmod dir
a2enmod autoindex
a2enmod access_compat
```

### MPM Selection Guide

| MPM          | Concurrency Model     | Best For                     |
| ------------ | --------------------- | ---------------------------- |
| `mpm_event`  | Async + threads       | Modern general purpose ✅    |
| `mpm_worker` | Threaded              | High concurrency, no PHP     |
| `mpm_prefork`| Process-per-request   | mod_php (legacy)             |

> **Production note:** Only **one** MPM can be loaded at a time. If you enable `mpm_event` while `mpm_prefork` is already enabled, Apache will fail. Use `a2dismod mpm_prefork` first if needed.

### Validate

```bash
apachectl -t
# Expected: Syntax OK
```

---

## 12. Phase 9 — Network Port Verification & Final Validation

### Check for Port Conflicts

Before starting Apache, verify no other service is occupying port 80:

```bash
ss -tulpn | grep :80
```

If Nginx or another service is listening:

```bash
systemctl stop nginx
systemctl disable nginx   # prevent it from starting on boot
```

### Start Apache

```bash
systemctl restart apache2
systemctl enable apache2
```

### Final Verification

```bash
# Config syntax
apachectl -t
# → Syntax OK

# Service status
systemctl status apache2
# → active (running)

# Port binding
ss -tulpn | grep :80
# → *:80   users:(("apache2",...))

# HTTP response
curl -I http://localhost
# → HTTP/1.1 200 OK
```

If all four checks pass, Apache is successfully recovered.

---

## 13. Post-Recovery State Assessment

### What Was Recovered

| Component              | Status      | Notes                              |
| ---------------------- | ----------- | ---------------------------------- |
| Root login             | ✅ Working   | Password reset via chroot         |
| SSH access             | ✅ Working   | Host keys regenerated             |
| Package manager        | ✅ Working   | DEB822 config corrected           |
| Apache2                | ✅ Working   | Force-reinstalled + modules wired |
| System identity        | ✅ Working   | machine-id regenerated            |

### What Remains Broken / Missing

| Component              | Status      | Recovery Path                         |
| ---------------------- | ----------- | ------------------------------------- |
| Original user accounts | ❌ Lost      | Restore from backup or recreate      |
| Custom SSH configs     | ❌ Lost      | Rebuild or restore from IaC          |
| TLS certificates       | ❌ Lost      | Re-issue from CA or restore backup   |
| Firewall rules         | ❌ Lost      | Rebuild `iptables`/`ufw`/`nftables`  |
| Custom Apache vhosts   | ❌ Lost      | Restore from VCS or backup           |
| Nginx configuration    | ❌ Lost      | Reinstall + restore configs          |
| `/etc/fstab`           | ⚠️ Generic  | Verify mount points match disks       |
| crontab system entries | ❌ Lost      | `/etc/cron.*` dirs gone              |
| PAM configuration      | ⚠️ Generic  | May break advanced auth (LDAP, 2FA)   |
| NSS configuration      | ⚠️ Generic  | `/etc/nsswitch.conf` from ISO         |
| Sudoers                | ❌ Lost      | `/etc/sudoers` + `/etc/sudoers.d/`   |

This is a **Frankenstein system** — functional enough to serve HTTP, but missing customization, hardening, and non-default configuration.

---

## 14. Gotchas & Lessons Learned

### 14.1 The DEB822 Trap

Ubuntu 24.04 uses two repo config systems simultaneously. If both `/etc/apt/sources.list` and `/etc/apt/sources.list.d/ubuntu.sources` exist, they can conflict or duplicate entries.

**Best practice:** On Noble+, use **only** the DEB822 format. Move or remove `sources.list`.

```bash
mv /etc/apt/sources.list /etc/apt/sources.list.bak
# All config in /etc/apt/sources.list.d/ubuntu.sources
```

### 14.2 Package Name Changes Between Releases

Package names are **not stable across Ubuntu releases**. Always verify:

```bash
apt-cache search <keyword> | grep <keyword>
apt-cache policy <package-name>
```

`apache2-common` → merged into `apache2-bin` + `apache2-data` in Noble.

### 14.3 Service Users Are Invisible Dependencies

Services fail silently or cryptically when their expected user/group does not exist. Always check:

```bash
id www-data          # Apache/Nginx
id mysql             # MySQL
id postgres          # PostgreSQL
id sshd              # OpenSSH
```

After transplanting `/etc`, none of these will exist.

### 14.4 Symlinks Are Configuration

Apache's `mods-enabled`, `sites-enabled`, and `conf-enabled` directories work entirely through symlinks to their `-available` counterparts. The `--force-confmiss` reinstall restores the **available** directory but not always the **enabled** symlinks.

```
mods-available/mpm_event.load    ← file exists
mods-enabled/mpm_event.load      ← symlink MISSING → Apache won't load the module
```

Use `a2enmod`, `a2ensite`, `a2enconf` to rebuild these.

### 14.5 Machine ID Duplication Is Subtle

A duplicated `machine-id` will not cause an obvious crash. Instead you get:

* Journals mixed between systems
* Cloud-init refusing to re-run
* DHCP lease confusion
* D-Bus address conflicts in multi-container setups

Always regenerate after transplanting `/etc`.

### 14.6 `/etc/shadow` Permissions Are a Landmine

| Permission | Outcome                                       |
| ---------- | --------------------------------------------- |
| `644`      | World-readable hashes — **security hole**, some daemons refuse to start |
| `600`      | Only root — PAM shadow group checks may fail  |
| `640`      | ✅ Correct — root can write, shadow group can read |

### 14.7 SSH Host Key Mismatch Isn't Optional

After recovery, every SSH client that previously connected will show `REMOTE HOST IDENTIFICATION HAS CHANGED`. This is not cosmetic — it is a security mechanism.

**For the client to reconnect:**

```bash
ssh-keygen -R <server-ip>
```

**For automated systems (Ansible, CI/CD):**

Update `known_hosts` files or distribute new host keys via configuration management.

### 14.8 dpkg Half-Configured State

If a package installation fails midway (e.g., missing group for `ssl-cert`), dpkg enters a half-configured state. Future installs will refuse until resolved:

```bash
dpkg --configure -a        # attempt to configure all pending
apt install -f             # fix broken dependencies
```

---

## 15. Production-Grade Recovery Alternatives

Manual `/etc` reconstruction is an **educational exercise**, not a production strategy. Here is what real-world DR looks like:

### 15.1 Snapshot Recovery (Fastest)

```
  ┌──────────────────────────────────────────────────────────┐
  │                    SNAPSHOT RECOVERY                     │
  │                                                          │
  │  ┌────────────┐    ┌─────────────┐    ┌──────────────┐   │
  │  │ Detect     │───►│ Power off   │───►│ Restore from │   │
  │  │ disaster   │    │ instance    │    │ last good    │   │
  │  │            │    │             │    │ snapshot     │   │
  │  └────────────┘    └─────────────┘    └──────┬───────┘   │
  │                                              │           │
  │                                       ┌──────▼───────┐   │
  │                                       │ Boot into    │   │
  │                                       │ restored     │   │
  │                                       │ system       │   │
  │                                       └──────────────┘   │
  │  RTO: minutes | RPO: last snapshot interval              │
  └──────────────────────────────────────────────────────────┘
```

| Provider       | Snapshot Method                                  |
| -------------- | ------------------------------------------------ |
| DigitalOcean   | Droplet → Snapshots → Create Snapshot            |
| AWS            | EBS Snapshot / AMI                               |
| GCP            | Persistent Disk Snapshot                         |
| Azure          | Managed Disk Snapshot                            |

**Best practice:** Automated daily snapshots with retention policy.

### 15.2 Configuration Backup

Targeted `/etc` backups are lightweight and fast to restore:

```bash
# Manual — cron this daily
tar czf /backup/etc-$(date +%F).tar.gz /etc

# With rsnapshot (incremental, space-efficient)
rsnapshot daily

# With borgbackup (deduplicated, encrypted)
borg create /backup/repo::etc-{now} /etc

# With restic (encrypted, multi-backend)
restic -r /backup/repo backup /etc
```

**Recovery:**

```bash
cd / && tar xzf /backup/etc-2026-03-05.tar.gz
```

> **Key insight:** Backing up just `/etc` (typically 10–50 MB) is trivial compared to full disk backups, and it captures 90% of what you need for service recovery.

### 15.3 Infrastructure as Code (IaC)

In mature environments, `/etc` is **declaratively managed** — no manual changes allowed.

| Tool       | Approach                      | Recovery                            |
| ---------- | ----------------------------- | ----------------------------------- |
| Ansible    | Playbooks push configs        | Re-run playbook on fresh instance   |
| Terraform  | Provisions infrastructure     | `terraform apply` rebuilds infra    |
| Puppet     | Agent enforces desired state  | Agent self-heals on next run        |
| Chef       | Recipes define configs        | Re-converge the node                |
| SaltStack  | State files define system     | Re-apply highstate                  |

**The philosophy:**

```
  ┌──────────────────────────────────────────────────┐
  │  Servers are cattle, not pets.                   │
  │  Configuration is code, stored in version        │
  │  control, tested, and reproducible.              │
  │                                                  │
  │  If a server breaks → destroy it → redeploy.     │
  │  Manual repair is a sign of missing automation.  │
  └──────────────────────────────────────────────────┘
```

### 15.4 etckeeper — Version Control for `/etc`

`etckeeper` tracks `/etc` changes in Git automatically:

```bash
apt install etckeeper
etckeeper init
etckeeper commit "initial commit"
```

Every `apt install` auto-commits. Recovery becomes:

```bash
cd /etc && git log         # find last good commit
git checkout <commit>      # restore
```

> **Lightweight, zero-cost insurance.** Should be on every server that is not fully managed by IaC.

---

## 16. Variables That Change Everything

Recovery complexity depends heavily on the environment. What worked in this lab may not work elsewhere.

### 16.1 Filesystem Type

| Filesystem | Recovery Options                                                   |
| ---------- | ------------------------------------------------------------------ |
| **ext4**   | No native snapshots. Recovery depends on external backups. `extundelete` may recover recently deleted files if no writes have occurred |
| **XFS**    | Similar to ext4 — no snapshots. `xfs_undelete` is experimental    |
| **BTRFS**  | ✅ Native snapshots. `btrfs subvolume snapshot` + `btrfs send/receive`. Can rollback `/etc` instantly if subvolume snapshots exist |
| **ZFS**    | ✅ Native snapshots. `zfs snapshot` + `zfs rollback`. Near-instant recovery |

> **Production recommendation:** If your infrastructure allows it, BTRFS or ZFS snapshots provide the cheapest possible DR for filesystem-level disasters. openSUSE uses BTRFS by default; Ubuntu supports it optionally.

### 16.2 OS Distribution

| Distro           | Package Manager | Repo Format         | Apache Package       | Key Differences                    |
| ---------------- | --------------- | ------------------- | -------------------- | ---------------------------------- |
| Ubuntu 24.04     | `apt`           | DEB822 (`.sources`) | `apache2`            | DEB822 is new, `universe` separate |
| Ubuntu 22.04     | `apt`           | `sources.list`      | `apache2`            | Traditional format                 |
| Debian 12        | `apt`           | `sources.list`      | `apache2`            | No `universe` — uses main/contrib/non-free |
| RHEL 9 / Rocky 9 | `dnf`           | `.repo` files       | `httpd`              | SELinux enforcing, firewalld       |
| Arch Linux       | `pacman`        | `pacman.conf`       | `apache` (pkg name)  | Rolling release, different paths   |
| openSUSE         | `zypper`        | `.repo` files       | `apache2`            | BTRFS default, YaST integration    |

The same disaster on RHEL would require different recovery steps:

* **SELinux relabeling** (`touch /.autorelabel`)
* **firewalld** rule restoration
* **dnf** instead of `apt`
* Config paths differ (`/etc/httpd/` not `/etc/apache2/`)

### 16.3 Init System

| Init System | Distros                    | Recovery Consideration                              |
| ----------- | -------------------------- | --------------------------------------------------- |
| systemd     | Most modern distros        | `machine-id`, journal, unit files in `/etc/systemd` |
| OpenRC      | Gentoo, Alpine             | `/etc/init.d/` scripts, `/etc/conf.d/` configs      |
| runit       | Void Linux, some containers | `/etc/sv/` service directories                     |

### 16.4 Cloud Provider vs Bare Metal

| Factor            | Cloud VM                             | Bare Metal                          |
| ----------------- | ------------------------------------ | ----------------------------------- |
| Recovery Console  | Web-based VNC / serial console       | Physical KVM / IPMI / iDRAC / iLO   |
| Boot Media        | Provider recovery ISO / rescue mode  | USB drive / PXE boot                |
| Snapshot Recovery | API-driven, minutes                  | Requires external backup solution   |
| Reprovisioning    | Destroy + recreate via API/Terraform | Reinstall from scratch (hours)      |

---

## 17. Best Practices — Mapping Lab Lessons to Production

Each phase of this lab maps directly to a production best practice:

### Prevention Layer

| Lab Lesson                        | Production Practice                                       |
| --------------------------------- | --------------------------------------------------------- |
| `/etc` deletion = total loss      | **Automated `/etc` backups** (daily tar, etckeeper, borg) |
| Manual recovery took hours        | **IaC for all config** (Ansible playbooks in Git)         |
| Package names change per release  | **Pin and document dependencies** in runbooks             |
| Service users are invisible deps  | **Automate user creation** in provisioning scripts        |

### Detection Layer

| Lab Lesson                        | Production Practice                                       |
| --------------------------------- | --------------------------------------------------------- |
| System was silently broken        | **File integrity monitoring** (AIDE, Tripwire, OSSEC)     |
| No alert on `/etc` deletion       | **Auditd rules** on critical directories                  |
| Discovery was manual              | **Automated health checks** (Nagios, Prometheus, Datadog) |

### Recovery Layer

| Lab Lesson                        | Production Practice                                       |
| --------------------------------- | --------------------------------------------------------- |
| Recovery ISO = last resort        | **Regular snapshot schedule** (hourly/daily)              |
| chroot was needed for repair      | **Documented runbooks** for DR procedures                 |
| Port conflict blocked Apache      | **Service dependency maps** in documentation              |
| Machine-id duplication was subtle | **Post-recovery validation checklist**                    |

### Recovery Priorities Hierarchy

```
  ┌──────────────────────────────────────────────────────────────┐
  │  RECOVERY STRATEGY PREFERENCE (fastest → slowest)            │
  │                                                              │
  │  1. Filesystem snapshot rollback (BTRFS/ZFS)     ← seconds   │
  │  2. Cloud provider snapshot restore              ← minutes   │
  │  3. Redeploy via IaC (Ansible/Terraform)         ← minutes   │
  │  4. Restore /etc from backup (tar/borg/restic)   ← minutes   │
  │  5. etckeeper git rollback                       ← minutes   │
  │  6. Force-reinstall packages (--force-confmiss)  ← hours     │
  │  7. Manual reconstruction (this lab)             ← hours+    │
  │                                                              │
  │  ⚠️  Manual reconstruction is LAST RESORT.                   │
  └──────────────────────────────────────────────────────────────┘
```

### Minimum Viable DR Setup for Any Server

```bash
# 1. Install etckeeper (5 minutes, zero maintenance)
apt install etckeeper && etckeeper init

# 2. Daily /etc backup via cron
echo "0 2 * * * root tar czf /backup/etc-\$(date +\%F).tar.gz /etc" \
  >> /etc/cron.d/etc-backup

# 3. Cloud snapshot schedule via provider API/UI
# (provider-specific — automate via Terraform or API)

# 4. Store all custom configs in Git
# /etc/apache2/sites-available/mysite.conf → version controlled
```

---

## 18. Suggested Extensions

Advanced follow-up exercises to deepen understanding:

| Exercise                              | Skill Practiced                             |
| ------------------------------------- | ------------------------------------------- |
| Recover `/etc` using `extundelete`    | Filesystem-level file recovery              |
| Rebuild `/etc/fstab` from `lsblk`    | Disk/mount understanding                     |
| Reconstruct firewall rules from logs  | `iptables`/`nftables` forensics             |
| Restore users from `/var/log/auth.log`| User account forensics                      |
| Rebuild Nginx config from scratch     | Nginx internals                             |
| Perform same lab on CentOS/RHEL       | Cross-distro recovery (dnf, SELinux)        |
| Automate recovery with Ansible        | IaC-based DR                                |
| Set up BTRFS and test snapshot rollback | Filesystem snapshot mechanics             |
| Implement AIDE and trigger on `/etc`  | File integrity monitoring                   |
| Use `debsums` to verify package files | Package integrity checking                  |

---

## 19. Final Thoughts

Deleting `/etc` reveals how Linux is fundamentally structured:

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                          │
  │   The kernel runs the system.                            │
  │   /etc defines its behavior.                             │
  │                                                          │
  │   Without /etc, Linux is alive but confused —            │
  │   a brain without memory.                                │
  │                                                          │
  └──────────────────────────────────────────────────────────┘
```

This lab demonstrated that manual reconstruction is **possible but impractical**. Every phase — from repository conflicts to missing symlinks to invisible service users — reinforces why production systems invest in:

* **Automated backups** — because disasters happen
* **Infrastructure as Code** — because servers should be disposable
* **Monitoring and alerting** — because silent failures are the worst kind
* **Documented runbooks** — because recovery under pressure needs a checklist

> **The DevOps lesson:** Reliable infrastructure depends more on **reproducibility** than on manual repair skill. The correct recovery for a broken production server is almost never manual reconstruction — it is **redeployment from known-good configuration**.

But understanding *how* to reconstruct a system from nothing teaches you *what* to automate and *why* every config file matters.

---

> **Lab performed by:** Vishvam Moliya  
> **Date:** March 2026  
> **Environment:** DigitalOcean Ubuntu 24.04 LTS  
> **Status:** Educational — not a production runbook

---

[🏠 Back to Home](../README.md) · [Labs](README.md)
