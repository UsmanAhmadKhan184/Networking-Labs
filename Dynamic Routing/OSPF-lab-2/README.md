# Lab 02 — OSPF Multi-Area Advanced Concepts

## What I Built

Same multi-area topology as Lab 1 but with four additional
concepts layered on top: passive interfaces, cost manipulation,
DR/BDR election, and route summarization. Three areas — Area 1,
Area 0 (backbone), and Area 2 — connected through R4-ABR and
R12-ABR, with a dedicated multi-access segment added in Area 0
specifically for DR/BDR election.

## Topology

<img width="1844" height="875" alt="OSPF-lab-2" src="https://github.com/user-attachments/assets/d63270f7-488b-4cc7-9f68-649c341af4c1" />

## What I Learned

### Passive Interfaces

LAN interfaces connecting to end devices should not continuously
send OSPF Hello packets. Doing so wastes router CPU and bandwidth
since end hosts do not run OSPF and have no use for these packets.
It also reduces the risk of an unauthorized device forming an
OSPF neighbor relationship and injecting false routing
information into the network.

In this topology R3 Ethernet0/2 (connecting to 192.168.1.0/24),
R11 Ethernet0/3 (connecting to 192.168.3.0/24), and R14
Ethernet1/0 (connecting to 192.168.2.0/24) are configured as
passive interfaces.

### DR/BDR Election

OSPF elects a Designated Router (DR) and Backup Designated
Router (BDR) on multi-access networks to minimize routing
traffic between routers. Instead of every router forming a full
adjacency with every other router on the segment, all routers
form a full adjacency only with the DR and BDR. Between two
non-DR/BDR routers the relationship stays at 2WAY.

DR/BDR election only happens on multi-access segments — never on
point-to-point links. A point-to-point link only ever has two
routers, so there is nothing to elect. A multi-access segment can
have many routers, and without DR/BDR every router would form
adjacencies with every other router, creating unnecessary overhead.

The election depends primarily on interface priority — the
higher the priority, the higher the chance of becoming DR or BDR.
A priority of 0 means the router can never become DR or BDR. If
priorities are equal, the tiebreaker is the Router ID — the
highest Router ID wins.

Routers that are neither DR nor BDR are called DROthers.

In this lab the DR/BDR segment is 10.0.91.0/24 connected through
DR-SW, with R9, R10, and R11 participating in the election.

### Cost Manipulation

Cost is the metric OSPF uses to determine the best path to a
destination — lower cost wins. It works exactly like a real world
map application: if a longer route has a lower total distance
than a shorter looking direct route, the longer route is chosen.

R1 ───cost 5─── R2 ───cost 4─── R3

└──────── cost 10 ──────────┘

Even though R1 to R3 directly looks shorter on paper (cost 10),
the path through R2 has a lower total cost (5+4=9), so OSPF
prefers R1 → R2 → R3.

Cost is calculated using:
Cost = Reference Bandwidth / Interface Bandwidth

In this lab the R5-R6 link was set to cost 10 (primary path) and
the R7-R8 link was set to cost 100 (backup path). With both links
up, traffic between Area 1 and Area 2 prefers the R5-R6 path.
When R6 was shut down, traffic automatically failed over to the
R7-R8 path with the higher cost reflected in the routing table.
When R6 was brought back up, traffic returned to the original
low cost path automatically — no manual intervention needed.

**Important discovery — cost is per interface, per direction.**
Setting `ip ospf cost 100` on R7's interface toward R8 only
affects traffic leaving R7 in that direction. R8's interface
toward R7 keeps its default cost unless explicitly changed. This
caused R8 to see two equal cost paths to Area 1 (via R6 and via
R7) even though only one direction of the R7-R8 link was made
expensive — resulting in OSPF load balancing across both paths
(ECMP). For a true asymmetric primary/backup design, cost must be
set on both directions of the link.

### Route Summarization

Summarization combines multiple contiguous network subnets into
a single advertised route, reducing the size of routing tables
in other areas.

**Bit Calculation Method**

Take four networks:
192.168.1.0/24 \
192.168.2.0/24 \
192.168.3.0/24 \
192.168.4.0/24 

The first two octets are identical, so look at the third octet
in binary:
1 = 00000001 \
2 = 00000010 \
3 = 00000011 \
4 = 00000100 

5 bits match from the left

Summary prefix = 16 (first two octets) + 5 (matching bits) = /21 \
Summary network = take the 5 matching bits, set the rest to 0 \
= 192.168.0.0/21 

