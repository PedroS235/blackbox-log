---
name: bump-betaflight
description: Add support for a new Betaflight firmware version to blackbox-log — diff the upstream enums, write/symlink the version's YAML mappings, wire up InternalFirmware, regenerate. Use when asked to bump, add, or support a new Betaflight version (e.g. "support BF 2026.12", "bump Betaflight support"), or when a log fails to parse because its firmware version is out of the supported range.
---

# Bump Betaflight support

Goal: a new `types/data/Betaflight/<VERSION>/` mapping set + `InternalFirmware` variant, with codegen re-run and the build/tests/clippy clean.

## 0. Pick the upstream tag

List release tags

```bash
for p in 1 2 3 4; do curl -sS "https://api.github.com/repos/betaflight/betaflight/tags?per_page=100&page=$p"; done \
  | grep -o '"name": *"[^"]*"' | sed 's/.*: *"//;s/"//' | grep -E '^20' | sort -Vr | head
```

Use the newest **non-rc** patch of the target release (e.g. `2026.6.1`), and the newest patch of the previous supported release as the baseline for diffing (e.g. `2025.12.5`).

Directory name is `MAJOR.MINOR` only (`2026.6`) — patch versions share one mapping.

## 1. Diff every source of truth

Do **not** trust any "this file never changes" claim — 2026.6 broke two of them. Diff all seven:

| Upstream file | YAML | Enum | Index rule |
|---|---|---|---|
| `fc/rc_modes.h` | `flight_mode.yaml` | `boxId_e` | ordinal (skip aliases like `BOXID_FLIGHTMODE_LAST`) |
| `build/debug.h` | `debug_mode.yaml` | `debugType_e` | ordinal |
| `config/feature.h` | `features.yaml` | `features_e` | `N` from `(1 << N)` — gaps are normal |
| `blackbox/blackbox_fielddefs.h` | `disabled_fields.yaml` | `FlightLogFieldSelect_e` | ordinal |
| `flight/failsafe.h` | `failsafe_phase.yaml` | `failsafePhase_e` | ordinal |
| `drivers/motor_types.h` | `pwm_protocol.yaml` | `motorProtocolTypes_e` | ordinal, skipping commented-out entries (`DSHOT1200`) |
| `fc/runtime_config.h` | `state.yaml` | `stateFlags_t` | `N` from `(1 << N)` |

Notes learned the hard way:
- The pwm enum used to live in `drivers/motor.h` and moved to `drivers/motor_types.h`; if a fetch turns up nothing, grep for `MOTOR_PROTOCOL_ONESHOT125` across likely headers instead of assuming no change.
- `runtime_config.h` also holds `armingDisableFlags_e` and `flightModeFlags_e` — those are **not** mapped here; only `stateFlags_t` feeds `state.yaml`.

Fetch and diff:

```bash
cd "$SCRATCH" && mkdir -p old new
for f in fc/rc_modes.h build/debug.h config/feature.h blackbox/blackbox_fielddefs.h \
         flight/failsafe.h drivers/motor_types.h fc/runtime_config.h; do
  n=$(basename $f)
  curl -sS -o new/$n "https://raw.githubusercontent.com/betaflight/betaflight/$NEW_TAG/src/main/$f"
  curl -sS -o old/$n "https://raw.githubusercontent.com/betaflight/betaflight/$OLD_TAG/src/main/$f"
  echo "=== $n"; diff -u old/$n new/$n
done
```

**Insertions and removals shift every later index.** Never append blindly — regenerate the whole ordinal list from the new enum.

## 2. Extract indices mechanically, then self-check

Generate the list from the header rather than hand-counting, e.g. for debug modes:

```bash
sed -n '/typedef enum {/,/} debugType_e/p' new/debug.h \
  | grep -oE '^\s+DEBUG_[A-Z0-9_]+' | sed 's/^ *DEBUG_//' | grep -v '^COUNT$' \
  | nl -v0 -ba -w1 -s': '
```

Sanity gate: run the same extraction on the **old** tag and diff the names against the existing YAML for the previous version. If they don't match, the extraction is wrong — fix it before writing anything.

## 3. YAML names are project-chosen, not upstream strings

Names are historical and carry over unchanged from the previous version's YAML — `BOXCRASHFLIP` is `TURTLE`, `BOXAIRMODE` is `AIRMODE` (not msp_box's `"AIR MODE"`), `BOXOSD` is `OSD`. **Keep existing names verbatim**; only choose a name for genuinely new entries. For those, use the enum name minus its prefix, with spaces where the upstream `boxName` in `msp/msp_box.c` has them (`BOXWPCAPTURE` → `WP CAPTURE`).

Codegen PascalCases names automatically; add an entry to `types/meta/<file>.yaml`'s `rename:` map only when the automatic conversion is wrong.

The practical move for `flight_mode.yaml` is: copy the previous version's list of names, insert/remove per the diff, renumber from 0.

## 4. Write the version directory

Unchanged file → symlink to the version that last changed it (chase to the real file, don't symlink a symlink):

```bash
ln -s ../4.5/state.yaml state.yaml
```

Changed file → real file, one `NAME: index` per line, `\n`-terminated.

## 5. Wire up Rust

`src/headers.rs` — three places, all exhaustive matches (the compiler catches misses):

```rust
pub(crate) enum InternalFirmware { … BetaflightXXXX_YY, … }
// is_betaflight(): add to the `=> true` arm
// From<Firmware>: Firmware::Betaflight(FirmwareVersion { major: XXXX, minor: YY, .. }) => Self::BetaflightXXXX_YY,
```

`src/lib.rs` — `BETAFLIGHT_SUPPORT` is a **half-open range**; the upper bound is *excluded*. To support 2026.6 the bound must be strictly above every 2026.6.x, so set it to the next expected release (`2026.12.0`), matching how earlier bumps did it (4.5 support → bound `4.6.0`).

## 6. Regenerate and verify

```bash
cargo run -p codegen   # rewrites src/generated/
cargo build
cargo test
cargo clippy --all-targets
cargo fmt --check
```

Skim `git diff src/generated/` for the new variant: every shared index should gain `BetaflightXXXX_YY` in its firmware list, and new entries should appear as `(N, BetaflightXXXX_YY) => Self::…`. If an index silently lost the new firmware, a YAML is wrong.

## 7. Report

State which enums changed and how indices shifted — that's the part a reviewer can't see from the diff. Commit only when asked; `types/data/**` symlinks must stay mode `120000` in the commit.
