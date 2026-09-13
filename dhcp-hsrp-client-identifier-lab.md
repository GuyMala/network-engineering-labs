# Centralized DHCP with HSRP Redundancy — Client-Identifier Troubleshooting

**Date:** [fill in]
**Certification Track:** CCNP Enterprise
**Tools Used:** EVE-NG, Cisco IOS

---

## 1. Objective
Configure centralized DHCP service from a core router across two VLANs (workstations and servers), with HSRP providing redundant default gateways at the distribution layer, and use per-host DHCP reservations (via `client-identifier`) to give specific servers fixed IP addresses. Diagnose and fix a real case where three of four reservations silently failed to bind.

## 2. Topology

![Topology diagram](./Screenshot%202026-09-13%20at%2009.40.29.png)

- **Devices:** CORE (DHCP server + L3 core), DSW1, DSW2 (distribution, HSRP pair), ASW1, ASW2 (access, cross-connected to both distribution switches for redundancy), WKS1/WKS2 (VLAN 200 workstations), SRV1–SRV4 (VLAN 100 servers)
- **Role of each device:**
  - CORE hosts the DHCP pools for both VLANs and sits at the top of the topology.
  - DSW1 and DSW2 run HSRP for VLAN 100 (Group 100) and VLAN 200 (Group 200), giving each VLAN a redundant virtual default gateway.
  - ASW1 and ASW2 are access switches, each dual-homed to both DSW1 and DSW2 for redundancy; WKS/SRV hosts connect here.

## 3. IP Addressing Table

| Device | Interface/SVI | IP Address | Role |
|---|---|---|---|
| CORE | Vlan1 | 172.16.10.1/24 | Management/transit |
| CORE | Vlan100 | 10.10.10.10/24 | DHCP server address (SRV-Pool) |
| CORE | Vlan200 | 192.168.1.10/24 | DHCP server address (WKS-Pool) |
| DSW1 | Vlan100 | 10.10.10.2/24 | HSRP Group 100 member |
| DSW1 | Vlan200 | 192.168.1.2/24 | HSRP Group 200 member |
| DSW2 | Vlan100 | 10.10.10.3/24 | HSRP Group 100 member |
| DSW2 | Vlan200 | 192.168.1.3/24 | HSRP Group 200 member |
| — | HSRP Group 100 VIP | 10.10.10.1 | VLAN 100 default gateway |
| — | HSRP Group 200 VIP | 192.168.1.1 | VLAN 200 default gateway |
| SRV1 | eth0 | 10.10.10.6/24 (reserved) | MAC 0050.7966.680b |
| SRV2 | eth0 | 10.10.10.7/24 (reserved) | MAC 0050.7966.6807 |
| SRV3 | eth0 | 10.10.10.8/24 (reserved) | MAC 0050.7966.6809 |
| SRV4 | eth0 | 10.10.10.9/24 (reserved) | MAC 0050.7966.680a |
| WKS1/WKS2 | eth0 | Dynamic (WKS-Pool, 192.168.1.0/24) | |

*DSW1/DSW2 HSRP and SVI configuration, and ASW1/ASW2 port assignments, reflect the lab's design diagram; this session's CLI verification focused specifically on the DHCP reservation issue below rather than re-confirming every device's full config.*

## 4. Configuration Steps

**DHCP pools on CORE:**
- *What I configured:* Two general pools (WKS-Pool for VLAN 200, SRV-Pool for VLAN 100) plus four individual host-reservation pools (SRV1–SRV4), each binding a specific server to a fixed address via `client-identifier`.
- *Why:* General pools handle dynamic clients (workstations); host reservations give servers stable, predictable addresses without static-configuring each one by hand.
- *Key commands (as configured):*
```
ip dhcp excluded-address 10.10.10.1 10.10.10.10
ip dhcp excluded-address 192.168.1.1 192.168.1.10

ip dhcp pool WKS-Pool
 network 192.168.1.0 255.255.255.0
 default-router 192.168.1.1
 dns-server 172.16.10.1
 domain-name boson.com
 lease 1 2

ip dhcp pool SRV-Pool
 network 10.10.10.0 255.255.255.0
 default-router 10.10.10.1
 dns-server 172.16.10.1
 domain-name boson.com
 lease 2

ip dhcp pool SRV1
 host 10.10.10.6 255.255.255.0
 client-identifier 0100.5079.6668.0b
 default-router 10.10.10.1
 dns-server 172.16.10.1
 domain-name boson.com
```

