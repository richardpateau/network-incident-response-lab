# INC-007 — NAT Translation Failure

**Status:** Resolved  
**Severity:** High (P2)  
**Device:** R1-EDGE  
**Issue:** Incorrect NAT ACL prevented source translation  
**Tools used:** Wireshark / tcpdump, Splunk, SolarWinds, Cisco IOS CLI

---

## ServiceNow Ticket

![ServiceNow Ticket](evidence/screenshots/01-servicenow-ticket.png)

**Short Description:**  
Users cannot reach external/public network resources

**Description:**  
Internal connectivity works, but traffic from the LAN toward the public network fails.

---

## Topology

![Topology](topology/topology_draw_io.png)

---

## Expected State

- PC1 can reach internal resources
- PC1 traffic to the public network is translated by NAT (Overload) on R1
- Outside source address should appear as `192.0.2.1`
- PC1 can reach `203.0.113.10`

---

## Initial Symptoms

- PC1 could ping the gateway and internal app server
- PC1 could not reach the public server
- Traceroute toward the public network failed

**PC1 ping to gateway successful**

![Gateway Ping Success](evidence/screenshots/02-pc1-ping-gateway-success.png)

**PC1 ping to APP-SERVER1 successful**

![App Server Ping Success](evidence/screenshots/03-pc1-ping-app-server-success.png)

**PC1 ping to Public Web Server failed**

![Public Ping Fail](evidence/screenshots/04-pc1-ping-public-fail.png)

**PC1 traceroute to Public Web Server failed***

![Public Traceroute Fail](evidence/screenshots/05-pc1-traceroute-public-fail.png)

---

## Investigation Evidence

### 1. Interface and Routing Checks

**R1: show ip interface brief**

![Interface Brief](evidence/screenshots/06-r1-show-ip-interface-brief.png)

**R1: show run interface g0/0**

![Inside Interface](evidence/screenshots/07-r1-show-run-interface-g0-0.png)

**R1: show run interface g0/2**

![Outside Interface](evidence/screenshots/08-r1-show-run-interface-g0-2.png)

**R2: show ip route**

![Routing Table](evidence/screenshots/09-r1-show-ip-route.png)

Interfaces were up and R1 had a default route toward the ISP. This was not a routing-table failure.

### 2. NAT Configuration

**R1: show run | section nat**

![NAT Config](evidence/screenshots/10-r1-show-run-nat.png)

**R1: show run | section access-lists**

![ACL Config](evidence/screenshots/11-r1-show-run-access-list.png)

**R1: show ip access-lists NAT-INSIDE**

![Broken NAT ACL](evidence/screenshots/12-r1-show-access-list-nat-inside-broken.png)

The NAT ACL permitted `10.10.20.0/30` instead of the client LAN `10.10.10.0/24`.

### 3. NAT Table and Statistics

**R1: show ip nat translations**

![Empty Translations](evidence/screenshots/13-r1-show-ip-nat-translations-empty.png)

**R1: show ip nat statistics**

![NAT Statistics Broken](evidence/screenshots/14-r1-show-ip-nat-statistics-broken.png)

No valid translation was created for PC1 (`10.10.10.10`).

### 4. Packet Capture

**tcpdump**

![tcpdump](evidence/screenshots/15-tcpdump.png)

**Wireshark Overview**

![Wireshark Broken Overview](evidence/screenshots/15a-wireshark-overview-broken.png)

**Wireshark ICMP echo request**

![ICMP Request](evidence/screenshots/16-wireshark-icmp-request-details.png)

**Wireshark ICMP echo reply**

![ICMP Reply](evidence/screenshots/17-wireshark-icmp-reply-details.png)

**Wireshark Destination Unreachable**

![Destination Unreachable](evidence/screenshots/18-wireshark-dest-unreachable.png)

Capture showed outside-bound traffic failing without a usable NAT translation.

## Configs
 
- Before: [`r1-nat-acl-broken.txt`](evidence/configs/r1-nat-acl-broken.txt)
```
interface GigabitEthernet0/0
 description INSIDE-LAN-SW1
 ip address 10.10.10.1 255.255.255.0
 ip nat inside
 no shutdown

interface GigabitEthernet0/2
 description ISP-OUTSIDE
 ip address 192.0.2.1 255.255.255.252
 ip nat outside
 no shutdown

ip access-list standard NAT-INSIDE
 permit 10.10.20.0 0.0.0.3

ip nat inside source list NAT-INSIDE interface GigabitEthernet0/2 overload
 ```
- After: [`r1-nat-acl-fixed.txt`](evidence/configs/r1-nat-acl-fixed.txt)
```
interface GigabitEthernet0/0
 description INSIDE-LAN-SW1
 ip address 10.10.10.1 255.255.255.0
 ip nat inside
 no shutdown

interface GigabitEthernet0/2
 description ISP-OUTSIDE
 ip address 192.0.2.1 255.255.255.252
 ip nat outside
 no shutdown

ip access-list standard NAT-INSIDE
 permit 10.10.10.0 0.0.0.255

ip nat inside source list NAT-INSIDE interface GigabitEthernet0/2 overload
```

### 5. Monitoring

**SolarWinds**

![SolarWinds](evidence/screenshots/19-solarwinds.png)

**Splunk**

![Splunk](evidence/screenshots/20-splunk.png)

---

## Root Cause

The NAT ACL on R1-EDGE did not match the inside client subnet.

- Configured: `permit 10.10.20.0 0.0.0.3`
- Required: `permit 10.10.10.0 0.0.0.255`

Because PC1 traffic did not match `NAT-INSIDE`, PAT was not applied. Private source addresses were not translated to `192.0.2.1`, so external reachability failed.

Internal routing remained healthy. This was a translation-selection problem, not an OSPF or interface-down issue.

---

## Resolution

Corrected the NAT ACL:

```cisco
no ip access-list standard NAT-INSIDE
ip access-list standard NAT-INSIDE
 permit 10.10.10.0 0.0.0.255
 ```
 
## Verification

After remediation:

- NAT translations were created for PC1
- PC1 could ping and traceroute to the public server
- Packet capture showed successful after-NAT traffic

**R1: show ip nat translation**

![Translations After](evidence/screenshots/23-r1-show-ip-nat-translations-after.png)

**R1: show ip nat statistics**

![NAT Statistics After](evidence/screenshots/24-r1-show-ip-nat-statistics-after.png)

**PC1 ping to Public Web Server successful**

![Public Ping Success](evidence/screenshots/25-pc1-ping-public-success.png)

**PC1 traceroute successful**

![Public Traceroute Success](evidence/screenshots/26-pc1-traceroute-public-success.png)

**Wireshark Overview post-remediation**

![Wireshark Working Overview](evidence/screenshots/27-wireshark-overview-working.png)

**Wireshark ICMP echo request post-remediation**

![After-NAT Request](evidence/screenshots/28-wireshark-after-nat-request.png)

**Wireshark ICMP echo reply post-remediation**

![After-NAT Reply](evidence/screenshots/29-wireshark-after-nat-reply.png)

**debug ip nat post-remediation**

![NAT Debug](evidence/screenshots/30-debug-ip-nat-after.png)

**ServiceNow resolved**

![ServiceNow Resolved](evidence/screenshots/31-servicenow-resolved.png)

---

## Lessons Learned

- Inside reachability does not prove NAT is working
- The NAT ACL must match the real inside source network
- `show ip nat translations` is the fastest way to confirm whether PAT is occurring
- Packet captures should compare inside source `10.10.10.10` with outside source `192.0.2.1`
- ISP should not have a route to the private LAN if the lab is meant to require NAT
