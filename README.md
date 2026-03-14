# Keep Weapons for Project Brutality

`Keep Weapons for Project Brutality` is a small ZScript addon for `Project Brutality 0.4.1` that changes how weapon upgrades behave.

By default, Project Brutality replaces certain base weapons when you pick up their upgraded versions. This addon prevents that removal, so you can keep both the original weapon and its upgraded counterpart in your inventory at the same time.

## What This Mod Does

When enabled, the mod changes upgrade behavior so that:

- Picking up an upgraded weapon no longer strips the base weapon from your inventory.
- Picking up an upgrade no longer forces you to lose access to the older version.
- Upgrade helper functions used by PB weapon scripts stop removing or auto-selecting replacement weapons.

In practice, this means you can keep and switch between both versions instead of being locked into the upgraded one.

## How It Works

This addon overrides Project Brutality weapon base code in two files:

- `zscript/Weapons/BaseWeapon.zc`
- `zscript/Weapons/BaseWeapon_Functions.zsc`

The behavior is controlled by the per-player CVar:

```text
pbx_keep_all_weapons
```

When that CVar is `true`, the addon intercepts the usual upgrade flow in three places:

1. `HandlePickup()` skips the logic that drops the old weapon when its upgrade is collected.
2. `PB_TakeIfUpgrade()` returns early so upgrade states do not remove the previous weapon.
3. `PB_SelectIfUpgrade()` returns early so the game does not force a switch to the upgraded weapon.

The internal CVar name remains `pbx_keep_all_weapons` so existing configs and console commands continue to work.

## Installation

1. Load `Project Brutality` normally.
2. Load this addon after Project Brutality.

Load order matters: this addon must have higher priority than PB so its overridden weapon scripts take effect.

## Usage

The mod is enabled by default through `CVARINFO`:

```text
user bool pbx_keep_all_weapons = true;
```

You can control it at runtime from the console:

```text
set pbx_keep_all_weapons true
set pbx_keep_all_weapons false
```

- `true`: keep both base and upgraded weapons
- `false`: use normal Project Brutality behavior

Because the CVar is declared as a `user` CVar, the setting is stored per player.

## Compatibility Notes

- This addon is meant for `Project Brutality 0.4.1`.
- It intentionally overrides upstream PB weapon base scripts.
- If PB changes `BaseWeapon.zc` or `BaseWeapon_Functions.zsc` in a future update, this addon may need to be rebased against the new upstream files.

## Included Files

- `CVARINFO`
- `zscript/Weapons/BaseWeapon.zc`
- `zscript/Weapons/BaseWeapon_Functions.zsc`

## Summary

If you want Project Brutality upgrades to add options instead of replacing them, `Keep Weapons for Project Brutality` does exactly that while still letting you toggle the behavior on or off with a single CVar.
