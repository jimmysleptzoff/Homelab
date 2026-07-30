Author: James Sleptzoff

File: 1_installing_active_directory.md

Created: July 28, 2026

Last Modified: July 30, 2026

# Goal

Now that the lab has a segmented network and a Servers/Infra VLAN to house it, I want to stand up a Windows Server domain controller running Active Directory Domain Services (AD DS) to gain hands-on experience with the identity/directory services that underpin most enterprise IT environments. This is the foundation for several future phases: centralized authentication and user/group management, Group Policy for baseline configuration, and eventually hybrid identity via Azure AD Connect syncing to Entra ID. AD will also become a log source once SIEM (Phase 3) is implemented, and a resolution point for future domain-integrated DNS.

## Getting Windows Server Set up

The process for getting Windows Server setup is the same for any other VM; download the iso and go through the install process.

Once the VM is installed, I set VLAN tagging up, configured a static IPv4 (192.168.20.100/24), with the 192.168.20.1 gateway in order to connect to the internet. After that, I installed updates and replaced MS Edge with Firefox.

# Desc

The main functional components of AD DS are as follows:

* **Forest:** Outermost security boundary containing everything within scope
* **Domain:** A logical partition that sits inside of the forest. Shares a common directory database, security polices, and namespace.
* **OU (Organizational Unit):** Sits inside of the domain and is used to organize objects such as computers, users, and groups.
* **Objects:** Sit inside OUs, the actual user, computer, or group that is being managed

The physical (or virtual) machine that hosts a copy of this directory is the DC (Domain Controller). A DC runs AD DS (Active Directory Domain Services), which is the role that lets a Windows Server function as a directory service. Promoting a server either creates a brand new forest (as in this build) or joins it to an existing one as an additional DC since a single forest can have multiple DCs replicating the same data.

## Plan

Getting AD DS setup requires a little bit of planning before I start making changes. The goal in this section will be to lay out specific steps, names/identifiers, groups, users, etc.

The first step is to choose a domain name, I'll go with zofflab.local.

## Installing the AD DS role

In order for this server to do anything, it must be set up as an actual DC which requires installing and promoting an AD DS role. For the first time setup, I need to go through the setup wizard.

`Manage > Add Roles and Features`

Once in the wizard, select `Active Directory Domain Services` on the Server Roles page. Keep everything else selected as-is and select install.

In order for this server to actually become a DC, I have to promote the AD DS role. In the top right, there's a flag with a caution symbol next to it. When clicked, it prompts me to promote the server to a domain controller. Clicking this brings up another wizard.

Since there is no existing forest, I have to create a new one by selecting the "Add a new forest" option and naming the root domain. After that, everything else is fine to leave as-is. Once it's setup, the server will need to reboot.

### Testing the DC

Now that I'm booted back up and logged into ZOFFLAB/Administrator, I want to run a few tests just to make sure that everything installed cleanly before I spend a lot of time configuring things. I will note testing methods here and will include the actual troubleshooting steps for failed tests in the [troubleshooting section](#troubleshooting).

1. Run `dcdiag`. This command runs a list of tests to determine the overall health of the DC. Each test will output passed/failed. (a couple failed)
2. Run `nslookup zofflab.local` to verify DNS is working (it is)
3. ping `8.8.8.8` to make sure that the internet is still accessible

### Basic Verification & Hardening

Before I start creating OUs and assigning objects, I want to make sure that basic security is in place.

**1. Update Windows** 

The most obvious first step is to check for Windows updates. Even though I installed all available updates before promoting to DC, DCs come with their own host of security patches.

**2. Scope RDP access**

Because I already have remote access to Proxmox via the VPN, I don't need a separate remote access ability specifically to the DC. RDP is disabled by default, but it's always good to check just in case.

`Server Manager > Local Server > Remote Desktop > Disabled`

**3. Check Windows Firewall**

Windows automatically configures the firewall for all three profiles (Domain, private, public), but it's always good to verify that the needed rules are implemented and enabled.

`Windows Defender Firewall with Advanced Security > Inbound Rules` and filter by `Active Directory Domain Services` group. This shows that all rules are implemented and enabled and allow communication between all profiles.

**4. Disable Print Spooler**

I'm not going to be using this VM for printing, and there have been attacks using this vector in the past (See [CVE-2021-34527](https://nvd.nist.gov/vuln/detail/cve-2021-34527)). Because of this, I'll just disable the print spooler altogether via PowerShell.

```
Stop-Service -Name Spooler
Set-Service -Name Spooler -StartupType Disabled
```

**5. Disable SMBv1**

This one is pretty obvious since SMBv1 was the reason EternalBlue worked. This is usually disabled by default, but considering the consequences if looked over, it's good to double-check.

