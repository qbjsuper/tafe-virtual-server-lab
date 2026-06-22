# Master Project Log — Anzac Airport Integrated Server and Virtualisation Lab

**Student:** Bojie Qiao  
**Student ID:** 450783451  
**Project scope:** ICTNWK539/540 AT2 and AT3; ICTNWK557/559 AT1 and AT2  
**Domain:** `bojieanzac.com`  
**Primary platform:** Microsoft Hyper-V  
**Linux platform:** Ubuntu Server 24.04 LTS  
**Workstation platform:** Windows 11 Pro  
**Status date:** 22 June 2026  
**Evidence repositories:**
- `https://github.com/qbjsuper/tafe-virtual-server-lab`
- `https://github.com/qbjsuper/539-540-AT3-Screenshots`

---

## 1. Purpose of This Log

This master log records the design, implementation, testing, troubleshooting, evidence capture, and documentation completed for the Anzac Airport integrated server and virtualisation project.

The work covers:

1. ICTNWK539/540 AT2 planning and design.
2. ICTNWK539/540 AT3 practical server implementation.
3. ICTNWK557/559 AT1 virtualisation platform evaluation.
4. ICTNWK557/559 AT2 virtualisation management, high availability, backup, and VLAN segmentation.
5. The final lab topology, system roles, major configuration decisions, testing outcomes, evidence names, issues, and current outstanding work.

The log distinguishes between work that is **completed**, **partially completed**, and **pending verification**. No successful result is recorded unless it was supported by practical lab output or screenshot evidence.

---

# PART A — APPROVED DESIGN AND LAB BASELINE

## 2. Business and Technical Objective

The project implemented a simulated integrated server solution for Anzac Airport, a small private airport requiring centralised authentication, managed addressing, file storage, printer deployment, web and mail services, secure file transfer, cross-platform integration, backup, monitoring, and improved availability.

The approved design replaced a peer-to-peer model with a managed client/server environment using:

- Active Directory Domain Services.
- AD-integrated DNS.
- Windows DHCP.
- Windows file and print services.
- Ubuntu-based web, mail, SFTP, Samba, proxy, and time services.
- Hyper-V virtualisation.
- pfSense firewalls and inter-site routing.
- A nested Hyper-V failover cluster for high availability evidence.
- Azure Arc and Azure Update Manager for update management evidence.

---

## 3. Locked Design Decisions

| Design item | Final decision |
|---|---|
| Domain | `bojieanzac.com` |
| NetBIOS domain | `BOJIEANZAC` |
| Windows server platform | Windows Server 2025 Standard/Evaluation in the lab |
| Linux server platform | Ubuntu Server 24.04 LTS |
| Workstation platform | Windows 11 Pro |
| Hypervisor | Microsoft Hyper-V |
| BIG subnet | `172.16.60.0/24` |
| SML subnet | `172.16.50.0/24` |
| BIG gateway | `172.16.60.1` on `BAA-BIG-PFS1` |
| SML gateway | `172.16.50.1` on `BAA-SML-PFS1` |
| Main workstation | `BAA-BIG-WS` |
| Primary domain controller | `BAA-BIG-DC1` |
| Secondary domain controller | `BAA-SML-DC1` |
| File server | `BAA-BIG-FILE1` |
| Main Linux integration/SFTP/proxy server | `BAA-BIG-LX1` |
| Samba server | `BAA-BIG-LX2` |
| Web server | `BAA-SML-WEB1` |
| Mail server | `BAA-SML-MAIL1` |
| Shared storage | `BAA-STOR1` |
| HA cluster | `BAA-NEST-CL1` |
| HA nodes | `BAA-BIG-Nest1` and `BAA-SML-Nest1` |
| Update management | Azure Update Manager, replacing WSUS by teacher direction |

---

## 4. Practical Assessment Mapping

| Assessment role | Real lab system | Main evidence function |
|---|---|---|
| BAA-DC1 | `BAA-BIG-DC1` | Primary AD DS, DNS, DHCP, Group Policy, Windows time |
| BAA-DC2 | `BAA-SML-DC1` | Secondary AD DS, DNS, resilience and availability |
| BAA-WS1 | `BAA-BIG-WS` | Main Windows 11 Pro domain workstation |
| BAA-FILE1 | `BAA-BIG-FILE1` | File, print, NTFS permissions, backup and restore |
| BAA-MAIL1 | `BAA-SML-MAIL1` | iRedMail and Roundcube |
| BAA-LNX1 | `BAA-BIG-LX1` | Ubuntu AD integration, SFTP, Squid, UFW, diagnostics |
| Linux web server | `BAA-SML-WEB1` | Apache, WordPress and HTTPS |
| Linux Samba server | `BAA-BIG-LX2` | Samba, Winbind and AD group-controlled SMB sharing |
| Shared storage | `BAA-STOR1` | iSCSI/cluster shared storage |
| HA node 1 | `BAA-BIG-Nest1` | Nested Hyper-V cluster node |
| HA node 2 | `BAA-SML-Nest1` | Nested Hyper-V cluster node |
| Cluster | `BAA-NEST-CL1` | High availability and VM migration |
| BIG firewall | `BAA-BIG-PFS1` | BIG gateway and firewall |
| SML firewall | `BAA-SML-PFS1` | SML gateway, IPsec and VLAN routing |

---

## 5. Core Addressing Record

