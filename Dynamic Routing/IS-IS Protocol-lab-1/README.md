# Lab 01 — IS-IS Protocol (Multi-Area)

## What I Built

A multi-area IS-IS network with three areas connected through
a dedicated L2 backbone. Each area contains L1 routers handling
intra-area routing and L1/L2 border routers connecting to the
backbone. Three pure L2 backbone routers form the horizontal
spine that carries inter-area traffic between all three areas.

Unlike OSPF which requires a mandatory Area 0, IS-IS forms its
backbone automatically through L2 adjacencies between L1/L2
and pure L2 routers — no specific area number is required for
the backbone.

---

## Topology

<img width="1695" height="865" alt="IS-IS Protocol lab-1" src="https://github.com/user-attachments/assets/1d50f918-cf0e-4a2d-927d-ff980a434165" /><br>

---

## What I Learned

### What IS-IS Is

IS-IS stands for Intermediate System to Intermediate System —
a link state routing protocol in the same family as OSPF. Every
router builds a complete map of the network and independently
calculates the shortest path using SPF algorithm. The term
Intermediate System comes from the OSI model where it means
router — so IS-IS literally means router to router protocol.

IS-IS was originally designed for the OSI network layer, not
TCP/IP. It was later extended to carry IP routes through a
standard called Integrated IS-IS (also called Dual IS-IS). It
is an open standard (ISO 10589) supported on Cisco, Juniper,
Huawei and every major vendor.

### Where IS-IS Is Used

IS-IS is rarely found in enterprise networks — OSPF dominates
there. IS-IS is the protocol of choice for:

- Service provider (ISP) backbone networks
- Large carrier infrastructure
- Internet Exchange Points
- Major cloud providers including Google and Amazon

The reasons ISPs prefer IS-IS over OSPF are that it scales
better to very large networks, is more stable under heavy route
changes, has less overhead in large topologies, and runs
directly on Layer 2 rather than on top of IP — making it
slightly more resilient to IP layer issues.

### How IS-IS Discovers Neighbors

This is the most fundamental difference from OSPF. OSPF
discovers neighbors using IP packets sent to multicast
224.0.0.5. IS-IS discovers neighbors using Layer 2 frames
sent directly to MAC multicast address 0180.C200.0014 —
completely bypassing IP. This means IS-IS can technically
form adjacencies even if IP is not configured on an interface,
though in practice IP is always configured.

### NET Address — The Most Unique IS-IS Concept

Instead of a Router ID like OSPF, every IS-IS router needs
a NET (Network Entity Title) address — an OSI format address
that identifies both the router and the area it belongs to.

NET format: 49.0001.0010.0100.1001.00 \
49           → AFI: 49 = private local address (like RFC1918) \
0001         → Area ID: which area this router belongs to \
0010.0100.1001 → System ID: unique 6-byte router identifier \
00           → NSEL: always 00 for a router

**How to derive System ID from loopback IP:**

Loopback: 1.1.1.1 \
Step 1 — pad every octet to 3 digits:  001.001.001.001 \
Step 2 — remove dots, write 12 digits: 001001001001 \
Step 3 — split into groups of 4:       0010.0100.1001 \
System ID = 0010.0100.1001 \
Full NET  = 49.0001.0010.0100.1001.00

### IS-IS Router Types

**L1 (Level 1) router** — knows only its own area topology.
For destinations outside the area it installs a default route
automatically pointing to the nearest L1/L2 router. It learns
this exit through the ATT bit — no manual default route needed.

**L2 (Level 2) router** — pure backbone router. Knows the
topology of all L2-capable routers across all areas but has
no knowledge of L1 intra-area topology. Used for dedicated
backbone devices with no area attachment.

**L1/L2 router** — border router between an area and the
backbone. Maintains two completely separate link state
databases — one for L1 (area topology) and one for L2
(backbone topology). Acts like an OSPF ABR but works
fundamentally differently.

### The ATT Bit — How L1 Routers Find Their Exit

When an L1/L2 router is connected to both its area and the
backbone, it sets the Attached bit (ATT bit) in its L1 LSP.
This signals to all L1 routers in the area:

"I am connected to the backbone — send inter-area traffic to me"

L1 routers automatically install a default route pointing to
the L1/L2 router when they detect the ATT bit. This is
verified in this lab by checking R2's routing table — it shows
a default route that was never manually configured, created
entirely by the ATT bit mechanism.

### Two Separate Databases on L1/L2 Routers

L1 database → area topology only
shared only with routers in the same area
used to calculate intra-area paths

