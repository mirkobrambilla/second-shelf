---
title: "Running a Minecraft Server at Home – Small Improvements"
date: 2026-04-16
tags: [home, minecraft, server, reused hardware, Java]
version: 1.1
---

# Running a Minecraft Server at Home – Small Improvements

## Introduction

After running the server for a while, a few things became clear.

Nothing was fundamentally broken. The server worked. It was stable. It did exactly what I needed.

But over time, small frictions started to show up:
- the machine was too big
- power consumption was higher than expected
- memory was tight
- some operations required being physically near the machine

None of these were critical problems.  
But all of them were worth fixing.

This article is not about rebuilding everything.  
It’s about **small, incremental improvements** that made the setup more practical over time.

## 1 – Replacing the hardware (from old PC to NUC)

The original setup was running on an old PC:
- bulky
- noisy
- ~100W power consumption

It worked, but it didn’t feel right for something that runs 24/7.

So I replaced it with a **NUC**:
- much smaller
- almost silent
- significantly lower power consumption

The migration itself was simple:
- same OS (Ubuntu)
- same folder structure
- same systemd service

This wasn’t a redesign. Just a better box.

## 2 – Upgrading RAM (from “bare minimum” to “comfortable”)

Both machines started with **4GB RAM**.

It technically worked, but:
- occasional lag
- less room for spikes
- no headroom for future changes

So I upgraded the NUC to **16GB RAM**.

Is it too much? Probably.  
But it removes an entire class of problems.

### Adjusting Minecraft memory

After the upgrade, I revisited the service configuration.
I came from something like this:
```
ExecStart=/usr/bin/java -Xmx4G -Xms2G -jar server.jar nogui
```
to something more complex:
```
WorkingDirectory=/home/mbrambilla/Minecraft
ExecStart=/usr/bin/java -Xms6G -Xmx8G \
-XX:+UseG1GC \
-XX:MaxGCPauseMillis=150 \
-XX:+ParallelRefProcEnabled \
-XX:+UnlockExperimentalVMOptions \
-XX:+DisableExplicitGC \
-jar server.jar nogui
Restart=on-failure
RestartSec=10
```
At first I tried allocating around 12 to 14GB of memory, leaving room for the OS, but that didn’t help. On the contrary, the server was slower than with the previous configuration using 2GB of RAM.

Instead of pushing limits, the goal was:
- give enough memory
- leave room for the OS
- avoid constant tuning

### Rule of thumb

- Don’t allocate all available RAM  
- Leave at least 2–4GB for the system  
- Prefer stability over squeezing performance  

## 3 – Remote power control (Wake-on-LAN)

One annoying limitation of the first setup:

The server was either always on… or physically managed.

That didn’t fit well with real usage:
- sometimes I don’t play for weeks
- no need to keep it running all the time

So I added **Wake-on-LAN (WoL)**.

### What this enables

- Turn on the server remotely
- Keep it powered off when not needed
- Reduce power consumption even further

### Basic idea

- Enable WoL in BIOS
- Enable it on the network interface
- Send a “magic packet” from another device

### 1. BIOS

- Enable **Wake on LAN from S4/S5 → Power On – Normal Boot**.
- Disable any **Deep Sleep / ErP / EuP** power-saving option.
  This one silently removes standby power from the NIC even when WoL otherwise
  looks enabled — the classic reason a correctly-configured machine won't wake.

### 2. Find the interface and connection name

```bash
nmcli c show
ip link show
```

On the NUC: interface `enp0s25`, connection `netplan-enp0s25`.

### 3. Enable WoL via NetworkManager

`sudo` is required.

```bash
sudo nmcli connection modify "netplan-enp0s25" 802-3-ethernet.wake-on-lan magic
sudo nmcli connection up "netplan-enp0s25"
```

NetworkManager reapplies this on every connection, so it persists across reboots.

### 4. Verify it landed on the NIC

```bash
sudo ethtool enp0s25 | grep -i wake
```

Expected:

```
Wake-on: g
```

Reading with `ethtool` is fine — only *setting* WoL via `ethtool` is the thing
that doesn't stick.

