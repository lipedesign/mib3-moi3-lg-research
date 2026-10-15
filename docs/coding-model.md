# The coding / adaptation model (MIB3 MOI3 / LG)

How the head unit's HMI is **configured** on the LG-built MOI3 variant of VW Group MIB3:
the "sets" of coding, adaptation, car-function and identification data the HMI reads at
start-up, the field layout inside them, and what the important fields mean. This is the
**legitimate configuration mechanism** — the same data a diagnostic tool (ODIS / VCDS /
OBDeleven) reads and writes — and it is **distinct from modifying firmware**.

It is the companion to [cluster-map.md](cluster-map.md): the coding behind the native map on
the cluster is traced there and summarised in §9 below.

> **Scope reminder.** Reading and writing these configuration values involves **no**
> protection: coding/adaptation is what the car's own diagnostics are for. This document is
> about that mechanism. Where a capability is **not** reachable by coding (it is compiled
> into signed firmware), we say so and stop — we do **not** describe how to defeat Secure
> Boot, dm-verity or VW's firmware signing. Nothing here was written to a vehicle; the field
> model was read from our own copies of firmware and exercised in a PC sandbox, read-only.
> No vehicle-identifying data (VIN, serial, part numbers of a specific car, autoscan dumps)
> appears here — only the platform structure.

## Confidence tags

| Tag | Meaning |
|---|---|
| `[disasm]` | Verified by static reverse engineering (disassembly / decompilation of platform binaries). |
| `[runtime]` | Verified by running real platform code in a PC sandbox (our own copies, no vehicle, read-only firmware image). |
| `[fw-static]` | Read directly from firmware files — the HMI's configuration tables and label files — not executed. |
| `[inferred]` | Author inference from the above. |
| `[field]` | Community / forum / field report. |

Firmware referenced throughout: trains `MOI3_EU_VW_R8030L` and neighbours, i.e. MU **0803**,
**0841**, **0883** (the copy most of the static/runtime work was done on). HMI software seen:
`H40.272.151` (MU 0803) and `H40.272.304` in the **OISmall** variant (MU 0883). The field
tables in the `oilarge` and `oismall` HMI variants were checked and are identical. `[fw-static]`

---

## 1. What "coding" is on this platform

In the car, the head unit (**`5F`**, see [architecture.md](architecture.md) §1) does not hold
its own configuration. At every start-up its microcontroller hands the HMI a handful of
**sets** of bytes — **coding**, **adaptation**, **car-function adaptation**, **records** and
**identification** — and the HMI interprets them against fixed internal tables. The HMI only
accepts these sets **once, at start-up**, before its first read of persisted state; changing
them with the HMI already running has no effect. `[runtime]`

Each set is addressed by a **DID** (UDS *Data Identifier*). The platform's persistence service
logs the arrival of each set as an **"RDID"** (*Read Data by Identifier*) event — the same
`ReadDataByIdentifier` service a diagnostic tester uses over the wire. So the bytes the HMI
consumes at boot and the bytes ODIS / VCDS / OBDeleven read and write over UDS are **the same
objects**; `0600` is the coding, `F187` is the spare-part number, and so on. `[fw-static]`

That is the whole point: **coding/adaptation is the configuration surface**. A tester changes
these values; the module stores them; at the next start the HMI reads them and behaves
accordingly. This is categorically different from changing the firmware image — the code that
*interprets* these values. Everything in this document is the former.

### 1.1 The five access classes

The HMI exposes five access classes, each a thin wrapper the firmware calls `…Impl`, each with
its own internal namespace tag: `[fw-static]`

