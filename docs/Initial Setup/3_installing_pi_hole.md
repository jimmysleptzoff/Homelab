Author: James Sleptzoff

File: 3_installing_pi_hole.md

Date: July 15, 2026

Last Modified: July 27, 2026

---

# Goal

Pi-hole is an ad blocker that attempts to block ads by not returning DNS resolutions using a domain blacklist containing common ad-serving domain names. My goal is to set up and configure Pi hole using Proxmox, and then verify that it works using the web GUI.

## Installation

All of the documentation for pi-hole can be found on their [official website](https://pi-hole.net/).

After looking around, I discovered that there is a community script for installing pi-hole directly through the pve shell which can be accessed [here](https://community-scripts.org/scripts?q=pi-hole).

Once Pi-hole is installed, I gave it a static IP and reserved it on my router. In order to use it, I have to reroute DNS traffic through it, instead of through my default DNS on my router. Unfortunately, my router does not allow network-wide DNS rerouting by default, so I instead have to manually configure which DNS server to use on each of my individual devices.

This may be fixable with port forwarding, but for now, I'll just keep it on a device-by-device basis until I can test and verify Pi-holes reliability. 

Now that everything is properly configured, I can access the web GUI and verify that my traffic is being rerouted to Pi-hole:

![Pi-hole dashboard up and working](/assets/images/pi-hole-dashboard.png)
