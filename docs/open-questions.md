# Open questions & how you can help

These notes are incomplete by design — a lot of what's here was found on a small number of units and firmware versions. The most valuable thing the community can add is **verification across different models, markets and firmware trains**, plus filling the gaps below.

Please keep every contribution **sanitized** (no VIN, plate, location, serial numbers or raw autoscan dumps) and **within scope** (see the bottom of this page).

## 1. Verify on your car — no special tools

You only need your phone and the car:

- **Layer 2 — phone turn-by-turn on the cluster.** With CarPlay (or Android Auto) connected and navigating, do the **turn-by-turn directions appear on the instrument cluster**? Reports conflict: some say it shows with **Apple Maps** but not **Google Maps**, others say Google Maps via Android Auto works too. Tell us: **model, market, firmware train / MU version, head-unit part-number family, and which app** — and what you saw. *So far:* a Preh / JOYNEXT unit (MU 0330, iPhone) shows arrows from **Apple Maps and Google Maps** ([issue #1](https://github.com/lipedesign/mib3-moi3-lg-research/issues/1)).
- **Native map on the cluster.** Does rotating to "Navigation" put the **built-in** map on the cluster on your unit? Some markets ship without built-in navigation.

## 2. Verify with a diagnostic tool — read-only, sanitized

OBDeleven / VCDS / ODIS, reading only:

- **`Dashboard_Display_Configuration` (channel `0xB5A`) field values** on your unit — `navigation_map_transmission_mode`, `navigation_map_compression_mode`, `navigation_map_resolution`, `ability_switch_nav_maps`. Collecting these across MU versions and markets would build a real table. *So far:* two units, an LG (MU 0803) and a Preh / JOYNEXT (MU 0330), read the same except `ability_switch_nav_maps` — see [cluster-map.md](cluster-map.md) §4.2.
- **`ability_switch_nav_maps = no` on an LG unit.** Does the native map still reach the cluster? And with `no`, does the centre screen keep its own map while the cluster shows one, or can the map not be moved? This would confirm the correction in [cluster-map.md](cluster-map.md) §4.1.
- **Coding landscape for MOI3 LG**: which one-click / adaptation items actually exist (cluster themes, cluster layouts, "wireless CarPlay" on the head unit) and which don't. Help separate "coding" from "needs firmware change".

## 3. RE / understanding — analysis only, within scope

- **SWaP / FoD function IDs on the `5F`.** The unit exposes a licensing table of function codes; mapping **what each code enables** would clarify what is feature-gated vs firmware-gated.
- **The cluster-side receiver.** The head unit is the *producer* of the map video; the *consumer* (the cluster ECU that subscribes to the multicast group and decodes it) is a different ECU we don't have images of. How does it subscribe/decode? What are the real stream parameters (resolution, fps, SPS/PPS)?
- **Version drift.** Do MU 0803 / 0841 / 0883 / … differ in the pipeline, the gate, or the adaptation layout?
- **Preh vs LG.** Does the Preh / JOYNEXT variant (`MOI3_EU_VWMQB_P…`) use the same IP-video pipeline to the cluster, or a different transport? *Partly answered:* a `P33000P` unit is coded `IP_streaming` / `H264` like the LG one ([issue #1](https://github.com/lipedesign/mib3-moi3-lg-research/issues/1)). Still open: the multicast group and port, and whether its navigation libraries match the LG ones.
- **What actually enables the map.** Trace `navigation_map_transmission_mode` (`0B5A` byte 0) to the `ipStreamingEnabled` flag in `libMIBNavSDClientLib_CarIF` — the step that would turn the `[inferred]` enable in [cluster-map.md](cluster-map.md) §4.1 into a verified one.
- **Maps.** Region/compatibility rules (e.g. South-America vs Europe map data) and how they couple to FEC/SWaP.
- **Track the runtime PoC.** `daChillout` & `Meich` published a runtime proof-of-concept of the phone-app map on the MOI3 cluster — a custom RSI service run from `tmpfs`, **not a firmware modification** — and said it will be open-sourced ([drive2.ru](https://www.drive2.ru/l/745563717583775343/)). Worth following, reproducing within scope, and comparing to the pipeline in [cluster-map.md](cluster-map.md). The **display path** is the interesting, in-scope part; the **code-execution / access** method is the author's and stays out of scope here.

## Explicitly NOT in scope

Anything that requires **defeating Secure Boot, dm-verity or VW's firmware signing**. Understanding *how the system is built* — including that these protections exist and where the software gate sits — is in scope. **Bypassing them is not**, and pull requests that do that will be declined. This keeps the project useful as a reference without becoming a piracy/brick guide.
