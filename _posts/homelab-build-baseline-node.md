# Building a Bare-Metal k3s Home Lab — Brief (Part 1)
### From the first idea to a node that's ready for k3s

> Scope of this brief: the *thinking* (why a home lab, hardware choices, distro choice,
> cluster design) and the *single-node preparation* — i.e. everything up to the point
> where a node is baselined and ready, but k3s is **not yet installed**. The actual
> k3s bootstrap, HA, networking add-ons, observability, etc. belong to later parts.

---

## 1. Why a home lab at all

The goal was never "run a couple of containers." It was to have a **real, self-hosted
platform I fully own** — a place to learn cluster orchestration properly, run my own
services, and leave the door open for a longer-term ambition: a local AI experimentation
environment (distributed inference, model serving, self-hosted LLMs) down the line.

A few constraints shaped everything from day one:

- **Bare metal, at home.** No cloud bill, no someone-else's-computer. I wanted to feel
  the physical layer — nodes, power, cabling, DNS — not abstract it away.
- **Poor home internet.** This quietly ruled out anything that depended on heavy public
  exposure. The cluster is designed to live and be useful *inside* the LAN first.
- **Room to grow.** A rack with space to scale toward ~10 worker nodes eventually, so
  the design had to be incremental — start tiny, add nodes without re-architecting.

The mindset throughout: **start cheap, start small, keep it clean enough to scale.**

---

## 2. Choosing the hardware

### 2.1 The false start: two broken laptops

The very first attempt used **two old laptops with broken monitors** — machines that were
useless as normal laptops but, in theory, perfectly fine as headless compute. Free
hardware, why not.

In practice they were awkward as cluster nodes:

- Inconsistent form factor — nothing rack- or shelf-friendly, awkward to stack or cable.
- Batteries and lid/monitor quirks that get in the way of a clean always-on, headless setup.
- Two machines is barely a cluster, and they weren't a basis I could *repeat* — I couldn't
  go buy "five more of the same broken laptop."

They proved the concept but weren't something to build on. Lesson: for a cluster, **uniform,
small, repeatable hardware** matters more than "free."

### 2.2 The pivot: Intel NUCs

The natural answer was **NUCs** — small, low-power, uniform, and genuinely rack/shelf
friendly. They stack neatly, sip power, and you can buy several identical units, which is
exactly what a cluster wants.

### 2.3 Used, not new — the 40–50 CHF sweet spot

I didn't want to spend a fortune just to *start*. So instead of buying new NUCs, I went
hunting for **used** ones — and found units for around **40–50 CHF each**. For a home lab
that's an excellent trade-off: cheap enough to buy a few without flinching, capable enough
to run a real cluster, and uniform enough to script against.

> Worth calling out in the article: the philosophy isn't "buy the best," it's "buy the
> *repeatable enough, cheap enough* baseline and scale it." Used NUCs at ~45 CHF hit that
> exactly.

---

## 3. Full Kubernetes vs. k3s

With hardware sorted, the next big fork: **vanilla/full Kubernetes (kubeadm) vs. k3s.**

**The case for full k8s (kubeadm):**
- It's "the real thing" — closest to what you'd run/see in production environments.
- Maximum flexibility and component-level control.

**Why that's overkill here:**
- Heavier resource footprint — a poor fit for cheap, modest used NUCs.
- More moving parts to install, configure, and *keep* healthy (separate etcd, more
  components, more that can drift or break).
- More operational overhead than a home lab needs to *start* learning the concepts.

**Why k3s won:**
- It's a **simplified, lightweight Kubernetes distribution** — same core APIs and concepts,
  packaged into a single small binary with sensible defaults.
- Dramatically lighter on resources, which suits the NUCs.
- Much faster to stand up and far less to babysit, without giving up the actual Kubernetes
  learning surface — `kubectl`, manifests, controllers, ingress, etc. all behave as expected.
- Batteries-included defaults can be swapped where I want my own opinions later (this matters
  for the design choices in §4).