| Access class | Namespace | DID(s) | Bytes | Fields | Carries |
|---|---|---|---|---:|---|
| **coding** | `0x2000000` | `0600` | 25 | 24 | brand, car class, generation, body style, region, drivetrain, steering side, vehicle-type / power-stage flags, MOST wake-up. HMI interprets **only the first 7 bytes**. |
| **adaptation** | `0x1000000` | 39 DIDs, `04FE`…`2442` | 397 | 400 | per-function enable/parameter switches: navigation (`0558`), map-to-cluster (`0B5A`), sound (`0550`), connectivity / App-Connect (`0555`), tuner (`0559`, `09EB`), rear camera (`0505`, `055C`), skin & disclaimers (`0556`), device/bus list (`0B1D`), and more. |
| **car-function adaptation** | `0x1000100` | `0B1B` | 256 | 336 | the car menus the HMI offers ("ACC: Menu display", "Clock: Menu display", …) and their operating conditions. |
| **records** | `0x4000000` | `7209`, `720D` (+ `0001`, `0002`) | 144 + 34 | 158 | stored numeric parameter tables (rating / weighting values, timings). Mostly **unlabelled** (no `.swap`, §5). |
| **identification** | `0x3000000` | `F187`, `F189`, `F18C`, `F191`, `F1A3` | 49 | 5 | read-only identity: spare-part number, software version, serial, hardware number and version. |

A few DIDs **look** like records but are not served from these tables: the language records
`720B` / `720C` and `F1B1` are not in the HMI's table — the HMI asks the persistence service
for them over the bus. `720B` in particular makes the persistence service **rewrite** the
HMI's stored language. Language is therefore **not** coding on this platform; it comes from
persisted state. `[runtime]`

---

## 2. How the HMI reads a set

A field is located by **(DID, byte, bit, width)** and decoded by three primitives in the
HMI's access layer — `getValue` (integer), `getBoolean` (single bit) and `getByteArray`
(text / blob). `[fw-static]`

- **Byte** counts from 0. Below, byte offsets are given **within that DID's own data block** —
  the per-channel view a diagnostic tool shows for that channel. (The field table also stores a
  second offset into the concatenated access-class block; it is an internal detail.)
- **Bit** is where the field starts, with **bit 0 = least significant**.
- A field up to 8 bits wide lives inside a single byte. A field of 16, 24 or 32 bits starts at
  bit 0 and is **big-endian** (most-significant byte first).
- The value is an integer that must fit the field width.

**Worked example** — the "Map Switch" bit (the one traced in §9):

```
DID 0B5A (adaptation), byte 7, bit 1, width 1, type bool
            └─ ID_ABILITY_SWITCH_NAV_MAPS — "Map Switch" — 0 = no, 1 = yes
```

`getBoolean(0B5A, byte 7, bit 1)` returns that one bit. In a tester this is the
`Dashboard_Display_Configuration` channel, sub-field "Map Switch".

---

## 3. The field table and what its columns mean

The platform's field model was dumped to a tab-separated table of **1 188 fields** (one row per
field). The columns, and where each comes from, are: `[fw-static]`

| Column | Meaning | Source in firmware |
|---|---|---|
| `conjunto` | access class (coding / adaptation / car-function / records / identification) | which `…Impl` the field belongs to |
| `namespace` | the class's internal namespace tag (table above) | the `…Impl` object |
| `did_hex` / `did_dec` | the DID | the key list each class passes to the access layer |
| `byte_no_did` | byte offset **within the DID's own block** (per-channel view) | the class field table |
| `byte_no_conjunto` | byte offset within the concatenated access-class block (internal) | the class field table |
| `bit_inicial` | start bit, 0 = LSB | the class field table |
| `bits` | field width in bits | the class field table |
| `tipo` | `int`, `bool`, or `bytes` | the class field table |
| `padrao` | the value the HMI assumes when no data is supplied | the class field table |
| `valores_aceitos` | accepted range (e.g. `0..12`) | the class field table |
| `id` | internal field id | the class field table |
| `constante` | the `ID_…` constant name | the config interfaces `CodingMib3`, `AdaptationMib3`, `CarFuncAdap`, `Records`, `Identification` |
| `requisito` | the ODIS requirement tag (`Dia_Cod_*`, `Dia_Anp_*`, `Dia_Ident_*`) | the `.swap` label files |
| `descricao` | the human label (often German, as a tester shows it) | the `.swap` label files |
| `significado_dos_valores` | per-value enum meaning | the `.swap` label files |
| `fonte` | provenance of the row | — |

### 3.1 Why the labels line up with a diagnostic tool

