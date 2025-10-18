---
# University / Campus Network — Cisco Packet Tracer

## Objective
Design and implement a multi-building **campus network** in Cisco Packet Tracer with VLAN segmentation, inter-VLAN routing, centralized services, and controlled internet access.  
This lab is my end-to-end design and write-up inspired by a campus networking walkthrough; the topology, addressing, and configs presented here are my own implementation choices. :contentReference[oaicite:0]{index=0}

---

## Topology Overview
Logical roles and devices:
- **Core/Distribution:** 1 multilayer (L3) switch providing SVIs and inter-VLAN routing
- **Access Layer:** 2–3 access switches (one per building/floor segment)
- **Edge:** 1 internet edge router performing NAT/PAT to the ISP cloud
- **Servers:** DHCP/DNS, Web/Portal (optional AAA/RADIUS in later iterations)
- **User Segments:** Admin/Faculty PCs & printers, Student lab PCs, Staff devices
- **Wireless (optional):** 1–2 APs mapped to a Student/Guest VLAN

Primary goals:
- Layer-2 segmentation with VLANs per role/department/building
- Centralized IP addressing (DHCP scopes per VLAN)
- Inter-VLAN routing at the L3 switch; NAT at the edge
- Clear verification and troubleshooting workflow

Optional diagram:
```

![Topology Diagram](./screenshots/campus-topology.png)

````

---

## Design Rationale

### Segmentation & VLANs
Campus networks benefit from clear trust and broadcast boundaries. One `/24` per VLAN keeps addressing readable and simplifies ACLs.

| VLAN | Name        | Subnet         | Gateway (SVI) | Notes                          |
|-----:|-------------|----------------|---------------|--------------------------------|
| 10   | Admin       | 10.20.10.0/24  | 10.20.10.1    | Faculty/IT ops                 |
| 20   | Staff       | 10.20.20.0/24  | 10.20.20.1    | Department staff               |
| 30   | Students    | 10.20.30.0/24  | 10.20.30.1    | Labs & classrooms              |
| 40   | Servers     | 10.20.40.0/24  | 10.20.40.1    | DHCP/DNS/Web                   |
| 50   | Guest Wi-Fi | 10.20.50.0/24  | 10.20.50.1    | Internet-only (restricted)     |
| 99   | Native/Mgmt | 10.20.99.0/24  | 10.20.99.1    | Switch mgmt (optional)         |

**Routing model:** SVIs on the multilayer switch (`ip routing` enabled).  
**NAT model:** PAT on the edge router toward the ISP.

### Addressing & Services
- **DHCP:** Centralized on the server (10.20.40.10), with `ip helper-address` on each SVI.
- **DNS:** Local resolver (10.20.40.10) or public DNS for Guests.
- **Static reservations:** Printers/APs within `.10–.30` of each VLAN.
- **Security posture:** Guests may reach only the internet and optionally the campus web portal; no east-west access to Admin/Staff/Servers.

---

## Configuration Overview (Key Snippets)

### Distribution (L3) Switch — SVIs, Routing, Trunks
```text
conf t
ip routing
vlan 10,20,30,40,50,99

interface vlan 10
 ip address 10.20.10.1 255.255.255.0
 ip helper-address 10.20.40.10
 no shut

interface vlan 20
 ip address 10.20.20.1 255.255.255.0
 ip helper-address 10.20.40.10
 no shut

interface vlan 30
 ip address 10.20.30.1 255.255.255.0
 ip helper-address 10.20.40.10
 no shut

interface vlan 40
 ip address 10.20.40.1 255.255.255.0
 no shut

interface vlan 50
 ip address 10.20.50.1 255.255.255.0
 ip helper-address 10.20.40.10
 no shut

! Uplink to edge router (routed port)
interface g1/0/24
 description Uplink-to-Edge
 no switchport
 ip address 192.0.2.2 255.255.255.252
 no shut

! Trunks to access switches
interface g1/0/1
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30,40,50,99
````

### Edge Router — NAT & Default Route

