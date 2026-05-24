# Repository Guidelines

## Project Structure & Module Organization

This repository contains the VESC BMS STM32L476 firmware. Core application modules live at the repository root, with paired `.c`/`.h` files such as `comm_can.c`, `bms_if.c`, `commands.c`, and `selftest.c`. Hardware-specific implementations are in `hwconf/`; select or add hardware through `HW_SRC` and `HW_HEADER`. Configuration metadata is in `config/`, sensor and peripheral drivers are in `drivers/`, STM32 HAL shims are in `st_hal/`, and embedded third-party/support code is in `ChibiOS_20.3.0/`, `blackmagic/`, and `compression/`. Documentation assets are under `documentation/`.

## Build, Test, and Development Commands

- `make`: builds the default firmware image in `build/` using `arm-none-eabi-gcc`.
- `make HW_SRC=hw_stormcore_bms.c HW_HEADER=hw_stormcore_bms.h`: builds for a specific hardware definition in `hwconf/`.
- `make clean`: removes generated build output when supported by the included ChibiOS rules.
- `make upload`: flashes `build/vesc_bms.elf` through OpenOCD using `stm32l4_stlinkv2.cfg`.
- `make server`: starts OpenOCD and halts the target for debugging.
- `pandoc -f markdown-implicit_figures -o README.pdf README.md`: regenerates the PDF documentation.

Install an ARM embedded GCC toolchain and OpenOCD before building or flashing.

## Coding Style & Naming Conventions

Use C with the existing firmware style: tabs for indentation, K&R-style braces for functions and control blocks, lowercase snake_case for functions and variables, and uppercase names for macros/constants. Keep module APIs in matching headers, include project headers before standard library headers, and prefer small firmware modules over broad utility files. Do not reformat vendored directories unless the change is required.

## Testing Guidelines

There is no host-side unit test framework in this tree. Validate changes by building with `make` and, for behavior changes, flashing hardware and exercising the relevant subsystem through CAN, USB, UART, or VESC Tool. Keep self-test additions in `selftest.c`/`selftest.h`, and document any manual hardware checks in the pull request.

## Commit & Pull Request Guidelines

Recent history uses concise, imperative commit subjects such as `Add support for COMM_FW_INFO` and `Embed full git commit hash`. Keep subjects specific and under roughly 72 characters. Pull requests should describe the affected hardware/configuration, list build and hardware test results, link related issues, and include screenshots only when VESC Tool UI or documentation images change.

## Safety & Configuration Notes

Battery firmware changes can affect charging, balancing, and fault handling. Treat defaults in `conf_general.*`, `config/settings.xml`, and `hwconf/` as safety-critical; explain threshold or fault-policy changes clearly and test them on the intended hardware revision.
