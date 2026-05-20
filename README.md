# Networking Labs

Hands-on network engineering labs built and tested in EVE-NG 
Community Edition. Each lab is self-contained with its own 
topology, configurations, and verification steps.

---

## Environment

| Tool | Detail |
|---|---|
| Simulator | EVE-NG Community Edition |
| Host | VMware Workstation |
| Vendors | Cisco · Huawei · Juniper · Fortinet |
| Images | Cisco IOL · IOU · CSR1000v |

---

## Labs

| # | Lab | Concepts |
|---|---|---|
| 01 | Static Routing | Manual routes, subnetting, loopbacks |
| 02 | RIPv2 | Distance vector, auto-summary |
| 03 | EIGRP | DUAL algorithm, feasible successor |
| 04 | OSPF Single Area | Link state, DR/BDR |
| 05 | OSPF Multi Area | ABR, ASBR, LSA types |
| 06 | BGP | eBGP, path attributes |
| 07 | Redistribution | OSPF ↔ EIGRP, metric translation |
| 08 | Fortinet Firewall | Zones, policies, NAT |

> Table updates as labs are completed.

---

## Structure

Each lab folder contains:
- `README.md` — full lab documentation
- `topology.png` — network diagram
- `configs/` — per-device configuration files
