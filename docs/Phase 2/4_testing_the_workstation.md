Author: James Sleptzoff

File: 4_testing_the_workstation.md

Created: July 1, 2026

Last Modified: August 1, 2026

# Goal

The goal of this task is to stand up a Windows 11 Enterprise VM to act as the AD workstation. Because of resource limitations, this will act as a shared workstation for all users connected on the domain.

Once installed, I will need to join the workstation to the domain, and verify that I can access the shared folders that I created from the fileserver in the previous task file. From here, I will need to also verify that permissions were properly configured to allow users to access only their respective folder, with managers having read-only access.

## Set up the Workstation

Now that the AD and fileserver are both setup and configured, I now want to set up the main workstation computer that the users in the domain will use. To do this I have opted to use Windows 11.

Once installed, I logged into the VM and joined the computer to the `zofflab.local` domain:

Navigate to `Settings > System > About > Domain or workgroup > change > Domain` and enter `zofflab.local`. When prompted for credentials, I used the admin account `zoff` in the `zodmin` group. After this, a restart is required.

## Testing the workstation

Once the VM restarts, I can see that this computer is now part of the ZOFFLAB domain. From here I was able to successfully log in as Alice to test the shared folders.

At first, I was confused as to why I couldn't see the files. This is because the files that I set up on the fileserver are not accessible through the file explorer by default. This means that I will just have to map it as a network drive:

1. Open File Explorer
2. Right-click on `This PC`
3. Select `Map Network Drive`
4. Enter `\\fileserver\Marketing`

I chose to use M for the drive letter (for marketing). Also, to ensure that it's persistent, I made sure that `Reconnect at sign-in` was checked.

Now, I'm able to access the drive from the GUI instead of having to type it into the file explorer search bar.

To test the permissions, I want to make sure that users in the Marketing group cannot access or modify files in the Accounting folder, and vice versa. I also want to make sure that Management has read-only access to both folders.

To my surprise, Alice and Bob were both able to access and modify files in the Accounting file, even though they're in the Marketing group. More on this in the [troubleshooting section](#troubleshooting).

After fixing permissions, I have now verified that Alice and Bob cannot access the Accounting folder. Lastly, I want to check management permissions to ensure that read-only access was properly granted.

![management-read-only](/assets/images/management-read-only.png)

As shown above, management (dave) is able to view both the Marketing and Accounting drives, but is unable to modify, create, or delete anything within them (Denied when trying to create a new file. I also tried to write to `passwords.txt` and was denied when trying to save.)

# Troubleshooting

