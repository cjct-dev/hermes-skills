---
name: linux-media-player-control
description: "Use when controlling a local media player on Linux."
version: 1.1.0
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
- The user asks to build or load a playlist in a local player.
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
   Standard methods: `Play` `Pause` `PlayPause` `Next` `Previous` `Stop` `Raise` `Quit`. There is NO queue-clear or queue-replace method in MPRIS.

4. **Read state:**
   ```bash
   busctl --user call <busname> /org/mpris/MediaPlayer2 org.freedesktop.DBus.Properties Get ss org.mpris.MediaPlayer2.Player Metadata
   ```
   `xesam:title` / `xesam:album` / `xesam:url` carry the current track; read `PlaybackStatus` and `Position` the same way (one property per call). Always verify `PlaybackStatus` actually returns `Playing` after `Play` — a rc 0 from the method call alone means nothing.

5. **Queue one track at runtime (KDE single-instance):** invoking the player binary with a URI routes it to the running instance, appends it to the queue, and switches playback to it:
   ```bash
   /usr/bin/elisa "file:///path/to/track.mp3"
   ```
   The new process exits immediately (rc 0). Queue tracks this way IN ORDER to build a playlist on the fly. Each invocation jumps playback to the newly queued track, so the LAST queued track is the one playing — press `Previous` to rewind to the start of the queue.

6. **Build a persistent ordered playlist in Elisa (verified recipe):** Elisa exposes no MPRIS queue API, so write the persisted playlist state while the player is quit, relaunch, then `Play`:
   - Library DB: `~/.local/share/elisa/elisaDatabase.db`, table `Tracks` (ID, Title, ArtistName, AlbumTitle, TrackNumber, DiscNumber, FileName). `FileName` is a full `file://` URL in Elisa's own encoding — use it **verbatim**.
   - Quit Elisa (MPRIS `Quit` first; if the pid survives, `kill -TERM <pid>`). Back up the config first: `cp elisa.conf elisa.conf.bak-<purpose>`.
   - Write `~/.config/kde.org/elisa.conf` key `playListState` with PyQt6 (the value is a Qt QVariant; plain text edits are unreliable):
     ```python
     from PyQt6.QtCore import QSettings
     s = QSettings("/home/<user>/.config/kde.org/elisa.conf", QSettings.Format.IniFormat)
     entry = [str(id), title, artist, album, str(track), str(disc), "5", FileName]  # one per track, in order
     s.setValue("playListState", {"currentTrack": 0, "playList": entries,
                                  "randomMapping": [], "repeatMode": 0, "shuffleMode": 0})
     s.sync()
     ```
   - Relaunch Elisa, then MPRIS `Play`. It starts at `currentTrack` and steps through `playList` in written order (verify with `Next` + Metadata).

7. **Relaunching a GUI player from an agent shell** (agent shells usually lack display env vars):
   ```bash
   env DISPLAY=:0 WAYLAND_DISPLAY=wayland-1 XDG_RUNTIME_DIR=/run/user/<uid> \
       DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/<uid>/bus /usr/bin/elisa
   ```
   Run it as a background terminal call — it is a long-lived GUI process.

## Pitfalls

- MPRIS players live on the **user** session bus — always `busctl --user`; the system bus won't see them.
- `Metadata` embeds the album art as a huge base64 `data:image` URI — pipe through `head` or truncate before reading, or you'll dump megabytes of art into context.
- `playerctl` is not installed by default; on Arch it's `sudo pacman -S playerctl`, which may hit a sudo password prompt — hand the command to the user instead of retrying blind. Verify after install with `playerctl -l`.
- **`OpenUri` can be silently ignored** (rc 0, nothing queued, no log line — observed on recent Elisa). Prefer the single-instance invocation (step 5) or the config recipe (step 6); if you must use `OpenUri`, verify with `CanPlay`/`Metadata` afterwards.
- **URL-encoding breaks config-loaded playlists:** the `playListState` URL field must be the DB `FileName` verbatim (literal spaces, `%5B` as stored). Percent-encoding spaces (`%20`) makes Elisa treat the value as a relative path and fail with `cannot open file <music-root>file:///...` (the music root gets prepended to the URL).
- **No queue-clear exists.** To rebuild a queue from scratch: quit Elisa, rewrite `playListState`, relaunch — an old queue in the config is otherwise restored and appended to.
- Elisa's stderr is flooded with unrelated `RIST-ERROR` / rtsp retry noise — filter it out before reading for real errors.
- A player launched from an agent shell without the session env (step 7) may come up on the bus but refuse to start audio: stuck at `Stopped` with no `cannot open` errors means suspect the missing display/audio env, not the playlist.
- Electron apps (Brave, etc.) also register MPRIS names (`org.mpris.MediaPlayer2.brave...`) — pick the player by slug, not by position in the list.
- Don't treat `CanSeek=false` as a broken interface; it only disables seeking.

## Verification

- After `Play`: reading `PlaybackStatus` (step 4) returns `v s "Playing"`.
- `Next`/`Previous` while playing step through the queue; the returned `xesam:title` confirms queue order.
- `Raise` brings the player window to the front — a quick visual confirmation for the user.

## Known players

- **Elisa** (KDE): bus name `org.mpris.MediaPlayer2.elisa`; full `org.mpris.MediaPlayer2.Player` interface; config recipe (step 6) verified end-to-end — multi-track queue written from the DB, relaunched, `Play` → `Playing`, stepping through the queue in exact written order. `OpenUri` accepted-but-ignored on recent builds.