The decision: **k3s over kubeadm.** Keep the cognitive and resource budget for the parts I
actually want to learn and customize, not for hand-assembling a control plane.

---

## 4. Designing the cluster (high-level)

Even though this brief stops before k3s is installed, the **node and network design** is part
of the up-front thinking and is worth documenting here, because it dictates how each node is
baselined.

### 4.1 Node roles and count

A small but genuinely **HA-shaped** topology, with room to grow:

- **3 control-plane nodes** (`kube-cp-01`–`03`) — three so the control plane can tolerate a
  node loss rather than being a single point of failure.
- **2 worker nodes** (`kube-node-01`–`02`) — where the actual workloads land, scalable toward
  ~10 over time.

### 4.2 IP scheme

Static, predictable, and reserved with headroom (all on the `192.168.2.0/24` LAN):

| Purpose                         | Address(es)                |
|---------------------------------|----------------------------|
| Control-plane virtual IP (VIP)  | `192.168.2.200`            |
| Control-plane nodes             | `192.168.2.201` – `.203`   |
| Spare control-plane (reserved)  | `192.168.2.204` – `.205`   |
| Worker nodes                    | `192.168.2.206` – `.207`   |

Reserving `.204–.205` up front means a 4th/5th master can join later without renumbering.

> Design note for later parts (not part of node prep): only **node IPs** and the
> LoadBalancer pool consume real LAN addresses; pod/service networks live on internal
> overlays. So the address budget above is the real-world footprint of the cluster.

### 4.3 What's deliberately deferred

The control-plane VIP / HA layer, the LoadBalancer pool, ingress, DNS, and observability are
all design decisions already made — but they're **post-install** concerns. For node prep, the
only thing each node needs to know is its **own static IP** and that it can be woken and
managed headlessly.

---

## 5. Preparing a single node (up to "ready for k3s")

This is the repeatable part: take a used NUC, and turn it into a clean, predictable,
headless, k3s-ready node. The work is captured in a **`baseline-node.sh`** script so every
node is prepared *identically* — uniform hardware deserves uniform setup.

### 5.1 Base OS

A **light Ubuntu LTS desktop flavour** (Xubuntu / Lubuntu). That choice is deliberate: these
flavours ship with **NetworkManager** (which the baseline relies on) and keep a graphical
session available for the rare time a node needs hands-on attention — but the node is
configured to **boot straight to the CLI** (`multi-user.target`). The desktop is never running
in normal operation; it's brought up on demand with a small `gui` helper and dropped again with
`gui-off`. In practice the nodes behave as appliances: they boot to a text console, sit on the
shelf, and are driven entirely over SSH from the admin Mac. The same login user is used on
every node so the admin tooling can assume it.

### 5.2 What `baseline-node.sh` does

The script is parameterised — you pass it the node's identity and it does the rest, e.g.:

```
sudo ./baseline-node.sh --hostname kube-cp-01   --ip 192.168.2.201 --role server
sudo ./baseline-node.sh --hostname kube-node-01 --ip 192.168.2.206 --role agent
```

Gateway, CIDR, DNS and the network interface all have sensible defaults (the interface is
auto-detected from the default route) and can be overridden with flags. From there it works
through six concerns, in order:

**1. Packages.** A deliberately small, identical set on every node — k3s pulls its own runtime,
so there's nothing exotic here, just the things you'll want the first time you SSH in to debug:

| Package(s) | Why it's on every node |
|---|---|
| `openssh-server` | Remote management — the only way in once it's on the shelf |
| `htop`, `vim` | Day-to-day headless admin |
| `ethtool`, `wakeonlan` | Inspect the NIC's WoL capability / send-and-test wake packets |
| `net-tools` | Basic network diagnostics |
| `curl`, `wget`, `ca-certificates`, `gnupg` | Fetch and verify the k3s installer |
| `chrony` | **Time sync.** etcd and TLS both need the nodes' clocks in agreement — skew quietly breaks a cluster, so NTP is part of the baseline, not an afterthought |

