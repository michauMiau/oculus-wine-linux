# How to Play Minecraft VR Mods Through Flatpak in SteamVR

## Step 1: Give up.

No seriously, just go install an AppImage or something.

---

## Step 2: *Deep breath* OKAY.

So you're going to need to expose some folders to your launcher in Flatpak first.

### 1. Steam Logs Folder (read/write)

- **Path:** `~/.local/share/steam/logs`
- **Permissions:** read/write

This is required because the VR mods write debug logs here, and without access the application crashes or fails silently.

### 2. SteamVR Installation Folder

- **Path:** `~/.local/share/steam/steamapps/common/SteamVR`
- **Permissions:** read/write

Exposes the full SteamVR runtime so your launcher can communicate with OpenXR / SteamVR properly.

---

## Step 3: Still not working?

If it still doesn't work and you get an error in the log like:

> *"Unable to find libqt5.so.5"* or something similar...

Then give up and just use **Prism Launcher** — download the **AppImage**, it works, no folder-exposing nonsense required.

Alternatively, just use **[WIVRN](https://github.com/wivrn/wivrn)** instead of SteamVR altogether.

---

*Good luck. You'll need it.* 🎮
