# MIB3 architecture — how the system is assembled in the car

This document describes the **architecture of the Volkswagen Group MIB3** infotainment
platform: which module is which, how the modules talk to each other, where the position
signal comes from, how sound and radio are built, and how image/video moves around the
car. The focus of this repository is the **LG-built "MOI3" variant** (firmware trains
`MOI3_EU_VW_R####L`), but most of the architecture is shared across the brands of the
group, so this page separates what is common from what is brand- or vendor-specific.

**Primary source.** The backbone of this page is the Audi/VW Self-Study Programme
**SSP 970893, "Third-generation modular infotainment matrix"** (Audi Technical Training,
2021), which is **public via the US NHTSA**:
<https://static.nhtsa.gov/odi/tsbs/2021/MC-10189329-0001.pdf>. It is described and
referenced here, not reproduced. Because the SSP is written from the **Audi** point of
view (it says "Audi virtual cockpit", "Audi connect", and uses Audi module numbers such as
`J794`, `J525`, `J949`, `J285`), Audi-only details are tagged below. A handful of
platform facts (the LG SoC/OS, the Ethernet video path to the cluster) come from our own
**read-only firmware analysis** and from **public security teardowns** of other MIB3
units; those are attributed inline.

> **Nothing on this page was read from, or written to, a vehicle.** It is analysis of a
> public document plus read-only firmware inspection. Where a claim is a deduction rather
> than something stated in a source, it says so.

### How to read the tags

| Tag | Meaning |
|---|---|
| **[MQB general]** | Holds for the MIB3 platform across the group's MQB cars (all Tier-1 vendors), unless noted. |
| **[Audi]** | Specific to Audi units / the Audi wording in the SSP. May differ on VW/SEAT/Škoda. |
| **[LG]** | Specific to the LG-built MOI3 unit (Qualcomm + AGL), confirmed from firmware. |
| **[deduced]** | Our inference from the sources, not stated verbatim — treat as a lead to verify. |

Diagnosis addresses are given in the VW/VAG scheme (two hex digits, with the UDS/ODX
four-nibble form in parentheses, e.g. `5F` / `005F`). The SSP's Audi `J`-numbers are mapped
to those addresses where known.

---

## 1. The head unit is the center of everything — address `5F`

On MIB3 the infotainment "computer" is a **single module**: the head unit. Its VW/VAG
diagnosis address is **`5F` (`005F`)**, "Information electronics 1". **[MQB general]**

Radio, the MMI displays, sound, and media playback (e.g. the optical-disc player where
fitted) are **functions of this one module**, and all of them are diagnosed under `005F`.
In practice, almost everything that is "infotainment" lives behind `5F`. The Audi SSP
calls this module **`J794`** **[Audi naming]**.

| Function | SSP / Audi designation | VW/VAG diagnosis address |
|---|---|---|
| Head unit (radio / MMI / sound / media) | `J794` Information electronics 1 | **`5F` (`005F`)** |
| Emergency-call / connectivity unit (when fitted) | `J949` | **`0075`** |
| External sound amplifier (premium systems) | `J525` | **`47`** *(VW scheme; [deduced] mapping — the SSP uses the Audi `J525` name)* |
| Instrument cluster / digital cockpit | `J285` | **`17`** |
| Gateway (CAN gateway) | `J533` | **`19`** |

The head unit participates in **component protection** (module-to-vehicle binding); on MQB
the binding is administered by the **gateway (`19`)**. This page notes only that the
mechanism exists and which modules take part — see §9. Defeating it is out of scope.

---

## 2. Buses — how the modules talk

MIB3 uses several different buses depending on *what* is being carried. This is one of the
biggest changes from MIB2.

| Bus | Carries | Notes |
|---|---|---|
| **Infotainment CAN** | Control messages; vehicle sensor data (incl. gyroscope) into the head unit | The head unit's real-time core bridges to the rest of the car over this CAN. On the public **Preh** teardown this is labelled "CAN3"; a separate CAN goes to the gateway (`19`). |
| **A2B** (Automotive Audio Bus) | Inter-module **audio**, plus control, self-diagnosis and status | Replaces MOST for audio on MIB3 (see §4). **[MQB general]** |
| **Automotive Ethernet** (100BASE-T1) | High-bandwidth links, incl. the link to the **instrument cluster** and to the telematics/connectivity unit; IP video | See §6 and §7. The direction of the whole generation is "cluster link over Ethernet, not MOST". |
| **LVDS** | Raw display image to the MMI screen(s) | See §5. |
| **MOST** | Legacy image/data ring | Still **listed** by the SSP as one of five image paths, and used on some **[Audi]** routes (see §6); the MQB units analysed here do **not** use MOST to the cluster. |
| **USB** | Phone connection, media, camera-over-USB | — |

