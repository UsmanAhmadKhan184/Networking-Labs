# Lab 01 — Static Routing

## What I Built
Designed and deployed a 10-router network topology in EVE-NG, manually configuring static routes to achieve full connectivity
between multiple loopback interfaces and VPCs, and verified communication using ping and traceroute testing.

## Topology
<img width="1850" height="870" alt="Static-Routing-lab-1" src="https://github.com/user-attachments/assets/d3a7fc52-ab4b-4cd2-b03a-c31cc642a6fb" />

## What I Learned

Before starting this lab I had no idea what a loopback interface
was or why it existed. After building this topology I understood
that a loopback is a virtual interface that acts as the permanent
identity of a router. Unlike physical interfaces which go down
when a cable is unplugged or a link fails, the loopback stays
alive as long as the router itself is running. This is considered
a best practice in real networks because it gives every router a
stable IP that can always be used for management and routing
protocol configuration.

The second thing I learned is the real difference between static
routing and dynamic routing. With static routing everything is
in your hands, you manually tell each router exactly where to
send packets. This gives you full control and visibility over
every path in the network. However it also means if a link goes
down the router has no way to automatically find another path.
The manually configured route simply stops working until someone
fixes it. In a large network like this 10 router topology, writing
routes on every single router was repetitive and easy to make
mistakes on.

One mistake I actually made was forgetting that routing works in
both directions. I configured routes from R1 toward the far routers
but forgot that the far routers also needed routes back to R1.
This caused one-way communication where ping requests were sent
but replies never came back. After adding the return routes on
each router, full communication was established.

Static routing is best suited for small stable networks where the
topology rarely changes. For larger networks dynamic protocols like
OSPF or BGP make more sense because routers discover paths
automatically and adapt when links fail.

## IP Address Table
### Router-to-Router Links
| Link | Network | Router A | Interface | IP Address | Router B | Interface | IP Address |
|---|---|---|---|---|---|---|---|
| R1 — R2 | 172.168.12.0/30 | R1 | e0/0 | 172.168.12.1 | R2 | e0/0 | 172.168.12.2 |
| R2 — R3 | 172.168.23.0/30 | R2 | e0/1 | 172.168.23.1 | R3 | e0/1 | 172.168.23.2 |
| R1 — R4 | 172.168.10.0/30 | R1 | e0/1 | 172.168.10.1 | R4 | e0/1 | 172.168.10.2 |
| R1 — R5 | 172.168.30.0/30 | R1 | e0/2 | 172.168.30.1 | R5 | e0/2 | 172.168.30.2 |
| R2 — R6 | 172.168.19.0/30 | R2 | e0/2 | 172.168.19.1 | R6 | e0/2 | 172.168.19.2 |
| R3 — R7 | 172.168.16.0/30 | R3 | e0/0 | 172.168.16.1 | R7 | e0/1 | 172.168.16.2 |
| R5 — R6 | 172.168.2.0/30 | R5 | e0/0 | 172.168.2.1 | R6 | e0/0 | 172.168.2.2 |
| R5 — R8 | 172.168.22.0/30 | R5 | e0/1 | 172.168.22.1 | R8 | e0/1 | 172.168.22.2 |
| R5 — R9 | 172.168.25.0/30 | R5 | e1/0 | 172.168.25.1 | R9 | e1/0 | 172.168.25.2 |
| R6 — R7 | 172.168.20.0/30 | R6 | e0/1 | 172.168.20.1 | R7 | e0/1 | 172.168.20.2 |
| R6 — R9 | 172.168.29.0/30 | R6 | e0/3 | 172.168.29.1 | R9 | e0/3 | 172.168.29.2 |
| R6 — R10 | 172.168.45.0/30 | R6 | e1/0 | 172.168.45.1 | R10 | e1/0 | 172.168.45.2 |

### Router to VPC Links
| Link | Network | Device A | Interface | IP Address | Device B | Interface | IP Address |
|---|---|---|---|---|---|---|---|
| R4 — VPC2 | 192.168.20.0/30 | R4 | e0/0 | 192.168.20.1 | VPC2 | eth0 | 192.168.20.2 |
| R10 — VPC1 | 192.168.10.0/30 | R10 | e0/0 | 192.168.10.1 | VPC1 | eth0 | 192.168.10.2 |
### Loopback Interfaces
| Router | Interface | IP Address | Network |
|---|---|---|---|
| R1 | Loopback0 | 10.1.1.1 | 10.1.1.0/24 |
| R2 | Loopback0 | 10.2.2.1 | 10.2.2.0/24 |
| R3 | Loopback0 | 10.3.3.1 | 10.3.3.0/24 |
| R4 | Loopback0 | 10.4.4.1 | 10.4.4.0/24 |
| R5 | Loopback0 | 10.5.5.1 | 10.5.5.0/24 |
| R6 | Loopback0 | 10.6.6.1 | 10.6.6.0/24 |
| R7 | Loopback0 | 10.7.7.1 | 10.7.7.0/24 |
| R8 | Loopback0 | 10.8.8.1 | 10.8.8.0/24 |
| R9 | Loopback0 | 10.9.9.1 | 10.9.9.0/24 |
| R10 | Loopback0 | 10.10.10.1 | 10.10.10.0/24 |

## Verification
 <img width="1650" height="351" alt="Trace route Vpc2 to Vpc1" src="https://github.com/user-attachments/assets/de8b4b09-1926-410f-b385-226d475a324a" />
> Trace route Vpc2 to Vpc1

 <img width="643" height="190" alt="Trace route R1 to R10" src="https://github.com/user-attachments/assets/004094cc-dc8d-47af-a3f5-4c343fa0bd91" />
> Trace route R1 to R10

## Router Configs
[Router1 Config File](R1.txt)

[Router2 Config File](R2.txt)

[Router3 Config File](R3.txt)

[Router4 Config File](R4.txt)

[Router5 Config File](R5.txt)

[Router6 Config File](R6.txt)

[Router7 Config File](R7.txt)

[Router8 Config File](R8.txt)

[Router9 Config File](R9.txt)

[Router10 Config File](R10.txt)
