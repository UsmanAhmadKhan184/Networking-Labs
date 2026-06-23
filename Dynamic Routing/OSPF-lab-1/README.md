# Lab 01 — OSPF Multi-Area

## What is OSPF

OSPF stands for Open Shortest Path First. It is a link state
dynamic routing protocol — routers automatically discover each
other, share topology information, and build a complete map of
the network independently. Unlike static routing where you
manually write every route, OSPF handles all of this
automatically and adapts when links change.

Every router running OSPF:
- Discovers directly connected neighbors using Hello packets
- Shares its link information with every other router in the area
- Builds a complete topology map called the Link State Database
- Calculates the shortest path to every destination using
  Dijkstra's algorithm independently

---

## What I Built

A multi-area OSPF network with three areas — Area 1, Area 0
(backbone), and Area 2. Each area contains four routers in a
partial mesh topology. Area Border Routers connect the areas
and summarize routing information between them. Six VPCs
spread across all three areas verify end to end connectivity.

---

## Topology

<img width="1658" height="941" alt="OSPF Lab 1" src="https://github.com/user-attachments/assets/7d0d29a4-4cd3-452e-a2a7-0fca8bd88dcd" /><br>

---

## Area Structure

| Area | Routers | Role |
|---|---|---|
| Area 1 | R1, R2, R3 | Internal routers |
| Area 1 | R4 | ABR — connects Area 1 to Area 0 |
| Area 0 | R5, R7, R6 | Internal backbone routers |
| Area 0 | R8 | ABR — connects Area 0 to Area 2 |
| Area 2 | R9 | ABR — connects Area 2 to Area 0 |
| Area 2 | R10, R11, R12 | Internal routers |

---

## What I Learned

**Why multi-area OSPF exists**

Putting all routers in a single area creates a serious
scalability problem. Every time any link changes anywhere in
the network — even a single interface flapping — every router
in the area must recalculate its entire routing table using the
SPF algorithm simultaneously. In a large network this wastes
significant CPU and memory resources.

Multi-area OSPF contains topology changes inside their own
area. A link failure in Area 1 triggers SPF recalculation only
in Area 1 — Area 0 and Area 2 are completely unaffected.

**The ABR role**

An Area Border Router is a router with interfaces in two or
more areas. In this lab R4 connects Area 1 to Area 0, R8
connects Area 0 to Area 2, and R9 also sits between Area 0
and Area 2. ABRs maintain a completely separate link state
database for each area they belong to. They generate Type 3
Summary LSAs to advertise networks from one area into another.
Internal routers only know the detailed topology of their own
area — they see other areas only as summary routes.

**O vs O IA in the routing table**

When you run show ip route ospf you see two types of OSPF
routes. O means the route is intra-area — it was learned from
a router inside the same area and the full topology detail is
known. O IA means the route is inter-area — it came from
another area via an ABR as a summary. R1 for example sees all
Area 1 routes as O and everything from Area 0 and Area 2 as
O IA. This is working exactly as designed.

**Wildcard mask — bit flip method**

A wildcard mask tells OSPF which bits of an IP address to
check and which bits to ignore. 0 means check this bit,
1 means ignore this bit — the opposite logic of a subnet mask.

**Calculation using the bit-flip method:**

**Example: /20 subnet** 

Step 1 — Write /20 in binary (20 ones, 12 zeros): \
11111111.11111111.11110000.00000000 = 255.255.240.0 \
Step 2 — Flip every bit (0 becomes 1, 1 becomes 0): \
00000000.00000000.00001111.11111111 \
Step 3 — Convert to decimal: \
0.0.15.255 \
Wildcard for /20 = 0.0.15.255

This method works correctly for every subnet — classful or
classless — unlike the simple subtraction shortcut which
breaks for non-standard wildcard patterns used in ACLs.

**Note on passive interfaces**

Passive interfaces were not configured in this lab. They will
be added in the next lab. A passive interface stops OSPF Hello
packets from being sent out interfaces connected to switches
and end devices — saving bandwidth and preventing unnecessary
neighbor formation attempts on LAN segments.

---

## IP Address Table

### Router-to-Router Links

