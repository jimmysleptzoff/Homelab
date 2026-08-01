Author: James Sleptzoff

File: 2_implementing_pfsense.md

Created: July 15, 2026

Last Modified: July 27, 2026

---

# Goal

PfSense is a free and open-source firewall and router distribution tool. By the end of this task I want to have pfSense installed, configured, and working on proxmox. After I have it installed, my goal will be to implement basic security.

## Installing PfSense

In order to get pfSense set up without any managed switches, I will have to use two Proxmox bridges. `vmbr0` is the default bridge tied to the physical NIC on the server, and will be used as the WAN for pfSense to connect to the internet through my router. `vmbr1` is not connected directly to a physical NIC, and will just be used to connect to other VMs on Proxmox using a subnet isolated from my home network. I also made sure that `vmbr1` is VLAN-aware in case I want to further segment it from my network in the future.

Here is how I set up the bridges:

1. Navigate to Datacenter > your node (pve by default) > System > Network > Create > Linux Bridge
2. Name it 'vmbr1' and check "VLAN aware"
3. Apply configuration

The download for pfSense Community Edition can be found [here](https://atxfiles.netgate.com/mirror/downloads/). After downloading, make sure to check the hash with the corresponding hash file. Once verified, you can install and configure it.

Here's how I configured mine:

* 2 cores
* 4096Mb RAM
* 20Gib Disk space VirtIO Block
* vmbr0 iface w/ Intel E1000 virtual NIC

## Setting up pfSense

After booting and going through the initial setup, you'll be prompted to assign the WAN and LAN interfaces. Proxmox only allows you to add one network device to a VM when you create it so you can only add the WAN (vtnet0). After this, you'll be taken to the main pfSense console menu.

In order to add the LAN interface you will need to navigate to the hardware tab and add vmbr1 manually. After it's added, reboot pfSense and go through option 1 (Assign Interfaces). Assign WAN to vtnet0 again and assign LAN to vtnet1. In my case, an IP wasn't assigned to LAN, so I have to assign one manually and went with `192.168.10.1` with a DHCP range between 100-199 (/24).

In order to access the web GUI, I'll have to spin up another VM and connect it to the `vmbr1`.

## Testing the web GUI

Before I install another VM with a desktop interface to test access to the GUI, I used the Ubuntu Server VM that I set up when I first installed Proxmox. To do this, I changed the interface to `vmbr1` and used the ping and curl commands.

After getting the VM booted up, I checked to make sure that it was using the correct interface, with an IP address and route assigned. After verifying that it was all correct, I was successfully able to ping and curl the web GUI. [[See Troubleshooting]](#troubleshooting)

## Accessing the pfSense web GUI

As I mentioned in the section above, in order to access the web GUI, I of course need to use an OS with a desktop interface. I've been wanting to try out different Linux distributions, so to do this, I installed Xubuntu Desktop, as it's lightweight, but still has a similar feel to Ubuntu.

Once logged in, I navigated to the web GUI using my browser and verified that it was accessible:

![pfSense GUI dashboard accessible from Xubuntu VM](/assets/images/pfsense-on-xubuntu.png)

## Configuring & Hardening pfSense

As with almost any other software, pfSense needs to be properly configured to meet a minimum security baseline. In the following sections I will go over the steps I took to do this. Keep in mind that what others to do may be different.

### Configure the admin account

PfSense comes with a default admin account using a default password. Changing this is the immediate first step. Optionally, you can disable this account entirely and create a new admin with a different username and/or turn on MFA.

### Turn on HTTPS

Despite pfSense initially claiming to use HTTPS, it was instead using HTTP. Thankfully, switching to HTTPS is as simple as navigating to System > Advanced > Admin Access, and selecting it.

**Optional:**

To get rid of the "self-signed certificate" warning from your browser, you can either add the certificate to the browser itself, or create a new Certificate Authority (CA) under System > Certificate > Authorities > Add, and then add yourself as a trusted CA on your browser.

This will vary browser-to-browser, but here are the steps that I took with Firefox:

1. Create the CA in pfSense by going to System>Certificate>Authorities>Add and create a new certificate, name it whatever you want (Descriptive Name and Common Name.) Then, once it's created, click the "Export" button (spiky circle.)
2. Go to the certificates tab and click "Add/Sign" and create a new internal certificate signed by the CA you just created. Make the descriptive name is whatever you want but make sure the common name is the domain name of pfSense, in this case, and likely in yours, it will just be the LAN IP. Also, make sure that the certificate type is set to server, and add the IP as a SAN.
3. In Firefox, go to settings > privacy & security > manage certificates > authorities > import, and select the CA you downloaded earlier. Then select the use for website option.
4. On pfSense, navigate to System > Advanced > Admin Access, and next to SSL/TLS certificate, select the certificate that you created to bind it to the GUI.

To test this, you can just reload the page manually or wait for the page to reload automatically which it will do after ~20 seconds. [[See Troubleshooting]](#troubleshooting)

### SSH

SSH allows for remote console access and can be turned on/off from System > Advanced > Admin Access > SSH. In my case, I am leaving it disabled. 

### Login Protection

Login protection protects against brute-force login attacks and can be configured from System > Advanced > Admin Access > Login Protection. From here you can set a limit for failed attempts, blocktime, etc. as well as add IPs that can bypass the restriction.

### Other

* You can also set "Max Processes" in the admin access section to 1 or 2 which will limit the number of users able to use the GUI concurrently. 
* Check for and install updates

---

# Troubleshooting

[~/Section/Testing pfSense](#testing-the-web-gui)

* **Issue:** Initially, I was not getting a response when I pinged the web GUI
* **Solution:** Switch virtual NIC from VirtIO (paravirtualized) to Intel E1000 and reconfigure WAN/LAN

* **Issue:** After switching to the Intel E1000 and verifying that I could ping the web GUI, the curl command would still hang.
* **Solution:** Use HTTP instead of HTTPS
* **Deduction:** While letting the curl command hang, I inspected pftop via the pfSense console and noticed that the request *was* being received, but with no response sent back. From here, I decided to try HTTP instead of HTTPS which worked despite pfSense claiming it was using HTTPS.

[~/Section/Turn on HTTPS](#turn-on-https)
* **Issue:** After binding a certificate to the GUI, Firefox threw a `SEC_ERROR_INADEQUATE_KEY_USAGE` error and refused to load the pfSense dashboard entirely
* **Solution:** Used Chrome instead, which didn't enforce the same certificate restrictions and allowed access to the GUI. Fixed the certificate configuration properly while in Chrome, then switched back to using Firefox