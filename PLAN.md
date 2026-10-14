# Windows on Raspberry Pi Built-in Wi-Fi Driver Plan

**Status:** research and planning; no driver implementation or CI workflow exists yet.
**Target:** Raspberry Pi 400 built-in CYW43455 Wi-Fi on Windows 11 ARM64.
**Plan date:** 2026-10-14.

## Current baseline

### Repository

- The repository currently contains the project README and this plan; there is no driver source, firmware, build project, or Actions workflow yet.
- The intended first CI target is an ARM64 driver package built reproducibly from source.

### Local target machine

- Raspberry Pi 400, Broadcom BCM2711 / Cortex-A72, ARM64.
- Windows 11 Enterprise ARM64, OS build 22621 (22H2).
- 4 GB physical RAM; boot storage is a 64 GB SD card.
- Windows reports Code 28 for seven present devices. Two are SDIO functions with IDs `SD\VID_02D0&PID_A9A6&FN_2` and `SD\VID_02D0&PID_A9A6&FN_3`; both are children of `ACPI\BCM2847\0`.
- The separate RTL8188EUS USB Wi-Fi adapter has a driver installed. It is not the built-in Wi-Fi device.
- No WDK, Visual Studio, MSBuild, MSVC compiler, InfVerif, Inf2Cat, or SignTool was detected in PATH or the standard Windows Kits / Visual Studio locations. GitHub Actions should be the primary build environment; the Pi should be reserved for hardware validation.

This baseline intentionally excludes serial numbers, MAC addresses, network names, and credentials.

## Research findings so far

