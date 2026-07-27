Author: James Sleptzoff

File: 2_setup_and_configuration.md

Created: July 14 2026

Last Modified: July 27, 2026

---

# Goal

With the hardware set up, the next step is to install Proxmox on the laptop, which will act as the hypervisor. I will also need to ensure that the laptop is properly configured with a static IP on my home network, and that it can reach DNS and the internet. Last, I want to explore VM setup by installing ubuntu-server.

# Proxmox Install and Configuration

### 1. Installing Proxmox

From here on out I will be referring to my old laptop as a server or lab server. I decided to go with Proxmox VirtualEnvironment (VE) because it's free, open-source, and recommended. After backing up the server, getting Proxmox is straight forward as it's no different than any other OS install.

### 2. Connecting to network

Once Proxmox has been installed, it would not connect to the internet, or resolve hostnames. By default, Proxmox uses a `192.x.x.x` private IPv4 whereas my router assigns `10.x.x.x` addresses. The fix to this is to simply edit `/etc/network/interfaces` and assign it a new static IP on the same subnet as my home router. I then needed to add my default DNS server IP, and then configure my router to reserve the static IP I set to avoid collisions caused by DHCP.

After restarting the networking daemon on the server using `systemctl restart networking`, I verified that both DNS and the internet were working properly by pinging google.com and accessing the web GUI.

### 3. Running scripts

A common first step after getting Proxmox set up is to run scripts such as the [Proxmox Post Install](https://community-scripts.org/scripts/post-pve-install?from=scripts&fromQ=proxmon+VE) script which enables the no-subscription repo, disables the subscription nag, adds a test repo, updates Proxmox, and reboots.

There are countless other scripts that you can run, a lot of which are available on the site linked above. For now, I will stick with just the post-install script.


### 4. Creating the first VM

Now that Proxmox is properly configured and updated, I want to test setting up a VM. To do this, I will install ubuntu-server, a lightweight and very popular operating system.

The general steps for installing a VM in Proxmox are as follows:

1. Download the VM ISO from the internet
2. On the Proxmox web GUI, click on "Create VM" in the top right corner
3. Choose your ISO
4. Follow the setup steps (Name, ID, resource allocation, etc.)

Once the VM finished installing, I logged in and tested internet access and DNS using `curl parrot.live`, which displays a fun ASCII animation of a dancing parrot:

![A screenshot of ubuntu server running on my proxmox server with curl live.parrot going in the terminal](/assets/images/ubuntu-server-setup.png)

---

From hereon out, I will be including troubleshooting logs at the bottom of each file if warranted.

---

# Troubleshooting:

[<b>~/Section/Running Scripts/:</b>](#3-running-scripts)

* **Issue:** Had internet, couldn't resolve hostnames
* **Solution:** Had to reassign correct DNS address

---

* **Issue:** curl was getting a bad certificate
* **Solution:** Restarted chrony `systemctl restart chrony` to allow time sync now that DNS is set up