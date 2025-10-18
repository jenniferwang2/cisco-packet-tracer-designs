---

# Branch Office VLAN Network with DHCP, Wireless, and Inter-VLAN Routing

## Objective

I designed this network to simulate a **branch office setup for a fast-growing company** with three functional departments operating independently but requiring interconnectivity. This project builds on fundamental concepts such as **VLAN segmentation, subnetting, inter-VLAN routing, DHCP services, and wireless integration**, making it a realistic and well-rounded implementation scenario for a small office LAN.

---

## Topology Overview

The network simulates a branch office containing:

- **1 Router (Cisco 2911)**
- **1 Switch (Cisco 2960)**
- **3 Departments**, each with:
  - 1 PC
  - 1 Printer
  - 1 Wireless Access Point
  - 1 Mobile or Wireless Device (laptop/smartphone/tablet)

The network is designed to ensure:

- Logical segmentation using VLANs
- Automatic IP address assignment using **DHCP**
- Wireless connectivity per department
- Inter-VLAN communication using **router-on-a-stick**

---

## Design Rationale

### Case Study Summary

- The company, based in Eastern Australia, operates globally and is expanding by opening a **branch office near Bon Alba village**.
- The branch network must **operate separately from HQ**, requiring a self-contained, independently routed system.
- Departments:
  - **Admin / IT** (VLAN 10)
  - **Finance / HR** (VLAN 20)
  - **Customer Service / Reception** (VLAN 30)

### Key Design Choices

- **VLAN Segmentation**: Each department was placed in a separate VLAN for logical isolation.
- **Router-on-a-Stick**: Implemented sub-interfaces on a single physical router interface to enable **inter-VLAN routing** without multiple physical routers.
- **DHCP Pools**: Configured DHCP services on the router to automatically assign IPs per subnet.
- **Wireless Access Points**: Each department includes a dedicated access point for mobile device connectivity.
- **Class C Subnetting**: Performed CIDR-based subnetting on the base network to create 3 usable /26 subnets, each accommodating up to 62 hosts.

---

## Subnetting Plan

**Base Network Provided:** `192.168.1.0/24`  
**Required Subnets:** 3

Using **/26** subnets to provide 64 addresses per department (62 usable):

| VLAN | Department               | Network            | Range                          | Broadcast        | Gateway         |
|------|--------------------------|--------------------|----------------------------------|------------------|-----------------|
| 10   | Admin / IT               | 192.168.1.0/26     | 192.168.1.1 – 192.168.1.62       | 192.168.1.63     | 192.168.1.1     |
| 20   | Finance / HR             | 192.168.1.64/26    | 192.168.1.65 – 192.168.1.126     | 192.168.1.127    | 192.168.1.65    |
| 30   | Customer Service / Recpt | 192.168.1.128/26   | 192.168.1.129 – 192.168.1.190    | 192.168.1.191    | 192.168.1.129   |

---

## Configuration Overview

### VLANs on Switch

```bash
enable
configure terminal

# Assign ports to VLANs
interface range fastEthernet 0/2 - 0/4
 switchport mode access
 switchport access vlan 10

interface range fastEthernet 0/5 - 0/7
 switchport mode access
 switchport access vlan 20

interface range fastEthernet 0/8 - 0/10
 switchport mode access
 switchport access vlan 30

# Configure trunk link to router
interface fastEthernet 0/1
 switchport mode trunk
````

---

### 🔧 Inter-VLAN Routing (Router-on-a-Stick)

```bash
enable
configure terminal

# Enable router interface
interface gig0/0
 no shutdown

# Sub-interface for VLAN 10
interface gig0/0.10
 encapsulation dot1Q 10
 ip address 192.168.1.1 255.255.255.192

# Sub-interface for VLAN 20
interface gig0/0.20
 encapsulation dot1Q 20
 ip address 192.168.1.65 255.255.255.192

# Sub-interface for VLAN 30
interface gig0/0.30
 encapsulation dot1Q 30
 ip address 192.168.1.129 255.255.255.192
```

---

### DHCP Configuration (on Router)

```bash
service dhcp

# VLAN 10 - Admin
ip dhcp pool ADMIN
 network 192.168.1.0 255.255.255.192
 default-router 192.168.1.1
 dns-server 192.168.1.1
 domain-name admin.com

# VLAN 20 - Finance
ip dhcp pool FINANCE
 network 192.168.1.64 255.255.255.192
 default-router 192.168.1.65
 dns-server 192.168.1.65
 domain-name finance.com

# VLAN 30 - Customer Service
ip dhcp pool CS
 network 192.168.1.128 255.255.255.192
 default-router 192.168.1.129
 dns-server 192.168.1.129
 domain-name cs.com
```

---

### Wireless Access Points

Each access point was configured with a unique **SSID and WPA2 password**:

| Department             | SSID           | Password       |
| ---------------------- | -------------- | -------------- |
| Admin / IT             | `admin-wifi`   | `admin@123`    |
| Finance / HR           | `finance-wifi` | `finance@1234` |
| Customer Service / Rec | `cs-wifi`      | `customer@123` |

Wireless devices (phones, tablets, laptops) were successfully connected using the correct credentials and received DHCP addresses from their respective pools.

---

## Connectivity Testing

**DHCP**: All end devices (wired and wireless) successfully received IP addresses via DHCP.

**Wireless**: Devices connected to APs with correct credentials and were dynamically assigned addresses.

**Inter-VLAN Communication**: Successful `ping` tests performed:

* From **Smartphone (VLAN 10)** → **PC (VLAN 20)**
* From **Tablet (VLAN 30)** → **Printer (VLAN 10)**
* From **Laptop (VLAN 20)** → **Smartphone (VLAN 30)**

---

## 📁 Files Included

* `branch-office-vlan-network.pkt` – Cisco Packet Tracer project file
* Screenshots of DHCP leases, routing table, and successful ping tests (optional)

---

## Challenges & Solutions

* **VLAN Misconfiguration**: Initially assigned wrong VLAN IDs to switch ports; corrected using `switchport access vlan XX`.
* **Trunk Port Issue**: Communication failed until I configured the router-facing switch port as `trunk`.
* **DHCP Scope Overlap**: Ensured that each VLAN DHCP pool used correct subnet ranges and default gateways.

---

## Skills Demonstrated

* Subnetting and CIDR design (/26)
* VLAN creation and switchport assignment
* Inter-VLAN routing (router-on-a-stick)
* DHCP configuration with multiple scopes
* Wireless setup and client authentication
* Real-world problem decomposition and network design

---

## Summary

This lab was a complete simulation of a **multi-department branch office network**, including VLAN segmentation, DHCP, wireless access, and inter-VLAN routing — all within a constrained hardware footprint. It demonstrates core Layer 2 and Layer 3 concepts and reflects a practical implementation that could be applied in real branch deployments.

---

## Next Steps

In upcoming labs, I plan to:

* Introduce **Layer 3 switching**
* Integrate **NAT and ISP simulation**
* Build **redundant topologies** with STP and EtherChannel
* Add **server roles** like DNS, FTP, and web hosting

---
