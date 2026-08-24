# RSPAN (Remote Switched Port Analyzer) — Cross-Switch Traffic Mirroring

## Objective
Configure RSPAN across two switches so that traffic sourced on Switch1 can be
mirrored to a monitoring port physically attached to Switch2, using a dedicated
RSPAN VLAN carried over an 802.1Q trunk. Verify the capture using Wireshark on a
Linux analyzer host.

## Topology

```
                                                192.168.10.1/24
                                    +-------------+   e0/1        e0/0   +-----+
                              Gi0/0 |             |----------------------| Net |
                    +---------------|   Router1   |                     +-----+
                    |               |             |
                    |               +-------------+
                    |
              +-----------+                                    +-----------+
              |  Switch1  |------------ Trunk (Gi0/3) ----------|  Switch2  |
              +-----------+     RSPAN VLAN 999 carried here     +-----------+
             /      |                                                |    \
        Gi0/1     Gi0/2                                          Gi0/1   Gi0/2
          |          |                                              |       |
       eth0        eth0                                            eth0     e0
     +------+    +------+                                       +------+  +--------------------+
     | PC1  |    | PC2  |                                       | PC3  |  |LinuxAnalyzerMonitor|
     +------+    +------+                                       +------+  +--------------------+
  192.168.10.10  192.168.10.20                                192.168.10.40  192.168.10.30
```

| Device               | Interface | IP Address       | Role                                           |
|----------------------|-----------|------------------|-------------------------------------------------|
| PC1                  | eth0      | 192.168.10.10/24 | RSPAN source (traffic endpoint on Switch1)       |
| PC2                  | eth0      | 192.168.10.20/24 | RSPAN source (traffic endpoint on Switch1)       |
| PC3                  | eth0      | 192.168.10.40/24 | Traffic endpoint on Switch2 (optional 2nd host)  |
| Switch1              | Gi0/3     | trunk to Switch2 | Remote source session; VLAN 999 remote-span flag |
| Switch2              | Gi0/3     | trunk to Switch1 | Remote destination session; VLAN 999 remote-span flag |
| LinuxAnalyzerMonitor | e0        | 192.168.10.30/24 | RSPAN destination (Wireshark), attached to Switch2's Gi0/1 |
| Router1              | Gi0/1     | 192.168.10.1/24  | Default gateway / NAT to Net                     |

## Configuration

### VLAN setup (both switches)
The RSPAN VLAN must be explicitly flagged with `remote-span` on **every switch
in the path** — simply creating VLAN 999 and allowing it on the trunk is not
enough on its own (see Troubleshooting below).

```
Switch1(config)# vlan 999
Switch1(config-vlan)# remote-span
Switch1(config-vlan)# exit
```

```
Switch2(config)# vlan 999
Switch2(config-vlan)# remote-span
Switch2(config-vlan)# exit
```

### Trunk (both switches)
```
interface gi0/3
 switchport trunk allowed vlan add 999
```

### Source session — Switch1
```
Switch1(config)# monitor session 1 source interface gi0/1 , gi0/2 both
Switch1(config)# monitor session 1 destination remote vlan 999
```

### Destination session — Switch2
```
Switch2(config)# monitor session 1 source remote vlan 999
Switch2(config)# monitor session 1 destination interface gi0/1
```

> Note: the keyword is `remote vlan` (two words) inside the `monitor session`
> command — not `remote-span vlan`. `remote-span` is only a VLAN-configuration-mode
> keyword, set inside `vlan 999` / `remote-span` as shown above.

## Verification

**Switch1 — source session:**
```
Switch1#show monitor session all
Session 1
---------
Type                     : Remote Source Session
Source Ports             :
    Both                 : Gi0/1-2
Dest RSPAN VLAN        : 999
```

**Switch2 — destination session:**
```
Switch2#show monitor session all
Session 1
---------
Type                     : Remote Destination Session
Source RSPAN VLAN      : 999
Destination Ports      : Gi0/1
    Encapsulation      : Native
```

**Trunk carrying the RSPAN VLAN (both switches):**
```
show interfaces trunk
Port        Vlans allowed on trunk
Gi0/3       1,999
Port        Vlans allowed and active in management domain
Gi0/3       1,999
Port        Vlans in spanning tree forwarding state and not pruned
Gi0/3       1,999
```

