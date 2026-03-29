# Plasma 6 WiFi Crash Fix (CachyOS / Arch)

## The Problem

On Plasma 6 + Wayland, when connected to WiFi but with no actual internet (captive portal, router with no WAN, etc.), the entire Plasma shell crashes. The WiFi applet disappears and sometimes requires a hard reboot to recover.

**Affects:** Plasma 6 on Wayland, all distros. X11 is no longer an option in recent Plasma 6 builds.

---

## Why It Happens

NetworkManager (NM) periodically sends HTTP requests to a test URL to check for real internet connectivity — this is called **connectivity probing**. When it detects "connected but no internet," it fires a state change signal (`limited` or `portal`). The `plasma-nm` applet receives this signal and tries to update its UI. On Plasma 6 + Wayland, there is a bug where handling this specific state transition crashes the whole shell.

**Crash chain:**
```
NM probes internet → detects "no internet" → fires state signal → plasma-nm crashes Plasma shell
```

This is a known `plasma-nm` applet bug, not a NetworkManager bug. NM is doing its job correctly. The fix is to stop NM from sending the signal that triggers the bug.

---

## The Fix

Disable NetworkManager's connectivity probing via a drop-in config file.

```fish
echo -e "[connectivity]\nconnectivity-enabled=false" | sudo tee /etc/NetworkManager/conf.d/99-no-checks.conf
sudo systemctl restart NetworkManager
```

### Verify it worked

```fish
cat /etc/NetworkManager/conf.d/99-no-checks.conf
```

Should output:
```
[connectivity]
connectivity-enabled=false
```

Then check:
```fish
nmcli general
```

The `CONNECTIVITY` column should now show `unknown` instead of `full` or `limited`. This confirms NM has stopped probing and will no longer fire the crash-triggering signal.

---

## What This Changes

- NM stops sending HTTP probes to check for real internet
- Plasma no longer receives the `limited` state signal
- The "!" warning icon on the WiFi applet will no longer appear when internet is down
- **WiFi itself works exactly the same** — you just lose the visual "no internet" indicator

---

## What NOT To Do

These "fixes" were tried and caused more damage than the original problem:

| Fix | Why it's bad |
|-----|-------------|
| `kwriteconfig6 --file kded5rc --group Module-networkstatus --key autoload false` | Disables the KDE network status module entirely — WiFi applet vanishes from Plasma |
| Locking BSSID via `nmcli con edit` | Breaks NM connection config, causes applet to disappear |
| Disabling periodic WiFi scans (`iwd` or NM scan config) | Unrelated to the actual crash trigger |
| `modprobe -r iwlwifi` | Unloads the WiFi kernel driver — hardware stops working |

---

## Recovery (If WiFi Applet Disappears)

If the networkstatus module gets disabled accidentally:

```fish
# Re-enable networkstatus module
kwriteconfig6 --file kded6rc --group Module-networkstatus --key autoload true

# Restart user services
systemctl --user restart kded6.service

# Restart Plasma shell
killall plasmashell
plasmashell --replace &
```

Note: `qdbus6 org.kde.kded6 /kded org.kde.kded6.reloadConfig` will throw an error — that method doesn't exist in kded6. Ignore it, the config change from `kwriteconfig6` persists anyway.

---

## Status

- **KDE bug:** Open — no upstream patch as of Plasma 6.6.3
- **Workaround:** `connectivity-enabled=false` in NM conf (this document)
- **Distro:** CachyOS (Arch base), confirmed working
- **Plasma version:** 6.6.3
