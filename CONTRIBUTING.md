# Contributing

Thanks for helping document the **MIB3 "MOI3" (LG)** platform. This project is a reference built from **read-only analysis** and public documents. The more cars, markets and firmware versions people verify, the more useful it gets.

## Ground rules

1. **No vehicle-identifying or personal data.** Never include VINs, number plates, locations, owner names, serial numbers, FAZIT, or raw autoscan dumps. Talk about the platform generically. Redact before you paste.
2. **In scope:** architecture, firmware structure, the cluster-map pipeline, coding/adaptation fields, public tooling, and verification across models/markets/firmware. See [docs/open-questions.md](docs/open-questions.md) for where help is most useful.
3. **Out of scope:** anything that defeats **Secure Boot, dm-verity or VW firmware signing**. We document *that* these protections exist and *how* the system is built; we do **not** publish bypass methods, patch offsets, or step-by-step instructions aimed at them. PRs that do will be declined.
4. **No proprietary material:** no firmware binaries, no confidential manuals, no large blocks of decompiled proprietary code. Describe it and link the public source.
5. **Mark your confidence.** Say whether something was measured, disassembled, read from a diagnostic tool, or inferred — and cite the source.

## How

- **Open an issue** for a finding, a correction, or a question. Where relevant, include model, market and firmware train / MU version — **never a VIN**.
- **Pull requests:** keep them focused and follow the tone and rules above.

## Safety

Coding or firmware changes to a vehicle can affect safety, warranty and regulatory compliance, and can render modules inoperative. Everything here is informational — you act at your own risk.
