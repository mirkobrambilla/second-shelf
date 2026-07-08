---
title: "Redundant Local DNS for a Homelab (with AdGuard Home)"
date: 2026-xx-xx
tags: [home, homelab, dns, adguard, kubernetes, networking]
version: 1.0
---

# Redundant Local DNS for a Homelab (with AdGuard Home)
*A Second Shelf project – every problem I hit, so you don't have to*

---

## Introduction

This one started as a small annoyance.

Every time I exposed a new service on my Kubernetes cluster, I had to go and hand-edit `/etc/hosts` on my Mac.

One line per service.
Every time.
On every machine that needed it.

It worked. It always works. That's the trap.

But it's busywork, and busywork is exactly the kind of thing that should disappear once and never come back.

So the goal became simple:

> Adding a new service should be a Kubernetes problem. Nothing else.

No DNS step. No hosts file. No remembering which IP went where.

---

## The goal

Every service on the cluster gets a `<name>.home.lab` hostname.

All of them resolve to the same place: the Traefik ingress at `192.168.0.225` (a MetalLB IP).

The whole thing comes down to one record:

```
*.home.lab → 192.168.0.225
```

A single wildcard. Add a service, give it an ingress, done. The name just works.

---

## The environment

- Subnet `192.168.0.0/24`, router is a **Google Nest Wifi Pro**
- k3s cluster lives in `192.168.0.200–.254`, **Traefik ingress = `192.168.0.225`**
- Admin machine is a **Mac**
- **NUC = `192.168.0.5`**, **QNAP NAS = `192.168.0.6`**, gateway `192.168.0.1`

---

## Problem 0 — The router can't do any of this

Before writing a single config, the first wall.

> **Problem:** The Nest Wifi Pro cannot create local DNS records or wildcards. At all.

It's not a DNS server. It's a DNS *proxy*. It can forward to a custom upstream, and that's it.

Two consequences fall out of this, and both shaped the whole project:

1. All the local-DNS logic has to live on a server I run. The Nest just points at it.
2. The moment you set a custom upstream DNS on the Nest, it **silently disables its DNS-rebinding protection.**

That second one matters. Rebind protection blocks private-IP answers — which is *exactly* what I'm about to hand back (`192.168.0.225`).

So the custom-DNS switch isn't optional. It's mandatory. Know this before you start, or you'll spend an evening wondering why your `.lab` names return nothing.

---

## The design decisions (why before how)

### Why AdGuard Home over Pi-hole

Wildcard DNS rewrites are a **native UI feature** in AdGuard.

That's the whole point of this project, so I wanted it to be first-class, not a workaround. Pi-hole *can* be coerced into wildcards, but it's not built for it.

AdGuard also ships as a single lightweight binary and has encrypted upstreams built in. Easy choice.

### Why two instances, and why NOT on the cluster

Redundancy. DNS going down takes the *whole network* with it, so one resolver isn't enough.

But the critical rule:

> The resolvers must NOT live on the k3s cluster.

If they did, a cluster reboot — or a Wake-on-LAN power-off — would take down network-wide DNS. The thing keeping the network alive can't depend on the thing I reboot for fun.

So:

- **Primary:** the always-on **NUC** (Ubuntu) at `192.168.0.5`
- **Secondary:** the **QNAP NAS** via Container Station at `192.168.0.6`

### Why no public DNS as a safety net

Tempting to set the Nest's secondary resolver to `1.1.1.1`. Just in case.

Don't.

If a client ever queried the public one, the `.lab` names would fail. You'd get **intermittent** resolution — works, then doesn't, then works again. The worst kind of bug.

Both Nest resolvers have to be my own AdGuard boxes. AdGuard itself forwards normal internet traffic upstream (`1.1.1.1` / `8.8.8.8`).

### The accepted tradeoff

These two boxes are now the network's resolvers.

If *both* are down, internet name resolution is down until I revert the Nest to Automatic.

