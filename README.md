# Network Engineering Lab Portfolio — Guy Malake Edimo

Hands-on enterprise networking labs built in EVE-NG, documenting design decisions, configurations, and real troubleshooting — as part of CCNP-level self-study.

## Certifications
- CCNA
- CompTIA A+ / Network+
- IBM IT Support
- Google Cybersecurity
- CCNP (in progress)

## Labs

| Lab | Topics Covered | Status |
|---|---|---|
| [OSPF, HSRP/VRRP/GLBP, IP SLA, NetFlow, Syslog](./example-ospf-hsrp-netflow-lab.md) | Multi-area OSPF, FHRP, object tracking, NetFlow, centralized logging | Complete |
| AAA Infrastructure with TACACS+ | tac_plus-ng build from source, AAA on IOS devices | In progress |
| [Many-to-Many NAT](./many-to-many-nat-lab.md) | Dynamic NAT pool, inside/outside interfaces, ACL-based translation | Complete |
| IPsec / GRE / VRF Tunneling | Site-to-site VPN, transform-set compatibility | Complete |
| VTPv3 / STP-MST / EtherChannel | Layer 2 optimization and redundancy | Complete |

(This list is being filled in lab-by-lab with real configs and verification output — several more labs exist in the EVE-NG environment and will be added over time.)

## How This Portfolio Is Organized
Each lab follows a consistent format: objective, topology, configuration with reasoning, verification, troubleshooting, and lessons learned. The troubleshooting sections are intentionally detailed, they're the clearest evidence of real problem-solving ability, not just following a guide.
