# Lab Logbook

## 2026-05-11

Started the TAFE virtual server lab documentation repository.

Confirmed the current lab topology:

| Site | Hyper-V Host | Internal Network | Gateway | Server |
|---|---|---|---|---|
| Big Site | Big Hyper-V host | 172.16.60.0/24 | pfSense | Windows Server |
| Small Site | Small Hyper-V host | 172.16.50.0/24 | pfSense | Windows Server |

Current plan:

- use pfSense as the gateway for each site
- configure one Windows Server at each site
- build one Active Directory domain: bojieanzac.com
- use the Big Site server as the first domain controller
- use the Small Site server as the second domain controller
- document all build steps and testing evidence

## 2026-05-11

Decision: rebuild the current lab VMs from start.

Reason:

The lab is still early, and only four VMs are required at this stage. Rebuilding gives cleaner VM names, cleaner screenshots, cleaner VHDX/storage names, and easier documentation.

Final core VM names:

| VM Name | Site | Role | Network |
|---|---|---|---|
| BAA-BIG-PFS1 | Big Site | pfSense gateway | 172.16.60.0/24 |
| BAA-BIG-DC1 | Big Site | Windows Server domain controller | 172.16.60.0/24 |
| BAA-SML-PFS1 | Small Site | pfSense gateway | 172.16.50.0/24 |
| BAA-SML-DC1 | Small Site | Windows Server domain controller | 172.16.50.0/24 |

Next action:

Recreate the four VMs with the final naming standard before starting Active Directory configuration.

## 2026-05-11

Foundation VM rebuild completed.

The four foundation VMs are now up and running:

| VM Name | Site | Role | Network |
|---|---|---|---|
| BAA-BIG-PFS1 | Big Site | pfSense gateway | 172.16.60.0/24 |
| BAA-BIG-DC1 | Big Site | Windows Server domain controller candidate | 172.16.60.0/24 |
| BAA-SML-PFS1 | Small Site | pfSense gateway | 172.16.50.0/24 |
| BAA-SML-DC1 | Small Site | Windows Server domain controller candidate | 172.16.50.0/24 |

Next steps:

- configure pfSense LAN interfaces
- configure static IP addresses on both Windows Servers
- test local gateway connectivity
- configure inter-site routing/VPN
- start Active Directory only after network connectivity is confirmed

## 2026-05-14

Completed the first stage of the multi-site Active Directory foundation.

### Current lab status

The four foundation VMs are running:

| VM Name | Site | Role | Network |
|---|---|---|---|
| BAA-BIG-PFS1 | Big Site | pfSense gateway | 172.16.60.0/24 |
| BAA-BIG-DC1 | Big Site | First domain controller | 172.16.60.0/24 |
| BAA-SML-PFS1 | Small Site | pfSense gateway | 172.16.50.0/24 |
| BAA-SML-DC1 | Small Site | Second domain controller | 172.16.50.0/24 |

### Big Site domain controller

BAA-BIG-DC1 has been configured as the first domain controller for:

bojieanzac.com

Roles configured:

- Active Directory Domain Services
- DNS Server
- DHCP Server

Confirmed checks:

- AD DS service is running
- DNS service is running
- Netlogon service is running
- Kerberos KDC service is running
- SYSVOL and NETLOGON shares are present
- internal DNS resolution works
- AD SRV records resolve correctly
- external DNS forwarding works
- DHCP server is authorised in Active Directory
- DHCP scope BAA-BIG-LAN is active

Big Site DHCP scope:

| Setting | Value |
|---|---|
| Scope name | BAA-BIG-LAN |
| Network | 172.16.60.0/24 |
| Range | 172.16.60.100 - 172.16.60.199 |
| Gateway | 172.16.60.1 |
| DNS server | 172.16.60.10 |
| DNS domain | bojieanzac.com |

### Site-to-site pfSense connection

Configured IPsec site-to-site VPN between:

| Side | pfSense | LAN |
|---|---|---|
| Big Site | BAA-BIG-PFS1 | 172.16.60.0/24 |
| Small Site | BAA-SML-PFS1 | 172.16.50.0/24 |

Ping test across the tunnel was successful.

Result:

172.16.50.0/24 can reach 172.16.60.10.

This confirms that the Small Site network can reach the Big Site domain controller through the IPsec VPN.

### Small Site domain controller

BAA-SML-DC1 was renamed correctly before promotion.

BAA-SML-DC1 was joined to the existing domain and promoted as the second domain controller for:

bojieanzac.com

Roles configured:

- Active Directory Domain Services
- DNS Server
- Global Catalog

Confirmed checks:

- BAA-SML-DC1 can resolve bojieanzac.com
- BAA-SML-DC1 can resolve BAA-BIG-DC1.bojieanzac.com
- BAA-SML-DC1 can resolve BAA-SML-DC1.bojieanzac.com
- SYSVOL share is present
- AD replication summary shows 0 failures

Replication check:

repadmin /replsummary showed 0 failures between BAA-BIG-DC1 and BAA-SML-DC1.

### Notes

dcdiag /q showed DFSREvent and SystemLog warnings on BAA-SML-DC1.

These appear to be recent event log warnings after promotion and SYSVOL sharing. AD replication itself is healthy because repadmin /replsummary shows 0 failures.

The next actions are:

- create proper AD sites and subnets
- map 172.16.60.0/24 to BAA-BIG-SITE
- map 172.16.50.0/24 to BAA-SML-SITE
- configure DHCP on BAA-SML-DC1
- verify client DHCP and domain join