One box is always-on. The other is a NAS that basically never sleeps. Good enough.

---

## Building the primary: NUC (Ubuntu)

> Ordering rule that matters: **free port 53 before AdGuard starts**, or it can't bind the DNS port.

### Static IP with nmcli

Don't assume the connection name or gateway. Read them off the live system first.

```bash
nmcli device status
nmcli connection show
ip route | grep default      # read the real gateway
```

Mine turned out to be `netplan-enp0s25`. Then:

```bash
sudo nmcli connection modify "netplan-enp0s25" \
  ipv4.method manual \
  ipv4.addresses 192.168.0.5/24 \
  ipv4.gateway 192.168.0.1 \
  ipv4.dns "1.1.1.1,8.8.8.8"

sudo nmcli connection up "netplan-enp0s25"
```

**Worth calling out:** the NUC's own DNS points at public resolvers — **not at itself.**

This avoids a chicken-and-egg: if the box resolved through its own AdGuard and AdGuard was down, the box couldn't resolve anything. Including the things needed to fix AdGuard.

> **Gotcha (netplan):** the `netplan-` prefix means netplan generated this profile. My nmcli edits stuck — but if a future `netplan apply` ever reverts the IP, the static config also has to be mirrored in `/etc/netplan/*.yaml`.

Verify:

```bash
ip -4 addr show enp0s25      # expect 192.168.0.5/24
ip route | grep default
```

---

## Problem 1 — Ubuntu is already sitting on port 53

> **Problem:** `systemd-resolved` squats on `127.0.0.53:53`. AdGuard can't bind 53 until it lets go.

Disable the stub listener and repoint `resolv.conf`:

```bash
sudo sed -r -i.orig 's/#?DNSStubListener=yes/DNSStubListener=no/' /etc/systemd/resolved.conf
sudo mv /etc/resolv.conf /etc/resolv.conf.backup
sudo ln -s /run/systemd/resolve/resolv.conf /etc/resolv.conf
sudo systemctl reload-or-restart systemd-resolved
```

The symlink swap is the part people forget.

It repoints `/etc/resolv.conf` away from the stub (`127.0.0.53`) to the real upstream list — so the host can still resolve names after you take the stub away. Skip it and the box goes deaf.

Confirm 53 is actually free:

```bash
sudo ss -tulpn | grep ':53'   # should return nothing
```

---

### Install AdGuard Home

```bash
curl -s -S -L https://raw.githubusercontent.com/AdguardTeam/AdGuardHome/master/scripts/install.sh | sh -s -- -v
```

It installs as a service named `AdGuardHome`:

```bash
sudo systemctl status AdGuardHome
sudo /opt/AdGuardHome/AdGuardHome -s restart
```

### The setup wizard (`http://192.168.0.5:3000`)

- **Admin Web Interface:** all interfaces, port **3000**
- **DNS server:** all interfaces, port **53**
- Set admin credentials

> I picked **3000** for the admin UI deliberately, so both instances share the same UI port. Easier muscle memory now, and easier AdGuardHome-Sync later.

### The wildcard rewrite + upstreams

- **Filters → DNS rewrites:** domain `*.home.lab`, answer `192.168.0.225`
- **Settings → DNS settings → Upstream DNS servers:** `1.1.1.1`, `8.8.8.8`

### Verify

```bash
dig grafana.home.lab @192.168.0.5 +short   # expect 192.168.0.225
```

One resolver down. One to go.

---

## A decision point: do I have to make every change twice?

Yes.

Two independent instances means every new wildcard, every upstream, every filter change has to be entered on **both**.

And the drift genuinely matters here. The Nest hands out both boxes as resolvers, so a client might query either one. If only the NUC has a new rewrite, that hostname resolves intermittently.

The exact failure mode I avoided by not using a public secondary — sneaking back in through the side door.

