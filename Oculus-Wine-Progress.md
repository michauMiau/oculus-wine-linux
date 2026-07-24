# Oculus on Linux via Wine — Installation Progress Log

> **Status:** 🚧 In Progress (Blocked)  
> **Started:** 2026-07-24  
> **Platform:** Bottles + Wine (kron4ek-wine-11.3 + Proton patchset)  
> **Target Application:** Meta Horizon Link (Oculus PC Client v83.0.0.224.349)

---

## ⚠️ OCR Disclaimer

**Important note on data accuracy:** Some error messages and dialog text in this document were extracted using OCR from screenshots/photos of the installation process. The following errors may contain minor transcription inaccuracies due to OCR limitations:

- `LastError = Ox6be` → Likely `0x6be` (hexadecimal error code)
- `Wywolanie RPC nieudane` / `Wywołanie RPC nieudane` → Polish text meaning "RPC call failed"
- Error codes like `-3001`, `0x6be (1726)` should be verified against actual dialog output

When possible, the original error messages have been preserved as accurately as OCR allowed. For exact reproduction of errors, please refer to the original screenshots/log files.

---

## 🎯 Objective

Run the full Meta Horizon Link (Oculus PC Client) on Linux using Wine/Bottles with Revive integration, enabling launch of Oculus Store games through SteamVR OpenXR runtime without requiring Quest Link streaming.

**Desired Architecture:**
```
Linux Desktop → Bottles (Wine) → Meta Horizon Link installed
    ↓
OVRService running as Windows service (via Wine SCM emulation)
    ↓
Revive overlay DLL injected into game executable
    ↓
Game launches via SteamVR OpenXR runtime
```

---

## 🔍 Environment & Tools Used

| Component | Version/Details |
|-----------|-----------------|
| **Wine** | kron4ek-wine-11.3 with Proton patchset |
| **Alternative tried** | Proton GE 11-1 (did NOT launch installer) |
| **Bottles** | Flatpak/AppImage version |
| **Analysis Tool** | Eagle-exe-scanner for dependency analysis |
| **Oculus Client** | Meta Horizon Link v83.0.0.224.349 |

---

## 📋 Dependency Analysis (Eagle-exe-scanner Results)

### Critical Issues Detected:

#### 1. Kernel Driver / Service Installation ⚠️
```
Detected: CreateService, StartService
```
The application attempts to install Windows services during setup. This is a fundamental requirement — the OVRService must be registered as a Windows service to function properly.

**Impact:** 🔴 Critical blocker — Wine's SCM emulation is incomplete

#### 2. WPF Framework Requirement ⚠️
```
Detected: WPF (Windows Presentation Foundation)
Recommendation: Use runner with 'ChildWindow' patches  
(e.g., Soda, Wine-GE-Custom, Proton-GE)
```
The Oculus client uses WPF for UI rendering. Without proper ChildWindow patching, the application may fail to initialize or display correctly.

**Impact:** 🟡 Moderate — kron4ek-wine-11.3 with Proton patches should handle this

---

## 📝 Installation Attempts Log

### Attempt #1: Bottles + Proton GE 11-1

**Result:** ❌ Installer did not launch  

**Issue:** The application failed to start immediately upon execution under Proton GE 11-1.

**Possible causes:**
- Proton GE 11-1 may lack specific WPF ChildWindow patches required by newer Oculus builds
- Bottles configuration incompatibility with this Wine version
- Missing system dependencies (allfonts, .NET runtime)

---

### Attempt #2: Bottles + kron4ek-wine-11.3 + Proton Patchset ✅ Partial Success

**Result:** ⚠️ Installer launches but fails mid-process  

#### Sub-step 2.1: Initial Launch & Font Issues

**Observed:**
- ✅ Installer successfully launched (first attempt with this Wine build)
- ⚠️ **Broken/missing fonts detected** in UI elements

**Action taken:** Installed `allfonts` package via Bottles dependencies manager to resolve missing glyphs for Polish/Latin character sets.

**Result after fix:** Fonts rendered correctly, but next issue appeared immediately.

---

#### Sub-step 2.2: Disk Space Detection Failure ❌

**Observed:**
After clicking "Continue" in the installer, a dialog appeared complaining about **insufficient disk space**, despite adequate storage being available on the host system.

**Possible causes:**
- Wine's virtual filesystem abstraction does not correctly report available space to Windows installers
- Installer uses native Windows API calls (`GetDiskFreeSpaceEx`) that map incorrectly under Wine
- Bottles container configuration may have limited apparent disk quota

**Status:** 🔴 **BLOCKER** — Installation cannot proceed past this point without resolution

---

#### Sub-step 2.3: Virtual Desktop Incompatibility ⚠️

**Observed:**
The Oculus installer refuses to run when Wine's virtual desktop mode is enabled.

**Action taken:** Disabled "Virtual Desktop" in Bottles configuration for this bottle.

**Result:** Installer proceeded past the initial compatibility check after disabling virtual desktop.

**Note:** This must be documented as a hard requirement — virtual desktop cannot be used with this installer.

---

