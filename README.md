# MIB3 "MOI3" (LG) — Research Notes

Open, community-oriented reverse-engineering and research notes on the **Volkswagen Group MIB3** infotainment platform — specifically the **"MOI3" variant built by LG** (firmware trains `MOI3_EU_VW_R####L`, e.g. MU 08xx).

The central question these notes grew around: **how does the native navigation map get onto the instrument cluster (the "Active Info Display" / digital cockpit), and what is actually possible on this platform?** Along the way they document the platform architecture and the firmware structure.

> Status: living notes, compiled from reading public documents and analyzing firmware read-only. Contributions welcome — see [How to contribute](#how-to-contribute).

## Why this exists

Public information about MIB3 is overwhelmingly about the **Harman MHI2 / MHI2Q** platform (MOST bus, QNX). The **LG-built MOI3** variant is a different machine — **Qualcomm Snapdragon + Automotive Grade Linux (AGL)**, with an **Ethernet** transport — and it is almost undocumented in public. These notes try to be the reference that didn't exist, and to invite people with other cars, markets and firmware versions to verify and extend them.

## Scope / non-scope (please read)

- ✅ **In scope:** platform architecture, firmware package structure, the cluster-map video pipeline, the coding/adaptation fields involved, and the public tooling landscape.
- ❌ **Not a protection-bypass guide.** We document *that* the unit uses **Qualcomm Secure Boot + dm-verity + VW firmware signing**, and how the system is put together — but this repository is deliberately **not** a guide to defeating those protections. Running modified, unsigned firmware on the unit is out of scope.
- ❌ **No proprietary material.** No firmware binaries, no confidential manuals, no vehicle-identifying data. Everything here is our own analysis plus links to public sources.
- ℹ️ **Nothing here was written to any vehicle.** These are read-only analysis and documentation notes.

## Platform covered

| | |
|---|---|
| Generation | MIB3 ("Option Infotainment", "OI") |
| Hardware vendor | **LG** — firmware trains ending in `…R…L`; the Preh variant uses `…P…` (see architecture doc) |
| SoC / OS | Qualcomm Snapdragon + Automotive Grade Linux (AGL) |
| Transport to cluster | **Ethernet** (IP multicast), not MOST |
| Example firmware trains | `MOI3_EU_VW_R8030L` (MU 0803), `…R8410L` (MU 0841), `…R8830L` (MU 0883) |

## Contents

- **[docs/architecture.md](docs/architecture.md)** — platform architecture: diagnosis addresses, buses, GPS source, sound (A2B), radio, displays and the image-transport paths.
- **[docs/cluster-map.md](docs/cluster-map.md)** — the **navigation-map-to-cluster pipeline**: the multicast video stream, the codec, the adaptation that enables it, and the three "layers" of map-on-cluster (native / phone turn-by-turn / full app map). The core of this repo.
- **[docs/coding-model.md](docs/coding-model.md)** — the **coding / adaptation model**: how the HMI reads coding "sets" (DIDs), the field layout, the important fields, and the one coding bit that lights the native cluster map.
- **[docs/firmware.md](docs/firmware.md)** — firmware package structure: the AGL userland, the Qualcomm boot chain, the protection layers, and the update-container format.
- **[docs/bench-emulation.md](docs/bench-emulation.md)** — methodology for studying the HMI **read-only on a PC** (observing its behaviour without the hardware).
- **[docs/landscape.md](docs/landscape.md)** — related public projects and tools, and what does / doesn't carry over from the Harman MHI2 scene.
- **[docs/glossary.md](docs/glossary.md)** — the terms used across these notes.
- **[docs/open-questions.md](docs/open-questions.md)** — what we still don't know, and where contributors can help.

## How to contribute

This is most useful if people with **different MOI3 LG models, markets and firmware versions** confirm these findings and add their own. Especially helpful:

- Confirm or correct the behaviours described here **on your car** — tell us the model, market and firmware train / MU version.
- Add adaptation-field readings (**sanitized**: no VIN, no serial numbers).
- Fix anything that's wrong, with evidence.

Open an issue or a pull request. Please keep every contribution **free of vehicle-identifying data** and **free of protection-bypass instructions**, matching the scope above.

## Sources

The main public source referenced throughout is the Audi/VW Self-Study Programme **SSP 970893, "Third-generation modular infotainment matrix"**, which is public via the US NHTSA. Other sources are linked inline in each document.

## Disclaimer

Independent research. Not affiliated with, authorized by, or endorsed by Volkswagen AG, Audi AG, LG Electronics or Apple Inc. "CarPlay" is a trademark of Apple Inc.; "MIB", "MQB" and "Active Info Display" are used descriptively to refer to the systems under study. Information is provided as-is, with no warranty. Coding or firmware changes to a vehicle can affect safety, warranty and regulatory compliance, and can render modules inoperative — act at your own risk.

## License

Documentation: **CC BY 4.0** (proposed). Any code or snippets: **MIT**.
