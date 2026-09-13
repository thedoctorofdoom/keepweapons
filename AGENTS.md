# AGENTS.md — Keep Weapons for Project Brutality

## Project Overview

This is a **ZScript-based addon** for the [`PB_Staging`](https://github.com/pa1nki113r/Project_Brutality/tree/PB_Staging) branch of [Project Brutality](https://github.com/pa1nki113r/Project_Brutality), a total-conversion mod for classic Doom running on the **UZDoom/GZDoom** source ports. It is designed to coexist with **Monster Pack** (PB's companion monster addon).

The addon overrides PB's weapon upgrade system so that picking up an upgraded weapon **no longer removes** the base weapon from the player's inventory. Both base and upgraded versions remain available. Behavior is controlled at runtime via the `pbx_keep_all_weapons` CVar.

## File Structure

```
pbx_keep_all_weapons_addon/
├── AGENTS.md                              # This file
├── CVARINFO                               # CVar declaration (pbx_keep_all_weapons)
├── README.md                              # End-user documentation
└── zscript/
    └── Weapons/
        ├── BaseWeapon.zc                   # PB_WeaponBase class + subclasses (overrides PB upstream)
        └── BaseWeapon_Functions.zsc        # extend class PB_WeaponBase — utility functions (3-way merge: PB upstream + Monster Pack execution handlers + KeepWeapons CVar guards)
```

## How It Works

Three code locations implement the keep-all-weapons behavior:

1. **`BaseWeapon.zc` — `HandlePickup()` override (line ~88)**
   Reads `pbx_keep_all_weapons` from the owning player's CVars. When `true`:
   - Skips the `While(owner && self.amount > 0){ owner.DropInventory(self,1); }` loop that normally removes the base weapon on upgrade pickup.
   - Skips the `else if` branch that would consume a duplicate base-weapon pickup when the player already owns the upgrade.

2. **`BaseWeapon_Functions.zsc` — `PB_TakeIfUpgrade()` (line ~483)**
   Early-returns when `pbx_keep_all_weapons` is `true`, preventing individual weapon scripts from stripping old weapons during upgrade state sequences.

3. **`BaseWeapon_Functions.zsc` — `PB_SelectIfUpgrade()` (line ~499)**
   Early-returns when `pbx_keep_all_weapons` is `true`, preventing forced weapon-switch to the upgrade.

All three sites use the same CVar-check pattern:

```zscript
if(invoker && invoker.owner && invoker.owner.player && Cvar.GetCVar("pbx_keep_all_weapons", invoker.owner.player).GetBool())
    return;
```

The `HandlePickup` variant reads via `owner` directly (not `invoker`) because it executes in the context of the existing inventory item, not an action function.

### Addon Stack & VFS Ownership

The full runtime stack loads in this order: **PB → Glory Kills → Monster Pack → KeepWeapons**. Several files under `zscript/Weapons/` are provided by more than one layer of that stack; GZDoom's VFS only compiles the last-loaded archive's copy of a given path, so the table below tracks which layer's version actually wins at runtime:

| File | Provided by | Winner |
|---|---|---|
| `zscript/Weapons/BaseWeapon.zc` | PB, KeepWeapons | KeepWeapons |
| `zscript/Weapons/BaseWeapon_Functions.zsc` | PB, Monster Pack, KeepWeapons | KeepWeapons |
| `zscript/Weapons/BaseWeapon_Melee.zsc` | PB, Glory Kills | Glory Kills |
| `zscript/Weapons/BaseWeapon_Executions.zsc` | PB, Monster Pack | Monster Pack |
| `zscript/Weapons/BaseWeapon_Barrels.zsc` | PB only (new file, Aug 2026) | PB |
| `zscript/Weapons/BaseWeapon_Equipment.zsc` | PB only (new file, Sep 2026) | PB |
| `zscript/Weapons/Keepweapmanager.zsc` | PB only (new file, Sep 2026) | PB |

**Cross-addon staleness note (re-verified Sep 2026)**: Glory Kills was refreshed locally on 2026-09-13 (commit `8049fc2`, "Sync with PB_Staging: restore PB's bespoke executions under GloryKills"). Re-diffing its `zscript/Weapons/BaseWeapon_Melee.zsc` against the current PB upstream (CRLF/LF-normalized) shows it is current: it already uses `PB_BarrelToken` everywhere it checks for a held barrel, and differs from upstream only in its intentional changes (the `#Include` of a new `BaseWeapon_Glorykill.zsc`, a commented-out legacy `QuickMelee` label, and a trailing-newline difference). The previous staleness claim about this file no longer applies.

However, the same refresh introduced a new file, `zscript/GloryKills/BaseWeapon_Glorykill.zsc`, which doesn't collide with any KeepWeapons/PB path (so it raises no VFS question) but does deepen the existing cross-addon coupling: it also `extend class PB_WeaponBase`, and its `GK_ExecutionHandlerString()` default case calls `PB_ExecutionHandlerString(monster)` — the function KeepWeapons owns via `BaseWeapon_Functions.zsc`'s VFS win. More notably, that new file's own `PB_ExecuteGK()` function still gates on the **legacy** `GrabbedBarrel`/`GrabbedFlameBarrel`/`GrabbedIceBarrel` inventory tokens rather than the `PB_BarrelToken` class — i.e., the same class of staleness the old note used to describe for `BaseWeapon_Melee.zsc`, just relocated to this new file. Those legacy token actors still exist upstream, so it won't hard-error, but Glory Kill barrel-holding detection in this new path may silently stop matching. This remains a Glory Kills concern, not a KeepWeapons one, but is flagged here for the same reason as before: the coupling runs through a function this addon owns.

## Class Hierarchy

```
DoomWeapon (engine)
└── PB_WeaponBase            — main weapon base (BaseWeapon.zc), extended by BaseWeapon_Functions.zsc
    ├── PB_Weapon             — empty subclass used as an inheritance anchor
    │   └── PB_DualWeapon     — dual-wield weapon base
    └── (individual PB weapons inherit from PB_Weapon or PB_WeaponBase)

CustomInventory (engine)
└── PB_UpgradeItem            — pickup for weapon upgrades

Ammo (engine)
└── PB_WeaponAmmo             — magazine/internal ammo tracking
```

## Conventions

### Naming

| Element          | Convention                                | Examples                                      |
|------------------|-------------------------------------------|-----------------------------------------------|
| Classes          | `PB_` prefix, PascalCase                  | `PB_WeaponBase`, `PB_DualWeapon`              |
| Functions        | `PB_` prefix, PascalCase                  | `PB_TakeIfUpgrade`, `PB_CheckReload`          |
| Properties       | PascalCase (declared via `property`)      | `UpgradedWeapon`, `WheelInfo`                 |
| Member variables | camelCase                                 | `barrelHeat`, `chamberEmpty`, `akimboMode`    |
| CVars (addon)    | `pbx_` prefix, snake_case                 | `pbx_keep_all_weapons`                        |
| CVars (PB)       | `pb_` prefix, snake_case                  | `pb_toggle_aim_hold`                          |
| File extensions  | `.zc` for class defs, `.zsc` for extends  | `BaseWeapon.zc`, `BaseWeapon_Functions.zsc`   |

### Code Style

- **Indentation**: Tabs (not spaces).
- **Braces**: Opening brace on the same line as the declaration for functions and control flow. Class-level braces on their own line.
- **State definitions**: Follow GZDoom actor-state syntax (`Label: SPRT FRAME duration { ... }`).
- **CVar access**: Always use `Cvar.GetCVar("name", player).GetBool()` (or `.GetFloat()`, `.GetInt()`). Never cache a CVar handle across tics unless using the `transient` qualifier.
- **`extend class`**: `BaseWeapon_Functions.zsc` uses `extend class PB_WeaponBase` to add methods without modifying the primary class file. Prefer this pattern for adding functionality.
- **Action functions**: Functions called from weapon states use the `action` qualifier (e.g., `action void PB_TakeIfUpgrade(...)`). Inside action functions, use `invoker` to reference the weapon instance and `self` to reference the owning actor.

### Upstream Compatibility

These files are **full overrides** of PB upstream files from the `PB_Staging` branch.

#### BaseWeapon.zc (two-way merge: PB upstream + KeepWeapons)

If PB updates `BaseWeapon.zc`, this addon must be rebased:

1. Download the upstream file via raw HTTP (e.g., `Invoke-WebRequest` or `curl`) — do **not** rely on web-rendered content, as angle-bracket type parameters like `class<Ammo>` get silently stripped as HTML tags. The correct raw path is `https://raw.githubusercontent.com/pa1nki113r/Project_Brutality/PB_Staging/zscript/Weapons/BaseWeapon.zc` (capital `Weapons`).
2. Re-apply the `HandlePickup()` CVar guard.
3. Preserve all other upstream changes verbatim — match upstream byte-for-byte except for the guard. The simplest reliable method is to overwrite with upstream and re-apply only the three guard lines (the `keepAllWeapons` bool, the `if(!keepAllWeapons){...}` wrap around the `DropInventory` loop, and the `!keepAllWeapons &&` prefix on the duplicate-pickup `else if`).

**Overlay-layer constants**: As of the May 2026 PB_Staging update, `BaseWeapon.zc` references named overlay constants (`PSP_LEDGEGRAB`, `PSP_QUICKMELEE`, `PSP_MELEEEQHANDLER`, etc.) that are defined by the global `enum PB_OverlayLayers` at the top of `BaseWeapon_Functions.zsc`. Both files must be rebased together. Note that upstream's own `BaseWeapon.zc` is internally inconsistent (e.g., it uses `PSP_FIRSPERSONLEGS` in only one spot and hardcoded `-1000` elsewhere) — do not "tidy" these; mirror upstream exactly to avoid divergence noise on future rebases.

#### BaseWeapon_Functions.zsc (three-way merge: PB upstream + Monster Pack + KeepWeapons)

This file is a **three-way merge** of:
- **PB upstream** — the base code
- **Monster Pack** — adds ~1,200 lines of execution handler functions (28 `PB_Execution*()` handler functions and 35 `case` entries in `PB_ExecutionHandlerString()`, as of the 30-Jul-2026 build) for custom monster melee executions
- **KeepWeapons** — adds the two CVar guards in `PB_TakeIfUpgrade()` and `PB_SelectIfUpgrade()`

**VFS conflict**: Both KeepWeapons and Monster Pack provide `zscript/Weapons/BaseWeapon_Functions.zsc`. GZDoom's VFS uses the last-loaded archive's version. Since KeepWeapons must load after Monster Pack, KeepWeapons' copy wins. If KeepWeapons' copy does not include Monster Pack's execution handlers, Monster Pack enemies will go invisible during melee executions (the monster dies into an invisible `TNT1` state but no correct execution puppet is spawned because `PB_ExecutionHandlerString` falls through to generic).

**Rebasing procedure** when PB or Monster Pack updates:

1. Start from Monster Pack's version of `BaseWeapon_Functions.zsc` (which already includes PB upstream + execution handlers). If only PB updated (no new Monster Pack release), start from the PB raw file at `https://raw.githubusercontent.com/pa1nki113r/Project_Brutality/PB_Staging/zscript/Weapons/BaseWeapon_Functions.zsc` (capital `Weapons`) and re-inject the Monster Pack content (see below).
2. Download via raw HTTP — do **not** rely on web-rendered content, as angle-bracket type parameters like `class<Ammo>` get silently stripped as HTML tags.
3. Re-apply the two CVar guards in `PB_TakeIfUpgrade()` and `PB_SelectIfUpgrade()`.
4. Preserve all Monster Pack execution handler code and all other upstream changes verbatim. The Monster Pack content lives in **two interleaved locations**, not one appended block: (a) the ~28 `PB_Execution*()` handler functions inserted between `PB_ExecuteCacodemon()` and `PB_ExecuteShotguny()`, and (b) the extra `case` entries inside `PB_ExecutionHandlerString()` (between the base `PB_Cacodemon` case and `default:`). When rebasing onto a fresh PB pull, take the new upstream wholesale (it may have added top-of-file code such as the `PB_OverlayLayers` enum and the `PB_ReadyFire` / `PB_SetZoom` / `PB_ClearDualWield` / `PB_SetupDualWield` action functions) and inject only these two Monster Pack blocks plus the two CVar guards.

> **May 2026 rebase note**: PB_Staging refactored this file, adding `enum PB_OverlayLayers` and the `PB_ReadyFire`/`PB_SetZoom`/`PB_ClearDualWield`/`PB_SetupDualWield` functions near the top of `extend class PB_WeaponBase`. A stale KeepWeapons override that lacked these caused 37 compile errors (`Unknown identifier 'PSP_LEFTGUN'`, `PB_SetZoom: action function not found`, etc.). The fix was a full rebase onto the new upstream with the Monster Pack blocks and CVar guards re-injected.

> **August 2026 rebase note** (PB_Staging commit `1b5fdfb`, 2026-08-14): Several breaking upstream changes landed in this rebase. (1) **Barrel system rewrite** — a new `BaseWeapon_Barrels.zsc` and `PB_BarrelToken` class (`TYPE_EXPLOSIVE`/`TYPE_BURNING`/`TYPE_FROZEN`) replaced the old `HasBarrel`/`HasFlameBarrel`/`HasIceBarrel`/`Grabbed*Barrel` inventory tokens and `ReadyFlameBarrel`/`ReadyIceBarrel` states with a single `RaiseBarrel` → `ReadyBarrel` path; `PB_WeaponBase` gained a `bool hasBarrel` member, and `HandlePickup`'s ammo transfer is now clamped to `maxamount`. (2) **`QueueSmoke` signature change** — `NashGoreStatics.QueueSmoke()` now takes an `Actor self` parameter, and its five call sites moved out of this file into `zscript/Effects/Smoke.zs`; since KeepWeapons' guards don't touch these call sites, taking upstream's version wholesale was sufficient. (3) **Dual-wield tri-state fix** — `FiringLeftWeapon`/`FiringRightWeapon` changed from `bool` to a tri-state `int`, with new `A_RefireLeft`/`A_RefireRight` action functions and rewritten `A_DoPBLeftAction`/`A_DoPBRightAction`. (4) **Deliberate deviation from upstream** — at commit `1b5fdfb`, upstream's new `default:` case in `PB_ExecutionHandlerString()` was already a guarded ternary — `return (monster.FindState("Execution") || monster.FindState("Death.Execution")) ? PB_ExecuteGeneric() : PB_ExecuteFast();` — where only the else-branch reaches `PB_ExecuteFast()` (backed by new `Execution_Fast` states and a `PB_RotateCamera()` helper). KeepWeapons' `default:` instead called `PB_ExecuteGeneric()` unconditionally, because Monster Pack ships its own older `BaseWeapon_Executions.zsc` (which wins the VFS over PB's copy, per the [ownership table above](#addon-stack--vfs-ownership)) and that older file lacked the `Execution_Fast`/`Fast1`–`Fast3` states `PB_ExecuteFast()` requires — dispatching to it would have errored. This deviation was marked with a code comment in `BaseWeapon_Functions.zsc`. (Superseded by the September 2026 rebase note below, which replaced the unconditional call with a runtime probe.) (5) Monster Pack's 30-Jul-2026 build added a 28th execution handler, `PB_ExecuteD16Cyberdemon()` / `case 'D16Cyberdemon':`, re-injected alongside the other 27. **Verification**: both files were confirmed via a live compile test using a fresh upstream PB_Staging clone, Glory Kills, and the 30-Jul-2026 Monster Pack build, in both with- and without-Monster-Pack configurations — both reached `TITLEMAP - PB_Introduction` with zero ZScript errors/warnings in the log, and an in-game `pb_smg` upgrade pickup confirmed the CVar guard still works correctly. Note: on this UZDoom 4.14.3 macOS build, the "script parsing took NNNN ms" line reports `0.00 ms` (it's a DECORATE-parsing metric, not ZScript compile time, on this platform) — treat reaching `TITLEMAP - PB_Introduction` with a clean log as the reliable success signal instead of that timing number.

> **September 2026 rebase note**: PB's `ZSCRIPT.zc` bumped its version directive from `version "4.14.2"` to `version "5.0"` — **UZDoom 5.0 is now a hard compile prerequisite**; the mod will not compile at all on 4.14.3. UZDoom 5.0.1 was installed locally and used for this rebase's verification. Against that baseline, the drift in the two files we own was small: (1) two stale `Spin`-token lines that upstream removed after commit `1b5fdfb` — `A_TakeInventory("Spin",1);` in `BaseWeapon.zc` and `A_SetInventory("Spin",0);` in `BaseWeapon_Functions.zsc` — were deleted to restore byte-for-byte parity. (2) PB added two new upstream-only files under `zscript/Weapons/` — `BaseWeapon_Equipment.zsc` (equipment wheel / grenade / mine / leech / rev-launcher states) and `Keepweapmanager.zsc` (a `KeepWeapManager` inventory class whose weapon-upgrade logic is entirely commented out and whose `pb_keepweapons` CVar no longer exists in PB's `CVARINFO`, so it does not compete with `pbx_keep_all_weapons`) — both PB-only per the [ownership table above](#addon-stack--vfs-ownership), requiring no code changes. (3) The `default:` case in `PB_ExecutionHandlerString()` was changed from the unconditional `return PB_ExecuteGeneric();` (see item 4 of the August note above) to an adaptive runtime probe: `return (monster.FindState("Execution") || monster.FindState("Death.Execution") || !invoker.FindState("Execution_Fast")) ? PB_ExecuteGeneric() : PB_ExecuteFast();`. This is now more valuable than a hardcoded deviation because Monster Pack is currently **unverified** against this update (see the Testing section below) — the adaptive guard makes the file self-correct for either a PB-only stack (where `Execution_Fast` resolves and matches upstream behavior exactly) or a Monster-Pack-loaded stack (where the probe fails and it falls back to the previously-shipped `PB_ExecuteGeneric()` behavior), rather than depending on a test that could not be run at rebase time. **Verification**: compile-tested PB + Glory Kills + KeepWeapons (Monster Pack deliberately excluded, see Testing section) on UZDoom 5.0.1 — reached `TITLEMAP - PB_Introduction` with zero `Script error` lines (only 3 pre-existing PB-side `Dictionary` deprecation warnings, unrelated to KeepWeapons). The `script parsing took NNNN ms` line still reports `0.00 ms` on 5.0.1, so the August note's reasoning for treating `TITLEMAP - PB_Introduction` as the authoritative success signal still holds.

## Development Guidelines

### Adding New Behavior

When adding new CVar-gated behavior:

1. Declare the CVar in `CVARINFO` using the `user` scope (per-player):
   ```
   user bool pbx_my_feature = false;
   ```
2. Use the `pbx_` prefix to distinguish addon CVars from PB's `pb_` CVars.
3. Guard the behavior with the standard null-safe CVar check pattern shown above.
4. Minimize divergence from upstream — touch only the lines necessary to implement the feature.

### Testing

- Load order matters: the full stack loads as **PB → Glory Kills → Monster Pack → KeepWeapons**, with this addon loading last. See [Addon Stack & VFS Ownership](#addon-stack--vfs-ownership) for why.
- Test with `pbx_keep_all_weapons` both `true` and `false` to verify the toggle works and that `false` preserves stock PB behavior.
- Test upgrade pickups, downgrade paths, dual-wield tokens, and ammo transfer.
- **Monster Pack verification is currently deferred** (as of the September 2026 rebase): its author has not yet shipped a build against the new PB_Staging update, so testing melee executions on Monster Pack enemies (e.g., Behemoth, Hell Duke, Marauder) isn't meaningful right now — the stale 30-Jul-2026 build would be tested against upstream it predates. The addon's ~1,265 lines of Monster Pack execution-handler code in `BaseWeapon_Functions.zsc` remain in place untouched; they compile cleanly under the current ZScript version (confirmed by the September compile test) but are compiled-but-unexercised until an updated Monster Pack build lands. Resume full four-layer testing, including melee executions on Monster Pack enemies, once that build is available.

## Reference Documentation

These resources are authoritative for understanding and modifying ZScript/DECORATE code targeting GZDoom-family ports:

### ZScript & Engine

- **ZScript reference (primary)**: <https://github.com/zdoom-docs/stable>
- **UZDoom source (versioned engine context)**: <https://github.com/UZDoom/UZDoom/tree/5.0> (the `UZDoom/UZDoom` repo has no `5.0.1` tag or branch as of Sep 2026 — its patch releases aren't separately tagged, so the `5.0` branch is the closest available ref for the 5.0.1 build in use; check for a `5.0.1` tag again on the next rebase)

### DECORATE & Actor Reference

- **DECORATE format specifications**: <https://zdoom.org/w/index.php?title=DECORATE_format_specifications>
- **Action functions**: <https://zdoom.org/w/index.php?title=Action_functions>
- **Classes**: <https://zdoom.org/w/index.php?title=Classes>
- **Actor flags**: <https://zdoom.org/w/index.php?title=Actor_flags>
- **Actor properties**: <https://zdoom.org/w/index.php?title=Actor_properties>
- **Actor states**: <https://zdoom.org/w/index.php?title=Actor_states>
- **DECORATE expressions**: <https://zdoom.org/w/index.php?title=DECORATE_expressions>

### Upstream Mod

- **Project Brutality (PB_Staging branch)**: <https://github.com/pa1nki113r/Project_Brutality/tree/PB_Staging>
