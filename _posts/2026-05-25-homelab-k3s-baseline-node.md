---
title: "Building a Home Kubernetes Cluster (Part 1: Getting One Node Ready)"
date: 2026-05-25
tags: [home, homelab, kubernetes, k3s, networking]
version: 1.0
excerpt: ""
---

## Introduction

The goal was never "run a couple of containers."

It was bigger than that. A real, self-hosted platform I actually own — a place to learn how clusters work, not just how to click "deploy." And, further out, somewhere I could eventually point at a local AI experiment or two.

None of that happens in one step.

Before there’s a cluster, there has to be a node. One node, prepared the same way every other node will be, sitting there ready — but not yet running anything.

This is the story of getting there.

## What I wanted

Bare metal. At home.

Not because cloud is bad, but because I wanted to feel the physical layer — nodes, power, cables, DNS — instead of abstracting it all away behind an API.

My home internet isn’t great either, which quietly settled a few arguments before they started. Nothing that depends on being reachable from the internet was ever really on the table. The cluster had to be useful *inside* the LAN first.

And it had to grow. Not to ten nodes on day one — but to ten nodes eventually, without a redesign every time a new machine showed up.

Start cheap. Start small. Keep it clean enough to scale.

## Choosing the hardware

### Two broken laptops

The first attempt used two old laptops with broken screens — useless as laptops, but headless compute doesn’t need a screen. Free hardware. Why not.

In practice, they were awkward.

Different shapes, different quirks, nothing that stacked or cabled cleanly. Batteries and lids get in the way of an always-on headless box. And two machines isn’t really a cluster — it’s two machines. I couldn’t walk into a shop and buy "three more of the same broken laptop."

They proved the idea worked. They weren’t something to build on.

### Small, uniform, repeatable

The actual answer was NUCs. Small, low-power, stackable, and — this is the part that matters — you can buy several identical ones.

I didn’t buy them new. I went looking for **used** NUCs instead, and found them for around **40–50 CHF each**.

*The point was never to buy the best hardware. It was to find the cheapest hardware I could still repeat.*

Five used NUCs at 45 CHF is a cluster. Five new ones is a decision I’d still be putting off.

## Full Kubernetes, or k3s

With the hardware settled, the next fork: run the real thing — full Kubernetes, `kubeadm` — or run k3s, its lightweight cousin.

Full Kubernetes is the "correct" answer. It’s what you’d actually run in production, and it comes with total control over every component.

It’s also heavier than a 45 CHF NUC wants to carry, and it comes with more pieces that can quietly drift out of health while you’re not looking — a separate etcd, more daemons, more 2am surprises.

k3s is a stripped-down, single-binary Kubernetes distribution. Same APIs, same `kubectl`, same manifests and controllers — just packaged with sensible defaults instead of a hundred decisions to make up front.

k3s won. Not because full Kubernetes is wrong, but because I wanted to spend my attention on the parts I actually wanted to learn — not on hand-assembling a control plane just to prove I could.

## Designing the cluster, before touching a single node

Three control-plane nodes. Two worker nodes, for now.

Three control-plane nodes isn’t an arbitrary number — it’s the smallest count that can lose one node and keep going. Two is a single point of failure with extra steps.

Every node gets a static, reserved address, with room left on purpose:

| Purpose | Address(es) |
|---|---|
| Control-plane virtual IP (VIP) | `192.168.2.200` |
| Control-plane nodes | `192.168.2.201`–`.203` |
| Spare control-plane (reserved) | `192.168.2.204`–`.205` |
| Worker nodes | `192.168.2.206`–`.207` |

That gap at `.204`–`.205` costs nothing today. It means a fourth or fifth master can join later without renumbering anything that already exists.

Ingress, DNS, observability, the HA layer itself — all of that is decided, none of it is built yet. For a single node, none of it matters. A node only needs to know one thing: its own address, and that it can be woken and managed without anyone touching it.

## Getting one node ready

This is the part that has to be boring, on purpose.

I didn’t want to do this by hand on every NUC and trust myself to remember the steps the same way twice. So I wrote a script — `baseline-node.sh` — that runs the entire baseline install in one go. Every new node gets the same script, the same run, the same result.

```
sudo ./baseline-node.sh --hostname kube-cp-01   --ip 192.168.2.201 --role server
sudo ./baseline-node.sh --hostname kube-node-01 --ip 192.168.2.206 --role agent
```

Point it at a hostname and an IP, and it works through the same six things, every time.

A small, identical set of packages — nothing exotic, just what I’ll want the first time I SSH in at 11pm to figure out why something’s broken: `openssh-server` to get in at all, `htop` and `vim` for being there once I am, `ethtool` and `wakeonlan` for the networking problems I already knew were coming, and `chrony` for time sync — because etcd and TLS both quietly fall apart the moment two nodes disagree about what time it is.

SSH, enabled from first boot. A hostname, and a shared `/etc/hosts` map so every node — and the control-plane VIP, even though it isn’t a real machine — can find every other node by name, with no external DNS involved.

A static IP. And, in the same place, Wake-on-LAN.

That last pairing isn’t an accident.

> On these NUCs, setting Wake-on-LAN with `ethtool` alone doesn’t survive a reboot. It just quietly resets, and you don’t find out until the one time you actually need to wake the thing. Setting it on the NetworkManager connection instead makes it stick — NetworkManager re-arms it every time the connection comes up.
>
> ```
> nmcli con mod <connection> 802-3-ethernet.wake-on-lan magic
> ```
>
> The BIOS still gets a vote. "Deep Sleep" power-saving on these NUCs can stop a wake from a full shutdown no matter what the OS says — works in the demo, fails a week later, every time.

Boot to a CLI, not a desktop — `multi-user.target` by default, with a `gui` command to bring the desktop up when a node genuinely needs hands-on attention, and `gui-off` to put it back to sleep. Neither one changes what happens on the next boot. The node always comes back up headless.

And a small kernel baseline, so k3s lands on a node that’s already correct instead of one it has to fix on the way in: swap off, `overlay` and `br_netfilter` loaded, the sysctls that let bridged pod traffic actually get forwarded and seen.

At the end, the script prints the node’s MAC address. That gets written down and handed to the tool on my Mac that wakes and shuts down the cluster — the one manual step left in an otherwise scripted process.

## Reflection

None of this is a cluster yet. That’s the point.

I used to think "properly" meant handling every case up front — full Kubernetes, every option considered, nothing deferred. What I actually needed was the opposite: decide the shape once, on paper, and then make the *repeatable* part boring enough to script.

The broken laptops taught me that free isn’t the same as usable. The WoL setting taught me that "it works" and "it survives a reboot" are two different claims, and only one of them matters at 11pm on a Tuesday. And k3s taught me that the "real" tool isn’t always the right one — sometimes it’s just the heavier one.

*The philosophy was never "buy the best." It was "find the cheapest baseline I can repeat, and repeat it."*

## Conclusion

What exists right now is one NUC, on its correct address, reachable over SSH, wakeable from across the LAN, booting straight to a console, and carrying exactly the kernel baseline k3s expects.

Nothing is running on it. That’s next.

But it’s not a one-off setup either — it’s a script away from four more identical nodes, sitting on the same shelf, waiting for the same command.

That’s as far as this part goes: one node, ready, and a fleet that’s now just a matter of repeating it.
