# Lab Topology

This document records the current topology for the TAFE Virtual Server Lab.

The lab is based on the ANZAC Airport case study.

## Domain

| Item                    | Value                              |
|-------------------------|------------------------------------|
| Active Directory domain | bojieanzac.com                     |
| NetBIOS name            | BOJIEANZAC                         |
| Lab prefix              | BAA                                |
| Design model            | One forest, one domain, two sites  |

## Site Overview

| Site       | Site Code | Network          | Gateway      | Domain Controller |
|------------|-----------|------------------|--------------|-------------------|
| Big Site   | BIG       | 172.16.60.0/24   | 172.16.60.1  | BAA-BIG-DC1       |
| Small Site | SML       | 172.16.50.0/24   | 172.16.50.1  | BAA-SML-DC1       |

## Core Virtual Machines

| VM Name      | Site       | Role                                  | IP Address    |
|--------------|------------|---------------------------------------|---------------|
| BAA-BIG-PFS1 | Big Site   | pfSense gateway/firewall              | 172.16.60.1   |
| BAA-BIG-DC1  | Big Site   | First domain controller, DNS, DHCP    | 172.16.60.10  |
| BAA-SML-PFS1 | Small Site | pfSense gateway/firewall              | 172.16.50.1   |
| BAA-SML-DC1  | Small Site | Second domain controller, DNS         | 172.16.50.10  |

## Network Design

### Big Site

| Item                 | Value          |
|----------------------|----------------|
| Network              | 172.16.60.0/24 |
| Gateway              | BAA-BIG-PFS1   |
| Gateway IP           | 172.16.60.1    |
| Domain controller    | BAA-BIG-DC1    |
| Domain controller IP | 172.16.60.10   |

### Small Site

| Item                 | Value          |
|----------------------|----------------|
| Network              | 172.16.50.0/24 |
| Gateway              | BAA-SML-PFS1   |
| Gateway IP           | 172.16.50.1    |
| Domain controller    | BAA-SML-DC1    |
| Domain controller IP | 172.16.50.10   |

## Site-to-Site Connectivity

The two pfSense routers are connected using an IPsec site-to-site VPN.

| Side       | pfSense      | Local LAN      | Remote LAN     |
|------------|--------------|----------------|----------------|
| Big Site   | BAA-BIG-PFS1 | 172.16.60.0/24 | 172.16.50.0/24 |
| Small Site | BAA-SML-PFS1 | 172.16.50.0/24 | 172.16.60.0/24 |

The purpose of the VPN is to allow communication between:

```text
172.16.60.0/24
```

and:

```text
172.16.50.0/24
```

This allows the Small Site domain controller to communicate with the Big Site domain controller.

## Current Logical Topology

```text
                 Upstream / Physical Network
                           |
          -------------------------------------
          |                                   |
    BAA-BIG-PFS1                        BAA-SML-PFS1
    pfSense gateway                     pfSense gateway
    LAN: 172.16.60.1                    LAN: 172.16.50.1
          |                                   |
    Big Site LAN                         Small Site LAN
    172.16.60.0/24                       172.16.50.0/24
          |                                   |
    BAA-BIG-DC1                         BAA-SML-DC1
    172.16.60.10                        172.16.50.10
    AD DS / DNS / DHCP                  AD DS / DNS
```

## Role Separation

| Role                                      | Device            |
|-------------------------------------------|-------------------|
| Routing between local LAN and upstream    | pfSense           |
| Firewall rules                            | pfSense           |
| NAT / internet access                     | pfSense           |
| Site-to-site VPN                          | pfSense           |
| Active Directory Domain Services          | Windows Server DC |
| Internal DNS for bojieanzac.com           | Windows Server DC |
| DHCP                                      | Windows Server DC |
| User and group management                 | Windows Server DC |
| Group Policy                              | Windows Server DC |

## DNS Design

Domain clients should use the local domain controller as DNS.

| Site       | Client DNS Server |
|------------|-------------------|
| Big Site   | 172.16.60.10      |
| Small Site | 172.16.50.10      |

DNS forwarding design:

```text
Domain clients
    ↓
Local Windows DC DNS
    ↓
pfSense DNS Resolver
    ↓
Internet DNS
```

## DHCP Design

Final DHCP design:

| Site       | DHCP Server | Scope                         |
|------------|-------------|-------------------------------|
| Big Site   | BAA-BIG-DC1 | 172.16.60.100 - 172.16.60.199 |
| Small Site | BAA-SML-DC1 | 172.16.50.100 - 172.16.50.199 |

Big Site DHCP is already configured.

Small Site DHCP is planned after confirming the second domain controller is stable.

## Active Directory Site Plan

The final AD Sites and Services design will use:

| AD Site      | Subnet         |
|--------------|----------------|
| BAA-BIG-SITE | 172.16.60.0/24 |
| BAA-SML-SITE | 172.16.50.0/24 |

This will replace the default site-only design and make the domain topology match the physical lab topology.
