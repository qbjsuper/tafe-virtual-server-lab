# Nested Hyper-V Cluster Checkpoint — 2026-05-25

## 1. Current Lab Context

This checkpoint records the work completed today for the nested Hyper-V failover cluster lab.

The lab is part of the `bojieanzac.com` TAFE virtual server / cluster environment.

```text
Domain:        bojieanzac.com
Cluster:       BAA-NEST-CL1
Cluster nodes: BAA-BIG-Nest1
               BAA-SML-Nest1
Storage host:  BAA-STOR1
Storage IP:    172.16.50.101
CSV path:      C:\ClusterStorage\Volume1
Test VM:       BAA-TEST-VM01
```

The main goal is to prove that a nested Hyper-V failover cluster can run a highly available VM from shared iSCSI/CSV storage and later live migrate that VM between the two nested Hyper-V nodes.

---

## 2. Nested Hyper-V Cluster Nodes

Two new nested Hyper-V nodes were used:

```text
BAA-BIG-Nest1
BAA-SML-Nest1
```

Purpose:

```text
Physical Hyper-V host
└── Nested Windows Server Hyper-V nodes
    └── Clustered nested test VM
```

Both nodes are domain-joined to:

```text
bojieanzac.com
```

Cluster name:

```text
BAA-NEST-CL1
```

Cluster node health was confirmed with:

```powershell
Get-ClusterNode
```

Confirmed result:

```text
Name           State  Type
----           -----  ----
BAA-BIG-Nest1  Up     Node
BAA-SML-Nest1  Up     Node
```

Result:

```text
Both cluster nodes are up and participating in the cluster.
```

---

## 3. Shared Storage and CSV

Storage server:

```text
BAA-STOR1
172.16.50.101
```

Storage design:

```text
BAA-STOR1 provides iSCSI storage.
Both BAA-BIG-Nest1 and BAA-SML-Nest1 connect to the iSCSI target.
The shared disk is added to the failover cluster.
The shared disk is converted to Cluster Shared Volume.
```

Current CSV path:

```text
C:\ClusterStorage\Volume1
```

CSV check command:

```powershell
Get-ClusterSharedVolume | Select-Object Name, State, OwnerNode
```

Confirmed result:

```text
Name            State   OwnerNode
----            -----   ---------
Cluster Disk 1  Online  BAA-BIG-Nest1
```

Interpretation:

```text
The Cluster Shared Volume is online and currently coordinated by BAA-BIG-Nest1.
```

---

## 4. iSCSI Connectivity

The iSCSI session was checked with:

```powershell
Get-IscsiSession
```

Confirmed important fields:

```text
IsConnected:   True
IsPersistent:  True
Target:        iqn.1991-05.com.microsoft:baa-stor1-baa-nest-csv-tgt01-target
```

TCP connectivity to the iSCSI target was checked with:

```powershell
Test-NetConnection 172.16.50.101 -Port 3260
```

Confirmed result:

```text
ComputerName     : 172.16.50.101
RemotePort       : 3260
InterfaceAlias   : vEthernet (BAA-NEST-EXT-SW)
SourceAddress    : 172.16.60.100
TcpTestSucceeded : True
```

Interpretation:

```text
BAA-BIG-Nest1 can reach BAA-STOR1 over iSCSI TCP port 3260.
Storage connectivity is currently working.
```

Note:

```text
The storage path from BAA-BIG-Nest1 to BAA-STOR1 crosses from the BIG-side network to the SML-side storage network.
This works, but may be slower than running heavy storage operations from BAA-SML-Nest1.
```

---

## 5. Test VM Creation

Test VM name:

```text
BAA-TEST-VM01
```

Storage location:

```text
C:\ClusterStorage\Volume1\VMs\BAA-TEST-VM01
```

Confirmed VM files:

```text
C:\ClusterStorage\Volume1\VMs\BAA-TEST-VM01
C:\ClusterStorage\Volume1\VMs\BAA-TEST-VM01\Virtual Hard Disks
C:\ClusterStorage\Volume1\VMs\BAA-TEST-VM01\Virtual Machines
C:\ClusterStorage\Volume1\VMs\BAA-TEST-VM01\Virtual Hard Disks\BAA-TEST-VM01.vhdx
C:\ClusterStorage\Volume1\VMs\BAA-TEST-VM01\Virtual Machines\905379D1-98EA-49C8-BA75-B6E84EA2DFC5.vmcx
C:\ClusterStorage\Volume1\VMs\BAA-TEST-VM01\Virtual Machines\905379D1-98EA-49C8-BA75-B6E84EA2DFC5.vmgs
C:\ClusterStorage\Volume1\VMs\BAA-TEST-VM01\Virtual Machines\905379D1-98EA-49C8-BA75-B6E84EA2DFC5.VMRS
```

Current VM design:

```text
Name:             BAA-TEST-VM01
Generation:       2
Memory:           2048 MB currently
Storage:          VHDX on CSV
Clustered:        Yes
Guest OS:         Ubuntu Server test VM
Purpose:          Clustered VM / live migration test
```

---

## 6. Clustered VM Role

The test VM was added as a clustered role with:

```powershell
Add-ClusterVirtualMachineRole -VMName "BAA-TEST-VM01"
```

The current cluster role check shows:

```powershell
Get-ClusterGroup -Name "BAA-TEST-VM01"
```

Current confirmed result:

```text
Name           OwnerNode      State
----           ---------      -----
BAA-TEST-VM01  BAA-BIG-Nest1  Offline
```

Clustered VM resources were checked with:

```powershell
Get-ClusterResource | Where-Object OwnerGroup -eq "BAA-TEST-VM01" | Format-Table Name, ResourceType, State, OwnerNode, OwnerGroup -AutoSize
```

Current confirmed result:

```text
Name                                         ResourceType                   State    OwnerNode      OwnerGroup
----                                         ------------                   -----    ---------      ----------
Virtual Machine BAA-TEST-VM01                Virtual Machine                Offline  BAA-BIG-Nest1  BAA-TEST-VM01
Virtual Machine Configuration BAA-TEST-VM01  Virtual Machine Configuration  Online   BAA-BIG-Nest1  BAA-TEST-VM01
```

Interpretation:

```text
The VM itself is powered off, so the Virtual Machine resource is Offline.
The VM configuration resource is Online, which is normal for a clustered VM that is currently off.
```

---

## 7. Offline Move vs Live Migration

A key troubleshooting point was discovered.

This command failed while the VM role was offline:

```powershell
Move-ClusterVirtualMachineRole -Name "BAA-TEST-VM01" -Node "BAA-SML-Nest1"
```

Error:

```text
The virtual machine resource is not in an appropriate state for this operation.
```

Reason:

```text
Move-ClusterVirtualMachineRole is better suited for moving an online/running clustered VM role, especially when using live migration.
```

For an offline VM role, this command worked:

```powershell
Move-ClusterGroup -Name "BAA-TEST-VM01" -Node "BAA-SML-Nest1"
```

Result:

```text
Offline clustered VM role movement works.
```

Important rule:

```text
Offline clustered role movement:
Use Move-ClusterGroup.

Running VM live migration:
Use Move-ClusterVirtualMachineRole with -MigrationType Live.
```

---

## 8. Ubuntu ISO Handling

Ubuntu ISO filename:

```text
ubuntu-26.04-live-server-amd64.iso
```

ISO location:

```text
C:\ClusterStorage\Volume1\ISO\ubuntu-26.04-live-server-amd64.iso
```

Reason for storing the ISO on CSV:

```text
Both cluster nodes can access the same path.
This avoids migration or startup problems caused by local-only ISO paths.
```

The ISO was attached from the CSV path.

Important next cleanup step:

```powershell
Set-VMDvdDrive -ComputerName "BAA-BIG-Nest1" -VMName "BAA-TEST-VM01" -Path $null
```

Purpose:

```text
Detach the Ubuntu installer ISO after installation.
Prevent the VM from booting back into the installer.
Remove an unnecessary dependency before live migration.
```

---

## 9. Ubuntu Installation and Storage Warning

The Ubuntu installer started successfully.

The installer log showed normal disk activity such as:

```text
/dev/sda
sgdisk
udevadm
partitioning
SUCCESS configuring partition
```

However, during installation, the VM temporarily entered:

```text
State:  PausedCritical
Status: Disk(s) encountered critical IO errors
```

Interpretation:

```text
Hyper-V temporarily paused the VM because the VHDX storage path encountered critical I/O delay or error.
```

Likely cause:

```text
Nested Hyper-V + CSV + iSCSI + routed storage path created a temporary storage stall during heavy disk writes.
```

The VM later recovered to:

```text
State:    Running
CPUUsage: 5
Status:   Operating normally
```

Conclusion:

```text
This was a real warning, but not a fatal failure.
The test VM recovered and later became usable.
```

Recommendation:

```text
Avoid heavy install/update work from the node farther away from storage.
For heavy disk operations, prefer BAA-SML-Nest1 because BAA-STOR1 is in the SML-side network.
```

---

## 10. Ubuntu Test VM Status

The test VM was confirmed as up and running after the Ubuntu install process.

Later, the VM was shut down cleanly/offline.

Current Hyper-V state check:

```powershell
Get-VM -ComputerName "BAA-BIG-Nest1" -Name "BAA-TEST-VM01" | Select-Object Name, State, CPUUsage, MemoryAssigned, Uptime, Status
```

Current confirmed result:

```text
Name           : BAA-TEST-VM01
State          : Off
CPUUsage       : 0
MemoryAssigned : 0
Uptime         : 00:00:00
Status         : Operating normally
```

Interpretation:

```text
The VM is currently powered off and healthy.
```

---

## 11. Domain Join Decision

Question considered:

```text
Should BAA-TEST-VM01 join bojieanzac.com now?
```

Decision:

```text
Not yet.
```

Reason:

```text
The current purpose is to test Hyper-V cluster, CSV, VM startup, and live migration.
Joining Ubuntu to AD would add extra variables such as DNS, Kerberos, SSSD, time sync, and domain login.
```

Recommended sequence:

```text
1. Confirm Ubuntu boots normally.
2. Confirm network/time sync works.
3. Detach ISO.
4. Reboot once from disk.
5. Test live migration.
6. Confirm VM stays running after migration.
7. Join to bojieanzac.com later as a separate milestone.
```

---

## 12. Ubuntu Time Sync Plan

Recommended time sync tool:

```text
chrony
```

Basic Ubuntu commands:

```bash
sudo apt update
sudo apt install -y chrony
sudo systemctl enable --now chrony
systemctl status chrony
chronyc sources -v
chronyc tracking
```

For later AD/domain integration, point Ubuntu time sync to a domain controller:

```text
server BAA-BIG-DC1.bojieanzac.com iburst
```

or use the DC IP address if DNS is not ready:

```text
server <DC-IP-ADDRESS> iburst
```

Reason:

```text
Kerberos and domain authentication depend on accurate time sync.
```

---

## 13. Nested VM Network / External Switch

The Ubuntu test VM initially did not have external network access.

Design requirement for nested Hyper-V internet access:

```text
Physical Hyper-V host
└── Nest1 VM network adapter with MAC spoofing enabled
    └── Nested Hyper-V External vSwitch
        └── BAA-TEST-VM01
```

Nested switch name involved:

```text
BAA-NEST-EXT-SW
```

Important live migration rule:

```text
The same vSwitch name must exist on both cluster nodes.
```

So `BAA-NEST-EXT-SW` should exist on both:

```text
BAA-BIG-Nest1
BAA-SML-Nest1
```

Outer physical Hyper-V host requirement:

```powershell
Set-VMNetworkAdapter -VMName "BAA-BIG-Nest1" -MacAddressSpoofing On
Set-VMNetworkAdapter -VMName "BAA-SML-Nest1" -MacAddressSpoofing On
```

Purpose:

```text
Allow traffic from nested VMs behind the Nest1 Hyper-V hosts to pass through the outer Hyper-V switch.
```

---

## 14. SR-IOV Decision

For the nested External vSwitch, SR-IOV should remain disabled.

Correct setting:

```text
External network:                                                Yes
Allow management operating system to share this network adapter: Yes
Enable single-root I/O virtualization (SR-IOV):                  No
```

Reason:

```text
SR-IOV is not needed for this nested Hyper-V cluster lab.
It depends on physical NIC, firmware, driver, and host support.
It can reduce flexibility and is not required for live migration testing.
```

---

## 15. Current Confirmed State

Latest confirmed cluster state:

```text
Cluster nodes:
BAA-BIG-Nest1    Up
BAA-SML-Nest1    Up

Cluster groups:
Available Storage    Offline
BAA-TEST-VM01        Offline
Cluster Group        Online

CSV:
Cluster Disk 1       Online
OwnerNode:           BAA-BIG-Nest1

Test VM:
BAA-TEST-VM01        Off
Status:              Operating normally

Test VM cluster resources:
Virtual Machine resource:                Offline
Virtual Machine Configuration resource:  Online

iSCSI:
Connected:           True
Port 3260:           True
```

Interpretation:

```text
The cluster is healthy.
The CSV is online.
The test VM is cleanly powered off.
The VM configuration is managed by the cluster.
Storage connectivity is working.
```

`Available Storage` being offline is acceptable because the shared disk has already been converted to CSV.

The important resource is:

```text
Cluster Disk 1    Online
```

---

## 16. Next Recommended Steps

### Step 1 — Detach the Ubuntu ISO

```powershell
Set-VMDvdDrive -ComputerName "BAA-BIG-Nest1" -VMName "BAA-TEST-VM01" -Path $null
```

This detaches the Ubuntu installer ISO from the VM DVD drive.

Verify:

```powershell
Get-VMDvdDrive -ComputerName "BAA-BIG-Nest1" -VMName "BAA-TEST-VM01" | Select-Object VMName, Path
```

Expected result:

```text
Path should be blank.
```

### Step 2 — Start VM from disk

```powershell
Start-ClusterGroup -Name "BAA-TEST-VM01"
```

This starts the clustered VM role through Failover Clustering.

Verify:

```powershell
Get-ClusterGroup -Name "BAA-TEST-VM01"
Get-VM -ComputerName "BAA-BIG-Nest1" -Name "BAA-TEST-VM01" | Select-Object Name, State, CPUUsage, MemoryAssigned, Uptime, Status
```

Expected result:

```text
Cluster role: Online
VM state:     Running
Status:       Operating normally
```

### Step 3 — Test live migration

After the VM is running normally:

```powershell
Move-ClusterVirtualMachineRole -Name "BAA-TEST-VM01" -Node "BAA-SML-Nest1" -MigrationType Live
```

This performs the real live migration test from `BAA-BIG-Nest1` to `BAA-SML-Nest1`.

Verify after migration:

```powershell
Get-ClusterGroup -Name "BAA-TEST-VM01"
Get-VM -ComputerName "BAA-SML-Nest1" -Name "BAA-TEST-VM01" | Select-Object Name, State, Status, Uptime
```

Expected result:

```text
OwnerNode changes to BAA-SML-Nest1.
VM stays Running.
Uptime should continue.
```

---

## 17. Current Milestone Status

```text
Cluster foundation:        Working
Both Nest1 nodes:          Working
Domain membership:         Working
BAA-STOR1 iSCSI:           Working
CSV:                       Working
Clustered test VM role:    Working
Offline role move:         Working
Ubuntu boot/install:       Mostly working, with one temporary storage I/O warning
Nested external switch:    Being configured
Live migration:            Next major test
Domain join for Ubuntu:    Later milestone
```

---

## 18.About BAA-TEST-VM01 
Nested VM had no IPv4 address because Netplan did not enable DHCPv4. Created a new Netplan YAML file with dhcp4: true for the active interface, applied it with netplan apply, and IPv4 networking started working.


---

## 19.Live Migration worked
Live migration succeeded from BAA-BIG-Nest1 to BAA-SML-Nest1.
The VM stayed online, but guest network reachability failed after migration because the destination node does not present the same 172.16.60.0/24 guest network.

---

[2026-05-26 13:55 AEST]

Corrected sentence: **A full log of what we did today.**

## Full Log — Nested Hyper-V Cluster Live Migration and Guest Network Testing

### 1. Starting point

We resumed from the working nested Hyper-V cluster checkpoint.

Current lab components:

```text
Cluster nodes:
- BAA-BIG-Nest1
- BAA-SML-Nest1

Storage server:
- BAA-STOR1
- IP: 172.16.50.101
- Domain joined to bojieanzac.com

Shared storage:
- iSCSI virtual disk from BAA-STOR1
- Cluster Disk 1
- Mounted as Cluster Shared Volume:
  C:\ClusterStorage\Volume1

Test VM:
- BAA-TEST-VM01
- Ubuntu 26.04
- Also called Ubibi
```

