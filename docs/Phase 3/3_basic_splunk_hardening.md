Author: James Sleptzoff

File: 3_basic_splunk_hardening.md

Created: August 18, 2026

Last Modified: August 19, 2026

# Goal

The goal for this task is to configure basic security hardening for Splunk now that it's up and running. To do this, I will check/configure the following items:

#### OS-level

* Splunk user account access
* SSH

#### Splunk-level

* Admin account password policy
* Session timeouts
* SSL/TLS

#### Auditing/logging

* Check Splunk's own internal logs/index

## OS-level checks/configs

### Splunk user account

Splunk automatically creates a new `splunk` user when installed. They leave it up to the user to determine how they want to configure it. Generally, turning off the ability to log into it at all is a good idea.

```bash
sudo usermod -s /usr/sbin/nologin splunk
```

The above command switches the splunk shell to a special `nologin` shell that refuses login attempts.

### SSH Hardening

As I mentioned in the previous task file, SSH (for whatever reason,) wasn't included when I installed Debian on this VM. I therefore installed openSSH and used that for scp. When I did this, I only had it on temporarily, and then disabled it. It is good, however, to verify this.

```bash
sudo systemctl stop ssh 
sudo systemctl disable ssh
sudo systemctl status ssh
```

This verifies that SSH is not running and is indeed disabled.

### Deferments

I considered using `nftables` to configure the host firewall to restrict outbound traffic to VLAN10 and VLAN30 but ultimately decided to defer this for the time being. This *would* be a good defense-in-depth and redundancy task (since pfSense is already doing that,) however, I have decided that implementing the rules now may cause problems during alert/aggregation setup. I may revisit this at a later date once those are setup.

## Splunk-level checks/configs

### Password Policy

Found on the GUI: `Settings > Password management`

By default, there are only a few password policies that are enabled on Splunk (mainly lockout & password length requirements). I've opted to additionally require 2 numerals, 1 uppercase, 2 lowercase, and 1 special character. I've also enabled expiration after 90 days, but have left off password history.

### Session Timeouts

As I've been using Splunk, I have already noticed that sessions do indeed timeout after a set period of time. However, this is still something that I want to check/verify.

Found on GUI: `Settings > Server settings > General settings > Session timeout`

From here, I verified that session timeout is set to 1hr, which I will keep as-is.

### SSL/TSL Hardening

To prevent downgrade attacks, I want to force modern ciphers **only**. I did this by adding a few lines in `web.conf`

Full `web.conf` path: `/opt/splunk/etc/system/local/web.conf`

```ini
[settings]
...
sslVersions = tls1.2,tls1.3
cipherSuite = TLSv1.2:!eNULL:!aNULL
ecdhCurves = prime256v1, secp384r1, secp521r1
```

The above addition under `[settings]` will force a TLS1.2/1.3 connection (Note: whenever these versions become deprecated, this will need to be updated. Though, it's unlikely to cause issues anytime soon.) Additionally, it rejects null-encryption/null-auth cipher suites. Last, it restricts which elliptic curves are allowed during the handshake.

After the new settings are written, Splunk will need to be restarted:

```bash
sudo -u splunk /opt/splunk/bin/splunk restart
```

### Internal logs

Splunk generates and stores logs about itself, which is great, but, by default, there isn't a limit to how long they're stored for. For an enterprise with a lot of storage space, that's not as big of a deal, but for a homelab with limited space, it can become a problem.

This VM has 40GB of space allocated to it, with ~21.4GB available. I've opted to allocate at least 10GB to all logs and 1GB for internal logs. There are two ways that I can go about implementing this:

1. Hard cap on all logs

    Setting a ceiling for all logs at first seems like the better choice (it's definitely the easier choice,) however, once that limit is reached, Splunk decides what to get rid of. So for instance, DNS is pretty noisy and will generate a lot of logs. I would rather get rid of the DNS logs as opposed to something like Windows security events, but with a global cap, Splunk gets to decide which of the two to keep.

2. Source-by-source caps

    This method directly solves the issue presented above, but requires me to manually keep track of how much space I have left for overhead. Since this is a small environment with 1 admin, and only a handful of sources, this tradeoff is something I'm completely fine with, so I'll be going with this option.

To do this, I created a new file: `/opt/splunk/etc/system/local/indexes.conf` to write the rules in.

```ini
[volume:primary]
path = /opt/splunk/var/lib/splunk
maxVolumeDataSizeMB = 10240

[_internal]
homePath = volume:primary/_internal/db
coldPath = volume:primary/_internal/colddb
thawedPath = $SPLUNK_DB/_internal/thaweddb
maxTotalDataSizeMB = 1024
frozenTimePeriodInSecs = 1209600
```

After saving this file, I restarted Splunk and verified that the new rules were recognized.

```bash
sudo -u splunk /opt/splunk/bin/splunk restart
sudo -u splunk /opt/splunk/bin/splunk btool indexes list _internal --debug
```
