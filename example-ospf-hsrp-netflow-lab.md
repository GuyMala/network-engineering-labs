# Enterprise Redundancy & Monitoring Lab: OSPF, FHRP, IP SLA, NetFlow, Syslog

**Date:** [fill in]
**Certification Track:** CCNP Enterprise
**Tools Used:** EVE-NG, Cisco IOS/IOL images, Ubuntu 20.04 (rsyslog server)

---

## 1. Objective
Build a resilient enterprise network demonstrating OSPF stub/NSSA area design, first-hop redundancy (HSRP/VRRP/GLBP), proactive failure detection with IP SLA object tracking, traffic visibility via Flexible NetFlow, and centralized logging with rsyslog.

## 2. Topology
*Insert topology screenshot exported from EVE-NG here.*

- **Devices:** Core routers (OSPF backbone), Area routers (stub/NSSA), Distribution switches (HSRP/VRRP/GLBP), Ubuntu-Syslog-Server
- **Role of each device:** Core routers form Area 0; edge routers sit in stub/NSSA areas to reduce LSA flooding; distribution switches provide gateway redundancy for access-layer VLANs.

## 3. IP Addressing Table

| Device | Interface | IP Address | VLAN / Subnet | Notes |
|---|---|---|---|---|
| *(fill in from your lab)* | | | | |

## 4. Configuration Steps

**OSPF Stub/NSSA Design:**
- *What I configured:* Designated one area as totally stubby and another as NSSA to control external route injection.
- *Why:* Reduces LSA flooding and routing table size on edge routers that don't need full external route visibility.
- *Key commands:*
```
router ospf 1
 area 1 stub no-summary
 area 2 nssa
```

**HSRP / VRRP / GLBP:**
- *What I configured:* First-hop redundancy protocols on distribution switches for different VLANs, comparing behavior across all three.
- *Why:* To understand practical differences — HSRP/VRRP active/standby vs. GLBP's load-balancing across multiple gateways.

**IP SLA with Object Tracking:**
- *What I configured:* IP SLA probes tracking upstream reachability, tied to HSRP priority changes.
- *Why:* Enables automatic failover when an uplink fails, not just when the local interface goes down.

**Flexible NetFlow:**
- *What I configured:* NetFlow export from core routers toward a collector.
- *Why:* Traffic visibility is critical for troubleshooting and capacity planning in real networks.

**Syslog Centralization (rsyslog on Ubuntu 20.04):**
- *What I configured:* Centralized logging from all network devices to a single rsyslog server.
- *Why:* Centralized logs are essential for troubleshooting and security auditing at scale.

## 5. Verification
```
show ip ospf neighbor
show standby brief
show ip sla statistics
show flow monitor <name> cache
```
NetFlow export was verified with `tcpdump` on the collector interface, since the lab environment had no internet access for a full collector application.

## 6. Issues Encountered & Troubleshooting

| Issue | Root Cause | How I Diagnosed It | Fix |
|---|---|---|---|
| L2 features not working on some switches | EVE-NG L2 IOL image lacked certain feature support | Compared behavior against L3 image on identical config | Replaced affected nodes with L3 IOL images |
| Devices not reflecting new configs | Stale ARP cache | Pings failing despite correct config | Cleared ARP cache and re-tested |
| Syslog messages not arriving | Logging host IP was transposed (typo) | Checked `show logging` and compared against intended IP | Corrected the IP in the logging configuration |
| Static IP not applying reliably | Manual Netplan YAML edits were inconsistent | Compared results using `nmtui` vs. manual YAML | Switched to `nmtui` for reliable static IP assignment |

## 7. Lessons Learned
- Lab platform limitations (like EVE-NG's L2 IOL gaps) are worth knowing early — they can look like config errors but aren't.
- GUI tools like `nmtui` can be more reliable than manual config file edits when troubleshooting isn't the goal.
- Centralized logging surfaces typos and mistakes that are otherwise invisible until something breaks.

## 8. Skills Demonstrated
Multi-area OSPF design, FHRP comparison and troubleshooting, IP SLA-based failover, NetFlow configuration and verification, centralized syslog architecture, Linux server administration, EVE-NG platform troubleshooting.