Note — the summary address is built from the matching bits set
to a base value, not simply the first network's address. For this
group the matching bits produce 192.168.0.0, not 192.168.1.0.

**Power of 2 Method**

Only works on contiguous networks where the count is a power of 2
AND the starting network is on a valid boundary for that count.

192.168.4.0/24 \
192.168.5.0/24 \
192.168.6.0/24 \
192.168.7.0/24

4 networks = 2^2 → shrink prefix by 2 → /24 - 2 = /22 \
Boundary check: 4 ÷ 4 = 1 (whole number) → valid \
Summary = 192.168.4.0/22

**Why the two methods can disagree**

The power of 2 method only works when the networks are sequential
AND start on a boundary divisible by the group size. The earlier
192.168.1-4 example does NOT start on a /22 boundary (valid /22
boundaries are 0, 4, 8, 12...), so applying power of 2 there
would give a wrong answer. The bit method has no such
assumptions and is always correct — it should be the default
method, with power of 2 used only as a quick sanity check on
clearly sequential, boundary-aligned groups.

**Short Bit Calculation Method (for large groups)**

Writing every network in binary becomes impractical with 100+
networks. Instead, take only the first and last network: \
192.168.0.0/24 \
192.168.1.0/24 \
... \
192.168.127.0/24   (128 networks total) \
First: 0   = 00000000 \
Last:  127 = 01111111 

only 1 bit matches \
Summary prefix = 16 + 1 = /17 \
Summary = 192.168.0.0/17 

**Faster Mental Method (verification)**
Step 1 — Range = Last - First = 127 - 0 = 127 \
Step 2 — Next power of 2 ≥ range → 128 \
Step 3 — 128 = 2^7 → borrow 7 bits → /24 - 7 = /17 \
Step 4 — Boundary check: First ÷ 128 = 0 → valid \
Summary = 192.168.0.0/17

**Applied in this lab**

R4-ABR summarizes Area 1's four /30 links (10.1.12.0,
10.1.13.0, 10.1.24.0, 10.1.34.0) into a single 10.1.0.0/16
route advertised into Area 0, and R12-ABR does the same for
Area 2's links into 10.2.0.0/16. Both summarizations install a
Null0 route as a loop-prevention mechanism. Only R4 and R12 can
do this since they are the actual ABRs — R5 and R8 are internal
Area 0 routers with no interfaces in other areas.

---

## IP Address Table

### Router-to-Router Links

| Link | Network | Router A | Interface | IP Address | Router B | Interface | IP Address |
|---|---|---|---|---|---|---|---|
| R1 — R2 | 10.1.12.0/30 | R1 | e0/0 | 10.1.12.1 | R2 | e0/0 | 10.1.12.2 |
| R1 — R3 | 10.1.13.0/30 | R1 | e0/1 | 10.1.13.1 | R3 | e0/1 | 10.1.13.2 |
| R2 — R4 | 10.1.24.0/30 | R2 | e0/1 | 10.1.24.1 | R4 | e0/1 | 10.1.24.2 |
| R3 — R4 | 10.1.34.0/30 | R3 | e0/0 | 10.1.34.1 | R4 | e0/0 | 10.1.34.2 |
| R4 — R5 | 10.1.45.0/30 | R4 | e0/2 | 10.1.45.1 | R5 | e0/2 | 10.1.45.2 |
| R5 — R6 | 10.0.56.0/30 | R5 | e0/0 | 10.0.56.1 | R6 | e0/0 | 10.0.56.2 |
| R5 — R7 | 10.0.57.0/30 | R5 | e0/1 | 10.0.57.1 | R7 | e0/1 | 10.0.57.2 |
| R6 — R8 | 10.0.68.0/30 | R6 | e0/1 | 10.0.68.1 | R8 | e0/1 | 10.0.68.2 |
| R7 — R8 | 10.0.78.0/30 | R7 | e0/0 | 10.0.78.1 | R8 | e0/0 | 10.0.78.2 |
| R8 — R12 | 10.0.81.0/30 | R8 | e0/2 | 10.0.81.1 | R12 | e0/2 | 10.0.81.2 |
| R7 — DR-SW | 10.0.90.0/30 | R7 | e0/2 | 10.0.90.1 | DR-SW | e0/2 | — |
| R12 — R13 | 10.2.81.0/30 | R12 | e0/0 | 10.2.81.1 | R13 | e0/0 | 10.2.81.2 |
| R12 — R14 | 10.2.14.0/30 | R12 | e0/1 | 10.2.14.1 | R14 | e0/1 | 10.2.14.2 |
| R13 — R15 | 10.2.35.0/30 | R13 | e0/1 | 10.2.35.1 | R15 | e0/1 | 10.2.35.2 |
| R14 — R15 | 10.2.34.0/30 | R14 | e0/0 | 10.2.34.1 | R15 | e0/0 | 10.2.34.2 |

