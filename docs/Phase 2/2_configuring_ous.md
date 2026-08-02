Author: James Sleptzoff

File: 2_configuring_ous.md

Created: July 31, 2026

Last Modified: July 31, 2026

# Goal

Now that the domain controller is setup with basic security and logging, I can set up Organizational Units (OUs). Organizational units sit inside of the domain, and contain objects such as users, groups, and computers. By the end of this task, I want to have two OUs: one for administrators and one for standard users. Because this is a homelab with a small scope, I won't need department-specific units or further granularity, at least for now. Instead, I'll split departments into their own groups in the standard users unit.

**Administrators**

For the administrators unit, I want to have one admin user and one admin group. Since there's already a builtin Administrators local group, I'll name the user zoff, and the group zodmin.

**Standard Users**

For the standard users, I want to set this up in a way more similar to an actual enterprise environment. To do this, I'll make the following:

* **Users:** Alice, Bob, Carol, and Dave
* **Groups:** Marketing, Accounting, and Management
* **Computers:** They will all get one computer because the company is struggling to afford RAM.

Alice and Bob will both be in the marketing department so that they can bounce ideas off of each other, Carol will be the accountant, and Dave will be the manager.

Accounting and Marketing will only be able to access their own respective groups, and Dave will only be a member of the management group, but will have read-only access to accounting and marketing. I decided on read-only access for two main reasons:

1. To ensure that Dave does not inadvertently or purposefully modify anything in either group.
2. To ensure least-privilege should his account be compromised.

## Setting up the OUs

### Creating OUs and objects

New OUs and objects can be created from `Active Directory Users and Computers`. From there, I right-clicked on my local domain `zofflab.local` and selected `New > Organization Unit`.

Once both units are created, I can right-click on them and add new users, groups, and computers. For each of the new users, I made sure to disable them after creation. This will ensure that there's no attack window open while I'm configuring them, which is especially important for the administrator unit.

### Assigning users to groups

Adding users to groups can be done in one of two ways. Either you can double-click on each user individually, or you can double-click on the group and add users at the same time.

`Double-click on group > Members > Add` and for multiple users, type each name separated by a semi-colon and click okay, then apply.

I also made sure that Marketing and Accounting were both managed by dave. This isn't necessary, and doesn't affect permissions at all, but it's good practice for documentation purposes as it will allow users to see who other users are managed by in case they want to contact them.

### Setting permissions

**Administrators**

For the Administrators group, I want to give the zodmin group specific permissions since there's already a default domain admin account with full permissions. Setting up the Administrators unit would be pointless if they just have all the same abilities as domain admins (besides for logging purposes since you would see which admin account logs are coming from).

To do this, for each of the OUs, right-click and select `Delegate Control`. This will bring up a delegation wizard and add the zodmin group. After adding the zodmin group, I selected the following permissions:

* Create, delete, and manage user accounts
* Reset user passwords and force password change at next login
* Modify the membership of a group
* Join a computer to the domain

Each of these are standard administrator capabilities, but it still ensures that admins in this group do not have *full* control.

**Standard Users**

For the standard users, I set each of the groups' scope to **Global**. This just means that the groups collect users within the domain and be nested into other groups or granted permission elsewhere. Since these groups are just collections of users within a single domain, this is fine. This differs to how I implemented the `zodmin` group, which I set to the **Domain Local** scope. This scope is specifically intended for a group that holds delegated permissions directly, rather than one meant to be nested or portable across domains.

In my original plan, I stated that I want the Management group to just have read-only permissions to the Accounting and Marketing group for least-privilege. I considered adding Dave to the two groups directly, but since groups are binary, you cannot pick and choose permissions for different users within the group; they must inherit the *full* set of permissions. This is mainly because I have not yet decided *exactly* what to do with these three groups, and until I've decided, I don't want to be over-permissive. Technically, least privilege for a user or group with no *real* direction as of yet would be to not have it, but since I am using this to help progress my IT skills, read-only access feels like a reasonable choice considering it's a manager role, even if not completely necessary right now.

In order to implement this, I'll have to create shared folders, and grant the Management group read-only access to them. This means I will have to implement NTFS permissions on shared folders using a fileserver.

# Troubleshooting

[**~/Section/Setting up the OUs/Setting permissions**](#setting-permissions)

* **Issue:** Accidentally delegated zodmin control domain-wide.
* **Solution:** Turn on advanced features: `Active Directory Users and Computers (ADUC) > View > Advanced Features`. Then `Right-click domain > Properties > Security` and remove the zodmin group and apply. Thankfully, each user was disabled when I made them (for this exact reason).