The byte/bit/width/default/range come from the HMI's own **class tables** (`CodingImpl`,
`AdaptationImpl`, `CarFuncAdapImpl`, `RecordsImpl`, `IdentificationImpl`, in the `oilarge`
HMI build) exactly as the access layer walks them. `[fw-static]`

The **human labels and value meanings** — the `Dia_Cod_* / Dia_Anp_*` requirement tags, the
German field names, the `0x1 = aktiviert` enum text — come from four label files shipped in the
HMI config (`coding.swap`, `adaptation.swap`, `carfunction.swap`, `identification.swap`). These
are exactly what the HMI's own on-board diagnostic screen renders, and they are the same labels
a diagnostic tool prints. **Records have no `.swap`**, which is why most record fields are
numeric-only with no friendly name. `[fw-static]`

The HMI's assumed-default set was cross-checked: for each of the 1 188 fields, the documented
default equals what the HMI uses when it receives no data at all. `[runtime]`

---

## 4. The main coding block — DID `0600`

`0600` is the classic VW "long coding" vehicle descriptor. It is 25 bytes, but the HMI
**interprets only the first 7**; the rest are reserved/ignored by this module. `[fw-static]`
Byte / bit are within `0600`.

| Field (`ID_…`) | Byte / bit | Type | Range | ODIS tag | Meaning |
|---|---|---|---|---|---|
| `ID_BRAND` | 0 / 4–7 | int | 0..12 | `Dia_Cod_230` | manufacturer brand (**2 = VW**) |
| `ID_CAR_CLASS` | 0 / 0–3 | int | 1..9 | `Dia_Cod_244` | vehicle class (**3 = "A"**) |
| `ID_CAR_GENERATION` | 1 / 4–7 | int | 1..9, 16 | `Dia_Cod_246` | generation |
| `ID_CAR_BODYSTYLE` | 1 / 0–3 | int | 0..9 | `Dia_Cod_248` | body style (**1 = Stufenheck / notchback**) |
| `ID_CAR_BODYSTYLE_ADDITION` | 2 / 4–7 | int | 0..9 | `Dia_Cod_250` | body-style qualifier |
| `ID_CAR_PRODUCTION_REGION` | 2 / 0–3 | int | 0 | `Dia_Cod_252` | production region |
| `ID_CAR_TYPE_OF_DRIVETRAIN` | 3 / 4–7 | int | 0..6 | `Dia_Cod_254` | drivetrain |
| `ID_SUSPENSION_ARM_SIDE` | 3 / 3 | bool | — | `Dia_Cod_256` | steering/suspension side |
| `ID_MOST` / `ID_MOST_OPTICAL_WAKEUP` / `ID_EECL_WAKEUP` | 4 / 7, 6, 5 | bool | — | `Dia_Cod_277/280/283` | MOST presence & wake-up |
| `ID_VEHICLE_TYPE_*` (allroad, short/long wheelbase, pickup, jacking-head, customer-specific) | 5 / 7…2 | bool | — | `Dia_Cod_290…305` | vehicle-type flags |
| `ID_VEHICLE_TYPE_POWER_STAGE_1` | 5 / 1 | bool | — | `Dia_Cod_308` | power-stage flag 1 |
| `ID_VEHICLE_TYPE_POWER_STAGE_2` | 5 / 0 | bool | — | `Dia_Cod_311` | power-stage flag 2 |
| `ID_VEHICLE_TYPE_POWER_STAGE_3` / `…_MODIFIED_DRIVE_CONCEPT` / `…_PA1` / `…_PA2` | 6 / 7…4 | bool | — | `Dia_Cod_314…323` | more vehicle-type flags |

The value-name constants for brand, class, generation and body style (`BRAND_*`,
`CAR_CLASS_*`, `CAR_GENERATION_*`, `CAR_BODYSTYLE_*`) live in the `CodingMib3` interface.
`[fw-static]`

**Observed effects** (sandbox, generic — no specific car): `[runtime]`
- Setting brand/class/generation/body-style changes the **vehicle silhouette** the HMI draws
  (e.g. hatchback → notchback figure).
