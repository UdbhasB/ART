# Static Routing – Instructor Cheat Sheet
> Advanced Routing | Quick Reference & Teaching Notes

---

## 1. How a Router Makes a Forwarding Decision

```
Packet arrives → Check destination IP → Look up routing table
    → Match found?
        YES → Forward out the indicated interface
        NO  → Default route exists? → Forward there
                 No default route?  → Drop packet (ICMP unreachable)
```

**Longest prefix match always wins.**
A /28 route beats a /24 route for the same destination.

---

## 2. Routing Table Entry Types

| Code | Type | Added By |
|------|------|----------|
| `C` | Connected | Interface activation (`no shutdown`) |
| `L` | Local (host route) | IOS adds automatically (IOS 15+) |
| `S` | Static | Administrator manually |
| `S*` | Static default | Admin — candidate default |
| `R` | RIP | RIP process |
| `D` | EIGRP | EIGRP process |
| `O` | OSPF | OSPF process |
| `B` | BGP | BGP process |

---

## 3. Static Route – All Four Forms

### a) Next-Hop IP (most common)
```
ip route <network> <mask> <next-hop-ip>

ip route 10.0.3.0 255.255.255.0 10.0.2.1
```
Router does a **recursive lookup** to find which interface to use.

### b) Exit Interface
```
ip route <network> <mask> <interface>

ip route 10.0.3.0 255.255.255.0 Serial0/0/0
```
No recursive lookup. Router treats route as **directly connected** (`C` behaviour). Works cleanly on point-to-point WAN links. **Avoid on Ethernet** — causes ARP flooding.

### c) Fully Specified (Next-Hop + Interface) ← Best Practice
```
ip route <network> <mask> <interface> <next-hop-ip>

ip route 10.0.3.0 255.255.255.0 GigabitEthernet0/0 10.0.2.1
```
No recursive lookup + unambiguous next-hop. Recommended on **multi-access** (Ethernet) links.

### d) Default Static Route
```
ip route 0.0.0.0 0.0.0.0 <next-hop-ip | interface>

ip route 0.0.0.0 0.0.0.0 203.0.113.1   ! toward ISP
```
Matches **any** destination when no specific route exists.
Appears in table as `S*` and sets the **Gateway of Last Resort**.

---

## 4. Administrative Distance (AD)

AD is the **trustworthiness** of a route source. Lower = more trusted.

| Source | AD |
|--------|----|
| Directly Connected | 0 |
| Static Route | 1 |
| EIGRP Summary | 5 |
| OSPF | 110 |
| RIP | 120 |
| Unknown / Unreachable | 255 |

> **Teaching point:** If two routing protocols advertise the same network, the one with lower AD wins. Static (AD 1) always beats dynamic protocols — use carefully.

### Floating Static Route
A backup route with a **manually raised AD**, used when a dynamic route disappears:
```
ip route 10.0.3.0 255.255.255.0 10.0.99.1 150
!                                           ^^^ AD raised above OSPF (110)
```
Inactive while OSPF route exists; activates automatically on failure.

---

## 5. Summary (Aggregate) Routes

Combine multiple contiguous networks into one route to reduce table size.

**Steps:**
1. List all networks in binary.
2. Find the common leftmost bits.
3. The bit boundary is your prefix length.
4. Zero out the remaining bits → summary network address.

**Example:**
```
172.16.1.0   = 10101100.00010000.00000001.00000000
172.16.2.0   = 10101100.00010000.00000010.00000000
172.16.3.0   = 10101100.00010000.00000011.00000000
                                        ^^
                          common up to bit 22

Summary: 172.16.0.0 /22  (mask 255.255.252.0)
```

```
ip route 172.16.0.0 255.255.252.0 192.168.1.2
```

> ⚠️ A summary route can accidentally include **unintended** networks. Always verify the range covered.

---

## 6. Route Verification Commands

```bash
show ip route                        # Full routing table
show ip route static                 # Only static entries
show ip route 10.0.3.0               # Specific network lookup
show ip interface brief              # Interface status summary
ping <ip>                            # Basic reachability test
ping <ip> source <interface/ip>      # Ping from specific source
traceroute <ip>                      # Hop-by-hop path
debug ip routing                     # Live route add/remove events
undebug all                          # Turn off all debug
```

---

## 7. Common Mistakes & Fixes

| Mistake | Symptom | Fix |
|---------|---------|-----|
| Forgot `no shutdown` | Interface down, no route in table | `no shutdown` on interface |
| Missing **return** route | One-way ping only | Add static route on remote router |
| Exit interface on Ethernet | ARP issues, CEF drops | Use fully specified form instead |
| Summary covers wrong range | Traffic black-holed | Recalculate prefix carefully |
| Default route on wrong router | Some paths loop | Apply default only on stub routers |
| Clock rate missing on DCE | Serial link stays down | Add `clock rate` on DCE side |

---

## 8. Static vs Dynamic Routing – When to Use What

| Criteria | Static | Dynamic |
|----------|--------|---------|
| Network size | Small / predictable | Large / changing |
| Admin overhead | High (manual updates) | Low (auto-converges) |
| Convergence on failure | None (manual fix needed) | Automatic |
| Security | No routing updates to intercept | Needs authentication |
| Bandwidth use | Zero | Small (hello/update packets) |
| Typical use case | Stub networks, default routes, ISP links | Enterprise core, campus |

---

## 9. Key Concepts for Exam / Discussion

**Recursive Lookup**
When a next-hop IP is specified, the router looks up that IP in the table again to find the exit interface. Adds a tiny processing overhead.

**Stub Router**
A router with only one path out. A default route is sufficient — no need to know every remote network. Simplifies the routing table significantly.

**Null0 Route (Black Hole)**
```
ip route 10.0.0.0 255.255.0.0 Null0
```
Used with summary routes to discard packets that match the summary but have no more-specific route. Prevents routing loops.

**Bidirectional Routing Requirement**
A packet needs a route **in both directions** to succeed. A one-sided route causes the ping request to arrive but the reply to be dropped silently — a common troubleshooting trap.

---

## 10. Quick Decision Tree for Configuring Static Routes

```
Is the destination reachable via a single exit point?
├── YES → Use a default route (0.0.0.0/0)
│
└── NO  → Are multiple destinations reachable via the same next-hop?
           ├── YES → Can they be summarized cleanly?
           │          ├── YES → Use a summary static route
           │          └── NO  → Use individual static routes
           │
           └── NO  → Use individual static routes per destination
```

---

*Prepared for Advanced Routing | Use alongside Packet Tracer labs for hands-on reinforcement.*
