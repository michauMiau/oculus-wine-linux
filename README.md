# Oculus on Linux via Wine

Trying to run Meta Horizon Link (Oculus PC Client) under Wine/Bottles on Linux, with the goal of using Revive to play Oculus Store games through SteamVR.

## Why?

Because Quest Link and Air Link require the native Oculus app — which you don't want if you already use ALVR / Steam Link / WIVRN for streaming. This is an attempt to get the client running under Wine so it can do its job (auth, DRM) without taking over your desktop like a regular Windows install would.

## Status

🚧 **Blocked.** Two main issues:
1. Installer reports "no disk space" even when there's plenty of room
2. `OVRService` fails to register as a Windows service — OpenSCManager RPC call returns error 0x6be, and Wine can't handle it yet

## What's in here

- **`Oculus-Wine-Progress.md`** — full installation log with every attempt, error, blocker, and dead end. Updated as things change.
- **`how-to-play-minecraft-vr-flatpak.md`** — because apparently that's also a thing now ☺️

## What I tried

- Bottles + Proton GE 11-1 → installer didn't launch (probably missing Steam Runtime)
- Bottles Flatpak + kron4ek-wine-11.3 → installer launches, fails mid-way with the two blockers above

## What's next

Figure out how to either fix the disk space detection or bypass it entirely, then deal with the OVRService RPC failure. After that — Revive integration (not tested yet, too early).

## Links

- [LibreVR/Revive](https://github.com/LibreVR/Revive)
- [GloriousEggroll/proton-ge-custom](https://github.com/GloriousEggroll/proton-ge-custom)
- [kron4ek/Wine-Builds](https://github.com/kron4ek/Wine-Builds)
- [OpenSCManager docs](https://learn.microsoft.com/en-us/windows/win32/services/service-control-manager)

---

*No guarantees it'll ever work. Just documenting what happens along the way.* 🧡