### 5. Note the MAC address

```bash
ip link show enp0s25
```

The `link/ether` value (e.g. `aa:bb:cc:dd:ee:ff`) is the target for the magic
packet. Make sure it's the **wired** NIC's MAC, not the Wi-Fi one.

### 6. Test — full shutdown, then wake

The WoL setting applies as the connection goes down, so test with a real
shutdown, **not** a reboot:

```bash
sudo shutdown now
```

Then from another machine on the LAN:

```bash
wakeonlan -i 192.168.0.255 <MAC>
```

Using the subnet-directed broadcast (`-i 192.168.0.255`) rather than the default
`255.255.255.255` is the most common fix when the machine won't wake.

---

### Troubleshooting

Check the NIC port LED while the machine is off:

- **Dark** → the NIC has no standby power in S5. Go back to the BIOS and disable
  Deep Sleep / ErP / EuP. Nothing on the software side can fix this.
- **Lit / blinking** → power is fine, so it's a delivery problem. Send the magic
  packet **from an always-on node on the same subnet** to remove all
  routing/VLAN questions:

  ```bash
  # on an always-on node on 192.168.2.x
  sudo apt install wakeonlan
  wakeonlan <MAC>
  ```

Confirm the packet actually reaches the segment — from a sibling host that's up:

```bash
ip -br link                       # find that host's interface
sudo tcpdump -i <iface> -c 2 'udp port 9'
```

Then send the packet again; if `tcpdump` sees it, delivery to that L2 segment
works.

Now the server can stay off, and I can wake it up from my laptop when needed.

## 4 – Updating Minecraft from anywhere in the network
Originally, updating Minecraft meant:
- SSH into the server
- manually download the new version
- restart everything

It worked, but it was tedious.
So I automated it.
### Goal
- Trigger updates from any machine in the network
- Reduce manual steps
- Keep control, no auto-updates

```bash
#!/bin/bash

# Set the variables
VERSION_MANIFEST_URL="https://launchermeta.mojang.com/mc/game/version_manifest.json"
REMOTE_SERVER="minecraft-server.local"
REMOTE_PATH="~/Minecraft/server.jar"
USER="admin" # Remote server username

# Get the latest release URL using jq
LATEST_RELEASE_URL=$(curl -s $VERSION_MANIFEST_URL | jq -r '.latest.release')
LATEST_VERSION_URL=$(curl -s $VERSION_MANIFEST_URL | jq -r --arg version "$LATEST_RELEASE_URL" '.versions[] | select(.id == $version) | .url')

# Get the server.jar URL using jq
SERVER_JAR_URL=$(curl -s $LATEST_VERSION_URL | jq -r '.downloads.server.url')

# Download the latest Minecraft server version
echo "Downloading the latest Minecraft server version..."
curl -o server.jar $SERVER_JAR_URL

# Check if the download succeeded
if [ $? -ne 0 ]; then
    echo "Error downloading the Minecraft server."
    exit 1
fi

# Copy server.jar to the remote server
echo "Uploading server.jar to the remote server..."
scp server.jar $USER@$REMOTE_SERVER:$REMOTE_PATH

# Check if the copy succeeded
if [ $? -ne 0 ]; then
    echo "Error copying server.jar to the remote server."
    exit 1
fi

# Remove the local server.jar file
rm server.jar

# Restart the remote machine
echo "Restarting the remote machine..."
ssh -tt $USER@$REMOTE_SERVER "sudo reboot"

echo "Done."
```

### Result
- One command → server updates
- No need to remember URLs
- No manual file handling

Still simple. Just less friction.

## 5 – What didn’t change
Just as important as what changed is what didn’t.
- No Docker
- No orchestration
- No monitoring stack
- No external dependencies
The system is still:
- understandable
- debuggable
- local-first

### Conclusion
These changes didn’t transform the project.
They didn’t make it “production-ready”.
They didn’t make it scalable.
They just made it **nicer to live with**.
- smaller hardware
- quieter setup
- more memory
- remote control
- less friction in updates

That’s the real goal.
A Second Shelf project doesn’t need to be perfect.
It just needs to improve, one small step at a time.
