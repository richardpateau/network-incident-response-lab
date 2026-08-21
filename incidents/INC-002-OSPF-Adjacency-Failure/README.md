# INC-002 — OSPF Neighbor Adjacency Failure

**Status:** Resolved  
**Severity:** Medium  
**Devices:** R1-EDGE, R2, R3  
**Protocol:** OSPF  
**Tools used:** Wireshark / tcpdump, Splunk, SolarWinds, Cisco IOS CLI

---

## ServiceNow Ticket

![ServiceNow Ticket](evidence/screenshots/01-service-now-ticked.png)

**Short Description:** Users unable to reach remote network (`10.20.20.0/24`)

---

## Topology

![Topology](topology/topology_draw_io.png)

---

## Expected State

- OSPF Area `0` across R1, R2, and R3
- R1 should form a **FULL** adjacency with R2
- R1 should have an OSPF route to `10.20.20.0/24`
- PC1 should be able to reach `10.20.20.1`

---

## Initial Symptoms

- PC1 could not reach the remote network `10.20.20.0/24`
- Local connectivity appeared normal
- R1 could reach R2, but remote OSPF routes were missing

![PC1 Ping Fail](evidence/screenshots/02-pc1-ping-fail.png)

![R1 Ping Fail](evidence/screenshots/02a-r1-ping-fail.png)

---
## Detection

Splunk flagged the missing OSPF adjacency and the loss of the
10.20.20.0/24 route.

`%OSPF-4-ERRRCV: Received invalid packet: mismatched area ID from backbone area from 10.10.20.1, GigabitEthernet0/0`

![Splunk Alert](evidence/screenshots/09-splunk.png)

## Investigation Evidence

### 1. OSPF Neighbor State (Broken)

**R1:**

![R1 OSPF Neighbor Broken](evidence/screenshots/03-r1-show-ip-ospf-neighbor-broken.png)

**R2:**

![R2 OSPF Neighbor Broken](evidence/screenshots/03a-r2-show-ip-ospf-neighbor-broken.png)

**R3:**

![R3 OSPF Neighbor Broken](evidence/screenshots/03b-r3-show-ip-ospf-neighbor-broken.png)

R1 did not form a FULL adjacency with R2.

### 2. Routing Table (Broken)

**R1:**

![R1 OSPF Route Broken](evidence/screenshots/04-r1-show-ip-route-ospf-broken.png)

**R2:**

![R2 OSPF Route Broken](evidence/screenshots/04a-r2-show-ip-route-ospf-broken.png)

**R3:**

![R3 OSPF Route Broken](evidence/screenshots/04b-r3-show-ip-route-ospf-broken.png)

The route to `10.20.20.0/24` was missing on R1.

### 3. OSPF Interface Brief

**R1:**

![R1 OSPF Interface Brief](evidence/screenshots/05-r1-show-ip-ospf-interface-brief.png)

**R2:**

![R2 OSPF Interface Brief](evidence/screenshots/05a-r2-show-ip-ospf-interface-brief.png)

**R3:**

![R3 OSPF Interface Brief](evidence/screenshots/05b-r3-show-ip-ospf-interface-brief.png)

### 4. OSPF Configuration

**R1 (Area 0):**

![R1 OSPF Config](evidence/screenshots/06-r1-show-running-config-ospf.png)

**R2 (incorrect — transit link in Area 1):**

![R2 OSPF Config Broken](evidence/screenshots/06a-r2-show-running-config-ospf.png)

**R3:**

![R3 OSPF Config](evidence/screenshots/06b-r3-show-running-config-ospf.png)

### 5. Packet Capture

![TCP Dump](evidence/screenshots/07-tcp-dump.png)

![Wireshark OSPF Hellos](evidence/screenshots/08-wireshark-ospf-hellos-only.png)

OSPF Hello packets were exchanged, but the adjacency did not progress to FULL because of the area mismatch.

## Configs

