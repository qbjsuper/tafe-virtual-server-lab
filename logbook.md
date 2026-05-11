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