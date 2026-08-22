# Configure OSPF, VRRP, and EtherChannel (Multi-Area Distribution Design)

**Date:** [fill in]
**Certification Track:** CCNP Enterprise
**Tools Used:** EVE-NG, Cisco IOS (IOS 17.12 on routers, IOSv-L2 15.2 on switches)

**Status:** Complete — all configuration and verification confirmed.

---

## 1. Objective
Build a resilient two-site distribution design combining multi-area OSPF (Area 0 WAN backbone, Area 1 for VLAN 100, Area 2 for VLAN 200), VRRP with a deliberate active/active load split across two VLANs, and a 2-member LACP EtherChannel between the distribution switches.

## 2. Topology

- **Devices:** Router1, Router2 (WAN/Area 0 backbone), DSW1, DSW2 (distribution, ABRs), ASW1, ASW2 (access, pure Layer 2), PC1, PC2
- **Role of each device:**
  - Router1 and Router2 form the Area 0 backbone over a point-to-point serial link, each also connecting down to one distribution switch.
  - DSW1 and DSW2 are Area Border Routers — Area 0 on their uplink to the WAN routers, Area 1 for VLAN 100, Area 2 for VLAN 200. They're also the VRRP gateway pair and the two ends of the Po1 EtherChannel.
  - ASW1 and ASW2 are pure Layer 2 access switches with a single management SVI and a static default gateway pointed at the VRRP virtual IP.

## 3. IP Addressing Table

| Device | Interface/SVI | IP Address | Role | Notes |
|---|---|---|---|---|
| Router1 | Loopback0 | 1.1.1.1/32 | OSPF router-id source | Area 0 |
| Router1 | Serial1/0 | 198.51.100.1/30 | WAN link to Router2 | Area 0 |
| Router1 | Ethernet0/0 | 198.51.100.5/30 | Link to DSW1 | Area 0 |
| Router2 | Loopback0 | 2.2.2.2/32 | OSPF router-id source | Area 0 |
| Router2 | Serial1/0 | 198.51.100.2/30 | WAN link to Router1 | Area 0 |
| Router2 | Ethernet0/0 | 198.51.100.9/30 | Link to DSW2 | Area 0 |
| DSW1 | Loopback0 | 3.3.3.3/32 | OSPF router-id source | Area 0 |
| DSW1 | Gi0/0 (routed) | 198.51.100.6/30 | Link to Router1 | Area 0 |
| DSW1 | Vlan100 | 10.10.100.10/24 | VRRP Master (Grp 100, pri 120) | VIP 10.10.100.1, Area 1 |
| DSW1 | Vlan200 | 10.10.200.10/24 | VRRP Backup (Grp 200, pri 90) | VIP 10.10.200.1, Area 2 |
| DSW2 | Loopback0 | 4.4.4.4/32 | OSPF router-id source | Area 0 |
| DSW2 | Gi0/0 (routed) | 198.51.100.10/30 | Link to Router2 | Area 0 |
| DSW2 | Vlan100 | 10.10.100.20/24 | VRRP Backup (Grp 100, pri 90) | VIP 10.10.100.1, Area 1 |
| DSW2 | Vlan200 | 10.10.200.20/24 | VRRP Master (Grp 200, pri 120) | VIP 10.10.200.1, Area 2 |
| ASW1 | Vlan100 | 10.10.100.11/24 | Management only | GW: 10.10.100.1 |
| ASW2 | Vlan200 | 10.10.200.11/24 | Management only | GW: 10.10.200.1 |
| PC1 | eth0 | 10.10.100.111/24 | VLAN 100 host | GW: 10.10.100.1 (confirmed via `sh ip` on PC1) |
| PC2 | eth0 | 10.10.200.222/24 | VLAN 200 host | GW: 10.10.200.1 (confirmed via `sh ip` on PC2) |

## 4. Configuration Steps