[**~/Section/Testing the Workstation**](#testing-the-workstation)

* **Issue:** Alice and Bob are both able to access and modify files in the Accounting folder. This goes against the permissions that I set for them.
* **Deduction:**

1. My first instinct is to check the group permissions for the groups on the Domain Controller. This is set up correctly and is not a finding.
2. My second instinct is to check the access permissions that I set up on the fileserver.

    ![marketing-permisisons-from-fileserver](/assets/images/marketing-permissions-from-fileserver.png)

    `BUILTIN\Users` do have separate permissions, which I thought might be the issue. I checked to make sure that the users I created in the standard users OU weren't also added to the builtin users container. They were not, so this is also not a finding. (This became a finding, see below)

3. Next, I checked to make sure that the smb shares were properly configured on the file server.

![smbshares-from-fileserver](/assets/images/smbshares-from-fileserver.png)

Here's the problem. When I first configured the smb shares, I gave `Authenticated Users` full access at the *share* level. `Authenticated Users` is a built-in group that includes every logged-in domain account regardless of the department, so Alice and Bob get full control here.

Windows evaluates share-level and NTFS-level permissions together and applies whichever is more restrictive *per use*, based on whatever each layer actually grants them. Since NTFS didn't have an explicit entry restricting Alice or Bob below what the share allowed, the share's Full Control access effectively went through.

* **Solution:** 

To fix this, I revoked `Authenticated Users` from both shares and granted access specifically to each department's group instead:

```powershell
# Marketing
Revoke-SmbShareAccess -Name "Marketing" -AccountName "Authenticated Users" -Force
Grant-SmbShareAccess -Name "Marketing" -AccountName "ZOFFLAB\Marketing" -AccessRight Full -Force
Grant-SmbShareAccess -Name "Marketing" -AccountName "ZOFFLAB\Management" -AccessRight Read -Force

# Accounting
Revoke-SmbShareAccess -Name "Accounting" -AccountName "Authenticated Users" -Force
Grant-SmbShareAccess -Name "Accounting" -AccountName "ZOFFLAB\Accounting" -AccessRight Full -Force
Grant-SmbShareAccess -Name "Accounting" -AccountName "ZOFFLAB\Management" -AccessRight Read -Force
```

And I can verify:

![fixed-smbshareaccess](/assets/images/fixed-smbshareaccess.png)

And re-test from Alice's account:

![alice-cant-get-to-accounting-folder](/assets/images/alice-cant-get-to-accounting-folder.png)

Alice and Bob both no longer have access to the Accounting folder. I made sure to remove the network drive on the client, forcing Windows to drop the cached connection.

---

* **Follow-up Issue:** After fixing the share-level permissions, I went back to double check the NTFS ACLs directly and reconsidered the `BUILTIN\Users` permissions for both folders (Read & Execute, Append Data, and Write Data all granted). This meant that *any* domain user could still create or modify files in either folder at the NTFS level, independent of the share-level fix I just implemented.

* **Deduction:** Running `icacls` showed that the `BUILTIN\Users` entries were marked with an `(I)` flag, meaning they were inherited from the parent `C:\Shares` folder rather than explicitly set. In order to remove these rules I will have to explicitly remove the inherited flag, and then remove each of the permissions. Also, since these permissions are inherited from `C:\Shares\`, I want to make sure that I remove the permissions are removed from there as well to make sure that users cannot see what shared folders exist.

* **Solution:**

To fully remove the permissions, I had to break inheritance on each folder first, which will convert it to a self-contained ACL with no connection to the parent folder. 

There are two main ways to do this. Using the d option removes the inherited permissions and copies them back without the inherited flag. Using the r option disabled inheritance and removes all of the permission without copying them back:

```powershell
icacls "C:\Shares\Marketing" /inheritance:d
icacls "C:\Shares\Accounting" /inheritance:r
```

With inheritance stripped from the Marketing folder, I removed the `BUILTIN\Users` permissions directly:

```powershell
icacls "C:\Shares\Marketing" /remove "BUILTIN\Users"
```

For the Accounting folder, I need to add back the `SYSTEM`, `BUILTIN\Administrators`, and `CREATOR OWNER` permissions:

```powershell
icacls "C:\Shares\Accounting" /grant "SYSTEM:(OI)(CI)F"
icacls "C:\Shares\Accounting" /grant "BUILTIN\Administrators:(OI)(CI)F"
icacls "C:\Shares\Accounting" /grant "CREATOR OWNER:(OI)(CI)(IO)F"
```

Verifying with `icacls` afterwards confirmed that both folder no longer grant permissions to all users in the `BUILTIN\Users` container.

![builtin-users-perms-removed](/assets/images/builtin-users-perms-removed.png)

---

* **Issue:** As mentioned above in the previous issue, the `BUILTIN\Users` permissions that were assigned to the Marketing and Accounting folders were inherited from the `C:\Share\` folder. This of course means that that folder has those same permissions, and I want to remove them so that users cannot view all of the folder in `C:\Shares`. It also means that anything that I create in this folder in the future will also inherit the same built-in user permissions that I just spent time removing on the `Accounting` and `Marketing` folders.

![shares-with-user-perms](/assets/images/shares-with-user-perms.png)

* **Solution:**

Strip inheritance and remove the `BUILTIN\Users` permissions:

```powershell
icacls "C:\Shares" /inheritance:d
icacls "C:\Shares" /remove "BUILTIN\Users"
```

I also made sure to set the `FolderEnumerationMode` share permissions on both the `Marketing` and `Accounting` folders to `AccessBased` to ensure that they cannot be enumerated via SMB.

```powershell
Set-SmbShare -Name "Marketing" -FolderEnumerationMode AccessBased
Set-SmbShare -Name "Accounting" -FolderEnumerationMode AccessBased
```

Then of course I verified:

![enum-set-to-accessbased](/assets/images/enum-set-to-accessbased.png)

Now enumeration is set to access based and not unrestricted as it was before.
