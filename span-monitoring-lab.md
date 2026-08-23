# SPAN (Switched Port Analyzer) — Local Traffic Mirroring

## Objective
Configure a local SPAN session on a Layer 2 switch to mirror traffic from all VLAN 1
ports to a dedicated monitoring port, and verify the capture using Wireshark on a
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
              +-----------+
              |  Switch1  |  VLAN 1 192.168.10.2/24
              +-----------+
             /      |      \
        Gi0/1     Gi0/2     Gi0/3
          |          |          |
       eth0        eth0        e0
     +------+    +------+   +--------------------+
     | PC1  |    | PC2  |   | LinuxAnalyzerMonitor|
     +------+    +------+   +--------------------+
  192.168.10.10  192.168.10.20   192.168.10.30
```

| Device               | Interface | IP Address       | Role                         |
|----------------------|-----------|------------------|-------------------------------|
| PC1                  | eth0      | 192.168.10.10/24 | SPAN source (traffic endpoint)|
| PC2                  | eth0      | 192.168.10.20/24 | SPAN source (traffic endpoint)|
| Switch1              | Gi0/0     | trunk to Router1 | SPAN source (VLAN 1)          |
| LinuxAnalyzerMonitor | e0        | 192.168.10.30/24 | SPAN destination (Wireshark)  |
| Router1              | Gi0/1     | 192.168.10.1/24  | Default gateway / NAT to Net  |

## Configuration

SPAN was configured on Switch1 to mirror **both directions of all traffic on VLAN 1**
to the analyzer port (Gi0/3), rather than restricting the source to individual
physical ports. This captures everything traversing VLAN 1 — including inter-switch
control-plane traffic — not just the PC1/PC2 conversation.

```
Switch1(config)# monitor session 1 source vlan 1 both
Switch1(config)# monitor session 1 destination interface gi0/3
```

### Verification

```
Switch1#sh monitor session all
Session 1
---------
Type                     : Local Session
Source VLANs             :
    Both                 : 1
Destination Ports      : Gi0/3
    Encapsulation      : Native
```

This confirms:
- **Source**: VLAN 1, both directions (rx and tx)
- **Destination**: Gi0/3 (LinuxAnalyzerMonitor)
- **Encapsulation**: Native (mirrored frames sent untagged, matching the destination port's config)

## Capture Analysis (Wireshark on LinuxAnalyzerMonitor)

With the SPAN session active, a ping was run from PC1 (192.168.10.10) to PC2
(192.168.10.20). Because the source was set to the entire VLAN rather than a single
port, the capture picked up a realistic mix of traffic types, not just the targeted
ICMP conversation:

| Protocol | Observation |
|----------|-------------|
| **ICMP** | PC1 → PC2 echo request/reply pairs, confirming SPAN successfully mirrors the targeted host-to-host conversation |
| **ARP**  | Resolution traffic (e.g., "Who has 192.168.10.1/ Tell...") — confirms SPAN mirrors broadcast traffic, not just unicast |
| **STP**  | Spanning-Tree Conf. BPDUs — confirms SPAN mirrors Layer 2 control-plane traffic traversing the VLAN, not just end-host data |
| **NTP**  | Client/server exchange between LinuxAnalyzerMonitor and an external NTP server — confirms the analyzer itself has working internet connectivity through Router1's NAT |
| **LOOP** | A loopback-detection frame, consistent with STP's loop-guard/keepalive mechanism on a switched topology |

## Key Takeaway

A VLAN-based SPAN source mirrors **all traffic traversing that VLAN**, not just the
traffic between the two hosts of interest. This is useful for broad visibility (e.g.,
catching STP or ARP issues alongside the target conversation) but means the capture
will always include more than expected — worth accounting for with a display filter
in Wireshark (e.g., `ip.addr==192.168.10.10 && ip.addr==192.168.10.20`) when isolating
a specific conversation for analysis.

## Next Steps
- Apply a Wireshark display filter to isolate just the PC1 ↔ PC2 ICMP exchange for a cleaner before/after comparison
- Extend this lab to RSPAN: move LinuxAnalyzerMonitor to a second switch and mirror traffic across a trunk via a dedicated RSPAN VLAN
