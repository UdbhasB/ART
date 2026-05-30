# Basic Static Route Configuration – Lab Walkthrough

## Topology Overview

Three routers (R1, R2, R3) connected via serial WAN links, each with a LAN hosting one PC.

| Device | Interface | IP Address      | Subnet Mask     |
|--------|-----------|-----------------|-----------------|
| R1     | Fa0/0     | 172.16.3.1      | 255.255.255.0   |
| R1     | S0/0/0    | 172.16.2.1      | 255.255.255.0   |
| R2     | Fa0/0     | 172.16.1.1      | 255.255.255.0   |
| R2     | S0/0/0    | 172.16.2.2      | 255.255.255.0   |
| R2     | S0/0/1    | 192.168.1.2     | 255.255.255.0   |
| R3     | Fa0/0     | 192.168.2.1     | 255.255.255.0   |
| R3     | S0/0/1    | 192.168.1.1     | 255.255.255.0   |
| PC1    | NIC       | 172.16.3.10     | GW: 172.16.3.1  |
| PC2    | NIC       | 172.16.1.10     | GW: 172.16.1.1  |
| PC3    | NIC       | 192.168.2.10    | GW: 192.168.2.1 |

---

## Task 1 – Basic Router Setup

On each router, erase old config and apply base settings:

```
erase startup-config
reload

hostname R1
no ip domain-lookup
enable secret class

line console 0
 password cisco
 login

line vty 0 4
 password cisco
 login
```

---

## Task 2 – Configure Interfaces

Configure and activate each interface per the addressing table. On DCE serial interfaces, set the clock rate.

```
! Example on R1
interface fa0/0
 ip address 172.16.3.1 255.255.255.0
 no shutdown

interface s0/0/0
 ip address 172.16.2.1 255.255.255.0
 clock rate 64000        ! DCE side only
 no shutdown
```

Repeat for R2 and R3 using their respective addresses. R2's S0/0/1 is the DCE side toward R3.

Verify with:
```
show ip interface brief
show ip route
```

At this point, each router only knows its **directly connected** networks — cross-router pings will fail.

---

## Task 3 – Static Route with Next-Hop Address

Syntax:
```
ip route <destination-network> <mask> <next-hop-ip>
```

**R3** — add route to R2's LAN:
```
ip route 172.16.1.0 255.255.255.0 192.168.1.2
```

**R2** — add return route to R3's LAN:
```
ip route 192.168.2.0 255.255.255.0 192.168.1.1
```

> PC3 ↔ PC2 pings should now succeed.

---

## Task 4 – Static Route with Exit Interface

Syntax:
```
ip route <destination-network> <mask> <exit-interface>
```

**R3** — route to R1–R2 WAN:
```
ip route 172.16.2.0 255.255.255.0 serial0/0/1
```

**R2** — route to R1's LAN:
```
ip route 172.16.3.0 255.255.255.0 serial0/0/0
```

---

## Task 5 – Default Static Route on R1

R1 is a stub router; all unknown traffic should go to R2.

```
ip route 0.0.0.0 0.0.0.0 172.16.2.2
```

Verify:
```
show ip route
! You should see: Gateway of last resort is 172.16.2.2
```

> PC2 ↔ PC1 pings should now succeed.

---

## Task 6 – Summary Static Route on R3

Instead of separate routes for 172.16.1.0, 172.16.2.0, and 172.16.3.0, summarize them:

```
172.16.1.0, 172.16.2.0, 172.16.3.0  →  common prefix: 172.16.0.0/22
```

```
! Add summary route
ip route 172.16.0.0 255.255.252.0 192.168.1.2

! Remove old specific routes
no ip route 172.16.1.0 255.255.255.0 192.168.1.2
no ip route 172.16.2.0 255.255.255.0 serial0/0/1
```

> PC3 ↔ PC1 pings should now succeed.

---

## Final Connectivity Check

| Ping          | Expected Result |
|---------------|-----------------|
| PC1 → PC2     | ✅ Success       |
| PC2 → PC3     | ✅ Success       |
| PC1 → PC3     | ✅ Success       |

---

## Key Takeaways

- Routers only know **directly connected** networks by default.
- **Static routes** must be added manually for non-connected networks.
- A **default route** (`0.0.0.0/0`) catches all unmatched destinations — useful for stub routers.
- A **summary route** consolidates multiple routes into one, keeping the routing table smaller.
- Both next-hop IP and exit interface can be used in static routes; next-hop is more common on WAN links.
