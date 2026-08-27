# INC-006 — DHCP Relay Failure

**Status:** Resolved  
**Severity:** High (P2)  
**Device:** R1-EDGE  
**DHCP Server:** R2 (`10.20.20.1`)  
**Client Network:** `10.10.10.0/24`  
**Tools used:** Wireshark / tcpdump, Splunk, SolarWinds, Cisco IOS CLI

---

## ServiceNow Ticket

![ServiceNow Ticket](evidence/screenshots/01-servicenow-ticket.png)

**Short Description:**  
Users on 10.10.10.0/24 unable to obtain DHCP addresses

---

## Topology

![Topology](topology/topology_draw_io.png)

---

## Expected State

- PC1 obtains an address on `10.10.10.0/24` via DHCP
- R1 relays DHCP to server `10.20.20.1`
- Full DORA completes: Discover → Offer → Request → ACK

---

## Initial Symptoms

- PC1 failed to obtain a DHCP lease

![PC1 No DHCP IP](evidence/screenshots/02-pc1-no-dhcp-ip.png)

---

## Investigation Evidence

### 1. Relay Configuration on R1

**show run interface g0/0**

![Broken Interface Config](evidence/screenshots/03-r1-show-run-interface-g0-0-broken.png)

**show ip interface g0/0**

![Interface Details](evidence/screenshots/04-r1-show-ip-interface-g0-0.png)

**R1: show ip helper-address**

![Helper Address](evidence/screenshots/05-r1-show-ip-helper-address.png)

R1 had:

```cisco
ip helper-address 10.20.20.11
```
The correct DHCP server is `10.20.20.1`.

### 2. Path Checks

**R1: ping 10.20.20.1***

![Ping to real DHCP server](evidence/screenshots/06-r1-ping-10.20.20.1-success.png)

**R1: ping 10.20.20.11**

![Ping to wrong helper target](evidence/screenshots/07-r1-ping-10.20.20.11-fail.png)

**R1: show ip route 10.20.20.0**

![Route to server subnet](evidence/screenshots/08-r1-show-ip-route-10.20.20.0.png)

- `10.20.20.1` was reachable
- `10.20.20.11` was not

### 3. DHCP Server State on R2

**R2: show ip dhcp pool**

![DHCP Pool](evidence/screenshots/09-r2-show-ip-dhcp-pool.png)

**R2: show ip dhcp binding**

![Empty Binding](evidence/screenshots/10-r2-show-ip-dhcp-binding-empty.png)

No valid client lease was created while the helper was wrong.

### 4. Packet Capture / Debug

**tcpdump**

![tcpdump](evidence/screenshots/11-tcpdump.png)

**Wireshark Overview**

![Wireshark Broken Overview](evidence/screenshots/12-wireshark-overview-broken.png)

**Wireshark Discover**

![Discover Details](evidence/screenshots/13-wireshark-discover-details.png)


**Wireshark Relay PCAP**

![Relay Details](evidence/screenshots/14-wireshark-relay-details.png)

**debug**

![DHCP Debug Events](evidence/screenshots/15-debug-dhcp-server-events.png)

**debug**

![DHCP Debug Packet](evidence/screenshots/16-debug-dhcp-server-packet.png)

**PC1: mac-address verification**

![MAC Match](evidence/screenshots/17-debug-mac-match-pc1.png)

Client Discover traffic was present, but relay handling failed because the helper target was invalid.

### 5. Monitoring

**SolarWinds**

![SolarWinds](evidence/screenshots/18-solarwinds.png)


**Splunk**

![Splunk](evidence/screenshots/19-splunk.png)

---

## Configs
 
- Before: [`r1-helper-broken.txt`](evidence/configs/r1-helper-broken.txt)
```
interface GigabitEthernet0/0
 description SW1-PC1-LAN
 ip address 10.10.10.1 255.255.255.0
 ip helper-address 10.20.20.11
 no shutdown
```
- After: [`r1-helper-fixed.txt`](evidence/configs/r1-helper-fixed.txt)
```
interface GigabitEthernet0/0
 description SW1-PC1-LAN
 ip address 10.10.10.1 255.255.255.0
 ip helper-address 10.20.20.1
 no shutdown
```
---

## Root Cause

Incorrect DHCP relay configuration on R1-EDGE.

- Configured helper: `10.20.20.11`
- Actual DHCP server: `10.20.20.1`

Discover packets reached R1, but were relayed to a non-existent host, so no Offer was returned.

This differs from INC-001, which was a local DHCP pool misconfiguration. INC-006 is a relay-path failure to a remote DHCP server.

---

## Resolution

Corrected the helper address on R1:

```cisco
interface GigabitEthernet0/0
 no ip helper-address 10.20.20.11
 ip helper-address 10.20.20.1
 ```
 ## Verification

After remediation:

- PC1 received a DHCP address
- R2 showed an active binding
- PC1 could reach `10.20.20.10`
- Packet capture showed successful DORA

** PC1 obtains a valid IP address via relay agent**

![PC1 Got IP](evidence/screenshots/22-pc1-got-dhcp-ip.png)

**PC1 IP address confirmation**

![PC1 IP Verification](evidence/screenshots/23-pc1-ip-a-verification.png)

**R2: Post-remediation show ip dhcp binding**

![DHCP Binding After](evidence/screenshots/24-r2-show-ip-dhcp-binding-after.png)

**PC1 successfully pinging 10.20.20.10**

![Ping Success](evidence/screenshots/25-pc1-ping-10.20.20.10-success.png)

**Wireshark Overview post-remediation**

![Wireshark Working Overview](evidence/screenshots/26-wireshark-overview-working.png)

**Wireshark offer post-remediation**

![Offer Details](evidence/screenshots/27-wireshark-offer-details.png)

**Wireshark ACK post-remediation**

![ACK Details](evidence/screenshots/28-wireshark-ack-details.png)

**ServiceNow resolved**

![ServiceNow Resolved](evidence/screenshots/29-servicenow-resolved.png)

---

## Lessons Learned

- DHCP relay problems can look like client DHCP failure even when the DHCP server pool is correct
- Always verify `ip helper-address` points to the real server IP
- Compare reachability to the configured helper target vs the intended server
- Packet captures should confirm both client Discover and successful Offer/ACK after remediation
- Centralized DHCP depends on correct relay IP address configuration.
