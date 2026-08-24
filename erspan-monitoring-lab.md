# ERSPAN (Encapsulated Remote Switched Port Analyzer) — Layer 3 Traffic Mirroring

> **Note on this entry:** Unlike the other labs in this portfolio, this entry is
> **conceptual/design-level rather than hands-on verified**. ERSPAN requires
> Nexus-class (NX-OS) switches, and running two Nexus virtual nodes on this
> EVE-NG host consistently pegged CPU usage at ~99% (confirmed via the EVE-NG
> status dashboard), making the topology too unstable to boot and test
> reliably. The design, configuration, and reasoning below reflect a real
> attempt at this lab and genuine platform troubleshooting — documented
> honestly as a resource-constrained limitation rather than a completed
> capture, consistent with how this portfolio otherwise only marks labs
> "Complete" once verified.

## Objective
Design an ERSPAN topology that mirrors traffic from a source switch to a
destination switch across a **routed (Layer 3) boundary**, using GRE
encapsulation — as opposed to SPAN (same switch) or RSPAN (same Layer 2
domain via a dedicated VLAN over a trunk).

## Why ERSPAN, and how it differs from SPAN/RSPAN

| Feature | Scope | Transport |
|---|---|---|
| SPAN  | Same switch | Direct port copy |
| RSPAN | Multiple switches, same L2 domain | Dedicated RSPAN VLAN over trunks |
| ERSPAN | Anywhere, even across L3/WAN | GRE-encapsulated, routed |

RSPAN requires the source and destination switches to share a Layer 2 domain
(a VLAN carried over a trunk). ERSPAN removes that constraint entirely by
wrapping mirrored frames in a **GRE tunnel** and routing them over IP — the
monitoring destination can sit anywhere reachable via Layer 3, including
across a WAN. This is why ERSPAN requires Nexus/Catalyst-class hardware
support: it needs the switch to originate and terminate a GRE tunnel as part
of its packet-mirroring pipeline, which IOSv/IOSvL2 platforms (used
elsewhere in this portfolio's labs) do not support — confirmed via
`monitor session ... type erspan-source` returning `% Invalid input
detected` on those platforms during initial testing, before switching to
Nexus nodes.

## Proposed Topology

```
              +-----------+                              +-----------+
              |  Switch1  |                              |  Switch2  |
              +-----------+                              +-----------+
             /      |                                          |     \
        Gi0/1     Gi0/2                                     Gi0/1    Gi0/2
          |          |                                         |        |
       eth0        eth0                                       eth0      e0
     +------+    +------+                                  +------+  +--------------------+
     | PC1  |    | PC2  |                                  | PC3  |  |LinuxAnalyzerMonitor|
     +------+    +------+                                  +------+  +--------------------+
  192.168.10.10  192.168.10.20                          192.168.20.40  192.168.20.30

              Switch1 Gi0/0                                Switch2 Gi0/0
                    \                                            /
                     \                                          /
                      +------------------ Router1 ------------+
                        Gi0/1 = 192.168.10.1/24  Gi0/2 = 192.168.20.1/24
                        (Router1 routes between the two subnets)
```

| Device               | Interface | IP Address        | Role                                              |
|----------------------|-----------|------------------|----------------------------------------------------|
| PC1                  | eth0      | 192.168.10.10/24  | ERSPAN source (traffic endpoint on Switch1)         |
| PC2                  | eth0      | 192.168.10.20/24  | ERSPAN source (traffic endpoint on Switch1)         |
| PC3                  | eth0      | 192.168.20.40/24  | Traffic endpoint on Switch2 (optional)              |
| Switch1              | Gi0/0     | 192.168.10.2/24   | ERSPAN source session; GRE origin                   |
| Switch2              | Gi0/0     | 192.168.20.2/24   | ERSPAN destination session; GRE terminus            |
| Router1              | Gi0/1     | 192.168.10.1/24   | Routes between Switch1's and Switch2's subnets      |
| Router1              | Gi0/2     | 192.168.20.1/24   | Routes between Switch1's and Switch2's subnets      |
| LinuxAnalyzerMonitor | e0        | 192.168.20.30/24  | ERSPAN destination (Wireshark), attached to Switch2  |

Putting Switch1 and Switch2 on **separate, routed subnets** (rather than a
shared trunk, as in the RSPAN lab) is intentional — it's what actually
exercises the Layer 3 / GRE part of ERSPAN, rather than just reusing the
RSPAN topology with different commands.

## Planned Configuration (NX-OS syntax)

Confirmed available on Nexus via CLI help (`monitor session 1 type ?` listed
`erspan-source` as a valid option, unlike on IOSv/IOSvL2):

**Source session — Switch1 (near PC1/PC2):**
```
switch(config)# monitor session 1 type erspan-source
switch(config-erspan-src)# source interface eth1/1 , eth1/2 both
switch(config-erspan-src)# destination ip 192.168.20.2
switch(config-erspan-src)# erspan-id 1
switch(config-erspan-src)# ip ttl 64
switch(config-erspan-src)# ip prec 0
switch(config-erspan-src)# no shut
```

**Destination session — Switch2 (near LinuxAnalyzerMonitor):**
```
switch(config)# monitor session 1 type erspan-destination
switch(config-erspan-dst)# destination interface eth1/1
switch(config-erspan-dst)# source ip 192.168.20.2
switch(config-erspan-dst)# erspan-id 1
switch(config-erspan-dst)# no shut
```

Key points captured from planning this out:
- The `erspan-id` must match on both source and destination sessions — it
  functions as a session key so the destination switch knows which GRE
  traffic belongs to which ERSPAN session (relevant if running multiple
  ERSPAN sessions simultaneously).
- `destination ip` on the source session and `source ip` on the destination
  session must reference the **same IP** — this is the GRE tunnel endpoint,
  typically a routed/L3 interface on the destination switch.
- Unlike IOS `monitor session` commands, NX-OS ERSPAN sessions are
  **admin-down by default** and require an explicit `no shut` before they
  become active.
- Router1 needs standard inter-VLAN/inter-subnet routing between
  192.168.10.0/24 and 192.168.20.0/24 for the GRE-encapsulated ERSPAN
  traffic (and the underlying PC1/PC2 traffic itself) to actually reach
  Switch2.

## Platform / Resource Findings

- IOSv and IOSvL2 (used for the SPAN and RSPAN labs in this portfolio) do
  **not** support ERSPAN — `monitor session ... type erspan-source` is
  rejected outright on those images.
- Nexus virtual switches (e.g., Nexus 9000v) do support it, confirmed via
  CLI availability, but are significantly more resource-intensive to
  virtualize than IOSv/IOSvL2.
- Running 2 Nexus nodes on this EVE-NG host drove CPU usage to ~99%
  (EVE-NG status dashboard), which is not a stable baseline to build and
  verify a working lab on top of.

## Next Steps
- If access to a higher-resource EVE-NG host (or a lighter Nexus-class
  image) becomes available, complete hands-on verification of this design
  and upgrade this entry's status to Complete with real `show monitor
  session` output and a Wireshark capture, matching the format of the SPAN
  and RSPAN entries.
- In the meantime, this entry stands as evidence of ERSPAN's architecture,
  the GRE-based mechanism that differentiates it from RSPAN, and the
  platform-support research required to even attempt it.
