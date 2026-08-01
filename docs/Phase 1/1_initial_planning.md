Author: James Sleptzoff

File: 1_initial_planning.md

Created: July 15, 2026

Last Modified: July 27, 2026

---

# Goal

In this document I will lay out future considerations for tools, platforms, and features that I want to use on my homelab.

## Phase 1

There are a few specific tools/platforms/etc that I want to use and get more experience with in a controlled environment such as:

* pfSense
* AD (Entra ID or Samba AD)
* Azure
* Splunk
* Wazuh
* Ansible
* Terraform
* VPN
* NAS

The above list is in no specific order and is non-exhaustive. It is simply a list of things off the top of my head that I would like to use and/or gain more experience with.

## Phase 2

Now that I have listed out some broad guidelines for what I want, I can start to plan how I want to move forward. For right now, here is my current plan of action:

1. Set up pfSense
2. Set up a VPN
3. Set up a Windows server with Active Directory. I want to look into syncing it with Entra ID via Entra Connect to sync it with Azure.
4. Implement monitoring via Splunk to aggregate logs from pfSense and the AD. Optionally I might look into Wazuh as well.
5. Implement automation/IaC using Ansible and Terraform. This will allow me to create and configure VMs using a golden image instead of provisioning them individually in the future.

## Phase 3

**Optional**: The above are the core deliverables that I want to implement. When all of these are set up, I can make the decision to explore more options such as extending monitoring network-wide, implementing and hosting a website (which I want to do anyways), and/or buying a NAS (possibly to set up a private cloud).

Though I have marked this as optional, homelabbing is something I am genuinely interested in so by "optional," I more mean plans for after I have the core components implemented.
