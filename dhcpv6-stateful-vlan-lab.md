# DHCPv6 Stateful Addressing Across a Two-Tier Switched Access Layer

## Objective

Build a working DHCPv6 stateful addressing design across a Core -> Distribution -> Access topology in EVE-NG, with the Core switch acting as both the IPv6 gateway and DHCPv6 server for two client VLANs, plus two directly-attached routers each requesting their own DHCPv6-leased address.

## Topology

```
                         CORE
        Vlan100 2001:DB8:A::1/64   Vlan200 2001:DB8:B::1/64
                 Gi0/0 -- 2001:DB8:C::1/64 -- Gi0/0 Router3
        Gi0/2 --+                    +-- Gi0/1
                 |                    |
              DSW1                  DSW2 -- Gi0/0 -- Gi0/0 Router2
        Vlan100 ::2/64          Vlan100 ::3/64
        Vlan200 ::3/64          Vlan200 ::2/64
              Gi0/1                  Gi0/2
                 |                    |
              ASW1                  ASW2
              Gi0/0                 Gi0/0
                 |                    |
                PC1                  PC2
             (Vlan100)             (Vlan200)
```

## DHCPv6 pools (on CORE)

| Pool | Prefix | DNS server | Domain | Bound to |
|---|---|---|---|---|
| DHCPV6POOL100 | 2001:DB8:A::/64 | 2001:DB8:A::89 | lab.local | Vlan100 |
| DHCPV6POOL200 | 2001:DB8:B::/64 | 2001:DB8:B::89 | lab.local | Vlan200 |
| DHCPV6POOL300b | 2001:DB8:C::/64 | 2001:DB8:A::89 | lab.local | Gi0/0 (Router3 link) |

## Design

CORE hosts the VLAN 100 and VLAN 200 SVIs directly and is the DHCPv6 server for both. Gi0/1 and Gi0/2 on CORE are 802.1Q trunks carrying those VLANs down to DSW1 and DSW2, which in turn trunk them further down to ASW1 and ASW2, where PC1 and PC2 sit on plain access ports. Because CORE is Layer-2-adjacent to both client subnets through this trunk chain, no DHCPv6 relay is needed for PC1/PC2 -- only Router3 (directly routed off Gi0/0) needs its own dedicated pool since it sits on a separate point-to-point subnet.

*Note: this replaced an earlier design where DSW1/DSW2 held their own SVIs and relayed DHCPv6 back to CORE over routed point-to-point links. That design kept hitting Layer 2 delivery problems (see Troubleshooting Log below), so it was simplified to the trunked, Core-hosted-SVI model documented here.*

## Final configurations

### CORE

```
ipv6 unicast-routing
ipv6 cef

ipv6 dhcp pool DHCPV6POOL100
 address prefix 2001:DB8:A::/64
 dns-server 2001:DB8:A::89
 domain-name lab.local

ipv6 dhcp pool DHCPV6POOL200
 address prefix 2001:DB8:B::/64
 dns-server 2001:DB8:B::89
 domain-name lab.local

ipv6 dhcp pool DHCPV6POOL300b
 address prefix 2001:DB8:C::/64
 dns-server 2001:DB8:A::89
 domain-name lab.local

interface GigabitEthernet0/0
 no switchport
 no ip address
 ipv6 address 2001:DB8:C::1/64
 ipv6 dhcp server DHCPV6POOL300b

interface GigabitEthernet0/1
 switchport trunk encapsulation dot1q
 switchport mode trunk

interface GigabitEthernet0/2
 switchport trunk encapsulation dot1q
 switchport mode trunk

interface Vlan100
 ipv6 address 2001:DB8:A::1/64
 ipv6 nd managed-config-flag
 ipv6 nd other-config-flag
 ipv6 dhcp server DHCPV6POOL100

interface Vlan200
 ipv6 address 2001:DB8:B::1/64
 ipv6 nd managed-config-flag
 ipv6 nd other-config-flag
 ipv6 dhcp server DHCPV6POOL200
```

### DSW1

```
ipv6 unicast-routing
ipv6 cef
vlan 100,200

interface GigabitEthernet0/1
 switchport trunk encapsulation dot1q
 switchport mode trunk

interface GigabitEthernet0/2
 switchport trunk encapsulation dot1q
 switchport mode trunk

interface Vlan100
 no ipv6 address

interface Vlan200
 no ipv6 address
```

### DSW2

```
ipv6 unicast-routing
ipv6 cef
vlan 100,200

interface GigabitEthernet0/0
 switchport access vlan 200
 switchport mode access

interface GigabitEthernet0/1
 switchport trunk encapsulation dot1q
 switchport mode trunk

interface GigabitEthernet0/2
 switchport trunk encapsulation dot1q
 switchport mode trunk

interface Vlan100
 no ipv6 address

interface Vlan200
 no ipv6 address
```

### ASW1

```
vlan 100,200

interface GigabitEthernet0/0
 switchport access vlan 100
 switchport mode access

interface GigabitEthernet0/1
 switchport trunk encapsulation dot1q
 switchport mode trunk
```

### ASW2