#### Sub-step 2.4: .NET Runtime Dependencies ⚠️

**Eagle-exe-scanner finding:** The application requires **.NET Core / .NET Framework 5+**.

**Action taken:** Attempted to install required .NET runtime via Bottles dependencies or manual Wine installation.

**Status:** 🟡 Uncertain — Installer still failed at subsequent stages, so .NET may not have been fully satisfied or the issue lies elsewhere.

---

## 🚨 Current Blockers (Active Issues Preventing Progress)

### Blocker #1: Disk Space Detection Failure 🔴

**Error message (paraphrased from OCR):**  
*"Insufficient disk space on drive [C:]"*

**Root cause hypothesis:**
The Windows installer uses native APIs to enumerate available disk space. Under Wine/Bottles, the filesystem abstraction layer may not correctly report actual available storage, causing false "no space" errors even when plenty of room exists.

**Potential solutions being investigated:**
1. Manually configure Bottles drive C: with explicit size limits that appear sufficient
2. Modify environment variables (`WINEFSIZE` or similar) to override disk detection
3. Use Wine's registry editor to patch installer disk-check routines

---

### Blocker #2: RPC Service Installation Failure 🔴🔴🔴

This is the **primary architectural blocker**. The Oculus client requires running as a Windows service (`OVRService`), but Wine's Service Control Manager (SCM) emulation is incomplete.

#### Evidence from OVRServiceLauncher Output:

```
[LauncherService] Cannot install service:
OpenSCManager Failed, LastError = Ox6be (1726):
'Wywolanie RPC nieudane.'
'Do you have permissions to start Windows services?'
```

**Translation:** "RPC call failed" — Error code `0x6be` (decimal 1726) indicates the service cannot be started in the current execution context.

#### Context from launcher documentation (paraphrased):

> *"Pass in the '-install -start' arguments to install the  
> service launcher as a Windows service and start it.  
> Installing will not start the Oculus Runtime without  
> the additional -start argument."*

#### What this means:

The application expects to register itself with Windows SCM via `CreateService` API call. Wine's SCM emulation does not fully support this operation, causing the RPC call to fail immediately upon attempting service installation.

**Impact:** Without a running OVRService:
- ❌ No authentication tokens can be obtained from auth.meta.com
- ❌ DRM license keys cannot be delivered to games
- ❌ Oculus runtime cannot initialize tracking/hardware interfaces
- ❌ Revive integration is impossible without the base service layer

---

## 🧪 Additional Diagnostic Attempts

### Oculus Diagnostics Fixer App

**Result:** ❌ Fails immediately upon launch  

**Issue:** The built-in diagnostics/fixer tool does not execute successfully, providing no useful diagnostic output or repair functionality.

**Interpretation:** This confirms the fundamental service layer issue — the fixer itself depends on OVRService being available to function.

---

### Oculus Client Direct Launch

**Result:** ❌ Does not launch  

**Expected behavior:** Main client interface with library view, settings panel, and device management should appear.

**Actual behavior:** Application fails silently or crashes before reaching UI initialization.

---

### OVRLibrarian Launch Attempt

**Result:** ⚠️ Brief popup only  

**Observed:** A dialog titled **"Aktualizacja Link"** (Polish: "Link Update") appeared briefly — approximately one second — then disappeared immediately.

**Interpretation:** The library updater component attempted to initialize but failed due to service communication failure or missing dependencies, causing immediate termination.

---

### LibOVRRT DLL Loading Failure

**Error code:** `-3001`  
**Version reference:** `83.0.0.224.349`

**Issue:** Unable to load the core Oculus Runtime DLL (`LibOVRRT`). This suggests either:
- The DLL is missing from the installation directory
- The DLL exists but cannot be loaded due to dependency chain failure (likely .NET or OVRService)
- Architecture mismatch (32-bit vs 64-bit DLL under Wine)

---

## 📊 Summary Matrix

| Component | Status Under Wine/Bottles | Severity |
|-----------|---------------------------|----------|
| Installer execution | ✅ Launches (with allfonts fix) | Low |
| Font rendering | ✅ Fixed via allfonts package | Resolved |
| Virtual desktop mode | ❌ Incompatible with installer | Workaround: disable it |
| .NET runtime | 🟡 Uncertain (needs verification) | Medium |
| Disk space detection | 🔴 Fails (false "no space" error) | High — BLOCKER |
| WPF/ChildWindow patches | ✅ Provided by Proton patchset | Low |
| OVRService registration | 🔴🔴 Fails (OpenSCManager RPC) | Critical — PRIMARY BLOCKER |
| LibOVRRT loading | 🔴 Fails (error -3001) | High — dependent on service layer |
| Client authentication | 🔴 Blocked (no running service) | Critical |
| Game DRM access | 🔴 Blocked (depends on OVRPlatformService) | Critical |

---

## 🎯 Theoretical Path Forward (Requires Significant Development Effort)

To make this work end-to-end, the following components would need to be implemented or patched:

### 1. OVRService Wine Patching (Critical Path)

