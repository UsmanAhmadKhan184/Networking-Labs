# Lab 01 — RIPv2 (Real World Scenario)

## What I Built

A logistics company network representing a real world scenario
where RIPv2 was configured years ago and never upgraded to OSPF.
The network has one distribution center in Islamabad, two transit
routers handling backbone traffic, and four warehouse branches
in Lahore, Multan, Karachi and Quetta — each with two PCs.

A backup link between TRANSIT-A and TRANSIT-B provides
redundancy. However since RIP uses hop count as its only metric,
traffic was initially taking the backup link instead of going
through DC-Router because it had fewer hops. This was fixed
using the offset-list command to artificially increase the hop
count on the backup link — forcing traffic through DC-Router
as the primary path while keeping the backup link for failover.

---

## Topology

<img width="1616" height="859" alt="RIP-lab-1" src="https://github.com/user-attachments/assets/c55c6e7d-1121-4610-8786-c17b5011a669" /><br>

---

## What I Learned

### What RIP Is

RIP stands for Routing Information Protocol — one of the oldest
dynamic routing protocols still in production use. It is a
distance vector protocol, meaning each router only knows about
its directly connected neighbors and learns about distant
networks through those neighbors one hop at a time. Unlike OSPF
which builds a complete topology map, RIP routers only know
what their neighbors tell them.

RIP uses hop count as its only metric. Every router a packet
passes through counts as one hop. The maximum is 15 — a hop
count of 16 means unreachable. This is RIP's biggest
limitation: it does not consider bandwidth, delay, or link
quality at all. A path through three slow 1Mbps links beats
a path through four fast 1Gbps links in RIP's view, which
is clearly wrong for real traffic engineering.

### RIP Versions

RIPv1 is the original — classful only, no subnet mask in
updates, no authentication, uses broadcast. RIPv2 is the
modern version — carries subnet masks in every update,
supports VLSM, uses multicast 224.0.0.9, and supports
MD5 authentication. Always use RIPv2 in real networks.

### How RIP Works

**Step 1 — Full table updates every 30 seconds**

Every 30 seconds each RIP router sends its complete routing
table to all neighbors. There is no neighbor relationship
concept like in OSPF — RIP simply multicasts its table and
whoever is listening receives it.

**Step 2 — Bellman-Ford algorithm**

Each router receives its neighbor's table and adds 1 to every
hop count, then compares with what it already knows. If the
received route has a lower hop count it updates its table:

R2 tells R1: "I can reach 10.10.3.0 in 1 hop" \
R1 adds 1:   "I can reach 10.10.3.0 in 2 hops via R2" \
R1 updates if this is better than what it already has

**Step 3 — Convergence**

This continues until all routers have consistent tables.
RIP converges much slower than OSPF — it can take several
minutes in a large network. This slow convergence is RIP's
second biggest limitation.

### RIP Timers

| Timer | Default | Purpose |
|---|---|---|
| Update | 30 seconds | How often full table is sent to neighbors |
| Invalid | 180 seconds | Route marked invalid if no update received |
| Holddown | 180 seconds | Prevents accepting worse routes after failure |
| Flush | 240 seconds | Route completely removed from table |

A link failure can take up to 4 minutes before all routers
fully remove the bad route. OSPF converges in seconds by
comparison.

### RIP Loop Prevention Mechanisms

**Split Horizon** — a router never advertises a route back
to the neighbor it learned that route from. If TRANSIT-A
learned about WH3's network from DC-Router, it will not
send that route back to DC-Router.

**Route Poisoning** — when a network goes down the router
immediately advertises it with hop count 16 (infinity) to
tell all neighbors it is unreachable rather than waiting
for the timer to expire.

**Poison Reverse** — when a router receives a poisoned
route it sends the poison back to confirm the route is
dead. This overrides split horizon in that specific case.

**Holddown Timer** — after a route goes bad the router
ignores any updates claiming that route is reachable for
180 seconds. This prevents accepting false information
during network instability.

### Auto-Summary and no auto-summary

RIPv2 automatically reads the subnet mask from each
interface and includes it in every update — you never
need to tell RIP the mask separately. However by default
RIPv2 still summarizes routes at classful boundaries
(auto-summary). This means a /30 link in the 192.168.10.0
range could get advertised as 192.168.10.0/24 to neighbors,
causing routing confusion.

The `no auto-summary` command disables this and forces RIP
to advertise exact subnets. Always configure this in RIPv2.

Verification that it is off:
show ip protocols
Automatic network summarization is not in effect

### The Hop Count Problem — Discovered in This Lab

The backup link between TRANSIT-A and TRANSIT-B was
intended only for failover. But RIP chose it as the
primary path because:

Path via DC-Router:   WH1 → TRANSIT-A → DC → TRANSIT-B → WH3 \
Hop count: 3

Path via backup link: WH1 → TRANSIT-A → TRANSIT-B → WH3 \
Hop count: 2

RIP always picks lowest hop count regardless of what the
link is intended for. This demonstrated the core weakness
of RIP — no awareness of link purpose, bandwidth, or
administrative intent.

### Fix — offset-list Command

The offset-list command artificially adds hop count to
routes received on a specific interface, making RIP
prefer another path:

On TRANSIT-A:
router rip \
offset-list 0 in 2 Ethernet0/0

On TRANSIT-B:
router rip \
offset-list 0 in 2 Ethernet0/1

This added 2 hops to all routes received on the backup
link, making the DC-Router path (3 hops) cheaper than
the backup path (1+2=3... wait, now equal). Actually
adding offset 2 makes backup = 3 hops vs DC-Router = 3.
RIP broke the tie by preferring the already-installed
route via DC-Router. In practice the offset-list
resolved the issue and traffic returned to DC-Router.