**OSPF backbone (Router1, Router2):**
- *What I configured:* Loopback0 and both physical links placed in Area 0, with router-id explicitly pinned to each loopback.
- *Why:* Pinning router-id to a stable loopback avoids router-id churn if a physical interface flaps — standard practice for backbone routers.
- *Key commands:*
```
router ospf 1
 router-id 1.1.1.1
 network 1.1.1.1 0.0.0.0 area 0
 network 198.51.100.0 0.0.0.3 area 0
 network 198.51.100.4 0.0.0.3 area 0
```

**Multi-area OSPF on the distribution switches (DSW1, DSW2):**
- *What I configured:* Loopback and WAN-facing link in Area 0, VLAN 100 in Area 1, VLAN 200 in Area 2 — making both switches ABRs.
- *Why:* This is the core of the multi-area design — each distribution switch bridges the Area 0 backbone to its local access-layer areas, keeping VLAN-specific LSAs from flooding the whole backbone.
- *Key commands:*
```
! DSW1 (process ID differs from the routers' — fine, since it's only locally significant)
router ospf 2
 router-id 3.3.3.3
 network 3.3.3.3 0.0.0.0 area 0
 network 10.10.100.0 0.0.0.255 area 1
 network 10.10.200.0 0.0.0.255 area 2
 network 198.51.100.4 0.0.0.3 area 0
```

**VRRP — active/active load split across VLANs:**
- *What I configured:* DSW1 priority 120 on VLAN 100 / 90 on VLAN 200; DSW2 priority 90 on VLAN 100 / 120 on VLAN 200 — an intentional split so each switch is Master for one VLAN and Backup for the other, rather than one switch idling entirely.
- *Why:* This uses both distribution switches' forwarding capacity under normal conditions instead of leaving one purely as a cold standby.
- *Key commands:*
```
! DSW1
interface Vlan100
 vrrp 100 ip 10.10.100.1
 vrrp 100 priority 120
 vrrp 100 authentication text 80$0nL48

interface Vlan200
 vrrp 200 ip 10.10.200.1
 vrrp 200 priority 90
```
- *Worth flagging honestly:* VRRP Group 100 uses **plaintext** authentication (trivially sniffable, not a real security control), and Group 200 has **no authentication at all**. Also, only DSW2 has VRRP object tracking (`track 1 interface GigabitEthernet0/0 line-protocol` / `vrrp 200 track 1 decrement 50`) — if DSW2 loses its uplink to Router2, its priority drops below DSW1's and DSW1 correctly takes over. DSW1 has **no equivalent tracking** for VLAN 100 — if DSW1 loses its uplink to Router1, it would keep forwarding for VLAN 100 with no path out, since nothing would lower its priority. That's a real asymmetry in the design, not just a cosmetic gap.

**LACP EtherChannel (Po1) between DSW1 and DSW2:**
- *What I configured:* Gi1/0 and Gi1/1 bundled into Port-channel1 using active LACP on both switches.
- *Why:* Active LACP negotiates the bundle dynamically and will refuse to bundle a misconfigured link, which is safer than static (`on` mode) EtherChannel.
- *Key commands:*
```
interface Port-channel1
 switchport trunk encapsulation dot1q
 switchport mode trunk

interface GigabitEthernet1/0
 channel-protocol lacp
 channel-group 1 mode active

interface GigabitEthernet1/1
 channel-protocol lacp
 channel-group 1 mode active
```

**Access layer (ASW1, ASW2) — kept pure Layer 2:**
- *What I configured:* Single management SVI per switch, static default gateway pointed at the relevant VRRP virtual IP.
- *Why:* Consistent with the access/distribution split used in the other labs — no routing needed at the access layer.

## 5. Verification

**OSPF adjacencies (from DSW1) — full backbone convergence confirmed:**
```
DSW1#show ip ospf neighbor
Neighbor ID     Pri   State           Dead Time   Address         Interface
1.1.1.1           1   FULL/DR         00:00:38    198.51.100.5    GigabitEthernet0/0
4.4.4.4           1   FULL/DR         00:00:37    10.10.100.20    Vlan100
4.4.4.4           1   FULL/DR         00:00:35    10.10.200.20    Vlan200
```
Three full adjacencies: DSW1 to Router1 (Area 0), and DSW1 to DSW2 over both Vlan100 (Area 1) and Vlan200 (Area 2).