1. The Windows on Raspberry driver project lists CYW43455 Wireless LAN on Pi 4 / 400 as having no driver available. This is a driver-development project, not just a missing package install. See [WoR driver status](https://github.com/worproject/RPi-Windows-Drivers#status).
2. Microsoft describes WiFiCx as the Windows 11 Wi-Fi driver framework, built on KMDF and NetAdapterCx. For Windows 11 21H2 / 22H2, WiFiCx operation requires NetAdapterCx 2.2. WDI is in maintenance mode. See [WiFiCx overview](https://learn.microsoft.com/en-us/windows-hardware/drivers/netcx/wifi-wdf-class-extension-wificx), [WiFiCx version compatibility](https://learn.microsoft.com/en-us/windows-hardware/drivers/netcx/building-a-wificx-driver-for-multiple-versions-of-windows), and [WDI design guide](https://learn.microsoft.com/en-us/windows-hardware/drivers/network/wdi-miniport-driver-design-guide).
3. The Windows SD Bus guide documents the SDIO client-driver model and APIs. Its generic SDIO sample is old and marked no longer supported, so it is a reference rather than a production driver base. See [SD Bus Driver Design Guide](https://learn.microsoft.com/en-us/windows-hardware/drivers/sd/).
4. Microsoft supports ARM64 driver builds with current WDK versions, including builds from Windows on ARM or x64 development hosts. The exact Visual Studio, SDK, WDK, and target OS versions must be pinned in CI. See [Building ARM64 drivers](https://learn.microsoft.com/en-us/windows-hardware/drivers/develop/building-arm64-drivers) and [WDK downloads](https://learn.microsoft.com/en-us/windows-hardware/drivers/download-the-wdk).
5. Infineon's firmware repository has a Cypress licence that differs from the upstream linux-firmware licence. Firmware reuse, modification, and redistribution must be checked file-by-file before committing blobs or publishing packages. See [Infineon firmware repository](https://github.com/Infineon/ifx-linux-firmware) and its [LICENCE](https://github.com/Infineon/ifx-linux-firmware/blob/master/LICENCE).
6. GitHub documents standard Windows hosted runners and Windows ARM64 hosted runners. Start with a standard Windows x64 runner cross-building ARM64 for a stable CI baseline; evaluate the Windows ARM64 preview runner separately. See [GitHub-hosted runners](https://docs.github.com/en/actions/reference/runners/github-hosted-runners).

## Phases and exit criteria

### Phase 1 — Freeze the local hardware and OS baseline

**Work**

- Re-capture the SDIO devices' full hardware IDs, compatible IDs, parent chain, bus-reported descriptions, and current driver matches.
- Record the Arasan SDIO host-controller driver, WoR driver-package version, UEFI version, Windows servicing level, and relevant ACPI/DSDT device declarations.
- Collect only relevant SetupAPI / PnP diagnostics. Redact serials, MACs, SSIDs, usernames, and credentials before saving logs.
- Keep the USB Wi-Fi adapter separate from the built-in CYW43455 investigation.

**Deliverable:** `docs/system-baseline.md` with reproducible device evidence.
**Exit gate:** confirm the precise function-to-chip mapping and the host-controller path without changing installed drivers.

### Phase 2 — Internet research and source/licence audit

**Work**

- Read the WoR Pi 4/400 driver package, Raspberry Pi Windows BSP sources, Microsoft SDIO/WDF/WiFiCx samples, and the Linux `brcmfmac` implementation as protocol references.
- Identify the CYW43455 SDIO function roles, firmware boot sequence, required NVRAM/CLM data, mailbox/control commands, and data-path framing.
- Trace firmware and calibration/regulatory files to their authoritative sources; record hashes and individual licence terms. Do not copy binary firmware into this public repository until the distribution right is clear.
- Confirm the actual support floor for Windows build 22621 and whether the local WoR image/UEFI combination has constraints that affect WiFiCx or NetAdapterCx.

**Deliverable:** `docs/research.md` and a source/licence table.
**Exit gate:** protocol evidence and usable firmware path are documented. If either is unavailable, stop before implementing a pretend or nonfunctional WLAN driver.

### Phase 3 — Choose and document the driver architecture

**Work**

- Target WiFiCx + KMDF/NetAdapterCx for Windows 11, subject to confirming the exact 22H2 compatibility configuration for build 22621.
- Design clear layers for SDIO transport, firmware/NVRAM loading, device control/data path, WiFiCx integration, power management, and regulatory behavior.
- Define ownership and synchronization for SDIO interrupts, firmware state, and packet buffers before implementation.
- Record known limitations and supported features instead of claiming full WLAN support early.

**Deliverable:** `docs/driver-design.md`, interface map, and milestone checklist.
**Exit gate:** each required interface has a source reference, a test strategy, and a defined owner in the driver.

### Phase 4 — Establish GitHub Actions driver-build CI

**Work**

- Add a workflow for pull requests, pushes to `main`, and manual runs.
- Use a pinned Windows hosted runner image and pinned Visual Studio / Windows SDK / WDK versions. Start with a stable Windows x64 runner cross-building the `ARM64` target; use the ARM64 hosted runner only as a later compatibility check.
- Build with MSBuild for ARM64; validate INF files with `InfVerif`; generate and verify the package catalogue with `Inf2Cat`; publish `.sys`, `.inf`, `.cat`, and logs as a short-lived Actions artifact.
- Use minimal workflow permissions (`contents: read`) and pin third-party Actions to commit SHAs. CI should not need a personal access token.
- If test signing is needed for artifacts, create a temporary test certificate during the job and discard its private key. Never store a production signing key in the repo or expose a test key as a release credential.
- Do not auto-install the driver, change boot policy, or reboot a self-hosted Pi from CI.

**Deliverable:** `.github/workflows/driver-ci.yml` and a repeatable ARM64 package build.
**Exit gate:** a clean checkout builds the same target, INF/catalog checks pass, and artifacts contain no signing secrets or unlicensed firmware.

### Phase 5 — Implement in bring-up milestones

**Work**

1. Driver skeleton binds only to the intended SDIO function IDs and reports useful diagnostics.
2. Validate SDIO register access, interrupts, reset, and power transitions.
3. Load the permitted firmware/NVRAM data and prove firmware boot/mailbox communication.
4. Register a WiFiCx adapter and expose only validated capabilities.
5. Add scan, association, WPA2 connectivity, packet TX/RX, and then power-save/regulatory behavior.

**Deliverable:** CI-built driver packages and a short evidence log for each milestone.
**Exit gate:** each milestone works on the real Pi before the next feature is layered on.

### Phase 6 — Hardware validation and release packaging

**Work**

- Test manually on the Pi with a recovery/rollback path and the existing USB adapter available as fallback.
- Validate device start, scan results, connection stability, reboot/resume behavior, logs, and removal/rollback.
- Keep test-signed development artifacts distinct from any public release. Follow Microsoft's production signing path before distributing a driver package. See [Microsoft driver signing](https://learn.microsoft.com/en-us/windows-hardware/drivers/develop/signing-a-driver).

**Deliverable:** versioned ARM64 package, installation/rollback instructions, supported OS build, licence notices, and known limitations.
**Exit gate:** clean install, stable real-hardware tests, reproducible CI artifact, and cleared redistribution terms.

## CI versus hardware testing

GitHub CI can compile ARM64 code, validate INF/catalog packaging, and retain artifacts. A hosted runner cannot prove that SDIO communication, firmware boot, Wi-Fi scanning, or association works on the Raspberry Pi 400. Those checks require manual hardware validation on the Pi.

## Security note

A personal access token was pasted into chat earlier and should be revoked. CI must use the repository-scoped `GITHUB_TOKEN` with minimal permissions; no personal token is needed for building drivers or uploading workflow artifacts.