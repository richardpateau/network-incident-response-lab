
# INC-003 — Unauthorized Traffic Blocked by ACL

**Status:** Resolved  
**Severity:** High (P2)  
**Device:** R1-EDGE  
**Change Related:** CHG-0231  
**Tools Used:** Wireshark / tcpdump, Splunk, Cisco IOS CLI
---

## ServiceNow Ticket

![ServiceNow Ticket](evidence/screenshots/01-servicenow-ticket.png)

**Summary:**  
After change CHG-0231 ("Block insecure Telnet access from LAN to APP-SERVER01"), users on `10.10.10.0/24` reported that APP-SERVER01 (`10.20.20.10`) was completely unreachable. Ping and HTTPS monitoring also failed.

**Expected:** Only Telnet (TCP/23) should be blocked.  
**Actual:** All traffic to the server was blocked.

---

## Topology

![Topology](topology/topology_draw_io.png)

---

## Expected State

- PC1 (`10.10.10.10`) can reach APP-SERVER01 (`10.20.20.10`)
- ICMP and HTTPS (TCP/443) should work
- Telnet (TCP/23) should be denied

---

## Detection
 
The ticket originated from user reports, but once opened, ACL deny
events were already visible in Splunk.

`%SEC-6-IPACCESSLOGP: list PROTECT-SERVER denied tcp 10.10.10.10(39247) -> 10.20.20.10(443), 1 packet `
 
![Splunk ACL Log](evidence/screenshots/05-splunk-acl-log.png)

---

## Initial Symptoms

- Ping from PC1 to `10.20.20.10` failed
- HTTPS from PC1 to `10.20.20.10` failed

![PC1 Ping Fail](evidence/screenshots/02-pc1-ping-fail.png)

![PC1 HTTPS Fail](evidence/screenshots/03-pc1-https-fail.png)

---

## Investigation Evidence

### 1. ACL Hit Counters

![ACL Hits Broken](evidence/screenshots/04-r1-show-access-lists-hits-broken.png)

Both deny statements were matching traffic, inidicating that more than one rule was contributing to the outage, not just a single misconfigured line.


### 2. Packet Captures (Broken State)

**ICMP:**

![ICMP Overview](evidence/screenshots/06-wireshark-icmp-overview.png)

**ICMP echo request details**

![ICMP Echo Request](evidence/screenshots/06a-wireshark-icmp-echo-request.png)

**ICMP destination unreachable packet**

![ICMP Destination Unreachable](evidence/screenshots/06b-wireshark-icmp-dest-unreachable.png)

**HTTPS SYN details:**

![HTTPS SYN](evidence/screenshots/07-wireshark-https-syn.png)

**tcpdump ICMP:**

![tcpdump ICMP](evidence/screenshots/08-tcpdump-icmp.png)

**tcpdump HTTPS:**

![tcpdump HTTPS](evidence/screenshots/09-tcpdump-https.png)

---

## Configs
 
- Before: [`r1-acl-before.txt`](evidence/configs/r1-acl-before.txt)
	```
	ip access-list extended PROTECT-SERVER
 	deny tcp any host 10.20.20.10 eq 443 log
 	deny ip 10.10.10.0 0.0.0.255 host 10.20.20.10 log
 	permit ip any any3
	```
- After Stage 1: [`r1-acl-after-fix1.txt`](evidence/configs/r1-acl-after-fix1.txt)
- After Stage 2 (final): [`r1-acl-after-fix2.txt`](evidence/configs/r1-acl-after-fix2.txt)

---

## Root Cause

The ACL deployed by CHG-0231 contained two problems:

1. **Rule blocking all protocols**  
   `deny ip 10.10.10.0 0.0.0.255 host 10.20.20.10`  
   blocked **all** traffic from the LAN to the server, not just Telnet.

2. **Stale leftover rule**  
   `deny tcp any host 10.20.20.10 eq 443`  
   continued to block HTTPS even after the first rule was corrected.

---

## Resolution

### Stage 1 — Fix the deny ip ACL

Replaced the `deny ip ...` statement with a Telnet-only deny:

```cisco
ip access-list extended PROTECT-SERVER
 no deny ip 10.10.10.0 0.0.0.255 host 10.20.20.10
 deny tcp 10.10.10.0 0.0.0.255 host 10.20.20.10 eq 23
```
![ACL Fix 1](evidence/screenshots/10-r1-acl-fix1-applied.png)

**Result after Stage 1:**

- Ping restored
- HTTPS still blocked

![Ping Success After Fix 1](evidence/screenshots/11-pc1-ping-success-after-fix1.png)

![HTTPS Still Fails](evidence/screenshots/12-pc1-https-still-fails.png)

### Stage 2 — Remove the stale HTTPS deny ACL 

```cisco
ip access-list extended PROTECT-SERVER
 no deny tcp any host 10.20.20.10 eq 443

Final ACL:

```cisco
ip access-list extended PROTECT-SERVER
 deny tcp 10.10.10.0 0.0.0.255 host 10.20.20.10 eq 23
 permit ip any any
```
## Verification

After Stage 2:

- HTTPS to `10.20.20.10` succeeded
- Telnet remained blocked (intended behavior)
- ICMP continued to work

**PC1 HTTPS connection successful:**

![HTTPS Success](evidence/screenshots/14-pc1-https-success.png)

**Telnet still blocked from PC1:**

![Telnet Still Blocked](evidence/screenshots/15-pc1-telnet-blocked-successful.png)

**Working packet captures:**

**Both ICMP request and replies present:**

![ICMP Echo Reply](evidence/screenshots/16-wireshark-icmp-echo-reply.png)

**ICMP reply details:**

![ICMP Reply Details](evidence/screenshots/16a-wireshark-reply-details.png)

**Successful TCP handshake:**

![HTTPS Handshake](evidence/screenshots/17-wireshark-https-full-handshake.png)

**Splunk after fix:**
`%SEC-6-IPACCESSLOGP: list PROTECT-SERVER denied tcp 10.10.10.10(36001) -> 10.20.20.10(23), 9 packets `
 
![Splunk After](evidence/screenshots/18-splunk-after.png)

**ServiceNow resolved ticket**

![ServiceNow Resolved](evidence/screenshots/19-servicenow-resolved.png)

---

## Lessons Learned

- ACL changes should match the exact protocol and port intended by the change ticket
- Broad `deny ip` statements are a common cause of unwanted protocols being blocked by the ACL. 
- Stale ACEs left from previous changes can hide behind a more obvious problem
- Hit counters (`show access-lists`) and packet captures are the fastest way to prove where traffic is being dropped
- Always re-test all expected services after an ACL change, not only the one named in the ticket
