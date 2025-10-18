---
# Hotel Management Network — Cisco Packet Tracer

## Objective
Design and implement a multi-VLAN hotel network in Cisco Packet Tracer that separates business-critical services from guest traffic, supports both wired and wireless access, and enforces basic security policies. The topology models a small hotel property with reception, admin/back office, guest Wi-Fi, CCTV, and a server room.  
_Inspiration: a Packet Tracer “Hotel Management” walkthrough; this repo version is my own end-to-end design, configuration, and documentation._ :contentReference[oaicite:0]{index=0}

---

## Topology Overview
Functional areas and devices:
- **Reception VLAN**: 2 PCs, 1 network printer, desk IP phone (optional)
- **Admin/Back Office VLAN**: 3 PCs, finance printer
- **Rooms/Staff VLAN**: 2 PCs (housekeeping/maintenance)
- **Guest Wi-Fi VLAN**: 2× APs (connected to access switches)
- **CCTV VLAN**: 1 NVR + 2 IP cameras
- **Server VLAN**: 1 DHCP/DNS server, 1 Web/Booking server
- **Edge/Internet**: 1 router performing NAT to ISP cloud
- **Switching**: 1 L3-capable distribution switch (or router-on-a-stick) + 2 access switches

Primary goals:
- Traffic isolation with VLANs
- Centralized IP addressing (DHCP per VLAN)
- Guest Wi-Fi internet access without lateral visibility into business networks
- Deterministic verification and documented troubleshooting

Optional diagram:
```

![Topology Diagram](./screenshots/hotel-topology.png)

````

---

## Design Rationale

### Segmentation
Separate broadcast and trust domains to reduce lateral risk and simplify policy:
- Business ops: **Admin**, **Reception**, **Servers**
- Untrusted/IoT: **Guests**, **CCTV**
- Operational users: **Rooms/Staff**

### Routing Model
Two supported variants:
- **Router-on-a-Stick**: One router subinterface per VLAN (simple labs, fewer devices)
- **Multilayer Switch (SVIs)**: Inter-VLAN routing at the distribution switch (scalable, realistic)

This build uses **SVIs on a multilayer switch** for inter-VLAN routing, with a separate **edge router** for NAT.

### Services
- **DHCP**: Centralized scopes per VLAN (on server or the L3 switch). In this lab, DHCP runs on the **Server**; SVIs use `ip helper-address`.
- **DNS**: Local for internal name resolution; guests can also use it or be pointed to public DNS.
- **Web**: Internal booking/test site.
- **NAT**: PAT (overload) at the edge to reach the internet.

### Security/Policy
- **ACL** to block Guest VLAN access to Admin/Reception/Servers, except DNS and HTTP(S) to the booking site if required.
- **CCTV VLAN** isolated from Guests and PCs; allow NVR management from Admin only.
- **WPA2-PSK** on APs for basic guest access control (enterprise auth is out of scope for this lab).

---

## Addressing Plan

Base: `10.10.0.0/16` (ample room). One `/24` per VLAN for readability.

| VLAN | Name        | Subnet           | Gateway (SVI)     | Notes                          |
|-----:|-------------|------------------|-------------------|--------------------------------|
| 10   | Admin       | 10.10.10.0/24    | 10.10.10.1        | Back office, finance           |
| 20   | Reception   | 10.10.20.0/24    | 10.10.20.1        | Front desk                     |
| 30   | Rooms/Staff | 10.10.30.0/24    | 10.10.30.1        | Housekeeping/maintenance       |
| 40   | Guests      | 10.10.40.0/24    | 10.10.40.1        | Guest Wi-Fi SSID               |
| 50   | Servers     | 10.10.50.0/24    | 10.10.50.1        | DHCP/DNS/Web                   |
| 60   | CCTV        | 10.10.60.0/24    | 10.10.60.1        | IP cameras + NVR               |
| —    | WAN/Edge    | 203.0.113.0/30   | 203.0.113.1 (LAN) | Example public test space      |

DHCP scopes (if DHCP on server):
- Exclude `.1` (SVI) and reserve `.10-.20` for infra (APs, printers, NVR).
- Pools hand out default gateway = SVI, DNS = 10.10.50.10 (local) or public.

---

## Configuration Overview (Key Snippets)

### Distribution Switch (SVIs + Inter-VLAN Routing)
```text
conf t
ip routing

vlan 10,20,30,40,50,60

interface vlan 10
 ip address 10.10.10.1 255.255.255.0
 no shut
 ip helper-address 10.10.50.10

interface vlan 20
 ip address 10.10.20.1 255.255.255.0
 no shut
 ip helper-address 10.10.50.10

interface vlan 30
 ip address 10.10.30.1 255.255.255.0
 no shut
 ip helper-address 10.10.50.10

interface vlan 40
 ip address 10.10.40.1 255.255.255.0
 no shut
 ip helper-address 10.10.50.10

interface vlan 50
 ip address 10.10.50.1 255.255.255.0
 no shut

interface vlan 60
 ip address 10.10.60.1 255.255.255.0
 no shut

! Trunk to access switches
interface g1/0/1
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30,40,50,60

! Uplink to edge router
interface g1/0/24
 description Uplink-to-Edge
 no switchport
 ip address 192.0.2.2 255.255.255.252
 no shut
