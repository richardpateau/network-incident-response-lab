# INC-005 — SNMP Monitoring Failure

**Status:** Resolved  
**Severity:** High (P2)  
**Device:** R1-EDGE  
**Protocol:** SNMPv2c  
**Monitoring System:** SolarWinds NPM  
**Tools used:** Wireshark / tcpdump, Splunk, SolarWinds NPM, Cisco IOS CLI

---

## ServiceNow Ticket

![ServiceNow Ticket](evidence/screenshots/01-servicenow-ticket.png)

**Short Description:**  
R1-EDGE SNMP polling failure in SolarWinds

**Description:**  
SolarWinds reported R1-EDGE as SNMP unreachable. The device still responded to ICMP (ping), but monitoring data was not updating.

---

## Topology

![Topology](topology/topology_draw_io.png)

---

## Expected State

- R1-EDGE reachable on the management network
- SolarWinds polls R1 successfully using community `NOC-RO`
- Node status in SolarWinds is Up / managed

---

## Detection
 
- SolarWinds automatically flagged R1-EDGE as SNMP-unreachable when a
scheduled poll failed 
- The device, routing, and connectivity were all fully healthy; only the monitoring visibility
into R1 was gone.
 
![SolarWinds R1 Down](evidence/screenshots/02-solarwinds-r1-down.png)


## Initial Symptoms

- SolarWinds showed R1 as down / SNMP failed
- Monitoring dashboards were not updating for R1

![Node Status Details](evidence/screenshots/03-solarwinds-node-status-details.png)

![SolarWinds Dashboard](evidence/screenshots/04-solarwinds-dashboard.png)

---

## Investigation Evidence

### 1. Reachability Checks

**R1 was able to ping SolarWinds IP**

![Ping R1 to SolarWinds](evidence/screenshots/05-ping-r1-to-solarwinds.png)

**Windows was able to ping R1 (bidirectional communication verified)**

![Ping Windows to R1](evidence/screenshots/06-ping-windows-to-r1.png)

**Intended interfaces are in an (up/up) state**

![Interface Brief](evidence/screenshots/07-r1-show-ip-interface-brief.png)

### 2. ACL Check

![Access Lists](evidence/screenshots/08-r1-show-access-lists.png)

No ACL was blocking SNMP in a way that explained the failure.

### 3. SNMP Configuration

**show  run | section snmp**

![Broken SNMP Config](evidence/screenshots/09-r1-show-run-snmp-broken.png)

**show snmp**

![show snmp](evidence/screenshots/10-r1-show-snmp.png)

**show snmp community**

![show snmp community](evidence/screenshots/11-r1-show-snmp-community.png)

The configured community was `NOC-R0` (zero) instead of the expected `NOC-RO` (letter O).

### 4. Debug Output

![SNMP Debug](evidence/screenshots/12-r1-debug-snmp.png)

![Relevant Debug Detail](evidence/screenshots/13-r1-debug-snmp-details-relevant.png)

### 5. Packet Capture

**tcpdump**

![tcpdump](evidence/screenshots/14-tcpdump.png)

**Wireshark filtered (source: `10.10.10.1`)**

![No Response from R1](evidence/screenshots/15-wireshark-no-response-from-r1-10.10.10.1.png)

**Wireshark filtered (source: `172.16.109.140`)**

![No Response alternate view](evidence/screenshots/15-wireshark-no-response-from-r1-172.16.109.140.png)

**Wireshark get-requests**

![SNMP Requests to R1](evidence/screenshots/16-wireshark-snmp-requests-to-r1.png)

Capture showed SNMP GetRequests reaching R1, with no successful monitoring exchange while the wrong community was configured.

### 6. Logging

![Splunk No Logs](evidence/screenshots/17-splunk-no-logs.png)

No useful SNMP authentication-failure logs were present in Splunk during the incident window.

---

## Configs
 
- Before: [`r1-snmp-broken.txt`](evidence/configs/r1-snmp-broken.txt)

`snmp-server community NOC-R0 RO
snmp-server location "RCA Lab - Edge Router"
snmp-server contact "Network Operations"`

- After: [`r1-snmp-fixed.txt`](evidence/configs/r1-snmp-fixed.txt)

`snmp-server community NOC-RO RO
snmp-server location "RCA Lab - Edge Router"
snmp-server contact "Network Operations"
`

## Root Cause

SNMP community string mismatch on R1-EDGE.

- SolarWinds was polling with: `NOC-RO`
- R1 was configured with: `NOC-R0`

This was a one-character typo (`O` vs `0`).  
ICMP continued to work, so the failure was isolated to SNMP authentication.

---

## Resolution

Restored the correct community string:

```cisco
no snmp-server community NOC-R0
snmp-server community NOC-RO RO
```
![SNMP Fix Applied](evidence/screenshots/18-r1-snmp-fix-applied.png)

![Fixed SNMP Config](evidence/screenshots/19-r1-show-run-snmp-fixed.png)

---

## Verification

After remediation:

- SolarWinds Poll Now succeeded
- Node returned to a healthy monitored state
- Dashboard/node list reflected recovery

**Triggering a poll**

![Poll Now](evidence/screenshots/20-solarwinds-poll-now.png)

**Solarwinds node list post-remediation**

![Node List After](evidence/screenshots/21-solarwinds-node-list-after.png)

**Solarwinds dashboard post-remediation**

![Dashboard After](evidence/screenshots/22-solarwinds-dashboard-after.png)

**ServiceNow resolved**

![ServiceNow Resolved](evidence/screenshots/23-servicenow-resolved.png)

---

## Lessons Learned

- Reachability and SNMP health are independent checks, testing such as ping test does not prove healthy SNMP functionality.
- Small typos in community strings are easy to miss and can fully break polling.
- PCAPs via tcpdump and wireshark are useful for confirming SNMP requests arrive even when responses fail
- Debug commands are very useful when it comes to verifying mismatched SNMP community names. 
