# INC-001 — Clients Cannot Obtain an IP Address

**Status:** Resolved  
**Severity:** Medium  
**Device:** R1-EDGE  
**Network:** 10.10.10.0/24  

---

## ServiceNow Ticket

![ServiceNow Ticket](evidence/screenshots/01-servicenow-ticket.png)

**Short Description:** Users unable to obtain network connectivity

---

## Topology

![Topology](topology/topology.png)

---

## Expected State

- Network: `10.10.10.0/24`
- Gateway: `10.10.10.1`
- DHCP Server: R1-EDGE
- Clients should automatically receive IP addresses via DHCP

---

## Initial Symptoms

- PC1 was unable to obtain a valid IP address
- Client continuously sent DHCP Discover packets
- No network connectivity

![PC1 No IP](evidence/screenshots/02-pc1-no-ip.png)

---

## Investigation Evidence

### 1. Wireshark Capture (Broken State)

![Wireshark Discover Only](evidence/screenshots/03-wireshark-discover-only.png)

**Finding:** Repeated DHCP Discover packets from the client (`0.0.0.0` → `255.255.255.255`). No DHCP Offer was received.

![Discover Packet Details](evidence/screenshots/03a-discover-packet-details.png)

### 2. DHCP Pool

![Wrong DHCP Pool](evidence/screenshots/04-show-ip-dhcp-pool-wrong.png)

The pool was configured for `10.10.20.0/24` instead of `10.10.10.0/24`.

### 3. Interface Status

![Interface Brief](evidence/screenshots/05-show-ip-interface-brief.png)

### 4. DHCP Bindings

![Empty Bindings](evidence/screenshots/06-show-ip-dhcp-binding-empty.png)

No leases were issued.

### 5. DHCP Configuration

![Running Config DHCP](evidence/screenshots/07-show-running-config-dhcp.png)

```cisco
ip dhcp pool LAN
 network 10.10.20.0 255.255.255.0
 default-router 10.10.20.1
```

### 6. Monitoring

**Splunk:**

![Splunk](evidence/screenshots/09-splunk.png)

A `%DHCPD-4-PING_CONFLICT` log was observed for `10.10.10.2` (SW1 management IP). No other critical DHCP errors were found.

**SolarWinds:**

![SolarWinds Dashboard](evidence/screenshots/10a-solarwinds-dashboard.png)

![SolarWinds Nodes](evidence/screenshots/10b-solarwinds-managed-nodes.png)

All devices and interfaces remained up. No DHCP-related alerts were generated.

---

## Root Cause

The DHCP pool on R1-EDGE was configured for the wrong subnet (`10.10.20.0/24`) while the actual client LAN was `10.10.10.0/24`.

---

## Resolution

Corrected the DHCP pool configuration:

```cisco
ip dhcp pool LAN
 network 10.10.10.0 255.255.255.0
 default-router 10.10.10.1
```
## Verification

After remediation:

- PC1 successfully received a `10.10.10.x` address
- Full DORA process was observed in Wireshark
- Connectivity to the gateway and other clients was restored
- DHCP binding appeared on R1

![Successful DORA](evidence/screenshots/08-wireshark-successful-dora.png)

**Offer PCAP:**

![Offer Packet](evidence/screenshots/08a-wireshark-offer-packet-details.png)

**Request PCAP:**

![Request Packet](evidence/screenshots/08b-wireshark-request-packet-details.png)

**ACK PCAP:**

![ACK Packet](evidence/screenshots/08c-wireshark-ack-packet-details.png)

**Post Remediation DHCP Binding:**

![DHCP Binding After](evidence/screenshots/10-post-remediation-show-ip-dhcp-binding.png)

**ServiceNow Ticket (Resolved):**

![ServiceNow Resolved](evidence/screenshots/11-service-now-resolved.png)

---

## Lessons Learned

- Alway verify that the DHCP network matches the interface subnet providing the IP addresses.
- Packet Captures are useful for diagnosing DHCP related issues specifically inspecting DORA packets. 
- SNMP and SIEM platforms are not the best for troubleshooting a DHCP related issue such as a misconfiguration as they provide barely any alerts or logs. CLI and wireshark are sufficient. 
