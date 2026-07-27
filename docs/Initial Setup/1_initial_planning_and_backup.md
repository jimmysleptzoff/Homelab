Author: James Sleptzoff

File: 1_initial_planning_and_backup.md

Created: July 11, 2026

Last Modified: July 27, 2026

---

# Goal

The goal of this task is to determine, configure, and set up the initial homelab environment as well as create a non-exhaustive list of available hardware.

# Steps

## Identify hardware

In order to set up the environment I will first need to decide what I would like to run my lab on, as well as other components that I may want to use in the future. At my disposal I have:

1. Old unused laptop with 32GB of RAM
2. Old unused router
3. Personal desktop (I don't use it much anyways)
4. Mom's old computer (has 8gb of RAM, but a decent gpu)
5. Raspberry Pi 3 B+

I have opted to use my old laptop as it has a decent amount of RAM, takes up a minimal amount of space, has decent compute, a built in UPS (Uninterruptible Power Supply), and is no longer being used.

## Preparing the hardware

In order to use my laptop as a the main component of my homelab, I will have to wipe it and install Proxmox. I could opt to tri-boot (Windows, Arch, Linux), but since I already have limited resources, I have decided just to go with a clean slate.

After careful consideration, I have chosen to go with Backblaze to backup the laptop. This will also allow me to backups of other drives in separate b2 buckets.

This process is pretty straightforward, here is an example of one of the commands I used to backup my laptop using rclone:

`rclone copy <remote-name>:<bucket-name> --checksum --exclude-from "C:\rclone\excludes.txt" --transfers 24 --checkers 24 --progress --retries 10 --low-level-retries 20`

The above command uses the `checksum` flag which uses hashes to verify files were properly copied. It also uses `transfers` and `checkers` to increase the number of transfer and checker "workers" that run in parallel.

Even though checksums are enabled, I would still recommend running an `rclone check` to double check that files were properly copied over.

Note that for the above command you will have to
1. Run powershell as administrator to avoid access denied errors
2. Create an excludes file and add it to your path

Cross-check the backup and local disk:

`rclone check <remote-name>:<bucket-name> --checksum --one-way --exclude-from "C:\rclone\excludes.txt" --checkers 32 
--progress --retries 10 --low-level-retries 20 --log-level ERROR`

Once everything looks to be in order, we're ready to move on to the next step.