- `ID_VEHICLE_TYPE_POWER_STAGE_1 = 1` switches the HMI **accent colour** from blue to red
  (`#FF2832`) across all screens — a pure HMI appearance consequence of one coding bit.

---

## 5. Adaptation groups that matter

Adaptation is 39 DIDs. Below are the ones that shape what the user sees. Byte / bit are within
each DID. Fields without an ODIS tag are present in the HMI table but unlabelled in
`adaptation.swap`.

### 5.1 `0B5A` — Dashboard_Display_Configuration (the map-to-cluster group)

This is the group that governs what the head unit **pushes to the instrument cluster** and
other picture transport. `[fw-static]`

| Field (`ID_…`) | Byte / bit | Type | ODIS tag | Values |
|---|---|---|---|---|
| `ID_NAVIGATION_MAP_TRANSMISSION_MODE` | 0 | int (0..3, 5) | `Dia_Anp_4142` | `0x00 = off`, `0x01 = IP streaming`, `0x02 = MOST streaming`, `0x03 = LVDS` — most likely what actually turns the cluster map on (§9, inferred) |
| `ID_NAVIGATION_MAP_COMPRESSION_MODE` | 1 | int (0..2) | — | codec selection (**H.264**) |
| `ID_NAVIGATION_MAP_RESOLUTION` | 2 | int (0) | — | requested cluster-map resolution index |
| `ID_ABILITY_SWITCH_NAV_MAPS` ("Map Switch") | **7 / 1** | bool | `Dia_Anp_4182` | `0 = no`, `1 = yes` — tells the cluster the map can be switched on/off; **not** the map enable (§9) |
| `ID_DYNAMIC_ICON` | 7 / 2–3 | int (0..2) | `Dia_Anp_4178` | `0 = off`, `1 = BAP`, `2 = MOST` |
| `ID_COVERART` | 6 | int (0..2) | — | cover-art transmission to cluster |
| `ID_STATIONART` | 7 / 6–7 | int (0..2) | — | station-art transmission |
| `ID_CALL_PICTURE` | 7 / 4–5 | int (0..2) | — | call-picture transmission |

The **crossroad-detail** view is a **separate adaptation DID, `0B26`**, laid out like the map
fields: `ID_CROSSROAD_DETAIL_MAP_TRANSMISSION_MODE` (byte 0, same `0..3, 5` enum),
`…_COMPRESSION_MODE` (byte 1), `…_RESOLUTION` (byte 2). `[fw-static]`

On an in-service MOI3 LG unit this group commonly reads as the IP-video configuration
(transmission = IP streaming, compression = H.264, Map Switch = yes), matching the firmware's
own H.264-over-IP cluster stream. A Preh / JOYNEXT unit reads the same except Map Switch = no,
and still shows the native map on the cluster. Details and field reports are in
[cluster-map.md](cluster-map.md) §4.2. `[field]`

### 5.2 `0558` — navigation function

| Field (`ID_…`) | Byte / bit | Type | ODIS tag | Values |
|---|---|---|---|---|
| `ID_NAVIGATION_SYSTEM` | 1 / 7 | bool | `Dia_Anp_5153` | `0 = not activated`, `1 = activated` ("Funktion Navigation") |
| `ID_COUNTRY_NAVIGATION` | 0 | int (0..18) | `Dia_Anp_5133` | map region: `01 = EU`, `02 = NAR`, `03 = MSA`, … `18 = China FAW-VW` |
| `ID_VZA` / `ID_VZO` | 1 / 5, 4 | bool | `Dia_Anp_5159 / 5162` | traffic-sign-display coding |
| `ID_LGI` | 1 / 0 | bool | `Dia_Anp_5174` | local guidance info |
| `ID_TPEG_AVAILABILITY` / `ID_ADVANCED_RANGE_DISPLAY` | 1 / 1, 2 | bool | — | TPEG traffic / range display |

**Observed effect** (sandbox): `ID_NAVIGATION_SYSTEM = 1` makes the HMI build the
**"Navigation"** tile and respond to the NAV hard key ("Starting navigation…"). `[runtime]`
This is the function **existing in the menu** — not the same thing as the map reaching the
cluster (§9).

