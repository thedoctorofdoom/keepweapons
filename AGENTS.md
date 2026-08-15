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

**Known cross-addon staleness note**: Glory Kills' `BaseWeapon_Melee.zsc` was last synced before PB's Aug 2026 barrel rewrite (see rebase note below) and still checks the old `GrabbedBarrel`/`GrabbedFlameBarrel`/`GrabbedIceBarrel` tokens rather than the new `PB_BarrelToken` class. Those legacy token actors still exist upstream, so this won't hard-error, but barrel-holding melee detection may silently stop matching. This is a Glory Kills rebase concern, not a KeepWeapons one — but it's flagged here because Glory Kills' execution path calls `PB_ExecutionHandlerString(monster)`, which lives in the file KeepWeapons owns (`BaseWeapon_Functions.zsc`), coupling the two addons even though neither owns the other's file.

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

> **August 2026 rebase note** (PB_Staging commit `1b5fdfb`, 2026-08-14): Several breaking upstream changes landed in this rebase. (1) **Barrel system rewrite** — a new `BaseWeapon_Barrels.zsc` and `PB_BarrelToken` class (`TYPE_EXPLOSIVE`/`TYPE_BURNING`/`TYPE_FROZEN`) replaced the old `HasBarrel`/`HasFlameBarrel`/`HasIceBarrel`/`Grabbed*Barrel` inventory tokens and `ReadyFlameBarrel`/`ReadyIceBarrel` states with a single `RaiseBarrel` → `ReadyBarrel` path; `PB_WeaponBase` gained a `bool hasBarrel` member, and `HandlePickup`'s ammo transfer is now clamped to `maxamount`. (2) **`QueueSmoke` signature change** — `NashGoreStatics.QueueSmoke()` now takes an `Actor self` parameter, and its five call sites moved out of this file into `zscript/Effects/Smoke.zs`; since KeepWeapons' guards don't touch these call sites, taking upstream's version wholesale was sufficient. (3) **Dual-wield tri-state fix** — `FiringLeftWeapon`/`FiringRightWeapon` changed from `bool` to a tri-state `int`, with new `A_RefireLeft`/`A_RefireRight` action functions and rewritten `A_DoPBLeftAction`/`A_DoPBRightAction`. (4) **Deliberate deviation from upstream** — upstream's new `default:` case in `PB_ExecutionHandlerString()` dispatches to `PB_ExecuteFast()` (backed by new `Execution_Fast` states and a `PB_RotateCamera()` helper), but KeepWeapons' `default:` still calls `PB_ExecuteGeneric()` instead, because Monster Pack ships its own older `BaseWeapon_Executions.zsc` (which wins the VFS over PB's copy, per the [ownership table above](#addon-stack--vfs-ownership)) and that older file lacks the `Execution_Fast`/`Fast1`–`Fast3` states `PB_ExecuteFast()` requires — dispatching to it would error. This deviation is marked with a code comment in `BaseWeapon_Functions.zsc`. (5) Monster Pack's 30-Jul-2026 build added a 28th execution handler, `PB_ExecuteD16Cyberdemon()` / `case 'D16Cyberdemon':`, re-injected alongside the other 27. **Verification**: both files were confirmed via a live compile test using a fresh upstream PB_Staging clone, Glory Kills, and the 30-Jul-2026 Monster Pack build, in both with- and without-Monster-Pack configurations — both reached `TITLEMAP - PB_Introduction` with zero ZScript errors/warnings in the log, and an in-game `pb_smg` upgrade pickup confirmed the CVar guard still works correctly. Note: on this UZDoom 4.14.3 macOS build, the "script parsing took NNNN ms" line reports `0.00 ms` (it's a DECORATE-parsing metric, not ZScript compile time, on this platform) — treat reaching `TITLEMAP - PB_Introduction` with a clean log as the reliable success signal instead of that timing number.

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

- Load order matters: the full stack must load as **PB → Glory Kills → Monster Pack → KeepWeapons**, with this addon loading last. See [Addon Stack & VFS Ownership](#addon-stack--vfs-ownership) for why.
- Test with `pbx_keep_all_weapons` both `true` and `false` to verify the toggle works and that `false` preserves stock PB behavior.
- Test upgrade pickups, downgrade paths, dual-wield tokens, and ammo transfer.
- Test melee executions on Monster Pack enemies (e.g., Behemoth, Hell Duke, Marauder) to verify execution puppets spawn correctly and enemies do not vanish.

## Reference Documentation

These resources are authoritative for understanding and modifying ZScript/DECORATE code targeting GZDoom-family ports:

### ZScript & Engine

- **ZScript reference (primary)**: <https://github.com/zdoom-docs/stable>
- **UZDoom source (versioned engine context)**: <https://github.com/UZDoom/UZDoom/tree/4.14.3>

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
