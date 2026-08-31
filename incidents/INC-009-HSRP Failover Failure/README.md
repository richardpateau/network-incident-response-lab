# INC-009 — HSRP Failover Failure

**Status:** Resolved  
**Severity:** High (P2)  
**Devices:** R1-EDGE, R2-EDGE, SW1-ACCESS, PC1  
**Protocol:** HSRPv2  
**VIP:** `10.10.10.1`  
**Tools used:** Wireshark / tcpdump, Splunk, Cisco IOS CLI

---

## ServiceNow Ticket

![ServiceNow Ticket](evidence/screenshots/01-servicenow-ticket.png)

**Short Description:**  
Gateway failover failed after primary router became unavailable

---

## Topology

![Topology](topology/topology_draw_io.png)

---

## Expected State

- PC1 default gateway is HSRP VIP `10.10.10.1`
- R1 (`10.10.10.2`) is Active
- R2 (`10.10.10.3`) is Standby
- If R1 fails, R2 becomes Active for `10.10.10.1`

---

## Initial Checks

PC1 used `10.10.10.1` as its default gateway. While both routers were still up, the physical IPs were reachable:

**PC1 ping to default gateway (10.10.10.1)**

![PC1 Default Gateway](evidence/screenshots/02-pc1-default-gateway.png)

**PC1 ping to R1 g0/0**

![Ping R1 Physical](evidence/screenshots/03-pc1-ping-10.10.10.2-success.png)

**PC1 ping to R2 g0/0**

![Ping R2 Physical](evidence/screenshots/04-pc1-ping-10.10.10.3-success.png)

## Symptom After Primary Failure

After R1’s LAN interface was shut down, PC1 could not reach the VIP `10.10.10.1`.

![PC1 VIP Ping Fail](evidence/screenshots/14-pc1-ping-vip-fail.png)
---

## Investigation Evidence

### 1. R1 HSRP Configuration

**R1: show run interface g0/0**

![R1 Interface Config](evidence/screenshots/05-r1-show-run-interface-g0-0.png)

**R1: show standby brief**

![R1 Standby Brief](evidence/screenshots/06-r1-show-standby-brief.png)

**R1: show standby**

![R1 Standby](evidence/screenshots/07-r1-show-standby.png)

R1 was Active for VIP `10.10.10.1` with priority 110.

### 2. R2 HSRP Configuration (Broken)

**R2: show run interface g0/0**

![R2 Broken Config](evidence/screenshots/08-r2-show-run-interface-g0-0-broken.png)

**R2: show standby brief**

![R2 Standby Brief Broken](evidence/screenshots/09-r2-show-standby-brief-broken.png)

**R2: show standby**

![R2 Standby Broken](evidence/screenshots/10-r2-show-standby-broken.png)

R2 was configured with VIP `10.10.10.11` instead of `10.10.10.1`.

### 3. Failover Test While Misconfigured

R1 LAN interface was shut down:

**R1: show standby during failover**

![R1 After Shutdown](evidence/screenshots/11-r1-show-standby-brief-after-shutdown.png)

**R2: show standby during failover**

![R2 After R1 Failed](evidence/screenshots/12-r2-show-standby-after-r1-failed.png)

**R2: show standby brief during failover**

![R2 Brief After R1 Failed](evidence/screenshots/13-r2-show-standby-brief-after-r1-failed.png)

R2 did not take over `10.10.10.1`. PC1 lost the gateway.

### 4. Packet Capture

**tcpdump**

![tcpdump](evidence/screenshots/15-tcpdump.png)

**Wireshark Overview filtered**

![HSRP Overview](evidence/screenshots/16-wireshark-hsrp-overview.png)

**Wireshark hello details**

![HSRP Hello Details](evidence/screenshots/17-wireshark-hsrp-hello-details.png)

**Wireshark ARP**

![ARP Who Has VIP](evidence/screenshots/18-wireshark-arp-who-has-vip.png)

**Wireshark no ARP reply**

![No ARP Reply for VIP](evidence/screenshots/19-wireshark-arp-no-reply-vip.png)

