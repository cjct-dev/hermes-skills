---
name: linux-media-player-control
description: "Use when controlling a local media player on Linux."
version: 1.0.0
author: cjct-dev
license: MIT
platforms: [linux]
metadata:
  hermes:
    tags: [linux, mpris, media-player, elisa, dbus, busctl, playerctl, music]
    requires_toolsets: [terminal]
---

# Linux Media Player Control (MPRIS)

## When to Use

- The user asks to play, pause, skip, stop, or queue music in a local player (Elisa, VLC, Cantata, ...).
- The user asks "is there a plugin for <player>?" or "can Hermes control my player" — this is the control path even when no catalog plugin exists.
- A cron job or routine needs to trigger playback.

Desktop media players on Linux (KDE/GNOME: Elisa, VLC, Cantata, ...) expose the MPRIS2 interface on the **user** DBus session bus. No plugin needed — drive it with `busctl --user`; if `playerctl` is installed, prefer it for cleaner one-liners (`playerctl --player elisa play`).

## Procedure

1. **Find the bus name** — never guess it:
   ```bash
   busctl --user list | grep -i mpris
   ```
   Names look like `org.mpris.MediaPlayer2.<slug>`. The slug is NOT the desktop-entry name — Elisa registers as `org.mpris.MediaPlayer2.elisa`, not `...org.kde.elisa` (introspecting the wrong name returns nothing). Always copy the exact name from this step.

2. **Verify the interface before calling:**
   ```bash
   busctl --user introspect <busname> /org/mpris/MediaPlayer2 | grep -E '^(method|property)'
   ```
   Playback methods need `CanControl=true`; if `CanSeek=false` (radio/live source or nothing loaded), skip `Seek`/`SetPosition`. The grep keeps output focused — the raw dump includes all DBus boilerplate interfaces.

3. **Call methods:**
   ```bash
   busctl --user call <busname> /org/mpris/MediaPlayer2 org.mpris.MediaPlayer2.Player <Method>
   ```
   Standard methods: `Play` `Pause` `PlayPause` `Next` `Previous` `Stop` `Raise` `Quit`, `OpenUri` (takes a `file://` URI — queues a local track), `Seek`/`SetPosition` (microseconds).

4. **Read state:**
   ```bash
   busctl --user call <busname> /org/mpris/MediaPlayer2 org.freedesktop.DBus.Properties Get ss org.mpris.MediaPlayer2.Player Metadata
   ```
   `mpris:track1` carries title/artist/album; read `PlaybackStatus` and `Position` the same way (one property per call).

## Pitfalls

- MPRIS players live on the **user** session bus — always `busctl --user`; the system bus won't see them.
- `Metadata` embeds the album art as a huge base64 `data:image` URI — pipe through `head` or truncate before reading, or you'll dump megabytes of art into context.
- `playerctl` is not installed by default; on Arch it's `sudo pacman -S playerctl`, which may hit a sudo password prompt — hand the command to the user instead of retrying blind. Verify after install with `playerctl -l`.
- `OpenUri` needs an absolute `file://` URI — confirm the exact filename (search the music dir) before calling; a wrong path just fails silently in the player.
- Electron apps (Brave, etc.) also register MPRIS names (`org.mpris.MediaPlayer2.brave...`) — pick the player by slug, not by position in the list.
- Don't treat `CanSeek=false` as a broken interface; it only disables seeking.

## Verification

- After `Play`: reading `PlaybackStatus` the same way as step 4 returns `v s "Playing"`.
- `Raise` brings the player window to the front — a quick visual confirmation for the user.

## Known players

- **Elisa** (KDE, `org.kde.elisa`): bus name `org.mpris.MediaPlayer2.elisa`; full `org.mpris.MediaPlayer2.Player` interface; `OpenUri` supported, `LoopStatus` writable, `CanSeek=false` while stopped.