> The point isn't the exact list — it's that **every node gets the *same* list**, scripted, so
> there are no "works on node-01 but not node-02" surprises.

**2. SSH.** Enable and start `sshd` so the node is reachable headless from the first boot.
(After copying a key over, password auth can be turned off — noted as a follow-up.)

**3. Hostname + a cluster-wide `/etc/hosts` map.** Set the node's hostname, then write a shared
host map so **every node can resolve every other node — and the control-plane VIP — by name**,
independent of any external DNS. The VIP (`192.168.2.200`, `kube-api` / `kube-vip`) is included
here even though it isn't a physical machine. The block is fenced with markers so re-running the
script updates it cleanly rather than appending duplicates.

**4. Static IP + Wake-on-LAN (NetworkManager preferred, netplan fallback).** Assign the node's
reserved address from §4.2 and arm WoL in the *same* place. If NetworkManager is running it's
configured with `nmcli`; if not, the script falls back to writing a `netplan`/networkd config
(with `wakeonlan: true`). The principle is the same either way: **one network configuration
layer owns both the IP and the wake behaviour**, with no standalone daemon to drift.

> ⚠️ **The WoL lesson (great article anecdote):** On Intel NUCs, setting Wake-on-LAN with
> `ethtool` alone **does not survive a reboot** — the setting silently resets. Setting it on the
> NetworkManager connection instead makes it persist, because NM re-arms it every time the
> connection activates:
> ```
> nmcli con mod <connection> 802-3-ethernet.wake-on-lan magic
> ```
> Also watch the **BIOS "Deep Sleep" / deep power-saving settings** — on NUCs these can stop a
> machine waking from a *full* shutdown regardless of OS config, and the WoL/PCIe wake option
> has to be enabled in BIOS in the first place. Classic "works in the demo, fails next week."

**5. Boot to CLI, desktop on demand.** Set the default boot target to `multi-user.target` (no
desktop at boot) and drop two tiny helpers in place: `gui` to bring the desktop up for
maintenance and `gui-off` to return to the console. Neither changes the boot default — the node
always comes back up headless.

**6. Light kernel baseline for k3s.** The OS-level prep so k3s lands on a node that's already
correct rather than one that needs fixing mid-install. k3s is lighter here than full kubeadm —
it bundles much of its own networking — but a small, predictable baseline still pays off:

- **Swap** disabled (`swapoff -a`) and commented out in `/etc/fstab`, for predictable
  scheduling and memory behaviour.
- **Kernel modules** loaded and persisted via `/etc/modules-load.d/`: `overlay`, `br_netfilter`.
- **Sysctls** set and persisted via `/etc/sysctl.d/`: `net.ipv4.ip_forward=1`,
  `net.bridge.bridge-nf-call-iptables=1`, `net.bridge.bridge-nf-call-ip6tables=1`, so bridged
  pod traffic is forwarded and seen by iptables.

Finally the script prints a summary — including the node's **MAC address**, which you record and
hand to the cluster's wake/shutdown tool (`cluster-ctl.sh`) on the admin Mac.

### 5.3 End state

After `baseline-node.sh` runs on a NUC, the node is:

- On its **correct static IP**, reachable over SSH from the admin Mac.
- **Wakeable** over the network, with WoL config that *persists* across reboots.
- Booting cleanly to a headless CLI (with the desktop one `gui` command away if needed).
- Resolving every other node and the VIP by name via the shared `/etc/hosts` map.
- Clock-synced and carrying the kernel baseline k3s wants.

Two things are still done by hand, because they can't be scripted from inside the OS: enabling
**Wake-on-LAN / PCIe wake in the node's BIOS/UEFI**, and a **reboot to confirm** the node still
comes up CLI-only and stays wakeable. After that:

In other words: **a node that's ready for k3s to be installed** — which is exactly where this
brief stops. Repeat for every NUC and you have a uniform fleet waiting to be turned into a
cluster.
