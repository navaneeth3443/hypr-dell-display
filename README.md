# hypr-dell-display

Use a Dell laptop as an on-demand extended display for a Hyprland desktop.
The second screen is a virtual output on the Legion, served over VNC and
viewed from the Dell with TigerVNC Viewer. Nothing runs at boot: the whole
setup is toggled with a keybind.

## Architecture

```
 Legion (this machine, Hyprland)
 ┌───────────────────────────────────────────────┐
 │ eDP-1        1920x1200@165, scale 1.25, 0x0   │  built-in laptop screen
 │ HEADLESS-2   1920x1080@60,  scale 1,   1536x0 │  virtual, created on demand
 │                      │                        │
 │                 wayvnc (0.0.0.0:5900)         │  serves HEADLESS-2 over VNC
 └──────────────────────┼────────────────────────┘
                        │  TCP 5900
               hotspot / direct Ethernet
                        │
 Dell (client)
   TigerVNC Viewer (vncviewer) -> "Legion-IP:5900"
```

- **HEADLESS-2** is a headless Hyprland output pinned at `1920x1080@60`,
  placed to the right of `eDP-1`. The `1536x0` position is the logical width
  of `eDP-1` (1920 / 1.25 scale).
- **wayvnc** streams that output on port `5900`, bound to `0.0.0.0` so a peer
  on the local link can connect. The toggle script logs wayvnc stderr to
  `~/.local/state/wayvnc-dell.log`, verifies the port is listening before
  reporting success, and retries startup three times.
- **tigervnc-viewer** (`vncviewer`) runs on the Dell and shows the stream as
  its second screen. A Hyprland window rule floats/centres windows whose
  class is `Vncviewer`.
- This is a **Lua-configured** Hyprland: live monitor rules use
  `hl.monitor({ ... })` and window moves use `hl.dsp.window.move({ ... })`.
  The legacy string dispatchers (`hyprctl keyword monitor`,
  `"movetoworkspacesilent, N"`) are rejected by this build.

## Keybindings

Bound in `~/.config/hypr/user.lua` (with `hl.unbind` first, because a bound
chord stacks handlers):

| Key | Script | Action |
|---|---|---|
| `Super + Shift + M` | `bin/toggle-dell-monitor` | Toggle the extended display. ON: create `HEADLESS-2`, apply geometry, start wayvnc. OFF: stop wayvnc, remove the output. |
| `Super + Shift + D` | `bin/todell` | Find `HEADLESS-2`'s active workspace and silently move the active window there without following it. Requires the display to be toggled ON. |

Turn the display ON on the Legion first, then connect from the Dell.

## Network: direct Ethernet or hotspot

**Direct peer-to-peer Ethernet (preferred).** The Legion's wired interface
is `enp8s0` (NetworkManager profile `Wired connection 1`) with a static
`10.0.0.1/24`; the Dell should be `10.0.0.2/24`. The link comes up when the
cable is connected. No router or internet is needed.

**Hotspot / LAN.** Both machines may instead join the same hotspot or
network. Find the Legion's address there (`hostname -I` or `ip -4 addr`)
and pass it through `LEGION_IP`. Allow TCP `5900` through the Legion's
firewall in either case.

`bin/connect-legion` runs on the Dell and opens the viewer. It has no
hardcoded address:

```bash
# direct Ethernet (default Legion address 10.0.0.1)
bin/connect-legion

# hotspot / LAN override
LEGION_IP=192.168.43.10 bin/connect-legion

# extra viewer flags are forwarded
LEGION_IP=192.168.43.10 bin/connect-legion -Fullscreen
```

## Contents

- `bin/toggle-dell-monitor` - create/remove `HEADLESS-2` and manage wayvnc.
- `bin/todell` - send the active window to the Dell workspace silently.
- `bin/connect-legion` - portable TigerVNC Viewer launcher (`LEGION_IP`,
  default `10.0.0.1`; optional `VNC_PORT`, default `5900`).
- `config/opencode.json` - OpenCode agent/permission config; verified to
  contain no API keys, tokens, passwords, or other secrets.

## Install

1. Copy the scripts to `~/.local/bin/` and make them executable:
   `install -m 755 bin/* ~/.local/bin/`.
2. Copy `config/opencode.json` to `~/.config/opencode/opencode.json`.
3. Bind the chords in `~/.config/hypr/user.lua` and pin the monitor rule in
   `~/.config/hypr/monitors_user.lua` as described above, then reload with
   `ryoku wm act config.reload`.
4. On the Dell, install TigerVNC Viewer and use `bin/connect-legion`.