L2 database → backbone topology only
shared only with L2 and L1/L2 routers
used to calculate inter-area paths

This is visible in the lab on R1 — running
`show isis database` shows two distinct sections,
one labeled Level-1 and one labeled Level-2.

### How IS-IS Areas Work vs OSPF

OSPF:  Area 1 → Area 0 (mandatory) ← Area 2 must exist, must be area 0

IS-IS: Area 1 ←→ Area 2 ←→ Area 3 (through L2 links between L1/L2 and L2 routers) \
no mandatory area number — backbone is a level

In OSPF the backbone is an area with a specific number.
In IS-IS the backbone is formed by the collection of all L2
adjacencies between L2 and L1/L2 routers — it is a level,
not an area. The L2 backbone routers in this lab use area
49.0000 which is just a convention — no special significance.

### IS-IS Area Design Rules

Four rules that must not be broken:

1. All L2 routers must be contiguous — no isolated L2 islands
2. L1 routers must share the same area ID as their L1/L2 border
3. Area IDs must match exactly for L1 adjacency to form
4. L2 adjacency ignores area ID — only level matters for L2

### IS-IS Metric

Default IS-IS assigns metric 10 to every interface regardless
of bandwidth — identical to RIP's hop count problem. A 1Mbps
link and a 10Gbps link both cost 10 by default.

Wide metrics fix this: \
`router isis` \
`metric-style wide`

Then meaningful per-interface costs can be set: \
`interface Ethernet0/0` \
`isis metric 100`

### Problem I Encountered — L1 Adjacency Not Forming

During configuration R1 (L1/L2) and R2 (L1) were not forming
an adjacency even though their interfaces were up and IPs were
assigned correctly. After investigation the cause was an area
ID mismatch in the NET addresses — R1 and R2 had different
area IDs so the L1 Hello packets were being rejected.

The fix was correcting the NET address on one router to ensure
both had exactly 49.0001 as their area ID. L1 adjacency formed
immediately after the correction. This reinforced the most
critical IS-IS rule — area IDs must match exactly for L1
adjacency, even one digit different causes silent failure.

### CLNS Routing Command

IS-IS uses CLNS (Connectionless Network Service) for its
own PDU communication at Layer 2 even when only carrying
IP routes. The `clns routing` command enables OSI/CLNS
processing globally on the router. On some IOS versions
this is required before IS-IS adjacencies can form. On
newer versions it is enabled automatically. If IS-IS
neighbors are not forming despite correct configuration,
adding `clns routing` is the first troubleshooting step.

---

## IP Address Table

### Router-to-Router Links — Area 1

| Link | Network | Router A | Interface | IP Address | Router B | Interface | IP Address |
|---|---|---|---|---|---|---|---|
| R1 — R2 | 10.1.12.0/30 | R1 | e0/2 | 10.1.12.2 | R2 | e0/2 | 10.1.12.1 |
| R1 — R3 | 10.1.13.0/30 | R1 | e0/1 | 10.1.13.2 | R3 | e0/1 | 10.1.13.1 |

### Router-to-Router Links — Area 2

| Link | Network | Router A | Interface | IP Address | Router B | Interface | IP Address |
|---|---|---|---|---|---|---|---|
| R4 — R6 | 10.2.46.0/30 | R4 | e0/1 | 10.2.46.2 | R6 | e0/1 | 10.2.46.1 |
| R5 — R6 | 10.2.56.0/30 | R5 | e0/2 | 10.2.56.2 | R6 | e0/2 | 10.2.56.1 |

### Router-to-Router Links — Area 3

| Link | Network | Router A | Interface | IP Address | Router B | Interface | IP Address |
|---|---|---|---|---|---|---|---|
| R7 — R8 | 10.3.78.0/30 | R7 | e0/2 | 10.3.78.2 | R8 | e0/2 | 10.3.78.1 |
| R7 — R9 | 10.3.79.0/30 | R7 | e0/1 | 10.3.79.2 | R9 | e0/1 | 10.3.79.1 |

### L2 Backbone Links

