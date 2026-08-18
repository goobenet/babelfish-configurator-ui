# Babelfish — User Guide

Babelfish is a broadcast-radio logic hub. It glues together the contact closures,
tallies, netcues, and GPIO of a facility — on-air lights, mic tallies,
satellite-receiver triggers, links between consoles, codecs, automation systems
and relays — and lets outputs **follow** inputs with no PC or broker in the loop.

This guide covers the whole ecosystem. **Babelfish Config Studio** (the companion
app) is where you actually build the configuration; this document explains the
concepts behind it.

---

## Contents

1. [What Babelfish is](#1--what-babelfish-is)
2. [First-time setup (a fresh unit)](#2--first-time-setup-a-fresh-unit)
3. [The device family](#3--the-device-family)
4. [Setting a board's address](#4--setting-a-boards-address)
5. [Naming & describing devices](#5--naming--describing-devices)
6. [Wiring logic: the Follows engine](#6--wiring-logic-the-follows-engine)
7. [Events & TCP (VCom)](#7--events--tcp-vcom)
8. [MQTT & Home Assistant (optional)](#8--mqtt--home-assistant-optional)
9. [Discover, backup, restart & factory reset](#9--discover-backup-restart--factory-reset)
10. [Cheat-sheet](#10--cheat-sheet)

---

## 1 · What Babelfish is

A Babelfish install is a small distributed system, not one box:

- **The hub ("Model 42")** — an ESP32 with an IP address. It runs the web UI, the
  logic engine, MQTT, discovery, and is the master of three buses (**A**, **B**,
  **C**). This is the only part with a network address.
- **I/O boards** — cheap satellites, one per physical connector panel. Each is a
  dumb, robust board that reads its opto inputs and drives its relays / GPIO.
  Up to **8 boards per bus** (addresses 0–7).

The magic is the **follows** engine: an *output* mirrors one or more *sources*,
computed on the hub itself. Wire a contact closure to an input on one board and a
relay on another board follows it — even across buses — with **no MQTT broker
required**. MQTT and TCP are add-ons for remote visibility and cross-system glue.

> **ℹ "Input" and "output" are always from Babelfish's point of view.**
> An *input* is a signal coming *into* Babelfish; an *output* is something
> Babelfish drives *out* to your gear. Most opto / LIO pins are **active-low**:
> pulled electrically low = logically **active / ON** in the tool and in the
> config.

---

## 2 · First-time setup (a fresh unit)

> **💡 A factory-fresh Babelfish comes up on the static address `192.168.42.42`**
> (fitting for a "Model 42"). It is **not** on DHCP out of the box, so you must
> meet it on its own subnet for the first connection.

1. **Cable up.** Connect the hub's Ethernet directly to your PC (or a small switch).
2. **Put your PC on the 42 subnet.** Set your Ethernet adapter to a static address
   in `192.168.42.x`, for example:

   | Setting     | Value             |
   |-------------|-------------------|
   | IP address  | `192.168.42.10`   |
   | Subnet mask | `255.255.255.0`   |
   | Gateway     | *(leave blank)*   |

   *Windows: Settings → Network & Internet → Ethernet → IP assignment → Edit → Manual.*
3. **Connect.** In Config Studio click **New unit?** (fills `192.168.42.42`), then
   **Connect & Load**. Or browse to `http://192.168.42.42/`.
4. **Give it its real address.** On the **Network & Global** tab set the hostname
   and either a static `ethernet_ip` / mask / gateway / DNS, *or* clear all Ethernet
   fields for DHCP. **Save to unit**, then **Restart unit**.
5. **Reconnect on the new address.** Put your PC back on your normal LAN. Use
   **Discover** to find the unit at its new address (it broadcasts a beacon), or
   type the IP you assigned.

> **⚠ Changing the IP takes effect only after a Restart**, and the unit will move
> to the new address — you'll reconnect there. If you ever lose it, the factory
> reset in [chapter 9](#9--discover-backup-restart--factory-reset) brings it back
> to `192.168.42.42`.

---

## 3 · The device family

Each board is a generic I/O board whose electrical shape suits a class of gear. An
example configuration is pre-loaded showing *one typical use* per type, but that's
just an example — any board can drive or read anything compatible with its I/O.
Pick the type whose **I/O shape** matches the panel you're wiring:

| Type (in config) | Board            | I/O shape                                   | Typical use (example)               |
|------------------|------------------|---------------------------------------------|-------------------------------------|
| `baton`          | Baton            | 8 relay outputs (isolated dry closures)     | Devices that require a voltage connection |
| `gpio`           | General Porpoise | 8 in / 8 out, dry-contact                    | Any dry-contact GPI/GPO device      |
| `eastwind`       | East Wind        | 5 in / 5 out (15-pin D-SUB)                  | Axia xNode                          |
| `burbank`        | Burbank          | 5 in / 5 out                                 | SAS consoles                        |
| `banjo`          | Banjo            | 2×6 LIO, per-pin direction                   | Wheatstone LIO                      |
| `sputnik`        | Sputnik          | 16 cue in + alarm + 2 serial RX (DB37)       | Wegener / XDS satellite receivers   |

**All I/O boards are addressed by jumper straps** (see [chapter 4](#4--setting-a-boards-address)).

---

## 4 · Setting a board's address

Every board on a bus needs a unique address **0–7**. That address, plus the bus
letter, is how you refer to the board in the config. Two boards on the *same bus*
must never share an address; the same address on *different* buses is fine.

> **⚠ The exact strap layout isn't nailed down here.** The address is 3 bits, so
> it's set by three jumper straps — but *which* physical position is which bit, and
> whether "fitted" means 1 or 0, depends on the board revision and isn't in the
> material this guide was built from. **Don't trust a printed table for this — find
> it once on the bench** with the two-minute check below, then write it down.

### Find it empirically (two minutes, one board)

1. Wire **one** board to a bus and power up the hub.
2. Run a **Bus Scan** — click **Scan buses** on the **Devices** tab in Config
   Studio (or open the hub's own page at `http://<hub-ip>/busscan`). It probes
   every bus and address 0–7 and shows a grid of what it found. Config Studio can
   also add any detected board straight into your configuration.
3. Note where your board appears (which bus, which address).
4. **Change one strap, re-scan.** Watch which address it moves to. Two or three
   changes reveal the whole pattern — which strap is the 1s / 2s / 4s bit, and
   whether fitted counts as 1 or 0.
5. Record it in the worksheet below and set the rest of your boards to match.

### Worksheet — the usual 3-bit binary pattern

Most strapped boards use a plain binary scheme: three straps weighted **×1**,
**×2**, **×4** that add up to the address. Treat this as a *starting hypothesis*
and tick **Confirmed** once your bus scan agrees (if it reads inverted, swap
fitted ↔ removed and the pattern still holds):

| Address | strap ×4 | strap ×2 | strap ×1 | Confirmed |
|:-------:|:--------:|:--------:|:--------:|:---------:|
| **0**   | ○        | ○        | ○        | ☐         |
| **1**   | ○        | ○        | ●        | ☐         |
| **2**   | ○        | ●        | ○        | ☐         |
| **3**   | ○        | ●        | ●        | ☐         |
| **4**   | ●        | ○        | ○        | ☐         |
| **5**   | ●        | ○        | ●        | ☐         |
| **6**   | ●        | ●        | ○        | ☐         |
| **7**   | ●        | ●        | ●        | ☐         |

● fitted   ○ removed — but let the Bus Scan tell you which way *your* board reads.

> **ℹ** The number you set on the straps is the `addr=` value you type for that
> board on the **Devices** tab. Always verify with a **Bus Scan** after wiring.

---

## 5 · Naming & describing devices

On the **Devices** tab, give each board a short **ID** (letters, digits, `-`, `_`
— no spaces). Good IDs read like the room they're in: `relay-a0`, `xnode-studioB`,
`sat1`. That ID is how you pick the board in every dropdown and how it appears over
MQTT.

Add a free-text **description** for the board and, optionally, a description per
pin. Descriptions are purely for humans — they make the Follows and Events
dropdowns and the MQTT model readable.

**Banjo only:** each of its 12 LIO pins is individually an input or an output. Use
the click-to-toggle **direction editor** on the device card — `I` = Babelfish
receives (from an LIO output), `O` = Babelfish drives (to an LIO input). The
available pins update to match.

---

## 6 · Wiring logic: the Follows engine

This is the heart of Babelfish and needs no broker. On the **Follows** tab you say:

> *"This **output** turns on while that **source** is active."*

Pick a **target device** and one of its **output** pins, then a **source**:

- **Another device's input** — e.g. `relay3 follows xnode-b0:in3`. The relay is on
  whenever that input is active. Sources can be on any bus.
- **Serial RX match (netcue)** — e.g. `relay6 follows sat-c0:rx1=R72`. Fires a
  ~250 ms pulse each time the receiver's serial line equals `R72`.
- **MQTT boolean topic** — mirror a true/false topic from another system, verbatim.
- **MQTT topic match** — pulse when a topic's payload equals a string.

One output can follow **several** sources — it turns on if *any* of them is active
(logical OR). An output that follows anything becomes read-only over MQTT, because
its state is now owned by the logic engine. Because the wiring lives on the
*consuming* output, you can always read a config top-to-bottom and know exactly
what drives each output — there is no hidden "push".

---

## 7 · Events & TCP (VCom)

When following a level isn't enough — you want to *send a command* on an edge —
add a **VCom** device (Devices tab → Add → VCom) and use the **Events** tab. A
VCom is both a TCP server (on a `port` you pick) and an event scripter. Each event
reads:

> *"**when** source **goes on** / **goes off**, do an action."*

- **send** — broadcast a text payload to every client connected to this VCom's
  server port. Handy for automation systems that listen on TCP.
- **tcpsend** — open a one-shot connection to any `ip:port` and fire a payload.
  This is how a contact closure drives a networked device — classically **Axia
  LWRP** on port `93`:

  ```
  when gpio-a1:in2 goes on  tcpsend 172.22.22.90:93  \nLOGIN\nGPO 3 Lxxxx\n
  ```

  Payloads take C-style escapes (`\n \r \t \\ \"`). For string-match sources the
  ~250 ms pulse gives you a synthetic on→off pair, so you can send one thing at the
  start and another at the end.

---

## 8 · MQTT & Home Assistant (optional)

MQTT is entirely optional — the Follows engine runs without it. Turn it on by
filling `mqtt_server` (and user / password) on the **Network & Global** tab. Then
every board becomes a node under `homie/<hostname>/<device-id>/`, one property per
pin: inputs are read-only booleans, settable outputs are booleans, Sputnik serial
lines are strings.

Set `mqtt_home_assistant_discovery=1` and each pin shows up automatically in Home
Assistant as a switch or binary sensor, grouped by physical board. You can also
feed logic *from* MQTT using the `mqtt:` sources on the Follows and Events tabs —
that's the cross-system glue.

> **ℹ** With MQTT enabled, outputs stay safe on boot: they don't fire until their
> retained state has been received, so a reboot never slams a relay to a stale
> value.

---

## 9 · Discover, backup, restart & factory reset

- **Built-in web interface** — every unit also serves its own basic web pages at
  `http://<unit-ip>/`. Point any web browser at the unit's IP (outside this tool)
  for a status overview, the **Bus Scan** page, a raw config editor, and a restart
  button. Config Studio is just a friendlier front end to the same unit.
- **Discover** — every hub broadcasts a UDP beacon; the **Discover** button listens
  for it and lists units with their IP and firmware. No DHCP-table hunting.
- **Backup** — always **Download backup** before your first write to a unit; it
  saves the exact `config.txt` to a timestamped file you can paste back.
- **Save vs Restart** — **Save to unit** writes the config but *outputs don't change
  until you Restart unit*. A restart briefly drops the unit off the network and does
  **not** send unsaved edits — Save first.
- **Factory reset** — the factory-reset button is **inside the case**: remove the
  top cover to reach it. Hold it for ≥ 8 seconds at power-up; the hub
  reformats its filesystem and writes a fresh default config, returning to
  `192.168.42.42`. Use this if you lose the address or want a clean slate.
- **Firmware** — the hub updates over OTA (its default upload target is an IP). The
  board satellites rarely change.

---

## 10 · Cheat-sheet

| I want to…                        | Where                                   | Result in config.txt                    |
|-----------------------------------|-----------------------------------------|-----------------------------------------|
| Connect to a fresh unit           | PC on `192.168.42.x` → **New unit?**    | —                                       |
| Give a board an address           | Jumper straps (see ch. 4)               | `addr=3`                                |
| Name a board                      | Devices tab                             | `id=relay-a0`                           |
| Relay follows an input            | Follows tab                             | `relay3 follows xnode-b0:in3`           |
| Pulse on a netcue                 | Follows tab (RX match)                  | `relay6 follows sat-c0:rx1=R72`         |
| Drive an Axia GPO on a closure    | Events tab (tcpsend)                    | `when … goes on tcpsend ip:93 …`        |
| Publish everything to MQTT/HA     | Network & Global tab                    | `mqtt_server=…`                         |
| Apply changes                     | Save to unit → Restart unit             | —                                       |

*Welcome back to the good old days of device configuration. No monthly
subscription either.*