```text
conf t
interface g0/0
 description LAN-to-Distribution
 ip address 192.0.2.1 255.255.255.252
 ip nat inside
 no shut

interface g0/1
 description WAN-to-ISP
 ip address 203.0.113.2 255.255.255.252
 ip nat outside
 no shut

ip access-list standard CAMPUS_INSIDE
 permit 10.20.0.0 0.0.255.255

ip nat inside source list CAMPUS_INSIDE interface g0/1 overload
ip route 0.0.0.0 0.0.0.0 203.0.113.1
```

### Access Switch — VLAN Access + Trunk Upstream

```text
conf t
vlan 10,20,30,40,50,99

! Example lab PCs on VLAN 30
interface range f0/1 - 12
 switchport mode access
 switchport access vlan 30

! Printers on Staff VLAN 20
interface range f0/13 - 14
 switchport mode access
 switchport access vlan 20

! Trunk to distribution
interface g0/1
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30,40,50,99
```

### Optional ACLs — Restrict Guest VLAN

```text
ip access-list extended GUEST_EGRESS
 remark Allow DNS to campus server
 permit udp 10.20.50.0 0.0.0.255 host 10.20.40.10 eq 53
 remark Allow HTTP/HTTPS to campus portal
 permit tcp 10.20.50.0 0.0.0.255 host 10.20.40.20 eq 80
 permit tcp 10.20.50.0 0.0.0.255 host 10.20.40.20 eq 443
 remark Deny guest to protected VLANs
 deny ip 10.20.50.0 0.0.0.255 10.20.10.0 0.0.0.255
 deny ip 10.20.50.0 0.0.0.255 10.20.20.0 0.0.0.255
 deny ip 10.20.50.0 0.0.0.255 10.20.40.0 0.0.0.255
 permit ip 10.20.50.0 0.0.0.255 any
!
interface vlan 50
 ip access-group GUEST_EGRESS in
```

---

## Host IP Configuration (Packet Tracer)

Configure PCs via **Desktop → IP Configuration**; printers via **Config → Settings**. Prefer DHCP on user VLANs.

* **Admin (10):** DHCP pool `10.20.10.100–200`, gateway `10.20.10.1`
* **Staff (20):** DHCP pool `10.20.20.100–200`, gateway `10.20.20.1`
* **Students (30):** DHCP pool `10.20.30.100–220`, gateway `10.20.30.1`
* **Servers (40):** Static (e.g., DHCP/DNS `10.20.40.10`, Portal `10.20.40.20`)
* **Guest Wi-Fi (50):** DHCP pool `10.20.50.50–250`, gateway `10.20.50.1`
* **Mgmt (99):** Static for switches/APs as required

If DHCP is on the server at `10.20.40.10`, ensure each SVI uses `ip helper-address 10.20.40.10`.

---

## Verification

From representative hosts:

```text
# Student PC -> Internet (via NAT)
ping 8.8.8.8       (or use simulated ISP IP)
# Guest PC -> campus portal (allowed), -> Admin host (blocked)
ping 10.20.40.20
ping 10.20.10.50
```

On distribution switch:

```text
show ip interface brief
show vlan brief
show ip route
show run | section interface Vlan
```

On edge router:

```text
show ip nat translations
show ip route
```

Simulation mode:

* Observe DHCP DORA, DNS queries, and ICMP flows. Confirm ACL behavior for Guests.

---

## Troubleshooting Notes

* **No DHCP leases:** Verify `ip helper-address` and server pools/exclusions.
* **Guest can see Admin:** Recheck ACL direction and SVI binding; confirm access ports are correctly assigned.
* **No internet:** Verify NAT inside/outside roles and default route to ISP.

---

## Files Included

* `campus-network.pkt` — Packet Tracer topology
* `screenshots/` — CLI/verification captures
* `ip-plan.xlsx` — Addressing & reservations

---

## Skills Demonstrated

* Campus segmentation with VLANs and SVIs
* Centralized DHCP/DNS, `ip helper-address`
* Edge NAT/PAT and default routing
* Access-layer port/VLAN design and trunks
* ACL policy for guest isolation
* Structured verification and fault isolation

---

## Summary

This campus build models a realistic university network with clean VLAN boundaries, centralized services, and controlled guest access. The design is intentionally modular so it can scale to more buildings, add redundancy (HSRP/VRRP), and introduce dynamic routing (OSPF) in future iterations.

---
