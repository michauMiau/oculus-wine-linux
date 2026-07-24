# How to Play Minecraft VR Mods Through Flatpak in SteamVR

## Step 1: Give up.

Use Windows or something

## Step 2: No seriously, Give up.
Go install an AppImage or something instead of dealing with flatpak.

---

## Step 3: *Deep breath* OKAY.

So you're going to need to expose some folders to your launcher in Flatpak first.

### 1. Steam Logs Folder (read/write)

- **Path:** `~/.local/share/steam/logs`
- **Permissions:** read/write

This is required because the Vivecraft mod writes debug logs here, and without access the mod will freak out.

### 2. SteamVR Installation Folder

- **Path:** `~/.local/share/steam/steamapps/common/SteamVR`
- **Permissions:** read/write

Needed because Vivecraft needs access to the `vrmonitor` binary for some reason?!??.

---

## Step 4: Still not working?

If it still doesn't work and you get an error in the log like:

> *"Unable to find libqt5.so.5"* or something similar...

Then give up and just use an **AppImage** or a native installation of your favourite launcher (Prism launcher)

Alternatively, just use **[WIVRN](https://github.com/wivrn/wivrn)** which isn't compatible with Vivecraft ¯\_(ツ)_/¯ .

---

*Good luck ☺️ * 
