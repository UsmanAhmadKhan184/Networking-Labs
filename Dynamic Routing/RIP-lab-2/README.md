# Lab 02 — RIPv2 Advanced Concepts

## What I Built

Same topology as Lab 1 with one addition — an ISP router at
the top simulating internet connectivity. Four new RIPv2
concepts were layered on top of the existing configuration:
MD5 authentication on every router-to-router link, passive
interface default for cleaner and safer configuration, default
route injection from DC-Router toward the ISP, and timer
manipulation for faster convergence.

The lab also uncovered a real behavior of RIP — when
authentication fails, routes do not disappear immediately.
They stay in the routing table until the invalid timer expires,
which taught me how RIP timers and stale routes interact in
a real failure scenario.

---

## Topology

<img width="1669" height="941" alt="RIP-Lab-2" src="https://github.com/user-attachments/assets/199e15ba-fa6e-40a6-8451-83699433bea8" /><br>

---

## What I Learned

### MD5 Authentication

Without authentication any device that connects to the network
and runs RIP can inject fake routes and redirect traffic. MD5
authentication ensures routers only accept RIP updates from
neighbors that share the same key. If the keys do not match
the update is silently dropped — no error message appears,
routes simply stop being accepted.

Authentication in RIPv2 is configured in two places —
the key chain is created in global config mode and then
applied per interface in interface config mode. It is never
configured inside the `router rip` block.

! Step 1 — Create key chain (global config mode) \
key chain RIP-AUTH \
key 1 \
key-string LOG@1234

! Step 2 — Apply to every WAN-facing interface \
interface Ethernet0/1 \
ip rip authentication mode md5 \
ip rip authentication key-chain RIP-AUTH

Both sides of every link must have identical key-string
configuration. Even one character difference causes silent
authentication failure — routes disappear after the invalid
timer expires.

**Important — do NOT apply authentication on LAN interfaces
facing switches and PCs.** Those interfaces are passive and
carry no RIP updates, so authentication there is pointless.
Apply it only on router-to-router links.

### Passive Interface Default

In Lab 1 passive-interface was configured one by one on each
LAN interface. The professional approach is the opposite —
make all interfaces passive by default then selectively enable
RIP only on interfaces that actually need it.

router rip \
passive-interface default         ← all interfaces passive \
no passive-interface Ethernet0/1  ← enable RIP here only \
no passive-interface Ethernet0/2  ← and here

This is safer because forgetting to passive a single interface
in Lab 1 style means RIP updates leak to end devices. With
passive-interface default, forgetting to add a no passive
command simply means that interface stays silent — a much
safer failure mode. A potential attacker connected to any
LAN port cannot receive or inject RIP updates.

### Default Route Injection

In Lab 1 every router only knew about networks inside the
company. With a real internet connection at DC-Router, you
want all branches to automatically forward unknown traffic
toward the ISP without configuring anything on branch routers.

! On DC-Router only: \
ip route 0.0.0.0 0.0.0.0 203.0.113.1   ← static toward ISP \
router rip \
default-information originate           ← inject into RIP

Every other router automatically receives: \
R* 0.0.0.0/0 [120/1] via DC-Router

The R* means RIP-learned default route. Any packet with no
specific route match gets forwarded to DC-Router which sends
it to the ISP. The ISP router was given a loopback of 8.8.8.8
to simulate an internet address, and static routes back to
internal networks so replies could return to the PCs.

### Timer Manipulation

Default RIP timers mean a link failure can take up to 4
minutes before all routers remove the bad route. Reducing
timers speeds up convergence and makes testing much faster.

router rip \
timers basic 10 60 60 90

| Timer | Default | Lab Setting |
|---|---|---|
| Update | 30 sec | 10 sec |
| Invalid | 60 sec | 60 sec |
| Holddown | 180 sec | 60 sec |
| Flush | 240 sec | 90 sec |

Critical rule — timers must be identical on every router.
Mismatched timers cause route flapping and unpredictable
convergence. Always verify with `show ip protocols` on every
router after changing timers.

### Problem I Encountered — Stale Routes After Auth Failure

When I changed WH1's key-string to `@1234` (while all other
routers had `LOG@1234`), I expected WH1's routes to disappear
from other routers immediately. They did not — they stayed
visible for about 2 minutes after the key change.

After investigation I understood why. RIP has no real-time
neighbor relationship like OSPF. When TRANSIT-A started
rejecting WH1's updates due to wrong authentication, it still
had the old routes from WH1 in its table from before the key
change. Those stale routes remained until the invalid timer
(60 seconds) marked them as invalid, then the holddown timer
(60 seconds) expired, and finally the flush timer (90 seconds)
removed them completely.

The debug output confirmed authentication was failing: \
RIP: ignored v2 packet from 192.168.11.1 (invalid authentication)

