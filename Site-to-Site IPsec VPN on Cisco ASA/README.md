---
# Site-to-Site IPsec VPN on Cisco ASA — Cisco Packet Tracer

## Objective
Design and implement a site-to-site IPsec VPN between an HQ network and a Branch network using Cisco ASA firewalls in Cisco Packet Tracer. The tunnel must encrypt inter-site traffic, preserve local internet access through NAT/PAT at each site, and use a deterministic verification workflow (pings, SA status, packet captures).  
Inspiration: a walkthrough demonstrating ASA site-to-site IPsec VPN. This write-up is my own end-to-end design and configuration. :contentReference[oaicite:0]{index=0}

---

## Topology Overview
- **HQ site**
  - Inside LAN: 10.10.10.0/24
  - ASA outside toward ISP cloud
  - Default route to ISP; NAT/PAT for local internet egress
- **Branch site**
  - Inside LAN: 10.20.20.0/24
  - ASA outside toward ISP cloud
  - Default route to ISP; NAT/PAT for local internet egress
- **Transport**
  - Public “ISP” segment per site (simulated with static public IPs)

Optional diagram reference:
```

![Topology Diagram](./screenshots/asa-ipsec-s2s.png)

````

---

## Addressing Plan

| Site   | Inside LAN      | ASA Inside (nameif inside) | ASA Outside (nameif outside) | Public IP (outside) |
|--------|------------------|----------------------------|-------------------------------|---------------------|
| HQ     | 10.10.10.0/24    | 10.10.10.1/24             | 198.51.100.2/30               | 198.51.100.2        |
| Branch | 10.20.20.0/24    | 10.20.20.1/24             | 203.0.113.2/30                | 203.0.113.2         |

Default routes:
- HQ ASA: `route outside 0.0.0.0 0.0.0.0 198.51.100.1`
- Branch ASA: `route outside 0.0.0.0 0.0.0.0 203.0.113.1`

---

## Crypto Policy

**IKE Phase 1 (ISAKMP)**
- Encryption: AES-256
- Integrity: SHA-256
- DH Group: 14
- Lifetime: 86400

**IPsec Phase 2 (Transform-set / proposal)**
- ESP-AES-256
- ESP-SHA-256
- PFS: group 14 (optional)

Protected networks (interesting traffic):
- HQ LAN ⇆ Branch LAN (10.10.10.0/24 ⇆ 10.20.20.0/24)

NAT policy:
- **NAT exemption** for interesting traffic on both ASAs so it is encrypted, not translated.  
- Regular PAT for all other internet-bound traffic remains in effect. :contentReference[oaicite:1]{index=1}

---

## Configuration — HQ ASA (key snippets)

```text
enable
configure terminal

! Interfaces
interface gi0/0
 nameif outside
 security-level 0
 ip address 198.51.100.2 255.255.255.252
 no shut

interface gi0/1
 nameif inside
 security-level 100
 ip address 10.10.10.1 255.255.255.0
 no shut

! Default route
route outside 0.0.0.0 0.0.0.0 198.51.100.1

! Object networks for NAT/PAT
object network HQ_INSIDE
 subnet 10.10.10.0 255.255.255.0
object network HQ_OUTSIDE_IF
 host 198.51.100.2

! PAT for internet egress (non-VPN traffic)
nat (inside,outside) after-auto source dynamic HQ_INSIDE interface

! NAT exemption for VPN traffic (HQ->Branch)
object network BRANCH_LAN
 subnet 10.20.20.0 255.255.255.0
nat (inside,outside) source static HQ_INSIDE HQ_INSIDE destination static BRANCH_LAN BRANCH_LAN no-proxy-arp route-lookup

! IKEv1 policy (Packet Tracer ASA typically defaults to IKEv1)
crypto ikev1 policy 10
 authentication pre-share
 encryption aes-256
 hash sha
 group 14
 lifetime 86400

! Enable IKEv1 on outside
crypto ikev1 enable outside

! IPsec transform set
crypto ipsec ikev1 transform-set TS-AES256 esp-aes-256 esp-sha-hmac

! Access-list for interesting traffic
access-list ACL-VPN-HQ extended permit ip 10.10.10.0 255.255.255.0 10.20.20.0 255.255.255.0

! Crypto map binding
crypto map CMAP 10 match address ACL-VPN-HQ
crypto map CMAP 10 set ikev1 transform-set TS-AES256
crypto map CMAP 10 set peer 203.0.113.2
crypto map CMAP 10 set security-association lifetime seconds 3600
crypto map CMAP interface outside

! Pre-shared key for the peer
tunnel-group 203.0.113.2 type ipsec-l2l
tunnel-group 203.0.113.2 ipsec-attributes
 ikev1 pre-shared-key cisco123
