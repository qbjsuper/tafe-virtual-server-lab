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