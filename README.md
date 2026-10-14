# Rpi_codex_win_drivers

Experimental work on Windows on ARM drivers for Raspberry Pi hardware.

## Target

- Raspberry Pi 400 / Raspberry Pi 4
- Windows 11 ARM64

## Current status

This repository is a project scaffold. It does not yet contain a Wi-Fi driver, firmware, or a buildable driver package.

## Initial investigation

- Identify the built-in Broadcom/Cypress CYW43455 SDIO functions and their Windows device IDs.
- Find an ARM64-compatible driver and firmware path, and validate the source and documentation before attempting an implementation.