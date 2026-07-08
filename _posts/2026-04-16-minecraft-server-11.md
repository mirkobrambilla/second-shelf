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

> [TODO: insert your story about discovering NUCs and second-hand market]

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

After the upgrade, I revisited the service configuration:
> [TODO: insert the research about RAM consumption}
```
ExecStart=/usr/bin/java -Xmx4G -Xms2G -jar server.jar nogui
```
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

> The server was either always on… or physically managed.

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

> [TODO: add the wakeOnLan instruction/scripts with the public repo if needed]

Example:

```bash
wakeonlan <MAC_ADDRESS>
```
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

> [TODO: insert your update script here]

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