| System | Address / network | Notes |
|---|---:|---|
| BAA-BIG-PFS1 | `172.16.60.1` | BIG gateway |
| BAA-BIG-DC1 | `172.16.60.10` | Primary AD DS/DNS/DHCP |
| BAA-BIG-FILE1 | `172.16.60.11` | Windows file/print/backup server |
| BAA-BIG-LX1 | `172.16.60.102` | Ubuntu AD/SFTP/Squid server |
| BAA-BIG-LX2 | `172.16.60.105` | Ubuntu Samba/Winbind server |
| BAA-SML-PFS1 | `172.16.50.1` | SML gateway |
| BAA-SML-DC1 | `172.16.50.10` | Secondary AD DS/DNS |
| BAA-STOR1 | `172.16.50.101` | Shared storage |
| BIG network | `172.16.60.0/24` | Primary site |
| SML network | `172.16.50.0/24` | Secondary site |
| Accounts VLAN | `172.16.51.0/24` | VLAN 10 |
| Administration VLAN | `172.16.52.0/24` | VLAN 20 |

The BIG and SML sites communicate through pfSense and the existing site-to-site IPsec connection.

---

# PART B — ICTNWK539/540 AT2 DESIGN WORK

## 6. AT2 Project Proposal

The AT2 proposal documented the existing business gaps and two server design options.

### 6.1 Design Option 1 — Physical Server Design

The physical option proposed five business servers based on Dell PowerEdge R360 or equivalent hardware:

- BAA-DC1.
- BAA-DC2.
- BAA-FILE1.
- BAA-MAIL1.
- BAA-LNX1.

The design included Windows Server for Microsoft infrastructure services and Ubuntu Server for Linux-based services.

### 6.2 Design Option 2 — Virtualised Server Design

The virtualised option proposed:

- Two Windows Server Hyper-V hosts.
- One shared storage/backup server.
- Five core server VMs.
- Windows 11 Pro workstations.
- Hyper-V failover clustering and shared storage.

This option became the main practical baseline because it aligned with the existing home lab and supported high availability evidence.

### 6.3 Services Planned

The proposal covered:

- Active Directory.
- DNS.
- DHCP.
- Group Policy.
- File sharing and NTFS permissions.
- Linux/Windows integration.
- Backup and restore.
- Print management.
- Update management.
- Web, mail and SFTP/FTP.
- Squid proxy.
- NTP.
- Firewalls and antivirus.
- Monitoring and diagnostics.
- High availability.

### 6.4 AT2 Documentation Completed

| Document | Outcome |
|---|---|
| Project proposal | Completed |
| Two design options | Completed |
| Task schedule and budget spreadsheet | Completed |
| Pre-implementation test plan | Completed |
| Project approval request | Completed |
| Domain and naming decision | Finalised as `bojieanzac.com` |
| AT2 submission | Completed on 12 June 2026 |

---

# PART C — ICTNWK539/540 AT3 PRACTICAL IMPLEMENTATION

## 7. AT3 Implementation Approach

The AT3 work used the approved AT2 virtualised design as the baseline, but mapped assessment server roles to the existing lab VMs.

The implementation approach was:

1. Verify existing configuration before changing it.
2. Reuse working VMs where possible.
3. Create a new VM only when an existing VM could not safely meet the requirement.
4. Capture before-and-after screenshots for important changes.
5. Record real problems in the troubleshooting log.
6. Keep evidence aligned with the marking criteria and portfolio requirements.
7. Use one or two strong screenshots where possible instead of excessive evidence.

---

## 8. Foundation: Hyper-V, Network and Domain Services

### 8.1 Hyper-V and VM Foundation

The required Windows, Linux, firewall, workstation, storage, and cluster VMs were present in Hyper-V and connected to the required virtual switches.

Evidence covered:

- VM names and running states.
- CPU, RAM, disks, and uptime.
- VM network adapter assignments.
- Hyper-V Manager health.
- Storage allocation.

### 8.2 Active Directory

The domain `bojieanzac.com` was implemented with:

- `BAA-BIG-DC1` as the primary domain controller.
- `BAA-SML-DC1` as the secondary domain controller.
- Active Directory replication between the two sites.
- AD-integrated DNS.
- Windows DHCP.
- Group Policy.
- Domain time hierarchy.

### 8.3 DNS

DNS was hosted on both domain controllers.

Testing confirmed:

- Resolution of `bojieanzac.com`.
- Resolution of server hostnames.
- DNS service availability from the workstation.
- Required SRV records on the secondary domain controller.
- Corrected host records where addresses had changed.

### 8.4 DHCP

Windows DHCP provided client configuration for the lab networks.

The workstation received:

- IPv4 address.
- subnet mask/prefix.
- default gateway.
- DNS servers.
- domain suffix.

A later resilience test identified that the BIG DHCP scope originally supplied only `172.16.60.10` as DNS. The scope was corrected to also supply `172.16.50.10`.

### 8.5 Windows 11 Workstation

`BAA-BIG-WS` was:

- Installed as Windows 11 Pro.
- Joined to `bojieanzac.com`.
- Used for domain-user login testing.
- Used for file, web, mail, SFTP, Samba, printer, proxy, firewall, DNS, DHCP, and availability tests.

---

## 9. Active Directory Organisational Structure

### 9.1 Department Groups

The practical file-access groups included:

- `BAA-GRP-Finance-RW`
- `BAA-GRP-Administration-RW`
- `BAA-GRP-IT-RW`
- `BAA-GRP-Company-Common-RW`

Additional groups were used for ground operations and later VLAN assessment work.

### 9.2 Test Users

The main AT3 test users included:

- `BAA-FIN-User1`
- `BAA-ADM-User1`
- `BAA-IT-User1`
- `BAA-GOP-User1`

A dedicated `samba-test` account was later created for a valid negative Samba permission test because the original departmental users were all members of the common-share group.

### 9.3 Password Policy Issue

The initial `New-ADUser` process created disabled user objects because the temporary password did not meet the domain policy.

Resolution:

- Set a stronger temporary password.
- Used `Set-ADAccountPassword`.
- Enabled the accounts with `Enable-ADAccount`.
- Verified `Enabled : True` with `Get-ADUser`.

---

## 10. Windows File Server

### 10.1 Server Build

`BAA-BIG-FILE1` was implemented as the practical BAA-FILE1 server.

Recorded storage:

| Volume | Approximate allocation | Use |
|---|---:|---|
| C: | 60 GB | Operating system |
| D: | 80 GB | Data and departmental shares |
| E: | 100 GB | Backup/update evidence storage |

### 10.2 Shared Folder Structure

The implemented shares included:

- Administration.
- Finance.
- IT.
- Company_Common.
- Tarmac.

The shares were created under `D:\Shares`.

### 10.3 Permissions

Access was controlled by:

- SMB share permissions.
- NTFS permissions.
- Active Directory security groups.
- Domain-user membership.

### 10.4 Testing

The following results were verified:

- An authorised Finance user could access the permitted shared location.
- The authorised user could create, edit, save, and read files.
- An unauthorised user received access denied.
- The workstation could reach SMB services on the file server.

### 10.5 Hyper-V Enhanced Session Login Issue

`BAA-FIN-User1` initially could not sign in through the enhanced Hyper-V console because Enhanced Session uses Remote Desktop Services.

Resolution:

- Confirmed the account and password were valid.
- Identified that the failure was an RDP rights issue, not an AD issue.
- Used a basic Hyper-V console session for the AT3 permission test.
- Later enabled RDP access where required for the 557/559 workstation work.

---

## 11. Linux Domain Integration

### 11.1 BAA-BIG-LX1

`BAA-BIG-LX1` was used for:

- Ubuntu Server 24.04 evidence.
- IP configuration.
- AD integration.
- Kerberos authentication.
- OpenSSH/SFTP.
- Squid proxy.
- UFW firewall.
- Linux monitoring and diagnostics.

The server successfully authenticated against `bojieanzac.com`.

### 11.2 Authentication Verification

Testing included:

- `realm`/domain membership checks.
- `kinit` using an AD account.
- `klist` to confirm a Kerberos ticket.
- DNS and time checks required for Kerberos.

---

## 12. Samba and Winbind Server

### 12.1 Rebuild Decision

The original `BAA-BIG-LX2` used Ubuntu 26.04 and experienced an out-of-memory killer storm during package installation.

It was replaced with a fresh VM:

- Ubuntu Server 24.04 LTS.
- 4096 MB RAM.
- 30 GB disk.
- 2 vCPUs.
- IP address `172.16.60.105`.

### 12.2 DHCP Client Identifier Fix

LX1 and LX2 were using DUID-based DHCP identifiers, which caused unstable reservations.

The Netplan configuration was changed to use:

```yaml
dhcp-identifier: mac
```

Final reservations:

| VM | MAC | Reserved IP |
|---|---|---:|
| BAA-BIG-LX1 | `00:15:5d:01:fb:07` | `172.16.60.102` |
| BAA-BIG-LX2 | `00:15:5d:01:fb:0a` | `172.16.60.105` |

### 12.3 Samba/Winbind Configuration

Installed components included:

- Samba.
- Winbind.
- realmd.
- Kerberos tools.
- NSS/PAM Winbind support.
- smbclient.
- ACL tools.

`BAA-BIG-LX2` was configured as an AD domain member, not a Samba domain controller.

The accidental `samba-ad-dc` service was disabled and masked to avoid conflict with `smbd`.

### 12.4 Domain Join Issue

`realm join` reported that the server was already joined, but the Samba machine secret was missing.

Symptoms:

- `net ads testjoin` failed.
- `wbinfo -t` showed the wrong trust state.
- Winbind could not retrieve the domain SID.

Resolution:

- Ran `net ads join -U Administrator`.
- Populated `secrets.tdb`.
- Restarted `smbd` and `winbind`.
- Verified `Join is OK`.
- Verified the domain trust secret.
- Verified domain users were visible.

### 12.5 Samba Share

Share name:

- `AnzacLinuxShare`

Path:

- `/srv/samba/anzac-share`

Access group:

- `BAA-GRP-Company-Common-RW`

The share used group-based access control and set-group-ID permissions so files remained associated with the required group.

### 12.6 Windows-to-Linux File Sharing Test

From `BAA-BIG-WS`:

- TCP port 445 was reachable.
- The share mapped successfully using domain credentials.
- A test file was created and read.
- A non-member test account received System Error 5 / access denied.

### 12.7 Key Samba Evidence Names

- `SS_6-5A_BAA_BIG_LX2_Pre_Domain_Join_Check`
- `SS_6-5B_BAA_BIG_LX2_Domain_Join_Verification`
- `SS_6-5C_BAA_BIG_LX2_AD_Computer_Object`
- `SS_6-5D_Samba_Winbind_Service_Running_On_BAA_BIG_LX2`
- `SS_6-5E_Samba_Share_Configured_With_AD_Group`
- `SS_6-5F_Samba_Port_445_Reachable_From_BAA_BIG_WS`
- `SS_6-5G_Samba_Share_Access_From_Windows_Domain_User`
- `SS_6-5H_Samba_File_Create_Read_Test_Domain_User`
- `SS_6-5I_Samba_Unauthorised_User_Access_Denied`

---

## 13. SFTP Service

### 13.1 Implementation

`BAA-BIG-LX1` provided secure file transfer using OpenSSH SFTP.

The client was:

- WinSCP 6.5.6 on `BAA-BIG-WS`.

### 13.2 Testing Completed

The assessment evidence covered:

- SFTP service running.
- Authorised login.
- Upload.
- Download.
- Large file transfer.
- Multiple/concurrent sessions.
- Linux firewall allowing SSH/SFTP traffic.

### 13.3 Path Troubleshooting

An SFTP test initially used an incorrect or unsuitable remote path.

Resolution:

- Confirmed the authenticated user’s valid SFTP directory.
- Used the correct writable path.
- Repeated upload/download testing successfully.

---

## 14. Web Service

### 14.1 Server Reuse

`BAA-SML-WEB1` was repurposed from the previous Daydream Travel Agency WordPress lab.

The VM was retained rather than creating a new web server.

### 14.2 Changes

The server was updated to:

- Anzac Airport branding.
- Hostname and domain aligned with `bojieanzac.com`.
- Apache virtual host configuration.
- WordPress site URL and home URL corrected.
- Self-signed HTTPS certificate.
- HTTP-to-HTTPS redirection.
- WordPress content updated.

### 14.3 Troubleshooting

Two main issues were fixed:

1. Incorrect redirects caused by old WordPress URL values.
2. WordPress JSON/REST 404 errors caused by permalink/plugin configuration.

Testing confirmed the site could be opened from the workstation by the intended internal address.

---

## 15. Mail Service

### 15.1 Implementation

`BAA-SML-MAIL1` ran:

- Ubuntu Server.
- iRedMail 1.8.2.
- Roundcube webmail.
- Mail domain `bojieanzac.com`.

### 15.2 Mailboxes and Testing

Mailboxes included:

- `finance.user1`
- `admin.user1`

Testing covered:

- Webmail login.
- Sending a message.
- Receiving the message in the second mailbox.

The service was implemented and tested even though the final report was later simplified and did not emphasise the mail system.

---

## 16. Update Management

### 16.1 Design Change

The AT2 proposal originally used WSUS. The teacher directed the implementation to use Azure Update Manager because it is the preferred future-facing approach.

This decision was recorded and applied to AT3 evidence.

### 16.2 Azure Arc Onboarding

The following servers were onboarded:

| Server | OS | Result |
|---|---|---|
| BAA-BIG-FILE1 | Windows Server | Connected and assessed |
| BAA-SML-MAIL1 | Ubuntu | Connected and assessed |
| BAA-SML-WEB1 | Ubuntu | Connected and assessed |

### 16.3 Update Evidence

The Azure evidence showed:

- Machines onboarded.
- Periodic assessment enabled.
- Update compliance.
- Assessment details.
- Monthly maintenance schedule.
- Schedule association with the three machines.

### 16.4 Evidence Names

- `SS_8-1_Azure_Update_Manager_Machines_Onboarded.jpg`
- `SS_8-1B_Azure_Update_Manager_Periodic_Assessment_Enabled.jpg`
- `SS_8-2_Azure_Update_Manager_Update_Compliance_Status.jpg`
- `SS_8-2B_Azure_Update_Manager_Assessment_Result.jpg`
- `SS_8-2C_Azure_Update_Manager_Update_Schedule_Configured.jpg`
- `SS_8-2D_Azure_Update_Manager_Associated_Update_Schedule.jpg`

---

## 17. Print Management and Group Policy

### 17.1 Physical Printer

A real Canon G4070 printer was added through `BAA-BIG-FILE1`.

Connection:

- Direct TCP/IP printing.
- Port 9100.
- Printer address `192.168.1.31`.

### 17.2 Print Server and Deployment

The printer was:

- Added to the Windows print server.
- Shared.
- Deployed using Group Policy.
- Made available to the required workstation/user context.

### 17.3 Practical Verification

The following were successfully demonstrated:

- `Test-NetConnection` to TCP port 9100 succeeded.
- Windows printed a server test page.
- The physical printer produced the page.
- The Finance user could see/use the printer from `BAA-BIG-WS`.

---

## 18. Squid Proxy

### 18.1 Implementation

Squid ran on:

- `BAA-BIG-LX1`
- Address `172.16.60.102`
- Port `3128`

### 18.2 Rules

Squid ACLs allowed the BIG and SML lab networks.

UFW allowed the BIG network to connect to TCP 3128.

### 18.3 Workstation Test

`BAA-BIG-WS` used the Windows manual proxy configuration.

The Squid access log showed successful workstation traffic, including:

- `TCP_MISS/200`
- `TCP_TUNNEL/200`

This proved that traffic was reaching and using the proxy.

---

## 19. Time Synchronisation

Time synchronisation was verified because consistent time is required for Kerberos, logs, certificates, and troubleshooting.

### 19.1 Windows

`BAA-BIG-DC1` was used for Windows/domain time evidence.

### 19.2 Linux

`BAA-BIG-LX2` was configured to use the domain controllers as NTP sources.

Verified Linux state included:

- System clock synchronised.
- NTP service active.
- Time zone set to Australia/Brisbane.
- Domain controller used as the active source.

### 19.3 Evidence Names

- `SS_8-8_NTP_Time_Sync_Status_On_Domain_Controller`
- `SS_8-9_NTP_Time_Sync_Status_On_BAA_BIG_LX2`

---

## 20. Backup and Restore

### 20.1 Windows Server Backup

Windows Server Backup was installed on `BAA-BIG-FILE1`.

### 20.2 Backup Test

The D: data volume was backed up to the E: backup volume.

Recorded backup version:

- `06/18/2026-10:56`

### 20.3 Restore Test

Test path:

- `D:\Shares\Company_Common\BackupRestore_Test\AT3_Backup_Restore_Test.txt`

Process:

1. Created the test file.
2. Completed the backup.
3. Deleted the original test file.
4. Restored it to `C:\Restore_Test`.
5. Opened and verified the restored file.