> **The fix (still on the to-do list): AdGuardHome-Sync.**
> NUC = origin, QNAP = replica. It reads the origin's config over the API and pushes it to the replica on a schedule.
> Caveat: it's a *one-way overwrite*. Once it's running, you only ever edit the origin. Edits on the replica get clobbered.

For now, double-entry it is.

---

## Building the secondary: QNAP NAS — where the problems lived

The plan: **host networking** as primary, with a bridge-mode compose kept as a fallback. Persistent volumes for the work and conf directories.

Host mode compose, pasted into Container Station → Create Application:

```yaml
services:
  adguardhome:
    image: adguard/adguardhome:latest
    container_name: adguardhome-secondary
    restart: unless-stopped
    network_mode: host
    volumes:
      - /share/Container/adguardhome/work:/opt/adguardhome/work
      - /share/Container/adguardhome/conf:/opt/adguardhome/conf
```

> Host mode is preferred for a DNS server because it sees real client IPs. Bridge mode would make every query look like it came from the Docker gateway.

Then the problems started arriving in order.

---

### Problem 2 — There's no terminal on the NAS

The QTS web UI has no built-in terminal. And I needed a shell to figure out the port-53 situation.

**The fix:** enable SSH from the GUI, then use the *Mac's* terminal.

- QTS: **Control Panel → Network & File Services → Telnet / SSH** → tick *Allow SSH connection* → Apply
- Then from the Mac: `ssh admin@192.168.0.6`

> **Important non-fix:** switching to the bridge compose does *not* dodge this. Bridge mode still binds host port 53, so whatever's on 53 has to be dealt with either way. The terminal isn't optional.

(GUI-only escape hatches if SSH were off-limits: check Container Station for a leftover container holding 53, the **DNS Server** QPKG in App Center, or a DHCP/DNS service under Network & Virtual Switch.)

---

### Problem 3 — "Port 53 already in use" — but used by *what?*

```bash
netstat -tulpn | grep ':53'
```

Two things happened.

First: `netstat: can't scan /proc - are you root?` — the PID column came back blank because I wasn't root. Rerun with `sudo` to see process names.

Second, and more useful, the binding pattern:

```
127.0.1.1:53      ← loopback
10.0.3.1:53       ← Container Station virtual-network gateway
10.0.5.1:53       ←   "
10.0.7.1:53       ←   "
::1:53            ← IPv6 loopback
```

**Diagnosis:** loopback plus a set of `10.0.x.1` gateways is **Container Station's own dnsmasq** — the resolver QNAP runs so containers can resolve each other. The `10.0.x.x` range is QNAP's Docker bridges; each `.1` is a gateway.

> **Do not kill it.** That breaks inter-container DNS.

**Noise to ignore:** `grep ':53'` also matched `0.0.0.0:5353` (mDNS/Bonjour) and a few `53xxx` lines (`53020`, `53549`, `53666`, `53682`). Those are random ephemeral *source* ports. None of them touch port 53.

**The key insight:** 53 was bound only on loopback and the virtual gateways — **not** on the NAS's LAN IP (`192.168.0.6`), and **not** on `0.0.0.0`.

So AdGuard can have 53 on the LAN interface with zero conflict. As long as it binds to *that specific IP* instead of "all interfaces."

---

### Problem 4 — The wizard refuses to bind 53

```
listen tcp 0.0.0.0:53: bind: address already in use
```

This is Problem 3 surfacing in the wizard. AdGuard defaults its DNS listener to "All interfaces" (`0.0.0.0:53`), which can't coexist with dnsmasq's specific binds.

> **Do not ignore and continue.** Pushing past this means AdGuard never actually grabs 53 and serves no DNS at all.

**The fix:** on the wizard's DNS server step, change *Listen interfaces* from "All interfaces" to the NAS's specific LAN IP: **`192.168.0.6`**.

In host mode the dropdown lists the host's real interfaces. The port check then validates against `192.168.0.6:53` — which is free.

