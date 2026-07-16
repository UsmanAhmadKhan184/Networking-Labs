# Lab 02 — IS-IS Advanced Concepts

## What I Built

A multi-area IS-IS network with four areas and a dedicated L2
backbone implementing three advanced concepts — MD5
authentication on all IS-IS links, metric manipulation to
control primary and backup paths within areas, and route
leaking from L2 into L1 so pure L1 routers can make
intelligent exit decisions instead of blindly following
the default route.

Area 1 has a dual exit design with two L1/L2 border routers
(R1 and R2) providing redundancy — if one border router fails
the other maintains full connectivity with all leaked routes
already present in the L1 database.

---

## Topology

<img width="1565" height="827" alt="IS-IS Protocol Lab-2" src="https://github.com/user-attachments/assets/dd2bbe64-19c1-4a1f-8253-d1ff7ceacab1" /><br>

---

## What I Learned

### IS-IS Authentication

Just like RIP has MD5 authentication, IS-IS supports
authentication at two separate levels — interface level
for Hello packets and domain level for LSPs. Without
authentication any device connecting to the network can
inject fake LSPs and corrupt the entire IS-IS topology
database.

**Interface authentication — Hello packets:**
Authenticates IIH packets between neighbors. If keys do
not match adjacency never forms. The failure is immediate
unlike RIP where routes disappear only after timers expire.

**Domain/Area authentication — LSPs:**
Authenticates LSPs, CSNPs, and PSNPs as they flood through
the domain. Prevents fake topology information from being
accepted even if the device managed to form a Hello adjacency.

`key chain ISIS-AUTH` \
`key 1` \
`key-string SecureISIS@2026` \
`interface Ethernet0/0` \
`isis authentication mode md5` \
`isis authentication key-chain ISIS-AUTH` \
`router isis` \
`authentication mode md5` \
`authentication key-chain ISIS-AUTH`

**Important behavior discovered:** When authentication is
applied on one interface the neighbor immediately starts
seeing authentication failures and logs them:
> %CLNS-4-AUTH_FAIL: ISIS: LAN IIH authentication failed

This error repeats every Hello interval until the matching
key is configured on the neighbor side. It is expected
behavior — not a misconfiguration on the router that applied
authentication first.

### IS-IS Metric Manipulation

IS-IS by default assigns metric 10 to every interface
regardless of bandwidth — identical to RIP's flat metric
problem. With wide metrics enabled you can assign meaningful
costs to force IS-IS to prefer specific paths.

`router isis` \
`metric-style wide` \
`interface Ethernet0/0` \
`isis metric 10`    ← primary path \
`interface Ethernet0/1` \
`isis metric 100`   ← backup path

**Narrow vs Wide metrics:**

| Type | Max per interface | Max path total |
|---|---|---|
| Narrow (default) | 63 | 1023 |
| Wide | 16777214 | 4261412864 |

Always use wide metrics in any real or lab IS-IS topology.

**Metric test performed in this lab:**

R1 normally reaches R4 through R3 with total metric 20
(10 + 10). When the R3 link was shut down R1 went directly
to R4 with metric 50 — the backup path. When R3 was
restored traffic automatically returned to the primary
path with metric 20. This mirrors the OSPF cost failover
test from the OSPF labs.

### Route Leaking (L2 into L1)

Without route leaking a pure L1 router only sees a default
route pointing to the nearest L1/L2 router. It has no
knowledge of what networks exist in other areas and cannot
make intelligent exit decisions when multiple border routers
are available.

**Before leaking — R3 routing table:**
> i*L1 0.0.0.0/0 [115/20] via R1   ← blind default only

**After leaking — R3 routing table:**
> i*L1 0.0.0.0/0 [115/20] via R1 \
> i ia 10.2.0.0/16 via R1 or R2    ← Area 2 routes \
> i ia 10.3.1.0/24 via R1 or R2    ← Area 3 LANs \
> i ia 10.4.1.0/24 via R1 or R2    ← Area 4 LANs

**Configuration issue discovered:**

The classic distribute-list syntax did not work on this
IOS version:

`redistribute isis ip level-2 into level-1 distribute-list prefix LEAK-AREA3`

After research I found that Cisco IOL does not support
the `prefix` keyword directly in the IS-IS leaking command.
The solution is to use a route-map instead:

`ip prefix-list LEAK-AREA2 permit 10.2.0.0/16 le 32` \
`ip prefix-list LEAK-AREA3 permit 10.3.0.0/16 le 32` \
`ip prefix-list LEAK-AREA4 permit 10.4.0.0/16 le 32` \
`route-map RM-LEAK-ALL permit 10` \
`match ip address prefix-list LEAK-AREA2` \
`route-map RM-LEAK-ALL permit 20` \
`match ip address prefix-list LEAK-AREA3` \
`route-map RM-LEAK-ALL permit 30` \
`match ip address prefix-list LEAK-AREA4` \
`router isis` \
`redistribute isis ip level-2 into level-1 route-map RM-LEAK-ALL`

The route-map approach is actually more powerful — it
allows setting metrics per leaked prefix and denying
specific sensitive routes before the permit sequences.

**Why dual exit areas need leaking on both border routers:**

Area 1 has two border routers R1 and R2. Leaking was
configured on both — not just one. If only R1 leaked
Area 3 routes and R1 went down, R3 would lose the
specific Area 3 routes and fall back to blind default
routing via R2. With leaking on both routers R3 always
has specific routes regardless of which border router
is active.

**The le 32 keyword explained:**

`ip prefix-list LEAK-AREA3 permit 10.3.0.0/16 le 32`

Without le 32 only the exact /16 summary matches.
With le 32 the prefix-list matches 10.3.0.0/16 AND
any more specific route within that range up to /32 —
including all the /24 LANs and /30 point-to-point links
inside Area 3.

---

## IP Address Table

### Area 1 Internal Links

| Link | Network | Router A | Interface | IP | Router B | Interface | IP |
|---|---|---|---|---|---|---|---|
| R3 — R4 | 10.1.34.0/30 | R3 | e0/1 | 10.1.34.1 | R4 | e0/3 | 10.1.34.2 |
| R3 — R1 | 10.1.13.0/30 | R3 | e0/2 | 10.1.13.2 | R1 | e0/2 | 10.1.13.1 |
| R4 — R1 | 10.1.14.0/30 | R4 | e0/3 | 10.1.14.2 | R1 | e0/3 | 10.1.14.1 |
| R4 — R2 | 10.1.24.0/30 | R4 | e0/0 | 10.1.24.2 | R2 | e0/1 | 10.1.24.1 |
| R5 — R2 | 10.1.25.0/30 | R5 | e0/1 | 10.1.25.2 | R2 | e0/2 | 10.1.25.1 |

### Area 2 Internal Links

| Link | Network | Router A | Interface | IP | Router B | Interface | IP |
|---|---|---|---|---|---|---|---|
| R7 — R8 | 10.2.64.0/30 | R7 | e0/2 | 10.2.64.1 | R8 | e0/3 | 10.2.64.2 |
| R7 — R6 | 10.2.68.0/30 | R7 | e0/2 | 10.2.68.2 | R6 | e0/2 | 10.2.68.1 |
| R8 — R6 | 10.2.67.0/30 | R8 | e0/3 | 10.2.67.2 | R6 | e0/3 | 10.2.67.1 |
| R9 — R6 | 10.2.69.0/30 | R9 | e0/0 | 10.2.69.2 | R6 | e0/0 | 10.2.69.1 |

### Area 3 Internal Links

| Link | Network | Router A | Interface | IP | Router B | Interface | IP |
|---|---|---|---|---|---|---|---|
| R10 — R11 | 10.3.74.0/30 | R10 | e0/1 | 10.3.74.1 | R11 | e0/1 | 10.3.74.2 |
| R10 — R12 | 10.3.72.0/30 | R10 | e0/2 | 10.3.72.1 | R12 | e0/2 | 10.3.72.2 |
| R11 — R12 | 10.3.78.0/30 | R11 | e0/3 | 10.3.78.1 | R12 | e0/3 | 10.3.78.2 |

### Area 4 Internal Links

| Link | Network | Router A | Interface | IP | Router B | Interface | IP |
|---|---|---|---|---|---|---|---|
| R13 — R14 | 10.4.82.0/30 | R13 | e0/3 | 10.4.82.1 | R14 | e0/2 | 10.4.82.2 |
| R13 — R15 | 10.4.85.0/30 | R13 | e0/1 | 10.4.85.1 | R15 | e0/1 | 10.4.85.2 |
| R14 — R15 | 10.4.83.0/30 | R14 | e0/2 | 10.4.83.1 | R15 | e0/2 | 10.4.83.2 |

