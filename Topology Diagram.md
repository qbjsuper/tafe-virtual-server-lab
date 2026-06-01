## Current Logical Topology

```mermaid
flowchart TB

    UPSTREAM["Upstream / Physical Network"]

    subgraph BIG["Big Site - BAA-BIG<br/>172.16.60.0/24"]
        BIGPFS["BAA-BIG-PFS1<br/>pfSense Gateway / Firewall<br/>LAN: 172.16.60.1"]
        BIGDC["BAA-BIG-DC1<br/>AD DS / DNS / DHCP<br/>IP: 172.16.60.10"]
        BIGNEST["BAA-BIG-Nest1<br/>Nested Hyper-V Cluster Node<br/>Windows Server Datacenter"]
        BIGLAN["Big Site LAN<br/>172.16.60.0/24"]
    end

    subgraph SML["Small Site - BAA-SML<br/>172.16.50.0/24"]
        SMLPFS["BAA-SML-PFS1<br/>pfSense Gateway / Firewall<br/>LAN: 172.16.50.1"]
        SMLDC["BAA-SML-DC1<br/>AD DS / DNS<br/>IP: 172.16.50.10"]
        STOR["BAA-STOR1<br/>iSCSI Storage Server<br/>IP: 172.16.50.101"]
        SMLNEST["BAA-SML-Nest1<br/>Nested Hyper-V Cluster Node<br/>Windows Server Datacenter"]
        SMLLAN["Small Site LAN<br/>172.16.50.0/24"]
    end

    UPSTREAM --> BIGPFS
    UPSTREAM --> SMLPFS

    BIGPFS --> BIGLAN
    BIGLAN --> BIGDC
    BIGLAN --> BIGNEST

    SMLPFS --> SMLLAN
    SMLLAN --> SMLDC
    SMLLAN --> STOR
    SMLLAN --> SMLNEST

    BIGPFS <-. "IPsec Site-to-Site VPN" .-> SMLPFS
    BIGDC <-. "AD / DNS / Replication" .-> SMLDC
    BIGNEST <-. "Cluster heartbeat / management" .-> SMLNEST
    BIGNEST <-. "iSCSI access to shared storage" .-> STOR
    SMLNEST <-. "iSCSI access to shared storage" .-> STOR
```

## Nested Hyper-V Cluster Extension

```mermaid
flowchart TB

    subgraph PHYBIG["Physical Host Layer - BIG"]
        BIGPHY["BIG Physical Host<br/>Windows 11 + Hyper-V"]
    end

    subgraph PHYSML["Physical Host Layer - SML"]
        SMLPHY["SML / Mini Physical Host<br/>Windows 11 + Hyper-V"]
    end

    subgraph NESTBIG["First-Level VM - BIG"]
        BIGNEST["BAA-BIG-Nest1<br/>Windows Server Datacenter<br/>Hyper-V Role<br/>Failover Cluster Node"]
        BIGVSW["BAA-NEST-EXT-SW<br/>Nested External vSwitch"]
    end

    subgraph NESTSML["First-Level VM - SML"]
        SMLNEST["BAA-SML-Nest1<br/>Windows Server Datacenter<br/>Hyper-V Role<br/>Failover Cluster Node"]
        SMLVSW["BAA-NEST-EXT-SW<br/>Nested External vSwitch"]
    end

    subgraph CLUSTER["Nested Hyper-V Failover Cluster - BAA-NEST-CL1"]
        CSV["Cluster Shared Volume<br/>C:\\ClusterStorage\\Volume1"]
        UBIBI["BAA-TEST-VM01 / Ubibi<br/>Ubuntu Server 26.04<br/>Clustered VM Role"]
    end

    STOR["BAA-STOR1<br/>iSCSI Target Server<br/>172.16.50.101"]

    BIGPHY --> BIGNEST
    SMLPHY --> SMLNEST

    BIGNEST --> BIGVSW
    SMLNEST --> SMLVSW

    BIGNEST <-. "Cluster node link" .-> SMLNEST

    STOR -->|"iSCSI LUN"| CSV
    CSV --> UBIBI

    BIGVSW -->|"Current VM network path"| UBIBI
    SMLVSW -. "Network path after live migration" .-> UBIBI

    BIGNEST -. "Owns VM before migration" .-> UBIBI
    SMLNEST -. "Owns VM after migration" .-> UBIBI
```

## Nested VM Current Troubleshooting State

```text
Current nested test VM:
BAA-TEST-VM01 / Ubibi

Guest OS:
Ubuntu Server 26.04

Current issue:
Ubibi has no usable IPv4 address yet.

Confirmed:
- `ip -4 addr` only shows loopback: 127.0.0.1/8
- SSH service is not installed yet
- Gateway 172.16.60.1 is not reachable from Ubibi

Next checks:
1. Run `ip link` inside Ubibi.
2. Verify BAA-TEST-VM01 has a VM network adapter.
3. Verify the adapter is connected to BAA-NEST-EXT-SW.
4. Verify BAA-NEST-EXT-SW is External on both nested Hyper-V nodes.
5. Verify MAC spoofing is enabled on BAA-BIG-Nest1 and BAA-SML-Nest1 from the physical Hyper-V hosts.
```

## New TOPO
Given how many layers you've now confirmed, a single diagram would be an unreadable tangle — so here's the comprehensive picture as four clean views: the physical metal, the logical lab, the cluster/storage layer, and the remote-access overlay that's the whole point of the project.

First, the physical layer — the actual hardware and how it sits on your home network:That's the metal. Inside those two minis lives the actual `bojieanzac.com` environment — two virtual sites, each behind its own pfSense gateway:The trickiest part to picture is the clustering, because both clusters span the two sites and share storage across the IPsec link. Pulling them out on their own:And finally the reason for all of this — how `qlaptop` at TAFE reaches the lab, with the design decisions we landed on baked in:That's the whole system, top to bottom. A couple of things worth saying about it as a set rather than re-describing the pictures.

The four views stack as layers of the same lab: the metal carries the virtual sites, the virtual sites host the two clusters, and Tailscale overlays the lot to give you access from TAFE. The amber boxes in the last view are the only pieces not yet built — everything below them already works.

The two genuine action items are exactly what the last view flags: Tailscale is logged out on both minis right now (the APIPA tunnels), and the pfSense subnet routers aren't configured yet. Those two, plus approving the routes, are all that stands between you and reaching the lab from Coomera.

These diagrams are also a clean addition to the `tafe-virtual-server-lab` repo — a `docs/topology.md` with the physical, logical, cluster, and remote-access views in one place would replace the older single-view topology file and match the reality we just nailed down. I can hand you that as Markdown (Mermaid versions of these) whenever you want.

Where do you want to go next — write the BIG-side pfSense subnet-router steps now that both sites are healthy, get Tailscale logged back in on the minis first, or set the two host lab legs static so that's off the list? Any of the three is a sensible next move.