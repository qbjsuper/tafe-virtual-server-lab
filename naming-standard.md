# Naming Standard

This document records the naming standard for the TAFE Virtual Server Lab.

The lab is based on the ANZAC Airport case study.

## Lab Prefix

The lab prefix is:

BAA

BAA means:

Bojie ANZAC Airport

This prefix is used to clearly separate this lab from other labs and projects.

## Naming Format

The standard naming format is:

BAA-[SITE]-[ROLE][NUMBER]

Example:

BAA-BIG-DC1

## Name Parts

| Part | Meaning |
|---|---|
| BAA | Bojie ANZAC Airport lab |
| BIG | Big Site |
| SML | Small Site |
| ROLE | Device or server role |
| NUMBER | Sequence number |

## Site Codes

| Site | Code | Network |
|---|---|---|
| Big Site | BIG | 172.16.60.0/24 |
| Small Site | SML | 172.16.50.0/24 |

## Role Codes

| Role Code | Meaning |
|---|---|
| PFS | pfSense router/firewall |
| DC | Domain controller |
| WIN | Windows client |
| LNX | Linux server |
| FS | File server |
| WEB | Web server |
| BK | Backup server |
| MGMT | Management server or jump box |

## Core VM Names

| VM Name | Site | Role | IP Address |
|---|---|---|---|
| BAA-BIG-PFS1 | Big Site | pfSense gateway | 172.16.60.1 |
| BAA-BIG-DC1 | Big Site | First domain controller | 172.16.60.10 |
| BAA-BIG-WIN1 | Big Site | Windows client | 172.16.60.101 |
| BAA-BIG-LNX1 | Big Site | Linux server | 172.16.60.20 |
| BAA-SML-PFS1 | Small Site | pfSense gateway | 172.16.50.1 |
| BAA-SML-DC1 | Small Site | Second domain controller | 172.16.50.10 |
| BAA-SML-WIN1 | Small Site | Windows client | 172.16.50.101 |
| BAA-SML-LNX1 | Small Site | Linux server | 172.16.50.20 |

## Domain Naming

The Active Directory domain name is:

bojieanzac.com

The NetBIOS name is:

BOJIEANZAC

Example full computer names:

BAA-BIG-DC1.bojieanzac.com

BAA-SML-DC1.bojieanzac.com

## Active Directory Site Names

| AD Site Name | Subnet |
|---|---|
| BAA-BIG-SITE | 172.16.60.0/24 |
| BAA-SML-SITE | 172.16.50.0/24 |

## Hyper-V Switch Names

### Big Site

| Switch Name | Type | Purpose |
|---|---|---|
| BAA-BIG-WAN | External | pfSense WAN and upstream network |
| BAA-BIG-LAN | Private/Internal | Big Site internal lab network |

### Small Site

| Switch Name | Type | Purpose |
|---|---|---|
| BAA-SML-WAN | External | pfSense WAN and upstream network |
| BAA-SML-LAN | Private/Internal | Small Site internal lab network |

## Notes

This naming standard should be used for:

- Hyper-V VM names
- Windows Server computer names
- Linux server hostnames
- pfSense VM names
- documentation
- screenshots
- testing records

The goal is to keep the lab simple, clear, and easy to identify.