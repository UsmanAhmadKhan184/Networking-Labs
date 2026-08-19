# Lab 01 — EIGRP Protocol (Real World Scenario)

## What I Built

A national telecommunications company network with one
headquarters in Islamabad, two data centers in Lahore and
Karachi acting as transit hubs, and three regional offices
in Lahore, Karachi and Peshawar. The entire network runs
EIGRP AS 100 as the Interior Gateway Protocol.

The topology has a deliberate backup link between DC-A and
DC-B with delay 500 — intentionally expensive — to demonstrate
metric manipulation and the Feasible Successor concept. KHI-R
connects to both data centers giving it dual paths and ECMP
behavior. LHR-R and PSH-R each connect to one data center
only, relying on the backup link as a secondary path through
the data center mesh.

---

## Topology

<img width="1430" height="809" alt="EIGRP Protocol lab-1" src="https://github.com/user-attachments/assets/f620e592-a2f4-4fc6-86f4-f16633dc94a0" /><br>

---

## What I Learned

### What EIGRP Is

EIGRP stands for Enhanced Interior Gateway Routing Protocol.
It was created by Cisco in 1993 as the successor to IGRP and
was made an open standard in 2013 (RFC 7868). It is classified
as an Advanced Distance Vector protocol — sometimes called a
Hybrid protocol — because it borrows concepts from both
distance vector and link state protocols.

Unlike RIP which sends its entire routing table every 30
seconds, EIGRP only sends partial updates when topology
changes occur and only to affected neighbors. This makes
it significantly more efficient in large networks.

### AS Number — Most Critical Concept

Every router in the same EIGRP domain must use the same
Autonomous System number. Unlike the OSPF process ID which
is locally significant only, the EIGRP AS number must match
on both sides of every neighbor relationship.

router eigrp 100 ← all routers in this lab use AS 100

If two routers have different AS numbers they will never
form a neighbor relationship — no error message appears,
the neighbor simply never shows up in the neighbor table.

### EIGRP Tables — Three Separate Tables

EIGRP maintains three tables simultaneously:

**Neighbor table** — lists all directly connected EIGRP
neighbors with their interface, hold time and uptime.

**Topology table** — contains ALL routes learned from ALL
neighbors including both Successors and Feasible Successors.
This is much more than what goes into the routing table and
is the key to EIGRP's fast convergence.

**Routing table** — contains only the best routes
(Successors). Marked with D in the routing table.

### DUAL Algorithm and Feasible Successor

EIGRP uses the Diffusing Update Algorithm (DUAL) to
guarantee loop-free paths at every moment. DUAL maintains
two types of routes for every destination:

**Successor** — the best path, installed in the routing
table. The router actively uses this path for forwarding.

**Feasible Successor** — a pre-calculated backup path
stored only in the topology table. When the primary path
fails EIGRP immediately promotes the Feasible Successor
without running any recalculation — convergence happens
in milliseconds.

**Feasibility Condition** — a route qualifies as a
Feasible Successor only if the neighbor's Reported
Distance is less than the current Successor's Feasible
Distance:

Neighbor's RD < Current Successor's FD

This condition mathematically guarantees the backup path
is loop-free. EIGRP will never use a path that does not
satisfy this condition — not even with variance configured.

### Metric Manipulation via Delay

EIGRP uses a composite metric calculated from bandwidth
and delay by default (K1=1, K3=1, all other K values = 0):

Metric = 256 × (10^7/Min.Bandwidth + Total.Delay)

The delay value is the primary tool for path manipulation
in EIGRP labs because changing bandwidth affects other
protocols like QoS while delay only affects the EIGRP
metric calculation.

In this lab delay 10 was set on all primary links and
delay 500 was set on the DC-A to DC-B backup link:

interface Ethernet0/0 ← backup link DC-A to DC-B
delay 500 ← high delay = high metric = less preferred

This made the backup link's metric approximately 26 times
higher than the primary paths — ensuring EIGRP always
prefers the primary paths through HQ.

### Feasibility Condition Discovery — Lab Finding

After configuring variance 2 and examining the topology
tables on LHR-R and PSH-R a critical finding emerged.
Neither router showed any Feasible Successors despite
having a backup path available through the DC-A to DC-B
link:

LHR-R topology table:

P 192.168.3.0/24, 1 successors, FD is 414720 \
via 10.0.24.1 (414720/412160) ← Successor only \
no Feasible Successor

The reason is that the delay 500 on the backup link makes
the backup path's Reported Distance higher than LHR-R's
current Feasible Distance — violating the Feasibility
Condition. EIGRP mathematically cannot use it as a backup
without risking a routing loop.

The fix would be to reduce the backup link delay to
something more proportional — around delay 50 instead of
delay 500. With delay 50 the ratio becomes approximately
3.5x instead of 26x, making the Feasibility Condition
satisfiable and allowing variance to work.

