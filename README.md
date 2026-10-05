# Oculus on Linux via Wine

Trying to run Meta Horizon Link (Oculus PC Client) under Wine/Bottles on Linux, with the goal of using Revive to play Oculus Store games through SteamVR.

## Why?

Quest Link and Air Link require the native Oculus app, which you do not want if you already use ALVR / Steam Link / WIVRN for streaming. This is an attempt to get the client running under Wine so it can do its job (auth, DRM) without taking over your desktop like a regular Windows install would.

## Status

🚧 **Installer runs, preflight check fails.** The installer reaches its "Can't Connect" screen and aborts.

The two blockers recorded before 2026-10-05 were both measured and both turned out to be wrong. See [`wine-scm-measured-findings.md`](wine-scm-measured-findings.md).

What blocks it now: the installer is a .NET application that needs Mono, and it fetches its config over HTTPS. That fetch fails, so `Dawn.Preflight.ConfigInitialisedCheck` fails and the installer aborts before copying any files.

## What's in here

- [`wine-scm-measured-findings.md`](wine-scm-measured-findings.md), the measurements that disprove the old root causes
- [`Oculus-Wine-Progress.md`](Oculus-Wine-Progress.md), installation log with every attempt, error, blocker and dead end
- [`0x6be-error-research.md`](0x6be-error-research.md), the original 0x6be research, kept for history and marked superseded
- [`how-to-play-minecraft-vr-flatpak.md`](how-to-play-minecraft-vr-flatpak.md), because apparently that's also a thing now

## What I tried

- Bottles + Proton GE 11-1: installer did not launch, probably missing Steam Runtime
- Bottles Flatpak + kron4ek-wine-11.3: installer launches, fails mid-way
- Debian wine 10.0, 32-bit prefix, `wine-mono-11.3.0`: installer gets as far as "Can't Connect"

## Working setup

```bash
export WINEPREFIX=<bottle> WINEARCH=win32
wineboot --init
# wine-mono is required; download.mono-project.com/wine/ is dead (404),
# use the GitHub releases instead
curl -sSL -o wine-mono.msi \
  https://github.com/wine-mono/wine-mono/releases/download/wine-mono-11.3.0/wine-mono-11.3.0-x86.msi
wine msiexec /i wine-mono.msi
```

The installer is `PE32 i386`, so the prefix must be 32-bit.

## What's next

Work out why the installer's own HTTPS fetch fails when a plain Mono HTTPS request in the same prefix succeeds. The installer's TLS stack is not the same one a normal Mono program uses. After that, Revive integration, which is still untested.

## Links

- [LibreVR/Revive](https://github.com/LibreVR/Revive)
- [GloriousEggroll/proton-ge-custom](https://github.com/GloriousEggroll/proton-ge-custom)
- [kron4ek/Wine-Builds](https://github.com/kron4ek/Wine-Builds)
- [wine-mono releases](https://github.com/wine-mono/wine-mono/releases)
- [OpenSCManager docs](https://learn.microsoft.com/en-us/windows/win32/services/service-control-manager)

---

*No guarantees it'll ever work. Just documenting what happens along the way.* 🧡