**Wireshark ARP reply via 10.10.10.3**

![R2 ARP for 10.10.10.3](evidence/screenshots/22-wireshark-hsrp-10.10.10.3-arp-reply.png)

PC1 ARPed for `10.10.10.1` and received no reply. R2 was answering for its own physical IP / wrong VIP path, not for the client gateway.

### 5. Monitoring

`*Aug 29 06:04:51.417: %HSRP-4-DIFFVIP1: GigabitEthernet0/0 Grp 1 active routers virtual IP address 10.10.10.1 is different to the locally configured address 10.10.10.11`

![Splunk](evidence/screenshots/23-splunk.png)

---

## Configs
 
- Before: [`r2-hsrp-broken.txt`](evidence/configs/r2-hsrp-broken.txt)
```
interface GigabitEthernet0/0
 description USER-LAN-to-SW1
 ip address 10.10.10.3 255.255.255.0
 standby version 2
 standby 1 ip 10.10.10.11
 standby 1 priority 100
 standby 1 preempt
 no shutdown
 ```
- After: [`r2-hsrp-fixed.txt`](evidence/configs/r2-hsrp-fixed.txt)
```
interface GigabitEthernet0/0
 description USER-LAN-to-SW1
 ip address 10.10.10.3 255.255.255.0
 standby version 2
 standby 1 ip 10.10.10.1
 standby 1 priority 100
 standby 1 preempt
 no shutdown

 ```
## Root Cause

HSRP virtual IP mismatch.

- R1 VIP: `10.10.10.1`
- R2 VIP: `10.10.10.11`

The routers were not protecting the same gateway address. When R1 failed, R2 became Active for `10.10.10.11`, so VIP `10.10.10.1` disappeared and clients lost their default gateway.

---

## Resolution

Corrected the VIP on R2:

```cisco
interface GigabitEthernet0/0
 no standby 1 ip 10.10.10.11
 standby 1 ip 10.10.10.1
 ```
 
## Verification

Verification was performed **during failover**, with R1 still down.

After the VIP correction:

- R2 became Active for `10.10.10.1`
- PC1 could ping the VIP
- ARP replies for `10.10.10.1` were observed
- ICMP echo request/reply to the VIP succeeded

**R2: show standby post-remediation during failover**

![R2 Active During Failover](evidence/screenshots/26-r2-show-standby-fixed-r1-still-down.png)

**PC1 ping to 10.10.10.1 post-remdiation during failover**

![PC1 VIP Ping Success](evidence/screenshots/27-pc1-ping-vip-success.png)

**Wireshark ICMP echo request and replies post-remediation**

![Echo Request Reply](evidence/screenshots/28-wireshark-echo-request-reply.png)

**Wireshark ARP reply post-remediation**

![VIP ARP Reply](evidence/screenshots/29-wireshark-arp-vip-reply.png)

**Wirehshark filtered by HSRPv2 post-remediation**

![HSRP Keepalives After](evidence/screenshots/30-wireshark-keepalives-after.png)

**Splunk post-remediation after failover (R1 back up)**

`Aug 29 04:28:56 10.10.10.3 96: *Aug 29 08:26:13.912: %HSRP-5-STATECHANGE: GigabitEthernet0/0 Grp 1 state Speak -> Standby`

`Aug 29 04:28:46 172.16.109.140 136: *Aug 29 08:06:19.524: %HSRP-5-STATECHANGE: GigabitEthernet0/0 Grp 1 state Listen -> Active`

![Splunk After Failover](evidence/screenshots/31-splunk-after-failover.png)

**ServiceNow resolved**

![ServiceNow Resolved](evidence/screenshots/34-servicenow-resolved.png)

---

## Lessons Learned

- HSRP members must use the same VIP or else it will not provide immediate redudancy
- A standby router can be up and still fail to protect the client gateway if the VIP does not match
- Test failover, not just “both routers are up”
- ARP for the VIP is a fast way to prove which device owns `10.10.10.1`
