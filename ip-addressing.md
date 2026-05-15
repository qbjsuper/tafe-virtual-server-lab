# IP Addressing Plan

This document records the IP addressing design for the ANZAC Airport virtual server lab.

The lab uses two separated virtual sites:

- Big Site
- Small Site

Each site has its own subnet, gateway, domain controller, DNS service, and client machines.

## Network Summary

| Site | Site Code | Network Address | Subnet Mask | CIDR | Gateway |
|---|---|---:|---:|---:|---:|
| Big Site | BIG | 172.16.60.0 | 255.255.255.0 | /24 | 172.16.60.1 |
| Small Site | SML | 172.16.50.0 | 255.255.255.0 | /24 | 172.16.50.1 |

## Big Site IP Addressing

| Device / VM | Hostname | Role | IP Address | Notes |
|---|---|---|---:|---|
| pfSense Gateway | BAA-BIG-PFS1 | Gateway, firewall, VPN endpoint | 172.16.60.1 | Default gateway for Big Site |
| Domain Controller | BAA-BIG-DC1 | AD DS, DNS, DHCP | 172.16.60.10 | Primary domain controller |
| Windows Client | BAA-BIG-WIN1 | Domain-joined Windows client | DHCP | Used for domain and GPO testing |
| Linux Client | BAA-BIG-LX1 | Domain-joined Linux client | DHCP or static | Joined to domain using realmd and SSSD |

## Small Site IP Addressing

| Device / VM | Hostname | Role | IP Address | Notes |
|---|---|---|---:|---|
| pfSense Gateway | BAA-SML-PFS1 | Gateway, firewall, VPN endpoint | 172.16.50.1 | Default gateway for Small Site |
| Domain Controller | BAA-SML-DC1 | AD DS, DNS | 172.16.50.10 | Additional domain controller |
| Windows Client | BAA-SML-WIN1 | Domain-joined Windows client | DHCP | Planned or optional client |
| Linux Client | BAA-SML-LX1 | Domain-joined Linux client | DHCP or static | Planned or optional client |

## DNS Settings

Active Directory requires reliable DNS resolution. Domain-joined machines should use the domain controllers as their DNS servers.

### Big Site DNS

| Client Location | Preferred DNS | Alternate DNS |
|---|---:|---:|
| Big Site clients | 172.16.60.10 | 172.16.50.10 |

### Small Site DNS

| Client Location | Preferred DNS | Alternate DNS |
|---|---:|---:|
| Small Site clients | 172.16.50.10 | 172.16.60.10 |

## Domain Information

| Item | Value |
|---|---|
| AD domain name | bojieanzac.com |
| NetBIOS name | BOJIEANZAC |
| Forest model | Single forest |
| Domain model | Single domain |
| Site model | Two-site lab design |

## DHCP Plan

DHCP is provided by Windows Server where required.

### Big Site DHCP Scope

| Item | Value |
|---|---:|
| Scope network | 172.16.60.0/24 |
| DHCP range | 172.16.60.100 - 172.16.60.199 |
| Default gateway | 172.16.60.1 |
| DNS server 1 | 172.16.60.10 |
| DNS server 2 | 172.16.50.10 |
| Domain name | bojieanzac.com |

### Small Site DHCP Scope

| Item | Value |
|---|---:|
| Scope network | 172.16.50.0/24 |
| DHCP range | 172.16.50.100 - 172.16.50.199 |
| Default gateway | 172.16.50.1 |
| DNS server 1 | 172.16.50.10 |
| DNS server 2 | 172.16.60.10 |
| Domain name | bojieanzac.com |

## Reserved IP Address Ranges

| Range | Purpose |
|---|---|
| .1 - .9 | Gateways and network infrastructure |
| .10 - .29 | Servers and domain controllers |
| .30 - .49 | Future servers or services |
| .50 - .99 | Static client or testing devices |
| .100 - .199 | DHCP clients |
| .200 - .254 | Reserved for future use |

## Routing Design

The two site networks are connected through pfSense site-to-site VPN.

| Source Network | Destination Network | Connection Type |
|---|---|---|
| 172.16.60.0/24 | 172.16.50.0/24 | pfSense IPsec VPN |
| 172.16.50.0/24 | 172.16.60.0/24 | pfSense IPsec VPN |

The VPN allows:

- domain controller replication
- DNS queries between sites
- client access to domain services
- cross-site administration and testing

## Important Notes

- Domain-joined clients should not use public DNS servers directly.
- DNS should point to the domain controllers, not to pfSense or external DNS.
- pfSense is used as the gateway and firewall, not as the main DNS server for Active Directory clients.
- Static IP addresses are used for infrastructure servers.
- DHCP is used for normal client machines.