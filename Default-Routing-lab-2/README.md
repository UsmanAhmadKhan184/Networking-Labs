# Lab 02 — Default Routing (Three Tier Hierarchy)

## What I Built

A three tier real world network scenario representing a company
with one headquarters in Islamabad, two regional hubs 
and four branch offices in Faisalabad, Multan,
Islamabad and Quetta. The key concept — branches have zero
knowledge of the wider network. They just forward everything
to their regional hub and the hub handles the rest.

The three tiers each have a different routing behavior:
- Branches use a single default route toward their hub
- Hubs use a default route toward HQ plus specific static
  routes back to their own branches
- HQ carries full static routes to every network

---

## Topology

<img width="1648" height="832" alt="Default-Routing lab 2" src="https://github.com/user-attachments/assets/91b92d0c-4938-4095-a67f-5d06b42b87db" /><br>

---

## What I Learned

**Three tier routing hierarchy**

This lab made me understand that default routing is not just
a one level concept. In a real network different tiers of
routers carry different levels of routing knowledge depending
on their role. A branch does not need to know about other
branches. A hub does not need to know about other hubs. Only
HQ needs the full picture.

**The hub return route mistake**

The most important lesson from this lab came from a real
troubleshooting situation. The traceroute from VPC1 was
reaching hop 1 — the BR-FSD gateway — but dying at hop 2
with all stars. I checked the BR-FSD routing table and the
default route was correctly configured pointing to HUB-A.

The problem was not on the branch at all. HUB-A was receiving
the packet from BR-FSD but had no specific route back to
192.168.1.0/24 or 192.168.2.0/24. It was using its own
default route toward HQ which caused the packet to loop
and eventually die.

The fix was adding specific static routes on HUB-A for its
own branch networks:
ip route 192.168.1.0 255.255.255.0 172.16.11.2
ip route 192.168.2.0 255.255.255.0 172.16.12.2

Once those two routes were added, full end to end
communication across all five hops was restored immediately.

**Key rule learned:**
> A hub router using a default route toward HQ must still
> have specific static routes back to its own branches.
> Otherwise return traffic has no path and communication
> is one-way only.

---

## IP Address Table

### Router-to-Router Links

| Link | Network | Router A | Interface | IP Address | Router B | Interface | IP Address |
|---|---|---|---|---|---|---|---|
| HQ-R — HUB-A | 172.16.1.0/30 | HQ-R | e0/1 | 172.16.1.1 | HUB-A | e0/1 | 172.16.1.2 |
| HQ-R — HUB-B | 172.16.2.0/30 | HQ-R | e0/2 | 172.16.2.1 | HUB-B | e0/2 | 172.16.2.2 |
| HUB-A — BR-FSD | 172.16.11.0/30 | HUB-A | e0/2 | 172.16.11.1 | BR-FSD | e0/2 | 172.16.11.2 |
| HUB-A — BR-MUL | 172.16.12.0/30 | HUB-A | e0/3 | 172.16.12.1 | BR-MUL | e0/3 | 172.16.12.2 |
| HUB-B — BR-ISL | 172.16.21.0/30 | HUB-B | e0/3 | 172.16.21.1 | BR-ISL | e0/3 | 172.16.21.2 |
| HUB-B — BR-QUET | 172.16.22.0/30 | HUB-B | e0/1 | 172.16.22.1 | BR-QUET | e0/1 | 172.16.22.2 |

### LAN Networks