This was the most valuable lesson from this lab — variance
alone is not enough. The Feasibility Condition must be
satisfied first. Variance only controls the acceptable
metric range — it cannot override the loop-free guarantee.

### ECMP on KHI-R — Unexpected Finding

KHI-R connects to both DC-A and DC-B with equal delay 10
links. When checking its topology table for the backup
link network 10.0.23.0/30, something interesting appeared:

P 10.0.23.0/30, 2 successors, FD is 386560 \
via 10.0.25.1 (386560/384000), Ethernet0/3 \
via 10.0.35.1 (386560/384000), Ethernet1/0


Two successors with identical metrics — EIGRP automatically
installed both paths as equal cost successors and performs
ECMP load balancing without any variance configuration
needed. This is because KHI-R has two paths to the backup
link with exactly equal total delay — one via DC-A and one
via DC-B.

### Unequal Cost Load Balancing — Variance

EIGRP is the only IGP that supports unequal cost load
balancing. The variance command allows EIGRP to use
multiple paths with different metrics simultaneously:

`router eigrp 100` \
`variance 2`


Variance N means use any path whose metric is within
N times the best path metric. Traffic is distributed
proportionally — the better path carries more traffic.

In this lab variance was configured but had no visible
effect on LHR-R and PSH-R because no Feasible Successors
existed due to the delay 500 issue described above.
KHI-R did not need variance because its two paths were
already equal cost (ECMP).

The real world value of variance is in networks where
a 100 Mbps primary link and a 50 Mbps secondary link
both connect the same two routers. Instead of leaving
the 50 Mbps link idle, variance allows both links to
carry traffic proportionally — the primary carries 67%
and the secondary carries 33%.

Note: variance effects are only visible with continuous
real traffic flows. In a lab environment with ping only
the routing table and topology table show the configured
behavior but actual packet distribution cannot be
observed without traffic generation tools.


### Passive Interface

LAN-facing interfaces on regional office routers were
configured as passive — stopping EIGRP Hello packets
from being sent toward switches and PCs while still
advertising the LAN network into EIGRP:

`router eigrp 100` \
`passive-interface default` \
`no passive-interface Ethernet0/2` ← WAN facing only


### no auto-summary

EIGRP summarizes at classful boundaries by default just
like RIPv2. This was disabled on every router:

`router eigrp 100` \
`no auto-summary`


Without this a /30 WAN link in the 10.0.0.0 range would
be advertised as 10.0.0.0/8 — causing routing confusion
exactly as seen in the RIP labs.

---

## IP Address Table

### Router-to-Router Links

| Link | Network | Router A | Interface | IP | Router B | Interface | IP | Delay |
|---|---|---|---|---|---|---|---|---|
| HQ — DC-A | 10.0.12.0/30 | HQ-R | e0/1 | 10.0.12.1 | DC-A | e0/1 | 10.0.12.2 | 10 |
| HQ — DC-B | 10.0.13.0/30 | HQ-R | e0/2 | 10.0.13.1 | DC-B | e0/2 | 10.0.13.2 | 10 |
| DC-A — DC-B | 10.0.23.0/30 | DC-A | e0/0 | 10.0.23.1 | DC-B | e0/0 | 10.0.23.2 | 500 backup |
| DC-A — LHR-R | 10.0.24.0/30 | DC-A | e0/2 | 10.0.24.1 | LHR-R | e0/2 | 10.0.24.2 | 10 |
| DC-A — KHI-R | 10.0.25.0/30 | DC-A | e0/3 | 10.0.25.1 | KHI-R | e0/3 | 10.0.25.2 | 10 |
| DC-B — KHI-R | 10.0.35.0/30 | DC-B | e1/0 | 10.0.35.1 | KHI-R | e1/0 | 10.0.35.2 | 10 |
| DC-B — PSH-R | 10.0.36.0/30 | DC-B | e0/1 | 10.0.36.1 | PSH-R | e0/1 | 10.0.36.2 | 10 |

### LAN Networks

| Device | Interface | IP Address | Subnet | Default Gateway |
|---|---|---|---|---|
| LHR-R | e0/0 | 192.168.1.100 | 255.255.255.0 | — |
| PC1 | eth0 | 192.168.1.1 | 255.255.255.0 | 192.168.1.100 |
| PC2 | eth0 | 192.168.1.2 | 255.255.255.0 | 192.168.1.100 |
| KHI-R | e0/0 | 192.168.2.100 | 255.255.255.0 | — |
| PC3 | eth0 | 192.168.2.1 | 255.255.255.0 | 192.168.2.100 |
| PC4 | eth0 | 192.168.2.2 | 255.255.255.0 | 192.168.2.100 |
| PSH-R | e0/0 | 192.168.3.100 | 255.255.255.0 | — |
| PC5 | eth0 | 192.168.3.1 | 255.255.255.0 | 192.168.3.100 |
| PC6 | eth0 | 192.168.3.2 | 255.255.255.0 | 192.168.3.100 |

