# ZoffLab: Cybersecurity & IT Infrastructure Lab

This is a self-directed homelab built on Proxmox that's being used both as a personal project, and as a way to become more proficient with common cybersecurity tools. Each phase will be documented here and will include a goal or goals, methodology, and troubleshooting section when needed.

## Goals

* Build practical, hands-on experience with tools used in IT/security environments (pfSense, Splunk/Elastic, Active Directory, Azure AD, vulnerability scanners, etc.)
* Document the process both for myself and for others such as hiring managers, recruiters, and friends
* Practice the full lifecycle of a home network build, i.e. design, implement, troubleshooting, and harden
* Use for personal items such as Pi-hole, a private cloud, and/or a web server

## Environment

| Component | Tool |
| --- | --- |
| Hypervisor | Proxmox VE |
| Firewall / Router | pfSense |
| DNS Filtering | Pi-hole |
| VPN | WireGuard (via pfSense) |
| Hardware | Repurposed laptop |

## Roadmap

* [x] **Phase 1 — Network Foundation**
  * [x] Proxmox installed and configured
  * [x] pfSense deployed (WAN/LAN setup, admin hardening)
  * [x] Pi-hole deployed for DNS filtering
  * [x] WireGuard VPN — client subnet isolation, scoped firewall rules
  * [x] VLAN segmentation
  * [x] ~~DNS hardening with Unbound~~ (deprioritized, not currently relevant)
* [x] **Phase 2 — Identity & Directory Services**
  * [x] Implement Windows Server Active Directory
  * [x] Harden Windows Server Active Directory
  * [x] Implement Organizational Units
  * [x] Configure OU Objects
  * [x] Set up a Fileserver & Shared Folders
  * [x] Implement a Windows Workstation
* [ ] **Phase 3 — Monitoring & SIEM**
  * [ ] Splunk or Elastic Stack, with log forwarding from pfSense/Pi-hole/AD (In progress)
* [ ] **Phase 4 — Vulnerability Management**
  * [ ] OpenVAS / Nessus Essentials
  * [ ] Scan → triage → remediate → rescan cycle
* [ ] **Phase 5 — Cloud Integration**
  * [ ] Azure AD (Entra ID)
  * [ ] Hybrid identity via Azure AD Connect
  * [ ] Optional: Microsoft Sentinel
* [ ] **Phase 6 — Offensive/Defensive Practice**
  * [ ] Kali Linux + intentionally vulnerable targets in an isolated VLAN
* [ ] **Phase 7 (Continuous) — Documentation**
  * [ ] Architecture diagrams
  * [ ] Per-phase write-ups (this repo)

## Documentation

Each completed phase has (or will have) its own write-up under [`/docs`](./docs), covering:

* What was built and why
* Configuration steps and key decisions
* Problems encountered and how they were resolved

## Status

Currently working through **Phase 3**: I've successfully set up Windows AD with two scoped Organizational Units, as well as a file server with shared folders. I also implemented NTFS and Share permissions on each of the shared folders to allow for least-privilege. Next, I will be implementing Splunk for log aggregation and network monitoring.