The important takeaway: **the transport that moves a given piece of image/video differs by
brand and by Tier-1 vendor.** The SSP lists five possible image paths (next sections); the
actual wiring on a specific unit is one choice out of those five.

---

## 3. Positioning / GPS source — the head unit has none of its own

A frequently-missed point: **the MIB3 head unit has no GPS receiver, no SIM, and no SD /
memory-card slot of its own.** **[MQB general]** It does not contain the satellite
front-end. Instead:

- The **GPS/GNSS position comes from the emergency-call / connectivity module**
  (`J949` **[Audi]** / address **`0075`**) and is delivered to the head unit **over
  Ethernet**.
- The **vehicle's own sensors** (wheel speeds, gyroscope, etc.) reach the head unit over
  the **infotainment CAN**.
- The **head unit fuses** the satellite position with the sensor data to compute the
  navigation position (dead-reckoning when satellite reception drops). The position is
  *calculated in `5F`*, even though the raw fix arrives from `0075`.

The connectivity module comes in two forms, which changes its diagnosis transport:

- **"Connectivity box"** variant → diagnosed over **Ethernet**.
- **"OCU"** (online connectivity unit) variant → diagnosed over **CAN**.

It also takes part in component protection (see §9).

> **Platform variation.** Not every market/trim ships this module. **On a unit without an
> emergency-call / connectivity module, the GPS source described above is simply absent**
> — there is nothing feeding `5F` a satellite fix over Ethernet. Whether a given unit has
> it is read from the autoscan (is address `0075` present?). This is a platform variation,
> not a defect. **[MQB general / market variation]**

### Antennas

- **[MQB general]** On MQB, the **Wi-Fi and Bluetooth antennas live in the head unit
  (`5F`)**, as do the radio antennas.
- **[Audi — MLB]** On the larger MLB platform the Wi-Fi/BT antennas sit in the `J949`
  connectivity module instead. This is the non-MQB case and is called out only to avoid
  confusion when reading Audi material.

---

## 4. Sound — A2B bus, and the head unit is the "master"

MIB3 changed the audio architecture in two ways relative to MIB2:

1. **Inter-module audio moved from MOST to A2B** (Automotive Audio Bus). A2B is a serial
   audio bus (up to ~50 Mbit/s) on a twisted pair — the same kind of wiring used for
   FlexRay. It carries the audio streams **and** control, self-diagnosis and status. It
   links the head unit (`5F`), the external amplifier, and the connectivity module (`0075`)
   where present. **[MQB general]**
2. **The head unit (`5F`) became the sound "master"** — on MIB2 the external amplifier was
   the master. The head unit now provides up to **8 channels**, up to **180 W** of internal
   amplification, the sound processor, volume control, and the noise-handling features
   **VNC** (vehicle-noise compensation) and **ANC** (active noise cancellation).
   **[MQB general]**

**External amplifier.** On premium sound systems, part of the speaker set is driven by an
**external amplifier** and the rest by the head unit's internal amplification. The external
amplifier is a separate ECU on the A2B bus, with the head unit as master. The SSP names it
`J525` **[Audi]**; in the VW diagnosis scheme the external sound amplifier is address
**`47`** (**[deduced]** mapping — verify in the autoscan whether address `47` is present
on a given premium-sound unit).

---

## 5. Displays (MMI) — CAN control, LVDS image

The MMI touchscreen(s) are **controlled by the head unit over CAN**, while the **image is
delivered over LVDS**. In two-display layouts, a single LVDS "super frame" is split between
the two MMI screens. The displays are diagnosed under the head unit's address, **`5F`**.
**[MQB general]**

> **[Audi] engineering/service menu.** The SSP documents that Audi units expose a hidden
> engineering/service menu reached by a multi-touch-and-hold gesture on the MMI screen. The
> VW equivalent may differ and is not documented here, and the SSP does not enumerate what
> the menu can change. Treat any such menu as **read-only until its effects are understood**
> — this document does not use it as a procedure.

---

## 6. Image transport — the five paths, and the Audi-vs-LG cluster split

The SSP states that MIB3 moves image over **five possible transports**:
**MOST, LVDS, composite video, Ethernet, and USB.** A given signal uses one of them; the
choice varies by version and by vendor.

**What the head unit RECEIVES (image in):**