```
vlan 100,200

interface GigabitEthernet0/0
 switchport access vlan 200
 switchport mode access

interface GigabitEthernet0/2
 switchport trunk encapsulation dot1q
 switchport mode trunk
```

### Router2 / Router3 (client-facing interface)

```
interface GigabitEthernet0/0
 ipv6 address dhcp
```

## Troubleshooting log

The lab reported "nothing works -- PCs can't get an address, neither can Router2 or Router3." Root causes were found and fixed one layer at a time:

1. **DHCPv6 pool bindings swapped on CORE.** Gi0/1 and Gi0/2 were originally bound to the wrong pools for their subnets. Fixed by matching each interface's `ipv6 dhcp server` binding to its actual subnet.

2. 2. **DSW1 missing `ipv6 unicast-routing` / `ipv6 cef`.** Without these, DSW1 wasn't functioning as an IPv6 router at all -- no Router Advertisements, no relay. Added both globally.
  
   3. 3. **Missing `ipv6 nd managed-config-flag`** on the client-facing SVIs. Without it, clients aren't told via RA to actually request an address over DHCPv6 (stateful) rather than just SLAAC.
     
      4. 4. **Router2 and Router3 configured for SLAAC instead of DHCPv6.** Both had `ipv6 address autoconfig` where `ipv6 address dhcp` was needed -- SLAAC and DHCPv6 client mode are mutually exclusive on an interface.
        
         5. 5. **Trunk/access-port mismatches, found and fixed at three separate links, all with the same signature (zero DHCPv6 bindings, RA/relay traffic silently dropped):**
            6.    - **CORE <-> DSW1 / DSW2:** an early design had CORE's downstream ports as plain routed (`no switchport`) interfaces while DSW1/DSW2 sent 802.1Q-tagged trunk traffic -- a non-VLAN-aware routed port can't parse tagged frames. Resolved by redesigning so both ends use matching 802.1Q trunks (see **Design** above).
                  -    - **DSW1 <-> ASW1:** DSW1's downlink (Gi0/1) was a trunk, but ASW1's corresponding port had no switchport configuration at all -- sitting in default VLAN 1. Confirmed via `show cdp neighbors`, then fixed by trunking the matching port on ASW1.
                       -    - **DSW2 <-> ASW2:** same symptom, subtler cause -- ASW2 already had a trunk configured, but on the *wrong physical port* (Gi0/1 instead of Gi0/2, the actual CDP-confirmed uplink to DSW2). Fixed by trunking the correct port.
                        
                            - 6. **Leftover Layer 3 configuration from the earlier relay-based design.** After redesigning so CORE hosts the VLAN 100/200 SVIs directly, DSW1 and DSW2 still had their own `ipv6 address` / `ipv6 nd managed-config-flag` / `ipv6 dhcp relay destination` configured on those same VLANs -- creating two competing gateways on one subnet, with a relay agent sitting uselessly on the same broadcast domain as the actual server. Cleared the leftover Layer 3 config from DSW1/DSW2's Vlan100/Vlan200 interfaces so CORE is the sole router for those subnets.
                             
                              7. Each of the port-mismatch issues was diagnosed the same way: `show ipv6 dhcp binding` on CORE showing zero leases for a given subnet -> `ping` between the two ends of the suspect link failing -> `show cdp neighbors` to confirm which physical port was actually in play -> compare `show run` on both ends of that link to find the mismatch.
                             
                              8. ## Verification
                             
                              9. ```
                                 CORE#show ipv6 dhcp binding
                                 Client: FE80::F16D:5B99:2A34:BD96          (PC1, VLAN 100)
                                     Address: 2001:DB8:A:0:9C6B:764C:FA6F:1897
                                 Client: FE80::5200:FF:FE09:0               (Router2, VLAN 200)
                                     Address: 2001:DB8:B:0:D54A:B33C:D56D:5E04
                                 Client: FE80::4C3B:F103:BF9C:172D          (PC2, VLAN 200)
                                     Address: 2001:DB8:B:0:75DF:70B1:4D84:A612
                                 Client: FE80::5200:FF:FE08:0               (Router3, VLAN C)
                                     Address: 2001:DB8:C:0:4D81:33FB:23A4:2654
                                 ```

                                 All four clients -- both PCs and both routers -- hold active stateful DHCPv6 leases from CORE.

                                 ## Key takeaways

                                 - A trunk on one end of a link and an unconfigured (default VLAN 1) or mismatched port on the other is the single most common reason DHCPv6/RA traffic silently disappears in a switched topology -- it produces no error, just zero bindings.
                                 - - `show cdp neighbors` is the fastest way to confirm which physical port is actually in play before trusting a topology diagram or assumed port numbering.
                                   - - `ipv6 nd managed-config-flag` is easy to forget and easy to miss, since clients can still appear to get some IPv6 connectivity via SLAAC without it -- it only breaks the specific requirement of stateful DHCPv6 addressing.
                                     - - Mixing a DHCPv6 relay agent onto the same broadcast domain as the actual DHCPv6 server (leftover from an earlier design) creates a redundant, confusing point of failure -- clean out Layer 3 config that's no longer doing anything when a design changes.
                                       - 