### L2 Backbone Links

| Link | Network | Router A | Interface | IP | Router B | Interface | IP |
|---|---|---|---|---|---|---|---|
| R16 — R17 | 10.0.20.0/30 | R16 | e0/1 | 10.0.20.1 | R17 | e0/0 | 10.0.20.2 |
| R17 — R18 | 10.0.23.0/30 | R17 | e0/1 | 10.0.23.1 | R18 | e0/0 | 10.0.23.2 |

### L1/L2 to Backbone Links

| Link | Network | Router A | Interface | IP | Router B | Interface | IP |
|---|---|---|---|---|---|---|---|
| R1 — R16 | 10.0.18.0/30 | R1 | e0/0 | 10.0.18.1 | R16 | e0/0 | 10.0.18.2 |
| R2 — R16 | 10.0.28.0/30 | R2 | e0/0 | 10.0.28.1 | R16 | e0/3 | 10.0.28.2 |
| R6 — R18 | 10.0.72.0/30 | R6 | e0/1 | 10.0.72.1 | R18 | e0/1 | 10.0.72.2 |
| R10 — R16 | 10.0.10.0/30 | R10 | e0/3 | 10.0.10.1 | R16 | e0/3 | 10.0.10.2 |
| R13 — R18 | 10.0.16.0/30 | R13 | e0/2 | 10.0.16.1 | R18 | e0/2 | 10.0.16.2 |

### LAN Networks

| Device | Interface | IP Address | Subnet | Default Gateway |
|---|---|---|---|---|
| R3 | e0/0 | 10.1.1.100 | 255.255.255.0 | — |
| PC1 | eth0 | 10.1.1.1 | 255.255.255.0 | 10.1.1.100 |
| R5 | e0/0 | 10.1.2.100 | 255.255.255.0 | — |
| PC2 | eth0 | 10.1.2.1 | 255.255.255.0 | 10.1.2.100 |
| R7 | e0/0 | 10.2.1.100 | 255.255.255.0 | — |
| PC3 | eth0 | 10.2.1.1 | 255.255.255.0 | 10.2.1.100 |
| R8 | e0/0 | 10.2.2.100 | 255.255.255.0 | — |
| PC4 | eth0 | 10.2.2.1 | 255.255.255.0 | 10.2.2.100 |
| R11 | e0/0 | 10.3.1.100 | 255.255.255.0 | — |
| PC5 | eth0 | 10.3.1.1 | 255.255.255.0 | 10.3.1.100 |
| R12 | e0/0 | 10.3.2.100 | 255.255.255.0 | — |
| PC6 | eth0 | 10.3.2.1 | 255.255.255.0 | 10.3.2.100 |
| R14 | e0/0 | 10.4.1.100 | 255.255.255.0 | — |
| PC7 | eth0 | 10.4.1.1 | 255.255.255.0 | 10.4.1.100 |
| R15 | e0/0 | 10.4.2.100 | 255.255.255.0 | — |
| PC8 | eth0 | 10.4.2.1 | 255.255.255.0 | 10.4.2.100 |

### Loopback Interfaces and NET Addresses

