Author: James Sleptzoff

File: 1_planning_and_deployment_decisions.md

Created: August 9, 2026

Last Modified: August 18, 2026

# Goal

The goal of this task will be to determine the general steps towards successfully implementing Splunk and forwarding all logs to it. To do this I will list out what I will need to take into consideration:

* Splunk type (Free or Enterprise Trial)
* OS that Splunk will sit on
* VLAN segment placement
* Index strategy

## Plan

### Splunk Version

The first thing that I will need to consider will be the type of Splunk that I want to use. Fortunately, Splunk allows users to use the Enterprise trial and will automatically convert to the free version after the 60-day trial period has ended.

There are however some drawbacks that come from using the free version that I would like to quickly mention:

* No alerting (which really stinks)
* No scheduled searches
* No RBAC (only a single admin user, so really any AC wouldn't make a lot of sense)
* No forwarding management console

These constraints for the most part are fine for my applications in the homelab, even if no alerting or RBAC is pretty hard-hitting (and I will still get to gain experience with them with the free enterprise trial.)

### VM OS

I mentioned in file 2 of phase 1 that throughout this project, I would like to try different flavors of linux. While this is still true, it's mainly for workstation or non-critical VMs, like for the one I'm using to access the pfSense web GUI on. Even if that VM goes down, pfSense wouldn't be affected.

However, in this case, Splunk will be running directly in the VM instead of on its own virtual device like pfSense is. Because of this, I will use Debian, since it's well-known for its stability and is a standard deployment option for Splunk (also it's not Windows which is a plus.)

### VLAN Placement

Splunk will be considered an infrastructure server as it will act as the primary log aggregation endpoint. Because of this, it will sit on VLAN20.

VLAN20 already has outbound access to the internet via the `Allow VLAN20 -> any` rule, so there doesn't need to be a new outbound rule for splunk-specifically. Also, VLAN10 is already permitted to send traffic to VLAN20, and VLAN30 will be used as a controlled testing environment. Because of this, no new firewall rules will need to be made unless I decide to add logs from the Pi-hole as well.

### Index Strategy

Indexing allows you to separate different log sources which can make them a lot easier to differentiate and search through. Because I do not currently have a large list of sources, I will create an index for each source.
