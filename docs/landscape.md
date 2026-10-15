# The public landscape — related work and tooling

A survey of the **public projects and tools** around VW Group MIB3 that are relevant to
this repository, and an honest account of **what does and does not carry over to the
LG-built "MOI3" variant** (Qualcomm Snapdragon + Automotive Grade Linux, Ethernet
transport). This is the "related work" page: it maps the ecosystem so that a reader can
tell, at a glance, which projects target *our* platform and which target a neighbouring one.

It expands on the shorter landscape table in [cluster-map.md](cluster-map.md) §7; that
document remains the core of the repo, and [architecture.md](architecture.md) explains the
platform facts (buses, SoC/OS, transports) this page relies on. Gaps and calls for help are
in [open-questions.md](open-questions.md).

> **Scope reminder.** Everything here is a description of public, third-party work, cited by
> repository name (and URL where a public one is known). We reference and summarise; we do not
> reproduce code or proprietary material. As elsewhere in this repo, we document *that* the
> MOI3 unit is protected by Qualcomm Secure Boot, dm-verity, VW firmware signing and SMACK,
> and we do **not** describe how to defeat any of those — that is out of scope. Nothing here
> was written to a vehicle.

> **Snapshot.** This landscape was surveyed in **October 2026** (two sweeps: GitHub repository
> search and code search for the platform's internal strings). It is a moving target — the
> MHI2/MHI2Q scene in particular is very active — so treat specific project states as "as of
> that survey" and verify upstream.

## How to read this: four platform families

"MIB3" and "MIB2" are commercial umbrella names; the engineering reality is several distinct
machines. Four families matter for this survey:

| Family | SoC / OS | Cluster-video transport | Head-unit openness |
|---|---|---|---|
| **MHI2** (Harman MIB2 High) | NVIDIA Tegra T30 / **QNX 6.5** | **MOST150** optical ring | open — Java (J9) HMI, `LD_PRELOAD` hooks, SSH/root |
| **MHI2Q** (Harman MIB2 High, Qualcomm) | Qualcomm / **QNX 6.5** | **MOST150** | open — same MIB2 stack as MHI2 |
| **MPR3** (Audi Premium MIB3) | Linux/**AArch64** | cluster endpoint **unknown** in public RE | signed; no public root |
| **MOI3 (LG) — *this repo*** | Qualcomm Snapdragon / **AGL Linux** | **IP multicast over Ethernet** (`ff14::2:4`) | signed, dm-verity + SMACK; no public root |

The active public scene lives in the first two rows. This repository is the last row. The
gap between them is the whole point of this page.

---

## 1. The active scene: CarPlay / phone-map in the cluster on **Harman MHI2 / MHI2Q**

Virtually all of the living "put the phone's map on the digital cockpit" work targets the
**Harman MHI2 / MHI2Q** head units (Audi/VW/Škoda/SEAT/CUPRA, and Porsche's related MH2P).
These are **QNX** machines on the **MOST150** bus, with an **open, hookable** MIB2-family head
unit. The projects below are listed for context and contrast — **none targets MOI3 LG**, and
none is portable to it (§2). Repository names only; platform tag in brackets. `[other-platform]`

| Repository | Family | What it does (in brief) |
|---|---|---|
| `harman-f/mhi2_altscreen_carplay` | MHI2 | The base reverse-engineering effort. Proves the full path *CarPlay Auxiliary / ScreenAlt → H.264 → MPEG-TS → **MOST150** → Virtual Cockpit*, vehicle-proven on a reference Škoda MHI2 unit. Explicitly "not a finished product"; needs pre-existing **SSH** on the unit; warns it can brick the unit. GPL-3.0. |
| `omonob/MHI2-Carplay-Maps` | MHI2 | Packages the above idea for end users: sends the CarPlay map as video to the factory cluster, steering-wheel buttons for zoom / view. Recognises firmware by the exact size of a system file and refuses to flash anything it doesn't recognise. |
| `yuedizhibo/MHI2Q-CarPlay-AltScreen` | MHI2Q | The full-map "AltScreen" receiver on the Qualcomm-QNX edition: connects to CarPlay's private "type-111" second-display stream, reuses the stock decode, converts to NV12 (colour conversion moved to the GPU to cut CPU load). SD-card install. |
| `Lanye-z/MHI2Q-CarPlay-MMI-Mirror` | MHI2Q | Mirrors the MMI / CarPlay screen to the cluster on the same Qualcomm-QNX edition. |
| `kamgurgul/mib2q-carplay` | MHI2Q | Route-guidance arrows + album art + experimental full-map AltScreen to the Virtual Cockpit (and Android Auto). Built with a QNX 6.5 ARMv7 toolchain; SD-card or SSH install. |
| `chefranov/mhi2-au37x-carplay` | MHI2 | CarPlay-in-cluster for the Audi AU37x train. |
| `stevenzhr/MU1320-CarPlay-RGI` | MHI2Q | Route-guidance-into-cluster for an Audi MU1320 train. |
| `aberbic/Audi-MHI2-Virtual-Cockpit-Altscreen-CarPlay-AndroidAuto` | MHI2 | AltScreen CarPlay **and** Android Auto to the Audi Virtual Cockpit. |
| `Mr-MIBonk/M.I.B.` | MIB2.x (Harman) | The general-purpose "swiss-army knife" for Harman MIB 2.x units (ifs-root patch, SVM fix, widescreen patch, FEC-container generation). Its own README states it runs on MHIG (MIB1 High) and MHI2/MHI2Q (MIB2 High) **only** — i.e. **not** MIB3/MOI3. |

The common shape across all of them: a binary hook or Java-image patch on an **open head
unit**, installed by **SD card** (via a service menu) or over **SSH/root**, moving video onto
the cluster over **MOST**. That shape is exactly what MOI3 LG does not provide.

A useful tell from the scene itself: a consolidated MHI2Q install guide circulating in the
community **excludes MIB3 explicitly** ("cars that already have MIB3 … won't work"). The
people closest to this work treat MIB3 as a separate, unsolved world.

---

## 2. Why none of it transfers to MOI3 LG

The Harman projects are not "almost there" for our platform — they are built on three
foundations that MOI3 LG does not share. Each foundation alone would block a port; together
they mean a MOI3 solution would be a **separate reverse-engineering effort from scratch**, not
an adaptation.

### 2.1 Transport — MOST vs Ethernet IP

- **MHI2 / MHI2Q:** cluster video travels over the **MOST150** optical ring. The receiver side
  is written against MOST primitives (device nodes like `/dev/mlb/isoTX2`, MPEG-TS chunking onto
  the ring). The whole "put video on the cluster" trick *is* a MOST-writing trick.
- **MOI3 LG:** there is **no MOST** on the cluster path. The map is **video-over-IP**: an IPv6
  **multicast** stream (group `ff14::2:4`, UDP, on a dedicated VLAN), produced by the head unit
  and consumed by a separate cluster ECU — see [cluster-map.md](cluster-map.md) §2. A MOST
  writer has nothing to write to here; a working MOI3 path would mean *writing an IP-multicast
  receiver* on the cluster ECU, which is a different ECU whose firmware is not even in the
  head-unit images.

### 2.2 Operating system — QNX (+ Java) vs AGL/Linux (signed)

- **MHI2 / MHI2Q:** **QNX 6.5**, with the HMI running as an **IBM J9 Java image**. The exploited
  surface is QNX-and-Java-specific: `LD_PRELOAD` interposition on a system process, patching the
  Java image, a writable application partition, install via a green service menu or telnet/root.
- **MOI3 LG:** **Automotive Grade Linux** (Yocto, kernel 4.4.x), with a **signed, dm-verity-
  protected read-only root filesystem** and **SMACK** mandatory access control — see
  [architecture.md](architecture.md) §8 and [firmware.md](firmware.md). None of the QNX/Java
  hooking surface exists: there is no `lsd.jxe` to patch, no writable `/mnt/app`, no telnet root,
  and the binaries are **AArch64 Linux ELF**, not **ARM32 QNX ELF** — not loadable across the two.

### 2.3 SoC — and the "Qualcomm" false bridge

- A natural assumption is that **MHI2Q**, being **Qualcomm**, is a step toward MOI3 (also
  Qualcomm). It is not. MHI2Q puts a Qualcomm SoC under the **same open MIB2 QNX/Java stack** —
  MOST bus, J9 HMI, `LD_PRELOAD` hooks, SD-card install. The chip vendor is almost irrelevant to
  the exploitation surface. MOI3 LG shares only the **SoC vendor** (Qualcomm) with MHI2Q and
  nothing of the OS, boot chain, transport or install path.
- **MHI2** proper is **NVIDIA Tegra T30**, a different SoC again. CPU/clock details from that
  world are Tegra-specific and do not describe our Snapdragon unit.

### 2.4 Access / install path

- **MHI2 / MHI2Q:** the **head unit is open** — SD-card service menu or SSH/root on a writable
  system. The install packages literally copy files into the running system.
- **MOI3 LG:** **no SD-card slot** (USB only), no public root, a signed rootfs. There is no
  supported place to drop a package, and unsigned system changes are rejected by the boot chain.
  (We describe *that* this is so; defeating it is out of scope.)

### Side-by-side

| | MHI2 (Harman) | MHI2Q (Harman, Qualcomm) | **MOI3 (LG) — this repo** |
|---|---|---|---|
| SoC | NVIDIA Tegra T30 | Qualcomm | Qualcomm Snapdragon |
| OS | QNX 6.5 + Java J9 | QNX 6.5 + Java J9 | **AGL / Linux** (signed, dm-verity, SMACK) |
| Cluster transport | MOST150 | MOST150 | **IP multicast** over Ethernet |
| Hook surface | `LD_PRELOAD` + Java patch | `LD_PRELOAD` + Java patch | none public |
| Install | SD card / SSH-root | SD card / SSH-root | no SD slot, no public root |
| Binaries | ARM32 QNX ELF | ARM32 QNX ELF | AArch64 Linux ELF |

The honest reading: the overlap with MOI3 LG is close to zero at the level that matters. What
*is* shared with these projects is **the Apple side of the wire** — the CarPlay protocol for a
secondary (cluster) screen is the iPhone's, identical for any car — but that is only half the
problem, and it is the half that was never the blocker. The blocked half is the car-side
receiver, which on MOI3 LG does not exist publicly.

---

## 3. The one MIB3-family RE project — `Squallman/mpr3-carplay-vc` (Audi MPR3)

The closest public work in **architecture** is a reverse-engineering project for **Audi MIB3
Premium (MPR3)**: `Squallman/mpr3-carplay-vc`. It is the only public project in the **MIB3
(Linux/AArch64) family**, and reading it carefully is valuable because it **independently
confirms the architecture** mapped in [cluster-map.md](cluster-map.md) — while also showing the
wall. `[other-platform]`

- **Same architecture, with addresses.** It documents the same pipeline shape —
  video → *displayable* → encoder → cluster — including a VIWI bridge binary analogous to the
  one on MOI3, a `videoencoderservice`, and a `setActiveDisplayable(displayID, displayable)`
  contract. This is strong, independent corroboration that the MIB3-Linux cluster path is built
  the way we describe.
- **Platform caveat (important).** MPR3 is the **Audi Premium** variant, **not** LG MOI3. Its
  reference firmware is a specific Audi train, and its concrete literals — the numbered
  displayables (e.g. 26/29/40/42/60, and 93 for the external-smartphone surface) and the SETUP
  addresses — are **P3695 values, not MOI3 values**. They are a *cross-reference*, not something
  to copy onto an LG unit. (Interestingly, in one respect the LG picture is *ahead*: that project
  lists the active **cluster endpoint** as unknown, whereas on MOI3 LG the endpoint is the known
  `ff14::2:4` multicast — different variants, so not a literal transfer, but the architecture
  lines up.)
- **It stops at the wall, on purpose.** The author's own conclusion is that native secondary
  CarPlay video on that cluster is *not proven*: the stock receiver accepts the primary CarPlay
  display stream but **refuses the second "AltScreen" stream type**, and the project states that
  this is **not** merely a range check to flip — a real path would need a whole re-implemented
  secondary-video route plus **runtime-loader authorization** (the platform's code-signing /
  AppArmor layer). The repository is explicitly **"not deployment ready"**, ships no loader, no
  preload option and no vehicle mechanism, and is tested only against host-side fakes.

For MOI3 LG the takeaway is threefold: (a) the architecture we mapped is confirmed by a second,
independent team; (b) the "full app map on the cluster" depends on a **secondary CarPlay video
path** that is **not native even on Audi MPR3**; and (c) the **signature / loader wall is real
and has not been overcome publicly** — not even by the most thorough MIB3 RE. That matches this
repo's own scope line in [cluster-map.md](cluster-map.md) §6.

---

## 4. Public tooling that *is* for MOI3 LG — appearance, data, forensics only

There **is** a small body of public tooling for MIB3 LG. Every piece of it is **appearance,
data-format or forensics** work. **None performs a vehicle function, and none writes to the
unit** — the signed-rootfs gap in §2.4 is exactly why.

- **The `n3oka` tools** — the most advanced public author for MIB3 **LG**. A cluster of Windows
  utilities (freeware, closed binaries, "use at your own risk"), all of which **edit files on a
  PC** rather than program the unit:
  - `n3oka/MIB3-Boot-Screen-Simulator` — plays the MOI3 boot animation (LG trains) 1:1 on a PC;
    swaps images and exports GIFs. The tool states outright that it **"never writes to the
    unit … how and whether you transfer images to a unit is your own responsibility."** That line
    is the whole landscape in miniature: even the best LG tool stops at the PC.
  - `n3oka/MIB3-Image-Resolver` — reads the image assets in MIB3 data, names them, and swaps
    same-resolution images.
  - `n3oka/MIB3-3D-Viewer` — opens and edits the `.MCF` 3D files (the 3D car model / graphics the
    HMI shows); works with `.MCF` from an MOI3 LG unit and swaps textures.
  - `n3oka/MQBevo-Dataset-Editor` — a general MQB-evo vehicle **dataset** editor.
  - `n3oka/A5-2Q0-5WA-Dataset-Editor` — edits **camera** datasets (`2Q0` / `5WA` front cameras).
    (Dataset editing is a data-file activity; whether a dataset can then be written to a module is
    a separate, protection-gated question this repo does not cover.)
- **`jilleb/mib3-toolbox`** — a file-format RE toolbox dedicated to MIB3 (distinct from the same
  author's much larger MIB2 toolbox). It includes a boot-animation extractor and 010-Editor
  templates for MIB3 container formats. It is early-stage and, importantly, carries **no function
  install, no navigation and no cluster** code — it is a format/asset repo.
- **`nicolasnt3-a11y/MIB3tosaurus`** — a **forensics** parser (Python). It reads *logical
  extractions* (coredumps, logs) exported from MIB3 "OI" units, scavenges text/JSON, **extracts
  stored GPS coordinates** and plots them on an OpenStreetMap view, and recovers images from
  coredump folders. The "map" it draws is the unit's **recorded location history**, plotted after
  the fact — **not** live vehicle navigation. It is also a **privacy** signal worth flagging: it
  demonstrates that the head unit retains location history and imagery, so any such extraction is
  sensitive and should never be published un-sanitized. (This repo is ODIS-focused and does not
  use the extraction path the tool consumes.)

What this body of work confirms from the *tooling* side is the same thing §6 of
[cluster-map.md](cluster-map.md) argues from the *architecture* side: the native map (layer 1)
is coding and already works, and the full phone-app map on the cluster (layer 3) has **no
public, non-firmware path** on MOI3 LG.

---

## 5. Open-source infrastructure for understanding the internal bus

Two general-purpose open-source projects are useful *reference implementations* for the
internal **VIWI / RSI** REST-over-JSON bus that MOI3 uses between its own services (see
[cluster-map.md](cluster-map.md) §3.3). They are not VW code and are not specific to any car —
they are an independent implementation of the same VIWI concept, handy for reasoning about how
the bus behaves without touching a unit:

- `wzr1337/rsi.server`
- `wzr1337/rsi.serviceregistry`

They help explain the *shape* of the service registry (clients discovering display/stream
services over a local WebSocket/REST bus) that the firmware implements with its own registry.

---

## 6. Honest conclusion

For MOI3 LG, **the function everyone wants — CarPlay, or the phone app's map, on the instrument
cluster (layer 3) — has no public base.** The October 2026 sweeps found **zero MOI3 repositories
and zero code hits** for the platform's internal strings. The entire active "map-in-cluster"
ecosystem is **Harman MHI2 / MHI2Q** (QNX, MOST, Tegra or Qualcomm-under-QNX), and it does not
transfer (§2). The single MIB3-family RE project is **Audi MPR3**, which confirms the
architecture but stops at the signature/loader wall and ships nothing deployable (§3).

What *does* exist publicly for MOI3 LG is **appearance, data and forensics tooling** — boot
graphics, image and 3D-asset editing, dataset editors, a format toolbox, and a forensic log/GPS
parser (§4) — plus **general-purpose infrastructure analogues** for the internal bus (§5). All
of it is read/edit-on-PC; none of it is a vehicle function, and none writes to the signed unit.

In short: on this platform the **native map (layer 1) is coding and already works**; the
**phone turn-by-turn (layer 2)** appears to be factory CarPlay behaviour; and the **full app
map (layer 3)** is a signed-firmware problem with no public solution. This repository documents
the architecture and the landscape — not a way through the protections.

## Protections (scope note)

Where this page touches the protection boundary, it only states **that** the boundary exists:
MOI3 LG enforces **Qualcomm Secure Boot, dm-verity, VW firmware signing and SMACK**, and the
Audi MPR3 RE adds a **runtime-loader authorization** layer on top. Describing that these exist,
and that they are why the public scene has not reached MOI3 LG, is in scope. **Describing how to
defeat any of them is not**, and is deliberately absent here. See
[open-questions.md](open-questions.md) for what remains open *within* scope.

---

## Public references

Cited by repository name; URLs given where a public repository is known. Listing a project here
is description and attribution, not endorsement, and says nothing about its safety or legality on
any given car.

**Harman MHI2 / MHI2Q scene (other platform — contrast only):**

- `harman-f/mhi2_altscreen_carplay` — <https://github.com/harman-f/mhi2_altscreen_carplay>
- `omonob/MHI2-Carplay-Maps` — <https://github.com/omonob/MHI2-Carplay-Maps>
- `yuedizhibo/MHI2Q-CarPlay-AltScreen` — <https://github.com/yuedizhibo/MHI2Q-CarPlay-AltScreen>
- `Lanye-z/MHI2Q-CarPlay-MMI-Mirror` — <https://github.com/Lanye-z/MHI2Q-CarPlay-MMI-Mirror>
- `kamgurgul/mib2q-carplay` — <https://github.com/kamgurgul/mib2q-carplay>
- `chefranov/mhi2-au37x-carplay` — <https://github.com/chefranov/mhi2-au37x-carplay>
- `stevenzhr/MU1320-CarPlay-RGI`
- `aberbic/Audi-MHI2-Virtual-Cockpit-Altscreen-CarPlay-AndroidAuto`
- `Mr-MIBonk/M.I.B.` — <https://github.com/Mr-MIBonk/M.I.B._More-Incredible-Bash>

**MIB3-family RE (Audi MPR3 — analogous architecture, not LG):**

- `Squallman/mpr3-carplay-vc` — <https://github.com/Squallman/mpr3-carplay-vc>

**MOI3 LG tooling (appearance / data / forensics):**

- `n3oka/MIB3-Boot-Screen-Simulator` — <https://github.com/n3oka/MIB3-Boot-Screen-Simulator>
- `n3oka/MIB3-Image-Resolver` — <https://github.com/n3oka/MIB3-Image-Resolver>
- `n3oka/MIB3-3D-Viewer` — <https://github.com/n3oka/MIB3-3D-Viewer>
- `n3oka/MQBevo-Dataset-Editor` — <https://github.com/n3oka/MQBevo-Dataset-Editor>
- `n3oka/A5-2Q0-5WA-Dataset-Editor` — <https://github.com/n3oka/A5-2Q0-5WA-Dataset-Editor>
- `jilleb/mib3-toolbox` — <https://github.com/jilleb/mib3-toolbox>
- `nicolasnt3-a11y/MIB3tosaurus` — <https://github.com/nicolasnt3-a11y/MIB3tosaurus>

**Internal-bus infrastructure analogues (VIWI / RSI):**

- `wzr1337/rsi.server` — <https://github.com/wzr1337/rsi.server>
- `wzr1337/rsi.serviceregistry` — <https://github.com/wzr1337/rsi.serviceregistry>

---

*Independent research. Not affiliated with, authorized by, or endorsed by Volkswagen AG, Audi
AG, LG Electronics or Apple Inc. "CarPlay" is a trademark of Apple Inc. Repository names are
cited for attribution only. See [README](../README.md) for overall scope and license.*