`Get-WindowsOptionalFeature -Online -FeatureName SMB1Protocol`

Returns: Disabled

**6. Password and Lockout Policies**

The default password policies for Windows are fine for this environment but I'll change a couple of them anyways just to be safe. The lockout policy however is set to 0, meaning unlimited login attempts with no lockout. That's something I definitely need to change.

`Group Policy Management > Forest > Domains > zofflab.local > Default Domain Policy > Right click > Edit > Computer Configuration > Policies > Windows Settings > Security Settings > Account Policies > Password Policy`

The maximum password age is 42 days by default, but for convenience, I'm going to set this to 90 days. I'll also change the minimum length to 12 since NIST values length over complexity ([NIST SP 800-63B](https://pages.nist.gov/800-63-4/sp800-63b/passwords/)), and double-check that passwords are not stored with reversible encryption.

From the same path as above but under `Account Lockout Policy`

By default there is no account lockout threshold, I set mine to 5 and left the default lockout times.

**7. Disabling LLMNR & NetBIOS Name Resolution**

LLMNR (Link-Local Multicast Name Resolution) and NetBIOS Name Resolution (an older, similar fallback) are both fallbacks for DNS. If DNS fails, these protocols will broadcast a query to the local subnet asking if any device knows the answer. The problem with this is that *anyone* can reply, which means an attacker can just sit on the local network and wait to respond to a domain resolution query. This is especially problematic if this results in the attacker receiving NTLM auth hashes.

Disabling LLMNR can be done directly through GPO:

`Default Domain Policy > Computer Configuration > Policies > Administrative Template > Network > DNS Client`

Enable `Turn off multicast name resolution`

To turn off NetBIOS:

`Control Panel > Network and Internet > Network and Sharing Center > Change adapter settings > Right click on adapter > Properties > Internet Protocol Version 4 (TCP/IPv4) > Properties > Advanced > go to the WINS tab > select Disable NetBIOS over TCP/IP`

### Advanced Auditing Policy

Setting up advanced auditing will knock out two birds with one stone since it's good practice, and the logs will also be used in the future for log aggregation tools like Splunk/SIEM. It's also good to have these configured before creating any objects or OUs so that I'm not missing any logs.

In the same Group Policy Management Editor window from before, `Computer Configuration > Windows Settings > Security Settings > Advanced Audit Policy Configuration`

There are a few specific policies here that I want to configure. Since I already used the GUI for the password and account lockout policies, I'll use Powershell for these.

```
# Account Logon
auditpol /set /subcategory:"Kerberos Authentication Service" /success:enable /failure:enable
auditpol /set /subcategory:"Kerberos Service Ticket Operations" /success:enable /failure:enable

# Logon/Logoff
auditpol /set /subcategory:"Logon" /success:enable /failure:enable
auditpol /set /subcategory:"Special Logon" /success:enable /failure:disable

# Account Management
auditpol /set /subcategory:"User Account Management" /success:enable /failure:enable
auditpol /set /subcategory:"Security Group Management" /success:enable /failure:enable
```

Note: Failure is turned on for all policies except "special logon". This is because special logons don't have a distinct "failure" concept. Instead, failed special logons are logged with regular "logon" logs.

Verified with `auditpol /get /category:"Account Logon","Logon/Logoff","Account Management"`.

Extra Note: these settings were applied locally via `auditpol` instead of using the GPO's Advanced Audit Policy Configuration node itself. For a single-DC lab where the same admin controls both, this is low-risk, but it's worth noting that local `auditpol` settings can theoretically be overwritten by a future Group Policy refresh if the GPO node is ever configured differently.

# Troubleshooting

[**~Section/Testing the DC**](#testing-the-dc)

* **Issue:** `dcdiag` reported a failed `DFSREvent` test, citing warnings/errors related to SYSVOL within the last 24 hours
* **Deduction:** After doing some research on what this meant, I initially assumed that this test was failing because there's only one DC (DFSR's name (Distributed File System Replication) implies multiple nodes are needed). I still wanted to double check exactly what the error was, however. In the event viewer logs for DFSREvent, there's one initial error at 1:56PM followed by info-level logs and another warning at 2:20PM. The error output states, "The specified domain either does not exist or could not be contacted". At the time of this error, DNS was not yet fully functional (it takes some time after promotion to start working properly), so it's highly likely that that was the problem.
* **Solution:** No fix required besides waiting. Subsequent DFSR events showed routine information-level activity without warnings or errors, suggesting that the issue has been resolved. The DFSREvent test will fail if there are any errors within the last 24 hours, so this test will pass after that amount of time. (Hello, it's me 24 hours later. All tests pass now.)