| Router | Type | Loopback | NET Address |
|---|---|---|---|
| R1 | L1/L2 Area 1 | 1.1.1.1/32 | 49.0001.0010.0100.1001.00 |
| R2 | L1/L2 Area 1 | 2.2.2.2/32 | 49.0001.0020.0200.2002.00 |
| R3 | L1 Area 1 | 3.3.3.3/32 | 49.0001.0030.0300.3003.00 |
| R4 | L1 Area 1 | 4.4.4.4/32 | 49.0001.0040.0400.4004.00 |
| R5 | L1 Area 1 | 5.5.5.5/32 | 49.0001.0050.0500.5005.00 |
| R6 | L1/L2 Area 2 | 6.6.6.6/32 | 49.0002.0060.0600.6006.00 |
| R7 | L1 Area 2 | 7.7.7.7/32 | 49.0002.0070.0700.7007.00 |
| R8 | L1 Area 2 | 8.8.8.8/32 | 49.0002.0080.0800.8008.00 |
| R9 | L1 Area 2 | 9.9.9.9/32 | 49.0002.0090.0900.9009.00 |
| R10 | L1/L2 Area 3 | 10.10.10.10/32 | 49.0003.0100.1001.0010.00 |
| R11 | L1 Area 3 | 11.11.11.11/32 | 49.0003.0110.1101.1011.00 |
| R12 | L1/L2 Area 3 | 12.12.12.12/32 | 49.0003.0120.1201.2012.00 |
| R13 | L1/L2 Area 4 | 13.13.13.13/32 | 49.0004.0130.1301.3013.00 |
| R14 | L1 Area 4 | 14.14.14.14/32 | 49.0004.0140.1401.4014.00 |
| R15 | L1 Area 4 | 15.15.15.15/32 | 49.0004.0150.1501.5015.00 |
| R16 | L2 Backbone | 16.16.16.16/32 | 49.0000.0160.1601.6016.00 |
| R17 | L2 Backbone | 17.17.17.17/32 | 49.0000.0170.1701.7017.00 |
| R18 | L2 Backbone | 18.18.18.18/32 | 49.0000.0180.1801.8018.00 |

---

## Authentication Verification

<img width="1464" height="331" alt="Key chain Verification" src="https://github.com/user-attachments/assets/3e59f002-a59d-4393-99d4-478da868bf52" /><br>
> show key chain on R1 confirming SecureISIS@2026 is the
> active key with always valid lifetime

<img width="1640" height="371" alt="R1 Neighbors" src="https://github.com/user-attachments/assets/0ace5ab9-8a24-426d-9a1e-c9617343e8b7" /><br>
> R1 IS-IS neighbor table showing all adjacencies UP after
> authentication was matched on both sides of every link

<img width="1568" height="302" alt="Auth Failed" src="https://github.com/user-attachments/assets/abdd0164-6448-4fdd-81da-7b0e328577e0" /><br>
> %CLNS-4-AUTH_FAIL error appearing after authentication
> was applied on R1 but before neighbors had matching config.
> Expected behavior — clears automatically once all neighbors
> have the same key configured

---

## Metric Manipulation

<img width="1685" height="684" alt="R1 to R4 Path m20" src="https://github.com/user-attachments/assets/ac89bf07-dcf9-43dc-b085-9ea02119b4c1" /><br>
> show isis topology on R1 showing path to R4 through R3
> Total metric 20 = R1 to R3 (metric 10) + R3 to R4 (metric 10)

<img width="1704" height="722" alt="R1 to R4 Path m50" src="https://github.com/user-attachments/assets/1f6ad198-a30b-4eec-bea1-6b3556922cd4" /><br>
> R3 link shut down — R1 now reaches R4 directly via backup
> path with metric 50. IS-IS automatically reconverged
> to the next best available path

<img width="1454" height="635" alt="R1 to R4 Path m20 again" src="https://github.com/user-attachments/assets/536cd072-521f-41a7-9d84-8767d3c77a64" /><br>
> R3 link restored — traffic returned to primary path via R3
> with total metric 20. Automatic recovery with no manual
> intervention required.

---

## Route Leaking

<img width="1344" height="659" alt="R3 L1 Default Route" src="https://github.com/user-attachments/assets/64168a84-3059-442c-879a-ced540bba683" /><br>
> R3 (L1 only) routing table before leaking — only the
> default route is visible. R3 has no knowledge of what
> networks exist in Areas 2, 3, or 4.

<img width="776" height="659" alt="R3 L1 Route After Leaking" src="https://github.com/user-attachments/assets/e62f1843-04f5-4d9d-96a3-adc4c5426344" /><br>
> R3 routing table after leaking — specific i ia routes
> for Areas 2, 3, and 4 now visible alongside the default
> route. R3 can now choose the correct exit for each area.

<img width="662" height="184" alt="Prefix-list R1" src="https://github.com/user-attachments/assets/55c65203-fe22-49d0-a2d8-29b3e841fab7" /><br>
> Prefix lists on R1 — leaks Area 2, 3, and 4 routes
> into Area 1 for redundancy with R2

<img width="810" height="231" alt="Prefix-list R2" src="https://github.com/user-attachments/assets/dc0fee49-78dd-493b-b8c8-3be5ba4ba4dd" /><br>
> Prefix lists on R2 — same leaking as R1 ensuring full
> redundancy when either border router fails

