# mac-display

Turns this PC's DP-0 monitor into an extra display for the MacBook Air (M2),
by streaming a BetterDisplay virtual screen over RustDesk on the office cable LAN.

## Layout

| Path | Installed as | What |
|---|---|---|
| `bin/mac-display` | `~/bin/mac-display` (symlink) | One-click toggle the launcher runs: picks the route, starts the codec server, connects, picks the display, goes fullscreen |
| `bin/mac-monitor` | `~/bin/mac-monitor` (symlink) | Opens the RustDesk session and parks it on the monitor |
| `bin/rd-pick-display` | `~/bin/rd-pick-display` (symlink) | Switches the session to the Mac's display N by driving the RustDesk toolbar, verified from the toolbar's own state |
| `bin/mac-display-watchdog` | `~/bin/mac-display-watchdog` (symlink) | Started per session: reconnects on a frozen picture, and moves a tailnet session onto the cable |
| `bin/display-keeper` | `~/bin/display-keeper` (symlink) | Re-applies DP-0's mode after the panel powers back on |
| `linux/mac-monitor.desktop` | `~/Desktop/`, `~/.local/share/applications/` (symlinks) | Launcher icon |
| `linux/display-keeper.service` | `~/.config/systemd/user/` (symlink) | User unit for display-keeper |
| `mac/lan-split-routes` | `/usr/local/sbin/` on the Mac | Keeps 192.168.0.0/24 on the cable (see below) |
| `mac/local.lan-split-routes.plist` | `/Library/LaunchDaemons/` on the Mac | Runs it at boot and every 10 s |

Install the Mac side (on the Mac, from a copy of `mac/`):

```
sudo mkdir -p /usr/local/sbin
sudo install -m 755 -o root -g wheel lan-split-routes /usr/local/sbin/
sudo install -m 644 -o root -g wheel local.lan-split-routes.plist /Library/LaunchDaemons/
sudo launchctl bootstrap system /Library/LaunchDaemons/local.lan-split-routes.plist
```

## Why it is built this way

Findings behind the non-obvious parts, with how each was established.

- **Route: cable, not tailnet.** On 2026-10-20 the tailnet path ran through the
  Tokyo DERP relay at ~70 ms against 0.5 ms on the cable. `mac-display` tries
  the saved LAN address first; the watchdog moves a tailnet session to the
  cable once the LAN has answered for ~30 s.
- **Mac routing bug.** With an exit node and "Allow local network access",
  Tailscale routes every local subnet via the primary service's gateway. Wi-Fi
  (school network, 10.100.88.1) is primary, so 192.168.0.0/24 went out Wi-Fi
  and was lost. Confirmed by toggling the option: the route followed it
  (off: via utun4, on: via 10.100.88.1). Putting the cable first fixes it but
  breaks DERP (the wired network blocks it), so `lan-split-routes` adds two /25
  routes on the cable instead, which win by being more specific. Tailscale
  itself still relays PC<->Mac (its own traffic follows the primary
  interface); RustDesk, Syncthing and `ssh ...@192.168.0.242` do not use it.
- **Hardware decode.** RustDesk on Linux gets its hwcodec list from its own
  `--server` process over IPC; with none running every session decodes on the
  CPU (`Failed to get hwcodec config`). `mac-display` starts one; with
  `stop-service = 'Y'` it accepts no incoming connections.
- **60 fps.** `custom-fps` only applies when the peer's `image_quality` is
  `custom`; otherwise both ends fall back to 30 (client `src/client.rs`,
  `io_loop.rs`; server `video_qos.rs`, RustDesk 1.4.6).
- **Freezes.** When the client falls behind it asks the Mac to refresh; the
  Mac's video service then restarts about once a second without ever producing
  a first frame (`encode fail: no valid frame`), and only a reconnect clears it.
  In every logged session, `Refresh display N to reduce delay` in the client log
  meant this freeze, so the watchdog reconnects on that line.
- **Display picking.** The toolbar's "1|2" control only opens a dropdown; the
  dropdown buttons switch. The client log cannot confirm a switch back to a
  display already shown (no new `video handler` line), so the picker reads the
  highlighted segment instead.
- **Fullscreen.** Only RustDesk's own fullscreen hides its title bar; it restores
  that from `wm_RemoteDesktop` in the peer file, which is saved as false
  whenever a session is killed. `mac-display` re-seeds it before each connect.
- **Compositor.** `mac-display` sets `_NET_WM_BYPASS_COMPOSITOR` on the session
  window; KWin's `Compositing.active` went true -> false. Saves up to one frame
  (not measured end to end); may allow tearing.
- **Sharpness vs fps.** Matching the wired panel exactly needs the 5K HiDPI
  virtual screen (5120x2880); RustDesk on Linux handles that at ~21-30 fps
  because frames are copied back to the CPU for conversion. The current
  2560x1440 LoDPI screen reaches 60 fps but is upscaled 1.5x.
- **Monitors.** HDMI-0 and the Mac's wired panel are the same physical monitor
  (serial YPGP5JA000618), so DP-0 and HDMI-0 both sitting at 0,0 is harmless.