| Source | Transport | Notes |
|---|---|---|
| Rear-view camera | **Composite video** *or* **Ethernet** | depends on version |
| Surround / top-view ("360") camera | **LVDS** *or* **Ethernet** | depends on version |
| Driver-assist camera feed | **LVDS** | — |
| Optical-disc player (where fitted) | internal | — |
| Connected phones | **USB** | CarPlay / Android Auto video |

**What the head unit SENDS (image out):**

| Destination | Transport | Notes |
|---|---|---|
| MMI display(s) | **LVDS** | §5 |
| Instrument cluster / digital cockpit | **see below** | the brand split |
| Head-up display (where fitted) | — | — |

### The navigation-map-to-cluster path — the key brand/vendor difference

This is the single most important architectural fork for this project, and it is where
**[Audi]** and **[LG]** diverge:

- **[Audi]** Per the SSP, the **navigation map is sent to the instrument cluster
  (`J285` / `17`) over LVDS**, and the menu/list imagery and album art travel over **MOST**.
  This is the Audi routing.
- **[LG]** On the LG-built MOI3 unit, the map video for the digital cockpit ("Active Info
  Display") is carried as **video-over-IP on Automotive Ethernet** — an **IP multicast**
  stream (group `ff14::2:4`), observed in read-only firmware analysis. There is **no MOST**
  on this path.

These two statements do **not** contradict each other: the SSP itself lists **Ethernet** as
one of the five legal image paths, and simply documents the **Audi** route as LVDS/MOST.
The exact route is a per-brand / per-vendor choice, and on the LG MQB unit it is Ethernet.

➡️ **The full cluster-map pipeline** — the multicast stream parameters, the codec, the
coding/adaptation field that enables it, and the three "layers" of map-on-cluster (native
map / phone turn-by-turn / full app map) — is documented separately in
**[cluster-map.md](cluster-map.md)**. That document is the core of this repository; this
page only places the path in the overall architecture.

---

## 7. Radio — three tuners

MIB3 radio uses **three tuners at once**. **[MQB general]**

- **"High" configuration.** Two tuners (call them 1a and 1b) both receive the *selected*
  station and combine their signals (phase diversity → better reception). The **third
  tuner runs a continuous background scan** of AM/FM: it receives TMC (traffic data),
  watches for traffic announcements, and **keeps the station list permanently up to date
  without interrupting the station on air**. AM is received on 1a, with the background
  tuner refreshing the AM list behind the scenes.
- **"Basic" configuration.** Tuner 1 is universal (FM/AM/DAB), tuner 2 is FM for diversity,
  and tuner 3 handles RDS and the station list.

This background-scan design is why, on a real car, **the station list updates by itself**.

- **DAB (where broadcast).** Three tuners plus an **MRC** combining scheme (which
  reconstructs bits lost on one tuner from the others) plus a background tuner for the list.
  **Region variation:** the SSP separates **ECE** and **NAR** regions; in **markets without
  DAB the radio is FM/AM only**. DAB channel separation is **coded per country**, and a
  wrong country code degrades reception. **[MQB general / region variation]**
- **Metadata:** radio text / radio-text-plus, traffic announcements, **station logos**
  (via DAB or online), and **album art via acoustic fingerprinting** (Gracenote — needs a
  mobile-data connection).
- **[Audi] online / "hybrid" radio** (Audi connect): internet streams of broadcast
  stations with seamless FM↔online handover, arriving through the `J949` connectivity
  module; depends on a data plan/licence, so it does not apply to units without that module.

---

## 8. Manufacturer identification — read it from the firmware train

"MIB3" is a **commercial name**, not one piece of hardware. Several Tier-1 vendors build it
on different silicon, and the group's cars ship whichever the model got. The cleanest way
to tell the vendor **without opening the unit** is the **firmware train name**.

`MOI3` marks the MIB3 generation/level ("Option Infotainment"). The rest of the train name
encodes region, brand/platform, and — by its *pattern* — the vendor:

| Vendor | Train pattern | Examples |
|---|---|---|
| **LG** | `MOI3_<region>_VW_R####L` — **`R`** before the number, **`L`** suffix | `MOI3_EU_VW_R8030L`, `…R8410L`, `…R8830L` |
| **Preh** | `MOI3_<region>_VWMQB_P####P` — note **`VWMQB`**, **`P`** prefix/suffix (release builds come as `MOI3_SEQREL_EU_VWMQB_P####P`) | `…P27800P`, `…P33000P` |

Reading the parts:

- **`MOI3`** — generation + level (MIB3 "Option Infotainment").
- **`EU` / `NAR` / …** — region.
- **`VW` vs `VWMQB`** — brand/platform string; in practice `VW` goes with the LG pattern and
  `VWMQB` with the Preh pattern.
- **Leading letter of the code (`R` / `P`)** — `P` is a production build; **all LG MOI3
  trains begin with `R`**, so for this family `R` does not mean "pre-series".
- **`####`** — the version, which lines up with a short **MU** number (e.g. `R8030L` ↔
  MU 0803).
- **Suffix `L` / `P`** — part of the vendor pattern above.

> A list of **compatible part numbers** inside a firmware package is **compatibility
> information, not a hardware identifier**; the same number can appear in packages from
> different vendors. The **train pattern** (and, definitively, the package's variant code —
> see below) is the real discriminator.

The **definitive** vendor read is the firmware package's **variant code**, of the form
`FM3-<size>-*-<region>-VW-MQB-<vendor>` (e.g. `FM3-S-*-EU-VW-MQB-LG`), found in the update
manifest. That, and the SoC/OS per vendor, are covered in **[firmware.md](firmware.md)**.
In short:

| Vendor | SoC | OS | Confidence |
|---|---|---|---|
| **LG** | Qualcomm Snapdragon | **Automotive Grade Linux (AGL)**, kernel 4.4.x | High — read from firmware |
| **Preh / JOYNEXT** | Renesas R-Car M3 | Yocto Linux, kernel 4.14.x | High — public teardown |
| **Aptiv** (ex-Delphi) | Texas Instruments | Linux | Medium — public teardown |

(Both of the main VW-group vendors run a **Linux** userland — this is the generational break
from MIB2's QNX. The two are *not* interchangeable platforms, though: different SoC,
kernel, and boot chain.)

---

## 9. Privacy and data collection (high level)

The SSP documents a **privacy / data-collection** design, summarised here at the
architectural level only:

- A **"Privacy mode"** settings area lets the user control what is transmitted; the in-car
  setting is treated as the "master" over remote settings. There is a **data-collection**
  category (anonymous / pseudonymous).
- There are **two data collectors** in the car: **one in the head unit (`5F`)** and **one
  in the gateway (`J533` / `19`)**. They gather and transmit data. This confirms, from an
  official source, that the **head unit itself collects and transmits data** (the surface
  that forensic tooling inspects).
- **[Audi] Personalization** (Audi connect): cloud-stored user profiles (carried between
  cars, PIN-protected) — licence-dependent, so it does not apply to units without that
  service.

---

## 10. Protections (scope note)

For completeness, this platform enforces several protections that bear on architecture:

- **Component protection** — selected modules are bound to the vehicle they were registered
  in; on MQB the binding is managed by the **gateway (`19`)** and resolved online with the
  factory tooling. A module that is new or moved from another car reports component
  protection active and runs in a limited state until it is bound.
- **Firmware integrity** — the unit uses **signed firmware, a verified boot chain, and a
  verified read-only root filesystem**. These are described (not as a how-to) in
  **[firmware.md](firmware.md)**.

**This repository documents *that* these protections exist and *where* they sit, because
that is part of understanding how the system is built. It does not provide, and will not
accept, instructions for defeating them.** Running modified or unsigned firmware on the unit
is out of scope.

---

## Sources

- **SSP 970893**, "Third-generation modular infotainment matrix", Audi of America, 2021 —
  public via the US NHTSA:
  <https://static.nhtsa.gov/odi/tsbs/2021/MC-10189329-0001.pdf>. Primary source for §§1–7
  and §9. It is an **Audi** document; Audi-specific items are tagged **[Audi]** above.
- **Public security teardown of the Preh variant** — used only for the Renesas SoC, the
  Linux userland, and the 100BASE-T1 Ethernet link to the cluster/telematics:
  PCAutomotive, "Over the Air — Compromise of Modern Volkswagen Group Vehicles"
  (Black Hat Europe 2024),
  <https://i.blackhat.com/EU-24/Presentations/EU-24-Parnishchev-OverTheAirVW.pdf>, and the
  associated PCA advisory. The exploitation content of those sources is **not** used here.
- **LG SoC/OS and the Ethernet multicast map path** — from this project's own **read-only
  firmware analysis**; detail and the cluster pipeline are in
  [firmware.md](firmware.md) and [cluster-map.md](cluster-map.md).

*Independent research. Not affiliated with or endorsed by Volkswagen AG, Audi AG or LG
Electronics. Descriptive use of trademarks only.*