### 20.4 Evidence Names

- `SS_9-1_Windows_Server_Backup_Installed_On_BAA_BIG_FILE1.jpg`
- `SS_9-2_9-3_Backup_Job_Configured_And_Completed_Successfully.jpg`
- `SS_9-4_9-5_Test_File_Deleted_And_Restored_To_Recovery_Location.jpg`
- `SS_9-6_Restored_File_Opened_Successfully.jpg`

---

## 21. Firewall Testing

### 21.1 Windows Firewall

The test sequence covered:

1. Ping successful before the rule.
2. Windows Firewall rule configured to block ICMP echo.
3. Ping failed after the rule.
4. Required application services remained reachable.

This demonstrated that the rule blocked only the intended traffic.

### 21.2 Linux Firewall

UFW status and application rules were shown on Linux.

Required service access was verified for services such as:

- SSH/SFTP.
- Squid proxy.
- Other approved application ports.

### 21.3 DNS Correction

The DNS A record for `BAA-BIG-LX1` was corrected from an outdated address to `172.16.60.102` before final service testing.

---

## 22. Performance and Diagnostics

### 22.1 Windows Evidence

Evidence was captured from `BAA-BIG-FILE1` showing:

- Task Manager performance.
- CPU, memory, disk, and network usage.
- Event Viewer review.
- No critical service errors affecting the assessment.
- Server Manager roles and services.

### 22.2 Linux Evidence

Linux evidence showed:

- CPU usage.
- memory usage.
- disk usage.
- network state.
- service status for SFTP, proxy, and time synchronisation.

### 22.3 Hyper-V Evidence

Hyper-V Manager showed the health and running state of the virtual machines.

### 22.4 Evidence Names

- `SS_11-1_Windows_Task_Manager_Performance_On_BAA_BIG_FILE1.jpg`
- `SS_11-2_Windows_Event_Viewer_No_Critical_Service_Errors_On_BAA_BIG_FILE1.jpg`
- `SS_11-3_Server_Manager_Roles_And_Services_Running_On_BAA_BIG_FILE1.jpg`
- `SS_11-4_Ubuntu_CPU_Memory_Disk_Network_Status_On_BAA_BIG_LX1.jpg`
- `SS_11-5_Ubuntu_Service_Status_For_SFTP_Proxy_NTP_On_BAA_BIG_LX1.jpg`
- `SS_11-6_Hyper-V_Manager_Showing_VM_Health.jpg`
- `SS_11-7_Diagnostic_Report_Command_Output_On_BAA_BIG_LX1.jpg`

---

## 23. High Availability and Cluster Testing

### 23.1 Cluster Components

The HA lab used:

- Cluster: `BAA-NEST-CL1`
- Node 1: `BAA-BIG-Nest1`
- Node 2: `BAA-SML-Nest1`
- Shared storage: `BAA-STOR1`
- Clustered test VM: `BAA-TEST-VM01`
- iSCSI target: `BAA-NEST-CSV-TGT01`

### 23.2 Testing

The practical test included:

1. Verified both cluster nodes.
2. Verified Cluster Shared Volume/shared storage.
3. Verified the test VM as a clustered role.
4. Confirmed the VM owner node.
5. Performed a controlled move from BIG to SML.
6. Confirmed the VM was running after the move.
7. Verified network/SSH access before and after the move.

### 23.3 Result

The planned migration test was successful and demonstrated service availability after VM movement.

Evidence section:

- `SS_12-1` through `SS_12-6`

---

## 24. Domain Controller Availability Test

### 24.1 Failure Test

`BAA-BIG-DC1` was shut down to test whether `BAA-BIG-WS` could find and use the secondary domain controller.

Initial result:

- `nltest /dsgetdc:bojieanzac.com /force`
- Failed with `ERROR_NO_SUCH_DOMAIN` / error 1355.

### 24.2 Diagnostics

The following were checked:

- DNS server configuration on the workstation.
- DHCP option 006.
- DNS SRV records on `BAA-SML-DC1`.
- LDAP TCP 389.
- Kerberos TCP 88.
- SMB/DC reachability.
- SYSVOL and NETLOGON shares on the secondary DC.

The secondary DC services were operational.

### 24.3 Root Cause

The BIG DHCP scope supplied only:

- `172.16.60.10`

It did not supply the secondary DNS server:

- `172.16.50.10`

When the primary DC/DNS server was offline, the workstation had no usable DNS server for domain-controller discovery.

### 24.4 Resolution

DHCP option 006 was updated to include both domain-controller DNS addresses.

The workstation renewed its DHCP configuration and received both DNS servers.

### 24.5 Retest

With `BAA-BIG-DC1` unavailable, the workstation successfully discovered `BAA-SML-DC1`.

This became the strongest end-to-end troubleshooting example in the AT3 evidence.

### 24.6 Evidence Names

- `SS_13-12_Domain_Controller_Availability_Failed_During_BIG_DC1_Shutdown.jpg`
- `SS_13-15_DHCP_DNS_Option_Before_Fix_Missing_SML_DC1.jpg`
- `SS_13-18_Domain_Controller_Availability_Successful_After_DHCP_DNS_Fix.jpg`

---

## 25. AT3 Troubleshooting Register