**Challenge:** Emulate `CreateService` / `StartService` calls from OpenSCManager API under Wine.

**Possible approaches:**
- Develop a Wine patch that redirects Windows SCM operations to a Linux-compatible service manager (systemd, or custom Wine-specific daemon)
- Create a mock OVRService that satisfies the API contract without actual service registration
- Intercept and redirect RPC calls at the DLL level using Wine's `wineproxy` framework

**Estimated effort:** Weeks to months of work requiring deep knowledge of both Wine internals and Meta's Oculus SDK architecture.

---

### 2. LibOVRRT DLL Compatibility Layer

**Challenge:** Ensure the correct version of `LibOVRRT.dll` is available and loadable under Wine.

**Actions needed:**
- Extract or copy the Windows version of `LibOVRRT.dll` from a working installation
- Verify architecture compatibility (likely 64-bit, must match Wine prefix bitness)
- Ensure all dependent DLLs are present in the Wine prefix's `System32` directory
- Patch any hardcoded paths or dependencies that fail under Wine

---

### 3. Disk Space Detection Override

**Challenge:** Trick the installer into perceiving sufficient disk space.

**Possible solutions:**
- Manually configure Bottles drive C: with explicit size allocation that exceeds installer requirements
- Use environment variables (`WINEFSIZE`, `DRIVE_C_SIZE`) to override filesystem reporting
- Modify the installer's disk-check routine via registry patches or DLL injection
- Create a virtual drive mount that presents as having ample space

---

### 4. Revive Integration Layer

**Challenge:** Adapt Revive overlay DLL for Wine injection and OpenXR translation.

**Components needed:**
- Patch Revive DLL to work with Wine's process injection model
- Create an OpenXR ↔ Oculus SDK translation layer
- Ensure SteamVR OpenXR loader receives correct HMD tracking, controller input, and timing data
- Handle the authentication/DRM flow that normally passes through OVRPlatformService

**Note:** This depends entirely on Blocker #1 being resolved first — Revive cannot function without the base service layer running.

---

### 5. Authentication & DRM Flow Workaround

**Challenge:** Obtain valid tokens and license keys without native Windows service infrastructure.

**Approaches under consideration:**
- Mock HTTPS responses from `auth.meta.com` to simulate successful authentication
- Create a local token cache that bypasses remote verification (likely violates ToS)
- Develop a working HTTPS tunnel through Wine's winhttp implementation to reach actual servers
- Intercept and replay valid tokens from a Windows installation (requires existing access)

**Legal/ethical note:** Bypassing DRM or authentication may violate Meta's Terms of Service. This document is for research purposes only.

---

## 📚 References & Related Projects

### Archived / Legacy:
- **[jspenguin/oculus-wine-wrapper](https://github.com/jspenguin/oculus-wine-wrapper)** — Wrapper from DK1/DK2 era (archived November 2023). Useful as historical reference but not applicable to modern Oculus Store architecture. Last commit ~11 years ago.

### Active / Partially Applicable:
- **[GloriousEggroll/proton-ge-custom](https://github.com/GloriousEggroll/proton-ge-custom)** — Extended Proton with additional VR SDK patches. May contain relevant patches for newer builds.
- **[kron4ek/Wine-Builds](https://github.com/kron4ek/Wine-Builds)** — Custom Wine forks with gaming optimizations. The version used in this project (11.3) is based on these builds.
- **[Eagle-exe-scanner](https://github.com/evereststudios/eagle-exe-scanner)** — Tool used for dependency analysis in this investigation.

---

## 📌 Next Steps & Investigation Priorities

1. **Investigate OpenSCManager emulation improvements**  
   Check if newer Wine versions (9.x+) have added partial service manager support that might satisfy the `CreateService` call.

2. **Test disk space override methods**  
   Experiment with Bottles drive configuration and environment variables to bypass the false "no space" error.

3. **Attempt manual OVRService launch with patched DLLs**  
   Bypass the installer's service registration requirement by manually placing patched `OVRService.exe` in the prefix and attempting direct execution.

4. **Verify Proton GE patch coverage for newer builds**  
   Check if more recent Proton GE versions (post-11.x) include specific Oculus SDK patches that were missing when Attempt #1 was made.

5. **Document intermediate successes**  
   As any blocker is resolved, update this log with exact commands, configurations, and results for reproducibility.

6. **Engage with community**  
   Check r/linux_gaming, Bottles forums, and ProtonDB for others who may have attempted similar setups or discovered workarounds.

---

## 📝 Methodology Notes

This document was compiled from:
- Direct observation of installation dialogues and error messages
- Eagle-exe-scanner dependency analysis output
- OCR extraction of Polish-language error dialogs (with disclaimer above regarding potential transcription inaccuracies)
- Systematic testing of Wine version variants and Bottles configurations

All findings are documented as observed during actual installation attempts on Linux with Bottles + kron4ek-wine-11.3.

---

*This document will be updated continuously as new progress is made, blockers are resolved, or additional failures are encountered.*

**Last updated:** 2026-07-24  
**Author:** [User]  
**Repository:** `oculus-wine-linux` (local Git)
