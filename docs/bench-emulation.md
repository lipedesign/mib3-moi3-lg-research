# Bench Emulation — Studying the MOI3 (LG) HMI Read-Only on a PC

Reverse-engineering / research notes on the **methodology** of studying the Volkswagen
Group **MIB3 "MOI3" (LG-built)** human-machine interface (HMI) on an ordinary PC, as a
**read-only analysis bench**. The goal is to understand how the interface *behaves* —
start-up, screen composition, input, the coding/adaptation fields it respects — without the
head unit's hardware, and without ever touching a vehicle.

This document describes a **technique**, not a product. It is the companion to
[`firmware.md`](firmware.md) (how the package is *read* statically) and
[`cluster-map.md`](cluster-map.md) (one result the bench helped confirm). Where those
describe *what* the firmware is, this describes *how* one can observe it running in a
sandbox — and, just as importantly, where that observation stops.

> ## Read-only, and what this is not (please read first)
>
> This is the most sensitive document in the repository, so the boundaries are stated up
> front and repeated where they matter:
>
> 1. **This is a methodology, not a kit.** It describes the *approach*. It does **not**
>    distribute firmware, binaries, HMI assets or any package contents, and it is **not** a
>    "run-the-image" recipe — there are **no commands, scripts, download sources or
>    copy-the-asset steps** here. Everything assumes you already lawfully possess a
>    publicly-distributed update package and want to understand the platform.
> 2. **Running the HMI in a PC sandbox is not modifying the unit, and bypasses nothing.**
>    The firmware's own AArch64 user-space programs are executed in an isolated container on
>    the PC, reading an **untouched, read-only** copy of the package. This **does not**
>    defeat, weaken or interact with the head unit's **Qualcomm Secure Boot**, **dm-verity**
>    or **VW FDS "Serie" signing**. Those three protections guard *the unit* (see
>    [`firmware.md` §6](firmware.md#6-protection-layers-architectural-fact)); they are
>    **out of scope** and are never in the loop when code runs on a PC instead of the ECU.
> 3. **No bypass content.** There are **no patch offsets, no unlock values, no
>    signature/verity/secure-boot circumvention** in this document — none were needed, and
>    none are described. Making the HMI *start* in a sandbox is a question of providing the
>    services and devices it expects, not of breaking anything.
>
> Nothing here was ever written to a vehicle. The bench is an analysis tool.

---

## Confidence convention

Matching the rest of the repo:

- **[verified]** — observed by running the real binaries in the sandbox, or by reading the
  package/partition directly.
- **[static]** — read from files, strings, unit files or kernel config, but not exercised.
- **[inference]** — reasoned from the evidence; plausible, not confirmed.

---

## 1. Why emulate the HMI on a PC at all

The MOI3 LG HMI is a large **Java application on OpenJDK 8**, running as **OSGi bundles**,
rendering through **OpenGL ES 3 from Java** onto a **Weston/Wayland** compositor, and
talking to the rest of the car over two internal buses (DSI and VIWI/RSI). See
[`firmware.md` §5](firmware.md#5-the-hmi) for the static anatomy. Reading the class files
tells you a great deal, but it cannot tell you **what the software actually does at runtime**:
which services it blocks on, in what order screens compose, how a coding/adaptation value
flows from a datapool bit to an on-screen feature, which key codes it accepts.

A read-only bench answers those questions by **letting the firmware's own code run and
observing it** — the dynamic half of reverse engineering — while guaranteeing that:

- the firmware image on disk is **never altered** (it is mounted read-only; all writes land
  on throw-away memory, see §2);
- there is **no network** and no connection of any kind to a car;
- nothing produced is written back into any package or to any ECU.

The decisive enabling fact, from [`firmware.md` §4](firmware.md#4-operating-system--soc):
the application-processor user-space is **entirely AArch64** — there is no 32-bit ARM code.
So on an ARM64 analysis host the binaries run **natively**, with **no CPU emulation**, which
is what makes a fast edit-run-observe loop possible. **[verified]**

---

## 2. Preparing the assets for reading (read-only)

The static reading technique is documented in
[`firmware.md` §7](firmware.md#7-how-the-package-was-read-read-only-methodology); it is
summarised here only as the foundation the dynamic bench stands on. Every step is **read-only
by construction**:

```
update package (.7z)                      ── read with libarchive (bsdtar), never rewritten
  └─ CPU component (update.zip)
       ├─ system.new.dat + transfer.list  ── a single contiguous "new" range from block 0,
       │                                      so the .dat IS the ext4 image byte-for-byte;
       │                                      reconstructed to a plain ext4 file (no squashfs/UBI)
       └─ boot.img                         ── parsed for header + kernel + appended DTBs (read-only)
  └─ HMI component (asset zip)             ── the /vw HMI tree (skins, jars, config)
```

- **Container:** opened and listed with **libarchive / `bsdtar`** — libarchive reads the
  `.7z` directly, so no proprietary tooling is involved. **[verified]**
- **The `system` partition** is **plain ext4** once reconstructed. It is then read in one of
  two non-destructive ways: with **`debugfs`** (e2fsprogs), which reads an ext4 image without
  mounting it at all; or **loop-mounted read-only** inside the container (`ro,noload`), which
  enters the backing file as read-only so it **cannot be altered**. **[verified]**
- **Binaries** are inspected with standard tools (`readelf` / `nm` / `objdump` / `strings`)
  and decompilers (a native decompiler for the C/C++ daemons; a Java decompiler for the HMI
  jars). No binary is ever modified or redistributed. **[verified]**

### The writable layer is throw-away memory, not the image

The firmware expects to write to `/data`, `/persist`, `/tmp`, `/run`, `/var/lib/nav` and a
few `/dev` nodes. The bench provides those as **`tmpfs`** (RAM) mounted *over* the specific
paths, plus the handful of device nodes it needs, with `/proc` mounted. **[verified]** Two
consequences matter here:

- the **read-only firmware image is never the target of a write** — every runtime change the
  software makes lives in ephemeral RAM and vanishes when the container exits;
- stacking a writable overlay *on top of* the whole root (overlay-on-overlay) was tried and
  **does not work** on the nested-filesystem setups tested; `tmpfs` on the exact paths is the
  working approach. **[verified]** (A method note, not a protection matter.)

> **Protection line, restated.** Reading an image read-only and running its AArch64 programs
> against a RAM scratch layer is ordinary offline analysis. It does not involve, and cannot
> affect, the unit's secure boot, dm-verity or FDS signing — those live on the ECU, not in
> the file you are reading. This document gives the *technique*, not a packaged way to obtain
> or launch the proprietary image.

---

## 3. Running the HMI in an isolated environment

### 3.1 Two equivalent environments

The user-space being all-AArch64 gives two interchangeable sandboxes, both proven to reach
the same point: **[verified]**

| Environment | What it is | Why use it |
|---|---|---|
| **Native container** | an isolated Linux (ARM64) container on the analysis host, with the read-only `system` image loop-mounted and entered by `chroot`, a `tmpfs` writable layer, **no network** | fast native loop (no CPU emulation); the default for day-to-day work |
| **`-M virt` VM** | **QEMU** `-M virt` with a stock mainline ARM64 kernel, the firmware `system` as a read-only virtio disk | needed only when a *custom kernel* matters: real SMACK LSM, a framebuffer/DRM device, or an actual systemd boot |

Comparing the two was itself a finding: the HMI reaches the **same milestones and the same
stopping point in both**, so the container's missing kernel features are not what blocks it.
That told us the walls are above the kernel, in the GPU/bus/data layers (§6), not in the
choice of sandbox. **[verified]**

### 3.2 The role of systemd, and the shortcut the bench takes

On the real unit, **systemd** (v232) orchestrates ~217 units to `graphical.target`:
compositor, display daemons, the car-interface bridge, the HMI service, and so on — see
[`firmware.md` §8](firmware.md#8-subsystem--component-map-high-level). Ordering is expressed
as unit dependencies; for example the **navigation/cluster producer chain** is, by unit
files, roughly `navi-al` → `naviengine` (renders the map) → `multi-display` /
`multi-screen-server` (encodes and streams it) — the producer side described in
[`cluster-map.md` §3](cluster-map.md#3-components-and-roles). **[static]**

The bench **does not** boot the full systemd target. **[verified that it was not exercised;
the full boot remains [inference].]** Instead it **launches the HMI process directly** on the
firmware's own bundled JVM, with the firmware's own class path and resources, and brings up
*only* the specific daemons a given experiment needs. This is deliberate: it keeps each run
small, reproducible and legible, and it keeps the question "what does the HMI need in order to
progress?" in the foreground. The order that actually governs a run is therefore the HMI's
**own internal OSGi start-up**, which the bench observes end to end: **[verified]**

```
SPI bundles  →  autostart bundles  →  managed bundles  →  work units  →  ViewHandler / UI thread
```

A subtle environment detail belongs here as a method note (not a protection matter): the HMI
sizes an internal worker-thread pool from the visible CPU count, and a host with many cores
trips an "insufficient configured threads" guard, while the car's 4-core count works. The
bench pins the JVM's visible processor count to match the SoC. **[verified]**

---

## 4. Observing what the HMI draws — the OpenGL forwarder ("glhost")

The hardest-looking wall is graphics, and it is the clearest illustration of the bench
philosophy: **intercept at a clean API boundary and forward, rather than fake or patch.**

The HMI's JNI graphics bridge (`libjopengl_es3.so`) is `NEEDED`-linked straight to the
Qualcomm Adreno driver sonames `libEGL_adreno.so` / `libGLESv2_adreno.so`, which on the real
unit talk to the GPU through DRM. On a PC there is no such GPU. The technique:

```
HMI JVM  (chroot, firmware glibc 2.23)                  analysis host (modern glibc)
  libjopengl_es3.so
      ├─ libGLESv2_adreno.so  = a forwarding shim ── local socket ──►  "glhost" process
      └─ libEGL_adreno.so     = a thin shim                              │  runs each GL call on
                                                                         │  Mesa llvmpipe (software
                                                                         │  OpenGL ES, no GPU)
                                                                         ├─ captures each frame offscreen
                                                                         └─ streams frames to a PC window
```

Key points of the method: **[verified]**

- The two Adreno sonames are occupied by a **shim that does not draw**. It serialises each
  EGL/GL call and sends it over a local socket to an out-of-process **glhost**, which executes
  it on **Mesa's `llvmpipe`** software rasteriser (OpenGL ES by CPU, no GPU), renders to an
  **offscreen pbuffer**, and captures each buffer swap as a frame.
- **Why out-of-process:** the shim must load *inside* the HMI's JVM, so it is linked against
  the **firmware's own glibc (2.23)**; Mesa needs a newer glibc. Keeping Mesa in a *separate*
  process bridged by a socket avoids mixing two C libraries in one address space — the mistake
  that defeated earlier in-process attempts. The two glibcs never meet.
- The shim exports exactly the symbol surface the real driver exports (OpenGL ES 3.2 + EGL
  1.4), verified against the bridge's import list, so the firmware loads it without noticing
  the substitution. No firmware binary is modified to do this — the loader simply resolves the
  sonames to the shim.
- A small **bridge/forwarder** carries captured frames out of the **no-network** container
  (over the container exec channel's standard streams) to a viewer on the PC — a browser page
  or a native window — and carries input events back in. Only differential frame data crosses;
  the HMI's draw thread is never blocked by a slow or absent viewer.

What this buys, and what it does **not**: it lets you **see and record the HMI's actual
rendering** — start-up screens, the menu grid, settings, pop-ups — and confirm it issues **no
invalid GL** (a measured run used ~60 distinct GL/EGL operations, essentially an OpenGL ES 2.0
feature set, with zero rejected shaders). **[verified]** It is **software** rendering: correct
pixels, not GPU-accurate timing, and only for the head-unit screen. It does **not** give you
the hardware video pipeline to the cluster — that is a separate wall (§6).

---

## 5. Mocking the service buses so the HMI can come up

Out of the box the HMI blocks early, waiting for services that, in the car, are published by
other processes and other ECUs. The bench supplies just enough of that surface — using, wherever
possible, **the firmware's own mock infrastructure** — so the interface progresses. Two buses:

### 5.1 VIWI / RSI — the firmware already ships a mock

The HMI discovers services through a **service registry** over WebSocket, and blocks until a
set of **RSI** services (a REST/JSON service bus; see the glossary) register — user
management, online-registration and so on. The firmware itself carries, under its read-only
app partition: **[verified]**

- the **service registry** app the HMI looks for;
- an **LG-provided RSI mock** with its service list, plus **~80 service specifications** and
  per-service test data;
- command-line RSI helpers.

The method is to run the firmware's **own registry + mock** (they start once the IPC transport
in §5.2 is available), add any missing service to the mock's list, and overlay a couple of
**synthetic data files** so the records the HMI requires are present. With that, the HMI
finishes its loading screen and proceeds into its normal screens. **[verified]**

> **Nothing was bypassed here, and this is central to the read-only line.** The registry
> accepted the LG mock exactly as shipped; the HMI spoke plain `ws://` with no certificate and
> no token because that is what the loopback service surface uses. No authentication,
> signing, dm-verity or secure boot was touched, weakened or circumvented — the HMI simply
> found, on loopback, the services it always looks for. A real security-backed daemon that
> wants a hardware challenge chip was **left alone** (imitating it would step outside scope,
> and the LG mock already provides the same data to the HMI without it). **[verified]**

### 5.2 DSI — a userspace IPC transport + synthetic vehicle data

The HMI is also a **client of ~50 DSI services** carrying **protobuf** messages (display
management, power management, persistence, key panel, the car-state services, …). These ride
an LG inter-process transport (**kipc**, exposed as the device nodes `/dev/csr` / `/dev/css` —
an LG kernel driver that exists on no public kernel). In the car those daemons ultimately talk
to the MCU and CAN; the PC has neither. The method: **[verified]**

- Provide the kipc transport **in userspace** — intercept the node open/read/write and route
  messages through a small in-process broker — so the firmware's **real** service daemons
  (persistence, power/MCU interpreter, touch, …) start and register over it **without** the
  car's hardware.
- For the services that need vehicle state, supply **synthetic, generic data** — a fabricated
  vehicle profile standing in for the car bus (power "on", display geometry, language/units,
  a plausible equipment set). This is deliberately **generic platform data**, not any real
  vehicle's values, and it is how the HMI is persuaded to leave its "waiting for the car"
  states and compose real screens. **[verified]**

The protocol details (message framing, the request→response api-id pairs, the display-size and
power-state handshakes) were recovered by decompiling the relevant libraries and confirmed in
execution; they are **described and referenced** in the private bench log, **not reproduced
here** — they are ordinary service internals, but pasting them adds nothing to a methodology
note and would bloat it with proprietary-derived strings. The *shape* is what matters: a
client HMI, a registry, a set of service mocks, and synthetic car data, all on loopback.

---

## 6. What does **not** work in a PC container, and why

Honesty about the ceiling is the point of a read-only bench. Some things are genuinely
unreachable on a PC, and the *reason* is as informative as the result.

1. **The hardware video encoder → no cluster map video.** The native map reaches the
   instrument cluster as an **IP-multicast H.264 stream** produced by the Qualcomm video
   encoder, fed through the SoC's shared-memory path (the Android-style **ION**/DMA-BUF
   allocator present in this kernel). A PC container has **no Qualcomm video core and no ION
   device**, so the producer chain that encodes and streams the map (`naviengine` →
   `multi-screen-server`, §3.2) **cannot run** in the sandbox. The transport and codec are
   documented from the code in [`cluster-map.md`](cluster-map.md); the bench can study the
   *setup* and *gating* of that pipeline, but it cannot generate the actual video. **[verified
   that the encoder path is hardware; [inference] on exact encoder parameters, which live
   cluster-side.]**
2. **The native GPU driver.** The firmware's Adreno driver needs Qualcomm silicon via DRM;
   there is no software fallback inside it. The §4 forwarder works around this for the
   *head-unit screen* by substituting a software GL implementation at the API boundary — but
   genuine GPU-accelerated rendering, and anything that reads back from a real framebuffer, is
   unavailable. Software rendering is correct but slow, and it is the head-unit screen only.
   **[verified]**
3. **The cluster is a different ECU, not in this image.** The head unit is the *producer* of
   the map video; the *consumer* — the cluster ECU (VW diagnosis address 17) that joins the
   multicast group and rasterises it — has its **own firmware, which is not in the head-unit
   package**. No amount of head-unit emulation shows the cluster rendering; that receiver side
   is an **open question** (see [`open-questions.md`](open-questions.md)). **[verified]**
4. **Hardware-backed security elements.** Daemons that expect a security/challenge chip over a
   dedicated bus cannot be satisfied on a PC and are **deliberately not imitated** — out of
   scope, and unnecessary for studying the HMI (§5.1). **[verified]**

**What this implies for study.** Everything *above* the hardware abstraction is observable
read-only: HMI start-up and OSGi lifecycle, screen composition, input handling, the
coding/adaptation behaviour, and the on-device service protocols. Everything that *is* the
Qualcomm hardware — the real GPU timing, the hardware video encoder, the cluster's own
rendering — is not reproducible on a PC and must be reasoned about from code plus the mocked
surface, or deferred to the open-questions list. A bench that is clear about this line is more
useful than one that pretends to render a cockpit it cannot.

---

## 7. What the methodology yields (the value)

A **reproducible, read-only RE methodology for the MOI3 LG HMI** — to our knowledge the first
time this specific platform (AGL + Adreno + glibc 2.23, as opposed to the older Harman QNX
units) has been driven this way in the open. Concretely, the bench lets a researcher, entirely
in sandbox analysis:

- **run the real HMI** far enough to watch it compose its real screens, and **record exactly
  what it draws**, with no GPU and no car; **[verified]**
- **exercise input** (keys, rotary controls, touch) and observe the HMI's response; **[verified]**
- **trace a coding/adaptation value end to end** — from an adaptation bit, through the native
  code, to an enabled feature — as the dynamic half of the proof in
  [`cluster-map.md` §4.1](cluster-map.md#41-the-enable-chain-proven-static-and-dynamic);
  **[verified]**
- do all of the above **repeatably**, because each run reads an untouched image against a RAM
  scratch layer and brings up only the mocks a given question needs.

Its honest limits are the §6 walls and the fact that only one HMI variant/resolution and a
small number of firmware trains have been exercised — exactly the kind of gap the project asks
contributors to help close ([`open-questions.md`](open-questions.md)). A physical approach
(a real head unit on a bench with a frame grabber) is the only way to see the true cluster
video, and is a different, non-PC undertaking outside the scope of this read-only method.

---

## Sources (public tools and concepts)

The bench is built entirely from **public, open-source tooling** operating on a package the
researcher already possesses; no package contents, binaries, HMI assets or vehicle data are
reproduced in this document.

- **libarchive / `bsdtar`** — reads the update container read-only: <https://www.libarchive.org/>
- **e2fsprogs / `debugfs`** — reads the ext4 `system` image without mounting:
  <https://e2fsprogs.sourceforge.net/>
- **QEMU** — the `-M virt` ARM64 environment: <https://www.qemu.org/>
- **Mesa 3D / `llvmpipe`** — software OpenGL ES used out-of-process by the glhost:
  <https://docs.mesa3d.org/> · <https://docs.mesa3d.org/drivers/llvmpipe.html>
- **Khronos OpenGL ES** — the API surface the forwarder reproduces:
  <https://www.khronos.org/opengles/>; software GLES alternatives:
  <https://github.com/google/angle> · <https://github.com/google/swiftshader>
- **Wayland / Weston** — the compositor/display protocol the HMI renders through:
  <https://wayland.freedesktop.org/>
- **Automotive Grade Linux** and **Yocto/Poky** — the firmware's distribution and build
  system: <https://www.automotivelinux.org/> · <https://www.yoctoproject.org/>
- **systemd** — the unit/ordering model described in §3.2: <https://systemd.io/>
- **Ghidra** and the **CFR** Java decompiler — static RE of the native daemons and HMI jars:
  <https://ghidra-sre.org/> · <https://www.benf.org/other/cfr/>

> This document is a structural/methodological description for public research. It contains no
> firmware, no proprietary assets, no vehicle-identifying data, and **no instructions for
> defeating the unit's protections** — running the HMI on a PC neither modifies a unit nor
> bypasses Secure Boot, dm-verity or FDS signing, which remain out of scope. For the package
> structure and the protection layers see [`firmware.md`](firmware.md); for the cluster
> pipeline see [`cluster-map.md`](cluster-map.md); for the open gaps see
> [`open-questions.md`](open-questions.md).
