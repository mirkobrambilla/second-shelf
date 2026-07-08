---
title: "Running a Minecraft Server at home"
date: 2026-03-24
tags: [home, minecraft, server, reused hardware, Java]
version: 1.0
---

# Running a Minecraft Server at Home  
*A Second Shelf project – step-by-step narrative*

---
## Introduction

I had been playing Minecraft a few times with friends and family, usually on public servers somewhere around the world. We would build things together, and after a good session, we’d just hope to find everything still there the next time we logged in.

It would have been great to have our own world—our own place to create and destroy things together—and then just leave it there, waiting for the next session.

So I started looking into private servers. The cost isn’t that high (around 4–5 euros per month), but it’s still a recurring cost. And sometimes I don’t play for months, so it felt like a waste.

Then I looked at my shelf and saw an old PC sitting there, a spare machine with nothing to do. I thought: *why not turn this into a Minecraft server?* Something small, fun, and just for us. Nothing enterprise-grade, no cloud, no containers—just us, a machine, and a game.

This is the story of how I set it up, step by step, and what I learned along the way.

---

## Step 0 – The starting point

The machine was humble:
- Old PC
- On my home network
- Headless (or willing to go headless)
- Enough RAM to run Minecraft (~4GB free)

I wasn’t chasing perfection. I wanted **something that works, and I understand completely**.


## Step 1 – Installing Ubuntu

I chose **Ubuntu** because it’s reliable, stable, and well documented.  
I wanted a system that just works, without extra fluff.

You don’t need a special admin user. Just make sure there’s a `minecraft` user with `sudo` access so you can follow along.

I decided not to use **Ubuntu Desktop** because it adds unnecessary overhead and takes time to configure. I only needed a minimal setup, with full control over each component of that old PC.


## Step 2 – Disabling the GUI

Minecraft doesn’t need a graphical interface, neither I do. Removing it had immediate benefits:
- Faster boot
- Less RAM usage
- Fewer background services

```bash
sudo systemctl set-default multi-user.target
sudo reboot
```
Now the machine boots straight into the terminal. Simple. Clean. Headless.


## Step 3 – Remote access via SSH

Once the GUI was gone, I needed a way to control the server. Enter OpenSSH.
```bash
sudo apt update
sudo apt install openssh-server
sudo systemctl enable ssh
sudo systemctl start ssh
```

now, make sure you named the old PC as minecraft server and made it visible in your local network.
You can do it with this command:
```bash
sudo hostnamectl set-hostname minecraft-server
```
And, as I mentioned before, make sure there’s a `minecraft` user with `sudo` access so you can follow along. 

```bash
sudo adduser minecraft
sudo usermod -aG sudo minecraft
```

From my laptop I can now connect with:

```bash
ssh minecraft@minecraft-server
```
I could now manage the server from anywhere on my LAN.
No monitor. No keyboard. No mouse. Just me, my laptop, and a terminal.
 

## Step 4 - Basic system setup

Before installing Minecraft, a little hygiene:
```bash
sudo apt update && sudo apt upgrade
```
You are now working with a clean environment

## Step 5 - Installing Java

Minecraft only needs Java, nothing else.
```bash
sudo apt install openjdk-21-jre-headless
java -version
```
No IDEs, no build tools, just a runtime to launch the server.
Make sure you use an LTS version so you don’t have to change it often.

## Step 6 - Preparing the server

I created a dedicated folder:
```bash
mkdir -p ~/minecraft-server
cd ~/minecraft-server
```
Download the last version of the **official Minecraf server JAR** from Mojang:
```bash
# create a folder for the server
mkdir -p ~/minecraft-server && cd ~/minecraft-server

# get the latest server download URL from Mojang
LATEST_URL=$(curl -s https://launchermeta.mojang.com/mc/game/version_manifest.json \
  | grep -A 10 '"latest"' \
  | grep '"release"' \
  | cut -d '"' -f4 \
  | xargs -I {} curl -s https://launchermeta.mojang.com/mc/game/version_manifest.json \
  | grep -A 20 "\"id\": \"{}\"" \
  | grep '"url"' \
  | head -n 1 \
  | cut -d '"' -f4 \
  | xargs curl -s \
  | grep '"server"' \
  | grep '"url"' \
  | cut -d '"' -f4)

# download the server jar
wget -O server.jar "$LATEST_URL"
```

Now, you are ready to run the server for the first time
```bash
java -Xmx2G -Xms2G -jar server.jar nogui
```
and ups, server stopped!
The server stopped and complained about the EULA. I opened eula.txt:
```bash
vim eula.txt
```
and set:
```bash
eula=true
```

Now the server could start for real.

## Step 7 – Running Minecraft as a service

Running Minecraft in a terminal is fragile. I wanted it **reliable**, starting automatically on boot.

I created a `systemd` service
```bash
sudo vim /etc/systemd/system/minecraft.service
```
With this information:
```
[Unit]
Description=Minecraft Server
After=network.target

[Service]
User=minecraft
WorkingDirectory=/home/minecraft/minecraft-server
ExecStart=/usr/bin/java -Xmx2G -Xms2G -jar server.jar nogui
Restart=on-failure

[Install]
WantedBy=multi-user.target
```
Reload `systemd` and start the new service:
```bash
sudo systemctl daemon-reload
sudo systemctl start minecraft
sudo systemctl enable minecraft
systemctl status minecraft
```
Now Minecraft:
- starts automatically on boot
- restarts if it crashes
- runs in the background

## Step 8 - Testing the server
I joined from my LAN client, created a world, and restarted the server. Everything persisted.
It was alive. It was mine. And it worked.

## Step 9 - Enabling the whitelist
Since this server is private, I wanted control over who can join.
I then activated the whitelist feature on the minecraft server.

Edit `server.properties`:
```
white-list=true
```
Then I created a new file `whitelist.json` and filled the minecraft username of my family and friends
```json
[
  {
    "uuid": "UUID of the user",
    "name": "User1"
  },
  {
    "uuid": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
    "name": "user2"
  }
]
```
Now, only approved players can access the server.

## Step 10 - Networking

If you followed the configuration correctly, the server should already work in your LAN.
There is the potion to expose your minecraft server over internet, opening the port forwarding on port number 25565. I will not cover this in this article.

## Reflection - Why I’m not using Docker

One thing I didn’t use in this setup is Docker.

Not because it’s a bad idea — actually, it would bring some real benefits:
- easier migrations between machines  
- cleaner isolation  
- simpler upgrades and rollbacks  

But in this case, I chose not to use it.

This server runs on a machine that is **100% dedicated to Minecraft**.  
There are no other services, no competing workloads, nothing else to isolate.

Adding Docker would have meant introducing:
- another layer of abstraction  
- more moving parts  
- more things to debug  

For very little practical gain in this specific setup.

Yes, using containers would make hardware changes easier and reduce downtime.  
But this is not a production system.

This is a *Second Shelf* project.

If the hardware fails and the server is down for a day or two, that’s acceptable.  
There are no users paying for uptime. No SLAs. No pressure to recover instantly.

In this context, simplicity wins.

Less abstraction, fewer components, and a setup I can fully understand and fix quickly.

## Conclusion

At the end of the day, this is just a Minecraft server.

It’s not highly available.  
It’s not monitored.  
It’s not built to scale.  

But it works.

It runs on a machine that was doing nothing, in a corner of the house. It hosts a world that belongs only to us, where things stay exactly as we left them, waiting for the next session.

There’s something satisfying about that. Not just playing the game, but understanding the system behind it. Knowing where the data lives, how the service starts, how to fix it when it breaks.

And honestly, that’s more than enough.