### 5.3 `0559` — radio / tuner adaptation

| Field (`ID_…`) | Byte / bit | Type | ODIS tag | Values |
|---|---|---|---|---|
| `ID_BAND_SETTINGS_FM_TUNER` | 1 / 4–7 | int (0..8) | `Dia_Anp_5225` | `0 = none`, `1 = EU`, `2 = NAR`, `3 = JP`, `4 = KOR`, `5 = China`, `6 = Brasil`, `7 = KOR`, `8 = JP` |
| `ID_BAND_SETTINGS_AM_TUNER` | 1 / 0–3 | int (0..4) | `Dia_Anp_5235` | `0 = none`, `1 = EU`, `2 = NAR`, `3 = JP` |
| `ID_BAND_SETTINGS_DAB_TUNER_BAND_1` | 2 / 4–7 | int (0..4) | `Dia_Anp_5241` | `0 = off`, `1 = EU Band III`, … |
| `ID_AM_DISABLE` | 4 / 7 | bool | `Dia_Anp_5277` | `1 = AM blocked` |
| `ID_AF` | 3 / 4 | bool | `Dia_Anp_5262` | `0 = AF persistent`, `1 = AF temporary` |
| `ID_RADIO_TEXT_PLUS` | 4 / 0 | bool | `Dia_Anp_5298` | RadioText+ on/off |
| `ID_ONLINE_RADIO` / `ID_HYBRID_RADIO_*` | 6 | int | — | online / hybrid-radio modes |

**Observed effect** (sandbox): the FM band-setting value is directly visible in manual tuning —
`6 = Brasil` gives a **76.1–107.9 MHz** sweep in 0.2 MHz steps, versus the EU setting's
**87.5–108.0 MHz** in 0.1 MHz steps. A single adaptation integer reshapes the tuner UI.
`[runtime]`

A second, very large tuner DID, **`09EB`**, is a bank of ~150 booleans (`ID_FB_TUNER_*`,
`ID_FB_MEDIA_*`, `ID_FB_NAV_*`, `ID_FB_PHONE_*`, `ID_FB_CAR_*`, `ID_FB_MISC_*`), each
`0 = not blocked / 1 = blocked` — a per-feature block list. `[fw-static]`

### 5.4 Connectivity, phone & Bluetooth — `0555`, `0553`, `0B33`

`0555` is the connectivity group (WLAN, App-Connect, CarPlay, Android Auto, MirrorLink,
Bluetooth): `[fw-static]`

| Field (`ID_…`) | Byte / bit | Type | ODIS tag | Meaning |
|---|---|---|---|---|
| `ID_WLAN_MODULE` | 0 / 7 | bool | `Dia_Anp_4354` | WLAN function |
| `ID_WIFI_HOTSPOT_2_4GHZ` / `_5GHZ` | 0 / 4, 3 | bool | `Dia_Anp_4361 / 4364` | Wi-Fi hotspot bands |
| `ID_APP_CONNECT_OVER_WIFI` | 0 / 0 | bool | `Dia_Anp_4373` | wireless App-Connect |
| `ID_ONLINE_MEDIA` | 1 / 7 | bool | `Dia_Anp_4376` | online media |
| `ID_GOOGLE_ANDROID_AUTO` | 1 / 4–5 | int (0..2) | `Dia_Anp_4382` | Android Auto |
| `ID_APPLE_DIO` | 1 / 2–3 | int (0..2) | — | **CarPlay** ("Apple device I/O") |
| `ID_MIRROR_LINK` | 1 / 0–1 | int (0..2) | — | MirrorLink |
| `ID_BLUETOOTH_AVAILABILITY` | 4 / 3–4 | int (0..2) | `Dia_Anp_4428` | `0 = none`, `1 = without Low Energy`, `2 = with Low Energy` |
| `ID_BLUETOOTH_PHONE` / `_AUDIO` / `_MULTIMEDIA_FUNCTIONALITY` | 5 / 3, 2, 4 | bool | `Dia_Anp_6196 / 6199 / 6193` | BT phone / audio / multimedia |

