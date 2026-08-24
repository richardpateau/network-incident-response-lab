# INC-004 — OSPF Adjacency Down Due to MTU Mismatch

**Status:** Resolved  
**Severity:** High (P2)  
**Devices:** R1-EDGE, R2  
**Protocol:** OSPF  
**Related Change:** CHG-0245  
**Tools used:** Wireshark / tcpdump, Splunk, Cisco IOS CLI

---

## ServiceNow Ticket

![ServiceNow Ticket](evidence/screenshots/01-servicenow-ticket.png)

**Short Description:**  
OSPF adjacency down between R1-EDGE and R2 after interface change

**Description:**  
Following CHG-0245 (hardware/module replacement on R1-EDGE Gi0/3), routing between R1-EDGE and R2 stopped converging. Users behind R1-EDGE could not reach networks behind R2 (`10.20.20.0/24`).

Interfaces were up/up, but the OSPF neighbor relationship did not reach FULL.

---

## Topology

![Topology](topology/topology.png)

---

## Expected State

- R1 and R2 form a FULL OSPF adjacency over `10.10.20.0/30`
- Both sides use MTU 1500
- R1 learns `10.20.20.0/24` via OSPF
- PC1 can reach `10.20.20.10`

---

## Detection
 
 ![Splunk Before](evidence/screenshots/20-splunk-before.png)

## Initial Symptoms

- PC1 could not reach the application server
- Traceroute stopped at R1

![PC1 Ping Fail](evidence/screenshots/02-pc1-ping-fail.png)

![Traceroute Stops at R1](evidence/screenshots/02a-pc1-traceroute-stops-at-R1.png)

---

## Investigation Evidence

### 1. OSPF Neighbor State

Neighbor was stuck in **EXSTART** on both sides:

![R1 EXSTART](evidence/screenshots/03-r1-ospf-neighbor-exstart.png)

![R2 EXSTART](evidence/screenshots/04-r2-ospf-neighbor-exstart.png)

### 2. Interface MTU Comparison

![R1 MTU](evidence/screenshots/05-r1-interface-mtu.png)

![R2 MTU](evidence/screenshots/06-r2-interface-mtu.png)

- R1 Gi0/3: **MTU 1400**
- R2 Gi0/2: **MTU 1500**

### 3. OSPF Interface View

![R1 OSPF Interface](evidence/screenshots/07-r1-ospf-interface.png)

![R2 OSPF Interface](evidence/screenshots/08-r2-ospf-interface.png)

### 4. OSPF Configuration

![R1 OSPF Config](evidence/screenshots/09-r1-show-run-ospf.png)

![R2 OSPF Config](evidence/screenshots/10-r2-show-run-ospf.png)

### 5. Routing Table

![Missing Route](evidence/screenshots/11-r1-route-missing.png)

`10.20.20.0/24` was missing on R1.

### 6. Debug Output

![R1 Debug](evidence/screenshots/12-r1-debug-ospf-adj.png)

![R2 Debug](evidence/screenshots/13-r2-debug-ospf-adj.png)

Debug confirmed adjacency problems during DBD exchange.

### 7. Packet Capture

![tcpdump](evidence/screenshots/14-tcpdump.png)

![Wireshark Overview](evidence/screenshots/14a-wireshark-before-overview.png)

![No LSR/LSU/LSAck](evidence/screenshots/15-wireshark-no-lsr-lsu-lsack.png)

![R1 Hello](evidence/screenshots/16-wireshark-r1-hello-details.png)

![R2 Hello](evidence/screenshots/17-wireshark-r2-hello-details.png)

![R1 DBD](evidence/screenshots/18-wireshark-r1-dbd-details.png)

![R2 DBD](evidence/screenshots/19-wireshark-r2-dbd-details.png)

Hellos were present, but DBD exchange did not progress to LSR / LSU / LSAck. DBD details showed the MTU mismatch.

## Configs
 