**VRRP steady state (from DSW1) — matches the designed priorities:**
```
DSW1#show vrrp brief
Interface          Grp Pri Time  Own Pre State   Master addr     Group addr
Vl100              100 120 3531       Y  Master  10.10.100.10    10.10.100.1
Vl200              200 90  3648       Y  Backup  10.10.200.20    10.10.200.1
```
DSW1 is Master for VLAN 100 as designed; for VLAN 200, DSW1 correctly shows Backup with the master address at 10.10.200.20 (DSW2) — confirming the load-split is working exactly as configured.

**EtherChannel bundle status — confirmed independently from both ends, not just configured:**
```
DSW1#show etherchannel summary
Group  Port-channel  Protocol    Ports
------+-------------+-----------+-----------------------------------------------
1      Po1(SU)         LACP      Gi1/0(P)    Gi1/1(P)

DSW2#show etherchannel summary
Group  Port-channel  Protocol    Ports
------+-------------+-----------+-----------------------------------------------
1      Po1(SU)         LACP      Gi1/0(P)    Gi1/1(P)
```
`Po1(SU)` confirms Layer 2, in use, on both switches. All four member ports (two per side) show the `P` flag (bundled in port-channel) rather than `I` (standalone) or `s` (suspended) — meaning LACP successfully negotiated the bundle on both ends, not just that the config was applied on one side and assumed to work.

## 6. Issues Encountered & Troubleshooting

| Issue | Root Cause | How I Diagnosed It | Fix |
|---|---|---|---|
| DSW1 briefly became Master for VLAN 200 for ~7 seconds despite lower priority (90) | A syslog capture showed `%VRRP-6-STATECHANGE: Vl200 Grp 200 state Backup -> Master` at 06:07:37, then `Master -> Backup` at 06:07:44 — consistent with DSW1 missing one VRRP advertisement from DSW2 (Master), likely due to convergence activity happening around the same time (OSPF/VRRP coming up together), rather than any real fault. | Spotted the timestamped syslog lines while reviewing `show vrrp brief` output. Confirmed the state was correct and stable afterward — DSW1 back to Backup, DSW2 still Master. | No action needed — self-corrected within 7 seconds and steady-state is confirmed correct. Documented because a transient flap like this is exactly the kind of thing worth being able to explain if asked, rather than something to hide. |

**Both items originally outstanding were resolved after the lab environment came back up:**
- Po1's bundle status was confirmed via `show etherchannel summary` on DSW2 (see Section 5) — genuinely bundled via LACP, not just configured.
- PC2's default gateway was confirmed via `sh ip` on PC2 as **10.10.200.1**, correctly matching the VRRP virtual IP. The diagram's "10.10.200.2" label was a documentation typo, not a real device misconfiguration.

## 7. Lessons Learned
- A VRRP active/active split (each switch Master for a different VLAN) gets more value out of both distribution switches than a plain active/standby pair, but it only works safely if *both* switches have equivalent object tracking — otherwise one direction of failover works and the other doesn't.
- Different OSPF process IDs across devices (1 on the routers, 2 on DSW1, 3 on DSW2) is completely fine — process ID is locally significant only, and doesn't need to match for adjacencies to form.
- A brief VRRP state flap isn't automatically a problem — checking the actual timestamps and confirming the steady state afterward is what turns a scary-looking syslog line into a non-issue.
- A diagram and a running-config can disagree — the PC2 gateway discrepancy turned out to be a typo in the diagram, not a real misconfiguration, but it was worth confirming directly on the device (`sh ip` on PC2) rather than assuming either the diagram or the "expected" answer was correct.

## 8. Skills Demonstrated
Multi-area OSPF design (backbone + two access areas), VRRP active/active load-split configuration with object tracking, LACP EtherChannel configuration, distinguishing access-layer vs. distribution-layer roles, reading and correctly interpreting VRRP/OSPF syslog and verification output, identifying a real design asymmetry (unequal object tracking) rather than just confirming things "worked."