**Observed effect** (sandbox): enabling the CarPlay field (`ID_APPLE_DIO`) brings up the
head unit's **smartphone-integration** stack — an "App-Connect" tile appears, the call screen's
end-call control becomes live, and start-up takes a few seconds longer while that stack loads.
`[runtime]`

Related groups:
- **`0553`** — telephony detail: `ID_SUPPORT_SECOND_PHONE` (byte 0, `0..3`), `ID_PHONE_NAD`
  (byte 1 / bit 6, "Telefonmodul NAD", `Dia_Anp_4264`), `ID_LTE_MODUL`, eSIM / SMS / messaging
  support bits. `[fw-static]`
- **`0B33`** — a single field, `ID_BLUETOOTH_DEACTIVATION` (byte 0, `0, 1`), whose **default is
  `1`** — one of the few fields whose assumed default is "on". `[fw-static]`
- **`2442`** — `ID_APPLE_DEVICE` / `ID_OTHER_DEVICES` ("verbaut / nicht verbaut", device
  present or not). `[fw-static]`

### 5.5 Other notable adaptation groups

| DID | Purpose | Example fields |
|---|---|---|
| `0550` | sound system | `ID_SOUND_SYSTEM` (byte 0, `0..4`: none / internal / internal-individual / external MOST / external Ethernet, `Dia_Anp_143`) |
| `0505` / `055C` | rear-view camera | `ID_REAR_VIEW_LOW_RVC` (`0505` byte 0, `0 = off / 1 = active / 2 = active with legal screen`), `ID_RVC_VIDEO_INPUT` (`055C`) |
| `0556` | skin & disclaimers | `ID_SKINNING`, `ID_SCREENINGS` (vehicle image), `ID_HMI_LEGAL_DISCLAIMER`, `ID_HMI_INITIAL_DISCLAIMER` |
| `0557` | media functions | `ID_USB` (`0..3`: off / charge / full / iPod), `ID_AUX_IN`, `ID_IMPORT_MEDIA_DATA`, `ID_RIPPING_MEDIA_DATA`, Gracenote cover-art bits |
| `0B48` / `0B49` | startup-screen sticker / media sales-country | `ID_STARTUP_SCREEN_STICKER_HMI`, `ID_MEDIA_COUNTRY_CODE_MULTIMEDIA` |
| `055B` | update over the air | `ID_UOTA` |

---

## 6. The device / bus list — `0B1D` (adaptation)

`0B1D` is the module's view of **which vehicle devices exist and where they sit on the bus** —
the MQB-style "installation + bus assignment" list. Each device gets a pair of fields in one
byte: `[fw-static]`

- **Availability** (bits 6–7, `0x00 = not activated`, `0x01 = activated`) — e.g.
  `ID_PARKING_ASSIST_0X0A` (byte 10), `ID_CENTRAL_LOCKING_0X0D` (byte 13),
  `ID_EXTERIEUR_LIGHT_0X09` (byte 9), `ID_WIPER_COMFORT_0X0C` (byte 12),
  `ID_NIGHT_VISION_0X18` (byte 24), `ID_UNIT_MASTER_0X13` (byte 19).
- **Bus assignment** (bits 0–5, `0..36`) — which bus the device is reached on:
  `01 = CAN Powertrain`, `02 = CAN Infotainment`, `03 = CAN Extended`, `04 = CAN Chassis`,
  `05 = Flexray`, `07..1A = Cluster 1..20`, `20 = CAN MIB`, `21 = CAN Kombi`,
  `23 = CAN Comfort`, `24 = Ethernet`, etc.

The device hex ids (`0x09`, `0x0A`, `0x0D`, …) are the BAP function ids
(see [glossary.md](glossary.md): BAP).

---

## 7. Car menus — DID `0B1B` (car-function adaptation)

`0B1B` (its own access class, namespace `0x1000100`) decides **which car menus the HMI builds**
and under what conditions they can be operated. For each device the HMI reads a block of bits:
`[fw-static]`

- **`… : Menu display`** (bit 7) — `0x00 = not activated`, `0x01 = activated`. This is the bit
  that makes the menu appear at all.
