# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A personal [ZMK](https://zmk.dev) firmware config (a "zmk-config" repo) — it holds only the
user-side keymap/config; the firmware source itself lives upstream in `zmkfirmware/zmk`.
Current hardware: a split **Corne** (42-key, 3x6+3) on two **nice!nano v2** controllers.

There is no application code, no test suite, and no lint step. The deliverable is a pair of
`.uf2` files produced by CI.

## Build

Builds happen in **GitHub Actions**, not locally. `.github/workflows/build.yml` calls the reusable
`zmkfirmware/zmk/.github/workflows/build-user-config.yml@v0.3`, which reads `build.yaml` as a job
matrix and uploads the resulting `.uf2` artifacts.

- Trigger: pushes to `main` and manual `workflow_dispatch` only. Branches and PRs deliberately do
  not build — firmware is only cut from what's actually on `main`.
- To flash: download the run's artifact zip, double-tap reset on each half to mount the bootloader
  volume, copy `corne_left-nice_nano_v2-zmk.uf2` / `corne_right-...uf2` across.
- Verify a change by pushing to `main` and checking the run (`gh run list` / `gh run watch`), or by
  kicking a branch build manually with `gh workflow run "Build ZMK firmware" --ref <branch>`. A
  keymap syntax error only surfaces there — it's the only feedback loop available.

There is no local west toolchain installed (no `west`, no Zephyr SDK). Don't attempt a local build
unless explicitly asked to set that up.

## Layout and how the pieces connect

- `build.yaml` — the CI matrix. Each `include` entry is one firmware image (`board` + `shield`,
  optionally `snippet`, `cmake-args`, `artifact-name`). Adding a keyboard means adding entries here
  **and** a matching `config/<shield>.keymap` / `.conf`.
- `config/west.yml` — pins the ZMK version (`revision: v0.3`). Bumping this changes every behavior
  and binding available; also bump the `@v0.3` ref in `.github/workflows/build.yml` to match.
- `config/corne.keymap` — devicetree overlay defining the `keymap` node. Filename must match the
  shield name minus the `_left`/`_right` suffix; both halves share one keymap.
- `config/corne.conf` — Kconfig for that shield (RGB underglow and OLED display are present but
  commented out).
- `zephyr/module.yml` + `boards/` — declares this repo as a Zephyr module with `board_root: .`, so
  custom boards/shields dropped in `boards/shields/` are discovered by the build. Currently empty.
- `.zmk/` — **gitignored**, local only. A partial west checkout of ZMK source (zephyr/lvgl filtered
  out) used by editor tooling. Read `.zmk/zmk/app/` for behavior definitions, `dt-bindings` keycode
  names, and Kconfig options when you need to check what a binding or setting is called. Never edit
  or commit anything under it.

## Editing the keymap

Layers are positional: `&mo 1` / `&mo 2` refer to the second and third layer nodes in source order,
so reordering or inserting a layer silently rebinds them.

Each layer's `bindings` is a flat array of 42 entries in row-major order — 12 per row for three
rows, then 6 thumb keys — with the left half's 6 columns preceding the right half's on every row.
The ASCII comment above each layer is the map of that order; keep it in sync when changing bindings,
since it is the only thing making the flat array readable. Use `&trans` (fall through to the layer
below) rather than deleting a slot — the array length is fixed by the shield.

Keycode names come from `dt-bindings/zmk/keys.h`; behaviors (`&kp`, `&mo`, `&bt`, `&mt`, ...) from
`behaviors.dtsi`. Both are in `.zmk/zmk/app/` — check there rather than guessing a name.
