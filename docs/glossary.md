# Glossary

Short definitions of the terms used across these notes. Brand-specific terms are marked.

- **MIB** — *Modularer Infotainment-Baukasten*, VW Group's modular infotainment family.
- **MIB3** — the third MIB generation. "OI" (*Option Infotainment*) is the internal tier naming seen in the software.
- **MOI3** — the MIB3 software line covered here; firmware trains named `MOI3_<region>_VW_R####L`. In this repo, "MOI3 (LG)" means the **LG-built** variant.
- **Vendor variant** — MIB3 head units come from different suppliers; the firmware-train letter tells them apart: `…R…L` = **LG**, `…P…` = **Preh**.
- **Train / MU** — a *train* is the software line (e.g. `MOI3_EU_VW_R8030L`); **MU** is the version within it (e.g. MU 0803).
- **OISmall** — the HMI variant/skin for the smaller displays (e.g. `OISmall2_EU`).
- **Head unit (`5F`)** — the central infotainment ECU; diagnosis address `5F` (`005F`). Radio, screens, sound and media are functions of it.
- **Cluster** — the instrument panel behind the wheel. Marketing names: **Active Info Display** (VW), **Virtual Cockpit** (Audi), "digital cockpit".
- **AGL** — *Automotive Grade Linux*; the Linux distribution the LG unit runs (here AGL 3.0 "chinook", a Yocto/Poky build). <https://www.automotivelinux.org/>
- **Yocto / Poky** — the embedded-Linux build system AGL is based on.
- **DSI** — the display/service interface layer the HMI uses (display-management services).
- **VIWI / RSI** — the internal REST/JSON service bus the HMI talks to (a service registry plus service handlers).
- **BAP** — *Bedien- und Anzeigeprotokoll*, VW's operation/display protocol carried over the vehicle bus; functions are addressed by BAP IDs.
- **A2B** — *Automotive Audio Bus*; the MIB3 inter-module audio transport (replaces MOST for audio).
- **MOST** — *Media Oriented Systems Transport*; the optical bus used by older/Harman units — **not** the LG MOI3 cluster path.
- **LVDS** — differential video link used for some displays.
- **Coding / adaptation** — configuration values the firmware reads and respects, changed with a diagnostic tool (ODIS/VCDS/OBDeleven). Distinct from modifying firmware.
- **SWaP / FoD** — *Software as a Product* / *Function on Demand*; the mechanism that licenses features by code. (Enabling it is out of scope here.)
- **FEC** — *Feature Enabling Code*, tied to SWaP/FoD licensing.
- **ZDC** — a dataset container format applied to modules (e.g. sound datasets).
- **MQB** — *Modularer Querbaukasten*, VW Group's transverse-engine vehicle platform.
- **Secure Boot / dm-verity / FDS signing** — the three protection layers: Qualcomm's signed boot chain, the verified read-only root partition, and VW's production firmware signature. Described in [firmware.md](firmware.md); defeating them is **out of scope**.
- **`.mbn`** — Qualcomm's signed boot-chain image format.
- **Weston / Wayland** — the compositor / display protocol the HMI renders through.
- **`ff14::2:4`** — the IPv6 multicast group the head unit streams the cluster map video to; see [cluster-map.md](cluster-map.md).