**Understanding the client-identifier format:**
- *What it is:* Cisco's `client-identifier` isn't just the device's MAC address — it's a media-type byte (`01` for Ethernet) followed by the MAC. The full value is 7 bytes, not 6.
- *Why it matters:* When a DHCP client sends its identifier, IOS compares it byte-for-byte against the configured value. A 6-byte value (just the raw MAC) will never match the 7-byte value the client actually sends, no matter how correct the MAC portion is.

## 5. Verification

**Before the fix** — SRV2, SRV3, and SRV4 held unexpected, non-sequential addresses instead of their reserved ones:
```
SRV2  10.10.10.15/24   (expected .7)
SRV3  10.10.10.13/24   (expected .8)
SRV4  10.10.10.14/24   (expected .9)
```
SRV1 also held a stale dynamic lease (`10.10.10.11`) from before its reservation existed, despite having a correctly formatted `client-identifier`.

**After correcting the client-identifiers and renewing each lease:**
```
SRV1#sh ip → 10.10.10.6/24
SRV2#sh ip → 10.10.10.7/24
SRV3#sh ip → 10.10.10.8/24
SRV4#sh ip → 10.10.10.9/24
```
All four now hold exactly their intended reserved addresses.

## 6. Issues Encountered & Troubleshooting

| Issue | Root Cause | How I Diagnosed It | Fix |
|---|---|---|---|
| SRV2, SRV3, and SRV4's DHCP reservations weren't binding — each got an unrelated dynamic address instead of its reserved one | Their `client-identifier` values were configured as the raw 6-byte MAC address (e.g., `0050.7966.6807`), missing the leading `01` media-type byte that Cisco's format requires. SRV1's pool had the correct 7-byte value (`0100.5079.6668.0b`) and worked correctly. | Compared all four pools side by side in `show running-config \| section ip dhcp` — SRV1's identifier was visibly longer (14 hex digits) than SRV2/3/4's (12 hex digits). Confirmed by computing the correct identifier from each device's real MAC (`sh ip`) and comparing against the configured value. | Re-entered the `client-identifier` line on SRV2, SRV3, and SRV4 with the correct `01` + MAC value, then renewed each lease (`ip dhcp` on the VPCS client). Verified via `sh ip` that all three now hold their reserved addresses. |
| SRV1 held a stale address (`10.10.10.11`) that didn't match its own (correctly configured) reservation (`10.10.10.6`) | DHCP reservations only take effect on the *next* lease request — they don't retroactively reclaim an address a client is already holding from before the reservation existed. | Noticed the mismatch while reviewing `sh ip` output against the pool config. | Renewed SRV1's lease the same way as the others; it correctly picked up `10.10.10.6` on the next request. |

## 7. Lessons Learned
- A DHCP reservation match failure is silent by design — IOS doesn't log an error or refuse the request, it just quietly falls back to a normal dynamic lease from the pool. A device that "gets an IP but the wrong one" is the main symptom to watch for, not an obvious failure.
- Cisco's `client-identifier` format is media-type byte + MAC, not the MAC alone. Dropping the media-type byte produces a value that looks almost right but will never match.
- Non-sequential, seemingly random dynamic IPs on devices that were supposed to have reservations is a strong tell that the reservations aren't matching — worth checking the identifiers before assuming a broader DHCP or VLAN problem.
- Existing leases don't retroactively honor a newly added reservation; a renewal is required before the correct address takes effect.

## 8. Skills Demonstrated
Centralized multi-VLAN DHCP design, HSRP-based redundant gateway architecture, DHCP host reservation via client-identifier, diagnosing a silent DHCP binding failure from indirect symptoms (unexpected dynamic addresses) rather than an explicit error, and computing/validating Cisco client-identifier values from raw MAC addresses.
