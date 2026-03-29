# Plasma 6 Wayland — Fractional Scaling Not Working Properly

## The Problem

On some distros, setting fractional scaling (e.g. 125%, 150%) in Plasma's
System Settings → Display does scale UI elements up — but doesn't make them
crisp. Everything looks enlarged but blurry, as if the display resolution
itself wasn't changed, just the content was stretched. This is because the
Plasma GUI setting alone doesn't always properly propagate the scale factor
to Qt apps at the DPI level.

**Affects:** Plasma 6 on Wayland, particularly on Arch-based distros (CachyOS,
EndeavourOS, Manjaro) and others where the GUI scaling toggle doesn't fully
apply.

---

## Why It Happens

The Plasma fractional scaling setting tells the compositor to scale, but Qt
apps also have their own internal scale factor handling. When the two don't
agree, Qt ends up rendering at the wrong effective resolution and then getting
upscaled by the compositor — resulting in blurry output. Forcing the policy
via an environment variable makes Qt handle scaling itself correctly from the
start, bypassing the miscommunication.

---

## The Fix

Set `QT_SCALE_FACTOR_ROUNDING_POLICY` in your shell config so Qt applies
scaling properly on every launch.

### Fish (simplest)

Add to `~/.config/fish/config.fish`:

```fish
set -gx QT_SCALE_FACTOR_ROUNDING_POLICY RoundPreferFloor
```

### Bash

Add to `~/.bashrc`:

```bash
export QT_SCALE_FACTOR_ROUNDING_POLICY=RoundPreferFloor
```

### Zsh

Add to `~/.zshrc`:

```bash
export QT_SCALE_FACTOR_ROUNDING_POLICY=RoundPreferFloor
```

After adding the line, either reboot or re-login for it to take full effect
across all apps.

---

## All Available Policy Options

If `RoundPreferFloor` doesn't look right on your specific scale factor, try
the others:

| Policy | Behaviour |
|--------|-----------|
| `RoundPreferFloor` | Rounds to nearest integer, prefers rounding down on 0.5. **Most widely recommended.** |
| `Round` | Standard rounding — 0.5 and above rounds up. Qt's built-in default. |
| `Floor` | Always rounds down aggressively. May make things slightly smaller but very consistent. |
| `Ceil` | Always rounds up. Can cause slight clipping but everything is sharp. |
| `PassThrough` | No rounding — exact fractional value used. Only works well on displays and apps that fully support it. |

`RoundPreferFloor` works for most setups. If you're on an exact 150% or 200%
scale, `Floor` or `Round` may look equally good.

---

## Verify It's Applied

After reboot, open a terminal and run:

**Fish:**
```fish
echo $QT_SCALE_FACTOR_ROUNDING_POLICY
```

**Bash / Zsh:**
```bash
echo $QT_SCALE_FACTOR_ROUNDING_POLICY
```

Should output `RoundPreferFloor`.

---

## Notes

- This only affects **Qt apps**. GTK apps (Firefox, GIMP, Nautilus etc.) have
  their own scaling pipeline and won't be fixed by this — those need
  `GDK_SCALE` or similar if they're also blurry.
- The Plasma GUI scaling setting in System Settings should still be set to your
  desired percentage — this env var works alongside it, not instead of it.
- This is a workaround. A proper fix would require Qt or Plasma to correctly
  propagate the scale factor without needing manual override.

---

## Status

- **Upstream fix:** Not yet resolved as of early 2026
- **Workaround:** `QT_SCALE_FACTOR_ROUNDING_POLICY=RoundPreferFloor`
- **Tested on:** CachyOS, Arch, EndeavourOS — Plasma 6 + Wayland