| Link | Network | Router A | Interface | IP | Router B | Interface | IP |
|---|---|---|---|---|---|---|---|
| R1 — R2 | 172.16.1.0/30 | R1 | e0/0 | 172.16.1.1 | R2 | e0/0 | 172.16.1.2 |
| R1 — R3 | 172.16.3.0/30 | R1 | e0/1 | 172.16.3.1 | R3 | e0/1 | 172.16.3.2 |
| R2 — R4 | 172.16.2.0/30 | R2 | e0/1 | 172.16.2.1 | R4 | e0/0 | 172.16.2.2 |
| R3 — R4 | 172.16.4.0/30 | R3 | e0/0 | 172.16.4.1 | R4 | e0/0 | 172.16.4.2 |
| R4 — R5 | 172.16.5.0/30 | R4 | e1/0 | 172.16.5.1 | R5 | e1/0 | 172.16.5.2 |
| R5 — R6 | 172.16.6.0/30 | R5 | e0/0 | 172.16.6.1 | R6 | e0/0 | 172.16.6.2 |
| R5 — R7 | 172.16.9.0/30 | R5 | e0/1 | 172.16.9.1 | R7 | e0/1 | 172.16.9.2 |
| R6 — R8 | 172.16.7.0/30 | R6 | e0/1 | 172.16.7.1 | R8 | e0/1 | 172.16.7.2 |
| R7 — R8 | 172.16.8.0/30 | R7 | e0/0 | 172.16.8.1 | R8 | e0/0 | 172.16.8.2 |
| R8 — R9 | 172.16.10.0/30 | R6 | e1/0 | 172.16.10.1 | R9 | e1/0 | 172.16.10.2 |
| R9 — R10 | 172.16.13.0/30 | R9 | e0/0 | 172.16.13.1 | R10 | e0/0 | 172.16.13.2 |
| R9 — R11 | 172.16.14.0/30 | R9 | e0/1 | 172.16.14.1 | R11 | e0/1 | 172.16.14.2 |
| R11 — R12 | 172.16.11.0/30 | R11 | e0/0 | 172.16.11.1 | R12 | e0/0 | 172.16.11.2 |
| R10 — R12 | 172.16.12.0/30 | R10 | e0/1 | 172.16.12.1 | R12 | e0/1 | 172.16.12.2 |

### LAN Networks

| Device | Interface | IP Address | Subnet | Default Gateway |
|---|---|---|---|---|
| R3 | e0/3 | 192.168.1.100 | 255.255.255.0 | — |
| VPC1 | eth0 | 192.168.1.1 | 255.255.255.0 | 192.168.1.100 |
| VPC2 | eth0 | 192.168.1.2 | 255.255.255.0 | 192.168.1.100 |
| R7 | e0/3 | 192.168.3.100 | 255.255.255.0 | — |
| VPC3 | eth0 | 192.168.3.1 | 255.255.255.0 | 192.168.3.100 |
| VPC4 | eth0 | 192.168.3.2 | 255.255.255.0 | 192.168.3.100 |
| R10 | e0/2 | 192.168.2.100 | 255.255.255.0 | — |
| VPC5 | eth0 | 192.168.2.1 | 255.255.255.0 | 192.168.2.100 |
| VPC6 | eth0 | 192.168.2.2 | 255.255.255.0 | 192.168.2.100 |

### Loopback Interfaces

| Router | Interface | IP | Network | Area |
|---|---|---|---|---|
| R1 | Lo0 | 1.1.1.1 | 1.1.1.0/24 | Area 1 |
| R2 | Lo0 | 2.2.2.1 | 2.2.2.0/24 | Area 1 |
| R3 | Lo0 | 3.3.3.1 | 3.3.3.0/24 | Area 1 |
| R4 | Lo0 | 4.4.4.1 | 4.4.4.0/24 | Area 1 |
| R5 | Lo0 | 5.5.5.1 | 5.5.5.0/24 | Area 0 |
| R6 | Lo0 | 6.6.6.1 | 6.6.6.0/24 | Area 0 |
| R7 | Lo0 | 7.7.7.1 | 7.7.7.0/24 | Area 0 |
| R8 | Lo0 | 8.8.8.1 | 8.8.8.0/24 | Area 0 |
| R9 | Lo0 | 9.9.9.1 | 9.9.9.0/24 | Area 2 |
| R10 | Lo0 | 10.10.10.1 | 10.10.10.0/24 | Area 2 |
| R11 | Lo0 | 11.11.11.1 | 11.11.11.0/24 | Area 2 |
| R12 | Lo0 | 12.12.12.1 | 12.12.12.0/24 | Area 2 |