### DR/BDR Multi-Access Segment

| Network | Router | Interface | IP Address | OSPF Priority | Role |
|---|---|---|---|---|---|
| 10.0.91.0/24 | R9 | e0/1 | 10.0.91.1 | 100 | DR |
| 10.0.91.0/24 | R10 | e0/3 | 10.0.91.2 | 50 | BDR |
| 10.0.91.0/24 | R11 | e0/1 | 10.0.91.3 | 0 | DROther |

### LAN Networks

| Device | Interface | IP Address | Subnet | Default Gateway |
|---|---|---|---|---|
| R3 | e0/2 | 192.168.1.100 | 255.255.255.0 | — |
| VPC1 | eth0 | 192.168.1.1 | 255.255.255.0 | 192.168.1.100 |
| VPC2 | eth0 | 192.168.1.2 | 255.255.255.0 | 192.168.1.100 |
| R11 | e0/3 | 192.168.3.100 | 255.255.255.0 | — |
| VPC3 | eth0 | 192.168.3.1 | 255.255.255.0 | 192.168.3.100 |
| VPC4 | eth0 | 192.168.3.2 | 255.255.255.0 | 192.168.3.100 |
| R14 | e1/0 | 192.168.2.100 | 255.255.255.0 | — |
| VPC5 | eth0 | 192.168.2.1 | 255.255.255.0 | 192.168.2.100 |
| VPC6 | eth0 | 192.168.2.2 | 255.255.255.0 | 192.168.2.100 |

### Loopback Interfaces

| Router | Interface | IP Address | Area |
|---|---|---|---|
| R1 | Lo0 | 1.1.1.1/32 | Area 1 |
| R2 | Lo0 | 2.2.2.2/32 | Area 1 |
| R3 | Lo0 | 3.3.3.3/32 | Area 1 |
| R4 | Lo0 | 4.4.4.4/32 | Area 1 (ABR) |
| R5 | Lo0 | 5.5.5.5/32 | Area 0 |
| R6 | Lo0 | 6.6.6.6/32 | Area 0 |
| R7 | Lo0 | 7.7.7.7/32 | Area 0 |
| R8 | Lo0 | 8.8.8.8/32 | Area 0 |
| R9 | Lo0 | 9.9.9.9/32 | Area 0 |
| R10 | Lo0 | 10.10.10.10/32 | Area 0 |
| R11 | Lo0 | 11.11.11.11/32 | Area 0 |
| R12 | Lo0 | 12.12.12.12/32 | Area 2 (ABR) |
| R13 | Lo0 | 13.13.13.13/32 | Area 2 |
| R14 | Lo0 | 14.14.14.14/32 | Area 2 |
| R15 | Lo0 | 15.15.15.15/32 | Area 2 |

---

## Verification

### DR/BDR Election Check

<img width="1336" height="199" alt="R9 DR-BDR Election Confirmation" src="https://github.com/user-attachments/assets/94aa8c4c-4d7b-46b0-a182-f777b7a41e55" /><br>
> R9 — elected DR (priority 100)

<img width="1436" height="217" alt="R10 DR-BDR Election Confirmation" src="https://github.com/user-attachments/assets/c997bbae-fff5-4980-86d2-77dd767c47a4" /><br>
> R10 — elected BDR (priority 50)

<img width="1453" height="226" alt="R11 DR-BDR Election Confirmation" src="https://github.com/user-attachments/assets/9a504073-5a7c-4ee9-ad13-493568b216d3" /><br>
> R11 — DROther (priority 0)

<img width="1502" height="252" alt="R7 DR-BDR Election Confirmation" src="https://github.com/user-attachments/assets/10d816a5-8195-4b20-9269-fcbee8685447" /><br>
> R7's view of the DR/BDR segment, confirming election result
> from a neighboring router's perspective

### Passive Interfaces

<img width="1632" height="619" alt="R3 Passive ineterface" src="https://github.com/user-attachments/assets/1b3e40bc-a9df-4bae-8a9e-a176069fe523" /><br>
> R3 — Ethernet0/2 passive, connecting to 192.168.1.0/24

<img width="1382" height="439" alt="R11 Passive ineterface" src="https://github.com/user-attachments/assets/caa3de09-2e18-4531-9552-77fdfa891c20" /><br>
> R11 — Ethernet0/3 passive, connecting to 192.168.3.0/24

