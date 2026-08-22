# Configure Many-to-Many NAT

**Date:** [fill in]
**Certification Track:** CCNP Enterprise
**Tools Used:** EVE-NG, Cisco IOS (IOSv, version 15.9)

---

## 1. Objective
Configure dynamic NAT (many-to-many, not overloaded/PAT) on an edge router so that multiple inside hosts translate to distinct addresses from a defined outside pool, rather than sharing a single outside IP.

## 2. Topology

- **Devices:** Router3 (outside/ISP-side router), Router4 (NAT router), Switch1, Switch2 (Layer 2, unconfigured), PC1, PC2
- **Role of each device:**
  - Router3 represents the outside/ISP network — no NAT configuration, just a routed interface.
  - Router4 is the NAT boundary — inside interface faces the LAN, outside interface faces Router3.
  - Switch1 and Switch2 provide Layer 2 connectivity between Router4 and the two PCs.

## 3. IP Addressing Table

| Device | Interface | IP Address | Role | Notes |
|---|---|---|---|---|
| Router3 | Gi0/0 | 180.10.1.1/24 | Outside network | No NAT config |
| Router4 | Gi0/0 | 180.10.1.2/24 | NAT outside | `ip nat outside` |
| Router4 | Gi0/1 | 192.168.1.1/24 | NAT inside / default gateway | `ip nat inside` |
| Switch1 / Switch2 | — | N/A | L2 switching only | No IP configured |
| PC1 | eth0 | 192.168.1.2/24 | Inside host | GW: 192.168.1.1 |
| PC2 | eth0 | 192.168.1.3/24 | Inside host | GW: 192.168.1.1 |

## 4. Configuration Steps

**Interface roles (Router4):**
- *What I configured:* Marked Gi0/0 as the NAT outside interface and Gi0/1 as the NAT inside interface.
- *Why:* NAT needs to know which side of the router is "inside" (private) and "outside" (public) to know which direction to translate traffic.
- *Key commands:*
```
interface GigabitEthernet0/0
 ip address 180.10.1.2 255.255.255.0
 ip nat outside

interface GigabitEthernet0/1
 ip address 192.168.1.1 255.255.255.0
 ip nat inside
```

**Defining the NAT pool and matching traffic:**
- *What I configured:* A pool of 11 outside addresses (180.10.1.5–180.10.1.15) and an ACL matching the entire inside subnet, then bound the two together.
- *Why:* This is what makes it "many-to-many" — each inside host gets mapped to its own address from the pool, rather than all hosts sharing one address via port overloading.
- *Key commands:*
```
ip nat pool NAT-Pool 180.10.1.5 180.10.1.15 netmask 255.255.255.0
access-list 2 permit 192.168.1.0 0.0.0.255
ip nat inside source list 2 pool NAT-Pool
```

## 5. Verification
```
Router4#show ip nat translations
Pro Inside global      Inside local       Outside local      Outside global
--- 180.10.1.6         192.168.1.2        ---                ---
--- 180.10.1.5         192.168.1.3        ---                ---
```
This confirms the translation is working as designed: PC1 (192.168.1.2) is mapped to 180.10.1.6, and PC2 (192.168.1.3) is mapped to 180.10.1.5 — two distinct outside addresses for two distinct inside hosts, with no port numbers involved (confirming this is dynamic NAT, not PAT).

## 6. Issues Encountered & Troubleshooting
None reported for this lab. *(If anything came up while building this — an ACL typo, a pool exhaustion, a translation that wouldn't clear — let me know and I'll add it here; this section is usually the most valuable part for an interviewer.)*

## 7. Lessons Learned
- The distinction between dynamic NAT (pool-to-pool, one-to-one) and PAT/NAT overload (many-to-one with port translation) is visible directly in the `show ip nat translations` output — no port numbers means no overloading.
- ACL 2 here isn't a security ACL; it's being used purely to define which traffic qualifies for translation.

## 8. Skills Demonstrated
Dynamic NAT pool configuration, inside/outside interface designation, standard ACL use for traffic matching, NAT translation verification.