<img width="817" height="223" alt="Prefix-list R6" src="https://github.com/user-attachments/assets/c0895ead-88a3-4fa7-8f10-9a2f542e7f69" /><br>
> Prefix lists on R6 — leaks Area 1, 3, and 4 routes
> into Area 2

<img width="837" height="219" alt="Prefix-list R10" src="https://github.com/user-attachments/assets/9c752e6c-7564-4158-af8b-1c0a330cc94f" /><br>
> Prefix lists on R10 — leaks Area 1, 2, and 4 routes
> into Area 3

<img width="833" height="220" alt="Prefix-list R13" src="https://github.com/user-attachments/assets/6741f5f2-d9a7-4a2f-b59b-78c0fe778812" /><br>
> Prefix lists on R13 — leaks Area 1, 2, and 3 routes
> into Area 4

---

## IS-IS Database

<img width="888" height="663" alt="ISIS Database R1" src="https://github.com/user-attachments/assets/b228c133-ca3b-4bed-b7ad-da72280bc27d" /><br>
> R1 (L1/L2) database showing both Level-1 and Level-2
> sections — Area 1 internal topology in L1, backbone
> and inter-area topology in L2

---

## Connectivity

<img width="1184" height="335" alt="Ping PC1 to PC4" src="https://github.com/user-attachments/assets/5b4c98d3-1be3-4cf1-b134-cd136aea1dd7" /><br>
> Ping from PC1 (Area 1) to PC4 (Area 2) — crosses
> Area 1 → backbone → Area 2

<img width="1255" height="345" alt="Ping PC1 to PC6" src="https://github.com/user-attachments/assets/8b48b7c2-7de7-4810-b170-87aee7a668be" /><br>
> Ping from PC1 (Area 1) to PC6 (Area 3) — crosses
> Area 1 → backbone → Area 3

<img width="1184" height="349" alt="Ping PC1 to PC8" src="https://github.com/user-attachments/assets/aaa42009-440e-4632-a60b-c30349c68a1b" /><br>
> Ping from PC1 (Area 1) to PC8 (Area 4) — crosses
> all areas confirming full end to end connectivity

<img width="1137" height="441" alt="Trace PC1 to PC8" src="https://github.com/user-attachments/assets/1511011a-8fb3-45d1-92ed-0b7b2f89b2c4" /><br>
> Traceroute from PC1 to PC8 showing full path through
> R3 → R1 → R16 → R17 → R18 → R13 → R15 → PC8

---

## Router Configs

| Router | Type | Area | Config |
|---|---|---|---|
| R1 | L1/L2 border | Area 1 | [R1-conf.txt](R1-conf.txt) |
| R2 | L1/L2 border | Area 1 | [R2-conf.txt](R2-conf.txt) |
| R3 | L1 only | Area 1 | [R3-conf.txt](R3-conf.txt) |
| R4 | L1 only | Area 1 | [R4-conf.txt](R4-conf.txt) |
| R5 | L1 only | Area 1 | [R5-conf.txt](R5-conf.txt) |
| R6 | L1/L2 border | Area 2 | [R6-conf.txt](R6-conf.txt) |
| R7 | L1 only | Area 2 | [R7-conf.txt](R7-conf.txt) |
| R8 | L1 only | Area 2 | [R8-conf.txt](R8-conf.txt) |
| R9 | L1 only | Area 2 | [R9-conf.txt](R9-conf.txt) |
| R10 | L1/L2 border | Area 3 | [R10-conf.txt](R10-conf.txt) |
| R11 | L1 only | Area 3 | [R11-conf.txt](R11-conf.txt) |
| R12 | L1 only | Area 3 | [R12-conf.txt](R12-conf.txt) |
| R13 | L1/L2 border | Area 4 | [R13-conf.txt](R13-conf.txt) |
| R14 | L1 only | Area 4 | [R14-conf.txt](R14-conf.txt) |
| R15 | L1 only | Area 4 | [R15-conf.txt](R15-conf.txt) |
| R16 | L2 only | Backbone | [R16-conf.txt](R16-conf.txt) |
| R17 | L2 only | Backbone | [R17-conf.txt](R17-conf.txt) |
| R18 | L2 only | Backbone | [R18-conf.txt](R18-conf.txt) |
