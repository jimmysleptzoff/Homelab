Author: James Sleptzoff

File: 2_installing_splunk.md

Created: August 9, 2026

Last Modified: August 18, 2026

# Goal

The goal for this task will be to setup a VM and install Splunk on it. After this, I will need to ensure that there is internet access, and install needed updates. The next task will cover basic hardening steps, index configuration, and other general setup.

## Installing the VM

The process here is the same as it's been throughout the project: download the ISO, add it to Proxmox, and create the VM.

Splunk recommends a minimum of 12 cores and 12GB of RAM to run Splunk, but because that's for real-life enterprise environment, I don't need to give it nearly that much. Since this laptop is being used purely for the homelab, I'm going to give it 2 cores, 6GB of RAM, and 40GB of disk space.

## Installing Splunk

Now that the VM is setup and Debian is installed, I can install Splunk. The Enterprise free trial can be found on the [Official Splunk Website](https://www.splunk.com/en_us/products/splunk-enterprise.html).

Once downloaded on the VM, I can verify the hash using `sha512sum`, and then use `dpkg -i` to install Splunk.

Splunks `.deb` package automatically creates a dedicated `splunk` system user. This is created so that Splunk can run as a non-root user (which you can still do, but is deprecated). By default, this account is able to be logged into, but this should be disabled (I've opted to put this in the next task file since it's basic hardening).

Since `dpkg -i` installs and owns everything as root regardless of who invokes it, I handed ownership of the install directory to the `splunk` user before starting it for the first time:

```bash
sudo chown -R splunk:splunk /opt/splunk
```

Once ownership is changed, I started Splunk:

```bash
sudo -u splunk /opt/splunk/bin/splunk start --accept-license
```

Once this completes, I'm able to access the Splunk web GUI.

![splunk-web-gui](/assets/images/phase3/splunk-web-gui.png)

By default, Splunk uses HTTP instead of HTTPS. This is an easy switch:

Navigate to `Settings > Server Settings > General Settings` and turn on HTTPS. After this, Splunk will need to be restarted from `Settings > Server Control`.

**Note:** Apparently the only way to update splunk is to go to their website, download the update, stop splunk, install the update, and then start splunk back up again. If that seems kind of ridiculous, I think so too. (I *could* make a script for this but considering my free trial will run out at some point, I will hold off on it.)

Last, to make sure Splunk boots at startup, I can enable `boot-start`:

```bash
sudo -u splunk /opt/splunk/bin/splunk enable boot-start
```

## Setting up the CA

Just like with pfSense, since this is a locally hosted web GUI, I will get a certificate error since it's being self-signed. This isn't a big deal, but it's slightly annoying and I would like to get rid of it. Thankfully, this should be easier than last time since I already have the internal CA setup.

1. Create a new certificate

    On pfSense, go to `System > Certificates > Certificates` and click `Add/Sign`. This will be the certificate that I use for Splunk, so I named it as such and added the splunk.zofflab.local domain name.

2. Export key, cert, & CA

    On the same page, click the `Export Certificate` and `Export Key` buttons. This will download both of them.

    On the `Authorities` page, click the `Export CA` button. This is what will be added to my browser.

3. Copy the files to the server

    First, I made a new directory on the Splunk server to hold the certs so that they'll be easier to find/manage in the future if I ever need to:

    `sudo mkdir /opt/splunk/etc/auth/splunk_certs/`

    All of the files I just downloaded needed to be sent over to the Splunk server. To do this, I just used `scp`:

    ```bash
    scp ZoffLab-CA.crt james@192.168.20.102:/home/james/Downloads
    scp splunk-cert.key james@192.168.20.102:/home/james/Downloads
    scp splunk-cert.crt james@192.168.20.102:/home/james/Downloads
    ```

    [See Troubleshooting](#troubleshooting)

4. Add keys to Splunk

    From here, I moved the `splunk-cert.key` and `splunk-cert.crt` files to the `splunk_certs` directory using:

    ```bash
    mv ~/Downloads/splunk-cert.key /opt/splunk/etc/auth/splunk_certs/
    mv ~/Downloads/splunk-cert.crt /opt/splunk/etc/auth/splunk_certs/
    ```

    After, I added both of their respective path to the `web.conf` file:

    ```ini
    [settings]
    enableSplunkWebSSL = true
    sslPassword = <password>
    privKeyPath = /opt/splunk/etc/auth/splunk_certs/splunk-cert.key
    serverCert = /opt/splunk/etc/auth/splunk_certs/splunk-cert.crt
    ```

5. Add CA to browser

    The last thing to do is to add the CA as a trusted CA on the browser. This will vary from browser-to-browser, but in FireFox I just searched "certificate" in the settings and imported the CA from the file copied.

# Troubleshooting

[**~/Section/Installing Splunk**](#installing-splunk)

* **Issue:** When installing Debian, I had to make a user account which (annoyingly but understandably) isn't added as a sudoer.
* **Solution:** Add to sudoers group:

```bash
su - root
usermod -aG sudo james
```

---

[**~/Section/Setting up the CA**](#setting-up-the-ca)

* **Issue:** `scp` wasn't working from local to remote
* **Deduction:** `scp` uses `ssh` to send files securely so I figured that ssh probably just wasn't on the remote machine. After checking with `sudo systemctl status ssh`, I discovered that for some reason, ssh isn't even installed. This is usually packaged with Debian out of the box, but isn't a big deal (unless I got some other ssh service).
* **Solution:** Installed `openssh-server` and ran it temporarily using `sudo systemctl start ssh`. After copying all files, I stopped & disabled it using `sudo systemctl stop ssh && sudo systemctl disable ssh`.
