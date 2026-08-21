# AAA Infrastructure with TACACS+ (tac_plus-ng)

**Date:** [fill in]
**Certification Track:** CCNP Enterprise
**Tools Used:** EVE-NG, Cisco IOS, Ubuntu 20.04 (tac_plus-ng server)

**Status:** In progress — foundation complete, AAA configuration next.

---

## 1. Objective
Build a full AAA (Authentication, Authorization, Accounting) infrastructure using TACACS+ to centrally control administrative access to Router1, Switch1, and Switch2.

## 2. Topology
*Insert topology screenshot from EVE-NG here.*

- **Devices:** Router1, Switch1, Switch2, Ubuntu-TACACS-Server
- **Role of each device:** Router1/Switch1/Switch2 = AAA clients; Ubuntu server = TACACS+ daemon (tac_plus-ng)

## 3. IP Addressing Table

| Device | Interface | IP Address | VLAN / Subnet | Notes |
|---|---|---|---|---|
| *(fill in from your lab)* | | | | |

## 4. Configuration Steps

### Stage 1: DHCP, NAT/PAT, and Internet Connectivity — ✅ Complete
- *What I configured:* DHCP services, NAT/PAT for outbound access, and verified internet connectivity for lab devices.
- *Why:* Establishes the foundational connectivity the AAA infrastructure depends on before adding authentication controls.
- *Key commands:*
```
(fill in your actual DHCP/NAT config here)
```

### Stage 2: Building and Installing tac_plus-ng from Source — ✅ Complete
- *What I configured:* Compiled and installed `tac_plus-ng` from source on an Ubuntu 20.04 VM.
- *Why:* tac_plus-ng isn't available as a standard package on most distros, so building from source was required to get a working TACACS+ daemon.
- *Key commands:*
```
(fill in your actual build/install commands here)
```

### Stage 3: User Accounts and AAA Blocks — ⏳ Not started yet
- *What I'll configure:* User accounts and authentication/authorization/accounting blocks in the tac_plus-ng config.
- *Why:* This defines who can log in and what they're authorized to do.

### Stage 4: IOS-Side AAA Configuration (Router1, Switch1, Switch2) — ⏳ Not started yet
- *What I'll configure:* `aaa new-model`, TACACS+ server definitions, and method lists on each device.
- *Why:* This is what actually points the devices at the TACACS+ server for login authentication.

## 5. Verification
*(To be filled in once AAA is configured — e.g. `test aaa group tacacs+`, login testing, `debug tacacs`)*

## 6. Issues Encountered & Troubleshooting
*(To be filled in as they come up)*

## 7. Lessons Learned
*(To be filled in at the end)*

## 8. Skills Demonstrated
TACACS+ daemon compilation and deployment, centralized AAA design, Cisco IOS AAA configuration *(update once complete)*.
