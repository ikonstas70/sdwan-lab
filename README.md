# SD-WAN Lab Blueprint — Cisco CSR1000v v17.03.03

**Platform:** GNS3 / EVE-NG / CML  
**Image:** Cisco CSR1000v IOS XE 17.03.03  
**Author:** IT Solutions USA

A fully replicable SD-WAN lab covering topology, dual-transport IP addressing,
underlay OSPF, overlay configuration, vManage templates, and failover testing.

---

## Topology

```
                    +------------------+
                    |     vBond        |
                    |   10.10.0.100    |
                    +--------+---------+
                             |
                    +--------+---------+
                    |     vSmart       |
                    |   10.10.0.101    |
                    +--------+---------+
                             |
                    +--------+---------+
                    |     vManage      |
                    |   10.10.0.102    |
                    +--------+---------+
                             |
        +--------------------+--------------------+
        |                    |                    |
  +-----+------+      +------+-----+      +-------+----+
  | CSR-Hub    |      | CSR-Branch1|      | CSR-Branch2|
  | Site 100   |      | Site 101   |      | Site 102   |
  +------------+      +------------+      +------------+

  Transport 1 (ISP-A):  10.10.0.0/24
  Transport 2 (ISP-B):  10.20.0.0/24
```

> Each edge router has **two WAN interfaces** — one per transport — to exercise
> SD-WAN path selection and failover between ISP-A and ISP-B.

---

## IP Addressing

### Transport Layer (VPN 0)

| Device       | ISP-A Gig0/0    | ISP-B Gig0/2    |
|---|---|---|
| CSR-Hub      | 10.10.0.1/24    | 10.20.0.1/24    |
| CSR-Branch1  | 10.10.0.11/24   | 10.20.0.11/24   |
| CSR-Branch2  | 10.10.0.21/24   | 10.20.0.21/24   |

### LAN Layer (VPN 1)

| Device       | Interface    | IP Address      |
|---|---|---|
| CSR-Hub      | Gig0/1       | 192.168.1.1/24  |
| CSR-Branch1  | Gig0/1       | 192.168.2.1/24  |
| CSR-Branch2  | Gig0/1       | 192.168.3.1/24  |

### Controllers

| Device   | IP             | Role                             |
|---|---|---|
| vBond    | 10.10.0.100    | Orchestrator — device onboarding |
| vSmart   | 10.10.0.101    | Controller — policy distribution |
| vManage  | 10.10.0.102    | Manager — config and monitoring  |

---

## Underlay Configuration (OSPF — VPN 0 Transport Only)

> Underlay OSPF covers **transport interfaces only**. LAN subnets (VPN 1)
> are distributed across the SD-WAN overlay by vSmart — do not include
> them in underlay OSPF.

### CSR-Hub
```ios
router ospf 1
 router-id 1.1.1.1
 network 10.10.0.0 0.0.0.255 area 0
 network 10.20.0.0 0.0.0.255 area 0
```

### CSR-Branch1
```ios
router ospf 1
 router-id 2.2.2.2
 network 10.10.0.0 0.0.0.255 area 0
 network 10.20.0.0 0.0.0.255 area 0
```

### CSR-Branch2
```ios
router ospf 1
 router-id 3.3.3.3
 network 10.10.0.0 0.0.0.255 area 0
 network 10.20.0.0 0.0.0.255 area 0
```

---

## Overlay SD-WAN Configuration Steps

1. **Onboard devices in vManage** using device certificates (ZTP or manual)
2. **Configure TLOCs** — assign a color per transport:
   - Gig0/0 (ISP-A) → color `public-internet`
   - Gig0/2 (ISP-B) → color `biz-internet`
3. **Assign VPNs:**
   - VPN 0 — Transport (WAN interfaces)
   - VPN 1 — LAN/enterprise (branch subnets)
4. **Apply Control policies via vSmart** — route propagation, VPN membership
5. **Apply Data policies** — SLA-based path selection, QoS DSCP marking
6. **Verify:**

```ios
show sdwan control connections
show sdwan bfd sessions
show sdwan ipsec statistics
```

---

## vManage Device Template Fields

| Field            | CSR-Hub          | CSR-Branch1      | CSR-Branch2      |
|---|---|---|---|
| System IP        | 1.1.1.1          | 2.2.2.2          | 3.3.3.3          |
| Site ID          | 100              | 101              | 102              |
| WAN Interface 1  | Gig0/0           | Gig0/0           | Gig0/0           |
| TLOC Color 1     | public-internet  | public-internet  | public-internet  |
| WAN Interface 2  | Gig0/2           | Gig0/2           | Gig0/2           |
| TLOC Color 2     | biz-internet     | biz-internet     | biz-internet     |
| VPN IDs          | 0, 1             | 0, 1             | 0, 1             |
| QoS Policy       | DSCP EF for VoIP | same             | same             |

---

## Testing and Verification

### Ping across the overlay
```ios
ping 192.168.2.1 source 192.168.1.1
ping 192.168.3.1 source 192.168.1.1
```

### Verify TLOCs and BFD
```ios
show sdwan tloc
show sdwan bfd sessions
show sdwan control connections
```

### Simulate WAN failover
```ios
! Shut ISP-A on CSR-Branch1
interface GigabitEthernet0/0
 shutdown

! Verify SD-WAN switches to ISP-B automatically
show sdwan bfd sessions
show sdwan tunnel statistics

! Restore and confirm failback
interface GigabitEthernet0/0
 no shutdown
```

---

## Optional Enhancements

| Enhancement | Purpose |
|---|---|
| DMVPN fallback in underlay | Resilience if SD-WAN overlay fails |
| Zone-Based Firewall (ZBF) | Filter branch LAN traffic at the edge |
| NAT/PAT on VPN 0 | Internet exit for branch users |
| vAnalytics / SLA monitoring | Real-time path quality metrics in vManage |
| Second vSmart controller | High availability for control plane |

---

## Prerequisites

- Licensed CSR1000v v17.03.03 images (Cisco DevNet Sandbox or direct download)
- vManage, vSmart, vBond OVA/QCOW2 images (Cisco SD-WAN controller bundle)
- GNS3 2.x, EVE-NG Pro, or CML 2.x
- Minimum 32 GB RAM for full controller + 3-edge topology
