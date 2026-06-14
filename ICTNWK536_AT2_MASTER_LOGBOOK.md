# ICTNWK536 AT2 — Master Project Logbook
## Daydream Travel Agency — Enterprise Communication Solutions
**Student:** Bojie Qiao (Jay Qiao) — Student ID: 450783451
**Assessor:** Lewis Moore
**Unit:** ICTNWK536 — Plan, implement and test enterprise communication solutions
**Organisation:** TAFE Queensland
**Submission deadline:** 12 June 2026
**Submitted:** Yes (both documents submitted prior to deadline)

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Methodology Framework](#2-methodology-framework)
3. [Lab Environment — BOJIEANZAC](#3-lab-environment--bojieanzac)
4. [Client Context — Daydream Travel Agency](#4-client-context--daydream-travel-agency)
5. [AT1 Context — Solution Selection](#5-at1-context--solution-selection)
6. [AT2 Project Scope — Three VMs](#6-at2-project-scope--three-vms)
7. [DNS Infrastructure — daydreamtravel.com Zone](#7-dns-infrastructure--daydreamtravelcom-zone)
8. [BAA-SML-MAIL1 — iRedMail Email Server](#8-baa-sml-mail1--iredmail-email-server)
9. [BAA-SML-WEB1 — WordPress CMS](#9-baa-sml-web1--wordpress-cms)
10. [BAA-SML-NC1 — Nextcloud Collaboration + Talk](#10-baa-sml-nc1--nextcloud-collaboration--talk)
11. [VoIP Decision — Nextcloud Talk vs FreePBX](#11-voip-decision--nextcloud-talk-vs-freepbx)
12. [Architecture Decision Records (ADRs)](#12-architecture-decision-records-adrs)
13. [Cross-VM Issues and Troubleshooting Records](#13-cross-vm-issues-and-troubleshooting-records)
14. [AT2 Assessment Documents](#14-at2-assessment-documents)
15. [Key Learnings and Principles](#15-key-learnings-and-principles)
16. [Chronological Timeline](#16-chronological-timeline)
17. [Outstanding Work](#17-outstanding-work)

---

## 1. Project Overview

### What this project is

Bojie Qiao (external IT contractor, Uptown IT — jay.qiao@uptownit.com.au) was engaged to design, deploy, and document an enterprise communication solution for a fictional client, Daydream Travel Agency (8 staff, 4 offices). The solution was assessed under ICTNWK536 at TAFE Queensland.

The project involved:
- Selecting an open-source communication stack in AT1
- Deploying three VMs in the home lab to instantiate that stack in AT2
- Producing an Implementation Plan and a Test Plan as AT2 written deliverables

### Who the contractor is

Bojie is playing the role of an **external IT contractor** from Uptown IT. Peter Webb (client IT Technician) is the on-site project lead at Daydream Travel. A senior consultant at Uptown IT is available for escalation.

### The solution stack

| Component | Technology | Server |
|---|---|---|
| Email | iRedMail 1.8.2 (Ubuntu 22.04) | BAA-SML-MAIL1 |
| CMS | WordPress 7.0 on LAMP (Ubuntu 22.04) | BAA-SML-WEB1 |
| File sharing + VoIP | Nextcloud 33.0.5snap1 + Talk 23.0.6 (Ubuntu 22.04) | BAA-SML-NC1 |

All three VMs live on the BAA internal switch (172.16.50.0/24), behind pfSense (BAA-SML-PFS1), hosted on Bojie-Mini (SML site of the BOJIEANZAC home lab).

---

## 2. Methodology Framework

All build and design work follows an 8-step engineering methodology defined in METHODOLOGY.md.

```
1. REQUIREMENTS   → What exactly are we trying to solve? What must not change?
2. EVALUATE       → At least two genuinely different options, each honestly assessed
3. DECIDE         → Choose one. Record why. Record what was ruled out. (ADR)
4. IMPLEMENT      → Numbered steps. One at a time. Know why before running.
5. VERIFY         → Positive test (intended works) + Negative test (unintended blocked)
6. TROUBLESHOOT   → SYMPTOM → HYPOTHESIS → TEST → RESULT → FIX → ROOT CAUSE → PREVENTION → RE-VERIFY
7. LOG            → Full logbook entry ready to commit. Assumes you forget everything in 6 months.
8. UNDERSTAND     → Name the concept. Own the knowledge. Can you explain it in plain words?
```

### Why this exists

Most IT problems are caused by either:
- Skipping requirements (building the wrong thing)
- Skipping understand (fixing the symptom but not learning from it)

The framework closes both gaps. The LOG step is not a formality — it is raw material for assessment documents.

### UNDERSTAND steps deferred

The UNDERSTAND reflection step for all three AT2 VMs (MAIL1, WEB1, NC1) was explicitly deferred until after the 12 June submission deadline. Deadline pressure and build-first / document-second strategy were the reason.

---

## 3. Lab Environment — BOJIEANZAC

### Domain

| Item | Value |
|---|---|
| AD domain | bojieanzac.com |
| NetBIOS | BOJIEANZAC |
| Lab prefix | BAA |
| Design model | One forest · one domain · two sites |
| Forest root DC | BAA-BIG-DC1 |

### Physical hosts

| Host | Role | Home LAN IP | Lab leg IP |
|---|---|---|---|
| Bojie-Mini | SML site — all AT2 VMs live here | 192.168.1.135 | 172.16.50.2 (vEthernet BAA internal switch) |
| Bojiemini2 | BIG site | 192.168.1.252 | 172.16.60.2 (vEthernet BOJIEANZAC-Internal) |

Both hosts are workgroup machines — not domain-joined. This is an intentional architecture decision (see docs/02-remote-access-design.md).

### Sites

**SML site — 172.16.50.0/24 (hosted on Bojie-Mini)**

| VM | Role | IP |
|---|---|---|
| BAA-SML-PFS1 | pfSense gateway / firewall / VPN | 172.16.50.1 |
| BAA-SML-DC1 | AD DS / DNS / additional DC | 172.16.50.10 |
| BAA-SML-HA1 | Failover cluster node | 172.16.50.21 |
| BAA-STOR1 | iSCSI storage server | 172.16.50.101 |
| BAA-SML-NETH1 | Rocky Linux 9 / NethServer 8 | 172.16.50.60 |
| BAA-SML-MAIL1 | iRedMail (AT2) | 172.16.50.30 |
| BAA-SML-WEB1 | WordPress (AT2) | 172.16.50.31 |
| BAA-SML-NC1 | Nextcloud + Talk (AT2) | 172.16.50.32 |

**BIG site — 172.16.60.0/24 (hosted on Bojiemini2)**

| VM | Role | IP |
|---|---|---|
| BAA-BIG-PFS1 | pfSense gateway / firewall / VPN | 172.16.60.1 |
| BAA-BIG-DC1 | AD DS / DNS / DHCP / forest root | 172.16.60.10 |
| BAA-BIG-HA1 | Failover cluster node | 172.16.60.21 |

### Site-to-site connectivity

IPsec VPN between BAA-BIG-PFS1 and BAA-SML-PFS1. Confirmed working as of 2026-06-01.

### Remote access

Tailscale. Each physical host is a Tailscale node advertising its lab subnet. CLI and SSH sessions run from Windows Terminal with PowerShell + Start-Transcript for session logging.

### Infrastructure DNS

- BAA-SML-DC1: 172.16.50.10 (primary for SML site)
- BAA-BIG-DC1: 172.16.60.10 (secondary)
- **Critical rule:** pfSense gateway addresses (.1) must never be used as DNS servers for VMs. Domain controller addresses (.10) are correct. This was explicitly caught during VM builds to prevent a hard-to-diagnose future failure.

### Naming convention

`BAA-[SITE]-[ROLE][NUMBER]` — e.g., BAA-SML-MAIL1

---

## 4. Client Context — Daydream Travel Agency

### Staff (8 total)

| Name | Role |
|---|---|
| Sharon Webb | CEO |
| Karen Fells | Office Manager |
| Margret Baker | Receptionist |
| John Kennedy | Senior Sales Rep |
| Mitchel Brown | Sales Rep |
| Coney Smith | Sales Rep |
| Alice Watson | Sales Rep |
| Peter Webb | IT Technician and on-site project lead |

### Business requirements

- Email platform supporting all 8 staff with distribution lists (all staff, sales team, management)
- Company website / content management system for travel packages and promotions
- File sharing and collaboration platform across four offices
- VoIP / internal audio-video calling capability
- Each staff member to have appropriate role-based permissions on all systems

---

## 5. AT1 Context — Solution Selection

AT1 was completed separately. The outcome that matters for AT2:

**Selected stack:** iRedMail + WordPress + Nextcloud (all open-source, self-hosted)

This selection was documented in AT1 and carried forward as the scope of AT2 without re-evaluation. AT2 Task 1 (acquire) was satisfied by this prior decision.

---

## 6. AT2 Project Scope — Three VMs

### What was built vs what was dropped

| Item | Status | Notes |
|---|---|---|
| BAA-SML-MAIL1 | ✅ Complete | iRedMail 1.8.2 |
| BAA-SML-WEB1 | ✅ Complete | WordPress 7.0 on LAMP |
| BAA-SML-NC1 | ✅ Complete | Nextcloud 33.0.5snap1 + Talk 23.0.6 |
| BAA-SML-VOIP1 (FreePBX) | ❌ Dropped | VoIP requirement fully covered by Nextcloud Talk inside NC1 |

### Framing for AT2 documents

The Implementation Plan was written as a **forward-looking project plan** (future tense), not a post-build report. This is the required academic framing.

The scope includes a two-server Hyper-V failover cluster as the HA/DR infrastructure. Hardware procurement and cluster build are included in the project schedule as part of the full project delivery — not framed as pending or out of scope.

Physical host machine names (Bojie-Mini, Bojiemini2) do not appear in the assessment documents.

---

## 7. DNS Infrastructure — daydreamtravel.com Zone

### Zone created on BAA-SML-DC1 (172.16.50.10)

Created during MAIL1 build. All subsequent VM builds only added their own A records.

```powershell
# Zone creation
Add-DnsServerPrimaryZone -Name "daydreamtravel.com" -ReplicationScope "Domain" -DynamicUpdate "Secure"
```

### All DNS records in the zone

| Type | Name | Value | Purpose | Added during |
|---|---|---|---|---|
| A | mail | 172.16.50.30 | Mail server | MAIL1 build |
| MX | @ | mail.daydreamtravel.com (priority 10) | Mail routing | MAIL1 build |
| TXT | @ | v=spf1 mx ~all | SPF email authentication | MAIL1 build |
| TXT | _dmarc | v=DMARC1; p=quarantine; rua=mailto:postmaster@daydreamtravel.com | DMARC policy | MAIL1 build |
| TXT | mail._domainkey | (DKIM public key) | DKIM signing verification | MAIL1 build |
| A | www | 172.16.50.31 | Web server | WEB1 build |
| A | nextcloud | 172.16.50.32 | Nextcloud server | NC1 build |

### DKIM key — known issue

The DKIM public key TXT record could not be added via PowerShell due to WIN32 203 error (key string too long for PowerShell's DNS module). Added successfully via DNS Manager GUI instead.

**Root cause:** Long DKIM keys exceed the string length that PowerShell's `Add-DnsServerResourceRecord` can handle. The DNS Manager GUI handles multi-string TXT records correctly.

**Prevention:** For future long TXT records (DKIM, DMARC RUF, etc.) — use DNS Manager GUI directly, not PowerShell.

---

## 8. BAA-SML-MAIL1 — iRedMail Email Server

**Build date:** 2026-06-04
**Build log source:** logbook_md___BAA-SML-MAIL1_entry.md

### VM Specifications

| Setting | Value |
|---|---|
| VM Name | BAA-SML-MAIL1 |
| Host | Bojie-Mini (SML site) |
| vSwitch | BAA internal switch |
| Generation | 2 (UEFI) |
| Secure Boot | Enabled — Microsoft UEFI CA template |
| vCPUs | 2 |
| RAM | 3072 MB static (Dynamic Memory disabled) |
| Disk | 30 GB VHDX dynamic — local SSD |
| IP | 172.16.50.30 /24 static |
| Gateway | 172.16.50.1 |
| DNS Primary | 172.16.50.10 |
| DNS Secondary | 172.16.60.10 |
| OS | Ubuntu Server 22.04 LTS |
| Hostname | mail |
| FQDN | mail.daydreamtravel.com |

**RAM note:** ClamAV requires at least 3 GB to hold its virus signature database in memory. Initial allocation of 2 GB was insufficient. Accepted RAM change during build (3072 MB static, Dynamic Memory disabled).

### Software Stack

| Component | Package / Version | Role |
|---|---|---|
| MTA | Postfix | SMTP send/receive |
| IMAP/POP3 | Dovecot | Mailbox access |
| Web server | Nginx | Reverse proxy for webmail/admin |
| Database | MariaDB | Stores virtual mailboxes, domains, aliases |
| Webmail | Roundcube | Browser-based email client |
| Admin panel | iRedAdmin (free tier) | Mailbox management UI |
| Antivirus | ClamAV + freshclam | Email virus scanning |
| Antispam | SpamAssassin via amavis | Spam filtering |
| Brute-force | Fail2ban | SSH/IMAP/SMTP/webmail protection |
| Installer | iRedMail 1.8.2 | All-in-one installer |

**netdata excluded** — would consume RAM needed for ClamAV.

### Architecture Decision

**Decision:** iRedMail 1.8.2 over manual Postfix/Dovecot configuration

**Reason:** iRedMail is a production-grade all-in-one installer that configures Postfix, Dovecot, ClamAV, SpamAssassin, amavis, Fail2ban, Nginx, and Roundcube as a coherent, pre-integrated stack. Manual configuration of the same components would take days and introduce integration bugs. For a TAFE assessment demonstrating enterprise email deployment, the focus should be on correct configuration and validation, not on reinventing integration work that iRedMail has already solved.

**Ruled out:** Manual Postfix + Dovecot + ClamAV + SpamAssassin build — too complex, too much time for no additional assessment evidence.

### Pre-install requirements

```bash
# FQDN must be set before running iRedMail installer
sudo nano /etc/hosts
# Added line: 172.16.50.30 mail.daydreamtravel.com mail

hostname -f
# Must return: mail.daydreamtravel.com
```

### iRedMail installer configuration choices

| Choice | Selection |
|---|---|
| Backend | MariaDB |
| Web server | Nginx |
| Domain | daydreamtravel.com |
| Components | Roundcube, iRedAdmin, Fail2ban |
| Components excluded | netdata |

### Mailboxes created (8 total, 1024 MB quota each)

| Username | Staff member |
|---|---|
| sharon | Sharon Webb (CEO) |
| karen | Karen Fells (Office Manager) |
| margret | Margret Baker (Receptionist) |
| john | John Kennedy (Senior Sales Rep) |
| mitchel | Mitchel Brown (Sales Rep) |
| coney | Coney Smith (Sales Rep) |
| alice | Alice Watson (Sales Rep) |
| peter | Peter Webb (IT Technician) |

### Distribution lists (3 total) — created via MariaDB

**Why MariaDB directly?** iRedAdmin free tier does not support alias/distribution list management. The `goto` column in the alias table was present in earlier versions but is absent from the iRedMail 1.8.2 schema. Members are stored in the `forwardings` table. Direct SQL inserts were required.

```sql
USE vmail;

-- all@daydreamtravel.com
INSERT INTO alias (address, domain, created) VALUES ('all@daydreamtravel.com', 'daydreamtravel.com', NOW());
INSERT INTO forwardings (address, forwarding, domain, is_alias) VALUES
  ('all@daydreamtravel.com', 'sharon@daydreamtravel.com', 'daydreamtravel.com', 1),
  ('all@daydreamtravel.com', 'karen@daydreamtravel.com', 'daydreamtravel.com', 1),
  ('all@daydreamtravel.com', 'margret@daydreamtravel.com', 'daydreamtravel.com', 1),
  ('all@daydreamtravel.com', 'john@daydreamtravel.com', 'daydreamtravel.com', 1),
  ('all@daydreamtravel.com', 'mitchel@daydreamtravel.com', 'daydreamtravel.com', 1),
  ('all@daydreamtravel.com', 'coney@daydreamtravel.com', 'daydreamtravel.com', 1),
  ('all@daydreamtravel.com', 'alice@daydreamtravel.com', 'daydreamtravel.com', 1),
  ('all@daydreamtravel.com', 'peter@daydreamtravel.com', 'daydreamtravel.com', 1);

-- sales@daydreamtravel.com (john, mitchel, coney, alice)
-- management@daydreamtravel.com (sharon, karen)
-- (same pattern as above, with appropriate members)
```

### Key commands run

```bash
# iRedMail download and extract
wget https://github.com/iredmail/iRedMail/archive/refs/tags/1.8.2.tar.gz
tar xvf 1.8.2.tar.gz
cd ~/iRedMail-1.8.2/
sudo bash iRedMail.sh

# Apply firewall rules
# (iRedMail generates nftables rules — applied after review, SSH port 22 preserved)

# Secure config file
sudo mv /home/bojie/iRedMail-1.8.2/config /root/iredmail-config.bak
sudo chmod 600 /root/iredmail-config.bak

# Service verification
sudo systemctl status postfix dovecot nginx mariadb clamav-daemon amavis fail2ban

# Antispam evidence
sudo grep "Passed CLEAN" /var/log/mail.log | tail -5

# Manual backup run
sudo bash /var/vmail/backup/backup_mysql.sh
# Databases backed up: mysql vmail roundcubemail amavisd iredadmin iredapd sa_bayes
# Location: /var/vmail/backup/mysql/2026/06/05/

# Fail2ban jail status
sudo fail2ban-client status
# 6 jails active: dovecot, nginx-http-auth, postfix, pregreet, roundcube, sshd

# Cron verification
sudo crontab -l -u root
# backup_mysql.sh at 03:30 daily confirmed
```

```powershell
# DNS records on BAA-SML-DC1
Add-DnsServerPrimaryZone -Name "daydreamtravel.com" -ReplicationScope "Domain" -DynamicUpdate "Secure"
Add-DnsServerResourceRecordA -ZoneName "daydreamtravel.com" -Name "mail" -IPv4Address "172.16.50.30"
Add-DnsServerResourceRecordMX -ZoneName "daydreamtravel.com" -Name "@" -MailExchange "mail.daydreamtravel.com" -Preference 10
Add-DnsServerResourceRecord -Txt -ZoneName "daydreamtravel.com" -Name "@" -DescriptiveText "v=spf1 mx ~all"
Add-DnsServerResourceRecord -Txt -ZoneName "daydreamtravel.com" -Name "_dmarc" -DescriptiveText "v=DMARC1; p=quarantine; rua=mailto:postmaster@daydreamtravel.com"
# DKIM: added via DNS Manager GUI (PowerShell WIN32 203 error due to key length)
```

### Issues encountered

| # | Issue | Symptom | Root cause | Resolution |
|---|---|---|---|---|
| M1 | iRedMail alias table missing 'goto' column | SQL error on INSERT | Schema changed in v1.8.2 — members now in forwardings table | Inserted into alias + forwardings separately via MariaDB |
| M2 | ClamAV freshclam log lock during install | Error in installer output | Race condition at install time | Resolved after reboot — ClamAV running cleanly |
| M3 | Config file not found at move step | mv: cannot stat | File already absent (installer cleanup) | No credentials left in home directory — confirmed safe |
| M4 | spamassassin.service not found | systemctl status failed | Not a standalone service in iRedMail — runs as library inside amavis | Confirmed via amavis log entries — no action needed |

### Verified state

| Check | Result |
|---|---|
| postfix | active (running) PID 1499 |
| dovecot | active (running) |
| nginx | active (running) |
| mariadb | active (running) |
| clamav-daemon | active (running) |
| amavis | active (running) |
| fail2ban | active (running) — 6 jails |
| Roundcube webmail | https://mail.daydreamtravel.com/mail/ — loads, accepts login |
| iRedAdmin | https://mail.daydreamtravel.com/iredadmin/ — loads, accepts login |
| Internal delivery | sharon → peter: delivered and scanned CLEAN |
| Distribution list | peter → all@: delivered to all 8 members, scanned CLEAN |
| DKIM | dkim:daydreamtravel.com signing confirmed in amavis log |
| SPF | v=spf1 mx ~all confirmed in DNS |
| DMARC | v=DMARC1; p=quarantine confirmed in DNS |
| DNS resolution | nslookup mail.daydreamtravel.com 172.16.50.10 → 172.16.50.30 |
| IMAP remote access | Port 993 — TcpTestSucceeded: True from Bojie-Mini host |
| Daily backup | Manual run confirmed — all 8 databases backed up |
| Backup cron | 03:30 daily — confirmed via crontab -l |
| Config file | /root/iredmail-config.bak — root only, chmod 600 |

### Hyper-V checkpoints

| Checkpoint | When taken |
|---|---|
| pre-iredmail-install | Before running iRedMail.sh |
| post-iredmail-install-verified | After full VERIFY passed |

---

## 9. BAA-SML-WEB1 — WordPress CMS

**Build dates:** 2026-06-05 to 2026-06-08
**Build log source:** AT2_build_log_WEB1.md

### VM Specifications

| Setting | Value |
|---|---|
| VM Name | BAA-SML-WEB1 |
| Host | Bojie-Mini (SML site) |
| vSwitch | BAA internal switch |
| Generation | 2 (UEFI) |
| Secure Boot | Enabled — Microsoft UEFI Certificate Authority template |
| vCPUs | 2 |
| RAM | 4096 MB static (Dynamic Memory disabled) |
| Disk | 30 GB VHDX dynamic — local SSD |
| IP | 172.16.50.31 /24 static |
| Gateway | 172.16.50.1 (BAA-SML-PFS1) |
| DNS Primary | 172.16.50.10 (BAA-SML-DC1) |
| DNS Secondary | 172.16.60.10 (BAA-BIG-DC1) |
| OS | Ubuntu Server 22.04 LTS |
| Hostname | web |
| FQDN | www.daydreamtravel.com |

### Software Stack

| Component | Package / Version | Role |
|---|---|---|
| Web server | Apache 2.4 | Serves WordPress over HTTP/HTTPS |
| Database | MariaDB 10.11.14 | Stores WordPress content and user data |
| PHP | PHP 8.3.6 + mod_php | Server-side scripting |
| CMS | WordPress 7.0 | Content management system |
| SSL | Self-signed (RSA 2048, 365 days) | HTTPS encryption |
| Modules | mod_rewrite, mod_ssl, mod_php | URL rewriting, SSL, PHP processing |

### PHP modules installed

curl, gd, intl, libxml, mbstring, mysqli, mysqlnd, pdo_mysql, soap, xml, xmlreader, xmlrpc, xmlwriter, zip

### PHP configuration — Apache (/etc/php/8.3/apache2/php.ini)

| Setting | Default | Configured |
|---|---|---|
| upload_max_filesize | 2M | 64M |
| post_max_size | 8M | 128M |
| memory_limit | 128M | 256M |
| max_execution_time | 30 | 300 |

**Critical note:** `php -r` and `php -i` in terminal read from the CLI php.ini (`/etc/php/8.3/cli/php.ini`), not Apache's php.ini. Terminal checks will always show CLI values regardless of Apache configuration. The correct verification method is the WordPress admin panel (Media → Add New shows the current upload limit).

### Architecture Decision — Apache (LAMP) over Nginx (LEMP)

**Reason:** Physical host Bojie-Mini has 64 GB RAM. The memory efficiency difference between Apache mod_php and Nginx + PHP-FPM is irrelevant at this scale. Apache is simpler for WordPress — `.htaccess` handles rewrites without requiring server block changes. Documentation coverage is near-universal. Fewer moving parts, less to troubleshoot, faster build.

**Ruled out:** Nginx + PHP-FPM — the RAM footprint advantage does not apply on a 64 GB host. PHP-FPM socket configuration adds time and risk with no measurable gain.

**Database:** MariaDB — same backend as MAIL1, already understood, correct for single-server WordPress.

### Apache Virtual Host Configuration

File: `/etc/apache2/sites-available/daydreamtravel.com.conf`

```apache
<VirtualHost *:80>
    ServerName www.daydreamtravel.com
    ServerAlias daydreamtravel.com
    Redirect permanent / https://www.daydreamtravel.com/
</VirtualHost>

<VirtualHost *:443>
    ServerName www.daydreamtravel.com
    ServerAlias daydreamtravel.com
    DocumentRoot /var/www/html/wordpress

    SSLEngine on
    SSLCertificateFile /etc/ssl/certs/daydreamtravel.crt
    SSLCertificateKeyFile /etc/ssl/private/daydreamtravel.key

    <Directory /var/www/html/wordpress>
        AllowOverride All
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/daydreamtravel-error.log
    CustomLog ${APACHE_LOG_DIR}/daydreamtravel-access.log combined
</VirtualHost>
```

Port 80 block issues a permanent redirect to HTTPS. Port 443 handles the actual site.

### SSL Certificate Details

| Field | Value |
|---|---|
| Type | Self-signed (lab — no public CA) |
| Algorithm | RSA 2048-bit |
| Validity | 365 days |
| Common Name | www.daydreamtravel.com |
| Organisation | Daydream Travel Agency |
| OU | IT |
| Country | AU |
| State | Queensland |
| Locality | Gold Coast |
| Email | peter@daydreamtravel.com |
| Certificate file | /etc/ssl/certs/daydreamtravel.crt |
| Key file | /etc/ssl/private/daydreamtravel.key |

### Database Configuration

| Setting | Value |
|---|---|
| Database name | wordpress |
| Database user | wpuser@localhost |
| Character set | utf8mb4 |
| Collation | utf8mb4_unicode_ci |
| Remote root login | Disabled |
| Anonymous users | Removed |
| Test database | Removed |

wp-config.php: `chmod 640` (owner www-data read/write, group read, others none)

### WordPress Configuration

| Setting | Value |
|---|---|
| Version | 7.0 |
| Site title | Daydream Travel Agency |
| Site URL | https://www.daydreamtravel.com |
| Admin account | peter.webb |
| Admin email | peter@daydreamtravel.com |
| Permalink structure | Post name (/about-us, /our-services) |
| Homepage display | Static page |
| Static homepage | Welcome to Daydream Travel Agency |
| Document root | /var/www/html/wordpress |

### Staff User Accounts and Roles

WordPress implements principle of least privilege — each staff member has only the access required for their role.

| Username | Full Name | Email | WordPress Role | Justification |
|---|---|---|---|---|
| peter.webb | Peter Webb | peter@daydreamtravel.com | Administrator | IT Technician — full system access |
| karen.fells | Karen Fells | karen@daydreamtravel.com | Editor | Office Manager — publish and manage all content |
| sharon.webb | Sharon Webb | sharon@daydreamtravel.com | Editor | CEO — publish and manage all content |
| john.kennedy | John Kennedy | john@daydreamtravel.com | Author | Senior Sales Rep — create and publish own content |
| mitchel.brown | Mitchel Brown | mitchel@daydreamtravel.com | Author | Sales Rep — create and publish own content |
| coney.smith | Coney Smith | coney@daydreamtravel.com | Author | Sales Rep — create and publish own content |
| alice.watson | Alice Watson | alice@daydreamtravel.com | Author | Sales Rep — create and publish own content |
| margret.baker | Margret Baker | margret@daydreamtravel.com | Subscriber | Receptionist — read only, no content management |

### Role Capability Summary

| Role | Create posts | Publish posts | Edit others | Manage users | Admin access |
|---|---|---|---|---|---|
| Administrator | ✅ | ✅ | ✅ | ✅ | Full |
| Editor | ✅ | ✅ | ✅ | ❌ | Content only |
| Author | ✅ | ✅ (own only) | ❌ | ❌ | Own posts only |
| Subscriber | ❌ | ❌ | ❌ | ❌ | Profile only |

### Published Content Pages

| Page title | Slug | Status | Content |
|---|---|---|---|
| Welcome to Daydream Travel Agency | / (static homepage) | Published | Intro to agency, four offices, travel specialists |
| About Us | /about-us | Published | Eight staff members, commitment to travel |
| Our Services | /our-services | Published | Int'l/domestic packages, corporate, group, cruise, visa |

Each page includes uploaded images.

### Full Implementation Steps

1. Created Gen 2 VM in Hyper-V Manager on Bojie-Mini
2. Set Secure Boot template to Microsoft UEFI Certificate Authority
3. Set vCPUs to 2, RAM 4096 MB static, disabled Dynamic Memory
4. Attached Ubuntu Server 22.04 ISO, set DVD first in boot order
5. Installed Ubuntu Server 22.04 — hostname: web, username: bojie
6. Set static IP 172.16.50.31 /24 during installer network config, no LVM
7. Enabled OpenSSH server during install
8. Detached ISO before reboot
9. Post-boot: edited /etc/hosts — `172.16.50.31 www.daydreamtravel.com web`
10. Verified: `hostname -f` → `www.daydreamtravel.com`
11. `sudo apt update && sudo apt upgrade -y`, rebooted
12. `sudo apt install apache2 -y`
13. `sudo apt install mariadb-server -y`
14. `sudo apt install php libapache2-mod-php php-mysql php-curl php-gd php-mbstring php-xml php-xmlrpc php-soap php-intl php-zip -y`
15. Enabled mod_rewrite: `sudo a2enmod rewrite && sudo systemctl restart apache2`
16. Verified: `apache2ctl -M | grep rewrite` → rewrite_module (shared)
17. Verified: `curl -s http://172.16.50.31 | grep title` → Apache2 Ubuntu Default Page: It works
18. `sudo mysql_secure_installation` — root password, removed anonymous users, disabled remote root, removed test DB
19. Created WordPress DB and user in MariaDB
20. Downloaded WordPress 7.0: `wget https://wordpress.org/latest.tar.gz`
21. Extracted: `tar xzvf latest.tar.gz`
22. Moved to web root: `sudo mv wordpress /var/www/html/wordpress`
23. Set ownership: `sudo chown -R www-data:www-data /var/www/html/wordpress`
24. Set permissions: directories 755, files 644
25. Created Apache virtual host config for HTTP (port 80)
26. `sudo a2ensite daydreamtravel.com.conf && sudo a2dissite 000-default.conf`
27. `sudo apache2ctl configtest` → Syntax OK
28. `sudo systemctl reload apache2`
29. Copied `wp-config-sample.php` to `wp-config.php`
30. Configured DB_NAME, DB_USER, DB_PASSWORD, DB_HOST in wp-config.php
31. `sudo chmod 640 /var/www/html/wordpress/wp-config.php`
32. Added DNS A record www → 172.16.50.31 on BAA-SML-DC1
33. Verified DNS from BAA-SML-DC1 and from WEB1
34. `sudo a2enmod ssl`
35. Generated self-signed SSL certificate with `openssl req -x509`
36. Updated virtual host config — added port 443 block and HTTP→HTTPS redirect on port 80
37. `sudo apache2ctl configtest` → Syntax OK && `sudo systemctl reload apache2`
38. Ran WordPress web installer at https://www.daydreamtravel.com
39. Created admin account peter.webb, site title Daydream Travel Agency
40. Created 7 additional staff accounts with role assignments (Users → Add New)
41. Edited `/etc/php/8.3/apache2/php.ini` — increased upload_max_filesize, post_max_size, memory_limit, max_execution_time
42. `sudo systemctl stop apache2 && sudo systemctl start apache2` (full restart required — reload does not reinitialise PHP module)
43. Verified upload limit: WordPress Media → Add New showed Maximum upload file size: 64 MB
44. Created and published three content pages with images
45. Settings → Reading → set static homepage to Welcome page
46. Settings → Permalinks → changed from Plain to Post name
47. Fixed .htaccess ownership: `sudo chown www-data:www-data .htaccess`
48. Wrote WordPress rewrite rules to .htaccess manually using `sudo -u www-data tee`
49. Took Hyper-V checkpoint: post-wordpress-install-verified

### Issues Encountered

| # | Issue | Symptom | Root cause | Resolution |
|---|---|---|---|---|
| W1 | Apache not installed | Unit apache2.service could not be found | Install command from previous session did not complete | Re-ran `sudo apt install apache2 -y` |
| W2 | mod_rewrite not active | `apache2ctl -M \| grep rewrite` returned empty | Not enabled by default on Ubuntu 22.04 | `sudo a2enmod rewrite` + restart |
| W3 | PHP upload limit not applying | WordPress showed 2M limit despite php.ini change | Apache `reload` does not reinitialise the PHP module — full stop/start required | `sudo systemctl stop apache2 && sudo systemctl start apache2` |
| W4 | PHP terminal showed 2M after apache php.ini edit | `php -r` showed 2M | Terminal reads CLI php.ini (`/etc/php/8.3/cli/php.ini`), not Apache's. Two separate files. | Confirmed correct value via WordPress Media page (64 MB) |
| W5 | 404 on /about-us | Apache returned 404 for all pretty URLs | WordPress .htaccess had markers but no rewrite rules — www-data lacked write permission at install time | Fixed ownership, wrote rewrite rules manually |
| W6 | Pages returning ?page_id=N URLs | /about-us showed as /?page_id=9 | Permalink structure was Plain (default) | Settings → Permalinks → Post name → Save Changes |

### Verified State

| Test | Type | Result |
|---|---|---|
| HTTP → HTTPS redirect | Positive | PASS |
| Homepage loads | Positive | PASS |
| /about-us loads | Positive | PASS |
| /our-services loads | Positive | PASS |
| User role counts (Admin 1, Editor 2, Author 4, Subscriber 1) | Positive | PASS |
| margret.baker admin restriction (Dashboard + Profile only) | Negative | PASS |
| apache2 active | Positive | PASS |
| mariadb active | Positive | PASS |
| DNS: www.daydreamtravel.com → 172.16.50.31 | Positive | PASS |
| SSL: self-signed, CN www.daydreamtravel.com | Positive | PASS |
| mod_rewrite loaded | Positive | PASS |
| mod_ssl loaded | Positive | PASS |
| php_module loaded | Positive | PASS |
| upload_max_filesize: 64M | Positive | PASS |

### Final Service State

```
apache2    active (running) — started 2026-06-07 22:05:44 UTC
mariadb    active (running) — started 2026-06-07 22:05:43 UTC
PHP        8.3.6
Modules    php_module, rewrite_module, ssl_module — all loaded (shared)
upload_max_filesize    64M
post_max_size          128M
memory_limit           256M
```

### Hyper-V Checkpoints

| Checkpoint | When taken |
|---|---|
| post-wordpress-install-verified | After full VERIFY stage passed |

---

## 10. BAA-SML-NC1 — Nextcloud Collaboration + Talk

**Build date:** 2026-06-08
**Build log source:** AT2_build_log_NC1.md

### VM Specifications

| Setting | Value |
|---|---|
| VM Name | BAA-SML-NC1 |
| Host | Bojie-Mini (SML site) |
| vSwitch | BAA internal switch |
| Generation | 2 (UEFI) |
| Secure Boot | Enabled — Microsoft UEFI Certificate Authority template |
| vCPUs | 2 |
| RAM | 2048 MB static (Dynamic Memory disabled) |
| Disk | 30 GB VHDX dynamic — local SSD |
| IP | 172.16.50.32 /24 static |
| Gateway | 172.16.50.1 (BAA-SML-PFS1) |
| DNS Primary | 172.16.50.10 (BAA-SML-DC1) |
| DNS Secondary | 172.16.60.10 (BAA-BIG-DC1) |
| OS | Ubuntu Server 22.04 LTS |
| Hostname | nextcloud |
| FQDN | nextcloud.daydreamtravel.com |

### Software Stack

| Component | Package / Version | Role |
|---|---|---|
| Nextcloud | 33.0.5snap1 (snap) | File sharing and collaboration |
| Nextcloud Talk | spreed 23.0.6 | WebRTC audio/video calling (VoIP) |
| Apache | Bundled inside snap | Serves Nextcloud over HTTP/HTTPS |
| PHP | Bundled inside snap | Server-side scripting |
| MySQL | Bundled inside snap | Nextcloud data and user accounts |
| SSL | Self-signed via snap command | HTTPS encryption |

The Nextcloud snap bundles its own Apache, PHP, and MySQL inside a self-contained snap sandbox. These are configured via `snap set` and `occ` commands — not via direct editing of system config files. System packages (apache2, php, mariadb) are **not** installed on the host OS.

### Architecture Decision — Snap over Manual LAMP

**Reason:** Manual LAMP skills are already fully evidenced on BAA-SML-WEB1 (Apache, MariaDB, PHP 8.3, manual SSL). Repeating the same method on NC1 adds no additional evidence. The Nextcloud snap is a Canonical-maintained production deployment method that bundles Apache, PHP, and MySQL in a self-contained unit. Using it here demonstrates a second legitimate deployment approach and reduces build time — a real factor with two documents still to write before 12 June.

**Ruled out:** Manual LAMP install — not wrong, but redundant. WEB1 already provides full evidence of manual LAMP configuration. The snap approach produces an identical outcome through a different and equally valid method.

**What this demonstrates:** Deployment method breadth. The same service can be deployed via manual configuration (WEB1) or via a managed package (NC1). Both are valid production approaches.

### Trusted Domains Configuration

Nextcloud restricts access to explicitly listed hostnames.

| Index | Value |
|---|---|
| 0 | 172.16.50.32 |
| 1 | nextcloud.daydreamtravel.com |

### Staff User Accounts and Groups

| Username | Full Name | Group |
|---|---|---|
| admin | admin | admin |
| peter.webb | Peter Webb | admin |
| sharon.webb | Sharon Webb | management |
| karen.fells | Karen Fells | management |
| john.kennedy | John Kennedy | sales |
| mitchel.brown | Mitchel Brown | sales |
| coney.smith | Coney Smith | sales |
| alice.watson | Alice Watson | sales |
| margret.baker | Margret Baker | reception |

### Groups

| Group | Members | Purpose |
|---|---|---|
| admin | admin, peter.webb | Full Nextcloud administration |
| management | sharon.webb, karen.fells | CEO and Office Manager |
| sales | john.kennedy, mitchel.brown, coney.smith, alice.watson | Sales team file sharing |
| reception | margret.baker | Receptionist access |

### Full Implementation Steps

1. Created Gen 2 VM in Hyper-V Manager on Bojie-Mini
2. Set Secure Boot template to Microsoft UEFI Certificate Authority
3. Set vCPUs to 2, RAM 2048 MB static, disabled Dynamic Memory
4. Attached Ubuntu Server 22.04 ISO, set DVD first in boot order
5. Installed Ubuntu Server 22.04 — hostname: nextcloud, username: bojie
6. Set static IP 172.16.50.32 /24 during installer network config
7. Left search domains blank — not required
8. Enabled OpenSSH server during install
9. Post-boot: SSH from Bojie-Mini — `ssh bojie@172.16.50.32`
10. Verified IP: `ip a` — 172.16.50.32/24 on eth0 confirmed
11. Edited /etc/hosts — `172.16.50.32 nextcloud.daydreamtravel.com nextcloud`
12. Verified: `hostname -f` → `nextcloud.daydreamtravel.com`
13. `sudo apt update && sudo apt upgrade -y` — kernel current, no reboot needed
14. Took Hyper-V checkpoint: pre-nextcloud-install
15. `sudo snap install nextcloud` — installed Nextcloud 33.0.5snap1
16. Set admin credentials: `sudo nextcloud.manual-install admin <password>`
17. Set trusted domain 0: `sudo nextcloud.occ config:system:set trusted_domains 0 --value=172.16.50.32`
18. Set trusted domain 1: `sudo nextcloud.occ config:system:set trusted_domains 1 --value=nextcloud.daydreamtravel.com`
19. Enabled HTTPS: `sudo nextcloud.enable-https self-signed`
20. Created 8 staff accounts via `occ user:add --password-from-env`
21. Created groups: management, sales, reception
22. Assigned all staff to appropriate groups
23. Added peter.webb to built-in admin group
24. Installed Nextcloud Talk: `sudo nextcloud.occ app:install spreed`
25. Added DNS A record on BAA-SML-DC1: nextcloud → 172.16.50.32
26. Verified DNS resolution from BAA-SML-DC1
27. Took Hyper-V checkpoint: post-nextcloud-install-verified

### Issues Encountered

None. Build completed without errors on first attempt.

### Verified State

| Test | Type | Result |
|---|---|---|
| HTTPS login page loads | Positive | PASS |
| Self-signed cert warning | Positive | PASS (expected — lab environment) |
| Admin login | Positive | PASS — Dashboard with admin account |
| Staff login (sharon.webb) | Positive | PASS — "Good morning, Sharon Webb" |
| Nextcloud Talk visible | Positive | PASS — Talk icon, Start meeting / Create conversation |
| User list via occ | Positive | PASS — 9 accounts: admin + 8 staff |
| Group list via occ | Positive | PASS — 4 groups, members correct |
| DNS: nextcloud.daydreamtravel.com → 172.16.50.32 | Positive | PASS |
| HTTP → HTTPS redirect | Negative | PASS |

### Final Service State

```
Nextcloud       33.0.5snap1 — running
Nextcloud Talk  spreed 23.0.6 — enabled
HTTPS           self-signed cert — active
HTTP redirect   port 80 → 443 — active
Trusted domains 172.16.50.32 / nextcloud.daydreamtravel.com
User accounts   9 (admin + 8 staff)
Groups          4 (admin, management, sales, reception)
```

### Hyper-V Checkpoints

| Checkpoint | When taken |
|---|---|
| pre-nextcloud-install | After Ubuntu install and apt upgrade, before snap install |
| post-nextcloud-install-verified | After full VERIFY stage passed |

---

## 11. VoIP Decision — Nextcloud Talk vs FreePBX

### Original plan

The project initially included a fourth VM, BAA-SML-VOIP1, running FreePBX to provide SIP-based VoIP with desk phone extensions.

### Decision to drop FreePBX

**The AT2 marking criteria requires VoIP capability, not a specific PBX technology.**

Nextcloud Talk (spreed app, version 23.0.6) provides browser-based WebRTC audio and video calling. This is a legitimate VoIP implementation for an internal office network. It satisfies the marking criteria requirement without the build complexity of a dedicated PBX.

**FreePBX was dropped because:**
- Nextcloud Talk fully satisfies the VoIP requirement
- FreePBX adds significant build time (SIP trunks, dial plans, extension configuration, codec negotiation)
- Two assessment documents still needed to be written before 12 June
- An additional VM adds more potential points of failure with no assessment benefit

**Ruling:** BAA-SML-VOIP1 dropped. VoIP requirement fully covered by Nextcloud Talk inside BAA-SML-NC1.

---

## 12. Architecture Decision Records (ADRs)

### ADR-001: iRedMail over manual Postfix/Dovecot

**Decision:** iRedMail 1.8.2 all-in-one installer
**Ruled out:** Manual Postfix + Dovecot + ClamAV + SpamAssassin
**Reason:** iRedMail provides a pre-integrated, production-grade stack. Manual configuration would take days and introduce integration bugs. No additional assessment evidence gained from manual integration.

### ADR-002: Apache (LAMP) over Nginx (LEMP) for WEB1

**Decision:** Apache 2.4 with mod_php
**Ruled out:** Nginx + PHP-FPM
**Reason:** 64 GB host RAM makes memory efficiency irrelevant. Apache .htaccess is simpler for WordPress. Fewer moving parts.

### ADR-003: MariaDB backend across all three VMs

**Decision:** MariaDB on all VMs (iRedMail, WordPress, Nextcloud snap internal)
**Ruled out:** PostgreSQL / MySQL — MariaDB is standard for all three applications, reducing cognitive overhead and troubleshooting surface.

### ADR-004: Nextcloud snap over manual LAMP for NC1

**Decision:** Canonical snap install (`sudo snap install nextcloud`)
**Ruled out:** Manual Apache + PHP + MySQL + Nextcloud
**Reason:** WEB1 already provides full evidence of manual LAMP deployment. Snap demonstrates a second valid deployment method. Significant time saving on NC1 frees capacity for the two assessment documents.

### ADR-005: Nextcloud Talk over FreePBX for VoIP

**Decision:** Nextcloud Talk (spreed 23.0.6) inside NC1
**Ruled out:** Dedicated BAA-SML-VOIP1 running FreePBX
**Reason:** Marking criteria requires VoIP capability, not a specific PBX. WebRTC satisfies this with far less build complexity. Scope of work remains within time budget.

### ADR-006: Physical hosts not domain-joined

**Decision:** Bojie-Mini and Bojiemini2 remain in workgroup
**Ruled out:** Joining physical hosts to bojieanzac.com
**Reason:** Physical hosts are management infrastructure, not workstations. Keeping them out of the domain separates management plane from lab plane. Documented in docs/02-remote-access-design.md.

### ADR-007: Static RAM (no Dynamic Memory) on all AT2 VMs

**Decision:** Static RAM allocation across MAIL1 (3072 MB), WEB1 (4096 MB), NC1 (2048 MB)
**Ruled out:** Dynamic Memory enabled
**Reason:** ClamAV on MAIL1 requires resident memory for virus signature DB — Dynamic Memory can reclaim RAM and crash ClamAV. Same discipline applied to all VMs for consistency.

### ADR-008: All AT2 VMs on BAA internal switch

**Decision:** All SML site AT2 VMs attached to BAA internal switch (172.16.50.0/24, behind pfSense)
**Ruled out:** BAA external switch (direct home LAN attachment)
**Reason:** Lab VMs should be on the lab network segment, behind the lab firewall. External switch bypasses pfSense. This is the correct network architecture for every VM in the lab.

---

## 13. Cross-VM Issues and Troubleshooting Records

### T1 — MAIL1 stuck in "Shutting Down" state (Hyper-V)

**Symptom:** Hyper-V Manager showed MAIL1 as "Shutting Down" — could not start, stop, or modify the VM. Occurred during WEB1 build session.

**Hypothesis:** Host was shut down while VM was mid-shutdown. The vmwp.exe worker process was left holding the VM state.

**Test:** `Get-WmiObject Win32_Process | Where-Object { $_.Name -eq "vmwp.exe" }` — identified the specific worker process PID.

**Fix:** `Stop-Process -Id 24980 -Force`

**Root cause:** Hyper-V VM worker process (vmwp.exe) retained in memory after ungraceful host shutdown while VM was still stopping. No OS-level lock release mechanism — must be killed manually.

**Prevention:** Always allow guest OS to complete shutdown before shutting down the host. If forced shutdown is needed, confirm VM shows "Off" state in Hyper-V Manager before closing the host.

---

### T2 — MAIL1 Fail2ban banned_db errors after restart

**Symptom:** `sudo systemctl start fail2ban` failed with "ERROR Failed to start jail dovecot/pregreet/roundcube action banned_db" after MAIL1 was restarted following the stuck-VM incident.

**Hypothesis:** Race condition — fail2ban started before MariaDB socket was ready. banned_db action requires active MariaDB connection.

**Test:** Wait for MariaDB to fully initialise, then `sudo systemctl restart fail2ban`.

**Fix:** Restart fail2ban after MariaDB is fully up.

**Root cause:** Service startup ordering. MariaDB is not listed as a hard dependency for fail2ban in the iRedMail configuration. On fast boots, fail2ban can reach its jail initialisation step before MariaDB socket is available.

**Prevention:** Could add `After=mariadb.service` to fail2ban's systemd unit. For this lab, manual restart after ungraceful reboot is acceptable.

---

### T3 — iRedMail 1.8.2 schema change (alias.goto column missing)

**Symptom:** SQL INSERT to `alias` table failed — column `goto` does not exist.

**Root cause:** iRedMail 1.8.2 changed the schema. The `goto` column was removed from the `alias` table. Distribution list member addresses are now stored in the separate `forwardings` table.

**Fix:** Insert alias address into `alias` table (without goto), then insert each member as a row in `forwardings` with `address` = the alias and `forwarding` = the member's address.

**Prevention:** Check iRedMail changelog and vmail schema before writing SQL for distribution lists. Schema changes between versions are documented but easy to miss.

---

### T4 — WordPress .htaccess missing rewrite rules (404 on all pages)

**Symptom:** After setting permalink structure to "Post name", all pages except the homepage returned 404. /about-us, /our-services — all 404.

**Hypothesis:** WordPress should write rewrite rules to .htaccess when permalink structure is saved. If .htaccess is not writable by www-data, the rules are not written.

**Test:** `ls -la /var/www/html/wordpress/.htaccess` — owner was root, not www-data. www-data had no write permission.

**Fix:**
1. `sudo chown www-data:www-data /var/www/html/wordpress/.htaccess`
2. Wrote WordPress rewrite rules manually: `sudo -u www-data tee /var/www/html/wordpress/.htaccess`

**Root cause:** During WordPress installation, www-data did not own .htaccess. WordPress generates .htaccess at install time, but if the file already exists and is not writable by www-data, the rewrite block is not added. The file had the WordPress markers but not the actual rules.

**Prevention:** At install time, ensure www-data owns the entire WordPress document root including .htaccess before running the web installer. Set ownership with `chown -R www-data:www-data /var/www/html/wordpress` before any browser-based install steps.

---

### T5 — PHP upload limit not applying after Apache reload

**Symptom:** WordPress showed "Maximum upload file size: 2 MB" despite editing /etc/php/8.3/apache2/php.ini to set upload_max_filesize = 64M.

**Root cause:** `sudo systemctl reload apache2` sends a SIGHUP to Apache, which re-reads Apache configuration but does not reinitialise the PHP module. The PHP module is loaded once at startup — reload does not restart it.

**Fix:** Full stop/start: `sudo systemctl stop apache2 && sudo systemctl start apache2`

**Prevention:** PHP configuration changes on Apache with mod_php require a full service stop and start, not reload. Document this as a lab rule.

---

### T6 — PHP CLI vs Apache php.ini confusion

**Symptom:** Running `php -r "echo ini_get('upload_max_filesize');"` in terminal returned "2M" even after the Apache php.ini was correctly set to 64M.

**Root cause:** The PHP CLI (`/usr/bin/php`) uses `/etc/php/8.3/cli/php.ini`. Apache mod_php uses `/etc/php/8.3/apache2/php.ini`. These are entirely separate files. Changes to one do not affect the other.

**Prevention:** Always verify PHP configuration changes via the application (WordPress admin) rather than `php -r` or `php -i` in the terminal. Terminal PHP commands are never authoritative for Apache-served applications.

---

### T7 — DKIM TXT record WIN32 203 error (PowerShell)

**Symptom:** `Add-DnsServerResourceRecord -Txt` failed with WIN32 203 error when attempting to add the DKIM public key TXT record.

**Root cause:** DKIM public keys are long strings. PowerShell's DNS module has a string length limit for TXT record values. The key exceeded this limit.

**Fix:** Added the DKIM TXT record via DNS Manager GUI instead of PowerShell. The GUI handles multi-string TXT records correctly.

**Prevention:** For any TXT record with a long value (DKIM, complex DMARC, etc.) — use DNS Manager GUI directly.

---

## 14. AT2 Assessment Documents

### AT2 Written Document 1 — Implementation Plan

**File:** ICTNWK536_AT2_Implementation_Plan_Bojie_Qiao.docx
**Status:** Complete and submitted

**Content:**
- Six-week / 30-working-day project schedule with 15 tasks
- All 14 marking criteria verified as addressed
- Framing: Bojie as external IT contractor from Uptown IT
- HA infrastructure: two-server Hyper-V failover cluster
- Hardware procurement and cluster build included in project schedule as part of full project delivery
- Forward-looking project plan (future tense) — not a post-build report
- Physical host machine names do not appear in document text
- Font: 13pt body / 18pt headings

**Key framing decisions:**
- HA/DR: must not imply cluster will be built in the future as a separate concern. Hardware procurement and cluster build are included in the schedule as part of full project delivery.
- Scope must match what is documented in the Implementation Plan — no topology details introduced that are not already present in the document.
- "Pending procurement" must not appear outside of project schedule context.

---

### AT2 Written Document 2 — Test Plan

**File:** (submitted alongside Implementation Plan)
**Status:** Complete and submitted

**Content:**
- Six real issues documented across MAIL1 (3) and WEB1 (3)
- NC1: no issues encountered — clean first-attempt build
- Four refinement fixes applied during drafting:
  - Figure references corrected
  - "write permission" wording corrected
  - Teacher instruction placeholder text removed
  - HA/DR conclusion revised to match agreed framing

**Issues documented in Test Plan:**
1. MAIL1 — iRedMail alias table missing goto column
2. MAIL1 — ClamAV freshclam log lock during install
3. MAIL1 — spamassassin.service not found
4. WEB1 — WordPress .htaccess missing rewrite rules (404 on all pages)
5. WEB1 — PHP upload limit not applying after Apache reload
6. WEB1 — MAIL1 stuck in Shutting Down state (cross-VM issue observed during WEB1 session)

---

## 15. Key Learnings and Principles

### Email infrastructure

1. **iRedMail 1.8.2 schema change:** The alias `goto` column no longer exists. Distribution list member addresses are stored in the `forwardings` table. Direct MariaDB inserts required for distribution lists on the free iRedAdmin tier.

2. **ClamAV RAM requirement:** Requires at least 3 GB static RAM. 2 GB is insufficient. ClamAV holds its virus signature database in memory — if RAM is reclaimed, the daemon dies silently.

3. **SpamAssassin is not standalone in iRedMail:** Runs as a library inside amavis. No separate systemctl unit. Evidence of operation is in the amavis log, not a standalone service.

### Apache / PHP

4. **Two separate php.ini files:** CLI and Apache php.ini are completely separate. `/etc/php/8.3/cli/php.ini` for terminal, `/etc/php/8.3/apache2/php.ini` for Apache. Changes to one never affect the other.

5. **PHP module requires full restart:** `sudo systemctl reload apache2` does not reinitialise mod_php. PHP configuration changes require `stop && start`.

6. **www-data must own the WordPress document root at install time:** Missing write permission on .htaccess causes WordPress to skip writing rewrite rules, resulting in 404 on all pretty URLs. Set `chown -R www-data:www-data` before running the web installer.

### DNS / Windows Server

7. **DKIM key length and PowerShell:** Long DKIM keys cannot be added via PowerShell (WIN32 203 error). DNS Manager GUI is the reliable fallback.

8. **pfSense gateways must not be used as DNS servers:** pfSense (.1) does not serve the AD DNS zone. Domain controller (.10) is the correct DNS server for all lab VMs. This was explicitly verified during each Ubuntu install.

### Deployment methodology

9. **Snap vs manual install breadth:** Using snap for NC1 while using manual LAMP for WEB1 demonstrates deployment method breadth. Both are production-valid approaches. This was a deliberate architectural choice, not a shortcut.

10. **Build logs are raw material for assessment documents:** Everything logged during builds becomes the evidence base for the Implementation Plan and Test Plan. Log everything, especially errors and fixes.

11. **All SML site VMs on BAA internal switch:** VMs belong on the lab network behind pfSense, not on the external switch. External switch bypasses the lab firewall entirely.

12. **Hyper-V stuck VM (vmwp.exe):** A VM stuck in "Shutting Down" state can be resolved by identifying the vmwp.exe worker process PID via `Get-WmiObject Win32_Process` and killing it with `Stop-Process -Force`.

### Engineering methodology

13. **Universal troubleshooting core:** All major troubleshooting methodologies (CompTIA 6-step, scientific method, Kepner-Tregoe, Toyota 5 Whys) share the same logical core: observe → hypothesise → test → conclude. The TROUBLESHOOT step in Bojie's 8-step methodology maps cleanly to all of these, with ROOT CAUSE and PREVENTION fields as the distinguishing outputs.

14. **Implementation Plan tone discipline:** Must read as a forward-looking project plan (future tense), not a post-build report. Scope must match exactly what is documented — no topology details introduced that are not already in the plan.

---

## 16. Chronological Timeline

| Date | Activity |
|---|---|
| Pre-project | AT1 completed — open-source stack (iRedMail + WordPress + Nextcloud) selected |
| Pre-project | AT2 VM builds specified |
| 2026-06-01 | IPsec VPN between BIG and SML sites confirmed working |
| 2026-06-01 | DHCP event 1059 root cause resolved — service dependency chain set on both DCs |
| 2026-06-04 | BAA-SML-MAIL1 build — iRedMail 1.8.2 installed and verified |
| 2026-06-04/05 | DNS zone daydreamtravel.com created — mail A, MX, SPF, DMARC, DKIM records added |
| 2026-06-04/05 | DKIM TXT record added via DNS Manager GUI (PowerShell WIN32 203 error) |
| 2026-06-05 | BAA-SML-WEB1 build begins |
| 2026-06-05 | MAIL1 stuck in Shutting Down during WEB1 session — vmwp.exe fix applied |
| 2026-06-05 | MAIL1 fail2ban banned_db errors after restart — resolved by restarting fail2ban after MariaDB |
| 2026-06-05-08 | BAA-SML-WEB1 build completed — WordPress 7.0, HTTPS, 8 users, 3 pages, all issues resolved |
| 2026-06-07 | WEB1 VERIFY passes — checkpoint post-wordpress-install-verified taken |
| 2026-06-08 | BAA-SML-NC1 build — Nextcloud 33.0.5snap1 + Talk 23.0.6 installed and verified |
| 2026-06-08 | NC1 VERIFY passes — checkpoint post-nextcloud-install-verified taken |
| 2026-06-08 | Topology file (01-topology.md) last verified — NC1 still shows 🔨 (pre-completion) |
| ~2026-06-10 | AT2 Implementation Plan written and finalised |
| ~2026-06-11 | AT2 Test Plan drafted and refined (four fixes applied) |
| 2026-06-12 | Both AT2 documents submitted before deadline |

---

## 17. Outstanding Work

### UNDERSTAND steps (explicitly deferred until after submission)

| VM | UNDERSTAND deferred? |
|---|---|
| BAA-SML-MAIL1 | Yes — deferred until after 12 June |
| BAA-SML-WEB1 | Yes — deferred until after 12 June |
| BAA-SML-NC1 | Yes — deferred until after 12 June |

These are the Step 8 (UNDERSTAND) reflection outputs for each build. Each requires:
- Name the concept demonstrated
- How it applies beyond this specific lab
- Formal name for it

### Planned post-submission work

| Item | Priority |
|---|---|
| UNDERSTAND step for MAIL1 | High |
| UNDERSTAND step for WEB1 | High |
| UNDERSTAND step for NC1 | High |
| Level 2 interactive React methodology agent artifact | Post-submission project |
| Review and consolidate lab topology documentation | Post-submission cleanup |
| Topology file update — NC1 status from 🔨 to ✅ | Minor |

### GitHub repo

Repository: github.com/qbjsuper/tafe-virtual-server-lab

Build logs and logbook entries should be committed to the repo. This master logbook is suitable for inclusion as `logbook-master.md` or similar.

---

*End of ICTNWK536 AT2 Master Project Logbook*
*Generated: 2026-06-13*
*Methodology: REQUIREMENTS → EVALUATE → DECIDE → IMPLEMENT → VERIFY → TROUBLESHOOT → LOG → UNDERSTAND*