- Operating-condition bits around it: `… CLAMP_15_OFF` (operable with ignition off),
  `… AFTER_DISCLAIMER` (only after a shown disclaimer), `… STANDSTILL` / `STILLSTAND` (idle
  only), `… OVER_THRESHOLD_HIGH` (operable above a speed threshold).

Examples: `ID_MENU_AUXILIARY_CLIMATIZATION` (byte 46 / bit 7, `Dia_Anp_1243`),
`ID_MENU_CONNECTION_MANAGER` (byte 60 / bit 7, `Dia_Anp_1453`), `ID_MENU_STATISTIC`
(byte 62 / bit 7, `Dia_Anp_1483`).

**Bench finding** `[runtime]`: a menu's existence is decided by **`0B1B` alone**. The HMI reads
the "Menu display" bit, and only then subscribes to that menu's option service (the options
themselves come from the car over the bus, not from coding). The device-availability list
`0B1D` (§6) is **not** consulted to decide whether the menu is created — toggling a device in
`0B1D` with and without its `0B1B` bit produced the same screen. The two lists answer different
questions: `0B1B` = "offer this car menu"; `0B1D` = "this device exists on bus X".

---

## 8. Records and identification (read briefly)

- **Records** (`0x4000000`, `7209` / `720D`) are stored numeric parameter tables the HMI reads
  as RDIDs like everything else — e.g. `7209` holds a `ID_VERSION_VALUE` plus driving-style
  **rating / weighting** values (`ID_RATING_*`, `ID_WEIGHTING_*`, each `int 0..10`). Because
  records have **no `.swap`**, most fields are numeric with no friendly label. `[fw-static]`
- **Identification** (`0x3000000`) is read-only module identity: `ID_VW_SPARE_PART_NUMBER`
  (`F187`), `ID_VW_APPLICATION_SOFTWARE_VERSION_NUMBER` (`F189`), `ID_ECU_SERIAL_NUMBER`
  (`F18C`), `ID_VW_ECU_HARDWARE_NUMBER` (`F191`), `ID_VW_ECU_HARDWARE_VERSION_NUMBER` (`F1A3`).
  These are text; a specific unit's values are **vehicle-identifying and deliberately omitted**
  here. If a byte falls outside the printable set (`0`–`z`, space, hyphen, dot) the HMI's
  "System information" screen shows `DIAG ERROR` for that line. `[fw-static]`

---

## 9. The native cluster map — this **is** coding

The **native cluster map** (the head unit's own map, H.264-encoded and streamed to the cluster
over IP — see [cluster-map.md](cluster-map.md) §1, §5 layer 1) is governed by the `0B5A`
**adaptation**, `Dashboard_Display_Configuration`. Turning it on or off is ordinary coding:
no firmware change, no protection involved.

**Which field turns it on** is most likely `ID_NAVIGATION_MAP_TRANSMISSION_MODE` (byte 0):
its values are the transports themselves (off / IP / MOST / LVDS), and both units read so far
show the map on the cluster with it at `IP_streaming`. The link from byte 0 to the stream code
has not been traced in the disassembly yet. `[inferred]`

**What the "Map Switch" bit does** is traced end-to-end. From
[cluster-map.md](cluster-map.md) §4.1:

```
Dashboard_Display_Configuration  0B5A  byte 7 / bit 1   (ID_ABILITY_SWITCH_NAV_MAPS, "switchMapSupported")
        │   single store in the nav library
        ▼
   member 0x19a
        │
        ▼
   setBapEntity
        ▼
   FunctionSupportActivationDeactivationOfTheVideoNavigationSupported   (BAP FsgSetup  Df44 Bit0)
```

- `[disasm]`: there is **exactly one** instruction that writes member `0x19a`, fed from `0B5A`
  byte 7 / bit 1 on the `nvmDataChanged` path — the mapping is unique.
- `[runtime]`: running the real nav-client library in the PC sandbox (no vehicle), forcing
  `0x19a = 1` makes the `Df44 Bit0` getter return 1, and `0x19a = 0` makes it return 0. The
  chain holds **both statically and at runtime**.