````

### Edge Router (NAT + Default Route)

```text
conf t
interface g0/0
 description LAN-to-Distribution
 ip address 192.0.2.1 255.255.255.252
 no shut

interface g0/1
 description WAN-to-ISP
 ip address 203.0.113.2 255.255.255.252
 no shut

ip access-list standard NAT_INSIDE
 permit 10.10.0.0 0.0.255.255

ip nat inside source list NAT_INSIDE interface g0/1 overload

interface g0/0
 ip nat inside
interface g0/1
 ip nat outside

ip route 0.0.0.0 0.0.0.0 203.0.113.1
```

### Access Switches (VLAN Access + Trunks)

```text
conf t
vlan 10,20,30,40,50,60

! Example: reception PCs/printer ports
interface range f0/1 - 3
 switchport mode access
 switchport access vlan 20

! Trunk up to distribution switch
interface g0/1
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30,40,50,60
```

### Wireless APs (Guest SSID)

* SSID: `HotelGuest`
* VLAN: `40`
* Security: WPA2-PSK (demo environment)
* AP management IP: from VLAN 40 infra range (e.g., 10.10.40.10)

### Basic ACL Examples (Applied on Distribution or Edge)

* Deny guests to business VLANS; allow DNS, HTTP(S) to booking host:

```text
ip access-list extended GUEST_EGRESS
 remark Allow DNS to local server
 permit udp 10.10.40.0 0.0.0.255 host 10.10.50.10 eq 53
 remark Allow HTTP/HTTPS to booking server
 permit tcp 10.10.40.0 0.0.0.255 host 10.10.50.20 eq 80
 permit tcp 10.10.40.0 0.0.0.255 host 10.10.50.20 eq 443
 remark Block guests to business VLANs
 deny ip 10.10.40.0 0.0.0.255 10.10.10.0 0.0.0.255
 deny ip 10.10.40.0 0.0.0.255 10.10.20.0 0.0.0.255
 deny ip 10.10.40.0 0.0.0.255 10.10.50.0 0.0.0.255
 deny ip 10.10.40.0 0.0.0.255 10.10.60.0 0.0.0.255
 remark Permit guests to internet (default allow to NAT/edge)
 permit ip 10.10.40.0 0.0.0.255 any
!
interface vlan 40
 ip access-group GUEST_EGRESS in
```

---

## Host IP Configuration (Packet Tracer)

Configure via PC **Desktop → IP Configuration**; Printers via **Config → Settings**. DHCP recommended for user subnets; static for servers/AP/NVR.

* **Admin (10)**: DHCP pool `10.10.10.100–200`, gateway `10.10.10.1`
* **Reception (20)**: DHCP pool `10.10.20.100–200`, gateway `10.10.20.1`
* **Rooms/Staff (30)**: DHCP pool `10.10.30.100–200`, gateway `10.10.30.1`
* **Guests (40)**: DHCP pool `10.10.40.100–250`, gateway `10.10.40.1`
* **Servers (50)**: Static (e.g., DHCP/DNS `10.10.50.10`, Web `10.10.50.20`)
* **CCTV (60)**: Static for cameras/NVR (e.g., cams `10.10.60.11/12`, NVR `10.10.60.10`)

If DHCP is on the **Server** at `10.10.50.10`, ensure each SVI has `ip helper-address 10.10.50.10`.

---

## Verification

From representative hosts:

```text
# Business PC -> booking server
ping 10.10.50.20

# Guest PC -> booking server (should work), -> Admin PC (should fail)
ping 10.10.50.20
ping 10.10.10.50

# NVR -> cameras
ping 10.10.60.11
ping 10.10.60.12
```

On the distribution switch:

```text
show ip interface brief
show vlan brief
show ip route
show run | section interface Vlan
```

On the edge router:

```text
show ip nat translations
show ip route
```

Simulation mode checks:

* Filter ICMP/DNS/HTTP; confirm ARP, DHCP DISCOVER/OFFER/REQUEST/ACK, DNS query/response, and HTTP flows.

---

## Troubleshooting Notes

* **No DHCP leases**: Confirm `ip helper-address` on each SVI; verify DHCP pools and exclusions.
* **Guest sees business hosts**: Re-check ACL direction and interface binding; confirm no mis-tagged access ports.
* **NAT not translating**: Verify inside/outside interface roles and that ACL `NAT_INSIDE` matches 10.10.0.0/16.
* **AP VLAN**: Ensure AP uplink is a trunk carrying VLAN 40; set AP management IP in VLAN 40.

---

## Files Included

* `hotel-management-network.pkt` — Packet Tracer topology
* `screenshots/` — CLI outputs, topology images
* `ip-plan.xlsx` — Addressing plan and reservations

---

## Skills Demonstrated

* VLAN design and inter-VLAN routing (SVIs)
* Centralized DHCP/DNS with IP helpers
* Edge NAT/PAT for internet egress
* Basic policy enforcement with ACLs
* Wireless AP integration with VLAN tagging
* Structured verification and fault isolation

---

## Summary

This lab demonstrates a practical hotel network with clear segmentation, centralized services, wireless guest access, and foundational security controls. The configuration is intentionally modular so it can scale to additional floors, VLANs, redundancy, and dynamic routing in future iterations.

---