# Explore Layer 3 Switching

**Date:** [fill in]
**Certification Track:** CCNP Enterprise
**Tools Used:** EVE-NG, Cisco IOS (IOSv-L2, version 15.2)

---

## 1. Objective
Explore SVI-based Layer 3 switching by enabling routing on the distribution-layer switches (DSW1, DSW2) while keeping the access-layer switches (ASW1, ASW2) purely Layer 2 — and route between two host VLANs using EIGRP over a shared transit SVI.

## 2. Topology

*Same physical topology as the MST lab (reused base topology, different feature focus).*

- **Devices:** ASW1, ASW2 (access, Layer 2 only), DSW1, DSW2 (distribution, Layer 3 switching), PC1, PC2
- **Role of each device:**
  - ASW1 and ASW2 have `no ip routing` explicitly set — they are pure Layer 2 switches with a single management SVI and a static default gateway pointing up to their local distribution switch.
  - DSW1 and DSW2 are Layer 3 switches — routing is enabled by platform default (no explicit `ip routing` line needed on this image, but `ip cef` confirms it's active), with per-VLAN SVIs and EIGRP handling inter-VLAN routing between them.

## 3. IP Addressing Table

| Device | Interface/SVI | IP Address | Role | Notes |
|---|---|---|---|---|
| ASW1 | Vlan1 | 172.16.1.10/24 | Management only | `ip default-gateway 172.16.1.100` |
| ASW2 | Vlan1 | 172.16.1.20/24 | Management only | `ip default-gateway 172.16.1.200` |
| DSW1 | Vlan1 | 172.16.1.100/24 | Transit SVI (EIGRP peers here) | Also ASW1's default gateway |
| DSW1 | Vlan11 | 172.16.11.100/24 | PC1's gateway | |
| DSW2 | Vlan1 | 172.16.1.200/24 | Transit SVI (EIGRP peers here) | Also ASW2's default gateway |
| DSW2 | Vlan12 | 172.16.12.200/24 | PC2's gateway | |
| PC1 | eth0 | — | VLAN 11 host | IP not captured this session |
| PC2 | eth0 | — | VLAN 12 host | IP not captured this session |

## 4. Configuration Steps

**Access switches (ASW1, ASW2) — kept pure Layer 2:**
- *What I configured:* Explicit `no ip routing`, a single management SVI on Vlan1, and a static `ip default-gateway` pointing to the local distribution switch.
- *Why:* Access switches don't need routing capability — all inter-VLAN work happens at the distribution layer, so keeping them Layer 2-only is both simpler and closer to real-world design.
- *Key commands:*
```
no ip routing

interface Vlan1
 ip address 172.16.1.10 255.255.255.0

ip default-gateway 172.16.1.100
```

**Distribution switches (DSW1, DSW2) — SVIs for routing:**
- *What I configured:* A shared transit SVI (Vlan1, same subnet on both switches) plus a local host-facing SVI per switch (Vlan11 on DSW1, Vlan12 on DSW2).
- *Why:* SVI-based routing means no dedicated routed physical interfaces are needed — VLAN interfaces themselves become the Layer 3 gateways once routing is enabled on the switch.
- *Key commands:*
```
! DSW1
interface Vlan1
 ip address 172.16.1.100 255.255.255.0
interface Vlan11
 ip address 172.16.11.100 255.255.255.0

! DSW2
interface Vlan1
 ip address 172.16.1.200 255.255.255.0
interface Vlan12
 ip address 172.16.12.200 255.255.255.0
```

**EIGRP between DSW1 and DSW2:**
- *What I configured:* EIGRP AS 100 on both distribution switches with a single `network 172.16.0.0` statement.
- *Why:* This lets DSW1 and DSW2 dynamically exchange routes to each other's host VLANs (172.16.11.0/24 and 172.16.12.0/24) over the shared Vlan1 transit subnet, rather than relying on static routes.
- *Key commands:*
```
router eigrp 100
 network 172.16.0.0
```

## 5. Verification

**EIGRP adjacency (confirmed up on both sides):**
```
DSW2#show ip eigrp neighbors
EIGRP-IPv4 Neighbors for AS(100)
H   Address                 Interface              Hold Uptime   SRTT   RTO  Q  Seq
0   172.16.1.100            Vl1                      12 00:02:54 1030  5000  0  3

DSW1#show ip eigrp neighbors
EIGRP-IPv4 Neighbors for AS(100)
H   Address                 Interface              Hold Uptime   SRTT   RTO  Q  Seq
0   172.16.1.200            Vl1                      13 00:03:17 1522  5000  0  6
```

**Cross-learned routes (confirming inter-VLAN routing actually works):**
```
DSW2#show ip route eigrp
D        172.16.11.0/24 [90/3072] via 172.16.1.100, 00:04:49, Vlan1

DSW1#show ip route eigrp
D        172.16.12.0/24 [90/3072] via 172.16.1.200, 00:04:56, Vlan1
```
This confirms full bidirectional routing: DSW2 learned PC1's subnet from DSW1, and DSW1 learned PC2's subnet from DSW2 — both over the shared Vlan1 SVI, with no static routes involved.

## 6. Issues Encountered & Troubleshooting

| Issue | Root Cause | How I Diagnosed It | Fix |
|---|---|---|---|
| EIGRP neighbor briefly went down (`%DUAL-5-NBRCHANGE: ... is down: holding time expired` on DSW2) | Likely a round-trip latency spike, not a real link failure — `show ip eigrp neighbors` afterward showed unusually high SRTT (1030–1522ms) and RTO capped at 5000ms between two directly connected switches, consistent with EVE-NG virtualization delay rather than an actual network problem. | Noticed the syslog line in a `show running-config` paste. Re-checked `show ip eigrp neighbors` on both DSW1 and DSW2 afterward — adjacency was back up on both sides (Uptime 00:02:54 and 00:03:17 respectively), and `show ip route eigrp` confirmed routes were present and correct. | **No corrective action needed** — the adjacency self-recovered and routes remained accurate. Flagged here because in a real production network, RTT this high between adjacent switches would be worth investigating even if the timer didn't expire. |

## 7. Lessons Learned
- SVI-based Layer 3 switching means routing lives entirely in software VLAN interfaces — no dedicated routed physical ports are required, which keeps the physical topology identical to a pure Layer 2 design while still enabling routing.
- Access-layer switches genuinely don't need `ip routing` enabled at all — a single management SVI and a static default gateway is sufficient, which keeps their configuration simpler and their attack surface smaller.
- An EIGRP "holding time expired" message doesn't automatically mean a real outage — checking SRTT/RTO and neighbor uptime after the fact is what separates "transient lab latency" from "an actual problem," and that distinction matters before spending time troubleshooting a link that isn't actually broken.

## 8. Skills Demonstrated
SVI-based Layer 3 switching configuration, distinguishing access-layer vs. distribution-layer routing roles, EIGRP configuration and neighbor verification, interpreting EIGRP syslog/timer behavior to separate real faults from lab environment artifacts.
