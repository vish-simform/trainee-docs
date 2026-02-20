[🏠 Home](../README.md) · [Windows](README.md)

# 🖥️ Windows Server 2022 — Comprehensive DevOps/Cloud Cheatsheet

> **Audience:** DevOps / Cloud Engineers | **Focus:** Architecture, Mental Models, Why it Matters  
> **Tip:** This cheatsheet groups related services together for better revision flow.

---

## Table of Contents

1. [Identity & Access — AD DS, DC, RODC, FSMO](#1-identity--access--ad-ds-dc-rodc-fsmo)
2. [Organizational Structure — OUs, Groups, gMSA](#2-organizational-structure--ous-groups-gmsa)
3. [Policy Engine — Group Policy Objects (GPO)](#3-policy-engine--group-policy-objects-gpo)
4. [Name Resolution — DNS Servers, Zones & Records](#4-name-resolution--dns-servers-zones--records)
5. [IP Management — DHCP Servers & Failover](#5-ip-management--dhcp-servers--failover)
6. [Network Security — DMZ, RAS & NPS](#6-network-security--dmz-ras--nps)
7. [File Services — Sharing, DFS, BranchCache, FSRM](#7-file-services--sharing-dfs-branchcache-fsrm)
8. [Storage — Disks, Volumes, Pools, Replicas, File Systems](#8-storage--disks-volumes-pools-replicas-file-systems)
9. [Permissions — NTFS vs Share Permissions](#9-permissions--ntfs-vs-share-permissions)
10. [Automation — PowerShell ISE](#10-automation--powershell-ise)
11. [X vs Y — Head-to-Head Comparisons](#11-x-vs-y--head-to-head-comparisons)
12. [Viva / Trainee Review — Q&A Bank](#12-viva--trainee-review--qa-bank)

---

## 1. Identity & Access — AD DS, DC, RODC, FSMO

### 1.1 Active Directory Domain Services (AD DS)

#### What Is It?

AD DS is the **central identity and authentication backbone** of a Windows environment. Think of it as the **"user/device/service database + authentication engine"** for your entire organization.

#### Mental Model

```
┌─────────────────────────────────────────────────────────┐
│                      FOREST                             │
│  (Security & Trust Boundary — the whole organization)   │
│                                                         │
│  ┌───────────────────┐    ┌───────────────────┐         │
│  │    DOMAIN A        │◄──►│    DOMAIN B        │  ← Trust│
│  │  (e.g. corp.com)  │    │  (e.g. dev.corp.com)│       │
│  │                   │    │                     │       │
│  │  ┌─────┐ ┌─────┐ │    │  ┌─────┐ ┌─────┐   │       │
│  │  │ DC1 │ │ DC2 │ │    │  │ DC3 │ │RODC │   │       │
│  │  └─────┘ └─────┘ │    │  └─────┘ └─────┘   │       │
│  └───────────────────┘    └───────────────────┘         │
└─────────────────────────────────────────────────────────┘

Key Objects stored in AD:
  👤 Users    💻 Computers    👥 Groups    🔑 Service Accounts
  📂 OUs      📜 GPOs         🖨️ Printers   📋 Policies
```

#### Core Concepts

| Concept | What It Is | Analogy |
|---------|-----------|---------|
| **Forest** | Top-level container; security boundary | The entire company |
| **Domain** | Logical grouping of objects sharing a DB | A department/division |
| **Tree** | Hierarchy of domains sharing contiguous namespace | `corp.com` → `dev.corp.com` |
| **Site** | Physical/network topology mapping | Office location / data center |
| **Object** | Any entity (user, computer, group, etc.) | A row in a database |
| **Schema** | Blueprint defining what objects can exist | Database schema |

#### Why Does a DevOps/Cloud Engineer Care?

- **Hybrid Cloud Identity:** Azure AD Connect syncs on-prem AD to Azure AD (Entra ID). Understanding AD DS = understanding the *source of truth* for hybrid identity.
- **Service Authentication:** CI/CD pipelines, IIS apps, SQL Server — they all authenticate against AD. You manage their identities.
- **Infrastructure as Code:** You'll script AD object creation (users, groups, OUs) with PowerShell in automated provisioning.
- **Zero Trust Architecture:** AD is the foundation — Conditional Access, MFA, RBAC all build on top of it.
- **GPO for Config Management:** AD-linked GPOs are the Windows equivalent of Ansible/Chef for policy enforcement.

---

### 1.2 Domain Controller (DC) vs Read-Only Domain Controller (RODC)

#### Architectural Flow — Authentication

```
                    ┌──────────────┐
 User logs in ──────►│   Domain     │
 (Kerberos/NTLM)   │  Controller  │
                    │   (DC)       │
                    └──────┬───────┘
                           │ Replicates
                           ▼
                    ┌──────────────┐
                    │   Domain     │
                    │  Controller  │
                    │   (DC2)      │  ← Multi-Master Replication
                    └──────────────┘

 Authentication Flow (Kerberos):
 ┌────────┐  1. Request TGT  ┌────────┐
 │ Client ├──────────────────►│  DC    │
 │        │◄──────────────────┤ (KDC)  │
 │        │  2. Return TGT   │        │
 │        │                  │        │
 │        │  3. Request      │        │
 │        │  Service Ticket  │        │
 │        ├──────────────────►│        │
 │        │◄──────────────────┤        │
 │        │  4. Return ST    │        │
 │        │                  └────────┘
 │        │  5. Access Service
 │        ├──────────────────► 🖥️ Service
 └────────┘
```

#### DC vs RODC — Side by Side

| Aspect | Domain Controller (DC) | Read-Only Domain Controller (RODC) |
|--------|----------------------|-----------------------------------|
| **Write Access** | Full read/write to AD database | Read-only — cannot write directly |
| **Password Caching** | Stores all passwords | Caches only allowed passwords (PRP) |
| **Replication** | Multi-master (any DC can write) | One-way inbound only |
| **Use Case** | Primary DCs in secure data centers | Branch offices, DMZs, edge locations |
| **Security Risk** | High value target — full DB | Lower risk — limited data, no write |
| **DNS** | Read/Write DNS | Read-Only DNS |

#### RODC — When and Why?

```
  HQ Data Center                    Branch Office (Low Security)
  ┌──────────────┐                  ┌──────────────────┐
  │  DC (R/W)    │  ──── WAN ────► │  RODC            │
  │  Full AD DB  │  One-way        │  Cached subset   │
  │              │  replication    │  Read-only       │
  └──────────────┘                  └──────────────────┘
                                    │
                                    ├─ If RODC is compromised:
                                    │  • Attacker gets limited data
                                    │  • Cannot modify AD
                                    │  • Passwords not fully cached
                                    └─ Reset only cached credentials
```

**DevOps Relevance:** In cloud/hybrid, RODC concepts map to **read replicas** in databases, **edge caching** patterns, and the principle of **least privilege at the edge**.

---

### 1.3 FSMO Roles (Flexible Single Master Operations)

Even though AD uses multi-master replication (any DC can accept writes), **some operations MUST have a single authoritative owner** to prevent conflicts. These are the 5 FSMO roles.

#### Mental Model — The 5 Roles

```
┌─────────────────────────────────────────────────────────────────┐
│                    FOREST-WIDE (1 per Forest)                   │
│                                                                 │
│  ┌────────────────────────┐  ┌────────────────────────────┐     │
│  │  Schema Master         │  │  Domain Naming Master      │     │
│  │  • Controls schema     │  │  • Add/remove domains      │     │
│  │    changes (one at     │  │    from the forest          │     │
│  │    a time, forest-wide)│  │  • Ensures unique naming    │     │
│  └────────────────────────┘  └────────────────────────────┘     │
├─────────────────────────────────────────────────────────────────┤
│                    DOMAIN-WIDE (1 per Domain)                   │
│                                                                 │
│  ┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐ │
│  │  RID Master      │ │  PDC Emulator    │ │ Infrastructure   │ │
│  │  • Allocates RID │ │  • Time sync     │ │ Master           │ │
│  │    pools to DCs  │ │  • Password      │ │ • Cross-domain   │ │
│  │  • Ensures unique│ │    changes       │ │   reference      │ │
│  │    SIDs          │ │  • GPO edits     │ │   updates        │ │
│  │                  │ │  • Account lockout│ │                  │ │
│  └──────────────────┘ └──────────────────┘ └──────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

#### Quick Reference Table

| FSMO Role | Scope | What Happens If It Fails? | Urgency to Restore |
|-----------|-------|--------------------------|---------------------|
| **Schema Master** | Forest | Can't modify schema (rare operation) | Low — rarely needed |
| **Domain Naming Master** | Forest | Can't add/remove domains | Low — rare operation |
| **RID Master** | Domain | DCs eventually exhaust RID pools, can't create new objects | Medium |
| **PDC Emulator** | Domain | Time drift, password change delays, GPO edit issues | **HIGH — most used** |
| **Infrastructure Master** | Domain | Cross-domain group membership issues | Low in single-domain |

#### DevOps Relevance

- **PDC Emulator** is the most critical — it's the **time source** for Kerberos (auth fails with >5 min drift). In cloud, time sync = NTP servers = critical for certificate validation, logs, distributed systems.
- **Schema Master** matters when extending schema for Exchange, SCCM, or custom LDAP attributes.
- **Transfer vs Seize:** Transfer = graceful migration. Seize = force-take when old DC is permanently dead. Know the difference for disaster recovery.

---

## 2. Organizational Structure — OUs, Groups, gMSA

### 2.1 Organizational Units (OUs)

#### What Is It?

OUs are **containers within a domain** used to organize objects AND to **apply GPOs**. They are the **folder structure of AD**.

#### Mental Model

```
  Domain: corp.com
  │
  ├── 📁 OU: Headquarters
  │   ├── 📁 OU: IT Department
  │   │   ├── 👤 john.admin
  │   │   ├── 💻 SRV-WEB-01
  │   │   └── 📜 GPO: IT-Security-Policy ← applied here
  │   ├── 📁 OU: HR Department
  │   │   ├── 👤 jane.hr
  │   │   └── 📜 GPO: HR-Restrictions
  │   └── 📁 OU: Service Accounts
  │       └── 🔑 svc-cicd-agent
  │
  ├── 📁 OU: Branch Offices
  │   ├── 📁 OU: Mumbai
  │   └── 📁 OU: Bangalore
  │
  └── 📁 OU: Servers
      ├── 📁 OU: Production
      └── 📁 OU: Staging
```

#### Key Rules

- OUs are **NOT security principals** — you can't assign permissions to an OU.
- OUs **CAN have GPOs linked** — this is their primary power.
- OUs support **delegation** — give a team admin rights over just their OU.
- GPO Inheritance flows: **Domain → Parent OU → Child OU** (can be blocked or enforced).

---

### 2.2 Groups — Security vs Distribution

#### Mental Model

```
┌─────────────────────────────────────────────────────────┐
│                    AD Groups                            │
│                                                         │
│  ┌─────────────────────┐  ┌──────────────────────────┐  │
│  │  Security Groups    │  │  Distribution Groups     │  │
│  │  • Assign NTFS/     │  │  • Email distribution    │  │
│  │    Share permissions │  │    lists only            │  │
│  │  • Used in ACLs     │  │  • NO security/ACL use   │  │
│  │  • Have a SID       │  │  • Used by Exchange      │  │
│  └─────────────────────┘  └──────────────────────────┘  │
│                                                         │
│  Group Scopes:                                          │
│  ┌──────────────────────────────────────────────────┐   │
│  │ Domain Local  │ Can contain members from ANY     │   │
│  │               │ domain. Used to assign           │   │
│  │               │ permissions to LOCAL resources.  │   │
│  ├───────────────┼──────────────────────────────────┤   │
│  │ Global        │ Members from SAME domain only.   │   │
│  │               │ Used across the forest.          │   │
│  ├───────────────┼──────────────────────────────────┤   │
│  │ Universal     │ Members from ANY domain.         │   │
│  │               │ Cached in Global Catalog.        │   │
│  │               │ ⚠️ Replication overhead.          │   │
│  └───────────────┴──────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

#### Best Practice: AGDLP / AGUDLP Strategy

```
  A   → Accounts (Users/Computers)
  G   → Global Groups (organize by role/function)
  U   → Universal Groups (optional, for multi-domain)
  DL  → Domain Local Groups (for resource permissions)
  P   → Permissions (assigned to DL groups)

  Example:
  👤 john.dev ──► 👥 GG-Developers ──► 👥 DL-FileShare-ReadWrite ──► 📂 Permission on \\SRV\Code
```

**DevOps Relevance:**
- **RBAC in cloud mirrors this pattern** — Users → Roles → Resource Permissions.
- Azure AD groups, AWS IAM groups follow the same nesting logic.
- You'll automate group membership in CI/CD for access provisioning.

---

### 2.3 Group Managed Service Accounts (gMSA)

#### The Problem gMSA Solves

```
  ❌ Traditional Service Accounts:
  ┌──────────────┐        ┌──────────────┐
  │ IIS App Pool │        │ SQL Server   │
  │ Runs as:     │        │ Runs as:     │
  │ svc-webapp   │        │ svc-sqlprod  │
  │ Pass: P@ss1  │        │ Pass: P@ss2  │
  └──────────────┘        └──────────────┘
  Problems:
  • Passwords never rotated (security risk)
  • Passwords stored/shared in scripts
  • Password expiry breaks services
  • Manual management nightmare

  ✅ gMSA Solution:
  ┌──────────────┐        ┌──────────────┐
  │ IIS App Pool │        │ SQL Server   │
  │ Runs as:     │        │ Runs as:     │
  │ gMSA-webapp$ │        │ gMSA-sqlprd$ │
  │ Pass: AUTO   │        │ Pass: AUTO   │
  └──────────────┘        └──────────────┘
  • 240-char password, auto-rotated every 30 days
  • AD manages the password — no human knows it
  • Multiple servers can share one gMSA
  • Works with scheduled tasks, IIS, SQL, services
```

#### How gMSA Works — Architecture

```
  ┌──────────┐                  ┌──────────────┐
  │  DC      │  KDS Root Key    │  DC          │
  │  (KDC)   ├──────────────────┤  (generates  │
  │          │                  │   password)  │
  └────┬─────┘                  └──────────────┘
       │ Password retrieved
       │ automatically via
       │ Kerberos
       ▼
  ┌────────────┐  ┌────────────┐  ┌────────────┐
  │  Server A  │  │  Server B  │  │  Server C  │
  │  Uses      │  │  Uses      │  │  Uses      │
  │  gMSA-web$ │  │  gMSA-web$ │  │  gMSA-web$ │
  └────────────┘  └────────────┘  └────────────┘
  All servers authorized       Servers must be in the
  via PrincipalsAllowed        "PrincipalsAllowedTo
  ToRetrieveManagedPassword    RetrieveManagedPassword"
                               group/list
```

#### Key PowerShell Commands

```powershell
# Step 1: Create KDS Root Key (once per forest, wait 10hrs or skip for lab)
Add-KdsRootKey -EffectiveImmediately   # Lab only
Add-KdsRootKey -EffectiveTime ((Get-Date).AddHours(-10))  # Production

# Step 2: Create the gMSA
New-ADServiceAccount -Name "gMSA-WebApp" `
  -DNSHostName "gMSA-WebApp.corp.com" `
  -PrincipalsAllowedToRetrieveManagedPassword "WebServers-Group"

# Step 3: Install on target server
Install-ADServiceAccount -Identity "gMSA-WebApp"

# Step 4: Test
Test-ADServiceAccount -Identity "gMSA-WebApp"  # Returns True
```

**DevOps Relevance:**
- **gMSA eliminates hardcoded passwords** in CI/CD agents, build servers, scheduled tasks.
- Maps to **Workload Identity Federation** in cloud (GCP), **Managed Identities** (Azure), **IAM Roles for Service Accounts** (AWS).
- Essential for **secrets management** strategy — reduce secret sprawl.

---

## 3. Policy Engine — Group Policy Objects (GPO)

### What Is It?

GPOs are the **centralized configuration management engine** for Windows environments. They push settings (security, software, scripts, registry) to users and computers.

### Mental Model — GPO Processing Order

```
  GPO Application Order (LSDOU):

  ┌─────────────────────────────────────────────────┐
  │  1. LOCAL Policy        (on the machine itself) │  ← Lowest priority
  │  2. SITE Policy         (AD Site)               │
  │  3. DOMAIN Policy       (corp.com)              │
  │  4. OU Policy           (parent → child OUs)    │  ← Highest priority
  └─────────────────────────────────────────────────┘

  Last applied WINS (unless "Enforced" is set on a higher-level GPO)

  ┌────────────────────────────────────────────────────────┐
  │  Special Modifiers:                                    │
  │                                                        │
  │  🔒 Enforced (No Override):                            │
  │     Higher-level GPO CANNOT be overridden by child OUs │
  │                                                        │
  │  🚫 Block Inheritance:                                 │
  │     OU stops inheriting parent GPOs                    │
  │     (but "Enforced" GPOs still apply!)                 │
  │                                                        │
  │  🎯 Security Filtering:                                │
  │     Apply GPO only to specific groups/users            │
  │                                                        │
  │  📍 WMI Filtering:                                     │
  │     Apply GPO based on system properties               │
  │     (OS version, disk space, etc.)                     │
  └────────────────────────────────────────────────────────┘
```

### GPO Architecture — Behind the Scenes

```
  A GPO has TWO components:

  ┌──────────────────────────┐     ┌──────────────────────────────┐
  │  Group Policy Container  │     │  Group Policy Template       │
  │  (GPC)                   │     │  (GPT)                       │
  │                          │     │                              │
  │  Stored in: AD Database  │     │  Stored in: SYSVOL share     │
  │  Contains: metadata,     │     │  Contains: actual settings,  │
  │  status, links, version  │     │  scripts, .admx templates    │
  │                          │     │  Path: \\domain\SYSVOL\...   │
  └──────────────────────────┘     └──────────────────────────────┘
            │                                  │
            └──────────┬───────────────────────┘
                       │
              Both linked by GUID
              e.g. {31B2F340-016D-11D2-945F-00C04FB984F9}
```

### GPO — Two Configuration Sections

```
  ┌───────────────────────────────────────────────┐
  │              GPO: "Server Hardening"           │
  │                                               │
  │  ┌────────────────────┐ ┌──────────────────┐  │
  │  │ Computer Config    │ │ User Config      │  │
  │  │                    │ │                  │  │
  │  │ • Security settings│ │ • Desktop       │  │
  │  │ • Startup scripts  │ │   restrictions  │  │
  │  │ • Windows Firewall │ │ • Logon scripts │  │
  │  │ • Audit policies   │ │ • Drive mappings│  │
  │  │ • Software install │ │ • Folder redir. │  │
  │  │                    │ │ • Software inst.│  │
  │  │ Applied at:        │ │ Applied at:     │  │
  │  │ BOOT + every       │ │ LOGIN + every   │  │
  │  │ 90-120 min         │ │ 90-120 min      │  │
  │  └────────────────────┘ └──────────────────┘  │
  └───────────────────────────────────────────────┘
```

### Common GPO Use Cases for DevOps

| Use Case | GPO Setting | Why You'd Use It |
|----------|-------------|-----------------|
| Enforce password complexity | Computer → Security Settings → Account Policy | Compliance requirements |
| Disable USB storage | Computer → Admin Templates → System → Removable Storage | Server hardening |
| Map network drives | User → Preferences → Drive Maps | Standardize environment |
| Deploy startup scripts | Computer → Windows Settings → Scripts → Startup | Bootstrap servers |
| Configure Windows Firewall | Computer → Security Settings → Windows Firewall | Network segmentation |
| Audit logon events | Computer → Security Settings → Audit Policy | Security monitoring/SIEM |
| Restrict software | Computer → Security Settings → AppLocker | Application whitelisting |

### Key Commands

```powershell
# Force GPO refresh immediately
gpupdate /force

# View applied GPOs (Resultant Set of Policy)
gpresult /r             # Summary
gpresult /h report.html # Detailed HTML report

# Backup all GPOs
Backup-GPO -All -Path "C:\GPO-Backups"

# Create and link a GPO
New-GPO -Name "Server-Hardening" | New-GPLink -Target "OU=Servers,DC=corp,DC=com"
```

**DevOps Relevance:**
- GPO = **Windows-native configuration management** (like Ansible playbooks for Windows).
- In cloud migrations, GPOs are often replaced by **Intune policies**, **Azure Policy**, or **AWS SSM State Manager**.
- Understanding GPO helps you design **Desired State Configuration (DSC)** — the modern IaC equivalent.
- **GPO audit logs** feed into SIEM tools — critical for compliance (SOC2, HIPAA).

---

## 4. Name Resolution — DNS Servers, Zones & Records

### Why DNS Matters — It's Not Just Name Resolution

```
  DNS in Active Directory is CRITICAL because:

  ┌──────────────────────────────────────────────────┐
  │  AD DS literally DEPENDS on DNS to function:     │
  │                                                  │
  │  • Clients find DCs via DNS SRV records          │
  │  • Kerberos authentication uses DNS              │
  │  • Replication between DCs uses DNS              │
  │  • Domain join requires DNS                      │
  │  • GPO application requires DNS                  │
  │                                                  │
  │  NO DNS = NO AD = NO AUTHENTICATION = NOTHING    │
  └──────────────────────────────────────────────────┘
```

### DNS Architecture in Windows Server

```
  ┌────────────────────────────────────────────────────────┐
  │                   DNS Server                           │
  │                                                        │
  │  ┌──────────────────┐  ┌──────────────────────────┐    │
  │  │  Forward Lookup  │  │  Reverse Lookup Zone     │    │
  │  │  Zone            │  │                          │    │
  │  │                  │  │  IP → Name               │    │
  │  │  Name → IP       │  │  10.0.1.5 → srv01.corp  │    │
  │  │  srv01 → 10.0.1.5│  │                          │    │
  │  └──────────────────┘  └──────────────────────────┘    │
  │                                                        │
  │  ┌──────────────────────────────────────────────┐      │
  │  │  Zone Types:                                 │      │
  │  │                                              │      │
  │  │  • Primary Zone: R/W, authoritative          │      │
  │  │  • Secondary Zone: Read-only copy            │      │
  │  │  • Stub Zone: Only NS + SOA records          │      │
  │  │  • AD-Integrated: Stored in AD, replicated   │      │
  │  │    with AD replication (RECOMMENDED)         │      │
  │  └──────────────────────────────────────────────┘      │
  └────────────────────────────────────────────────────────┘
```

### DNS Record Types — Quick Reference

| Record | Purpose | Example | DevOps Usage |
|--------|---------|---------|-------------|
| **A** | Name → IPv4 | `web01 → 10.0.1.10` | Point domain to server |
| **AAAA** | Name → IPv6 | `web01 → 2001:db8::1` | IPv6 environments |
| **CNAME** | Alias → Canonical name | `www → web01.corp.com` | Blue/Green deployments — swap CNAME |
| **MX** | Mail exchange | `corp.com → mail.corp.com` | Email routing |
| **NS** | Nameserver for zone | `corp.com → ns1.corp.com` | DNS delegation |
| **SOA** | Start of Authority | Zone metadata | Replication serial tracking |
| **SRV** | Service locator | `_ldap._tcp.corp.com → dc01` | **AD depends on this!** DC discovery |
| **PTR** | IP → Name (reverse) | `10.0.1.10 → web01` | Reverse DNS for email/security |
| **TXT** | Text data | SPF, DKIM, DMARC records | Email security, domain verification |

### AD-Integrated DNS vs Standard DNS

```
  Standard DNS:                      AD-Integrated DNS:
  ┌──────────┐                       ┌──────────┐
  │ Primary  │──Zone Transfer──►     │  DC/DNS  │◄──► AD Replication
  │ DNS      │   (AXFR/IXFR)        │  Server  │     (multi-master)
  └──────────┘                       └──────────┘
       │                                  │
  ┌──────────┐                       ┌──────────┐
  │Secondary │                       │  DC/DNS  │
  │ DNS      │  (Read-only)          │  Server  │  (All can write!)
  └──────────┘                       └──────────┘

  AD-Integrated Advantages:
  ✅ Multi-master (any DC can update)
  ✅ Secure dynamic updates (only authenticated)
  ✅ Replicated with AD (no separate zone transfer)
  ✅ Better security (AD ACLs on records)
```

### Conditional Forwarders & Forwarding

```
  ┌────────────────────────────────────────────────┐
  │  Your DNS Server receives query for:           │
  │                                                │
  │  "api.partner.com" (not your zone)             │
  │                                                │
  │  Option 1: Forwarder (send ALL unknown queries)│
  │  ──────► 8.8.8.8 (Google DNS)                  │
  │                                                │
  │  Option 2: Conditional Forwarder               │
  │  IF query ends in "partner.com"                │
  │  ──────► 10.50.1.1 (partner's DNS)             │
  │                                                │
  │  Option 3: Root Hints (iterative resolution)   │
  │  ──────► Root servers → TLD → authoritative    │
  └────────────────────────────────────────────────┘
```

**DevOps Relevance:**
- **DNS is the #1 debugging skill** — most connectivity issues trace back to DNS.
- **Service Discovery:** Kubernetes DNS, Consul, cloud DNS — all build on these fundamentals.
- **CNAME swapping** for blue/green deployments.
- **Private DNS zones** in Azure/AWS = same concept as AD-Integrated zones.
- **Conditional forwarding** = hybrid cloud DNS resolution (on-prem ↔ cloud).

---

## 5. IP Management — DHCP Servers & Failover

### What Is DHCP?

DHCP automatically assigns IP addresses, subnet masks, gateways, and DNS servers to clients. Without it, you'd manually configure every device.

### DHCP Lease Process — DORA

```
  Client                          DHCP Server
    │                                  │
    │  1. DISCOVER (broadcast)         │
    │  "Anyone out there with an IP?"  │
    ├─────────────────────────────────►│
    │                                  │
    │  2. OFFER                        │
    │  "Here's 10.0.1.50 for you"     │
    │◄─────────────────────────────────┤
    │                                  │
    │  3. REQUEST                      │
    │  "I'll take 10.0.1.50 please"   │
    ├─────────────────────────────────►│
    │                                  │
    │  4. ACKNOWLEDGE                  │
    │  "Confirmed. It's yours for 8h" │
    │◄─────────────────────────────────┤
    │                                  │

  Lease Renewal:
  • At 50% lease time: Client tries to renew with same server
  • At 87.5% lease time: Client broadcasts for ANY server
  • At 100%: Lease expires, full DORA again
```

### DHCP Scope, Options & Reservations

```
  ┌─────────────────────────────────────────────────┐
  │  DHCP Scope: 10.0.1.0/24                       │
  │                                                 │
  │  Range:    10.0.1.100 - 10.0.1.200 (available) │
  │  Excluded: 10.0.1.1   - 10.0.1.99  (static)   │
  │                                                 │
  │  Options (pushed to clients):                   │
  │  ├── Option 003: Default Gateway (10.0.1.1)    │
  │  ├── Option 006: DNS Servers (10.0.1.10, .11)  │
  │  ├── Option 015: Domain Name (corp.com)        │
  │  └── Option 066: Boot Server (for PXE/WDS)     │
  │                                                 │
  │  Reservations (fixed IP by MAC):                │
  │  ├── AA:BB:CC:11:22:33 → 10.0.1.50 (printer)  │
  │  └── DD:EE:FF:44:55:66 → 10.0.1.51 (server)   │
  └─────────────────────────────────────────────────┘
```

### DHCP Failover — Two Modes

```
  Mode 1: HOT STANDBY
  ┌──────────┐                    ┌──────────┐
  │  DHCP    │  ──── Active ───►  │  Clients │
  │  Primary │                    │          │
  └──────────┘                    └──────────┘
  ┌──────────┐                         │
  │  DHCP    │  ◄── Standby (5% pool)──┘
  │  Secondary│     Takes over on failure
  └──────────┘

  Mode 2: LOAD BALANCE (50/50 or custom split)
  ┌──────────┐  ──── 50% ────►  ┌──────────┐
  │  DHCP    │                   │  Clients │
  │  Server A│  ◄── 50% ────    │          │
  └──────────┘                   └──────────┘
  ┌──────────┐  ──── 50% ────►       │
  │  DHCP    │                        │
  │  Server B│  ◄── 50% ─────────────┘
  └──────────┘
  Both servers share the scope, replicate lease info
```

**DevOps Relevance:**
- **PXE Boot (Option 066/067)** is used for automated bare-metal provisioning (MAAS, Cobbler, WDS).
- **DHCP Reservations** = predictable IPs for infrastructure without static config — useful in automation.
- Cloud equivalent: **VPC DHCP option sets** (AWS), **VNET DNS settings** (Azure) — same concepts.
- **IPAM (IP Address Management)** in Windows Server = centralized IP tracking — similar to cloud IP management.

---

## 6. Network Security — DMZ, RAS & NPS

### 6.1 DMZ (Demilitarized Zone)

#### Mental Model

```
  ┌──────────────────────────────────────────────────────────────┐
  │  INTERNET (Untrusted)                                       │
  └──────────┬───────────────────────────────────────────────────┘
             │
       ┌─────▼─────┐
       │ Outer     │  ← Only allows HTTP/S, SMTP, DNS
       │ Firewall  │
       └─────┬─────┘
             │
  ┌──────────▼────────────────────────────────────────┐
  │  DMZ (Semi-Trusted)                               │
  │                                                    │
  │  🌐 Web Server    📧 Mail Relay    🔒 Reverse Proxy │
  │  🛡️ WAF           📋 VPN Gateway                   │
  │                                                    │
  │  These servers are EXPENDABLE — they contain       │
  │  NO sensitive data. They proxy requests inward.    │
  └──────────┬────────────────────────────────────────┘
             │
       ┌─────▼─────┐
       │ Inner     │  ← Only allows specific ports from DMZ
       │ Firewall  │
       └─────┬─────┘
             │
  ┌──────────▼────────────────────────────────────────┐
  │  INTERNAL NETWORK (Trusted)                       │
  │                                                    │
  │  🗄️ Database    🔐 Domain Controllers    📁 File   │
  │  💼 App Servers  🏢 Internal Services    Servers   │
  └───────────────────────────────────────────────────┘
```

#### Why DMZ?

- **Defense in depth** — compromising the DMZ doesn't expose internal network.
- **Compliance** — PCI-DSS, HIPAA require network segmentation.
- **Cloud equivalent:** **Public subnet vs Private subnet** in VPC architecture is the same concept.

---

### 6.2 Remote Access Service (RAS)

#### What Is It?

RAS provides **VPN and DirectAccess** connectivity for remote users to reach internal resources.

#### Architecture

```
  Remote User                    Corporate Network
  ┌──────────┐                   ┌────────────────────────┐
  │ Laptop   │                   │                        │
  │          │ ════VPN Tunnel════►│  RAS/VPN Server       │
  │          │  (encrypted)      │  (in DMZ or edge)     │
  └──────────┘                   │         │              │
                                 │    ┌────▼────┐         │
                                 │    │ NPS     │         │
                                 │    │(RADIUS) │         │
                                 │    └────┬────┘         │
                                 │         │ Authenticate │
                                 │    ┌────▼────┐         │
                                 │    │  AD DS  │         │
                                 │    │  (DC)   │         │
                                 │    └─────────┘         │
                                 └────────────────────────┘
```

#### VPN Protocols in Windows Server

| Protocol | Security | Speed | Port | Notes |
|----------|----------|-------|------|-------|
| **SSTP** | High (SSL/TLS) | Good | 443 | Firewall-friendly, Windows-native |
| **IKEv2** | High (IPSec) | Best | 500/4500 | Best for mobile, auto-reconnect |
| **L2TP/IPSec** | High | Moderate | 1701/500 | Double encapsulation overhead |
| **PPTP** | ❌ Weak | Fast | 1723 | DEPRECATED — never use |

---

### 6.3 Network Policy Server (NPS) — RADIUS

#### What Is It?

NPS is Microsoft's **RADIUS server** implementation. It acts as the **central authentication and authorization gatekeeper** for network access.

#### Mental Model

```
  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
  │  VPN Client  ├────►│  RAS/VPN     ├────►│  NPS         │
  │              │     │  Server      │     │  (RADIUS)    │
  └──────────────┘     │ (RADIUS      │     │              │
                       │  Client)     │     │  Checks:     │
  ┌──────────────┐     └──────────────┘     │  • Who?      │
  │  WiFi Client ├────►┌──────────────┐     │    (AD auth) │
  │              │     │  WiFi AP     ├────►│  • Allowed?  │
  └──────────────┘     │ (RADIUS      │     │    (policy)  │
                       │  Client)     │     │  • How?      │
  ┌──────────────┐     └──────────────┘     │    (health)  │
  │  802.1x      ├────►┌──────────────┐     │              │
  │  Wired Client│     │  Switch      ├────►│  Responds:   │
  └──────────────┘     │ (RADIUS      │     │  Accept/     │
                       │  Client)     │     │  Reject      │
                       └──────────────┘     └──────┬───────┘
                                                   │
                                            ┌──────▼───────┐
                                            │  Active      │
                                            │  Directory   │
                                            └──────────────┘
```

#### NPS Policy Types

| Policy Type | Purpose |
|-------------|---------|
| **Connection Request Policy** | Determines WHERE to authenticate (local NPS or forward to another RADIUS) |
| **Network Policy** | Determines IF the user is ALLOWED and under WHAT CONDITIONS |
| **Health Policy** | NAP (deprecated) — checked OS patch level, antivirus status |

**DevOps Relevance:**
- **VPN/RAS** = how you securely access on-prem from cloud CI/CD agents or remote work.
- **RADIUS/NPS** concepts map to **AAA (Authentication, Authorization, Accounting)** in cloud — AWS IAM, Azure AD Conditional Access.
- **Site-to-Site VPN** between on-prem and cloud is configured using these same RAS concepts.
- **Zero Trust replaces traditional VPN** — but understanding VPN is essential for migration planning.

---

## 7. File Services — Sharing, DFS, BranchCache, FSRM

### 7.1 File Sharing & Access Management

#### Architecture

```
  ┌─────────────────────────────────────────────┐
  │  File Server (\\SRV-FILE01)                 │
  │                                              │
  │  📁 D:\Shares\                               │
  │  ├── 📂 Finance$     (hidden share — $)      │
  │  │   ├── NTFS Perms: Finance-Group: Modify  │
  │  │   └── Share Perms: Finance-Group: Change │
  │  │                                          │
  │  ├── 📂 Public                               │
  │  │   ├── NTFS Perms: Everyone: Read         │
  │  │   └── Share Perms: Everyone: Read        │
  │  │                                          │
  │  └── 📂 IT-Tools                             │
  │      ├── NTFS Perms: IT-Admins: Full Control│
  │      └── Share Perms: IT-Admins: Full Ctrl  │
  │                                              │
  │  Access via: \\SRV-FILE01\Finance$           │
  │  or mapped: net use F: \\SRV-FILE01\Finance$ │
  └─────────────────────────────────────────────┘

  ⚠️ EFFECTIVE PERMISSION = Most restrictive of (NTFS ∩ Share)
```

---

### 7.2 DFS (Distributed File System)

#### Two Components

```
  ┌─────────────────────────────────────────────────────────┐
  │  DFS Namespaces (DFS-N)                                 │
  │  "Virtual unified path to distributed shares"           │
  │                                                         │
  │  Users see:    \\corp.com\shares\finance                │
  │  Actually:     \\SRV-FILE01\finance-data                │
  │         or:    \\SRV-FILE02\finance-data  (failover!)  │
  │                                                         │
  │  ┌─────────────────┐    ┌──────────────────────┐        │
  │  │  Namespace Root │    │ \\corp.com\shares     │        │
  │  └────────┬────────┘    └──────────────────────┘        │
  │           ├── 📂 finance ──► \\SRV-FILE01\finance       │
  │           │                  \\SRV-FILE02\finance (repl)│
  │           ├── 📂 hr ──────► \\SRV-FILE03\hr-docs        │
  │           └── 📂 it ──────► \\SRV-FILE01\it-tools       │
  └─────────────────────────────────────────────────────────┘

  ┌─────────────────────────────────────────────────────────┐
  │  DFS Replication (DFS-R)                                │
  │  "Automatic file sync between servers"                  │
  │                                                         │
  │  ┌──────────┐     Replication     ┌──────────┐          │
  │  │  SRV-01  │◄═══════════════════►│  SRV-02  │          │
  │  │  Site: HQ│     (RDC - only    │ Site:     │          │
  │  │          │      changed blocks)│ Branch   │          │
  │  └──────────┘                     └──────────┘          │
  │                                                         │
  │  Uses Remote Differential Compression (RDC)             │
  │  Only replicates CHANGED PORTIONS of files              │
  └─────────────────────────────────────────────────────────┘
```

**DevOps Relevance:**
- DFS Namespaces = **abstraction layer** over physical storage — like **mount points, NFS exports, or S3 bucket aliases**.
- DFS-R = **file-level replication** — like **rsync, AWS S3 Cross-Region Replication**.
- In cloud migrations, DFS is often replaced by **Azure Files + Azure File Sync** or **AWS FSx**.

---

### 7.3 BranchCache

#### The Problem It Solves

```
  WITHOUT BranchCache:
  ┌───────────┐                          ┌──────────┐
  │ HQ File   │◄── WAN (slow, costly) ──►│ Branch   │
  │ Server    │   Every user downloads   │ Office   │
  │           │   the same file          │ 20 users │
  │           │   20 times!              │          │
  └───────────┘                          └──────────┘

  WITH BranchCache:
  ┌───────────┐                          ┌──────────────────┐
  │ HQ File   │◄── WAN (1 download) ───►│ Branch Office    │
  │ Server    │                          │                  │
  │           │                          │ User1 downloads  │
  └───────────┘                          │ → cached locally │
                                         │                  │
                                         │ User2-20 get it  │
                                         │ from LOCAL cache! │
                                         └──────────────────┘
```

#### Two Modes

| Mode | How It Works | Best For |
|------|-------------|----------|
| **Distributed Cache** | Clients cache and serve to each other (P2P) | Small offices (<50 PCs), no local server |
| **Hosted Cache** | Dedicated cache server at branch | Large branches, reliable local server |

**DevOps Relevance:**
- Same concept as **CDN (Content Delivery Network)** — cache content closer to users.
- Maps to **AWS CloudFront, Azure CDN, pull-through caches**.
- Container registries use similar patterns — **registry mirrors/pull-through caches**.

---

### 7.4 File Server Resource Manager (FSRM)

#### What Is It?

FSRM is a **governance and quota engine** for file servers — it controls what, how much, and monitors storage usage.

#### Capabilities

```
  ┌─────────────────────────────────────────────────┐
  │  FSRM Features:                                 │
  │                                                  │
  │  📊 Quotas                                       │
  │  ├── Hard Quota: BLOCK writes at limit          │
  │  └── Soft Quota: WARN but allow writes          │
  │                                                  │
  │  🚫 File Screening                               │
  │  ├── Active: BLOCK file types (.mp3, .exe)      │
  │  └── Passive: LOG/ALERT but allow               │
  │                                                  │
  │  📋 Storage Reports                               │
  │  ├── Large files, duplicate files                │
  │  ├── Files by owner, least recently accessed     │
  │  └── Quota usage reports                         │
  │                                                  │
  │  📁 File Classification                           │
  │  ├── Auto-classify files (PII, confidential)    │
  │  └── Apply policies based on classification     │
  │                                                  │
  │  🔧 File Management Tasks                        │
  │  └── Auto-expire/move files based on criteria   │
  └─────────────────────────────────────────────────┘
```

**DevOps Relevance:**
- **Quotas** = same as cloud storage quotas (S3 lifecycle policies, Azure Blob tiering).
- **File Screening** = preventing crypto-mining executables, ransomware file types.
- **Classification** maps to **data governance tools** (AWS Macie, Azure Purview).

---

## 8. Storage — Disks, Volumes, Pools, Replicas, File Systems

### 8.1 Disks & Volumes

#### Disk Types

```
  ┌─────────────────────────────────────────────────────────┐
  │  BASIC DISK                    │  DYNAMIC DISK          │
  │                                │                        │
  │  • Traditional partitions      │  • Volumes (flexible)  │
  │  • MBR: max 4 primary parts   │  • Spanned volumes     │
  │  • GPT: up to 128 partitions  │  • Striped volumes     │
  │  • Simple, most common        │  • Mirrored volumes    │
  │  • OS & data drives           │  • RAID-5 volumes      │
  │                                │  • Can be extended     │
  │  ⚠️ MBR: max 2TB disk          │    without reboot      │
  │  ✅ GPT: max 18 EB (exabytes)  │                        │
  └─────────────────────────────────────────────────────────┘
```

### 8.2 Storage Spaces (Storage Pools)

#### Mental Model

```
  Physical Disks → Storage Pool → Virtual Disks → Volumes

  ┌──────────────────────────────────────────────────┐
  │  STORAGE POOL                                    │
  │  (Aggregated from physical disks)                │
  │                                                  │
  │  💽 Disk1   💽 Disk2   💽 Disk3   💽 Disk4        │
  │  (1TB)     (1TB)     (2TB)     (2TB)            │
  │  ═══════════════════════════════════             │
  │  Total Pool: ~6TB raw                           │
  │                                                  │
  │  Virtual Disks carved from pool:                │
  │                                                  │
  │  ┌───────────────┐  ┌───────────────────────┐   │
  │  │ VDisk1        │  │ VDisk2                │   │
  │  │ Simple (no    │  │ Mirror (2 copies)     │   │
  │  │ redundancy)   │  │ Survives 1 disk fail  │   │
  │  │ 2TB usable    │  │ 1TB usable (2TB raw)  │   │
  │  └───────────────┘  └───────────────────────┘   │
  │                                                  │
  │  Resiliency Types:                              │
  │  • Simple: No redundancy (RAID-0 like)          │
  │  • Mirror: 2-way or 3-way copy (RAID-1 like)   │
  │  • Parity: Distributed parity (RAID-5/6 like)  │
  └──────────────────────────────────────────────────┘
```

### 8.3 Storage Replica

#### What Is It?

Block-level, volume-based replication for **disaster recovery**. Think of it as **synchronous/asynchronous data replication between servers or clusters**.

```
  ┌──────────────┐                    ┌──────────────┐
  │  Server A    │                    │  Server B    │
  │  (Primary)   │                    │  (DR Site)   │
  │              │                    │              │
  │  ┌────────┐  │   Block-level     │  ┌────────┐  │
  │  │ Data   │  │   Replication     │  │ Data   │  │
  │  │ Volume │══╪════════════════════╪══│ Volume │  │
  │  └────────┘  │                    │  └────────┘  │
  │  ┌────────┐  │                    │  ┌────────┐  │
  │  │  Log   │  │   (writes go to   │  │  Log   │  │
  │  │ Volume │══╪══ log first)  ═════╪══│ Volume │  │
  │  └────────┘  │                    │  └────────┘  │
  └──────────────┘                    └──────────────┘

  Modes:
  • Synchronous:  Zero data loss, higher latency (same site/metro)
  • Asynchronous: Possible data loss, lower latency (cross-region)
```

### 8.4 File Systems — NTFS vs ReFS

| Feature | NTFS | ReFS |
|---------|------|------|
| **Full Name** | New Technology File System | Resilient File System |
| **Boot Support** | ✅ Yes | ❌ No (data volumes only) |
| **Max Volume Size** | 256 TB | 35 PB (petabytes) |
| **Data Integrity** | Basic | Checksums, auto-repair |
| **Compression** | ✅ Yes | ❌ No |
| **Encryption (EFS)** | ✅ Yes | ❌ No |
| **Disk Quotas** | ✅ Yes | ❌ No |
| **Deduplication** | ✅ Yes | ✅ Yes |
| **Best For** | OS drives, general use | Hyper-V, Storage Spaces, large data |
| **File Permissions** | Full ACL support | Full ACL support |

**DevOps Relevance:**
- **Storage Spaces** = Software-defined storage — cloud equivalent: **AWS EBS, Azure Managed Disks, Ceph**.
- **Storage Replica** = **AWS cross-region EBS replication, Azure Site Recovery, DRBD**.
- **ReFS** is designed for **Hyper-V workloads** — instant VM checkpoint creation, data integrity.
- Understanding **block vs file vs object storage** fundamentals starts here.

---

## 9. Permissions — NTFS vs Share Permissions

### The Two Permission Layers

```
  ┌───────────────────────────────────────────────────┐
  │  Network Access to a Shared Folder                │
  │                                                    │
  │  User: \\server\share\file.docx                   │
  │                                                    │
  │  ┌─────────────────┐   ┌─────────────────────┐    │
  │  │ SHARE Permissions│   │ NTFS Permissions    │    │
  │  │ (Network layer) │   │ (File system layer) │    │
  │  │                 │   │                     │    │
  │  │ • Only apply    │   │ • Always apply      │    │
  │  │   over NETWORK  │   │   (local + network) │    │
  │  │ • Coarse-grained│   │ • Fine-grained      │    │
  │  │   (Read, Change,│   │   (Read, Write,     │    │
  │  │    Full Control)│   │    Execute, Modify,  │    │
  │  │ • Folder-level  │   │    Full Control,     │    │
  │  │   only          │   │    Special perms)    │    │
  │  │                 │   │ • Per file & folder  │    │
  │  └────────┬────────┘   └────────┬────────────┘    │
  │           │                     │                  │
  │           └──────────┬──────────┘                  │
  │                      │                             │
  │           EFFECTIVE PERMISSION =                   │
  │           Most Restrictive of Both                 │
  └───────────────────────────────────────────────────┘
```

### Permission Evaluation Examples

```
  Scenario 1:
  Share Permission: Read
  NTFS Permission:  Full Control
  ──────────────────────────────
  Effective (over network): READ  ← Share is more restrictive

  Scenario 2:
  Share Permission: Full Control
  NTFS Permission:  Read
  ──────────────────────────────
  Effective (over network): READ  ← NTFS is more restrictive

  Scenario 3:
  Share Permission: Change (R/W)
  NTFS Permission:  Modify
  ──────────────────────────────
  Effective (over network): Modify  ← Both are similar

  ⚠️ Best Practice:
  Set Share Permissions to "Full Control" for Authenticated Users
  Use NTFS Permissions for granular access control
  (This simplifies management — one layer to manage)
```

### NTFS Permission Inheritance

```
  📂 D:\Data                   Full Control: IT-Admins
  ├── 📂 Projects              Inherited: Full Control: IT-Admins
  │   ├── 📂 ProjectAlpha      Inherited + Read: Dev-Team
  │   │   └── 📄 spec.docx     Inherited from ProjectAlpha
  │   └── 📂 ProjectBeta       Inherited (inheritance BLOCKED here)
  │       └── 📄 secret.docx   Only explicit permissions apply
  └── 📂 Archive               Inherited + Modify: Archive-Team

  Key Concepts:
  • Explicit permissions OVERRIDE inherited
  • Deny ALWAYS overrides Allow
  • Inheritance can be blocked at any folder
  • When blocked: copy inherited or remove
```

**DevOps Relevance:**
- Maps directly to **IAM policies** — Deny overrides Allow, most restrictive wins.
- Understanding **permission inheritance** = understanding **AWS S3 bucket policies, Azure RBAC scope inheritance**.
- **Principle of least privilege** — give minimum necessary permissions.
- You'll automate permissions with PowerShell: `icacls`, `Set-Acl`, `Get-Acl`.

---

## 10. Automation — PowerShell ISE

### What Is ISE?

**Integrated Scripting Environment** — a built-in PowerShell IDE for writing, testing, and debugging scripts.

```
  ┌──────────────────────────────────────────────────────┐
  │  PowerShell ISE                                      │
  │  ┌──────────────────────────────────────────┐        │
  │  │  Script Pane (write/edit .ps1 files)     │        │
  │  │  ─────────────────────────────────────   │        │
  │  │  Get-ADUser -Filter * |                  │        │
  │  │    Select Name, Enabled |                │        │
  │  │    Export-Csv users.csv                   │        │
  │  └──────────────────────────────────────────┘        │
  │  ┌──────────────────────────────────────────┐        │
  │  │  Console Pane (interactive execution)    │        │
  │  │  PS C:\> _                               │        │
  │  └──────────────────────────────────────────┘        │
  │  ┌──────────────────────────────────────────┐        │
  │  │  Command Add-on (cmdlet explorer)        │        │
  │  └──────────────────────────────────────────┘        │
  │                                                      │
  │  Features: IntelliSense, Breakpoints, Debugger,      │
  │  Tab completion, Snippet library, Multi-tab editing   │
  └──────────────────────────────────────────────────────┘
```

> ⚠️ **ISE is considered legacy.** Microsoft recommends **VS Code + PowerShell Extension** for modern development. However, ISE is still built into Windows Server and commonly used for quick scripting.

### Essential PowerShell Commands for Windows Server (DevOps Focus)

```powershell
# ── Active Directory ──
Get-ADUser -Filter * -Properties * | Select Name, Enabled, LastLogonDate
New-ADUser -Name "svc-deploy" -AccountPassword (ConvertTo-SecureString "P@ss" -AsPlainText -Force) -Enabled $true
Get-ADGroup -Filter * | Select Name, GroupScope, GroupCategory
Add-ADGroupMember -Identity "Deploy-Agents" -Members "svc-deploy"
Get-ADComputer -Filter * | Select Name, OperatingSystem, IPv4Address

# ── DNS ──
Get-DnsServerZone
Add-DnsServerResourceRecordA -ZoneName "corp.com" -Name "app01" -IPv4Address "10.0.1.100"
Get-DnsServerResourceRecord -ZoneName "corp.com" -RRType A

# ── DHCP ──
Get-DhcpServerv4Scope
Add-DhcpServerv4Reservation -ScopeId 10.0.1.0 -IPAddress 10.0.1.50 -ClientId "AA-BB-CC-DD-EE-FF"

# ── GPO ──
Get-GPO -All | Select DisplayName, GpoStatus
New-GPO -Name "Security-Baseline" | New-GPLink -Target "OU=Servers,DC=corp,DC=com"
Backup-GPO -All -Path "\\backup\gpo"

# ── File & Share ──
New-SmbShare -Name "Deploy" -Path "D:\Deploy" -FullAccess "Administrators" -ReadAccess "Deploy-Agents"
Get-SmbShareAccess -Name "Deploy"
Get-Acl "D:\Deploy" | Format-List

# ── Storage ──
Get-Disk | Select Number, Size, PartitionStyle
Initialize-Disk -Number 1 -PartitionStyle GPT
New-Partition -DiskNumber 1 -UseMaximumSize -AssignDriveLetter | Format-Volume -FileSystem NTFS

# ── Server Info ──
Get-WindowsFeature | Where Installed
Install-WindowsFeature -Name "DHCP" -IncludeManagementTools
Get-Service | Where Status -eq "Running"
```

**DevOps Relevance:**
- PowerShell is the **scripting backbone** for Windows automation — CI/CD, DSC, Azure PowerShell.
- **PowerShell Remoting (WinRM)** = Ansible's connection method to Windows.
- **PowerShell DSC** (Desired State Configuration) = IaC for Windows (like Puppet/Chef).
- All Azure, AWS, and GCP CLIs have PowerShell modules.

---

## 11. X vs Y — Head-to-Head Comparisons

### 11.1 DC vs RODC

| Criteria | DC | RODC |
|----------|-----|------|
| Writable | ✅ Yes | ❌ No |
| Password Storage | All | Only cached (PRP) |
| Deployment Location | Secure data center | Branch office, DMZ |
| If Compromised | Full AD exposure | Limited exposure |
| Replication | Bi-directional | Inbound-only |
| **Cloud Analogy** | Primary DB | Read Replica |

### 11.2 GPO vs PowerShell DSC

| Criteria | GPO | PowerShell DSC |
|----------|-----|----------------|
| Scope | AD-joined machines only | Any Windows/Linux machine |
| Requires AD | ✅ Yes | ❌ No |
| Drift Detection | Reapplied every 90min | Continuous monitoring |
| Idempotent | Partially | ✅ Fully |
| Cross-Platform | ❌ Windows only | ✅ Windows + Linux |
| Cloud-Native | ❌ | ✅ Azure Automation DSC |
| **When to Use** | Domain-joined policy enforcement | IaC for any infrastructure |

### 11.3 NTFS vs Share Permissions

| Criteria | NTFS | Share |
|----------|------|-------|
| Applies When | Always (local + network) | Network access only |
| Granularity | Fine (Read, Write, Execute, Modify, Special) | Coarse (Read, Change, Full) |
| Per-File Control | ✅ Yes | ❌ Folder-level only |
| Inheritance | ✅ Yes | ❌ No |
| **Best Practice** | Use for ALL access control | Set to Full Control, let NTFS handle it |

### 11.4 DFS-N vs DFS-R

| Criteria | DFS Namespaces | DFS Replication |
|----------|---------------|-----------------|
| Purpose | Unified virtual path | Data synchronization |
| What It Does | Redirects to actual share locations | Copies files between servers |
| Redundancy | Location transparency + failover | Data redundancy |
| Analogy | DNS CNAME for file shares | rsync / S3 replication |
| **Together** | Often used together — DFS-N points to DFS-R synced shares |

### 11.5 BranchCache Distributed vs Hosted

| Criteria | Distributed Cache | Hosted Cache |
|----------|-------------------|--------------|
| Cache Location | Client PCs (P2P) | Dedicated server at branch |
| Branch Server Required | ❌ No | ✅ Yes |
| Best For | Small offices (<50) | Larger branches |
| Persistence | Lost when PCs off | Persistent on server |
| **Cloud Analogy** | P2P CDN | Edge cache / PoP |

### 11.6 Storage Spaces vs Traditional RAID

| Criteria | Storage Spaces | Hardware RAID |
|----------|---------------|---------------|
| Controller | Software (Windows) | Hardware RAID card |
| Flexibility | Mix disk sizes | Same size recommended |
| Hot Add | ✅ Yes | Depends on controller |
| Cost | ✅ Free (built-in) | 💰 RAID card + battery |
| Thin Provisioning | ✅ Yes | Rarely |
| **Cloud Analogy** | Ceph, AWS EBS, Azure Disks | Bare metal RAID |

### 11.7 NTFS vs ReFS

| Criteria | NTFS | ReFS |
|----------|------|------|
| Boot Volume | ✅ Yes | ❌ No |
| Data Integrity | Basic | Checksums + auto-repair |
| Compression | ✅ | ❌ |
| Encryption (EFS) | ✅ | ❌ |
| Max Volume | 256 TB | 35 PB |
| Best For | General purpose, OS | Hyper-V, Storage Spaces Direct |

### 11.8 Synchronous vs Asynchronous Storage Replica

| Criteria | Synchronous | Asynchronous |
|----------|-------------|--------------|
| Data Loss | Zero (RPO=0) | Possible (RPO>0) |
| Latency Impact | Higher (waits for ack) | Lower |
| Distance | Same site / metro (<5ms RTT) | Cross-region |
| Use Case | Critical databases | DR across geographies |
| **Cloud Analogy** | Multi-AZ replication | Cross-region replication |

### 11.9 Forward Lookup Zone vs Reverse Lookup Zone

| Criteria | Forward | Reverse |
|----------|---------|---------|
| Resolves | Name → IP | IP → Name |
| Record Type | A, AAAA, CNAME | PTR |
| Required for AD | ✅ Critical | Recommended |
| Use Case | "What IP is srv01?" | "Who owns 10.0.1.5?" |
| **DevOps Use** | Service discovery | Security auditing, email SPF |

---

## 12. Viva / Trainee Review — Q&A Bank

### Identity & AD DS

**Q: What is Active Directory and why is it important?**  
A: AD DS is the centralized identity and authentication service for Windows environments. It stores user, computer, and service objects and handles authentication (Kerberos/NTLM) and authorization. It's the single source of truth for identity in on-prem and hybrid-cloud architectures.

**Q: What's the difference between a Forest, Tree, and Domain?**  
A: A **Forest** is the top-level security boundary (the whole organization). A **Tree** is a hierarchy of domains sharing a contiguous namespace (e.g., `corp.com → dev.corp.com`). A **Domain** is the administrative boundary where objects (users, computers) live and share a common database and policies.

**Q: When would you deploy an RODC instead of a full DC?**  
A: In branch offices with limited physical security, DMZs, or edge locations. RODC holds a read-only copy of AD, caches only specified passwords, and if compromised, the damage is limited. The attacker can't modify AD.

**Q: Can you explain the FSMO roles? Which one is most critical for daily operations?**  
A: There are 5 FSMO roles — 2 forest-wide (Schema Master, Domain Naming Master) and 3 domain-wide (RID Master, PDC Emulator, Infrastructure Master). The **PDC Emulator** is most critical — it handles time sync (Kerberos depends on it), password changes, account lockouts, and GPO editing.

**Q: What's the difference between transferring and seizing a FSMO role?**  
A: **Transfer** is a graceful operation when both old and new DC are online — the role is properly handed off. **Seize** is a forced takeover when the old DC is permanently offline — the old DC must NEVER come back online, or you'll have conflicts.

---

### Groups & Service Accounts

**Q: Explain the AGDLP strategy.**  
A: Accounts → Global Groups → Domain Local Groups → Permissions. Users go into Global Groups (by role), Global Groups go into Domain Local Groups (by resource), and permissions are assigned to Domain Local Groups. This creates a clean, scalable, RBAC-like structure.

**Q: What problem does gMSA solve?**  
A: Traditional service accounts have static passwords that are never rotated, shared in scripts, and break services when expired. gMSA provides automatic 240-character password rotation every 30 days, managed by AD — no human ever knows the password. Multiple servers can share one gMSA.

**Q: Can you use gMSA for a scheduled task? How?**  
A: Yes. After installing the gMSA on the server (`Install-ADServiceAccount`), you configure the scheduled task to run under the gMSA. The server retrieves the password automatically from AD.

---

### DNS

**Q: Why is DNS critical for Active Directory?**  
A: AD DS is completely dependent on DNS. Clients find Domain Controllers through DNS SRV records (`_ldap._tcp.dc._msdcs.corp.com`). Without DNS, nothing works — no authentication, no replication, no GPO application, no domain joining.

**Q: What's the difference between AD-Integrated DNS and standard DNS zones?**  
A: AD-Integrated stores zone data in Active Directory itself, gets multi-master replication (any DC can update), supports secure dynamic updates, and uses AD's security (ACLs). Standard DNS uses primary/secondary zone transfers (AXFR/IXFR) and has a single writable primary.

**Q: What is a Conditional Forwarder and when would you use it?**  
A: A Conditional Forwarder sends DNS queries for a specific domain to a designated DNS server instead of using root hints. Used in hybrid scenarios — e.g., forward `*.azure.internal` to Azure's DNS, or `*.partner.com` to a partner's DNS server. Essential for hybrid cloud DNS resolution.

**Q: What DNS records does AD create automatically?**  
A: SRV records for LDAP (`_ldap._tcp`), Kerberos (`_kerberos._tcp`), Global Catalog (`_gc._tcp`), and site-specific records. Also A records for DCs. These are registered via secure dynamic DNS updates.

---

### DHCP

**Q: Explain the DORA process.**  
A: **D**iscover (client broadcasts for a DHCP server) → **O**ffer (server offers an IP) → **R**equest (client accepts the offer) → **A**cknowledge (server confirms the lease). All initial communication is broadcast because the client has no IP yet.

**Q: What are the two DHCP failover modes?**  
A: **Hot Standby** (active/passive — one server handles all, the other takes over on failure) and **Load Balance** (both servers share the load, typically 50/50). Load Balance provides better resource utilization; Hot Standby is simpler.

**Q: What's the difference between a DHCP Exclusion and a Reservation?**  
A: An **Exclusion** removes IPs from the DHCP pool entirely (for static devices not managed by DHCP). A **Reservation** guarantees a specific IP to a specific MAC address, but the device still uses DHCP to get its address — it's just always the same IP.

---

### GPO

**Q: What is LSDOU and why does order matter?**  
A: LSDOU = Local → Site → Domain → OU. This is the order GPOs are applied. Later-applied GPOs override earlier ones, so OU-level GPOs have the highest priority. This allows general policies at the domain level with specific overrides at the OU level.

**Q: What's the difference between "Enforced" and "Block Inheritance"?**  
A: **Block Inheritance** on an OU stops it from receiving parent GPOs. **Enforced** on a GPO means it CANNOT be blocked — it overrides Block Inheritance. Enforced always wins.

**Q: How often are GPOs refreshed?**  
A: Computer Configuration: at boot + every 90-120 minutes. User Configuration: at logon + every 90-120 minutes. Domain Controllers: every 5 minutes. `gpupdate /force` forces immediate refresh.

**Q: How would you troubleshoot a GPO not applying?**  
A: 1) `gpresult /r` to see which GPOs are applied. 2) Check security filtering — does the user/computer have "Apply" permission? 3) Check WMI filters. 4) Check inheritance — is it blocked? 5) Check link order and precedence. 6) `rsop.msc` for detailed analysis. 7) Event Viewer → Group Policy operational log.

---

### File Services & Storage

**Q: Why would you use DFS Namespaces?**  
A: To provide a single, unified path (e.g., `\\corp.com\shares`) that maps to file servers across multiple locations. Users don't need to know which physical server holds their files. It also enables transparent failover if a server goes down.

**Q: How does BranchCache save bandwidth?**  
A: When the first user at a branch office downloads a file from HQ, it's cached locally (either on their PC in Distributed mode, or on a local server in Hosted mode). All subsequent users at that branch get the file from the local cache instead of re-downloading over the WAN.

**Q: What's the effective permission when Share says "Read" and NTFS says "Full Control"?**  
A: **Read**. The effective permission over the network is the most restrictive of the two. NTFS Full Control is limited by the Share Read permission.

**Q: When would you choose ReFS over NTFS?**  
A: For Hyper-V workloads (instant VM checkpoints), Storage Spaces Direct (data integrity checksums), and very large data volumes (up to 35 PB). ReFS cannot be used for boot volumes or provide compression/encryption.

**Q: What are the three resiliency types in Storage Spaces?**  
A: **Simple** (no redundancy, best performance — like RAID-0), **Mirror** (2-way or 3-way copy — like RAID-1), and **Parity** (distributed parity — like RAID-5/6, best capacity efficiency with some redundancy).

---

### Network & Security

**Q: What is a DMZ and why is it important?**  
A: A DMZ is a network segment between the public internet and private network, protected by firewalls on both sides. Public-facing services (web servers, mail relays) sit in the DMZ. If compromised, the inner firewall still protects the internal network. It's the same concept as public vs private subnets in cloud VPCs.

**Q: What is NPS and how does it relate to RADIUS?**  
A: NPS is Microsoft's RADIUS server implementation. It centralizes authentication and authorization for network access (VPN, WiFi, wired 802.1x). Network devices (VPN servers, WiFi APs) forward auth requests to NPS, which checks against AD and applies policies to accept or reject.

**Q: Which VPN protocol would you recommend and why?**  
A: **IKEv2** for best performance and auto-reconnect (great for mobile), or **SSTP** when you need to traverse strict firewalls (uses port 443, looks like HTTPS). Never use PPTP — it's cryptographically broken.

---

### Cloud/DevOps Mapping

**Q: How do these Windows Server concepts map to cloud services?**

| On-Prem Concept | Azure Equivalent | AWS Equivalent |
|-----------------|-----------------|----------------|
| AD DS | Azure AD (Entra ID) | AWS Directory Service |
| GPO | Intune / Azure Policy | AWS SSM State Manager |
| DNS Server | Azure DNS / Private DNS | Route 53 |
| DHCP | VNET DHCP (automatic) | VPC DHCP Options |
| File Server + DFS | Azure Files + File Sync | Amazon FSx |
| Storage Spaces | Managed Disks | EBS Volumes |
| Storage Replica | Azure Site Recovery | Cross-region EBS replication |
| BranchCache | Azure CDN | CloudFront |
| NPS/RADIUS | Azure AD Conditional Access | IAM + VPN |
| DMZ | NSG + Public Subnet | Security Groups + Public Subnet |
| gMSA | Managed Identity | IAM Roles for Service Accounts |
| FSRM Quotas | Azure Blob Lifecycle | S3 Lifecycle Policies |
| PowerShell DSC | Azure Automation DSC | AWS OpsWorks / SSM |

---

## Quick Revision Cheat Card

```
  ┌─────────────────────────────────────────────────────┐
  │  🧠 MEMORY AIDS                                    │
  │                                                     │
  │  LSDOU    = Local → Site → Domain → OU (GPO order)  │
  │  DORA     = Discover → Offer → Request → Ack (DHCP) │
  │  AGDLP    = Accounts → Global → DL → Permissions    │
  │  FSMO     = Schema, Domain Naming, RID, PDC, Infra  │
  │             "S.D. R.P.I" — "Somebody Didn't         │
  │              Reboot, Please Investigate"             │
  │  Effective Perm = NTFS ∩ Share (most restrictive)   │
  │  Deny > Allow (always)                              │
  │  gMSA password = 240 chars, 30-day auto-rotate      │
  │  GPO refresh = 90-120 min (5 min for DCs)           │
  │  Kerberos = Max 5 min time skew allowed             │
  │  MBR = max 2TB, 4 partitions                        │
  │  GPT = max 18 EB, 128 partitions                    │
  └─────────────────────────────────────────────────────┘
```

---

> **Last Updated:** 20 Feb 2026  
> **Author:** Vishvam Moliya 
> **Purpose:** DevOps/Cloud Engineer — Windows Server 2022 Revision Sheet

---

[🏠 Home](../README.md) · [Windows](README.md)