- Before: [`r1-mtu-broken.txt`](evidence/configs/r1-mtu-broken.txt) 
```
interface GigabitEthernet0/3
 description TO-R2
 ip address 10.10.20.1 255.255.255.252
 mtu 1400
 no shutdown

router ospf 1
 router-id 1.1.1.1
 network 10.10.10.0 0.0.0.255 area 0
 network 10.10.20.0 0.0.0.3 area 0

	```
· [`r2-mtu-baseline.txt`](evidence/configs/r2-mtu-baseline.txt)
```
interface GigabitEthernet0/2
 description TO-R1
 ip address 10.10.20.2 255.255.255.252
 mtu 1500
 no shutdown

router ospf 1
 router-id 2.2.2.2
 network 10.10.20.0 0.0.0.3 area 0
 network 10.20.20.0 0.0.0.255 area 0
	```
- After: [`r1-mtu-after-fix.txt`](evidence/configs/r1-mtu-after-fix.txt)
```
interface GigabitEthernet0/3
 description TO-R2
 ip address 10.10.20.1 255.255.255.252
 mtu 1500
 no shutdown

router ospf 1
 router-id 1.1.1.1
 network 10.10.10.0 0.0.0.255 area 0
 network 10.10.20.0 0.0.0.3 area 0
```


## Root Cause

After CHG-0245, R1's interface was left with **MTU 1400** while R2 remained at **MTU 1500**.

OSPF parameters matched (hello/dead timers, area, process ID), so the adjacency advanced past 2-WAY, but Database Description (DBD) exchange failed because OSPF includes interface MTU in DBD packets. The neighbor remained stuck in **EXSTART**.

---

## Resolution

Restored matching MTU on R1:

```cisco
interface GigabitEthernet0/3
 mtu 1500
 ```
![MTU Fix Applied](evidence/screenshots/21-r1-mtu-fix-applied.png)

**Note:** `ip ospf mtu-ignore` was considered as a workaround. It would bring the adjacency up without fixing the underlying MTU mismatch, which can still cause problems for larger packets.

---

## Verification

After remediation:

- OSPF neighbor reached **FULL**
- `10.20.20.0/24` returned to the OSPF routing table
- Interface MTU on R1 was 1500
- PC1 could reach the application server
- Wireshark showed LSR / LSU / LSAck progression

** R1: show ip ospf neighbor **

![R1 Neighbor FULL](evidence/screenshots/22-r1-ospf-neighbor-full.png)

** R2: show ip ospf neighbor **

![R2 Neighbor FULL](evidence/screenshots/23-r2-ospf-neighbor-full.png)


![R1 OSPF Route Fixed](evidence/screenshots/24-r1-route-ospf-fixed.png)

![R2 OSPF Route Fixed](evidence/screenshots/25-r2-route-ospf-fixed.png)

![R1 MTU Fixed](evidence/screenshots/26-r1-interface-mtu-fixed.png)

** PC1 successful ping post-remediation **

![PC1 Ping Success](evidence/screenshots/27-pc1-ping-success.png)

** Wireshark overview post-remediation **
![Wireshark Post](evidence/screenshots/28-wireshark-post-lsr-lsu-lsack.png)

** Splunk dashboard post-remediation **
![Splunk After](evidence/screenshots/29-splunk-after.png)

** ServiceNow resolved **
![ServiceNow Resolved](evidence/screenshots/30-servicenow-resolved.png)

---

## Lessons Learned

- Interface up/up only proves Layer 1/2 health, not OSPF parameters 
- The stuck OSPF state matters: EXSTART/EXCHANGE points to DBD/MTU issues, while INIT/2-WAY points to Hello/area/authentication problems
- Always compare MTU on both sides after hardware or interface module changes
- `ip ospf mtu-ignore` masks the symptom; matching MTU is the correct fix
- DBD packets captured via Wireshark are also helpful to provide concrete evidence of a MTU mismatch in addition to Cisco CLI. 
