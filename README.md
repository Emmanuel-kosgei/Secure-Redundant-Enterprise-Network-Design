# Secure Redundant Enterprise Network Design

Topics: HSRP · OSPF · ACL · NAT · Port Security

## Summary

This lab builds a secure, redundant enterprise network with four routers (HQ, R-STAFF, R-MGMT, ISP) and three switches. HSRP gives VLAN 10 (Staff) and VLAN 20 (Management/Finance) gateway redundancy, and OSPF with MD5 authentication handles routing. An ACL lets only VLAN 20 reach the VLAN 30 server, which is also blocked from the Internet. Sticky-MAC port security in restrict mode logs violations for auditing without shutting ports down.

## Repository contents

| File | Description |
|------|-------------|
| `README.md` | This write-up |
| `topology.json` | Topology exported from the Sandbox |
| `configs/full-config.txt` | Complete configuration for every device, plus the test plan |
| `topology.png` | Screenshot of the topology (optional) |

## Objectives

- Provide first-hop redundancy with HSRP, load-shared between two routers.
- Run OSPF area 0 with MD5 authentication and passive LAN interfaces.
- Allow only the Management VLAN to reach the server.
- Keep the server unreachable from the Internet.
- Apply sticky-MAC port security (restrict mode) for auditing, then verify everything.

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

## Design

- HSRP: R-STAFF is active for VLAN 10 (priority 110) and R-MGMT for VLAN 20 (priority 110). Each is standby for the other, with preempt, MD5 authentication and uplink tracking.
- OSPF: Area 0 on all routers with MD5 authentication. LAN-facing subinterfaces and the server link are passive. HQ originates the default route toward the ISP.
- Server isolation: An outbound ACL on HQ G0/3 permits only 192.168.20.0/24 and denies and logs everything else.
- Server hidden from the Internet: An inbound ACL on HQ G0/2 blocks traffic to 192.168.30.0/24. VLAN 30 is excluded from NAT, so the server cannot be reached from or browse the Internet.
- Port security: Sticky MAC, maximum 1, violation mode `restrict` on every access port, plus PortFast and BPDU guard.

## Configuration

The full configuration for all devices is in [`configs/full-config.txt`]config.txt).

## Verification

| Test | Command or action | Expected result |
|------|-------------------|-----------------|
| Trunks | `show interfaces trunk` | Trunking, VLANs 10 and 20 allowed |
| HSRP roles | `show standby brief` | R-STAFF active VLAN 10, R-MGMT active VLAN 20 |
| HSRP failover | Shut R-STAFF Gi0/2, ping 192.168.10.1 | R-MGMT takes over, 1 to 3 pings lost |
| OSPF | `show ip ospf neighbor` | All neighbours FULL |
| Routes | `show ip route ospf` | VLAN 10, 20, 30 routes and default route learned |
| Inter-VLAN | STAFF1 ping 192.168.20.12 | Success |
| Internet | STAFF1 and MANAGER ping 8.8.8.8 | Success (NAT entries on HQ) |
| Server, VLAN 20 | MANAGER ping 192.168.30.10 | Success |
| Server, VLAN 10 | STAFF1 ping 192.168.30.10 | Fails (administratively prohibited) |
| Server from Internet | ISP ping 192.168.30.10 | Fails, deny counters rise on `FROM-INTERNET` |
| Sticky MAC | `show port-security interface g0/2` | Secure-up, 1 sticky MAC, mode Restrict |
| Violation | Connect an unknown host to a secured port | Frames dropped, port stays up, violation count and log entry increase |

## Notes

- MANAGER and FINANCEMANAGER share VLAN 20, so both can reach the server. To allow only MANAGER, change the first `TO-SERVER` line to `permit ip host 192.168.20.12 192.168.30.0 0.0.0.255`.
- If a switch rejects `switchport trunk encapsulation dot1q`, omit it. If HSRP interface tracking is rejected, use `track 1 interface Gi0/2 line-protocol` with `standby 10 track 1 decrement 20`.