But routes only disappeared after approximately 2 minutes —
the combined effect of all three timers. This is a fundamental
difference between RIP and OSPF. OSPF would detect the failure
within seconds via Hello packets. RIP only discovers the
problem when updates stop arriving and timers expire.

After fixing the key-string back to `LOG@1234` on WH1, routes
returned within 10 seconds — the update timer — proving
recovery is automatic and fast once the authentication is
corrected.

---

## IP Address Table

### Router-to-Router Links

| Link | Network | Router A | Interface | IP Address | Router B | Interface | IP Address |
|---|---|---|---|---|---|---|---|
| ISP-R — DC-Router | 203.0.113.0/30 | ISP-R | e0/1 | 203.0.113.2 | DC-R (R1) | e0/1 | 203.0.113.1 |
| DC — TRANSIT-A | 192.168.10.0/30 | DC-R (R1) | e0/3 | 192.168.10.1 | TRANSIT-A (R2) | e0/3 | 192.168.10.2 |
| DC — TRANSIT-B | 192.168.20.0/30 | DC-R (R1) | e0/0 | 192.168.20.1 | TRANSIT-B (R3) | e0/0 | 192.168.20.2 |
| TRANSIT-A — TRANSIT-B | 192.168.99.0/30 | TRANSIT-A (R2) | e0/0 | 192.168.99.1 | TRANSIT-B (R3) | e0/3 | 192.168.99.2 |
| TRANSIT-A — WH1 | 192.168.11.0/30 | TRANSIT-A (R2) | e0/1 | 192.168.11.1 | WH1 (R4) | e0/1 | 192.168.11.2 |
| TRANSIT-A — WH2 | 192.168.12.0/30 | TRANSIT-A (R2) | e0/2 | 192.168.12.1 | WH2 (R5) | e0/2 | 192.168.12.2 |
| TRANSIT-B — WH3 | 192.168.21.0/30 | TRANSIT-B (R3) | e0/1 | 192.168.21.1 | WH3 (R6) | e0/1 | 192.168.21.2 |
| TRANSIT-B — WH4 | 192.168.22.0/30 | TRANSIT-B (R3) | e0/2 | 192.168.22.1 | WH4 (R7) | e0/2 | 192.168.22.2 |

### LAN Networks

| Device | Interface | IP Address | Subnet | Default Gateway |
|---|---|---|---|---|
| WH1 (R4) | e0/0 | 10.10.1.100 | 255.255.255.0 | — |
| VPC1 | eth0 | 10.10.1.1 | 255.255.255.0 | 10.10.1.100 |
| VPC2 | eth0 | 10.10.1.2 | 255.255.255.0 | 10.10.1.100 |
| WH2 (R5) | e0/0 | 10.10.2.100 | 255.255.255.0 | — |
| VPC3 | eth0 | 10.10.2.1 | 255.255.255.0 | 10.10.2.100 |
| VPC4 | eth0 | 10.10.2.2 | 255.255.255.0 | 10.10.2.100 |
| WH3 (R6) | e0/0 | 10.10.3.100 | 255.255.255.0 | — |
| VPC5 | eth0 | 10.10.3.1 | 255.255.255.0 | 10.10.3.100 |
| VPC6 | eth0 | 10.10.3.2 | 255.255.255.0 | 10.10.3.100 |
| WH4 (R7) | e0/0 | 10.10.4.100 | 255.255.255.0 | — |
| VPC7 | eth0 | 10.10.4.1 | 255.255.255.0 | 10.10.4.100 |
| VPC8 | eth0 | 10.10.4.2 | 255.255.255.0 | 10.10.4.100 |

### Loopback Interfaces

| Router | Hostname | Interface | IP Address | Purpose |
|---|---|---|---|---|
| ISP-Router | ISP-R | Loopback0 | 8.8.8.8/32 | Simulated internet address |
| DC-Router | R1 | Loopback0 | 1.1.1.1/32 | Router identity |
| TRANSIT-A | R2 | Loopback0 | 2.2.2.2/32 | Router identity |
| TRANSIT-B | R3 | Loopback0 | 3.3.3.3/32 | Router identity |
| WH1 | R4 | Loopback0 | 4.4.4.4/32 | Router identity |
| WH2 | R5 | Loopback0 | 5.5.5.5/32 | Router identity |
| WH3 | R6 | Loopback0 | 6.6.6.6/32 | Router identity |
| WH4 | R7 | Loopback0 | 7.7.7.7/32 | Router identity |

---

## Verification

### Authentication Verification

<img width="1472" height="288" alt="Authentication key on DC-Router" src="https://github.com/user-attachments/assets/55ddccbe-0ffb-4b78-86fc-8cf823bbb12d" />
> show key chain on DC-Router confirming LOG@1234 is the
> active key — same key must exist on every router

