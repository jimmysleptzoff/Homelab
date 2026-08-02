Author: James Sleptzoff

File: 3_setting_up_a_fileserver.md

Created: July 31, 2026

Last Modified: August 1, 2026

# Goal

The goal of this task will be to implement a fileserver that will be used to set up shared files on the Windows Active directory. This will run on Windows Server using the CLI interface on the VLAN20 (Servers/Infrastructure) segment. Once the fileserver is setup, I will use it to create shared folders for the Marketing and Accounting groups and NTFS (New Technology File System) permissions to give the Management group read-only permissions to the shared folders.

In order to test both the fileserver and the Windows AD, I will setup another VM running Windows 11, and join it to the domain. Once joined, I will log in as one of the users from the groups I configured in the last task and ensure that the shared folders are accessible, with the correct permissions configured.

## Setting up Fileserver

To set up a fileserver, I will be using Windows server 2025 (Evaluation) without a Desktop interface. Not having a desktop interface means that I can allocate less resources to the VM and still have decent performance for my needs.

### Install and Configure the Fileserver

Once the iso is installed, I'm loaded into the SConfig menu. From here I can properly install configure the server.

1. Rename the computer to "fileserver"
2. Set static IPv4 to `192.168.20.101/24` (on the VLAN20 segment)
3. Point DNS to the Domain Controller at `192.168.20.100`
4. Disable remote management
5. Install updates
    * Wait way too long for the updates
    * Think the updates are finished and then install 4 more updates
    * Think about that one Reid Wiseman quote from Artemis II, "I have two Microsoft Outlooks and neither one is working"
6. Join the `zofflab.local` domain
7. Verify that the time/date is correct
8. Enable file and printer sharing: `Enable-NetFirewallRule -DisplayGroup "File and Printer Sharing"`
9. Install file and storage services, which are needed to set this VM up as an actual file server using: `Install-WindowsFeature -Name FS-FileServer -InstallManagementTools`

### Create the Shared Files

Now that I have fileserver configured properly, I can set up the actual Marketing and Accounting shared files using the following powershell commands:

```powershell
# Create the folders
New-Item -Path "C:\Shares\Marketing" -ItemType Directory
New-Item -Path "C:\Shares\Accounting" -ItemType Directory

# Share them (share-level permissions kept loose, NTFS does the real restricting)
New-SmbShare -Name "Marketing" -Path "C:\Shares\Marketing" -FullAccess "Authenticated Users"
New-SmbShare -Name "Accounting" -Path "C:\Shares\Accounting" -FullAccess "Authenticated Users"
```

## Configure File Permissions

Once the files are created, I have to configure the permissions so that users can modify files within their respective group folder. Then I can grant read-only access to the Management group using the following powershell commands:

```powershell
# Marketing folder: grant Marketing group Modify, Management group Read
icacls "C:\Shares\Marketing" /grant "ZOFFLAB\Marketing:(OI)(CI)M"
icacls "C:\Shares\Marketing" /grant "ZOFFLAB\Management:(OI)(CI)R"

# Accounting folder: grant Accounting group Modify, Management group Read
icacls "C:\Shares\Accounting" /grant "ZOFFLAB\Accounting:(OI)(CI)M"
icacls "C:\Shares\Accounting" /grant "ZOFFLAB\Management:(OI)(CI)R"
```

# Troubleshooting

[**~/Section/Set up the Workstation**](#set-up-the-workstation)

* **Issue:** When I originally setup the OUs, I pre-added a workstation computer to the standard users unit. However, Windows will automatically add computer to the `Computers` container once joined to the domain.
* **Solution:** The fix is easy, all I had to do was drag and drop the workstation computer from the `Computers` container into the `Standard Users` OU from ADUC, and delete the workstation computer that I had previously added.