| Link | Network | Router A | Interface | IP Address | Router B | Interface | IP Address |
|---|---|---|---|---|---|---|---|
| R1 — L2-R1 | 10.0.10.0/30 | R1 | e0/0 | 10.0.10.1 | L2-R1 | e0/0 | 10.0.10.2 |
| R4 — L2-R2 | 10.0.42.0/30 | R4 | e0/2 | 10.0.42.1 | L2-R2 | e0/2 | 10.0.42.2 |
| R5 — L2-R2 | 10.0.52.0/30 | R5 | e0/1 | 10.0.52.1 | L2-R2 | e0/1 | 10.0.52.2 |
| R7 — L2-R3 | 10.0.73.0/30 | R7 | e0/3 | 10.0.73.1 | L2-R3 | e0/3 | 10.0.73.2 |
| L2-R1 — L2-R2 | 10.0.12.0/30 | L2-R1 | e0/3 | 10.0.12.1 | L2-R2 | e0/3 | 10.0.12.2 |
| L2-R2 — L2-R3 | 10.0.23.0/30 | L2-R2 | e0/0 | 10.0.23.1 | L2-R3 | e0/0 | 10.0.23.2 |

### LAN Networks

| Device | Interface | IP Address | Subnet | Default Gateway |
|---|---|---|---|---|
| R2 | e0/0 | 10.1.1.100 | 255.255.255.0 | — |
| PC1 | eth0 | 10.1.1.1 | 255.255.255.0 | 10.1.1.100 |
| R3 | e0/0 | 10.1.2.100 | 255.255.255.0 | — |
| PC2 | eth0 | 10.1.2.1 | 255.255.255.0 | 10.1.2.100 |
| R6 | e0/0 | 10.2.1.100 | 255.255.255.0 | — |
| PC3 | eth0 | 10.2.1.1 | 255.255.255.0 | 10.2.1.100 |
| R8 | e0/0 | 10.3.1.100 | 255.255.255.0 | — |
| PC4 | eth0 | 10.3.1.1 | 255.255.255.0 | 10.3.1.100 |
| R9 | e0/0 | 10.3.2.100 | 255.255.255.0 | — |
| PC5 | eth0 | 10.3.2.1 | 255.255.255.0 | 10.3.2.100 |

### Loopback Interfaces and NET Addresses

| Router | Type | Interface | Loopback IP | NET Address |
|---|---|---|---|---|
| R1 | L1/L2 | Lo0 | 1.1.1.1/32 | 49.0001.0010.0100.1001.00 |
| R2 | L1 | Lo0 | 2.2.2.2/32 | 49.0001.0020.0200.2002.00 |
| R3 | L1 | Lo0 | 3.3.3.3/32 | 49.0001.0030.0300.3003.00 |
| R4 | L1/L2 | Lo0 | 4.4.4.4/32 | 49.0002.0040.0400.4004.00 |
| R5 | L1/L2 | Lo0 | 5.5.5.5/32 | 49.0002.0050.0500.5005.00 |
| R6 | L1 | Lo0 | 6.6.6.6/32 | 49.0002.0060.0600.6006.00 |
| R7 | L1/L2 | Lo0 | 7.7.7.7/32 | 49.0003.0070.0700.7007.00 |
| R8 | L1 | Lo0 | 8.8.8.8/32 | 49.0003.0080.0800.8008.00 |
| R9 | L1 | Lo0 | 9.9.9.9/32 | 49.0003.0090.0900.9009.00 |
| L2-R1 | L2 | Lo0 | 10.0.1.1/32 | 49.0000.0100.0001.0001.00 |
| L2-R2 | L2 | Lo0 | 10.0.2.2/32 | 49.0000.0100.0002.0002.00 |
| L2-R3 | L2 | Lo0 | 10.0.3.3/32 | 49.0000.0100.0003.0003.00 |

---

## Verification

### IS-IS Neighbor Verification

<img width="1632" height="316" alt="R1 ISIS Neghbor" src="https://github.com/user-attachments/assets/6e9a0b6a-4445-4ca7-b392-603a4156b7da" /><br>
> R1 (L1/L2) neighbor table showing L1 adjacencies with
> R2 and R3 inside Area 1, and L2 adjacency with L2-R1
> on the backbone link — both levels active simultaneously

<img width="1576" height="395" alt="L2R2 ISIS Neghbor" src="https://github.com/user-attachments/assets/1320a225-28a8-4562-a206-d1e7ab1c3b6d" /><br>
> L2-R2 backbone router neighbor table showing all three
> L2 adjacencies — to L2-R1, L2-R3 on backbone links,
> and to R4 and R5 from Area 2 connecting upward

<img width="1588" height="238" alt="R6 ISIS Neghbor" src="https://github.com/user-attachments/assets/392ee498-c70c-4c5d-915e-7ff294c26eb3" /><br>
> R6 (L1 only in Area 2) showing L1 adjacencies with
> both R4 and R5 — two L1/L2 border routers serving Area 2