exit
write memory
````

---

## Configuration — Branch ASA (key snippets)

```text
enable
configure terminal

! Interfaces
interface gi0/0
 nameif outside
 security-level 0
 ip address 203.0.113.2 255.255.255.252
 no shut

interface gi0/1
 nameif inside
 security-level 100
 ip address 10.20.20.1 255.255.255.0
 no shut

! Default route
route outside 0.0.0.0 0.0.0.0 203.0.113.1

! Object networks
object network BR_INSIDE
 subnet 10.20.20.0 255.255.255.0
object network BR_OUTSIDE_IF
 host 203.0.113.2

! PAT for internet egress (non-VPN traffic)
nat (inside,outside) after-auto source dynamic BR_INSIDE interface

! NAT exemption for VPN traffic (Branch->HQ)
object network HQ_LAN
 subnet 10.10.10.0 255.255.255.0
nat (inside,outside) source static BR_INSIDE BR_INSIDE destination static HQ_LAN HQ_LAN no-proxy-arp route-lookup

! IKEv1 policy (mirrors HQ)
crypto ikev1 policy 10
 authentication pre-share
 encryption aes-256
 hash sha
 group 14
 lifetime 86400

crypto ikev1 enable outside

! IPsec transform set
crypto ipsec ikev1 transform-set TS-AES256 esp-aes-256 esp-sha-hmac

! Interesting traffic ACL
access-list ACL-VPN-BR extended permit ip 10.20.20.0 255.255.255.0 10.10.10.0 255.255.255.0

! Crypto map
crypto map CMAP 10 match address ACL-VPN-BR
crypto map CMAP 10 set ikev1 transform-set TS-AES256
crypto map CMAP 10 set peer 198.51.100.2
crypto map CMAP 10 set security-association lifetime seconds 3600
crypto map CMAP interface outside

! Pre-shared key
tunnel-group 198.51.100.2 type ipsec-l2l
tunnel-group 198.51.100.2 ipsec-attributes
 ikev1 pre-shared-key cisco123
exit
write memory
```

---

## Host IP Configuration (Packet Tracer)

Configure PCs via **Desktop → IP Configuration**; use static addressing for quick tests.

**HQ**

* PC-HQ: IP `10.10.10.10`, Mask `255.255.255.0`, Gateway `10.10.10.1`

**Branch**

* PC-BR: IP `10.20.20.10`, Mask `255.255.255.0`, Gateway `10.20.20.1`

(Optional) DNS can be omitted for pure ICMP tests.

---

## Verification

On both ASAs:

```text
show crypto ikev1 sa
show crypto ipsec sa
show access-list ACL-VPN-HQ
show access-list ACL-VPN-BR
show nat detail
```

From **PC-HQ**:

```text
ping 10.20.20.10
```

From **PC-BR**:

```text
ping 10.10.10.10
```

Expected:

* First ping may trigger negotiation; subsequent pings succeed.
* `show crypto ipsec sa` should show incrementing packet counters for both inbound and outbound SAs.

Simulation mode (optional):

* Filter to ESP/ISAKMP equivalents in Packet Tracer and observe encapsulated traffic.

---

## Troubleshooting Notes

* **No SA formed**: Check pre-shared key, peer IPs, and Phase 1 policy parity (encryption, hash, DH group).
* **Pings time out**: Confirm NAT exemption rules; ensure interesting-traffic ACL matches both directions.
* **Only one direction works**: Verify crypto map ACLs are mirrored correctly and applied to the correct interface.
* **NAT interfering**: Ensure the non-translated (identity) NAT rule for LAN-to-LAN is evaluated before PAT.
* **Routing**: Confirm default routes to ISP on both ASAs; inside hosts must have correct gateways. ([gurutechnetworks.otombenard.com][1])

---

## Files Included

* `asa-ipsec-s2s.pkt` — Packet Tracer topology
* `screenshots/` — SA status, ping success, interface summaries
* `ip-plan.xlsx` — Addressing and policy matrix

---

## Skills Demonstrated

* ASA interface, nameif, security levels, and default routing
* IKEv1 Phase 1/Phase 2 policy design
* Crypto map and tunnel-group configuration
* NAT exemption vs PAT coexistence
* Deterministic verification of IPsec SAs
* Structured documentation of decisions and trade-offs

---

## Summary

This lab delivers a working ASA-to-ASA site-to-site IPsec VPN that encrypts inter-site traffic while preserving internet access through NAT at each location. The policy, NAT, and ACL structure is minimal but production-adjacent and ready to extend with IKEv2, PFS, stricter ACLs, and high-availability in future iterations.

---