The flag tells the cluster that the head unit supports **switching the video map on and off**
(moving it between the radio screen and the cluster). It is not the map itself: a Preh /
JOYNEXT unit shows the native map on the cluster with this bit at `no`
([issue #1](https://github.com/lipedesign/mib3-moi3-lg-research/issues/1)). `[disasm]` +
`[field]`

> **Correction (2026-10-15).** Earlier versions of this section and of
> [cluster-map.md](cluster-map.md) §4.1 called this bit "the native-map enable bit". The full
> BAP name and the field report above show it is the on/off-switch capability instead.

The other values the software reads along the way (`_iaa_function_coding`,
`_pso_function_coding`, `SemidynamicRouteGuidanceAvailable`, `FunctionList`) are likewise
ordinary function-coding / function-list items. `[disasm]`

---

## 10. The explicit contrast — the full phone-app map is **not** coding

Putting the **phone app's own map** (Waze / Google Maps / Apple Maps rendered as video) on the
cluster is a **different problem with no coding path**. Two independent architectural facts, both
detailed in [cluster-map.md](cluster-map.md) §6: `[disasm]`

1. **A hardcoded gate.** `RouteGuidanceByMobileDevice` (BAP `Df47`, pushed via `setBapEntity`)
   is set to **`false` in the binary** on the builds analysed, and it is **not exposed as an
   adaptation channel** anywhere — there is no `0B5A`-style value a tester can flip. The decision
   is compiled in.
2. **The stream source is the native engine, not the phone.** The surface bound to the cluster
   multicast is the native renderer, not the CarPlay/Android-Auto surface; even a flipped gate
   would still carry the native map.

Both are properties of **signed firmware**. The unit runs **Qualcomm Secure Boot + dm-verity +
VW firmware signing** (described, not defeated — see [cluster-map.md](cluster-map.md) §6.1 and
[open-questions.md](open-questions.md)). This document maps **where** the gate sits so the
coding-vs-firmware line is clear; it gives **no patch, offset or step** to change it, and that
is explicitly out of scope. The distinction to carry away:

| Capability | Mechanism | Coding? |
|---|---|---|
| Native cluster map (§9) | `0B5A` adaptation — most likely the transport field (byte 0, `IP_streaming`); the byte 7 / bit 1 Map Switch → `Df44 Bit0` only advertises on/off switching | **Yes** — adaptation (Map Switch chain proven static + runtime; byte 0 as the enable is inferred) |
| Phone turn-by-turn arrows on cluster | factory CarPlay behaviour (varies by app/market) | No — behaviour, not a coding item |
| Full phone-app map on cluster | hardcoded `RouteGuidanceByMobileDevice` gate in firmware | **No** — no adaptation channel exists |

(Feature **licensing** — SWaP / FoD function codes and FEC generation — is a separate
mechanism and is **out of scope** here; see [glossary.md](glossary.md) and
[open-questions.md](open-questions.md) §3.)

---

## 11. Cross-references & sources

- [cluster-map.md](cluster-map.md) — the map-to-cluster pipeline; the `0B5A` enable chain (§4),
  the three map "layers" (§5), and why the full app map is firmware-gated (§6).
- [architecture.md](architecture.md) — the head unit (`5F`), buses, and the image-transport
  paths that the `0B5A` transmission-mode field selects between.
- [open-questions.md](open-questions.md) — gaps worth filling: `0B5A` field values across MU
  versions and markets, which coding items really exist on MOI3 LG, and version drift of the
  field layout.
- [glossary.md](glossary.md) — BAP, DID, coding/adaptation, SWaP/FoD/FEC, the protection layers.

**How you can help** (sanitized, read-only): read your unit's `Dashboard_Display_Configuration`
(`0B5A`) and the connectivity / navigation groups with ODIS / VCDS / OBDeleven and report the
values **with no VIN, serial or part number** — model, market and firmware train / MU only. That
builds the cross-unit table the project needs.

Public source for the surrounding architecture: **SSP 970893, "Third-generation modular
infotainment matrix"** (VW/Audi Self-Study Programme, public via US NHTSA) —
<https://static.nhtsa.gov/odi/tsbs/2021/MC-10189329-0001.pdf>.
