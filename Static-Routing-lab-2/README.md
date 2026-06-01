# Lab 02 — Static Routing (Real World Scenario)

## What I Built

In this lab I moved beyond basic routing and built a network based
on a real world scenario. The topology represents a company with
one head office in Lahore and two branch offices in Islamabad and
Karachi, all connected through an ISP network in the middle.

The goals I set before starting:

- PC1 in Lahore can ping PC5 in Karachi
- PC3 in Islamabad can ping PC2 in Lahore
- All loopbacks are reachable from all offices
- ISP routers forward traffic between offices without knowing
  the internal LAN subnets

---

## Topology

<img width="1837" height="913" alt="Static-Routing-Lab-2" src="https://github.com/user-attachments/assets/5e429475-8c86-4b23-869a-f3e1c4a40aac" /><br>

---

## What I Learned

**ISP routers are transit devices — not full routers**

The biggest concept I understood in this lab is that ISP routers
do not care about your internal office networks. They only know
about the WAN links connecting them to each office router. Their
only job is to receive a packet and push it one step closer to
the destination. The office routers carry the full knowledge of
where everything lives.

**The /16 overlap mistake**

While configuring ISP-R1 I assigned 172.16.0.1/16 to interface
e0/0 for the backbone link between the two ISP routers. When I
then tried to assign 172.16.23.2/30 to e0/2 for the Islamabad
link, I got this error:

% 172.16.23.0 overlaps with Ethernet0/0

At first it confused me because the IPs looked different. Then I
realized the problem — a /16 mask on 172.16.0.0 covers the entire
range from 172.16.0.0 all the way to 172.16.255.255. So
172.16.23.2 was already inside the range claimed by e0/0 and the
router rejected it.

The fix was simple — use /30 on the ISP backbone link, not /16.
The /16 on the diagram was just showing which address block the
WAN space belongs to, not the actual mask to configure on the
interface.

**Rule I will not forget:**
> Always use /30 on point-to-point router links.
> Never use /16 or /24 between two routers.

**Never write static routes for your own networks**

Early in the lab I wrote static routes pointing back to my own
directly connected networks. The router already knows about its
own interfaces automatically — writing a static route for them
is not only unnecessary but causes routing confusion. Static
routes are only for networks on the other side of the network
that the router cannot reach directly.

---

## IP Address Table

### Router-to-Router Links

| Link | Network | Device A | Interface | IP Address | Device B | Interface | IP Address |
|---|---|---|---|---|---|---|---|
| R1 — ISP-R1 | 172.16.12.0/30 | R1-LHR | e0/1 | 172.16.12.1 | ISP-R1 | e0/1 | 172.16.12.2 |
| R2 — ISP-R1 | 172.16.23.0/30 | R2-ISL | e0/2 | 172.16.23.1 | ISP-R1 | e0/2 | 172.16.23.2 |
| R3 — ISP-R2 | 172.16.34.0/30 | R3-KHI | e0/1 | 172.16.34.1 | ISP-R2 | e0/1 | 172.16.34.2 |
| ISP-R1 — ISP-R2 | 172.16.0.0/30 | ISP-R1 | e0/0 | 172.16.0.1 | ISP-R2 | e0/0 | 172.16.0.2 |

### LAN Networks

| Device | Interface | IP Address | Subnet | Default Gateway |
|---|---|---|---|---|
| R1-LHR | e0/0 | 192.168.1.100 | 255.255.255.0 | — |
| VPC1 | eth0 | 192.168.1.1 | 255.255.255.0 | 192.168.1.100 |
| VPC2 | eth0 | 192.168.1.2 | 255.255.255.0 | 192.168.1.100 |
| R2-ISL | e0/0 | 192.168.2.100 | 255.255.255.0 | — |
| VPC3 | eth0 | 192.168.2.1 | 255.255.255.0 | 192.168.2.100 |
| VPC4 | eth0 | 192.168.2.2 | 255.255.255.0 | 192.168.2.100 |
| R3-KHI | e0/0 | 192.168.3.100 | 255.255.255.0 | — |
| VPC5 | eth0 | 192.168.3.1 | 255.255.255.0 | 192.168.3.100 |
| VPC6 | eth0 | 192.168.3.2 | 255.255.255.0 | 192.168.3.100 |

### Loopback Interfaces

| Router | Interface | IP Address | Network |
|---|---|---|---|
| R1-LHR | Loopback0 | 10.10.10.1 | 10.10.10.0/24 |
| R2-ISL | Loopback0 | 10.20.20.1 | 10.20.20.0/24 |
| R3-KHI | Loopback0 | 10.30.30.1 | 10.30.30.0/24 |

---

## Verification

<img width="1248" height="344" alt="Ping Vpc1 to Vpc6" src="https://github.com/user-attachments/assets/ba7aba72-f8da-486c-bc83-68dc808d8dd1" /><br>
> Ping from VPC1 (Lahore) to VPC6 (Karachi) — full end to end
> connectivity confirmed across both ISP routers

<img width="1684" height="349" alt="Traceroute Vpc1 to Vpc6" src="https://github.com/user-attachments/assets/4910162f-f1cc-464b-9ff3-e3a31f1e0e40" /><br>
> Traceroute shows the exact path — VPC1 → R1 → ISP-R1 →
> ISP-R2 → R3 → VPC6

<img width="1227" height="601" alt="Ip Route R1" src="https://github.com/user-attachments/assets/de6d54db-15ae-475f-9936-f3382c88cd93" /><br>
> Routing table on R1-LHR showing all static routes configured

<img width="1113" height="493" alt="Ip Route ISP-R1" src="https://github.com/user-attachments/assets/66ce2047-3d26-4fdb-92d4-6a927e26b679" /><br>
> Routing table on ISP-R1 showing only transit routes
> no internal LAN knowledge

---

## Router Configs

| Router | Config File |
|---|---|
| R1 — LHR | [R1.txt](R1.txt) |
| R2 — ISL | [R2.txt](R2.txt) |
| R3 — KHI | [R3.txt](R3.txt) |
| ISP-R1 | [ISP-R1.txt](ISP-R1.txt) |
| ISP-R2 | [ISP-R2.txt](ISP-R2.txt) |
