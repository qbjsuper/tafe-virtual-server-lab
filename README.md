# TAFE Virtual Server Lab

This repository is my personal logbook for the TAFE virtual server practical lab.

The lab is based on the ANZAC Airport case study and uses a two-site virtual infrastructure design.

## Lab Purpose

The purpose of this lab is to practise:

- virtual server deployment
- multi-site networking
- pfSense gateway configuration
- Windows Server domain controller deployment
- Active Directory Domain Services
- DNS and DHCP services
- Windows and Linux server management
- backup, restore, and testing documentation

## Current Design

Domain name:

bojieanzac.com

Sites:

| Site | Internal Network | Gateway | Main Server |
|---|---|---|---|
| Big Site | 172.16.60.0/24 | pfSense Big Site | Windows Server DC |
| Small Site | 172.16.50.0/24 | pfSense Small Site | Windows Server DC |

## Main Requirement File

The lab requirement instruction file is:

2_virtual_server_prac.xlsx

## Documentation Style

This repository is not a formal report.

It is a simple personal logbook to record:

- what I built
- what settings I used
- what problems I found
- how I fixed them
- what evidence I captured

## Naming Standard

This lab uses the following naming format:

BAA-[SITE]-[ROLE][NUMBER]

Example:

BAA-BIG-DC1

The full naming standard is recorded in:

naming-standard.md