This follows the earlier design direction: using nested Windows Server Hyper-V hosts, shared iSCSI storage, Cluster Shared Volume, and a small test VM for live migration demonstration. 

---

## 2. Ubibi / BAA-TEST-VM01 had no IPv4 address

The first issue today was that the Ubuntu test VM had no IPv4 address.

We confirmed:

```text
BAA-BIG-Nest1 had internet access.
BAA-TEST-VM01 / Ubibi did not have an IPv4 address.
```

That narrowed the problem. Since the nested Hyper-V host had internet, the likely causes were:

```text
- Wrong VM switch
- MAC spoofing issue
- DHCP not reaching the VM
- Ubuntu Netplan not requesting IPv4 DHCP
```

Inside Ubuntu, `dhclient` was not available, which is normal on newer Ubuntu versions.

We checked Netplan using:

```bash
sudo cat /etc/netplan/*.yaml
```

This displayed the Netplan configuration with root permission. It was required because normal `cat` received a permission denied error.

Finding:

```text
The Netplan file did not include dhcp4: true.
```

Fix:

```text
Created a new Netplan YAML file with dhcp4: true for the active interface.
```

Result:

```text
BAA-TEST-VM01 received an IPv4 address successfully.
```

Conclusion:

```text
Root cause:
Ubuntu 26.04 was not configured to request DHCPv4.

Fix:
Created a new Netplan YAML file enabling dhcp4: true.
```

---

## 3. Cluster Group was failed, then fixed

Before live migration, we checked the cluster groups.

Initial result:

```text
Available Storage    Offline
BAA-TEST-VM01        Online
Cluster Group        Failed
```

Important interpretation:

```text
BAA-TEST-VM01 was online.
The failed item was the cluster core group, not the VM role.
```

We checked the cluster core resources and brought the Cluster Group back online.

After fixing it, the cluster group showed:

```text
Cluster Group        Online
Cluster Name         Online
Cluster IP Address   Online
Cluster IP Address 172.16.50.120   Offline
```

We confirmed the dependency:

```text
Cluster Name depends on:
[Cluster IP Address] OR [Cluster IP Address 172.16.50.120]
```

This means only one of the cluster IP resources needs to be online at a time, which is acceptable for this multi-subnet cluster design.

Conclusion:

```text
Cluster core group became healthy again.
Offline secondary IP was acceptable because the dependency uses OR logic.
```

---

## 4. First live migration test succeeded

We confirmed:

```text
BAA-TEST-VM01 was online.
Cluster Group was online.
Both cluster nodes were available.
```

Then we ran live migration:

```powershell
Move-ClusterVirtualMachineRole -Name "BAA-TEST-VM01" -Node "BAA-SML-Nest1" -MigrationType Live
```

This command live-migrated the clustered VM role to `BAA-SML-Nest1`. It was required to test whether the nested Hyper-V cluster could move a running VM between nodes without shutting it down.

Result:

```text
BAA-TEST-VM01 moved to BAA-SML-Nest1.
State remained Online.
```

The Ubuntu VM console blinked briefly during migration. We treated that as normal because VM console sessions can briefly refresh during live migration cutover.

Then we moved it back:

```powershell
Move-ClusterVirtualMachineRole -Name "BAA-TEST-VM01" -Node "BAA-BIG-Nest1" -MigrationType Live
```

Result:

```text
BAA-TEST-VM01 moved back to BAA-BIG-Nest1.
State remained Online.
```

Confirmed result:

```text
Two-way live migration worked:
BAA-BIG-Nest1 → BAA-SML-Nest1
BAA-SML-Nest1 → BAA-BIG-Nest1
```

---

## 5. We discovered the guest network portability problem

The test VM had:

```text
IP address:       172.16.60.102
Default gateway:  172.16.60.1
```

This worked while the VM was on `BAA-BIG-Nest1`.

But after migrating to `BAA-SML-Nest1`, Ubuntu still kept:

```text
IP address:       172.16.60.102
Default gateway:  172.16.60.1
```

The SML node is in the `172.16.50.0/24` network. Therefore, after migration, the VM was still logically configured for the BIG network while running on the SML-side host.

