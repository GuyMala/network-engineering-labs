# Implement MST (Multiple Spanning Tree)

**Date:** [fill in]
**Certification Track:** CCNP Enterprise
**Tools Used:** EVE-NG, Cisco IOS (IOSv-L2, version 15.2)

---

## 1. Objective
Implement MST across a four-switch access/distribution topology so VLANs converge per-instance rather than all being treated identically (as RSTP or PVST+ would), and control root bridge placement per instance using priority tuning.

## 2. Topology

- **Devices:** ASW1, ASW2 (access), DSW1, DSW2 (distribution), PC1, PC2
- **Role of each device:**
  - ASW1 and ASW2 are access switches, each dual-homed to both distribution switches for redundancy.
  - DSW1 and DSW2 are distribution switches, directly interconnected, and split default-gateway duties (DSW1 owns VLAN 11's SVI, DSW2 owns VLAN 12's).
  - PC1 connects to ASW1 on VLAN 11; PC2 connects to ASW2 on VLAN 12. Both edge ports use PortFast + BPDU Guard.

## 3. IP Addressing Table

| Device | Interface/SVI | IP Address | Notes |
|---|---|---|---|
| ASW1 | Vlan1 | 172.16.1.10/24 | Management |
| ASW1 | Vlan11 | 172.16.11.10/24 | |
| ASW1 | Gi1/0 | — | Access port, VLAN 11, PortFast + BPDU Guard (PC1) |
| ASW2 | Vlan1 | 172.16.1.20/24 | Management |
| ASW2 | Vlan12 | 172.16.12.20/24 | |
| ASW2 | Gi1/0 | — | Access port, VLAN 12, PortFast + BPDU Guard (PC2) |
| DSW1 | Vlan1 | 172.16.1.100/24 | Management |
| DSW1 | Vlan11 | 172.16.11.100/24 | Gateway for VLAN 11 |
| DSW2 | Vlan1 | 172.16.1.200/24 | Management |
| DSW2 | Vlan12 | 172.16.12.200/24 | Gateway for VLAN 12 |

*Note: no HSRP/GLBP is configured — each distribution switch is the sole gateway for its own VLAN rather than sharing gateway redundancy for both. Worth keeping in mind as a design limitation, not an oversight to gloss over.*

## 4. Configuration Steps

**Enabling MST and defining the region (all four switches):**
- *What I configured:* MST mode, extended system ID, and a region definition (name, revision, VLAN-to-instance mapping).
- *Why:* All switches that should treat each other as native MST peers must agree on the exact same region name, revision number, *and* VLAN-to-instance mapping — this triple is hashed into a region digest, and any difference puts a switch in a different region entirely.
- *Key commands:*
```
spanning-tree mode mst
spanning-tree extend system-id

spanning-tree mst configuration
 name boson
 revision 1
 instance 1 vlan 1, 4, 6, 10-12
```

**Root bridge tuning for instance 1:**
- *What I configured:* DSW1 at priority 24576, DSW2 at priority 28672 (ASW1/ASW2 left at IOS default).
- *Why:* Lower priority wins root election — this deliberately makes DSW1 the root for instance 1, with DSW2 positioned as the natural backup, rather than leaving root placement to chance.
- *Key commands:*
```
spanning-tree mst 1 priority 24576
```

**Edge port hardening (ASW1 Gi1/0, ASW2 Gi1/0):**
- *What I configured:* PortFast + BPDU Guard on the PC-facing access ports.
- *Why:* These ports will never legitimately receive BPDUs from another switch, so PortFast skips the listening/learning delay for faster PC connectivity, and BPDU Guard shuts the port down if a BPDU ever does arrive — protecting against an accidental switch-to-switch loop on what should be an end-host port.
- *Key commands:*
```
interface GigabitEthernet1/0
 switchport access vlan 11
 switchport mode access
 spanning-tree portfast edge
 spanning-tree bpduguard enable
```

## 5. Verification
```
DSW1#show spanning-tree mst configuration
Name      [boson]
Revision  1     Instances configured 2
Instance  Vlans mapped
--------  ---------------------------------------------------------------------
0         2-3,5,7-9,13-4094
1         1,4,6,10-12

DSW1#show spanning-tree mst 1
##### MST1    vlans mapped:   1,4,6,10-12
Bridge        address 5000.0004.0000  priority      24577 (24576 sysid 1)
Root          this switch for MST1
Interface        Role Sts Cost      Prio.Nbr Type
---------------- ---- --- --------- -------- --------------------------------
Gi0/0            Desg FWD 20000     128.1    P2p
Gi0/1            Desg FWD 20000     128.2    P2p
Gi0/2            Mstr FWD 20000     128.3    P2p Bound(RSTP)
Gi0/3            Altn BLK 20000     128.4    P2p Bound(RSTP)
Gi1/0            Desg FWD 20000     128.5    P2p
Gi1/1            Desg FWD 20000     128.6    P2p
```
This confirms DSW1 is the elected root for instance 1 (as designed via priority), and shows normal MST loop prevention in action: one link to the neighboring switch forwards (Mstr) while the redundant link blocks (Altn/BLK).