---

## Verification

<img width="1672" height="362" alt="R4 Neighbor" src="https://github.com/user-attachments/assets/7e354195-9984-4388-ad52-250d8de9af46" /><br>
> R4 ABR neighbor table — shows neighbors in both Area 1
> and Area 0 confirming ABR is functioning correctly

<img width="1662" height="286" alt="R8 Neighbor" src="https://github.com/user-attachments/assets/8a013a6d-9559-4b16-8938-47407b77e65c" /><br>
> R8 neighbor table — shows full adjacency with Area 0 routers

<img width="1659" height="331" alt="R9 Neighbor" src="https://github.com/user-attachments/assets/aae873ab-f1d7-424f-bc58-08e4f6b31e59" /><br>
> R9 ABR neighbor table — shows neighbors in both Area 0
> and Area 2

**R1 Routing Table**

<img width="1217" height="562" alt="R1 ip ospf route" src="https://github.com/user-attachments/assets/e4fbe7d4-3dd7-47d7-a152-16855f6e1cc3" /><br>
> R1 intra-area routes (O) — networks learned from within Area 1

<img width="884" height="686" alt="R1 ip ospf route inter1" src="https://github.com/user-attachments/assets/ecc2bc10-9176-4259-b6c4-bd24048b28fa" /><br>
<img width="835" height="485" alt="R1 ip ospf route inter2" src="https://github.com/user-attachments/assets/08261b2c-2f1b-4527-bc61-32a5c65da2eb" /><br>
> R1 inter-area routes (O IA) — networks learned from Area 0
> and Area 2 via R4 ABR as summary routes

**R5 Routing Table**

<img width="1280" height="698" alt="R5 ip ospf route intra" src="https://github.com/user-attachments/assets/ea268fd9-805d-4691-b9c5-a0743d1bc45b" /><br>
> R5 intra-area routes (O) — full Area 0 topology visible

<img width="987" height="572" alt="R5 ip ospf route inter1" src="https://github.com/user-attachments/assets/b0c43711-daa1-481c-ae44-8f96bc243e02" /><br>
<img width="958" height="280" alt="R5 ip ospf route inter2" src="https://github.com/user-attachments/assets/c5fc5965-f562-49d7-b7a5-e206480c71a9" /><br>
> R5 inter-area routes (O IA) — summary routes from
> Area 1 and Area 2 visible through ABRs

<img width="1293" height="339" alt="Ping VPC1 to VPC5" src="https://github.com/user-attachments/assets/e7c21f4c-997d-484b-b000-20d334a819d0" /><br>
> Ping from VPC1 in Area 1 to VPC5 in Area 2 — traffic
> crosses all three areas confirming full end to end connectivity

<img width="1683" height="452" alt="Trace VPC1 to VPC5" src="https://github.com/user-attachments/assets/190a67f7-8d5a-40f2-bda9-81f3cacc64b1" /><br>
> Traceroute showing full path — VPC1 → R3 → R1/R2 →
> R4 → R5/R6 → R8/R9 → R10 → VPC5

---

## Router Configs

| Router | Area | Role | Config |
|---|---|---|---|
| R1 | Area 1 | Internal | [R1-config.txt](R1-config.txt) |
| R2 | Area 1 | Internal | [R2-config.txt](R2-config.txt) |
| R3 | Area 1 | Internal + LAN | [R3-config.txt](R3-config.txt) |
| R4 | Area 1 / Area 0 | ABR | [R4-config.txt](R4-config.txt) |
| R5 | Area 0 | Internal | [R5-config.txt](R5-config.txt) |
| R6 | Area 0 | Internal | [R6-config.txt](R6-config.txt) |
| R7 | Area 0 | Internal + LAN | [R7-config.txt](R7-config.txt) |
| R8 | Area 0 / Area 2 | ABR | [R8-config.txt](R8-config.txt) |
| R9 | Area 2 / Area 0 | ABR | [R9-config.txt](R9-config.txt) |
| R10 | Area 2 | Internal + LAN | [R10-config.txt](R10-config.txt) |
| R11 | Area 2 | Internal | [R11-config.txt](R11-config.txt) |
| R12 | Area 2 | Internal | [R12-config.txt](R12-config.txt) |