<img width="1053" height="306" alt="Key Change on WH1" src="https://github.com/user-attachments/assets/bf0e993d-90f3-42ec-bad5-1de3d8c371d9" /><br>
> WH1 key-string changed to wrong value @1234 to simulate
> authentication failure

<img width="1238" height="367" alt="Authentication failure on WH1" src="https://github.com/user-attachments/assets/e6759e13-a68d-4dd6-9176-ecbc7bea712d" /><br>
> debug ip rip output on WH1 showing
> "ignored v2 packet — invalid authentication"
> Both sides reject each other's updates every 10 seconds

<img width="1429" height="688" alt="Authentication Passed on WH1" src="https://github.com/user-attachments/assets/d0b3ae88-54af-401f-8df0-c347e37f4393" /><br>
> Key-string corrected back to LOG@1234 on WH1 — routes
> returned to all routers within 10 seconds (update timer)

### Passive Interface Default

<img width="1032" height="552" alt="IP Protocol Passive int" src="https://github.com/user-attachments/assets/6be2c819-d99f-4546-acc5-ae6a21ac069c" /><br>
> show ip protocols confirming passive-interface default is
> active and only WAN-facing interfaces have RIP enabled

### Timer Manipulation

<img width="888" height="629" alt="Timer Manipulation" src="https://github.com/user-attachments/assets/5b3edf46-41c1-4f55-82bb-45c10d2d863a" /><br>
> show ip protocols confirming timers set to 10/60/60/90
> on all routers — faster convergence for lab testing

### Default Route Injection

<img width="1052" height="695" alt="IP Route on WH1" src="https://github.com/user-attachments/assets/95a67e76-ad44-4f6b-be47-ba0667fbd7e6" /><br>
> R* 0.0.0.0/0 visible on WH1 — injected by DC-Router via
> default-information originate and learned via RIP

<img width="1253" height="334" alt="Ping PC1 to ISP-R" src="https://github.com/user-attachments/assets/52934b10-bf39-4a21-9442-3712e98ef844" /><br>
> Ping from PC1 to ISP-Router loopback 8.8.8.8 — confirms
> default route is working end to end through DC-Router

<img width="1687" height="358" alt="Trace to Google" src="https://github.com/user-attachments/assets/d387da44-f88a-46b9-b8a3-c1a9fa7ae592" /><br>
> Traceroute from PC1 to 8.8.8.8 showing full path through
> WH1 → TRANSIT-A → DC-Router → ISP-Router

### General Connectivity

<img width="1092" height="663" alt="IP Route on DC-Router" src="https://github.com/user-attachments/assets/8694acb0-3b69-4edf-82fc-516a06fa372c" /><br>
> DC-Router routing table showing all warehouse LANs and
> loopbacks learned via RIP

<img width="783" height="605" alt="IP RIP Database" src="https://github.com/user-attachments/assets/16a92f31-3d33-4031-b70b-ca4cef95ce67" /><br>
<img width="880" height="509" alt="IP RIP Database-2" src="https://github.com/user-attachments/assets/16213fc8-8fb2-411c-af1d-1fcd5319cfe7" /><br>
> Complete RIP database on DC-Router showing all networks
> with exact subnet masks confirming no auto-summary

<img width="1241" height="350" alt="Ping PC1 to PC6" src="https://github.com/user-attachments/assets/a319ef50-2ce7-4737-9e0e-d900ce06cd41" /><br>
> Ping from VPC1 (Lahore) to VPC6 (Karachi) confirming
> full end to end connectivity

<img width="1655" height="381" alt="Trace PC1 to PC6" src="https://github.com/user-attachments/assets/646d0df9-131a-423b-8e16-905ff630f865" /><br>
> Traceroute from VPC1 to VPC6 confirming traffic flows
> through DC-Router as primary path

---

## Router Configs

| Router | Hostname | Role | Config |
|---|---|---|---|
| ISP-Router | ISP-R | Internet simulation — no RIP | [ISP-R.txt](ISP-R.txt) |
| DC-Router | R1 | Hub — default-information originate | [R1.txt](R1.txt) |
| TRANSIT-A | R2 | Transit — Lahore/Multan side | [R2.txt](R2.txt) |
| TRANSIT-B | R3 | Transit — Karachi/Quetta side | [R3.txt](R3.txt) |
| WH1 | R4 | Warehouse Lahore — passive default | [R4.txt](R4.txt) |
| WH2 | R5 | Warehouse Multan — passive default | [R5.txt](R5.txt) |
| WH3 | R6 | Warehouse Karachi — passive default | [R6.txt](R6.txt) |
| WH4 | R7 | Warehouse Quetta — passive default | [R7.txt](R7.txt) |
