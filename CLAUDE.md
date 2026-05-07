# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

User-config repo for the **Zonkey**, a 42-key wireless split keyboard (row-staggered, with rotary encoder on the left half and a PMW3610 trackball on the right half). Firmware is built on **ZMK** with two Seeeduino XIAO BLE controllers (one per half). There is no local toolchain — firmware is produced by GitHub Actions.

## Build / firmware workflow

- Builds are driven by `.github/workflows/build.yml`, which delegates to `zmkfirmware/zmk/.github/workflows/build-user-config.yml@v0.3`. Trigger: any push, PR, or `workflow_dispatch`.
- The build matrix is declared in `build.yaml` at the repo root. Today it produces three artifacts, all on `seeeduino_xiao_ble`:
  - `zonkey_R rgbled_adapter` (right / central, has trackball)
  - `zonkey_L rgbled_adapter` (left / peripheral, has encoder)
  - `settings_reset` (recovery firmware to clear BLE pairings)
- Dependencies are pinned in `config/west.yml`: ZMK `v0.3`, plus out-of-tree modules `kureyakey/zmk-pmw3610-driver` (trackball driver) and `caksoylar/zmk-rgbled-widget@v0.3` (status LED widget). Bumping ZMK usually requires bumping `zmk-rgbled-widget` to a matching tag.
- To get firmware locally: push the change, open Actions, download the artifact zip, and flash the `.uf2` files via the XIAO BLE bootloader. There is no `west build` flow checked into this repo.

## Architecture

Split layout, **right side is central** (`ZMK_SPLIT_ROLE_CENTRAL=y` in `config/boards/shields/zonkey/Kconfig.defconfig`). Practical implications when editing:

- **Right-only hardware** (trackball, SPI bus, automouse layer wiring) lives in `config/boards/shields/zonkey_R.overlay` and `config/zonkey_R.conf`. The PMW3610 is the trackball; `automouse-layer = <6>` and `scroll-layers = <5>` mean layers 5/6 are reserved for pointer/scroll behavior — don't repurpose them without also updating the overlay.
- **Left-only hardware** (EC11 encoder) lives in `config/boards/shields/zonkey_L.overlay` and `config/zonkey_L.conf`. The encoder is declared in `zonkey.dtsi` as `left_encoder` (`status = "disabled"` by default; the L overlay enables it).
- **Shared shield definitions** are in `config/boards/shields/zonkey/`:
  - `zonkey.dtsi` — matrix transform (4 rows × 11 cols, `col2row` diodes), kscan GPIO mapping, encoder + sensors nodes.
  - `Kconfig.shield` / `Kconfig.defconfig` — split role and keyboard name per side.
  - `zonkey.zmk.yml` — shield metadata consumed by the ZMK build workflow.
- **Both halves share the keymap** at `config/zonkey.keymap`. The matrix transform in `zonkey.dtsi` plus `col-offset = <6>` in `zonkey_R.overlay` is what stitches the two halves into one logical 42-key map.

## Keymap conventions (`config/zonkey.keymap`)

- **7 layers** (`default_layer`, `layer_1` … `layer_6`). The default layer uses `&lt` (layer-tap) on the thumb cluster: space → L1, BS → L2, ENT → L3, plus `lt 4 B`, `lt 5 V`, `lt 6 C` on the inner column. Layers 1–3 are number row / symbols / F-keys, layer 4 is numpad+BT controls, layer 5 is scroll, layer 6 is mouse buttons (entered automatically when the trackball moves).
- The `JP_*` macros at the top of the file remap US-keycodes to the glyphs they produce on a **JIS / Japanese IME**. Many `mod-morph` behaviors (`mo2`…`moI`) implement JIS-correct shifted symbols. When adding symbol keys, prefer the existing `JP_*` aliases / mod-morphs over raw `LS(...)` so behavior stays consistent on both US and JIS hosts.
- `&mt` (mod-tap) global config is `flavor = "balanced"`, `quick-tap-ms = 0`. Hold-tap timing changes here affect every `&mt` binding, so adjust per-binding instead unless a global change is intended.
- The visual ASCII layout comments above each layer's `bindings = <...>` block are hand-maintained — keep them in sync when you change the bindings, since they're the only readable map of what each key does.

## When changing things

- **Adding a layer**: extend the keymap, but also remember anything that references layer numbers by index — currently `automouse-layer`, `scroll-layers` (in `zonkey_R.overlay`), and the `&lt N ...` / `&mo N` / `&to N` bindings in the keymap. Inserting a layer in the middle shifts all higher indices.
- **Changing matrix size or pin map**: edit `zonkey.dtsi` (transform + kscan) and the per-side overlays (`col-gpios`); the keymap's binding count must match `rows × cols` of the transform.
- **Symptom: keys produce wrong glyphs on macOS/Windows**: usually a JIS-vs-US IME issue, not a firmware bug — check whether the binding should be using a `JP_*` alias.

## Reference

- End-user setup, flashing, and pairing instructions are in `README.md` (Japanese). It is written for the keyboard's buyers, not contributors — treat it as product documentation, not architectural notes.
- Upstream keymap commentary: https://note.com/goya_k (linked from README).