### Loopback Interfaces

| Router | Interface | IP Address | Purpose |
|---|---|---|---|
| HQ-R | Loopback0 | 1.1.1.1/24 | Router identity |
| DC-A | Loopback0 | 2.2.2.2/24 | Router identity |
| DC-B | Loopback0 | 3.3.3.3/24 | Router identity |
| LHR-R | Loopback0 | 4.4.4.4/24 | Router identity |
| KHI-R | Loopback0 | 5.5.5.5/24 | Router identity |
| PSH-R | Loopback0 | 6.6.6.6/24 | Router identity |

---

## Verification

### EIGRP Neighbor Tables

<img width="1649" height="257" alt="Neighbor HQ-R" src="https://github.com/user-attachments/assets/3c1f8950-b3a1-4822-9309-da01c1b48460" /><br>
> HQ-R neighbor table showing adjacency with DC-A and DC-B
> — both formed successfully with uptime and hold timers

<img width="1626" height="327" alt="Neighbor DC-A Router" src="https://github.com/user-attachments/assets/e088344f-8256-40df-bd3a-e7692b2d2533" /><br>
> DC-A neighbor table showing all four neighbors —
> HQ-R, DC-B, LHR-R and KHI-R

### EIGRP Topology Tables

<img width="916" height="613" alt="ECMP Via DC-A and DC-B" src="https://github.com/user-attachments/assets/ef8e234b-0fb7-43b3-83de-54f761479cc6" /><br>
> KHI-R topology table showing ECMP on 10.0.23.0/30 with
> 2 successors via both DC-A and DC-B at equal metric 386560

<img width="577" height="617" alt="LHR-R Topology" src="https://github.com/user-attachments/assets/245c71a6-9a59-40aa-852d-a7e1edee1233" /><br>
> LHR-R topology table — only 1 successor per route
> No Feasible Successors due to delay 500 violating
> the Feasibility Condition from LHR-R perspective

<img width="616" height="622" alt="PSH-R Topology" src="https://github.com/user-attachments/assets/72ef3654-5e03-4cfc-8ada-eaeb4735a237" /><br>
> PSH-R topology table — same situation as LHR-R
> All routes via DC-B only, no Feasible Successors

### Routing Tables

<img width="960" height="564" alt="HD-Q Routes" src="https://github.com/user-attachments/assets/48250751-5049-45ef-8d4c-e0984f0f0893" /><br>
> HQ-R routing table showing D routes to all three
> regional office LANs and loopbacks

<img width="1277" height="603" alt="LHR-R Routes to other offices" src="https://github.com/user-attachments/assets/e4260c9c-44fa-4baf-8af7-03785f64d736" /><br>
> LHR-R routing table showing all remote networks
> learned via DC-A as the single exit point

### Metric Manipulation

<img width="1471" height="643" alt="Backup link delay" src="https://github.com/user-attachments/assets/3e369dc2-a486-4c65-8b89-2229f28751ed" /><br>
> show interface on DC-A to DC-B backup link confirming
> delay 500 configured — creating 26x metric difference
> vs primary path delay 10

### Connectivity

<img width="1267" height="350" alt="Ping PC1-PC5" src="https://github.com/user-attachments/assets/4291a28b-58fa-464f-a762-d3087995c3e9" /><br>
> Ping from PC1 (Lahore) to PC5 (Peshawar) — full end
> to end connectivity confirmed

<img width="1695" height="372" alt="TraceRoute PC1-PC5" src="https://github.com/user-attachments/assets/8281c3f5-998f-4b5b-9f11-20014a9d0982" /><br>
> Traceroute confirming path: PC1 → LHR-R → DC-A →
> HQ-R → DC-B → PSH-R → PC6

<img width="1213" height="343" alt="Ping PC1-HQ-R" src="https://github.com/user-attachments/assets/46b556d3-a1c1-4cd5-924e-80ac3d42594a" /><br>
> Ping from PC1 to HQ-R loopback 1.1.1.1 confirming
> loopback reachability via EIGRP advertisement

---

## Router Configs

| Router | Role | Config |
|---|---|---|
| HQ-R | Headquarters hub | [HQ-R.txt](HQ-R.txt) |
| DC-A | Data center A — Lahore side | [DC-A.txt](DC-A.txt) |
| DC-B | Data center B — Karachi side | [DC-B.txt](DC-B.txt) |
| LHR-R | Regional office Lahore | [LHR-R.txt](LHR-R.txt) |
| KHI-R | Regional office Karachi — dual exit | [KHI-R.txt](KHI-R.txt) |
| PSH-R | Regional office Peshawar | [PSH-R.txt](PSH-R.txt) |