> **Clarification that saved me a second mistake:** this applies only to the *listen* interface. The **Upstream DNS servers** stay `1.1.1.1` / `8.8.8.8`. Pointing the upstreams at `.170` would make AdGuard query itself in a loop. The Admin Web Interface can stay on "All interfaces" — the conflict was only ever on 53.

*(Bridge-mode equivalent, if host mode had failed: scope the published port to the IP — `"192.168.0.6:53:53/udp"` and `"192.168.0.6:53:53/tcp"` instead of plain `53:53` — and let the wizard keep "All interfaces", since that's the container's own namespace.)*

---

### Problem 5 — The UI came up on port 80, and you can't change it in the UI

After setup, the admin UI was on port 80 — the wizard's post-install default — not the 3000 I used on the NUC.

> **Problem:** the admin port is **not editable from inside the web UI.** It lives only in the config file.

With the volume mount, that file is at `/share/Container/adguardhome/conf/AdGuardHome.yaml`.

Stop the container first — so AdGuard doesn't rewrite the file on shutdown — then edit:

```yaml
http:
  address: 0.0.0.0:3000
```

> Older AdGuard format uses `bind_host` / `bind_port` instead. In that case change `bind_port: 80` to `bind_port: 3000`.

Start the container, then browse to `http://192.168.0.6:3000`.

`0.0.0.0:3000` is fine here — the conflict was only ever on 53, so the UI doesn't need to be scoped to `.170`.

Then add the **same** wildcard rewrite (`*.home.lab → 192.168.0.225`) and the **same** upstreams as the NUC.

---

## Final verification and cutover

```bash
dig grafana.home.lab @192.168.0.6 +short   # expect 192.168.0.225
```

Once *both* resolvers answer correctly:

- Set the Nest Wifi custom DNS — primary `192.168.0.5` (NUC), secondary `192.168.0.6` (NAS)
- **Only now** remove the temporary `/etc/hosts` line on the Mac

> Never cut over while down to a single resolver. Get both green first, then pull the safety net.

---

## Reflection

This wasn't an elegant build.

It was a string of small walls, hit one after another: a router that can't do DNS, an OS sitting on the port, a NAS with no terminal, a wizard that won't bind, a UI that hides its own setting.

None of them were hard once I understood them.
All of them would have cost an evening if I'd guessed instead of looked.

That's the actual lesson here, and it's a Second Shelf one:

> Most of the work in a homelab isn't building. It's figuring out what's already in the way.

I didn't reach for a fancier tool when port 53 was busy. I ran `netstat`, read the pattern, and realized the answer was *bind to one specific IP*. The constraint did most of the design for me.

Two boxes instead of one. Double-entry until I get around to the sync tool. Not perfect.

But the network resolves its own names now. Add a service, give it an ingress, and the hostname just exists.

No more `/etc/hosts`.

That was the whole point.

---

## Still on the to-do list

- **AdGuardHome-Sync** to keep the two instances identical, so future wildcards only have to be added once — on the NUC.

> [TODO: write this up once it's running — origin/replica config, the schedule, and what broke the first time]

---

## The problems, condensed (the highlight reel)

- The Nest can't do local DNS, and turning on custom DNS silently disables rebind protection. Know it before you start.
- Ubuntu: `systemd-resolved` holds port 53. Disable the stub listener *and* fix the `resolv.conf` symlink, or the host loses resolution.
- The NUC's own resolver should be public DNS, not itself (chicken-and-egg).
- QNAP has no web terminal — enable SSH in the GUI and use your Mac.
- "Port 53 in use" on QNAP is Container Station's dnsmasq on the bridge gateways. Don't kill it — bind AdGuard to the LAN IP instead.
- The wizard's `0.0.0.0:53 already in use` is fixed by choosing the specific LAN IP, not by ignoring it.
- Don't confuse the DNS *listen* interface with the *upstream* servers.
- The AdGuard admin web port can only be changed in `AdGuardHome.yaml`, not the UI.
- Two independent instances drift. Every change is double-entry until AdGuardHome-Sync is in place.
