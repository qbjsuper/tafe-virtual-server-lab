# Lab Requirements Mapping

This document maps the ANZAC Airport virtual server lab requirements to the implemented lab design.

The purpose of this file is to show how the lab design meets the expected practical requirements.

## Lab Scenario

The lab is based on the ANZAC Airport case study.

The environment is designed as a small business-style multi-site network. It uses virtual machines to simulate servers, clients, gateways, and site-to-site connectivity.

## Implemented Design Summary

| Area | Implementation |
|---|---|
| Business scenario | ANZAC Airport virtual infrastructure |
| Domain name | bojieanzac.com |
| NetBIOS name | BOJIEANZAC |
| Site model | Two-site design |
| Big Site subnet | 172.16.60.0/24 |
| Small Site subnet | 172.16.50.0/24 |
| Gateway platform | pfSense |
| Server platform | Windows Server |
| Client platforms | Windows 11 and Ubuntu Linux |
| Virtualisation platform | Hyper-V |

## Core Lab Components

| Component | Implementation | Status |
|---|---|---|
| Virtual network design | Two separated site networks | Complete |
| Gateway and firewall | pfSense gateway for each site | Complete |
| Site-to-site VPN | IPsec VPN between Big Site and Small Site | Complete |
| Active Directory domain | bojieanzac.com | Complete |
| Domain controllers | BAA-BIG-DC1 and BAA-SML-DC1 | Complete |
| DNS | AD-integrated DNS on domain controllers | Complete |
| DHCP | DHCP scope for client addressing | In progress / configured as required |
| Windows client | BAA-BIG-WIN1 joined to the domain | Complete |
| Linux client | BAA-BIG-LX1 joined to the domain | Complete |
| File sharing | Samba file sharing | Planned |
| Testing evidence | Command output and screenshots | In progress |

## Requirement Mapping

| Requirement Area | Lab Implementation | Evidence File |
|---|---|---|
| Create and manage virtual machines | pfSense, Windows Server, Windows 11, and Ubuntu VMs created | vm-inventory.md |
| Configure virtual networking | Big Site and Small Site networks created | ip-addressing.md |
| Configure IP addressing | Static IPs for servers and gateways, DHCP for clients | ip-addressing.md |
| Configure routing between networks | pfSense gateways route traffic between site networks | topology.md |
| Configure firewall and gateway services | pfSense used as firewall and gateway at both sites | topology.md |
| Configure site-to-site connectivity | IPsec VPN connects 172.16.60.0/24 and 172.16.50.0/24 | testing-notes.md |
| Install and configure Windows Server | Domain controllers deployed on Windows Server | build-notes.md |
| Configure Active Directory Domain Services | Domain bojieanzac.com created | build-notes.md |
| Configure DNS | AD DNS used for domain name resolution | testing-notes.md |
| Configure DHCP | DHCP scope used for client machines | ip-addressing.md |
| Join Windows client to domain | Windows 11 client joined to bojieanzac.com | testing-notes.md |
| Join Linux client to domain | Ubuntu client joined using realmd and SSSD | testing-notes.md |
| Test domain health | dcdiag used to check domain controller health | testing-notes.md |
| Test AD replication | repadmin used to confirm replication status | testing-notes.md |
| Record documentation | Lab files created in GitHub repository | README.md |

## Domain Controller Design

| Server | Site | Role | IP Address |
|---|---|---|---:|
| BAA-BIG-DC1 | Big Site | First domain controller, DNS, DHCP | 172.16.60.10 |
| BAA-SML-DC1 | Small Site | Additional domain controller, DNS | 172.16.50.10 |

The two-domain-controller design provides:

- domain service availability across both sites
- DNS redundancy
- Active Directory replication practice
- a more realistic business network design

## Client Design

| Client | Site | Operating System | Purpose |
|---|---|---|---|
| BAA-BIG-WIN1 | Big Site | Windows 11 | Test domain join, DNS, authentication, and GPO |
| BAA-BIG-LX1 | Big Site | Ubuntu Linux | Test Linux integration with Active Directory |
| BAA-SML-WIN1 | Small Site | Windows 11 | Optional small-site client testing |
| BAA-SML-LX1 | Small Site | Ubuntu Linux | Optional small-site Linux testing |

## Testing Plan

The following tests are used to verify the lab.

| Test | Tool / Command | Expected Result |
|---|---|---|
| Domain controller health | dcdiag /q | No critical AD DS errors |
| AD replication | repadmin /replsummary | 0 replication failures |
| DNS lookup | nslookup bojieanzac.com | Domain resolves successfully |
| AD SRV records | nslookup -type=SRV _ldap._tcp.dc._msdcs.bojieanzac.com | Domain controller records are returned |
| Domain controller discovery | nltest /dsgetdc:bojieanzac.com | A domain controller is found |
| Linux domain discovery | realm discover bojieanzac.com | AD domain is discovered |
| Linux domain join check | realm list | Linux machine shows as a configured domain member |
| Cross-site connectivity | ping between site subnets | Traffic passes through VPN |

## Current Completion Status

| Task | Status |
|---|---|
| Create Big Site pfSense VM | Complete |
| Create Small Site pfSense VM | Complete |
| Create Big Site domain controller | Complete |
| Create Small Site domain controller | Complete |
| Create bojieanzac.com domain | Complete |
| Configure site-to-site VPN | Complete |
| Confirm cross-site connectivity | Complete |
| Confirm AD replication | Complete |
| Join Windows client to domain | Complete |
| Join Linux client to domain | Complete |
| Document IP addressing | Complete |
| Document lab requirements mapping | Complete |
| Add Samba file sharing | Planned |
| Add backup and restore testing | Planned |
| Add screenshots as final evidence | In progress |

## Notes

This lab is designed for learning and assessment purposes. It is not a production network.

The design focuses on:

- clear network separation
- realistic site naming
- Active Directory fundamentals
- DNS and DHCP configuration
- site-to-site VPN concepts
- Windows and Linux domain integration
- repeatable testing and documentation