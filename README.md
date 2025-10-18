# Cisco Packet Tracer Topologies

A curated set of hands-on networking labs I designed and built in **Cisco Packet Tracer**.  
Each lab is a small, self-contained project that demonstrates practical network design: addressing plans, routing choices, device configuration, verification, and troubleshooting. I created these to deepen my understanding of real-world networking and to document my decisions the way I would on a professional project.

---

## Why I Built This

I learn best by building. Packet Tracer lets me move quickly from a design idea to a working network, then test it under different conditions. These labs reflect my interests in:
- Clear **network segmentation** and scalable addressing
- **Configuration discipline** and repeatable verification
- **Trade-offs** between simplicity, cost, and performance
- Writing **engineering-style documentation** that explains *why* a choice was made—not just *what* the commands were

My goal is to show deliberate design thinking, not just step-by-step imitation of tutorials.

---

## What’s Inside

Each folder is one lab with:
- The `.pkt` topology file
- A `README.md` describing the scenario, design rationale, configuration, verification, challenges, and takeaways
- Optional `screenshots/` (CLI outputs, topology images)
- Optional artifacts such as `ip-plan.xlsx`

Example structure:
```

cisco-topologies/
├── Hotel Management Network
│   ├── README.md
├── Branch Office VLAN Network with DHCP, Wireless, and Inter-VLAN Routing
│   ├── README.md
├── .DS_Store
├── Site-to-Site IPsec VPN on Cisco ASA
│   ├── README.md
├── Campus Network
│   ├── README.md
├── README.md <- This file!! >
├── SUBMODULE_GUIDE.md <- Guide for pinning projects to specific versions
├── Accounts and Delivery Department Network Design (DONE) [📌 v1.0]
│   ├── Screenshot_Topology.png
│   ├── written.jpg
│   ├── README.md
│   ├── Screenshot_Topology.pkz
│   ├── Ping C3.png

```

---

## How Each Lab Is Written

Every lab follows a consistent outline to make engineering review easy:

1. **Objective** – one-paragraph summary of the problem and constraints.  
2. **Topology Overview** – devices, links, and a diagram reference.  
3. **Design Rationale** – address plan, protocol choice, and the trade-offs considered.  
4. **Configuration Overview** – key configs (router/switch/hosts).  
5. **Verification** – deterministic tests (pings, routes, ARP, trunk state) and expected outputs.  
6. **Troubleshooting Notes** – what went wrong first and how I fixed it.  
7. **Takeaways** – what I learned and what I’d change in a v2.

---

## Skills Demonstrated

- Subnetting, CIDR planning, and IP allocation conventions
- Switch fundamentals (access/trunk ports, VLAN segmentation)
- Router configuration (SVIs, inter-VLAN routing, static routes; later: dynamic routing)
- DHCP, NAT, and basic ACLs (in later labs)
- Verification methodology (Real-Time vs Simulation mode, `show`/`ping`/`tracert`)
- Documentation quality: clear reasoning, reproducible steps, and testable outcomes

---

## How to Use

1. Open a lab’s folder and read its `README.md` first.  
2. Open the `.pkt` file in **Cisco Packet Tracer** (tested with version noted in each lab’s README).  
3. Follow the **Verification** section to reproduce the results.  
4. Compare outputs to the screenshots when provided.

> If you’re learning: try implementing the addressing plan from the README before looking at the final configuration.

---

### Pinning Projects to Specific Versions

Some completed projects are tagged to allow you to reference specific stable versions:
- **Accounts and Delivery Department Network Design**: `accounts-delivery-v1.0`

To checkout a specific pinned version:
```bash
git checkout accounts-delivery-v1.0
```

For more information on working with pinned versions and submodules, see [SUBMODULE_GUIDE.md](./SUBMODULE_GUIDE.md).


## Design Principles

- **Clarity over cleverness** – readable addressing, consistent names, deterministic tests.  
- **Single change at a time** – each lab focuses on one or two concepts so results are attributable.  
- **Explain the *why*** – every config block exists for a reason; I document that reason.  
- **Reproducibility** – anyone should be able to open the file, run the tests, and get the same results.

---

## Roadmap

- Inter-VLAN routing (router-on-a-stick vs multilayer switch)
- DHCP scopes per VLAN and relay
- Basic ACLs for inter-department policy
- NAT and simple WAN edge scenarios
- Intro to dynamic routing (OSPF single area, then multi-area)
- Redundancy and first-hop resiliency (conceptual introduction)

---

## Notes on Sources

I occasionally reference standard designs and vendor documentation for correctness, but every topology and write-up here is **built and reasoned by me**. Where I adapt a common pattern, I document my deviations and justify the trade-offs.

