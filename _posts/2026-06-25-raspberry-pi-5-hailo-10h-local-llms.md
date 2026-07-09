---
title: "Running Local LLMs on a Raspberry Pi 5 with the Hailo-10H"
date: 2026-06-25
tags: [home, homelab, raspberry pi, hailo, ai, llm, npu]
version: 1.0
excerpt: ""
---
## Introduction
I decided to build a local LLM box because I was tired of the only two options anyone seemed to offer: rent inference from someone else’s cloud, or buy a GPU loud enough to double as a space heater.

What I actually wanted was smaller than that. *A private, low-power LLM box that isn’t a GPU furnace*: something that sits on the shelf with the rest of the homelab, draws a couple of watts, and just answers questions without phoning home to do it.

The Hailo-10H’s numbers were the pitch: 40 TOPS, and, unlike the older Hailo boards, 8 GB of dedicated memory to actually run a language model, not just watch a camera feed. That’s the moment it stopped being "interesting hardware" and became something worth ordering.

The happy path is already covered by Raspberry Pi and Hailo, so there’s no point repeating it in full. I’ll link it where it’s relevant. What isn’t covered is the realistic failure that hits at the verification step: what it looks like, how to diagnose it top-down, and the model and architecture decisions that come after the card finally works.

## The Goal
Run small local LLMs through the Hailo-Ollama server.

Not much more than that, really. A short list of constraints, all self-imposed:

- **Low power.** Something that can stay on all day without showing up on the electricity bill.
- **Local, so unlimited.** No API, no per-token bill, no rate limit. Ask it a thousand questions or ten, it doesn’t care.
- **Not a gaming GPU.** No 300 W card, no separate PC just to host it, no fan spinning up like a jet engine every time I ask it something.
- **Light.** A board the size of a deck of cards, not a tower under the desk.

Expectations set accordingly: low.

It’s not going to out-think a frontier model. It’s not going to be fast in the way a GPU is fast. It just has to work, and be usable: a small model, answering small questions, for a couple of watts.

## Hardware you need to purchase
Raspberry Pi offers three different types of AI modules.

- *Raspberry Pi AI Kit*: an M.2 HAT+ with a pre-installed Hailo-8L NPU.
- *Raspberry Pi AI HAT+*: either an on-board Hailo-8L NPU or an on-board Hailo-8 NPU.
- *Raspberry Pi AI HAT+ 2*: an on-board Hailo-10H NPU.

