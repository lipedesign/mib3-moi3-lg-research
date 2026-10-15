# Firmware & Update-Package Structure — MIB3 "MOI3" (LG)

Reverse-engineering / research notes on the **structure** of the firmware and the
update package for the Volkswagen Group **MIB3 "MOI3" (LG-built)** infotainment head
unit. This is **static, read-only analysis** of publicly-distributed update containers.
Nothing here was ever written to a vehicle, and nothing here requires a vehicle.

> **Scope.** This document describes *how the package and the operating system are put
> together*. It deliberately does **not** describe how to defeat the unit's cryptographic
> protections — see [Protection layers](#protection-layers-architectural-fact). For the
> navigation-map-to-cluster pipeline see [`cluster-map.md`](cluster-map.md); for the
> wider platform (buses, ECUs, displays) see [`architecture.md`](architecture.md).

### Confidence convention

The source notes are careful about what was proven versus guessed, and this document
keeps that distinction:

- **[verified]** — confirmed by reading the package/partition directly, or by running the
  real binaries in a sandbox.
- **[static]** — read from files, strings or kernel config, but not exercised.
- **[inference]** — reasoned from the evidence; plausible, not confirmed.

---

## 1. Package identity

The platform ships as a **SWUP / UMC** software-update container (file extension `.7z`,
~6.2–6.8 GB). The packages analysed here are the **LG/VW** builds that are structurally
representative of the family. Identity is carried in three places:

| Field | Value / pattern | Notes |
|---|---|---|
| Train | `MOI3_<region>_VW_R<nnnn>L` | The trailing **`L`** marks the **LG** hardware vendor (the Preh/Renesas variant uses `…P…`). `MOI3` = the LG MIB3 line. |
| Example trains | `…R8030L` (MU 0803), `…R8410L` (MU 0841), `…R8830L` (MU 0883) | Successive builds of the same platform; MU 0883 was built 2024-04-26, MU 0841 was built 2023-03-14. **[verified]** |
| Region token | `EU`, `RW`, `W2`, `US` | Market/region of the package. |
| MU version | `MU<nnnn>` | The marketing/update version number (e.g. 0803, 0841, 0883). Higher = newer. |
| Trailing token | VW infotainment software part number (the `919360` family) | Identifies the head-unit software part; **omitted here as a specific value** per the repo's sanitization rules. |

The update layer identifies compatible units by a **variant tag** inside the manifest.
All compatibility variants end in **`…-VW-MQB-LG`**, e.g. `FM3-<size>-<…>-<region>-VW-MQB-LG`.
Decoding that tag **[static]**:

- **`FM3`** = the MIB3 firmware family (the voice/audio-family tag in this package).
- **`<size>`** = screen/trim size class (`S` / `SM` seen).
- **`<region>`** = `EU` / `RW` / `W2` / `US`.
- **`VW-MQB-LG`** = Volkswagen, MQB platform, **LG** build.

> A given car's *exact* train may be one that is not freely mirrored; the builds analysed
> here are the nearest LG/VW trains. For understanding **structure**, they are equivalent —
> the container layout, OS and boot chain are identical across MU 0841 and MU 0883, and a
> diff of the two showed **no structural change** to the OS, boot chain or cluster
> transport. **[verified]**

---

## 2. The update container (UMC / SWUP)

Opened read-only with `bsdtar` (libarchive reads/extracts the `.7z` directly — no
proprietary 7-zip needed). **[verified]** The container has two top-level trees:

```
<package>.7z
├── Meta/          manifests + checksums + signature  (tool banner: "UMC-V0.7.1")
└── Data/          the actual component images
```

### Manifests

- JSON **`.mnf`** manifests. The top manifest **`main.mnf`** carries `Release`,
  `MUVersion`, `Variant`, `BlockedTrains` and `Includes`.
- SHA-256 checksums for the manifest set live in **`main.mnf.cks`** (and per-MU
  `main.mnf_<mu>_<n>.cks`).
- Each component in `Data/` additionally has its own `HWIndex` (indices **21 / 31**
  observed), an `InstallerFile`, and its own SHA-256. **[static]**

### Signature — VW FDS "Serie" (authenticity gate)

`Meta/` contains **`main.mnf_cks_S.sig`**, a VW-format INI file: **[verified]**

- `TypeOfSignature = "Serie"` (i.e. **production** signing, not a developer/engineering key),
- an `FDS-Ident` field and a build `Date`,
- it signs the manifest-checksum file (`main.mnf_<mu>_<n>.cks`),
- an `[ECU-Signature]` block with `signature01…13` (16-byte values).

This is VW's **FDS** signing scheme: the head unit will accept the package only if the
signature verifies against the key provisioned in the ECU. It is one of three protection
layers (see §6).

### Component map (`Data/`)

| Component | Approx. size | What it is |
|---|---|---|
| **CPU** | ~1.1 GB | The main application processor image: the **AGL/Linux rootfs**, wrapped as an **Android/Qualcomm OTA** (`update.zip`). See §3. |
| **HMI** | ~1.8 GB | The VW human-machine interface assets, split `Small` / `Large` × `EU` / `NAR`. The small-screen EU build is **`OISmall_EU`** (internal variant `MQB_VW_LOW_EU`). See §5. |
| **SPEECH** | ~3.5 GB | Text-to-speech voices, one `.zip` per language (incl. pt-BR). |
| **POST_SCRIPT** | ~84 MB | Post-install shell scripts + `RSDB.sqlite` (a broadcast-station database). See §7. |
| **MCU** | ~22 MB | `mcu_image.bin` — firmware for the separate microcontroller (raw binary, `0xFF` padding). |
| **Mercury** | ~15 MB | The audio/radio DSP — **NXP SAF** (`SAF400x` / `SAF777x`; DAB, FM, HD Radio). A tuner/DSP distinct from the SoC's own audio DSP. |
| **ETHERNET_SWITCH** | ~74 KB | Firmware for the internal Ethernet switch (consistent with the Ethernet in-car transport). |

---

## 3. The CPU image: AGL Linux wrapped as an Android/Qualcomm OTA

This is the point most likely to be misread. The CPU component uses the **Android recovery
OTA format** as a *wrapper*, but the system it installs is **AGL/Linux**, not Android.
**[verified]**

`Data/.../update.zip` contains:

- **`boot.img`** — an Android boot image, **page size 4096, with no ramdisk**
  (*System-as-Root*). Inside: an uncompressed arm64 `Image` kernel (see §4) and the kernel
  command line
  `console=ttyHSL0,115200,n8 androidboot.hardware=qcom … rootfstype=ext4 root=/dev/dm-0 firmware_class.path=/firmware/image/ … ro`.
  `root=/dev/dm-0` → the root is a **device-mapper (dm-verity) target** over a read-only
  ext4 `system` partition. **[verified]**
- **`system.new.dat` + `system.transfer.list`** — the `system` partition in Android
  sparse-dat form. The transfer list is a single contiguous `new` range from block 0, so
  `system.new.dat` **is** the ext4 image byte-for-byte (no squashfs/UBI/sparse). **[verified]**
- **`firmware-update/*.mbn`** — the **Qualcomm boot chain** (all signed; see §6/§4).
- **`META-INF/com/google/android/updater-script`** — an Edify installer that does
  `block_image_update` on `system` (verifying `range_sha1`), writes `boot.img`, and calls
  `msm.boot_update("main")` to flash the Qualcomm partitions. **[static]**

There is **no Android userland** on the installed system: no `/system`, `/app`,
`/framework`, no `init.rc`, no binder. The kernel even builds **without binder**
(`CONFIG_ION`/`CONFIG_ASHMEM`/`CONFIG_SYNC` are present — Android memory/sync primitives —
but that is all). **[verified]** The "Android" is purely Qualcomm's boot/flash
infrastructure sitting beneath a Linux (AGL) userland. This matches the 2020 reports that
VW/Audi moved MIB3 to Automotive Grade Linux.

> **Exynos rumour: discarded.** The SoC is **Qualcomm**, not Samsung Exynos (see §4).

---

## 4. Operating system & SoC

### 4.1 Operating system

Read from the reconstructed `system` partition: **[verified]**

| Item | Value |
|---|---|
| Distro | **Automotive Grade Linux** — `/etc/os-release`: `NAME="Automotive Grade Linux"`, `VERSION="3.0.0+snapshot-20230314 (chinook)"`, `ID="poky-agl"` → **AGL 3.0 "chinook", built with Yocto/Poky**. |
| Build stamp | `/etc/version` = `202303141433` (snapshot 2023-03-14; the MU 0883 build is 2024-04-26). |
| Kernel | **Linux 4.4.115-perf**, `SMP PREEMPT`, arm64, gcc 5.3.0, built in KST (→ LG Korea). |
| init | **systemd** (v232). |
| Mandatory Access Control | **SMACK** (Smack is the default LSM; `CONFIG_SECURITY_SMACK=y`), with **Cynara** for policy (Tizen/AGL-style). |
| Root filesystem | Standard **FHS** (`usr bin lib lib64 sbin etc var run srv home mnt proc sys tmp dev media`), plus `firmware persist dsp radio smack cache data`, plus the vendor dirs **`/lge`** and **`/vw`**. |

**Vendor directories** (the LG + VW split): **[verified]**

- **`/lge/`** (LG Electronics): `app_ro` (read-only app partition), `app_rw`, `bt`,
  `secure`, **`webengine`** (a web rendering engine), `speech`, `log`, `user_storage`,
  `hmi`.
- **`/vw/`** — VW's HMI/config tree (e.g. `/vw/hmi/res`, `/vw/hmi/config`); shipped
  separately as the HMI component and mounted at `/vw` (the mount point is empty in the
  `system` image itself).

**Base package versions** (from the image's `opkg` database, 547 packages): **[verified]**

| Package | Version | | Package | Version |
|---|---|---|---|---|
| glibc | **2.23** | | Wayland | 1.9.0 |
| libstdc++ (gcc) | 5.3.0 | | Weston | **1.9.0** |
| systemd / udev | **232** | | libdrm | 2.4.62 |
| dbus | 1.10.6 | | pixman | 0.32.8 |
| cynara | 0.11.0 | | openssl | 1.0.2o |
| security-manager | 1.0.2 | | busybox | 1.24.1 |
| protobuf | 3.1.0 | | GPU driver pkg | `adreno 1.0+git` |

**Binary census** of the `system` partition: **1,673 ELF** files — 1,380 AArch64 shared
libraries, 270 AArch64 executables, 13 kernel modules + 1 relocatable object, and **9
32-bit Hexagon ELF** (audio-DSP blobs under `/usr/lib/rfsa/adsp`, plus the GPU
`a530_zap.*` in `/lib/firmware`). **There is no 32-bit ARM code in userspace** — every
application-processor binary is AArch64. **[verified]**

Notable kernel modules (`.ko`): `DWC_ETH_QOS.ko` (Synopsys DesignWare **Ethernet QoS** MAC —
consistent with the AVB/TSN in-car transport), `mlan.ko` / `moal.ko` (Marvell WLAN) and
`bcmdhd.ko` (Broadcom WLAN), the Tuxera exFAT/NTFS drivers, `msm_11ad_proxy.ko`, `rdbg.ko`,
and netfilter. **[static]**

### 4.2 SoC & boot chain

The application SoC is a **Qualcomm Snapdragon, automotive 820A class**: **[verified from
boot.img DTB + kernel IKCONFIG]**

- Device trees in `boot.img` declare `model = "Qualcomm Technologies, Inc. MSM 8996pro AUTO
  ADP"`, `compatible = "qcom,msm8996-adp","qcom,msm8996","qcom,adp"`, `qcom,msm-id = <0x13c
  0x10001>` (three DTBs differing only in `board-id`).
- Kernel config: `CONFIG_ARCH_MSM8996=y`, `CONFIG_MACH_APQ8096_MIB3=y`,
  `CONFIG_MACH_MIB3_MQB=y`, `CONFIG_LGE_CARPLAY=y`; the Yocto machine is `8x96auto`.
- **GPU: Adreno 530 (a5xx)** via the DRM `msm` driver (`CONFIG_DRM_MSM=y`; note
  `CONFIG_QCOM_KGSL` is **off** — userspace reaches the GPU through DRM `/dev/dri/card0`,
  even though a device-tree node is still named `qcom,kgsl-3d0`).
- **Hexagon ADSP** for Qualcomm audio (the 32-bit Hexagon blobs above). This is separate
  from the **NXP SAF** radio/DAB DSP shipped as the *Mercury* component, and separate again
  from the standalone **MCU**.
- So the SoC family is MSM8996/APQ8096 "pro" automotive. The precise `msm-id 0x13c` →
  marketing-name mapping was not re-confirmed against a Qualcomm source in these notes.
  **[inference]**

The **Qualcomm secure boot chain** ships in `firmware-update/` as signed **`.mbn`**
images. The set present: **[static]**

`xbl.elf` (eXtensible BootLoader / SBL) · `tz.mbn` (TrustZone) · `hyp.mbn` (hypervisor) ·
`rpm.mbn` (Resource Power Manager) · `emmc_appsboot.mbn` (aboot / Little Kernel) ·
`keymaster.mbn` · `cmnlib.mbn` / `cmnlib64.mbn` · `devcfg.mbn` · `pmic.mbn` ·
`NON-HLOS.mbn` (modem/baseband) · `adspso.mbn` (ADSP) · `BTFM.mbn` (Bluetooth/FM) ·
`lksecapp.mbn` · `sec.mbn` · **`lge_custom.mbn`** (LG's own customization blob).

The standard Qualcomm order is PBL → XBL/SBL → TZ/HYP/RPM → aboot (LK) → `boot.img`
(Linux kernel) → dm-verity-mounted `system`. Each stage verifies the next; the presence of
`keymaster.mbn` / `lksecapp.mbn` / `sec.mbn` indicates hardware-backed key storage in
TrustZone. (The exact chaining order is standard-Qualcomm, **[inference]** from the
component set + the `root=/dev/dm-0` cmdline.)

---

## 5. The HMI

The running VW HMI is **not** a native C++ GUI — it is a **Java application on OpenJDK 8**
(Zulu, aarch64), inherited from the MIB2 HMI and now re-hosted on AGL instead of QNX/J9:
**[verified]**

- `/lge/hmi` (~95 MB) holds `jvm/` (the bundled OpenJDK, ~69 MB) and `jar/` (9 jars,
  ~24 MB). The large jar is **`MIBHMI-DSI.jar`** (~23 MB, **13,884 classes** under
  `com/lge/mib3oi`, compiled with OpenJDK `1.8.0_222`).
- Rendering is **OpenGL ES3 from Java** (`jopengl_es3.jar` + the JNI bridge
  `libjopengl_es3.so`, which is NEEDED-linked straight to `libEGL_adreno.so` /
  `libGLESv2_adreno.so`). The compositor is **Weston/Wayland 1.9.0** (`ivi-shell`,
  Qualcomm SDE DRM backend `libsdedrm.so`; outputs `DSI-1` and `HDMI-A-1`). **[verified]**
- The HMI talks to the platform over **~50 DSI (Device Service Interface) services**, each
  carrying **protobuf** messages (`protobuf-java-3.1.0`). The DSI service surface (the
  HMI's view of the car) includes: `addressbook, androidauto2, bap, bluetooth, browser,
  caraircondition, carcomfort, cardriverassistance, cardrivingcharacteristics, careco,
  carhybrid, carkombi, carlight, carparkingsystem, carplay2, carseat, carstatistics,
  carvehiclestates, connectedradio, diagnose, displaymanagement, exlap, filebrowser,
  generalvehiclestates, infotainmentrecorder, keypanel, kombipictureserver, media,
  messaging, mirrorlink, networking, nfc, persistence, phonemanager, powermanagement,
  radio, search, smartphoneintegration, speechrec, swap, system, telephoneng, tts, upnp,
  wirelesscharging`. **[static]**

**On "OISmall over a web engine":** the HMI *assets* ship as **`OISmall_EU`** (variant
`MQB_VW_LOW_EU`; a `Large`/`OILarge` build and `EU`/`NAR` market splits also exist), and
the rootfs additionally carries **`/lge/webengine`** plus web-app plumbing (`webappmgr`,
`webapps.sh`). So both a Java/OpenGL HMI framework **and** a web rendering engine are
present; the precise division of rendering between them is only partly mapped. **[inference]**

---

## 6. Protection layers (architectural fact)

Three independent cryptographic protections sit between a modified image and a running
system. This repository documents them as **facts of the architecture** — not as a target:

1. **Qualcomm Secure Boot.** The `.mbn` boot-chain images (§4.2) are signed and verified
   stage-by-stage; hardware-backed keys live in TrustZone (`keymaster`, `lksecapp`,
   `sec`). **[static]**
2. **dm-verity on `system`.** The AGL root is a read-only ext4 mounted through
   device-mapper verity (`root=/dev/dm-0`), so any block change is detected at read time.
   `CONFIG_DM_VERITY=y` in the kernel. **[verified]**
3. **VW FDS "Serie" signature on the whole package** (§2). The ECU accepts an update only
   if the FDS signature verifies against its provisioned key. **[verified]**

**Consequence.** Running modified code on this unit would have to clear all three layers at
once. That is exactly what makes the LG MIB3 far more closed than the older MIB2, and
**there is no public method to do it for the LG/Qualcomm variant.**

> **Out of scope.** Defeating or circumventing any of these protections — secure boot,
> dm-verity, or FDS signing — is a security bypass and is **not described in this
> repository**. This document explains only the structure.

For context, published *defensive* security research exists on a **different** MIB3
variant (the **Preh / Renesas R-Car** build, not the LG/Qualcomm one), presented publicly
and reported fixed by the vendor; its findings target the Renesas boot chain and do **not**
transfer to the Qualcomm chain described here. It is cited only to show that the platform's
security has been studied in the open, and to keep the two hardware families distinct — see
the public advisory linked in [Sources](#sources). No offensive use of it is in scope.

---

## 7. How the package was read (read-only methodology)

Everything below is **analysis of files on a PC**. Nothing was mounted read-write, nothing
was written back into any package, and **nothing was ever written to a vehicle.** **[verified]**

- **Container:** `bsdtar` (libarchive) lists and extracts the `.7z` read-only.
- **`system` partition:** reconstructed from `system.new.dat` + `system.transfer.list` with
  a small hand-written `sdat2img` equivalent → a valid **ext4 image (~3.07 GB,
  3,069,050,880 bytes)**. Read with **`debugfs`** (e2fsprogs), and separately loop-mounted
  **`-o loop,ro,noload`** inside a Linux container so the backing file cannot be altered.
- **`boot.img`:** parsed for the Android boot header, kernel `Image`, embedded kernel
  config (IKCONFIG) and the appended device trees — all read-only.
- **Binaries:** inspected with standard tooling (`readelf`/`nm`/`objdump`/`strings`, plus a
  decompiler) to read symbols, NEEDED libraries and strings. No binaries were modified or
  redistributed.

A separate study effort additionally *ran* some of the real AArch64 binaries natively in a
container (the SoC userland is all AArch64, so no CPU emulation is needed) to observe
start-up behaviour. That work is sandbox-only and, by construction, never touches FDS,
dm-verity or secure boot. (It is why some facts above are marked **[verified]** rather than
**[static]**.)

### The post-install scripts (`POST_SCRIPT`) — the OS seen from the installer

`post_init_main.sh` runs per-subsystem scripts in order:
`bsp, system, media, network, audio, hmi, sai, onlinecore, connectivity, speech,
dev_config, early_service, appconnect, navigation, webapps, radio, internetradio, sdars,
security`. Tells: `security.sh` applies the SMACK labels (`/etc/smack/conf/config.sh`);
`appconnect.sh` creates the phone-projection area with a dedicated SMACK owner; the
scripts carry LG author/date comments (names omitted here). `sdars` = satellite radio
(NAR market). **[static]**

---

## 8. Subsystem / component map (high level)

systemd carries **217 units**; `/lge/app_ro` holds the read-only apps. By category
**[static]**:

- **Graphics / HMI:** `weston` (Wayland compositor), `vw-hmi` (the Java HMI, §5),
  `displaymgr`, **`multi-display`**, **`rsi-display-management`**, `touch-app`, `gesture`.
- **Phone projection:** `carplay` (+ `carplay-bonjour`, `carplay-iap2d`), `androidauto`,
  `mirrorlink`, `webappmgr`. Native projection binaries live under
  `/lge/app_rw/smartlink/` with a connection manager (`sdcm`).
- **Audio / radio:** `audiomgnts`, `media-mgr`, `waveplayer`, `init_qti_audio` (Qualcomm
  audio), `alsa-*`, `internetradio`, `sdars`, `speech`, `tts`.
- **Navigation:** `simplenav` + `/lge/app_ro/navi` (`naviengine`, `navi-al`); GNSS via
  `gnssmgr-csr` / `gnssmgr-ocu`.
- **Connectivity:** `connman`, `wlanmgr` (Wi-Fi), `bluetooth` / `btapp`, `onlinecore`,
  `stunnel`, `dhcpd`.
- **In-vehicle network — `carif`:** `carifservice` (the bridge to the car bus), the
  `CarIF` libraries (e.g. `libMIBNavSDClientLib_CarIF.so`, the native-nav client, project
  `MQB_OI`), `mcu-interpreter` (talks to the MCU), `vector-sip` (**SOME/IP**, Vector
  stack), the **VIWI** services (`viwics`/`viwios`/`viwioru`/`viwiproxy`), and `gptp`
  (gPTP — AVB/TSN time sync over **Ethernet**).
- **Camera / video:** `ais_server` (Qualcomm imaging / `qcarcam`), `early-cvbs`
  (early-boot rear camera).
- **Security:** `cynara` (+ `security-manager`), `sks` (Secure Key Store), `ockp` (LG
  keymaster in TrustZone), `smartcard`, `dropbear` (SSH).
- **System / update:** `swup` + `system-update.target` (the SWUP/UMC mechanism itself),
  `tuxera-fs` (exFAT/NTFS for USB media).

`/lge/app_ro` apps: `audiomgnt, bsp, carif, early-service, esp, ethernet, graphics, hmi,
internetradio, media, navi, network, ockp, onlinecore, radio, sai, sdars, sks, speech,
tuxera, update, webappmgr`.

### The RSI / VIWI service layer (`sai/spec`)

VW's internal infotainment bus is **VIWI** ("Volkswagen Infotainment Web Interface"), a
REST/JSON-over-HTTP service bus — the vendor implementation of the public **RSI** concept
(W3C Automotive WG, 2016). The firmware carries the whole thing on-device under
`/lge/app_ro/sai/`: **[verified]**

- **`MIB_ServiceRegistryApp-00`** — the service registry (port 443; the HMI discovers
  services at `ws://127.0.0.1:443/api/v1/`).
- **`rsi-mockup`** — an LG-provided mock of services (port 7770), with test data under
  `sai/mockup/`.
- **`sai/spec/`** — **81 service specifications**.
- Command-line helpers `rsi-send` / `rsi-subscribe`.

### Graphics / multi-display & the cluster

The graphics stack (`graphics` app, `multi-display`, `rsi-display-management`, Weston SDE
backend) composes the head-unit screen and feeds the **instrument cluster**. The native
navigation map reaches the cluster as an **IP-multicast video stream** (not MOST, unlike
MHI2). The transport, codec and the coding field that gates it are documented in
[`cluster-map.md`](cluster-map.md) and are **not** repeated here. One structural fact
belongs in this doc: **the instrument cluster is a separate ECU** (VW diagnostic address
17), and **its firmware is not contained in these packages** — these packages are the head
unit (diagnostic address 5F) only. **[verified]**

---

## 9. What is not (yet) opened

Honest limits of this teardown: **[static]**

- Per-service binary internals under `/usr/bin` and most of `/lge/app_ro/*/lib`, and the
  full contents of `/lge/webengine`.
- The HMI asset bundles inside `OISmall_EU.zip` (`hmi/`, `speech/`, `preload/`, `ESD/`).
- `RSDB.sqlite` (the station database) was not opened.
- The **receiver side** of the cluster video path (what actually joins the multicast group
  and rasterises on the cluster) lives in the **separate cluster ECU**, whose firmware is
  not in these images.
- The SPEECH voices for languages other than the samples, and the internal ext4 of some
  components, were not unpacked.

See [`open-questions.md`](open-questions.md) for the running list and where contributors
can help.

---

## Sources

Independent analysis of publicly-distributed `MOI3_EU_VW_R####L` SWUP/UMC update
containers, read offline and read-only; no package contents, binaries or vehicle data are
reproduced here. Public background sources:

- **Automotive Grade Linux** — the distribution confirmed in `/etc/os-release`
  (<https://www.automotivelinux.org/>), built with the **Yocto Project / Poky**
  (<https://www.yoctoproject.org/>). VW Group's move of MIB3 to AGL was reported publicly
  in 2020.
- **W3C Automotive Working Group** — the public RSI / vehicle-information service concept
  that VIWI implements (<https://www.w3.org/auto/>).
- **VW/Audi Self-Study Programme SSP 970893**, "Third-generation modular infotainment
  matrix" — public platform background (available via the US NHTSA document library).
- **Public MIB3 security advisory (different variant):** vulnerability research on the
  **Preh / Renesas R-Car** MIB3 build, presented at a 2024 security conference and
  reported fixed by the vendor —
  <https://pcacybersecurity.com/resources/advisory/vulnerabilities-in-vw-mib3-infotainment-2>.
  Cited to distinguish the Preh/Renesas and LG/Qualcomm families; its findings do not
  transfer to the Qualcomm boot chain, and no bypass is reproduced.

> Nothing in this document is proprietary code or a vehicle record, and nothing here is a
> guide to defeating the unit's protections. It is a structural description for public
> research.