**RSPAN VLAN flag (both switches):**
```
show vlan remote-span
Remote SPAN VLANs
------------------------------------------------------------------------------
999
```

## Capture Analysis (Wireshark on LinuxAnalyzerMonitor)

With both sessions correctly configured and VLAN 999 flagged as `remote-span`
on both switches, a ping was run from PC1's own console to PC2. The mirrored
traffic — sourced entirely on Switch1 — was successfully captured on
LinuxAnalyzerMonitor, physically attached to Switch2:

- **ARP** resolution between PC1 and PC2 (Who has 192.168.10.20? / reply)
- **ICMP echo request/reply pairs** between 192.168.10.10 and 192.168.10.20,
  matched cleanly by sequence number, confirming a complete mirrored exchange

This confirms the full path: PC1/PC2 traffic on Switch1 → RSPAN VLAN 999 →
trunk (Gi0/3) → Switch2 → Gi0/1 (destination port) → LinuxAnalyzerMonitor.

## Troubleshooting Notes

This lab surfaced three real issues worth documenting, since diagnosing them
was the most valuable part of the exercise:

### 1. `remote-span vlan` vs `remote vlan` syntax confusion
Initially entered `monitor session 1 source remote-span vlan 999` on the
destination switch, which IOS rejected:
```
% Invalid input detected at '^' marker.
```
`remote-span` is a VLAN-configuration-mode keyword only (`vlan 999` →
`remote-span`). Inside the `monitor session` command, the correct keyword is
`remote vlan` (two separate words).

### 2. Destination port shows "up/down" — this is expected
Once Gi0/1 on Switch2 became a monitor session destination port,
`show ip interface brief` reported:
```
GigabitEthernet0/1     unassigned      YES unset  up                    down
```
Status *up*, but Protocol *down*. This is normal, documented behavior — a
SPAN/RSPAN destination port stops participating in regular Layer 2 forwarding
(STP, CDP, normal switching) and becomes a dedicated, receive-only monitoring
port. LinuxAnalyzerMonitor correspondingly lost its own normal IP connectivity
(no more successful ARP for its gateway) on that port — also expected, not a
fault, since a destination port only exists to output mirrored frames.

### 3. Missing `remote-span` VLAN flag — the actual root cause
Even after both `monitor session` commands were correctly configured and the
trunk was confirmed to allow, actively forward, and carry VLAN 999
(`show interfaces trunk` looked correct on both switches), no mirrored traffic
was reaching the destination port. The cause:
```
Switch1#show vlan remote-span
Remote SPAN VLANs
------------------------------------------------------------------------------
```
Empty output — VLAN 999 existed and was allowed on the trunk, but had never
been flagged as an RSPAN transport VLAN with the `remote-span` VLAN-config
command. Without that flag, both switches treated VLAN 999 as an entirely
ordinary VLAN, so nothing was actually being mirrored into it, despite every
other piece of the configuration being correct. Adding `remote-span` under
`vlan 999` on both switches resolved it immediately — confirmed by
`show vlan remote-span` then listing VLAN 999, and a subsequent capture
showing PC1 ↔ PC2 traffic arriving on LinuxAnalyzerMonitor.

## Key Takeaway

RSPAN has three independent pieces that all have to be correct simultaneously:
the `monitor session` source/destination configuration, the trunk carrying the
RSPAN VLAN, and the VLAN itself being explicitly flagged with `remote-span`.
Each piece can look correct in isolation (VLAN exists, trunk allows it, sessions
reference it) while the mirror still silently fails if any one piece — in this
case the VLAN flag — is missing. `show vlan remote-span` is the fastest way to
confirm that final piece, and is worth checking early rather than last.

## Next Steps
- Extend to ERSPAN: mirror traffic across a routed (Layer 3) topology using GRE
  encapsulation instead of a Layer 2 RSPAN VLAN
- Add PC3 (Switch2) into the ping tests to confirm cross-switch, non-mirrored-source
  traffic is correctly excluded from the RSPAN destination capture