| Device | Interface | IP Address | Subnet | Default Gateway |
|---|---|---|---|---|
| BR-FSD | e0/1 | 192.168.1.100 | 255.255.255.0 | — |
| VPC1 | eth0 | 192.168.1.1 | 255.255.255.0 | 192.168.1.100 |
| VPC2 | eth0 | 192.168.1.2 | 255.255.255.0 | 192.168.1.100 |
| BR-MUL | e0/1 | 192.168.2.100 | 255.255.255.0 | — |
| VPC3 | eth0 | 192.168.2.1 | 255.255.255.0 | 192.168.2.100 |
| VPC4 | eth0 | 192.168.2.2 | 255.255.255.0 | 192.168.2.100 |
| BR-ISL | e0/1 | 192.168.3.100 | 255.255.255.0 | — |
| VPC5 | eth0 | 192.168.3.1 | 255.255.255.0 | 192.168.3.100 |
| VPC6 | eth0 | 192.168.3.2 | 255.255.255.0 | 192.168.3.100 |
| BR-QUET | e0/2 | 192.168.4.100 | 255.255.255.0 | — |
| VPC7 | eth0 | 192.168.4.1 | 255.255.255.0 | 192.168.4.100 |
| VPC8 | eth0 | 192.168.4.2 | 255.255.255.0 | 192.168.4.100 |

### Loopback Interfaces

| Router | Interface | IP Address | Network |
|---|---|---|---|
| HQ-R | Loopback0 | 10.0.0.1 | 10.0.0.0/24 |
| HUB-A | Loopback0 | 10.1.0.1 | 10.1.0.0/24 |
| HUB-B | Loopback0 | 10.2.0.1 | 10.2.0.0/24 |
| BR-FSD | Loopback0 | 10.1.1.1 | 10.1.1.0/24 |
| BR-MUL | Loopback0 | 10.1.2.1 | 10.1.2.0/24 |
| BR-ISL | Loopback0 | 10.1.3.1 | 10.1.3.0/24 |
| BR-QUET | Loopback0 | 10.1.4.1 | 10.1.4.0/24 |

---

## Verification

<img width="1261" height="287" alt="Ping PC1 to PC8" src="https://github.com/user-attachments/assets/591a72ba-f11e-4d79-a1a5-e2baa2f1b27a" /><br>
> Ping from VPC1 in Faisalabad to VPC8 in Quetta — full
> end to end connectivity confirmed across five router hops

<img width="1680" height="296" alt="Traceroute PC1 to PC8" src="https://github.com/user-attachments/assets/4f46144e-5b01-4315-9b4e-3803572aaee5" /><br>
> Traceroute showing full path — VPC1 → BR-FSD → HUB-A →
> HQ-R → HUB-B → BR-QUET → VPC8

<img width="1343" height="410" alt="BR-FSD Routing Table" src="https://github.com/user-attachments/assets/71cbefc2-a9b7-4a88-b835-385ccbed869a" /><br>
> Branch routing table — only one default route and
> directly connected networks. No knowledge of other branches.

<img width="1369" height="591" alt="HUB-A Routing Table" src="https://github.com/user-attachments/assets/d10d039f-d12c-4c8b-b1f6-1afb92e3a887" /><br>
> HUB-A routing table — default route toward HQ plus
> specific static routes back to Faisalabad and Multan branches

<img width="1432" height="590" alt="HUB-B Routing Table" src="https://github.com/user-attachments/assets/d093d744-aba6-4dee-914e-2fc8e981e6b0" /><br>
> HUB-B routing table — default route toward HQ plus
> specific static routes back to Islamabad and Quetta branches

---

## Router Configs

| Router | Role | Config File |
|---|---|---|
| HQ-R | Headquarters — full static routes | [HQ-R.txt](HQ-R.txt) |
| HUB-A | Lahore hub — default + branch statics | [HUB-A.txt](HUB-A.txt) |
| HUB-B | Karachi hub — default + branch statics | [HUB-B.txt](HUB-B.txt) |
| BR-FSD | Faisalabad branch — default only | [BR-FSD.txt](BR-FSD.txt) |
| BR-MUL | Multan branch — default only | [BR-MUL.txt](BR-MUL.txt) |
| BR-ISL | Islamabad branch — default only | [BR-ISL.txt](BR-ISL.txt) |
| BR-QUET | Quetta branch — default only | [BR-QUET.txt](BR-QUET.txt) |
