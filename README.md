# Network Incident Response Lab

A PNETLab-based portfolio of network incidents documented as realistic RCA cases.

Each incident follows the same workflow:

**Ticket → Symptoms → Investigation → Evidence → Root Cause → Fix → Verification → Close**

Tools used across the lab:

- Cisco CSR1000v / IOSvL2
- Wireshark / tcpdump
- Splunk
- SolarWinds NPM
- ServiceNow-style ticketing

---

## Purpose

This project shows how I approach network troubleshooting in an operations environment, not just that I can build a topology.

The incidents are ordered roughly by increasing complexity, starting with a straightforward Layer 2/3 service failure and building toward multi-stage, protocol-internals problems (OSPF DBD/MTU, BGP prefix advertisement, HSRP redundancy logic) where the fix requires understanding *why* the protocol behaves the way it does, not just which command to run.

The goal for every case is to produce evidence an employer can review:

- a ticket
- CLI outputs
- packet captures
- monitoring screenshots
- before/after configs
- a written root-cause analysis

---

## Lab Overview

The environment is a small enterprise-style network built in PNETLab and reused across incidents:

- Access LAN and clients
- Edge routing
- Internal application network
- WAN / ISP connectivity
- Centralized logging and SNMP monitoring

Individual incident folders include the topology used for that case.

---

## Incidents

| Incident | Issue | Skills Shown |
|---|---|---|
| [INC-001](incidents/INC-001-DHCP-Failure/) | Clients cannot obtain an IP address | DHCP, DORA, Wireshark, config mismatch |
| [INC-002](incidents/INC-002-OSPF-Adjacency-Failure/) | OSPF adjacency failure | OSPF states, routing table, packet analysis |
| [INC-003](incidents/INC-003-ACL-Traffic-Blocked/) | Change blocks more traffic than intended | Extended ACLs, two-stage RCA, security policy |
| [INC-004](incidents/INC-004-OSPF-MTU-Mismatch/) | OSPF stuck in EXSTART | MTU, DBD exchange, adjacency state machine |
| [INC-005](incidents/INC-005-SNMP-Monitoring-Failure/) | Device reachable but unmanaged via SNMP | SNMP, SolarWinds, UDP/161, layered checks |
| [INC-006](incidents/INC-006-DHCP-Relay-Failure/) | Clients fail to get addresses from a remote server | DHCP relay, helper-address, centralized DHCP |
| [INC-007](incidents/INC-007-NAT-Translation-Failure/) | Internal works, external fails | NAT/PAT, NAT ACL, translation table |
| [INC-008](incidents/INC-008-BGP-Route-Advertisement-Failure/) | BGP up, prefix missing | eBGP, network statement, advertised-routes |
| [INC-009](incidents/INC-009-HSRP-Failover-Failure/) | Gateway does not fail over | HSRP, VIP mismatch, ARP, failover testing |

---

## How Each Incident Is Structured

```text
INC-00X-Name/
├── README.md
├── topology/
└── evidence/
    ├── configs/
    ├── pcaps/
    └── screenshots/
```
Every README includes:

1. Ticket
2. Expected state
3. Symptoms
4. Investigation evidence
5. Root cause
6. Resolution
7. Verification
8. Lessons learned

---

## Tools and Methods

**CLI**

- Cisco routing, switching, DHCP, NAT, BGP, HSRP, ACL, and SNMP verification

**Packet analysis**

- DHCP DORA
- OSPF Hello/DBD
- ACL drops
- SNMP polling
- NAT translation
- HSRP/ARP failover

**Monitoring**

- SolarWinds for device/interface/routing neighbor status
- Splunk for syslog correlation
- Cases where monitoring showed nothing are documented as well

**Ticketing**

- ServiceNow-style intake and resolution notes

---

## Example Findings

- INC-001: DHCP pool did not match the client subnet
- INC-003: An over-broad deny hid a leftover HTTPS ACE
- INC-004: OSPF failed at DBD because of an MTU mismatch, not a down link
- INC-005: ICMP worked; SNMP failed because of a community typo (`NOC-R0` vs `NOC-RO`)
- INC-008: BGP remained Established while the wrong prefix was advertised
- INC-009: Standby was up, but the VIP did not match, so failover failed

---

## How To Review This Repo

Start with this file, then open any incident README. The strongest cases to sample first are:

1. [INC-003 ACL](incidents/INC-003-ACL-Traffic-Blocked/)
2. [INC-004 OSPF MTU](incidents/INC-004-OSPF-MTU-Mismatch/)
3. [INC-007 NAT](incidents/INC-007-NAT-Translation-Failure/)
4. [INC-009 HSRP](incidents/INC-009-HSRP-Failover-Failure/)

---

## Notes

- This is a lab environment, not a production network
- Faults were injected so the troubleshooting process could be documented end to end
- Monitoring gaps are included on purpose when SolarWinds or Splunk did not generate useful alerts
