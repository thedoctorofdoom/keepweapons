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

1. **`BaseWeapon.zc` — `HandlePickup()` override (line ~118)**
   Reads `pbx_keep_all_weapons` from the owning player's CVars. When `true`:
   - Skips the `While(owner && self.amount > 0){ owner.DropInventory(self,1); }` loop that normally removes the base weapon on upgrade pickup.
   - Skips the `else if` branch that would consume a duplicate base-weapon pickup when the player already owns the upgrade.

2. **`BaseWeapon_Functions.zsc` — `PB_TakeIfUpgrade()` (line ~386)**
   Early-returns when `pbx_keep_all_weapons` is `true`, preventing individual weapon scripts from stripping old weapons during upgrade state sequences.

3. **`BaseWeapon_Functions.zsc` — `PB_SelectIfUpgrade()` (line ~406)**
   Early-returns when `pbx_keep_all_weapons` is `true`, preventing forced weapon-switch to the upgrade.

All three sites use the same CVar-check pattern:

```zscript
if(invoker && invoker.owner && invoker.owner.player && Cvar.GetCVar("pbx_keep_all_weapons", invoker.owner.player).GetBool())
    return;
```

The `HandlePickup` variant reads via `owner` directly (not `invoker`) because it executes in the context of the existing inventory item, not an action function.

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
| Properties       | PascalCase (declared via `property`)      | `UpgradedWeapon`, `DualWieldToken`            |
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

1. Download the upstream file via raw HTTP (e.g., `Invoke-WebRequest` or `curl`) — do **not** rely on web-rendered content, as angle-bracket type parameters like `class<Ammo>` get silently stripped as HTML tags.
2. Re-apply the `HandlePickup()` CVar guard.
3. Preserve all other upstream changes verbatim.

#### BaseWeapon_Functions.zsc (three-way merge: PB upstream + Monster Pack + KeepWeapons)

This file is a **three-way merge** of:
- **PB upstream** — the base code
- **Monster Pack** — adds ~1,200 lines of execution handler functions (27 `PB_Execution*()` handler functions and 31 `case` entries in `PB_ExecutionHandlerString()`) for custom monster melee executions
- **KeepWeapons** — adds the two CVar guards in `PB_TakeIfUpgrade()` and `PB_SelectIfUpgrade()`

**VFS conflict**: Both KeepWeapons and Monster Pack provide `zscript/Weapons/BaseWeapon_Functions.zsc`. GZDoom's VFS uses the last-loaded archive's version. Since KeepWeapons must load after Monster Pack, KeepWeapons' copy wins. If KeepWeapons' copy does not include Monster Pack's execution handlers, Monster Pack enemies will go invisible during melee executions (the monster dies into an invisible `TNT1` state but no correct execution puppet is spawned because `PB_ExecutionHandlerString` falls through to generic).

**Rebasing procedure** when PB or Monster Pack updates:

1. Start from Monster Pack's version of `BaseWeapon_Functions.zsc` (which already includes PB upstream + execution handlers).
2. Download via raw HTTP — do **not** rely on web-rendered content, as angle-bracket type parameters like `class<Ammo>` get silently stripped as HTML tags.
3. Re-apply the two CVar guards in `PB_TakeIfUpgrade()` and `PB_SelectIfUpgrade()`.
4. Preserve all Monster Pack execution handler code and all other upstream changes verbatim.

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

- Load order matters: this addon must load **after** both Project Brutality and Monster Pack.
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
