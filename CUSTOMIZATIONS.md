# Customizations (fork of MohammedEl-sayedAhmed/clipman)

Personal fork of [clipman](https://github.com/MohammedEl-sayedAhmed/clipman) (Apache-2.0)
patched to run on **Bluefin-DX (Fedora atomic), GNOME Shell 50, Wayland**.
Upstream targets GNOME 45–48 and assumes `wtype` works — neither holds here.

## Why these patches

### 1. `wtype` is dead on GNOME/Mutter Wayland
`wtype` needs the `zwp_virtual_keyboard_manager_v1` protocol, which Mutter
refuses for security. Every `wtype` call returns:

```
Compositor does not support the virtual keyboard protocol   (exit 1)
```

So auto-paste never fired. Upstream's paste loop treated "process ran" as
success and short-circuited on `wtype`, never reaching the `ydotool` fallback.

### 2. `ydotool key` takes keycodes, not names
ydotool 1.0.4's `key` subcommand wants raw `<keycode>:<pressed>` pairs.
Upstream passed `ydotool key ctrl+v` — a non-interpretable arg that ydotool
silently turns into a delay (exit 0, **emits nothing**). Looked like success,
pasted nothing.

## Patches (`clipman/window.py`)

| Change | From | To |
|---|---|---|
| `_PASTE_COMMANDS` ydotool ctrl-v | `["ydotool","key","ctrl+v"]` | `["ydotool","key","29:1","47:1","47:0","29:0"]` |
| ydotool ctrl-shift-v | `ctrl+shift+v` | `29:1 42:1 47:1 47:0 42:0 29:0` |
| ydotool shift-insert | `shift+Insert` | `42:1 110:1 110:0 42:0` |
| `_simulate_paste` loop | return on any run | return only on `rc == 0`, else fall through to next backend |
| paste delay (`_paste_entry`, `_paste_snippet`) | `80` ms | `150` ms (focus-return settle on Mutter) |

Keycodes: ctrl=29, shift=42, v=47, Insert=110.

## Patch (`extension/metadata.json`)
`shell-version` extended `45–48` → `45–50`. The shell extension only does
`Meta.Selection` signal detection, so the small API surface survives the
45→50 jump; the GTK4 daemon is shell-version-independent.

## Runtime requirement (not code)
Auto-paste needs **ydotoold running** + the user in the **`input` group**
(for `/dev/uinput`). See `INSTALL-BLUEFIN.md`.
