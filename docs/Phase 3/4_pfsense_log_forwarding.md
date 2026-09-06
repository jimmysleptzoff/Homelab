Author: James Sleptzoff

File: 4_pfsense_log_forwarding.md

Created: August 20, 2026

Last Modified: September 6, 2026

# Goal

Now that Splunk is properly setup on its own Debian VM, with basic hardening and security in place, I want to start the process of actually forwarding and aggregating logs. First I'll start with pfSense since it's acting as the network firewall for the homelab. To do this, I'll have to forward syslog data events to Splunk using Splunk Connect for Syslog (SC4S). This follows the current best practice that Splunk recommends for syslog as opposed to direct syslog to indexer input.

## Reasoning

Originally, I was going to use direct data input from pfSense to Splunk using a TA (Technology Add-on), but after reading up on some [Splunk documentation](https://splunk.github.io/splunk-connect-for-syslog/1.110.1/sources/Pfsense/), I learned that if I use SC4S, I won't need to use a TA.

Beyond this, Splunk recommends using SC4S over other methods for general syslog data collection implementations without existing syslog infrastructure (Which can be found under "Syslog data collection implementations" [here](https://help.splunk.com/en/splunk-enterprise/splunk-validated-architectures/getting-data-in-forwarding-and-preprocessing/syslog-data-collection)).

## Environment

- Splunk Enterprise (trial), Debian VM, VLAN20, `192.168.20.102`
- Docker installed via Docker's official APT repo (not Debian's bundled `docker.io`)
- SC4S container image: `ghcr.io/splunk/splunk-connect-for-syslog/container3:latest` (hosted on GHCR, **not** Docker Hub — `splunk/scs` doesn't exist)
- pfSense: `192.168.20.1` (VLAN20 interface)

---

# Steps

**Note:** These steps are in a "final state" and have been modified or rewritten entirely. See [troubleshooting](#troubleshooting).

### 1. Indexes

In order to create a new HEC token (more on what that is in the next step), you have to create the indexes that you want to use. This will allow you to filter logs from certain sources so that they're not all clumped together under one giant aggregate.

Splunk uses CIM-style (Common Information Model) naming conventions for the pfSense source which can again be found in their [documentation](https://splunk.github.io/splunk-connect-for-syslog/1.91.5/sources/Pfsense/).

The two indexes that I will need to create are `netops` for general pfSense logs like cron, dhcp, nginx/web, daemons, etc. and `netfw` for firewall/filterlog events.

1. In Splunk, navigate to `Settings > Indexes > New Index` and create both of the mentioned indexes.

2. Each index will need to be moved into the `primary` volume that I created in the last task. This can be done by modifying the `/opt/splunk/etc/system/local/indexes.conf` file:

```ini
...

[netops]
homePath = volume:primary/netops/db
coldPath = volume:primary/netops/colddb
thawedPath = $SPLUNK_DB/netops/thaweddb
maxTotalDataSizeMB = 768

[netfw]
homePath = volume:primary/netfw/db
coldPath = volume:primary/netfw/colddb
thawedPath = $SPLUNK_DB/netfw/thaweddb
maxTotalDataSizeMB = 1280

```

This file is also where I set the size of each of the respective indexes. I decided to go with 0.75GiB for `netops` and 1.25GiB for `netfw`. This choice is because there is likely to be more firewall logs than general purpose logs. (Also Splunk seems to use MB when they mean MiB for some reason, which is why I'm using binary values).

### 2. HEC (HTTP Event Collector) setup

The SC4S implementation uses an HEC (HTTP Event Collector) to send syslog data to Splunk for indexing and needs to be set up on the Splunk-side in order to receive syslogs from pfSense.

**On Splunk:**

1. Navigate to `Settings > Data Inputs > HTTP Event Collector > Global Settings` and set `All Tokens` to `Enabled`. Also ensure that `Enable SSL` is checked.

2. Create a new token named `sc4s-token` and scope it to the two indexes that were just created for least-privilege enforcement. **Note:** the default index here doesn't really matter.

Take note of the `Token Value`. You don't need to write it down but it will be used shortly.

### 3. Docker Install

SC4S runs on a container so I'll be using Docker to set that container up, but any OCI-compliant (Open Container Initiative) container will work.

Debian comes with Docker installed but it usually lags behind since Debian was built for stability/security and will favor those over QoL of other major updates. I'd rather take the latter, however, so I'll install the latest version of docker manually.

```bash
# Update and install prerequisites
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg

# Add Docker's GPG key
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Add the repo
echo \
    "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian \
    $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
    sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
```

Once I had it installed I verified that it worked using:

```bash
docker run hello-world
```

### 4. Create SC4S container

Now that docker is installed and verified working, I can configure and start the SC4S container itself.

```bash
sudo docker run -d --name SC4S \
  --restart unless-stopped \
  -p 5514:5514/udp \
  --env SC4S_DEST_SPLUNK_HEC_DEFAULT_URL=https://192.168.20.102:8088 \
  --env SC4S_DEST_SPLUNK_HEC_DEFAULT_TOKEN=<token> \
  --env SC4S_DEST_SPLUNK_HEC_DEFAULT_TLS_VERIFY=no \
  --env SC4S_LISTEN_PFSENSE_FIREWALL_UDP_PORT=5514 \
  ghcr.io/splunk/splunk-connect-for-syslog/container3:latest
```

### 5. pfSense remote logging configuration

pfSense itself of course does not enable remote logging by default. Thankfully it's easy to turn on.

**On the pfSense GUI:**

1. Navigate to `Status > System Logs > Settings > Remote Logging Options > Remote Syslog Contents`, and check `Everything`

2. Right above that, enter the remote log server and port. In my case, it's `192.168.20.102:5514`

3. Click `Save`

# Troubleshooting

As you might have guessed if you skimmed over the top of this file, there was a lot of troubleshooting that needed to take place in order to get things working. I will try my best to give enough detail to explain what was happening, while staying concise. Note that this list is not fully comprehensive, it's just the biggest/most time-consuming steps issues.

Most of the following issues lead to syslogs not being properly processed or stored by Splunk, so while the solution may not have fixed the issue on an individual basis, all of them together fixed the general logging issue.

### Issue 1: HEC token index scoping

- **Symptom:** `{"text":"Incorrect index","code":7}` HEC error code 7, `index=main` rejected.

- **Cause:** Originally, I had set up a `pfsense` index, not knowing that Splunk uses `netops` and `netfw`.

- **Solution:** SC4S has built-in default mappings for pfSense sources that need the `netops` and `netfw` indexes. Because of this, I removed the `pfsense` index and replaced it with the correct ones.

### Issue 2: TLS_VERIFY

- **Symptom:** `syslog-ng-ctl stats` showed the HEC destination's `dropped` counter kept increasing while the `written` counter stayed at 0 despite manual testing via `curl` to the URL/token succeeding.

- **Initial Solution (Incorrect):** I had seen a bug report on GitHub citing issue #1126 on `splunk/splunk-connect-for-syslog` as a known bug noting that `SC4S_DEST_SPLUNK_HEC_DEFAULT_TLS_VERIFY=no` was not being honored and just decided to disable SSL for HEC for the time being. This didn't really change anything, and I soon realized that the bug wasn't the cause of my issue (I may have overlooked the 8 word reply on the report saying that it had been fixed).

- **Recurrence:** Towards the end of this task, when things started working properly, I decided to turn SSL back on. This caused logs to no longer go through again even after adding back the `SC4S_DEST_SPLUNK_HEC_DEFAULT_TLS_VERIFY=no` to the SC4S container environment variables.

- **Correct Solution:** Added the 's' in 'https' to the `SC4S_DEST_SPLUNK_HEC_DEFAULT_URL` SC4S container environment variable and verified using:

```bash
sudo docker inspect SC4S | grep -A 10 '"Env"'
```

### Issue 3: Traffic landing in index=main (Main Issue)

- **Issue:** Data was being received by Splunk on UDP port 514 (which is what it was before this issue was fixed), but kept getting a `Failed processing http input` error noting `parsing_err="invalid_index='main'"`.

![index=main_error](/assets/images/phase3/index_main_error.png)

- **Analysis:**

To determine what was causing this, I first checked to make sure that the token that I had set up had the correct indexes scoped to it, which it did.

I also wanted to determine if traffic was being properly routed through the SC4S container so I used `tcpdump` to capture the traffic on the 514 udp port.

```bash
sudo tcpdump -i any udp port 514 -n
```

This showed that incoming traffic on that port was indeed being routed through docker via the docker0 interface. As another check, I decided to configure remote logging on pfSense to firewall logs only. This resulted in no data being sent through udp port 514 at all, even when I purposefully triggered firewall events using my VPN.

After I reenabled remote logging for everything, I decided to manually send a log to the HEC using curl:

```bash
curl -k https://192.168.20.102:8088/services/collector/event \
  -H "Authorization: Splunk <token>" \
  -d '{"event": "manual test event", "index": "pfsense"}'
```

These tests went through and were stored correctly:

![manual_curl_tests](/assets/images/phase3/manual_curl_test.png)

Which made things even more odd.

- **Solution:**

After going through the SC4S and Splunk docs, I came across the "The SC4S 'fallback' sourcetype" section in the [splunk-connect-for-syslog](https://splunk.github.io/splunk-connect-for-syslog/main/sources/) docs.

This told me exactly what the issue was. Basically, SC4S has a bunch of different parsers for different log types, and port 514 is the default port that it uses. However, that port isn't allowed to have a parser type assigned to it. Instead, you can manually set a parser to a custom port so that SC4S can properly parse the input data.

Since I was trying to use port 514, which can't have a parser tied to it, it was being categorized as "fallback". The error that I was getting strongly pointed towards this being the issue.

The fix to this was frustratingly simple: all I needed to do was change the UDP port from 514 to 5514, and assign the pfsense firewall parser (the only one available for pfsense) to port 5514:

```bash
sudo docker run -d --name SC4S \
  --restart unless-stopped \
  -p 5514:5514/udp \ # MODIFIED LINE
  --env SC4S_DEST_SPLUNK_HEC_DEFAULT_URL=https://192.168.20.102:8088 \
  --env SC4S_DEST_SPLUNK_HEC_DEFAULT_TOKEN=<token> \
  --env SC4S_DEST_SPLUNK_HEC_DEFAULT_TLS_VERIFY=no \
  --env SC4S_LISTEN_PFSENSE_FIREWALL_UDP_PORT=5514 \ # ADDED LINE
  ghcr.io/splunk/splunk-connect-for-syslog/container3:latest
```
