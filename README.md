# Secure Redundant Enterprise Network Design

In this lab I designed and configured a small enterprise network that stays online if a router fails, keeps a server locked down, and logs suspicious devices without cutting anyone off. It covers HSRP, OSPF, ACLs, NAT and port security.

## What I built

The network has four routers (HQ, R-STAFF, R-MGMT and ISP-INTERNET) and three switches serving three VLANs:

- **VLAN 10, Staff:** 192.168.10.0/24
- **VLAN 20, Management and Finance:** 192.168.20.0/24
- **VLAN 30, Server:** 192.168.30.0/24

My goals were:

1. Give users a gateway that survives a router failure (HSRP).
2. Let all routers learn each other's networks automatically (OSPF).
3. Allow only the management VLAN to reach the server.
4. Keep the server completely unreachable from the Internet.
5. Add sticky-MAC port security that only logs violations, so it can be used for auditing.

## Repository contents

| File | Description |
|------|-------------|
| `README.md` | This write-up |
| `config.txt` | Full configuration for every device, plus the test plan |

## Addressing plan

| Segment | Network | Details |
|---------|---------|---------|
| VLAN 10 Staff | 192.168.10.0/24 | VIP .1, R-STAFF .2, R-MGMT .3 |
| VLAN 20 Mgmt/Finance | 192.168.20.0/24 | VIP .1, R-STAFF .2, R-MGMT .3 |
| VLAN 30 Server | 192.168.30.0/24 | HQ G0/3 .1, SERVER1 .10 |
| R-STAFF to R-MGMT | 10.0.1.0/24 | .1 / .2 |
| HQ to R-STAFF | 10.0.2.0/24 | HQ .1 / R-STAFF .2 |
| HQ to R-MGMT | 10.0.3.0/24 | HQ .1 / R-MGMT .2 |
| HQ to ISP | 203.0.113.0/30 | HQ .1 / ISP .2 |

Hosts: STAFF1 192.168.10.11, STAFF2 192.168.10.12, FINANCEMANAGER 192.168.20.11, MANAGER 192.168.20.12, SERVER1 192.168.30.10.

## How I approached it

**HSRP.** Both R-STAFF and R-MGMT connect to the switches over trunks, using router-on-a-stick subinterfaces. Instead of making one router do all the work, I split the load: R-STAFF is active for VLAN 10 and R-MGMT is active for VLAN 20. Each is the standby for the other's VLAN. I enabled preempt, MD5 authentication and tracking on the uplink to HQ, so a router with a dead uplink gives up its active role.

**OSPF.** All routers run area 0 with MD5 authentication. I made the LAN-facing subinterfaces and the server link passive so no routing updates leak toward end devices. HQ advertises a default route to the rest of the network using `default-information originate`.

**Server access control.** I put an outbound ACL on HQ G0/3, the port facing the server. It permits only 192.168.20.0/24 and denies and logs everything else. Putting it outbound filters traffic going to the server while leaving the server's replies alone.

**Keeping the server off the Internet.** I used three layers:
- An inbound ACL on HQ G0/2 that blocks anything addressed to 192.168.30.0/24.
- NAT that only translates VLAN 10 and VLAN 20, so VLAN 30 is never exposed.
- An ISP with no route to my private address space.

**Port security.** Every access port uses sticky MAC with a maximum of 1 and violation mode `restrict`. In this mode an unknown device's frames are dropped and the event is logged and counted, but the port stays up. That gives me an audit trail without causing outages. I also enabled PortFast and BPDU guard on access ports.

## Configuration

The complete configuration for all devices is in [`config.txt`](config.txt).

## How I verify it

| Test | Command or action | What I expect |
|------|-------------------|---------------|
| Trunks | `show interfaces trunk` | Trunking, VLANs 10 and 20 allowed |
| HSRP roles | `show standby brief` | R-STAFF active for VLAN 10, R-MGMT active for VLAN 20 |
| HSRP failover | Shut R-STAFF Gi0/2, ping 192.168.10.1 | R-MGMT takes over, 1 to 3 pings lost |
| OSPF | `show ip ospf neighbor` | All neighbours FULL |
| Routes | `show ip route ospf` | VLAN 10, 20, 30 routes and the default route learned |
| Inter-VLAN | STAFF1 ping 192.168.20.12 | Success |
| Internet | STAFF1 and MANAGER ping 8.8.8.8 | Success, NAT entries appear on HQ |
| Server from VLAN 20 | MANAGER ping 192.168.30.10 | Success |
| Server from VLAN 10 | STAFF1 ping 192.168.30.10 | Fails (administratively prohibited) |
| Server from Internet | ISP ping 192.168.30.10 | Fails, deny counters rise on `FROM-INTERNET` |
| Sticky MAC | `show port-security interface g0/2` | Secure-up, 1 sticky MAC, mode Restrict |
| Violation | Connect an unknown host to a secured port | Frames dropped, port stays up, violation count and log entry increase |