<img width="1612" height="542" alt="R14 Passive ineterface" src="https://github.com/user-attachments/assets/a1ad7574-d45c-4663-b727-9c52ee5a7932" /><br>
> R14 — Ethernet1/0 passive, connecting to 192.168.2.0/24

### OSPF Cost Test

<img width="1140" height="418" alt="R5 OSPF route bfore test" src="https://github.com/user-attachments/assets/2a914e9c-ef11-41a7-9da5-47be52e2aa48" /><br>
> R5 routing table — Area 2 traffic routed via R6 (low cost path)

<img width="1125" height="437" alt="R5 OSPF route During test" src="https://github.com/user-attachments/assets/a8fd84ac-38af-41a3-ab9b-3eb88ec1886e" /><br>
> R6 shut down — traffic automatically rerouted via R7-R8
> (higher cost reflected in the routing table)

<img width="1094" height="448" alt="R5 OSPF route after test" src="https://github.com/user-attachments/assets/fa4c09be-4133-44b7-81aa-f8c13b9b4aff" /><br>
> R6 restored — traffic automatically returned to the
> original low cost path via R6

### Route Summarization

<img width="938" height="523" alt="R4 Summarization proof" src="https://github.com/user-attachments/assets/ab5b65e3-532a-4f6c-b3cc-588df0df21ef" /><br>
> R4-ABR — 10.1.0.0/16 installed as a summary route to Null0

<img width="981" height="505" alt="R12 Summarization proof" src="https://github.com/user-attachments/assets/dbdf90e9-3247-4285-b368-70cd21f57619" /><br>
> R12-ABR — 10.2.0.0/16 installed as a summary route to Null0

<img width="1162" height="446" alt="R5 Summarization proof" src="https://github.com/user-attachments/assets/ab597ee8-bd56-41f4-8386-2d7e62521e12" /><br>
> R5 — sees 10.1.0.0/16 and 10.2.0.0/16 as single O IA routes
> instead of individual /30 entries

<img width="1043" height="507" alt="R8 Summarization proof" src="https://github.com/user-attachments/assets/5bd2a5d1-b718-4d07-9bf3-509824dfd9a8" /><br>
> R8 — shows equal cost paths to Area 1 via both R6 and R7,
> demonstrating that OSPF cost is per-interface and per-direction

### Communication Verification

<img width="1349" height="287" alt="Ping VPC1 to VPC5" src="https://github.com/user-attachments/assets/783902b4-e134-48ef-b6bb-643565c19363" /><br>
> Ping from VPC1 (Area 1) to VPC5 (Area 2) — full end to end
> connectivity confirmed across all three areas

<img width="1682" height="375" alt="Trace VPC1 to VPC5" src="https://github.com/user-attachments/assets/84cf922b-59c8-4b4a-9351-ba5de80cd875" /><br>
> Traceroute confirms the path follows the low cost
> R5-R6 link as expected

---

## Router Configs

| Router | Area | Role | Config |
|---|---|---|---|
| R1 | Area 1 | Internal | [R1-config.txt](R1-config.txt) |
| R2 | Area 1 | Internal | [R2-config.txt](R2-config.txt) |
| R3 | Area 1 | Internal + LAN (passive) | [R3-config.txt](R3-config.txt) |
| R4 | Area 1 / Area 0 | ABR + summarization | [R4-config.txt](R4-config.txt) |
| R5 | Area 0 | Internal + cost (primary) | [R5-config.txt](R5-config.txt) |
| R6 | Area 0 | Internal + cost (primary) | [R6-config.txt](R6-config.txt) |
| R7 | Area 0 | Internal + cost (backup) | [R7-config.txt](R7-config.txt) |
| R8 | Area 0 | Internal + cost (backup) | [R8-config.txt](R8-config.txt) |
| R9 | Area 0 | DR (priority 100) | [R9-config.txt](R9-config.txt) |
| R10 | Area 0 | BDR (priority 50) | [R10-config.txt](R10-config.txt) |
| R11 | Area 0 | DROther (priority 0) + LAN (passive) | [R11-config.txt](R11-config.txt) |
| R12 | Area 2 / Area 0 | ABR + summarization | [R12-config.txt](R12-config.txt) |
| R13 | Area 2 | Internal | [R13-config.txt](R13-config.txt) |
| R14 | Area 2 | Internal + LAN (passive) | [R14-config.txt](R14-config.txt) |
| R15 | Area 2 | Internal | [R15-config.txt](R15-config.txt) |