- Before: [`r1-ospf-before.txt`](configs/r1-ospf-before.txt) · [`r2-ospf-before.txt`](configs/r2-ospf-before.txt) · [`r3-ospf-before.txt`](configs/r3-ospf-before.txt)
- After: [`r1-ospf-after.txt`](configs/r1-ospf-after.txt) · [`r2-ospf-after.txt`](configs/r2-ospf-after.txt) · [`r3-ospf-after.txt`](configs/r3-ospf-after.txt)

### 6. Monitoring

**SolarWinds:**

![SolarWinds](evidence/screenshots/10-solarwinds.png)

---

## Root Cause

R2 had the transit network `10.10.20.0/30` configured in **OSPF Area 1**, while R1 had the same link in **Area 0**.

This area mismatch prevented the OSPF adjacency from reaching FULL. As a result, routes from R2 (including `10.20.20.0/24`) were not installed on R1.

OSPF requires routers on the same segment to be configured with the same area ID in the Hello packet. A mismatched area ID will lead to neighbors not forming a FULL adjacency, causing them to stay in a INIT/2-WAY state. 

Packet Capture shows that LSU, LSR and LSAack were never exchanged which is why route `10.20.20.0/24` was never installed into the routing table. 

---

## Resolution

Corrected the OSPF network statement on R2:

```cisco
router ospf 1
 no network 10.10.20.0 0.0.0.3 area 1
 network 10.10.20.0 0.0.0.3 area 0
 ```

## Verification

After remediation:

- R1 formed a FULL adjacency with R2
- The `10.20.20.0/24` route appeared in R1’s OSPF table
- PC1 and R1 could successfully reach `10.20.20.1`

**R1 show ip ospf neighbor:**
![R1 Neighbor Fixed](evidence/screenshots/11-r1-show-ip-ospf-neighbor-fixed.png)

**R2 show ip ospf neighbor:**
![R2 Neighbor Fixed](evidence/screenshots/11a-r2-show-ip-ospf-neighbor-fixed.png)

**R1 show ip route:**
![R1 OSPF Route Fixed](evidence/screenshots/12-r1-show-ip-route-ospf-fixed.png)

**R1 show ip route:**
![R1 Routing Table Fixed](evidence/screenshots/13-r1-show-ip-route-fixed.png)

**Post Remediation Wireshark:**
![Wireshark Post Remediation](evidence/screenshots/14-wireshark-post-remediation.png)

**Hello Packet Details:**
![Hello Details](evidence/screenshots/14a-wireshark-hello-details.png)

**DBD Packet Details:**
![DBD Details](evidence/screenshots/14b-wireshark-dbd-details.png)

**LSR Packet Details:**
![LSR Details](evidence/screenshots/14c-wireshark-lsr-details.png)

**LSU Packet Details:**
![LSU Details](evidence/screenshots/14d-wireshark-lsu-details.png)

**LSAck Packet Details:**
![LSAck Details](evidence/screenshots/14e-wireshark-ls-ack-details.png)

`%OSPF-5-ADJCHG: Process 1, Nbr 1.1.1.1 on GigabitEthernet0/0 from LOADING to FULL, Loading Done`
`%OSPF-5-ADJCHG: Process 1, Nbr 2.2.2.2 on GigabitEthernet0/3 from LOADING to FULL, Loading Done`
![Splunk After](evidence/screenshots/15-splunk-after-remediation.png)

**PC1 Successful Ping:**
![PC Ping After](evidence/screenshots/16a-pc-ping-after-remediation.png)

**R1 Successful Ping:**
![R1 Ping After](evidence/screenshots/16-r1-post-ping-after-remediation.png)

**Resolved Service Now Ticket:**
![ServiceNow Resolved](evidence/screenshots/17-service-now-resolved.png)

---

## Lessons Learned

- OSPF neighbors must use the same area on the same ethernet segment. 
- Hello packets alone do not mean the adjacency is healthy
- Wireshark is useful for confirming whether the adjacency forms beyond Hello (DBD / LSR / LSU / LSAck)