| Issue | Root cause | Resolution | Final result |
|---|---|---|---|
| AD users created disabled | Temporary password failed domain complexity policy | Reset stronger passwords and enabled accounts | Successful |
| Domain user could not log in through Enhanced Session | Enhanced Session uses RDP and the user lacked RDP rights | Used basic console; later configured RDP where needed | Successful |
| SFTP transfer path failed | Incorrect/non-writable remote path | Used the correct user SFTP directory | Successful |
| WordPress redirect issue | Old Daydream URL values remained | Updated WordPress home/site URL and Apache configuration | Successful |
| WordPress JSON 404 | Permalink/plugin configuration | Corrected WordPress settings and retested | Successful |
| Samba trust failed | `realm join` did not populate Samba machine secrets | Ran `net ads join`; restarted Samba/Winbind | Successful |
| Samba DNS record missing | Automatic DNS update failed | Added manual A record on the DC | Successful |
| Samba negative test was invalid | Selected user was already in the allowed common group | Created a dedicated non-member test account | Successful |
| Linux DHCP reservations unstable | DHCP clients used DUID instead of MAC identifier | Added `dhcp-identifier: mac` in Netplan | Successful |
| LX2 package installation OOM | Ubuntu 26.04 VM had insufficient resources | Rebuilt LX2 on Ubuntu 24.04 with 4 GB RAM | Successful |
| Secondary DC not discovered during primary DC outage | DHCP supplied only the primary DNS server | Added secondary DC DNS to DHCP option 006 | Successful |

---

## 26. AT3 Final Documentation

The following assessment files were completed or finalised:

| File | Status |
|---|---|
| Risk Assessment | Completed |
| Test Plan | Completed with practical results and evidence |
| Troubleshooting document | Completed with real resolved issues |
| Final report | Rewritten and finalised without tables |
| Screenshot checklist/index | Updated |
| Screenshot evidence repository | Organised |
| Final report/sign-off content | Prepared |

The AT3 Portfolio of Evidence required risk assessment, implementation, testing, troubleshooting, and post-installation documentation. The marking criteria were used as the completion checklist.

---

# PART D — ICTNWK557/559 AT1

## 27. Virtualisation Platform Evaluation

The AT1 evaluation compared:

- VMware ESXi with vCenter.
- Microsoft Hyper-V.
- VMware Workstation.
- Oracle VirtualBox.

The work explained:

- Type 1 and Type 2 hypervisors.
- Physical versus virtual servers.
- Virtual CPU, memory, disks, and network adapters.
- Benefits of virtualisation.
- High availability.
- Sustainable ICT practices.
- Lifecycle management and e-waste.
- Centralised management.
- Energy efficiency.
- Australian government sustainability guidance.

### 27.1 Recommendation

Microsoft Hyper-V was selected because it:

- Integrated with the existing Microsoft environment.
- Supported Windows, Linux, and pfSense VMs.
- Supported Failover Clustering and live migration.
- Aligned with the practical lab.
- Supported centralised management and PowerShell.
- Provided a suitable cost and skills fit.

AT1 Part 1 was completed on 12 June 2026.

---

# PART E — ICTNWK557/559 AT2 PRACTICAL WORK

## 28. Assessment Scope

The practical tasks covered:

- Hyper-V hosts and remote management.
- VM infrastructure.
- Connectivity, availability, and performance.
- VM cloning.
- VM template deployment.
- Checkpoints.
- Resource management.
- VM backup.
- High availability.
- Two domain workstations.
- VLAN segmentation.
- Different bandwidth settings.
- Security rules.

The existing `bojieanzac.com` lab was reused instead of building a separate domain.

---

## 29. VM Clone Task

### 29.1 Work Completed

A simple Windows Server VM was cloned/deployed as:

- `BAA-WIN-DEPLOY1`

The work demonstrated copying or deploying a VM as a separate instance.

### 29.2 Status

**Completed.**

---

## 30. VM Template and Deployment Task

### 30.1 Template

An Ubuntu template was prepared as:

- `BAA-UBU-TEMPLATE`

### 30.2 Deployed VM

The template was used to deploy:

- `BAA-UBU-DEPLOY1`

### 30.3 Status

**Completed.**

---

## 31. Checkpoint and Resource Pool

### 31.1 Checkpoint

A Hyper-V checkpoint was created for the test/deployed VM.

### 31.2 Processor Resource Pool

Created processor resource pool:

- `BAA-AT2-CPU-Pool`

Applied to:

- `BAA-UBU-DEPLOY1`

Verified settings:

- Virtual processor count: 2.
- Relative weight: 200.
- Resource pool name shown by `Get-VMProcessor`.

### 31.3 Status

- Checkpoint: **Completed**
- Resource pool: **Completed**

---

## 32. VM Backup Work

### 32.1 Intended Target

The initial plan was to back up/export a VM to:

- `\\BAA-STOR1\VM-Backups`

### 32.2 Verification

Checks showed:

- TCP 445 to `172.16.50.101` succeeded.
- DNS resolved `BAA-STOR1`.
- `Test-Path "\\172.16.50.101\VM-Backups"` returned `False`.
- The expected SMB share was not present or not accessible.

### 32.3 Revised Approach

The backup evidence was redirected toward local host storage on the E: drive as an acceptable alternative target.

### 32.4 Current Status

**Partially completed / final backup evidence still requires confirmation.**

Do not record the NAS share backup as successful unless the exported backup files are verified in the selected final location.

---

## 33. High Availability Task

The existing nested cluster was reused:

- `BAA-NEST-CL1`
- `BAA-BIG-Nest1`
- `BAA-SML-Nest1`
- `BAA-STOR1`
- `BAA-TEST-VM01`

The cluster and VM movement evidence from the 539/540 implementation also supported the 557/559 HA requirement.

The VM was moved between nodes and remained reachable.

### Status

**Completed.**

---

## 34. VLAN Segmentation

### 34.1 Hyper-V Virtual Switch

Created:

- `BAA-SML-VLAN-Trunk`

Switch type:

- Internal.

Bandwidth mode:

- Weight.

### 34.2 pfSense Trunk NIC

A third adapter named `VLAN-Trunk` was added to:

- `BAA-SML-PFS1`

Recorded MAC:

- `00155D382D17`

Hyper-V trunk configuration:

- Allowed VLAN 10.
- Allowed VLAN 20.
- Native VLAN 4094.

The pfSense parent interface was:

- `hn2`

### 34.3 VLAN Interfaces

| VLAN | Name | Subnet | Gateway |
|---:|---|---|---:|
| 10 | ACCOUNTS / VLAN10_ACCOUNTS | `172.16.51.0/24` | `172.16.51.1` |
| 20 | ADMINISTRATION / VLAN20_ADMINISTRATION | `172.16.52.0/24` | `172.16.52.1` |

No upstream gateways were configured on the internal VLAN interfaces.

### 34.4 DHCP

| VLAN | DHCP range | DNS servers |
|---:|---|---|
| 10 | `172.16.51.100–172.16.51.119` | `172.16.50.10`, `172.16.60.10` |
| 20 | `172.16.52.100–172.16.52.119` | `172.16.50.10`, `172.16.60.10` |

Domain suffix:

- `bojieanzac.com`

### 34.5 Workstations

| VM | VLAN | OS | Result |
|---|---:|---|---|
| BAA-ACC-WS1 | 10 | Windows 11 Pro | Domain joined |
| BAA-ADM-WS1 | 20 | Windows 11 Pro | Domain joined |

The Accounts workstation received:

- IP `172.16.51.100`
- Gateway `172.16.51.1`
- Domain suffix `bojieanzac.com`

### 34.6 Users and Groups

Actual accounts:

- `BAA-ACCOUNT-USER1`
- `BAA-ADMIN-USER1`

Groups:

- `BAA-GRP-ACCOUNT`
- `BAA-GRP-ADMIN`

Membership was verified.

### 34.7 RDP and Enhanced Session

Remote Desktop was enabled using the Windows GUI.

The domain test users were granted the required local Remote Desktop access on their matching workstations.

Enhanced Session could then be used with the domain users.

### 34.8 Bandwidth

The Accounts workstation was configured with:

- Minimum bandwidth weight 80.

The Administration workstation used the standard/lower allocation, but the exact final weight was not recorded in the available checkpoint.

A speed test did not show an exact 2:1 result, but the configured Hyper-V bandwidth weight was accepted as the required evidence.

### 34.9 Security Rules

pfSense rules were configured to:

- Permit required domain controller and DNS traffic.
- Permit required internet access.
- Restrict unwanted traffic between the Accounts and Administration VLANs.
- Demonstrate VLAN security separation.

### 34.10 Status

**Completed for the VLAN, domain, workstation, and security requirements.**

---

# PART F — EVIDENCE MANAGEMENT

## 35. Main AT3 Screenshot Sections

The screenshot checklist was organised into:

1. VM and host foundation.
2. Windows Server and domain foundation.
3. Workstation evidence.
4. Active Directory objects.
5. File server, folder structure, and permissions.
6. Windows and Linux integration.
7. Web, mail, and FTP/SFTP services.
8. Update management, printer GPO, proxy, and NTP.
9. Backup and restore.
10. Firewall testing.
11. Performance and diagnostics.
12. High availability and cluster evidence.
13. Troubleshooting evidence.
14. Final report evidence.

The evidence naming format used:

- `SS_<section>-<item>_<descriptive_name>.jpg`

Additional evidence used:

- `SS_EXTRA_<descriptive_name>.jpg`

The GitHub evidence repository contains the full screenshot index and image links.

---

## 36. Key Evidence Chains

### 36.1 File Permission Chain

1. File server running.
2. Shared folder structure.
3. NTFS permissions.
4. Authorised user access.
5. Unauthorised user denied.
6. File create/edit/save.

### 36.2 Samba Chain

1. Pre-join state.
2. Domain join verification.
3. AD computer object.
4. Samba/Winbind services.
5. Share configuration.
6. TCP 445 test.
7. Authorised domain-user access.
8. File create/read.
9. Unauthorised access denied.

### 36.3 Backup Chain

1. Backup feature installed.
2. Job configured.
3. Backup successful.
4. Test file deleted.
5. File restored to recovery location.
6. Restored file opened.

### 36.4 Firewall Chain

1. Ping works before rule.
2. Firewall rule shown.
3. Ping blocked after rule.
4. Required service remains available.

### 36.5 HA Chain

1. Cluster nodes.
2. Shared storage.
3. Clustered VM.
4. Owner before move.
5. Controlled move.
6. Reachability after move.

### 36.6 Domain Controller Resilience Chain

1. Primary DC shut down.
2. Workstation DC discovery failed.
3. DNS/DHCP settings diagnosed.
4. Missing secondary DNS identified.
5. DHCP option corrected.
6. Workstation renewed configuration.
7. Secondary DC discovery succeeded.

---

# PART G — CURRENT PROJECT STATUS

## 37. Completion Matrix