The AI Kit is no longer in production, so for a new build only the AI HAT+ or AI HAT+ 2 is actually available. Worth mentioning: only the AI HAT+ 2 can run Generative AI (GenAI) models. That’s the board we want. More details [here](https://www.raspberrypi.com/documentation/accessories/ai-hat-plus.html).

Where to buy it, though? Turns out it’s surprisingly hard to find used. I tried, and I failed. I also tried the usual consumer electronics sites here in Switzerland, but the listings’ photos and descriptions never quite matched what I was actually looking for.

So here’s the shop I ended up using for the [Raspberry Pi AI HAT+ 2 board](https://www.pi-shop.ch/raspberry-pi-ai-hat-plus-2). For reference, here it is without its passive heat sink:
![The 40 tera-operations per second (TOPS) Raspberry Pi AI HAT+ 2]({{ '/assets/images/2026-06-25/ai-hat-plus-2-hero.jpg' | relative_url }})

This board only works with the Raspberry Pi 5, and the official guide recommends the 16 GB RAM version. Who am I to argue with the guide?
Same story as the board: hard to find the Pi itself used, but at least easier to find new on the usual Swiss B2C sites. Here’s [an example](https://www.digitec.ch/en/s1/product/raspberry-pi-5-16gb-single-board-computer-kits-53945114).

Like every AI setup, it runs hot fast, so the guide *strongly* recommends a heat sink for the Pi itself too. Here’s the [official active heat sink](https://www.digitec.ch/en/s1/product/raspberry-pi-official-fan-heat-sink-for-5-development-board-accessories-38955610).

## Mount the hardware
Purchases done. Fast forward to the day the postman rings the bell and hands over all these boards.

Time to mount everything.

First up: the passive heat sink on the AI HAT+ 2 board. Two small things I got wrong at the very start:
- **Remove the plastic film carefully.** The thermal pads are already positioned correctly for you. I was too excited and knocked one loose peeling the film off.
- **Check the orientation**, using the Raspberry Pi logo on both the heat sink and the board. The heat sink physically fits either way round, but if it’s not aligned correctly the thermal pads won’t line up with the chips that actually need the cooling.

Here’s the result:
![The AI HAT+ 2]({{ '/assets/images/2026-06-25/IMG_2101.jpg' | relative_url }})

Next, the active heat sink on the Raspberry Pi itself. This one’s easier: there’s only one way it fits, and it connects the fan cable at the same time. You’ll know it’s seated correctly if the fan spins up for a couple of seconds the moment you power the Pi on.

Here it is:
![The Raspberry Pi 5 with heat sink]({{ '/assets/images/2026-06-25/IMG_2099.jpg' | relative_url }})

Both boards, ready to be connected:
![The Raspberry Pi 5 and the AI HAT+ 2]({{ '/assets/images/2026-06-25/IMG_2098.jpg' | relative_url }})

The AI HAT+ 2 connects to the Raspberry Pi 5 over its PCIe port, a high-speed interface built exactly for this kind of board-to-board connection.

Make sure the Pi is fully disconnected from power first, then:

- Fit the spacers. Using a crosshead screwdriver, attach the four spacers to the yellow holes on your Raspberry Pi 5 using the four longer screws.
- Connect the GPIO stacking header. Align the GPIO stacking header with the GPIO pins on your Raspberry Pi 5. Press down firmly until the header is fully seated. The orientation of the header doesn’t matter so long as all the GPIO pins are correctly aligned and inserted.
- Disconnect the PCIe ribbon cable from your AI HAT. Slide the retaining clips outwards from both sides of the PCIe connector on the AI HAT and then gently pull out the cable.
- Insert the PCIe ribbon cable into your Raspberry Pi. Insert the other end of the ribbon cable into the PCIe connector on your Raspberry Pi 5. To do this, first slide the retaining clip of the PCIe connector on your Raspberry Pi 5 upwards from both sides. Then, insert the ribbon cable into the PCIe connector. Ensure that the metallic contact points are facing inwards (towards the USB ports).
- Secure the PCIe ribbon cable to your Raspberry Pi. While holding the ribbon cable in place, push the retaining clip back into the connector from both sides, ensuring that the cable is evenly inserted.
- Mount the AI HAT. With the main components of your AI HAT facing upwards, align the mounting holes on the AI HAT with the spacers on your Raspberry Pi 5. Use the four remaining (shorter) screws to secure the AI HAT in place.
- Insert the PCIe ribbon cable into your AI HAT. Insert the unconnected end of the ribbon cable into the PCIe connector on your AI HAT. To do this, first slide the retaining clip of the PCIe connector on your AI HAT outwards from both sides. Then, insert the ribbon cable into the PCIe connector.
- Secure the PCIe ribbon cable to your AI HAT. While holding the ribbon cable in place, push the retaining clip back into the connector from both sides, ensuring that the cable is evenly inserted.

Here’s a good picture from the official guide:
![Mount the AI HAT+ 2]({{ '/assets/images/2026-06-25/ai-hat-plus-installation-02.jpg' | relative_url }})

And here’s my final result!
![The Raspberry Pi 5 and the AI HAT+ 2]({{ '/assets/images/2026-06-25/IMG_2199.jpg' | relative_url }})

## Install the OS
Hardware done. Time for software.

First thing: the OS. To use the AI HAT+ 2 board, it has to be the 64-bit Debian Trixie distribution; otherwise the OS simply can’t talk to the board.

I installed it with the Raspberry Pi Imager, onto a 32 GB SD card, plenty for a first test. I won’t go deep into this part, the Imager is genuinely self-explanatory. [Download it here](https://www.raspberrypi.com/software/).

## Follow the guide from Raspberry Pi
With the OS on the SD card, it’s time to boot the Pi for the first time.

Once it’s up, make sure it’s running Raspberry Pi OS Trixie with the latest software and the latest firmware:

```bash
sudo apt update
sudo apt full-upgrade -y
sudo rpi-eeprom-update -a
sudo reboot
```

With the Pi fully updated, the NPU needs three more things:
- The Hailo kernel device driver and firmware.
- The HailoRT middleware.
- The Hailo TAPPAS core post-processing libraries.

And here’s where the fun starts.

This next line is the single most important one in the whole setup: the AI HAT+ 2 uses a **different package** from the Hailo-8/8L boards.

```bash
sudo apt install dkms
sudo apt install hailo-h10-all
```

Most older guides and forum posts say `hailo-all`. That’s for the Hailo-8/8L. The two packages cannot co-exist. Install the wrong one and the firmware never matches the 10H, which is the classic reason verification returns nothing.

If `hailo-all` is already on the machine, purge it first:

```bash
sudo apt remove --purge hailo-all hailort hailo-dkms hailo-tappas-core
```
A correct H10 install looks like this:

```
hailo-h10-all 5.1.1            (metapackage)
h10-hailort 5.1.1              (runtime library)
h10-hailort-pcie-driver 5.1.1  (PCIe driver + firmware, DKMS source)
python3-h10-hailort 5.1.1
hailo-tappas-core 5.1.0
```
After installing the dependencies, reboot the Pi. Or better yet, shut it down and pull the power adapter: a cold restart that actually starts cold.

```bash
sudo reboot
```
After the reboot, check that everything’s running correctly:
```bash
hailortcli fw-control identify
```
A healthy result ends with the line that actually matters:
```
Device Architecture: HAILO10H
```
That’s the happy path.

Mine stopped one line short.

## The problem: a blank `fw-control identify`
After the full install, the verification command printed **nothing**.

No output. No error. A blank result.

And that’s the first useful clue, even though it doesn’t feel like one.

> A blank result, rather than an error message, almost always means the tool can’t find a usable device node. `/dev/hailo0` doesn’t exist, so there’s nothing to talk to.

So the question isn’t "why did it error?" There’s no error. The question is "where, between the hardware and the device node, did the chain break?"

You answer that top-down.

### The diagnostic path

I worked down the chain, one check at a time, ruling things in and out as I went:

| Check | Command | Result | Meaning |
|---|---|---|---|
| Right package? | `dpkg -l \| grep hailo` | `hailo-h10-all 5.1.1` present | Correct H10 stack, not a wrong-package problem |
| Board on the bus? | `lspci \| grep -i hailo` | `Hailo-10H AI Processor (rev 01)` at `0001:01:00.0` | Hardware present and enumerating, seating fine |
| Driver built? | `dkms status` | `hailo1x_pci/5.1.1, 6.12.75+rpt-rpi-2712: installed` | Module built, but note that kernel version |
| Device node? | `ls -l /dev/hailo*` | `No such file or directory` | No node → nothing to read |
| Module loaded? | `sudo modprobe hailo1x_pci` | `FATAL: Module hailo1x_pci not found in directory /lib/modules/6.18.33+rpt-rpi-2712` | The smoking gun |

Read that last row against the third one.

The driver was built for kernel **6.12.75**.

The system was now running kernel **6.18.33**.

### The root cause

The `apt full-upgrade` from the update step had quietly pulled in a **newer kernel**.

The Pi rebooted into it. And DKMS never rebuilt the Hailo driver for the new kernel, because the kernel headers for the *running* kernel were missing, so the auto-rebuild hook couldn’t fire.

The driver existed. Just for a kernel that was no longer running.

Every symptom traced back to that one cause: empty module list, no hailo lines in `dmesg`, no `/dev/hailo0`, blank verification.

One more detail that sends people down the wrong path: the 10H kernel module is named **`hailo1x_pci`**, not `hailo_pci` like the Hailo-8 stack. Wrong name, wrong search results, wrong fix.

### The fix

Install the headers for the running kernel, rebuild the driver against it, load it:

```bash
# install headers matching the running kernel
sudo apt update
sudo apt install linux-headers-rpi-2712

# rebuild the Hailo driver against the running kernel and load it
sudo dkms autoinstall -k $(uname -r)
sudo depmod -a
sudo modprobe hailo1x_pci

# verify
dmesg | grep -i hailo
ls -l /dev/hailo*
hailortcli fw-control identify
```

After the rebuild, `dmesg` showed the full firmware boot sequence (customer certificate, SCU firmware, U-Boot, fitImage, image-fs), then `Firmware loaded in 6504 ms` and `Added board ... /dev/hailo0`.

`fw-control identify` returned a real `HAILO10H` result.

A reboot confirmed the module auto-loads and the node comes back on its own. The fix is permanent.

> That ~6.5-second firmware load happens on *every* cold boot. The 10H reflashes its SOC image each time. It’s normal, not a fault.

### The lesson

Keep `linux-headers-rpi-2712` installed.

As long as the headers are present, DKMS rebuilds the Hailo driver automatically whenever a new kernel lands. The headers being *absent* during the initial install is the entire reason this happened. Present headers, and it never recurs.

And the diagnostic shape is worth remembering on its own:

> A blank `fw-control identify` is a "no device node" symptom, not a "no hardware" symptom. Diagnose top-down: package → `lspci` → `dkms status` → `/dev/hailo0`.

### Other ways this exact symptom shows up

I ruled these out, but they produce the same blank line and are worth knowing:

- **OS isn’t Trixie.** Bookworm gives the identical blank symptom.
- **No reboot after install.** The driver builds via DKMS and firmware loads at boot, so a reboot is mandatory before verification.
- **Warm reboot vs cold power-cycle.** The 10H loads firmware at PCIe probe, and its state can persist across a warm reboot. A full cold power-off (pull power ~10 s) sometimes fixes a board that enumerates but shows no node.
- **PSU headroom.** Underpowered supply → enumerates but firmware load fails silently. Check `dmesg | grep -i "under-voltage"`.

## Running the models
With the card alive, the GenAI path is short.

Install the Hailo Model Zoo GenAI package (5.1.1 at time of writing) from the Hailo Developer Zone:

```bash
sudo dpkg -i hailo_gen_ai_model_zoo_5.1.1_arm64.deb
```

Start the server and list what’s bundled:

```bash
hailo-ollama
# in a second terminal:
curl --silent http://localhost:8000/hailo/v1/list
```

```json
{"models":["deepseek_r1_distill_qwen:1.5b","llama3.2:3b","qwen2.5-coder:1.5b","qwen2.5-instruct:1.5b","qwen2:1.5b"]}
```

Pull one and chat:

```bash
curl --silent http://localhost:8000/api/pull \
     -H 'Content-Type: application/json' \
     -d '{ "model": "qwen2.5-instruct:1.5b", "stream": true }'

curl --silent http://localhost:8000/api/chat \
     -H 'Content-Type: application/json' \
     -d '{"model": "qwen2.5-instruct:1.5b", "messages": [{"role": "user", "content": "Explain what a reverse proxy does in two sentences."}]}'
```

A 3B model download is about 3.37 GB, roughly 3.4 GB on disk, which is in line for a 3B compiled for the 10H.

## `hailo-ollama` as a service
Running `hailo-ollama` by hand means a terminal has to stay open somewhere. Close the SSH session, or reboot the Pi, and the server dies with it. Not exactly "boots up and just works."

Same fix as the Minecraft server: a `systemd` service.

First, find where the binary actually lives. You’ll need the exact path for the unit file:

```bash
which hailo-ollama
```

Then create the service:

```bash
sudo vim /etc/systemd/system/hailo-ollama.service
```

With this content (adjust `User` and the `ExecStart` path to match your `which` output and your username):

```
[Unit]
Description=Hailo Ollama Server
After=network.target

[Service]
User=pi
ExecStart=/usr/bin/hailo-ollama
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Reload `systemd` and enable it:

```bash
sudo systemctl daemon-reload
sudo systemctl enable hailo-ollama
sudo systemctl start hailo-ollama
```

Check it’s actually alive:

```bash
systemctl status hailo-ollama
curl --silent http://localhost:8000/hailo/v1/list
```

Then reboot, to make sure it comes back on its own and not just because you started it manually:

```bash
sudo reboot
```

After the reboot, that same `curl` should answer immediately: no SSH-in-and-start-it-by-hand step in between. The Pi boots, and the model server is just there.

## The web client: Open WebUI
We want a UI for the LLMs, so let’s install Open WebUI. It runs in Docker: it’s incompatible with Python 3.13 on Trixie, hence the container.

### Installing Docker
Because of that Python incompatibility, the Raspberry Pi needs a specific Docker install path. The [official guide](https://www.raspberrypi.com/documentation/computers/ai.html#step3-llm) covers it, but here are the steps:

1. Remove any existing Docker packages:
```bash
sudo apt remove $(dpkg --get-selections docker.io docker-compose docker-doc podman-docker containerd runc | cut -f1)
```
2. Install the Docker apt repository:
```bash
# Add Docker's official GPG key:
sudo apt update
sudo apt install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/debian
Suites: $(. /etc/os-release && echo "$VERSION_CODENAME")
Components: stable
Signed-By: /etc/apt/keyrings/docker.asc
EOF

sudo apt update
```
3. Check that the `docker.sources` file was created correctly:
```bash
cat /etc/apt/sources.list.d/docker.sources
```

The output should look like this, where `Suites` is your OS’s `VERSION_CODENAME` (trixie):
```
Types: deb
URIs: https://download.docker.com/linux/debian
Suites: trixie
Components: stable
Signed-By: /etc/apt/keyrings/docker.asc
```
4. Install and start the Docker service:
```bash
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

sudo systemctl start docker
```

5. Create a `docker` group (skip this if it already exists, since the `docker-ce` package usually creates it for you):
```bash
sudo groupadd docker
```
6. Add your user to the `docker` group:
```bash
sudo usermod -aG docker $USER
```

### Pull and run Open WebUI
With Docker installed, pull the Open WebUI image and run it:

```bash
docker pull ghcr.io/open-webui/open-webui:main
docker run -d -e OLLAMA_BASE_URL=http://127.0.0.1:8000 \
  -v open-webui:/app/backend/data --name open-webui \
  --network=host --restart always ghcr.io/open-webui/open-webui:main
```

Then browse to `http://127.0.0.1:8080`.

## Running tests with multiple models
Finally, the part I’d been waiting for: running my own LLMs on the Raspberry Pi.

I downloaded every model available for the AI HAT+ 2 board (list them by calling `http://localhost:8000/hailo/v1/list`) and ran a batch of random prompts through each. It was genuinely fun. Here’s what I found:

- **Coding work: `qwen2.5-coder:1.5b`.** Purpose-built for code. It beats the 3B on programming tasks despite being half the size, and in live use it felt *noticeably faster*. The clear pick for homelab scripting and dev work.
- **General chat: `qwen2.5-instruct:1.5b`.** Same Qwen 2.5 family, same 1.5B size, same speed class as the coder, but tuned for conversation and instruction-following instead of code. My recommended general-purpose model.
- **Heavier fallback: `llama3.2:3b`.** The largest in the bundled list (the only 3B; everything else is 1.5B), so the best general coherence, but the slowest of the bunch. Keep it for when more general knowledge is worth the speed hit.
- **`deepseek_r1_distill_qwen:1.5b`** is the reasoning/math specialist. Strong at step-by-step problems, but it emits visible "thinking" tokens, so it’s slower and overkill for ordinary chat.
- **Skip `qwen2:1.5b`.** Oldest in the list, superseded by `qwen2.5-instruct:1.5b`. No reason to pick it.

> Final pairing: `qwen2.5-coder:1.5b` for dev work, `qwen2.5-instruct:1.5b` for everything else. Both fast 1.5B models. `llama3.2:3b` as the slower, heavier fallback.

### The counterintuitive bit

In live use the 1.5B coder was clearly faster than the 3B Llama. Which sounds obvious: fewer parameters, faster decode.

But the real reason is more interesting than parameter count. Here are the INT4 decode numbers (batch 1):

| Model | Params | ~Decode speed | Notes |
|---|---|---|---|
| Phi-2 | 2.7B | ~19 tok/s | Fastest on the chip (official native HEF) |
| Llama 3 8B | 8.0B | ~11 tok/s | Official native HEF |
| Llama 2 7B | 7.0B | ~10 tok/s | Older; beaten by Llama 3 8B |
| Qwen2.5-Coder 1.5B | 1.5B | ~7.9 tok/s | Community Q4_0 via hailo-ollama |
| DeepSeek-R1-Distill 1.5B | 1.5B | ~6.8 tok/s | Reasoning, verbose |
| Qwen2.5-Instruct 1.5B | 1.5B | ~6.8 tok/s | General chat pick |
| **Llama 3.2 3B** | 3.0B | **~2.65 tok/s** | Community Q4_0, the slow one |

Look at the two extremes.

A native-compiled **8B** runs at ~11 tok/s. A community Q4_0 **3B** runs at ~2.65 tok/s.

The bigger model is *four times faster*.

> The takeaway: native-compiled HEF models run much faster than community Q4_0 GGUF models routed through hailo-ollama. That’s why the 1.5B coder feels snappier than the 3B Llama, and why an *official* 8B HEF would beat the 3B too, despite being far larger. Native HEF Llama 3.2 1B reportedly hits 30–50 tok/s, 10–25× the Pi 5 CPU.

## "Can I run a bigger model?"

So why didn’t I just pick the biggest model from that table? Let me save you the investigation.

**Physically?** Yes. The 10H has a direct DDR interface and 8 GB on board, and supports roughly 1.5B–8B via INT4. Llama 3 8B at ~5.2 GB fits under the budget, and is the largest model that does.

**The catch:** every model has to be pre-compiled into Hailo’s HEF format. You can’t point hailo-ollama at an arbitrary Hugging Face model. The bundled list tops out at `llama3.2:3b`.

**The outcome:** Hailo has *demonstrated* Llama 3 8B on the 10H in benchmarks, but they haven’t shipped a downloadable 8B HEF for the chip. I checked the Developer Zone GenAI model zoo: no 8B listed. So today, on this hardware, **~3B is the practical ceiling for ready-made models.** An actual 8B would mean compiling your own HEF via the Dataflow Compiler, a multi-day, advanced project on a separate x86 host.

And here’s the honest framing, because it matters more than the disappointment:

> The Pi + Hailo-10H is built to run *small models fast at ~2.5 W*. The missing 8B HEF is a symptom of that design intent, not a bug. It’s a fast, private, low-power small-model engine, not a frontier-model host.

Once I stopped wanting it to be a GPU, it became a much better board.

## Reflection

I went into this wanting the board to be bigger than it is.

I wanted the 8B. I wanted it to join the cluster and accelerate the heavy model. I wanted the thing the marketing slides imply.

None of that is what this board is for.

The 10H is a small-model engine that runs fast and sips power. Every limit I hit (the missing 8B HEF, the no-distributed-acceleration answer, even the 3B being slow) points back to that one design intent. They’re not bugs. They’re the board telling me what it’s good at.

And the install failure taught the same lesson from the other side.

The blank `fw-control identify` looked like a dead board. It wasn’t. The hardware was fine, the package was right, the driver was even built, just for the wrong kernel. The failure wasn’t dramatic. It was a missing headers package and an auto-rebuild that never fired. Boring. Specific. Fixable in three commands once I stopped guessing and walked the chain top-down.

That’s the Second Shelf pattern again.

The interesting work wasn’t building something impressive. It was accepting what the hardware actually is, diagnosing the unglamorous real cause instead of the dramatic imagined one, and taking the path that works today over the path that’s theoretically fastest.

A fast, private, low-power box that runs small models well.

That’s not a compromise. That’s the whole point.

## Conclusion

What a journey! It was long, but it works.

The card is recognised, the driver survives kernel upgrades, and two fast 1.5B models cover real work (code and chat), fully local, no cloud, no monthly cost.

If you build the same thing, the only two parts you really need to get right are upstream of everything else:

- Install `hailo-h10-all`, not `hailo-all`.
- Keep `linux-headers-rpi-2712` installed so DKMS rebuilds the driver when the kernel moves.

Get those right and the blank line never appears.

Get them wrong and you’ll spend an evening staring at a cursor, like I did.