### IS-IS Database

<img width="1315" height="693" alt="L1-L2 Database via R1" src="https://github.com/user-attachments/assets/ad5248d5-cf98-41f2-bf24-6522b28ad3ac" /><br>
> R1 (L1/L2) database showing two distinct sections —
> Level-1 contains only Area 1 router LSPs, Level-2
> contains backbone router LSPs from all areas. This
> confirms R1 maintains two completely separate databases.

### Routing Tables

<img width="1496" height="439" alt="R2 default route via L1" src="https://github.com/user-attachments/assets/6124e4cf-071e-4780-804a-287cc7be53fa" /><br>
> R2 (L1 only) routing table showing default route via R1
> — this route was never manually configured. It was
> automatically installed when R2 detected the ATT bit
> set in R1's L1 LSP, proving the ATT bit mechanism works.

<img width="1246" height="637" alt="L2R2 ISIS Route " src="https://github.com/user-attachments/assets/59f90ecd-66fa-4ca1-b80c-6398cfef1769" /><br>
<img width="1236" height="543" alt="L2R2 ISIS Route 2" src="https://github.com/user-attachments/assets/d7c01ed4-2b1d-4049-8a4c-9213053a912d" /><br>
> L2-R2 routing table showing IS-IS learned routes from
> all three areas — confirming the backbone router has
> full inter-area visibility while L1 routers only see
> their own area plus the ATT-generated default route

### NET Address Verification

<img width="927" height="666" alt="Area 1 clns protocol" src="https://github.com/user-attachments/assets/ec1e55d8-7a98-41aa-9d4a-5c4e9712d477" /><br>
> show clns protocol on Area 1 router confirming NET
> address with area ID 49.0001 and IS-IS level type

<img width="818" height="649" alt="Area 2 clns protocol" src="https://github.com/user-attachments/assets/b282497d-d7f3-4e63-a38e-c2975cbaa3f1" /><br>
> show clns protocol on Area 2 router confirming NET
> address with area ID 49.0002

<img width="914" height="683" alt="Area 3 clns protocol" src="https://github.com/user-attachments/assets/c71a4f12-cf2f-416b-a355-96dd9ae24bc9" /><br>
> show clns protocol on Area 3 router confirming NET
> address with area ID 49.0003

### Connectivity

<img width="1200" height="278" alt="PC1 to PC5 Ping" src="https://github.com/user-attachments/assets/8fefe120-845d-4cb0-b65d-54b7a9e66048" /><br>
> Ping from PC1 (Area 1) to PC5 (Area 3) — traffic
> crosses all three areas through the L2 backbone
> confirming full end to end connectivity

<img width="1652" height="388" alt="PC1 to PC5 Trace" src="https://github.com/user-attachments/assets/dcf9fcfc-7faa-481b-89cb-75f9bd41cb59" /><br>
> Traceroute from PC1 to PC5 showing the full path:
> PC1 → R2 → R1 → L2-R1 → L2-R2 → L2-R3 → R7 → R9 → PC5

<img width="1228" height="305" alt="PC1 to L2R2 Ping" src="https://github.com/user-attachments/assets/7926b816-b1a0-4049-a821-64d9a9269fdd" /><br>
> Ping from PC1 to L2-R2 backbone loopback 10.0.2.2
> confirming L1 routers can reach backbone devices
> through the ATT bit default route mechanism

---

## Router Configs

| Router | Type | Area | Config |
|---|---|---|---|
| R1 | L1/L2 border | Area 1 | [R1.txt](R1.txt) |
| R2 | L1 only | Area 1 | [R2.txt](R2.txt) |
| R3 | L1 only | Area 1 | [R3.txt](R3.txt) |
| R4 | L1/L2 border | Area 2 | [R4.txt](R4.txt) |
| R5 | L1/L2 border | Area 2 | [R5.txt](R5.txt) |
| R6 | L1 only | Area 2 | [R6.txt](R6.txt) |
| R7 | L1/L2 border | Area 3 | [R7.txt](R7.txt) |
| R8 | L1 only | Area 3 | [R8.txt](R8.txt) |
| R9 | L1 only | Area 3 | [R9.txt](R9.txt) |
| L2-R1 | L2 only | Backbone | [L2-R1.txt](L2-R1.txt) |
| L2-R2 | L2 only | Backbone | [L2-R2.txt](L2-R2.txt) |
| L2-R3 | L2 only | Backbone | [L2-R3.txt](L2-R3.txt) |