| Work area | Status |
|---|---|
| ICTNWK539/540 AT2 proposal | Completed |
| ICTNWK539/540 AT2 schedule/budget | Completed |
| ICTNWK539/540 pre-implementation test plan | Completed |
| ICTNWK539/540 project approval document | Completed |
| AD DS, DNS, DHCP, GPO | Completed |
| Windows 11 workstation/domain login | Completed |
| AD OUs, users, and groups | Completed |
| Windows file server and permissions | Completed |
| Ubuntu AD integration | Completed |
| SFTP | Completed |
| WordPress web service | Completed |
| iRedMail/Roundcube mail service | Completed |
| Samba/Winbind cross-platform sharing | Completed |
| Azure Update Manager | Completed |
| Physical printer and GPO deployment | Completed |
| Squid proxy | Completed |
| Windows/Linux NTP evidence | Completed |
| Windows backup and restore | Completed |
| Windows and Linux firewall evidence | Completed |
| Performance and diagnostics | Completed |
| Hyper-V cluster and VM move | Completed |
| Domain controller resilience troubleshooting | Completed |
| AT3 Test Plan | Completed |
| AT3 Troubleshooting document | Completed |
| AT3 final report | Completed |
| AT3 screenshot index/repository | Completed |
| ICTNWK557/559 AT1 evaluation | Completed |
| 557/559 VM clone | Completed |
| 557/559 VM template/deployment | Completed |
| 557/559 checkpoint | Completed |
| 557/559 processor resource pool | Completed |
| 557/559 HA | Completed |
| 557/559 two domain workstations | Completed |
| 557/559 VLAN 10 and VLAN 20 | Completed |
| 557/559 VLAN security | Completed |
| 557/559 VM backup/export | Partially completed; final target verification required |
| 557/559 assessor verification/sign-off | Pending assessor action where required |

---

## 38. Remaining Work

The main remaining practical item is the 557/559 Task 16 VM backup evidence.

Required final verification:

1. Select the final backup/export destination.
2. Export the selected VM.
3. Confirm the exported configuration, virtual disk, and snapshot folders exist.
4. Capture the designed before/after evidence.
5. Record the actual result as successful or unsuccessful.
6. Present the completed work to the assessor for sign-off.

### Designed Evidence Names for the Remaining Backup Work

- `SS_557-16-3_VM_Backup_Export_To_Host_E_Drive.jpg`
- `SS_557-16-4_VM_Backup_Files_Verified_On_Host_E_Drive.jpg`

If `BAA-STOR1` is repaired and a working share is created instead, use:

- `SS_557-16-3_VM_Backup_Export_To_BAA_STOR1.jpg`
- `SS_557-16-4_VM_Backup_Files_Verified_On_BAA_STOR1.jpg`

Only use the evidence names that match the destination actually used.

---

# PART H — FINAL TECHNICAL SUMMARY

## 39. Final Environment

The lab now provides a working integrated environment containing:

- Two-site Active Directory.
- Redundant DNS.
- Windows DHCP.
- Windows 11 domain clients.
- Windows file and print services.
- NTFS and AD group-based permissions.
- Ubuntu domain integration.
- Samba/Winbind cross-platform SMB.
- SFTP.
- WordPress/Apache.
- iRedMail/Roundcube.
- Squid proxy.
- Windows and Linux firewall rules.
- NTP/time synchronisation.
- Azure Arc and Azure Update Manager.
- Windows backup and tested restore.
- Hyper-V clustering and controlled VM movement.
- pfSense VLAN routing and segmentation.
- Accounts and Administration workstations on separate VLANs.
- Resource pool and checkpoint evidence.
- Centralised screenshot and documentation repositories.

---

## 40. Lessons Learned

1. DNS configuration is critical to Active Directory availability. A second domain controller is not useful to clients unless the secondary DNS address is actually distributed.
2. A successful-looking domain join command does not always prove that Samba’s trust database is valid. `net ads testjoin` and `wbinfo -t` must be verified.
3. DHCP reservations for Linux are more reliable when the client presents the MAC address rather than a DUID.
4. Hyper-V Enhanced Session is effectively an RDP session and requires the appropriate user rights.
5. Permission tests must use users with clearly different group memberships.
6. A backup is not complete until the exported files or restored data are verified.
7. Strong evidence should show the device name, actual configuration, test result, and final status in the same chain.
8. Reusing the existing lab reduced risk and build time, but each reused service had to be checked for old names, URLs, certificates, and configuration.
9. Practical assessment evidence should be concise and directly tied to the marking criteria.
10. Real troubleshooting evidence is stronger than a perfect build with no recorded issues.

---

## 41. Source Documents Used for This Master Log

- `ICTNWK539_540_AT2_Part1_Bojie Qiao.docx`
- `ICTNWK539_540_AT2_Part2_Bojie_Qiao.xlsx`
- `ICTNWK539_540_Project_approval_Bojie Qiao.docx`
- `ICTNWK539_540_Test_Plan_Bojie Qiao(Before Implementation).docx`
- `ICTNWK539_540_AT3_PE_TQM_v1.docx`
- `ICTNWK539_540_AT3_MC_TQM_v1.docx`
- `ICTNWK539_540_Risk Assessment Template_LHO_TQM_v1.docx`
- `ICTNWK539_540_Trouble_shooting_Bojie Qiao.docx`
- `ICTNWK539_540_Report Template_LHO_TQM_v1.docx`
- `ICTNWK539-540 AT3 Screenshot Checklist.pdf`
- `AT3-Pre-implementation-test-plan.txt`
- `Update-about-the-wsus-part.txt`
- `logbook_BAA-BIG-LX2_Samba_Winbind.md`
- `Anzac_Airport_AT3_VM_Topology.md`
- `ICTNWK557_ICTNWK559_AT1_Part1_Bojie Qiao.docx`
- `ICTNWK557_559_AT2_O1_TQM_V2.docx`
- `Update-557559-AT2-plan.txt`
- Project checkpoints and verified command outputs recorded during implementation.

---

## 42. Log Closure Statement

The ICTNWK539/540 server design and AT3 implementation are documented as completed, including service testing, backup/restore, high availability, troubleshooting, and final reporting.

The ICTNWK557/559 virtualisation work is substantially completed. The remaining technical action is to finalise and verify the VM backup/export evidence for Task 16 and obtain assessor verification for the required observation checkpoints.

This log should be updated if the final backup destination, screenshot names, or assessor results change.