It also surfaces the region mismatch directly: Gi0/2 and Gi0/3 are marked **`Bound(RSTP)`** rather than a normal internal MST role — this is the CLI's way of showing "the neighbor on this link is not in my MST region." Based on the config diff below, this can only be the link(s) to ASW1.

Confirmed directly from ASW1's own perspective:
```
ASW1#show spanning-tree mst configuration
Name      [boson]
Revision  1     Instances configured 2
Instance  Vlans mapped
--------  ---------------------------------------------------------------------
0         2-3,5,7-9,11,13-4094
1         1,4,6,10,12

ASW1#show spanning-tree mst 1
##### MST1    vlans mapped:   1,4,6,10,12
Bridge        address 5000.0003.0000  priority      32769 (32768 sysid 1)
Root          this switch for MST1
Interface        Role Sts Cost      Prio.Nbr Type
---------------- ---- --- --------- -------- --------------------------------
Gi0/0            Desg FWD 20000     128.1    P2p
Gi0/1            Desg FWD 20000     128.2    P2p
Gi0/2            Desg FWD 20000     128.3    P2p
Gi0/3            Desg FWD 20000     128.4    P2p
```
This is the clearest possible confirmation: ASW1 considers **itself** the root for instance 1, and every port is Designated/Forwarding — there's no Alternate or Root port anywhere in its table. That's what a region of one looks like: with no neighbor sharing its digest, ASW1 has nothing to compare priority against, so it can only ever see itself as root within its own isolated region.

**After the fix** (see Section 6), re-checking DSW1 confirms the boundary is gone:
```
DSW1#show spanning-tree mst 1
##### MST1    vlans mapped:   1,4,6,10-12
Bridge        address 5000.0004.0000  priority      24577 (24576 sysid 1)
Root          this switch for MST1
Interface        Role Sts Cost      Prio.Nbr Type
---------------- ---- --- --------- -------- --------------------------------
Gi0/0            Desg FWD 20000     128.1    P2p
Gi0/1            Desg FWD 20000     128.2    P2p
Gi0/2            Desg FWD 20000     128.3    P2p
Gi0/3            Desg FWD 20000     128.4    P2p
Gi1/0            Desg FWD 20000     128.5    P2p
Gi1/1            Desg FWD 20000     128.6    P2p
```
Gi0/2 and Gi0/3 no longer carry the `Bound(RSTP)` tag — ASW1 is now a native MST peer instead of an RSTP-compatibility neighbor. (All six ports showing Designated/Forwarding is expected here and unrelated to the fix — a root bridge's ports are always Designated, since the root never has a Root port.)

## 6. Issues Encountered & Troubleshooting

| Issue | Root Cause | How I Diagnosed It | Fix |
|---|---|---|---|
| ASW1 sat in its own isolated MST region instead of peering with ASW2/DSW1/DSW2 | ASW1's `spanning-tree mst configuration` mapped instance 1 as `vlan 1, 4, 6, 10, 12` — excluding VLAN 11 — while ASW2, DSW1, and DSW2 all mapped it as `vlan 1, 4, 6, 10-12`, which includes VLAN 11. Same region name and revision, but a different VLAN-to-instance mapping produces a different region digest. | Compared `show running-config` across all four switches side by side; the mismatch only showed up by diffing the exact instance-mapping line, not from any error message. Confirmed two ways on the live network: DSW1's boundary ports toward ASW1 showed `Bound(RSTP)` instead of a normal MST role, and ASW1's own `show spanning-tree mst 1` showed it as root of its own region with every port Designated/Forwarding. | **Corrected.** Updated ASW1 with `instance 1 vlan 1, 4, 6, 10-12` to match the other three switches. Re-verified on DSW1: Gi0/2 and Gi0/3 no longer carry the `Bound(RSTP)` tag, confirming ASW1 rejoined the shared MST region as a native peer. |

## 7. Lessons Learned
- Region name and revision matching is necessary but not sufficient — the VLAN-to-instance mapping is part of the same digest and must match exactly, down to a single VLAN.
- A region mismatch doesn't take a switch offline; MST just falls back to RSTP compatibility on that boundary, which still prevents loops but loses per-instance load-balancing across that link. That makes it a quiet problem — worth actively checking for, not something that announces itself.
- `Bound(RSTP)` in `show spanning-tree mst <instance>` output is the fastest real-world signal of a region mismatch — much faster than manually comparing region digests.

## 8. Skills Demonstrated
MST region design and configuration, per-instance root bridge priority tuning, PortFast/BPDU Guard edge-port hardening, diagnosing MST region mismatches from live CLI output, cross-device configuration auditing.