We observed unreachable replies such as:

```text
Reply from 172.16.60.100: Destination host unreachable
```

Conclusion:

```text
Live migration succeeded.
Guest network continuity failed.
```

Reason:

```text
BAA-SML-Nest1 does not present the same 172.16.60.0/24 guest network to the VM.
```

Key design lesson:

```text
Live migration moves compute state.
It does not automatically change the guest OS IP address, gateway, or DNS.
```

---

## 6. Production design discussion

We discussed whether production VMs should change IP after migration.

Conclusion:

```text
For production VMs, changing IP after migration is usually not ideal.
```

Practical production designs include:

```text
1. Present the same guest VLAN/subnet on every cluster host.
2. Use stretched/overlay networking if same-IP mobility is required across sites.
3. Use separate site clusters and replication for disaster recovery.
4. Use DNS/load balancer/application-level HA for service continuity.
```

For a disaster scenario where the main site is destroyed, we clarified:

```text
That is not live migration.
That is disaster recovery / site failover.
```

Better model:

```text
Primary site runs production workloads.
DR site receives replicated data/workloads.
If primary site fails, the DR site starts services and clients are redirected by DNS, routing, load balancer, or application-level failover.
```

For seamless client service, we identified the better production pattern:

```text
Multiple service instances across sites
+ load balancer or traffic manager
+ replicated data layer
+ health checks
+ failover runbook
```

---

## 7. We tested manual IP refresh after migration

We decided to make the VM use the SML subnet after migration.

Expected SML-side network:

```text
IP network: 172.16.50.0/24
Gateway:    172.16.50.1
```

Inside Ubuntu, we tested DHCP refresh methods.

This did **not** work:

```bash
sudo networkctl renew eth0
```

This command tried to renew the DHCP lease only. It did not force Ubuntu to fully drop and rebuild the network configuration.

This **did** work:

```bash
sudo networkctl reconfigure eth0
```

This command reconfigured the `eth0` interface through `systemd-networkd`. It was stronger than `renew` and forced Ubuntu to request a new network configuration from the currently connected site network.

Confirmed behaviour:

```text
After moving to SML:
- VM still had 172.16.60.x
- networkctl reconfigure eth0 changed it to 172.16.50.x

After moving back to BIG:
- VM still had 172.16.50.x
- networkctl reconfigure eth0 changed it to 172.16.60.x
```

Conclusion:

```text
networkctl reconfigure eth0 is the working command for refreshing the VM IP after migration.
```

---

## 8. CSV / iSCSI health check after a temporary storage event

At one point, the cluster event log showed storage-related events around `Cluster Disk 1`.

The log included messages such as:

```text
Cluster Disk 1
ProcessingFailure
CannotComeOnlineOnThisNode
StorageFailure
exceeded its failover threshold
```

The same log also showed that the VM role moved and the cluster later recovered. The output showed `AutoFailbackType : 0` and no preferred owners, meaning the movement was not caused by a normal preferred-owner failback policy. 

We then verified the storage state.

Command:

```powershell
Get-ClusterSharedVolume | Format-List *
```

Result:

```text
Name: Cluster Disk 1
OwnerNode: BAA-SML-Nest1
SharedVolumeInfo: C:\ClusterStorage\Volume1
State: Online
```

Command:

```powershell
Get-ClusterSharedVolume | Select-Object -ExpandProperty SharedVolumeInfo
```

Result:

```text
FaultState: NoFaults
FriendlyVolumeName: C:\ClusterStorage\Volume1
MaintenanceMode: False
RedirectedAccess: False
```

Command:

```powershell
Invoke-Command -ComputerName BAA-BIG-Nest1,BAA-SML-Nest1 -ScriptBlock {
    Test-Path C:\ClusterStorage\Volume1
}
```

Result:

```text
True
True
```

Command:

```powershell
Invoke-Command -ComputerName BAA-BIG-Nest1,BAA-SML-Nest1 -ScriptBlock {
    Get-IscsiSession
}
```

Result:

```text
BAA-BIG-Nest1:
- IsConnected: True
- IsPersistent: True

BAA-SML-Nest1:
- IsConnected: True
- IsPersistent: True
```

Command:

```powershell
Invoke-Command -ComputerName BAA-BIG-Nest1,BAA-SML-Nest1 -ScriptBlock {
    Get-Disk | Select-Object Number, FriendlyName, OperationalStatus, HealthStatus, IsOffline, PartitionStyle
}
```

Result:

```text
Both nodes showed disks Online and Healthy.
```

Conclusion:

```text
CSV and iSCSI storage are currently healthy.
The earlier event was likely a temporary CSV ownership / failover event during movement, not a permanent storage failure.
```

---

## 9. We automated the guest network refresh

Since manual `networkctl reconfigure eth0` worked, we automated it with a systemd timer inside Ubuntu.

Confirmed interface name:

```text
eth0
```

Script created:

```bash
sudo nano /usr/local/sbin/reconfigure-network-after-migration.sh
```

Content:

```bash
#!/usr/bin/env bash
set -euo pipefail

IFACE="eth0"

networkctl reconfigure "$IFACE"
```

This script runs the working network reconfiguration command for `eth0`.

Made executable:

```bash
sudo chmod +x /usr/local/sbin/reconfigure-network-after-migration.sh
```

This gave the script execute permission so systemd can run it.

Systemd service created:

```bash
sudo nano /etc/systemd/system/reconfigure-network-after-migration.service
```

Content:

```ini
[Unit]
Description=Reconfigure network after Hyper-V migration

[Service]
Type=oneshot
ExecStart=/usr/local/sbin/reconfigure-network-after-migration.sh
```

Systemd timer created:

```bash
sudo nano /etc/systemd/system/reconfigure-network-after-migration.timer
```

Content:

```ini
[Unit]
Description=Run network reconfiguration every 30 seconds

[Timer]
OnBootSec=30
OnUnitActiveSec=30
AccuracySec=5

[Install]
WantedBy=timers.target
```

Enabled and started:

```bash
sudo systemctl daemon-reload
```

This reloaded systemd so it could see the new service and timer units.

```bash
sudo systemctl enable --now reconfigure-network-after-migration.timer
```

This enabled the timer at boot and started it immediately.

---

## 10. Final validation

We moved the VM again while the automation was enabled.

When the VM was on SML:

```text
eth0 automatically reconfigured.
The VM received a 172.16.50.x address.
The default gateway changed to 172.16.50.1.
```

Then we moved it back to BIG:

```powershell
Move-ClusterVirtualMachineRole -Name "BAA-TEST-VM01" -Node "BAA-BIG-Nest1" -MigrationType Live
```

After waiting for the timer:

```text
eth0 automatically reconfigured.
The VM received a 172.16.60.x address.
The default gateway changed to 172.16.60.1.
```

Final result:

```text
Working.
```

---

# Final Checkpoint

```text
Date: 2026-05-26

Nested Hyper-V cluster live migration test is successful.

BAA-TEST-VM01 / Ubibi can live migrate between:
- BAA-BIG-Nest1
- BAA-SML-Nest1

Shared iSCSI CSV storage is working:
- Cluster Disk 1
- C:\ClusterStorage\Volume1
- CSV online
- FaultState: NoFaults
- iSCSI connected on both nodes

Guest network finding:
- BIG node presents 172.16.60.0/24
- SML node presents 172.16.50.0/24
- Ubuntu keeps its previous DHCP lease after live migration
- Running networkctl reconfigure eth0 forces the VM to obtain the correct site IP

Automation added:
- systemd service:
  reconfigure-network-after-migration.service
- systemd timer:
  reconfigure-network-after-migration.timer
- runs every 30 seconds
- automatically executes:
  networkctl reconfigure eth0

Final behaviour:
- On BIG: VM gets 172.16.60.x with gateway 172.16.60.1
- On SML: VM gets 172.16.50.x with gateway 172.16.50.1
```

Clean report statement:

```text
The clustered Ubuntu VM successfully live migrated between the two nested Hyper-V cluster nodes. Because the two nodes present different guest networks, the VM initially retained its previous DHCP lease after migration. A guest-level systemd timer was implemented to run networkctl reconfigure eth0 every 30 seconds, allowing the VM to automatically obtain the correct site subnet after migration. This demonstrates live migration plus post-migration guest network adaptation, while also highlighting that production-grade seamless service continuity would normally require shared guest networks, DNS/load balancing, or application-level high availability.
```
