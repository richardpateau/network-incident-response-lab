# INC-008 — BGP Route Advertisement Failure

**Status:** Resolved  
**Severity:** High (P2)  
**Devices:** R1-EDGE, ISP-R1  
**Protocol:** BGP  
**Prefix:** `10.10.10.0/24`  
**Tools used:** Wireshark / tcpdump, Splunk, SolarWinds NPM (Routing Insights), Cisco IOS CLI

---

## ServiceNow Ticket

![ServiceNow Ticket](evidence/screenshots/01-servicenow-ticket.png)

**Short Description:**  
Missing BGP advertisement for 10.10.10.0/24 toward ISP

---

## Topology

![Topology](topology/topology.png)

---

## Expected State

- BGP session between R1-EDGE (`192.0.2.1`) and ISP-R1 (`192.0.2.2`) is Established
- R1 advertises `10.10.10.0/24` to ISP
- ISP installs the prefix in the BGP table and routing table
- External host can reach `10.10.10.10`

---

## Initial Symptoms

- External/web server ping to `10.10.10.10` failed

![Web Server Ping Fail](evidence/screenshots/02-web-server-ping-fail.png)

---

## Investigation Evidence

### 1. Local Route and Interfaces on R1

**R1: show ip interface brief**

![Interface Brief](evidence/screenshots/03-r1-show-ip-interface-brief.png)

**R1: show ip route 10.10.10.0**

![Local Route](evidence/screenshots/04-r1-show-ip-route-10.10.10.0.png)

`10.10.10.0/24` existed locally on R1.

### 2. BGP Session Health

**R1: show ip bgp summary**

![R1 BGP Summary](evidence/screenshots/05-r1-show-ip-bgp-summary.png)

**R1: show ip bgp neighbors**

![R1 BGP Neighbors](evidence/screenshots/06-r1-show-ip-bgp-neighbors.png)

The BGP neighbor remained **Established**. This was not an adjacency-down issue.

### 3. Advertisement Check on R1

**R1: show ip bgp**

![R1 BGP Table](evidence/screenshots/07-r1-show-ip-bgp.png)

**R1: show ip bgp 10.10.10.0**

![R1 BGP 10.10.10.0](evidence/screenshots/08-r1-show-ip-bgp-10.10.10.0.png)

**R1: show run | section bgp**

![Broken BGP Config](evidence/screenshots/09-r1-show-run-section-bgp-broken.png)

**R1**

![Advertised Routes Broken](evidence/screenshots/10-r1-advertised-routes-broken.png)

R1 was configured with:

```cisco
network 10.10.100.0 mask 255.255.255.0
```

instead of:

```cisco
network 10.10.10.0 mask 255.255.255.0
```
`10.10.10.0/24` was not advertised to ISP.

### 4. ISP View

**ISP: show ip bgp summary**

![ISP BGP Summary](evidence/screenshots/11-isp-show-ip-bgp-summary.png)

**ISP: show ip bgp**

![ISP BGP Table](evidence/screenshots/12-isp-show-ip-bgp.png)

**ISP: show ip bgp 10.10.10.0**

![ISP BGP 10.10.10.0](evidence/screenshots/13-isp-show-ip-bgp-10.10.10.0.png)

**ISP: show ip route 10.10.10.0**

![ISP Route Missing](evidence/screenshots/14-isp-show-ip-route-10.10.10.0.png)

**ISP:**

![ISP Advertised Routes](evidence/screenshots/15-isp-advertised-routes.png)

ISP did not have `10.10.10.0/24` via BGP.

### 5. Packet Capture

**tcpdump**

![tcpdump](evidence/screenshots/16-tcpdump.png)

**Wireshark Overview**

![Wireshark Broken Overview](evidence/screenshots/17-wireshark-overview-broken.png)

### 6. Monitoring

**SolarWinds dashboard**

![SolarWinds Dashboard](evidence/screenshots/18-solarwinds-dashboard.png)

**SolarWinds Routing Insights dashboard**

![Routing Insights Before](evidence/screenshots/19-solarwinds-routing-insights-before.png)

**Solarwinds**

![Routing Neighbors Still Up](evidence/screenshots/20-solarwinds-routing-neighbors-up.png)

**Splunk**

![Splunk](evidence/screenshots/21-splunk.png)

---

## Configs
 
- Before: [`r1-bgp-broken.txt`](evidence/configs/r1-bgp-broken.txt)
```
router bgp 65000
 bgp log-neighbor-changes
 neighbor 192.0.2.2 remote-as 65001
 network 10.10.100.0 mask 255.255.255.0
 ```
- After: [`r1-bgp-fixed.txt`](evidence/configs/r1-bgp-fixed.txt)
```
router bgp 65000
 bgp log-neighbor-changes
 neighbor 192.0.2.2 remote-as 65001
 network 10.10.10.0 mask 255.255.255.0

 ```
- ISP: [`r1-bgp-fixed.txt`](evidence/configs/isp-bgp-baseline.txt)
```
router bgp 65001
 bgp log-neighbor-changes
 neighbor 192.0.2.1 remote-as 65000
 network 203.0.113.0 mask 255.255.255.0
```
---

## Root Cause

Incorrect BGP `network` statement on R1-EDGE.

- Configured: `10.10.100.0/24`
- Required: `10.10.10.0/24`

BGP only advertises a `network` statement when that exact prefix exists in the routing table. `10.10.100.0/24` did not exist, so nothing matching was advertised. The session stayed up.

This differs from INC-002 and INC-004, which were OSPF adjacency failures. INC-008 is a prefix-advertisement failure with the neighbor still Established.

---

## Resolution

Corrected the BGP network statement on R1:

```cisco
router bgp 65000
 no network 10.10.100.0 mask 255.255.255.0
 network 10.10.10.0 mask 255.255.255.0
 ```
 
## Verification

After remediation:

- R1 advertised `10.10.10.0/24`
- ISP installed the prefix
- ISP successfully reached `10.10.10.10`

**R1: show ip bgp post-remediation**

![R1 BGP After](evidence/screenshots/23-r1-show-ip-bgp-after.png)

**R1: show ip post-remediation**

![R1 Advertised Routes After](evidence/screenshots/24-r1-advertised-routes-after.png)

**ISP: show ip bgp post-remediation**

![ISP BGP After](evidence/screenshots/25-isp-show-ip-bgp-after.png)

**ISP: show ip bgp 10.10.10.0 post-remediation**

![ISP BGP Prefix After](evidence/screenshots/26-isp-show-ip-bgp-10.10.10.0-after.png)

**ISP: show ip route 10.10.10.0 post-remediation**

![ISP Route After](evidence/screenshots/27-isp-show-ip-route-10.10.10.0-after.png)

**ISP ping to 10.10.10.10 successful**

![ISP Ping Success](evidence/screenshots/28-isp-ping-10.10.10.10-success.png)

**Wireshark Overview post-remediation**

![Wireshark After](evidence/screenshots/29-wireshark-overview-after.png)

**ServiceNow resolved**

![ServiceNow Resolved](evidence/screenshots/30-servicenow-resolved.png)

---

## Lessons Learned

- A BGP session can be established while a required prefix is still not being advertised
- The `network` statement must match the exact prefix and mask in the routing table
- Always compare local route presence, advertised-routes, and the peer’s BGP table
- Neighbor-up status in monitoring does not prove a prefix is being advertised