The backup link still activates automatically when
DC-Router goes down — proving failover still works.

### RIP vs OSPF — Key Takeaway

This lab made the OSPF cost concept make much more sense
in retrospect. OSPF cost is based on bandwidth so a
1Gbps link always beats a 10Mbps link regardless of hop
count. RIP has no such intelligence — it counts routers,
not bandwidth. For any network larger than a small office,
OSPF or EIGRP is the correct choice.

---

## IP Address Table

### Router-to-Router Links

| Link | Network | Router A | Interface | IP Address | Router B | Interface | IP Address |
|---|---|---|---|---|---|---|---|
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

| Router | Hostname | Interface | IP Address |
|---|---|---|---|
| DC-Router | R1 | Loopback0 | 1.1.1.1/32 |
| TRANSIT-A | R2 | Loopback0 | 2.2.2.2/32 |
| TRANSIT-B | R3 | Loopback0 | 3.3.3.3/32 |
| WH1 | R4 | Loopback0 | 4.4.4.4/32 |
| WH2 | R5 | Loopback0 | 5.5.5.5/32 |
| WH3 | R6 | Loopback0 | 6.6.6.6/32 |
| WH4 | R7 | Loopback0 | 7.7.7.7/32 |

---

## Verification

### Routing Tables

<img width="1210" height="707" alt="DC-Router IP Route" src="https://github.com/user-attachments/assets/545f759a-4a67-4a0b-bc32-01fa496b488e" /><br>
> DC-Router routing table showing all four warehouse LANs
> learned via RIP with correct hop counts

<img width="1178" height="692" alt="WH1 IP Route" src="https://github.com/user-attachments/assets/4ae636e3-8898-4c57-a2c6-fd64ebd1272b" /><br>
> WH1 routing table showing all other networks reachable
> through TRANSIT-A and DC-Router

<img width="939" height="621" alt="IP protocols on WH1" src="https://github.com/user-attachments/assets/5a2ab6c4-8c23-4b70-aba8-c32956ae51b1" /><br>
> show ip protocols on WH1 confirming RIPv2, no auto-summary,
> passive interface active, and 30 second update timer

### RIP Database

<img width="881" height="623" alt="IP RIP Database" src="https://github.com/user-attachments/assets/9706b138-ed35-40d8-b432-808f4d9301c8" /><br>
<img width="893" height="500" alt="IP RIP Database-2" src="https://github.com/user-attachments/assets/d8c907c1-30df-4f1e-bdba-b013831a2ad0" /><br>
> Full RIP database on DC-Router showing all networks with
> exact subnet masks — confirming no auto-summary is working

### Connectivity Tests

<img width="1247" height="283" alt="Ping Vpc1 to Vpc8" src="https://github.com/user-attachments/assets/de8dab6f-1db5-43b4-89b2-54ba7eadea0d" /><br>
> Ping from VPC1 (Lahore) to VPC8 (Quetta) — full end to
> end connectivity across five router hops confirmed

<img width="1669" height="378" alt="Trace Vpc1 to Vpc8" src="https://github.com/user-attachments/assets/d98a4d3a-5547-472e-a971-4f1e4bad8b07" /><br>
> Traceroute confirming path goes through DC-Router as
> primary after offset-list was applied

<img width="1218" height="275" alt="Ping Vpc1 to DC-Router" src="https://github.com/user-attachments/assets/9387564b-351a-4904-b83e-870b796b9948" /><br>
> Ping from VPC1 to DC-Router loopback 1.1.1.1 confirming
> loopback reachability via RIP advertisement

### Failover Test

<img width="1253" height="611" alt="IP Route Transit-A BF" src="https://github.com/user-attachments/assets/f149e8dd-983d-4589-abe5-b4d0346eae1e" /><br>
> TRANSIT-A routing table before failover — WH3 and WH4
> routes going via 192.168.10.1 (DC-Router) as primary path

> DC-Router link to TRANSIT-A was then shut down

<img width="1247" height="510" alt="IP Route Transit-A AF" src="https://github.com/user-attachments/assets/f1110b92-19c2-47b7-a6d5-5e86f670b926" /><br>
> After DC-Router link went down — WH3 and WH4 routes
> automatically switched to backup link 192.168.99.2
> RIP reconverged and found alternate path automatically

<img width="1688" height="334" alt="Trace Vpc1 to Vpc8 AF" src="https://github.com/user-attachments/assets/a7872470-e660-4099-b86c-e2d5eb619a6a" /><br>
> Traceroute during failover confirming traffic rerouted
> through backup link between TRANSIT-A and TRANSIT-B

<img width="1191" height="624" alt="IP Route Transit-A AR" src="https://github.com/user-attachments/assets/e71c3d48-dc50-4b2f-96f3-13af8757a258" /><br>
> After DC-Router link was restored — routes returned to
> primary path via DC-Router automatically

---

## Router Configs

| Router | Hostname | Role | Config |
|---|---|---|---|
| DC-Router | R1 | Distribution center hub | [R1.txt](R1.txt) |
| TRANSIT-A | R2 | Transit — Lahore/Multan side | [R2.txt](R2.txt) |
| TRANSIT-B | R3 | Transit — Karachi/Quetta side | [R3.txt](R3.txt) |
| WH1 | R4 | Warehouse Lahore — passive LAN | [R4.txt](R4.txt) |
| WH2 | R5 | Warehouse Multan — passive LAN | [R5.txt](R5.txt) |
| WH3 | R6 | Warehouse Karachi — passive LAN | [R6.txt](R6.txt) |
| WH4 | R7 | Warehouse Quetta — passive LAN | [R7.txt](R7.txt) |
