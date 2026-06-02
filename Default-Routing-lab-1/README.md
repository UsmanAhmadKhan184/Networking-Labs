# Lab 01 — Default Routing (Real World Scenario)

## What I Built

A default routing lab based on a real world scenario with one
headquarters and four branch offices — Karachi, Lahore,
Islamabad and Peshawar. Each branch router is configured with
a single default route pointing toward HQ. The HQ router uses
full static routes to reach all branch networks because it needs
to know exactly where to deliver traffic coming from every branch.

---

## Topology

<img width="1548" height="924" alt="Default-Routing-lab-1" src="https://github.com/user-attachments/assets/29eb644d-1dba-477e-9ec3-253d8f5c5433" /><br>

---

## What I Learned

**What default routing actually means**

A default route tells a router — if you do not have a specific
route for this packet, send it here. It does not mean the
destination decides where to send the packet. The branch router
itself makes the decision — it just uses one catch-all rule
instead of dozens of specific rules.

Think of it like this: a branch employee does not need to know
the address of every other office in the company. They just send
everything to headquarters and HQ figures out the rest.

**Why branch routers use default routing but HQ does not**

Branch routers only need to reach networks outside their own
office — and all of those are reachable through HQ. So one
default route covers everything.

HQ however receives traffic from all four branches and needs to
deliver it to the correct destination. If HQ also used a default
route it would not know which branch to forward the packet to.
HQ must have a specific static route for every branch network.

**The routing table difference**

Branch router (KHI-R) — just two lines that matter:

S*  0.0.0.0/0 via 172.168.30.2     ← default route — everything unknown goes to HQ \
C   192.168.3.0/24 connected        ← own LAN

HQ router — full knowledge of every branch:

S   192.168.1.0/24 via 172.168.10.2   ← LHR branch \
S   192.168.2.0/24 via 172.168.20.2   ← ISL branch \
S   192.168.3.0/24 via 172.168.30.2   ← KHI branch \
S   192.168.4.0/24 via 172.168.40.2   ← PSH branch 

**In real networks this scales massively**

Imagine 50 branch offices. Each branch still needs just one
default route. Only HQ grows in complexity. This is exactly
how enterprise hub-and-spoke networks are designed in real life.

---

## IP Address Table

### Router-to-Router Links

| Link | Network | Router A | Interface | IP Address | Router B | Interface | IP Address |
|---|---|---|---|---|---|---|---|
| HQ-R — KHI-R | 172.168.30.0/30 | HQ-R | e0/2 | 172.168.30.2 | KHI-R | e0/2 | 172.168.30.1 |
| HQ-R — LHR-R | 172.168.10.0/30 | HQ-R | e0/3 | 172.168.10.2 | LHR-R | e0/3 | 172.168.10.1 |
| HQ-R — ISL-R | 172.168.20.0/30 | HQ-R | e0/1 | 172.168.20.2 | ISL-R | e0/1 | 172.168.20.1 |
| HQ-R — PSH-R | 172.168.40.0/30 | HQ-R | e1/0 | 172.168.40.2 | PSH-R | e1/0 | 172.168.40.1 |

### LAN Networks

| Device | Interface | IP Address | Subnet | Default Gateway |
|---|---|---|---|---|
| KHI-R | e0/0 | 192.168.3.100 | 255.255.255.0 | — |
| KHI-PC1 | eth0 | 192.168.3.1 | 255.255.255.0 | 192.168.3.100 |
| KHI-PC2 | eth0 | 192.168.3.2 | 255.255.255.0 | 192.168.3.100 |
| LHR-R | e0/0 | 192.168.1.100 | 255.255.255.0 | — |
| LHR-PC1 | eth0 | 192.168.1.1 | 255.255.255.0 | 192.168.1.100 |
| LHR-PC2 | eth0 | 192.168.1.2 | 255.255.255.0 | 192.168.1.100 |
| ISL-R | e0/0 | 192.168.2.100 | 255.255.255.0 | — |
| ISL-PC1 | eth0 | 192.168.2.1 | 255.255.255.0 | 192.168.2.100 |
| ISL-PC2 | eth0 | 192.168.2.2 | 255.255.255.0 | 192.168.2.100 |
| PSH-R | e0/0 | 192.168.4.100 | 255.255.255.0 | — |
| PSH-PC1 | eth0 | 192.168.4.1 | 255.255.255.0 | 192.168.4.100 |
| PSH-PC2 | eth0 | 192.168.4.2 | 255.255.255.0 | 192.168.4.100 |

### Loopback Interfaces

| Router | Interface | IP Address | Network |
|---|---|---|---|
| HQ-R | Loopback0 | 10.50.50.1 | 10.50.50.0/24 |
| KHI-R | Loopback0 | 10.30.30.1 | 10.30.30.0/24 |
| LHR-R | Loopback0 | 10.10.10.1 | 10.10.10.0/24 |
| ISL-R | Loopback0 | 10.20.20.1 | 10.20.20.0/24 |
| PSH-R | Loopback0 | 10.40.40.1 | 10.40.40.0/24 |

---

## Verification

<img width="1278" height="287" alt="Ping KHI-PC to PSH-PC" src="https://github.com/user-attachments/assets/1dfe61d9-b831-42e8-8d40-9de02fdea4ab" /><br>
> Ping from KHI-PC1 to PSH-PC1 — confirmed full end to end
> connectivity across all four branches through HQ

<img width="1684" height="259" alt="TraceRoute KHI-PC to PSH-PC" src="https://github.com/user-attachments/assets/97ad82b7-84a0-4d8d-89d0-5342dee6d566" /><br>
> Traceroute shows exact path — KHI-PC1 → KHI-R → HQ-R →
> PSH-R → PSH-PC1. All traffic passes through HQ as expected.

<img width="1185" height="611" alt="IP Route HQ-R" src="https://github.com/user-attachments/assets/29b5cd9f-35cc-434c-a207-812b194f90ec" /><br>
> HQ routing table — contains specific static routes to all
> four branch LANs and loopbacks

<img width="1378" height="383" alt="IP Route KHI-R" src="https://github.com/user-attachments/assets/4b66f012-776b-4b7f-b0c1-5445320f9880" /><br>
> KHI branch routing table — contains only one default route
> pointing to HQ. No knowledge of other branches needed.

---

## Router Configs

| Router | Role | Config File |
|---|---|---|
| HQ-R | Headquarters — full static routes | [HQ-R.txt](HQ-R.txt) |
| KHI-R | Karachi branch — default route only | [KHI-R.txt](KHI-R.txt) |
| LHR-R | Lahore branch — default route only | [LHR-R.txt](LHR-R.txt) |
| ISL-R | Islamabad branch — default route only | [ISL-R.txt](ISL-R.txt) |
| PSH-R | Peshawar branch — default route only | [PSH-R.txt](PSH-R.txt) |
