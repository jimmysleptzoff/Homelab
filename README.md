# ZoffLab: Cybersecurity & IT Infrastructure Lab

This is a self-directed homelab built on Proxmox that's being used both as a personal project, and as a way to become more proficient with common cybersecurity tools. Each phase will be documented here and will include a goal or goals, methodology, and troubleshooting section when needed.

## Goals

* Build practical, hands-on experience with tools used in IT/security environments (pfSense, Splunk/Elastic, Active Directory, Azure AD, vulnerability scanners, etc.)
* Document the process both for myself and for others such as hiring managers, recruiters, and friends
* Practice the full lifecycle of a home network build, i.e. design, implement, troubleshooting, and harden
* Use for personal items such as Pi-hole, a private cloud, and/or a web server

## Environment

| Component | Tool |
|---|---|
| Hypervisor | Proxmox VE |
| Firewall / Router | pfSense |
| DNS Filtering | Pi-hole |
| VPN | WireGuard (via pfSense) |
| Hardware | Repurposed laptop |

## Roadmap

- [x] **Phase 1 — Network Foundation**
  - [x] Proxmox installed and configured
  - [x] pfSense deployed (WAN/LAN setup, admin hardening)
  - [x] Pi-hole deployed for DNS filtering
  - [ ] WireGuard VPN (in progress) — client subnet isolation, scoped firewall rules
  - [ ] VLAN segmentation
  - [ ] DNS hardening with Unbound
- [ ] **Phase 2 — Identity & Directory Services**
  - [ ] Windows Server Active Directory
- [ ] **Phase 3 — Monitoring & SIEM**
  - [ ] Splunk or Elastic Stack, with log forwarding from pfSense/Pi-hole/AD
- [ ] **Phase 4 — Vulnerability Management**
  - [ ] OpenVAS / Nessus Essentials
  - [ ] Scan → triage → remediate → rescan cycle
- [ ] **Phase 5 — Cloud Integration**
  - [ ] Azure AD (Entra ID)
  - [ ] Hybrid identity via Azure AD Connect
  - [ ] Optional: Microsoft Sentinel
- [ ] **Phase 6 — Offensive/Defensive Practice**
  - [ ] Kali Linux + intentionally vulnerable targets in an isolated VLAN
- [ ] **Phase 7 (Continuous) — Documentation**
  - [ ] Architecture diagrams
  - [ ] Per-phase write-ups (this repo)

## Documentation

Each completed phase has (or will have) its own write-up under [`/docs`](./docs), covering:
* What was built and why
* Configuration steps and key decisions
* Problems hit and how they were resolved
* Architecture diagrams where relevant

## Status

Currently working through **Phase 1**: finishing WireGuard VPN configuration before moving into VLAN segmentation and DNS hardening.