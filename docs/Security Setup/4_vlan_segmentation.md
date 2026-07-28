Author: James Sleptzoff

File: 4_vlan_segmentation.md

Created: July 27, 2026

Last Modified: July 27, 2026

# Goal

As of right now, the homelab is segmented on its own screened-subnet on my home network. However, within the lab itself, there is no segmentation. In order to properly segment devices within the lab (e.g. isolating management interfaces, servers, offensive/defensive targets, etc.), I will need to implement a VLAN with IEEE 802.1Q tagging.

To do this, each device's traffic will need to be tagged and routed through `vmbr1` which is the vlan-aware bridge I configured earlier, acting as the trunk port to carry traffic between pfSense and the device(s). After this, I can configure pfSense firewall rules with VLAN interface to route traffic and filter traffic between segments.

I will also need to reconfigure the VPN to ensure that I can reach the entire homelab environment when I'm away from home.

## Plan

### VLAN segments:

| VLAN ID | Subnet | Purpose |
|---|---|---|
| 10 | 192.168.10.0/24 | Management (Xubuntu Management Workstation) |
| 20 | 192.168.20.0/24 | Servers/Infra (AD/Splunk/etc) |
| 30 | 192.168.30.0/24 | Reserved (Offensive/Defensive practice, etc.) |

### Firewall rules

#### VLAN10 (Management)

* **Allow:** Management --> any (e.g. pfSense GUI, etc)
* **Deny:** None (Denies everything by default, no rule needed)

#### VLAN20 (Server/Infra)

* **Deny:** VLAN20 --> VLAN10
* **Deny:** VLAN20 --> VLAN30
* **Allow:** VLAN20 --> any (for internet access)

#### VLAN30 (Reserved)

* **Allow/Deny:** Will be heavily dependent on specific device on this segment and its intended purpose.

## Configuring pfSense

`Interfaces > Assignments > VLANs > add`

Add three interfaces, all on the em1 interface with their designated tags (10, 20, 30) and description.

`Interfaces > Assignments > Interface Assignments > LAN`

Change the LAN interface assignment to VLAN10. When this is applied, the connection to pfSense will be lost because Xubuntu (what i'm accessing the GUI on) is not being tagged. To fix this, in Proxmox, go to `Xubuntu > Hardware > Network Device`, and set the VLAN tag to 10. Once this has been applied, pfSense access is restored.

On the same page on pfSense, add VLAN20 and VLAN30. They will automatically be assigned to OPT1 and OPT2 respectively. Then enable and assign each of them a respective static IPv4 (192.168.20.1/24 and 192.168.30.1/24). I've opted not to use DHCP as of now because there isn't much of a need for it.

## Configure Firewall

#### VLAN10

Since VLAN10 was the LAN, it already has default allow to any to any, and everything inbound is blocked. No new rules needed.

#### VLAN20

These rules must be in order because they will be followed sequentially (top to bottom).

| State | Protocol | Source | Port | Destination | Gateway | 
|---|---|---|---|---|---|
| Deny | any | VLAN20 subnets | any | VLAN10 subnets | any |
| Deny | any | VLAN20 subnets | any | VLAN30 subnets | any |
| Allow | any | VLAN20 subnets | any | any | any |

#### VLAN30

Nothing is on this VLAN segment yet, everything outbound will be denied by default. No new rules needed.

## Testing the VLAN(s)

Now that all three VLANs are set up and (hopefully) configured correctly, it's time to test. To do this, just use `ping` from one device on each segment. Since Xubuntu (where I'm accessing the pfSense GUI) is already on segment 10, I can just open a terminal quickly and test. To test the other two segments, I can spin up a temporary VM 

### VLAN10

![vlan10-test](/assets/images/vlan10-test.png)

### VLAN20

![vlan20-test](/assets/images/vlan20-test.png)

### VLAN30

![vlan30-test](/assets/images/vlan30-test.png)

As shown above, all three VLAN segments are working properly. All that's left to do is reconfigure the VPN.

## Fix VPN

Fixing the VPN is straightforward. Since it was originally setup with access to the LAN, it only needs one new rule on the wireguard interface firewall that allows access to VLAN20 (Since LAN is now VLAN10). After this, 192.168.20.0/24 needs to be appended to the AllowedIPs field on the client-side.

# Consideration Process

I would like to note the reasons for some of my decisions made above.

* **Q:** Why did I not move the pi-hole onto a segment?
* **A:**  I wanted to set up pi hole to be accessible to my whole network, not something lab-specific. I don't want to move it behind the firewall and run into potential issues down the road.

---

* **Q:** Why set up VLANs after setting up the VPN?
* **A:** I knew I would be going out of town soon and wanted to make sure I could access the lab remotely if I wanted to. I also wanted to be able to test the VPN from a further distance to ensure there weren't any issues (there weren't).

---

* **Q:** Why didn't you allow VPN access to VLAN30?
* **A:** Currently there's nothing *on* VLAN30 to access anyways, and in the future, it will be a very controlled environment specifically for testing. I may add access in the future, but it will depend on what's on it.

---

* **Q:** Why is there no troubleshooting section?
* **A:** It was not needed. Any troubleshooting was very minor and took < 30 seconds to fix (e.g. forgetting to add a gateway address to xubuntu when testing vlan20).