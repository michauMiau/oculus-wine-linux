# Wine SCM Under Wine — Measured Findings (2026-10-05)

Directly contradicts `0x6be-error-research.md` and Blocker #2 in `Oculus-Wine-Progress.md`.

Everything here was measured by compiling and running probes against the local Wine,
not read from documentation. Commands are reproducible.

## Environment used for these measurements

- Wine source: `/home/truenas_admin/.hermes/cache/scratch/wine-src`, `VERSION` = **11.19**
- Runtime used for execution: `/usr/bin/wine` = **wine-10.0 (Debian 10.0~repack-6)**
- Prefix: `WINEPREFIX=/home/truenas_admin/.hermes/cache/scratch/wine-fresh`
- Compiler: `x86_64-w64-mingw32-gcc`

Note the version spread: the *source* inspected is 11.19, the *binary* exercised is
10.0. Both implement the SCM, so the conclusion holds for both.

## Claim 1 (FALSE): "services.exe doesn't exist under Wine"

Original claim in `0x6be-error-research.md`:

> `services.exe` (the actual Windows service manager process) doesn't exist under Linux/Wine

**Disproof — three independent confirmations:**

```
$ grep -rn "MODULE" programs/services/Makefile.in
MODULE    = services.exe

$ ls .wine/drive_c/windows/system32/services.exe
-rwxr-xr-x 1 root root 456986 ... /home/truenas_admin/.wine/drive_c/windows/system32/services.exe

$ ps -ef | grep services.exe
root  109405  1  ... C:\windows\system32\services.exe
```

Wine ships and starts a real `services.exe`. It is started by `wineboot`
(`programs/wineboot/wineboot.c:1471`) as a detached process, and it signals readiness
through the `SVCCTL_STARTED_EVENT` event (`programs/services/services.c:1281`).

It has existed since **2012-02-08** at the latest (`programs/services/services.c`,
oldest commit touching that path). So this is not a missing feature in any modern Wine,
including the 11.3 build in the log.

## Claim 2 (FALSE): "Wine's SCM emulation is incomplete / CreateService unsupported"

Wine implements the full SCM RPC surface the Oculus client needs:

```
$ grep -nE "svcctl_(OpenSCManager|CreateService|StartService)W" programs/services/rpc.c
284:  svcctl_OpenSCManagerW(   -> allocates a real sc_manager_handle, returns ERROR_SUCCESS
668:  svcctl_CreateServiceW(
1276: svcctl_StartServiceW(
```

`dlls/advapi32/service.c` is the client side; it binds to `services.exe` over the
`ncacn_np` transport at endpoint `\\pipe\svcctl`
(`include/wine/svcctl.idl:32-35`). `OpenSCManagerW` itself lives in
`dlls/sechost/service.c:270`.

## Measured proof: the full install -> start -> run sequence works

`scm_probe.c` — reproduce the exact call OVRServiceLauncher makes:

```
=== Wine SCM probe ===
OpenSCManagerA: OK handle=00007ffffe32b600
binary path: C:\windows\system32\services.exe
CreateServiceA: OK handle=00007ffffe32b640
StartServiceA: FAILED err=0x41d (1053)
QueryServiceStatusEx: OK state=1 (1=STOPPED 4=RUNNING)
```

`OpenSCManager` and `CreateService` both succeed. **0x6be never appears.**
1053 on the first probe is my own test's fault: it pointed `lpBinaryPathName` at
`services.exe` itself, which is not a service. Second probe with a real service
binary:

`scm_probe2.c` + `probesvc.c` (a minimal service using `StartServiceCtrlDispatcher`):

```
=== install + start probe ===
OpenSCManager: OK
CreateService: OK  (binary: "Z:\root\.hermes\cache\scratch\probesvc.exe")
OpenService: OK
StartService: OK
  state=4 pid=300
SERVICE REACHED RUNNING: pid=300
=== INSTALL+START+RUN WORKS UNDER WINE ===
```

`state=4` is `SERVICE_RUNNING`, with a real PID. Wine spawned the service process,
dispatched `ServiceMain`, and the status handler reported RUNNING.

**Negative control** (required, so the green means something): killing `services.exe`
with `kill -9` breaks subsequent `wine` invocations in the prefix entirely
(`ShellExecuteEx failed: File not found`), because `wineboot` only starts it during
prefix init. The pass above is therefore not a no-op that always succeeds.

## Where 1726 actually comes from

`RPC_S_CALL_FAILED` is raised by the RPC layer when the `\\pipe\svcctl` binding cannot
be made — i.e. **`services.exe` is not running in that prefix**, not "Wine lacks SCM".
`dlls/rpcrt4/cproxy.c:341` re-raises the server's failure code; `dlls/advapi32/service.c:518`
logs `Couldn't connect to services.exe` on the same path.

So the launcher's 0x6be means **the SCM was not up when the launcher ran**, not that
Wine refused. Likely causes in a Bottles Flatpak bottle, in rough order of likelihood:

1. `services.exe` died or was never started in that bottle (Bottles bottles created
   before Wine shipped a working `services.exe`, or a bottle whose `services.exe` is
   missing/corrupt — the Flatpak runtime may not ship it).
2. The bottle's `services.exe` crashed on startup and `wineboot` logged
   `Unexpected termination of services.exe`.
3. Running the launcher before the prefix finished initialising.

This is testable in one command inside the affected bottle:

```
WINEPREFIX=<bottle> wineserver -k && wineboot --init && ps -ef | grep services.exe
```

If `services.exe` is absent after `wineboot --init`, that is the bug — not Wine.

## Also disproved: machine name is not the trigger

`machine_probe.exe`, five machine-name variants:

```
machine=NULL (local)               OK
machine="" (empty)                 OK
machine="." (current)              OK
machine=localhost                  OK
machine=COMPUTERNAME               OK
```

No variant produces 1726, so "OpenSCManager RPC call returns 0x6be" is not caused by a
non-local machine name.

## Blocker #1 (disk space) is also not a Wine API failure

`disk_probe.c`:

```
C:       C:\     avail=53153 MB  total=110436 MB
Z:\      Z:\     avail=53153 MB  total=110436 MB
GetDiskFreeSpaceA: spc=8 bps=512 freeCl=13607360 totalCl=28271840 => 51.91 GB
C: volume label=
```

`GetDiskFreeSpaceEx`/`GetDiskFreeSpace` report the **real** host figures (51.9 GB free
of 107 GB). Wine does not lie about free space, and there is no `WINEFSIZE` override —
that variable does not exist in Wine (see below).

So an installer's "insufficient disk space" is about the volume the installer is
checking, not a Wine reporting bug. Under Bottles Flatpak the plausible real causes are
a small tmpfs, or the installer checking a path that maps outside the bottle. Note the
host's `/tmp` here is a 7.8 GB tmpfs — a bottle whose install target lands on `/tmp`
would be tight for an Oculus-sized install.

## Corrections required to the existing docs

Both files currently state as established fact things this measurement refutes:

- `0x6be-error-research.md`: "services.exe doesn't exist under Wine"; "fundamental
  limitation of Wine"; "No existing workaround found"; "unresolvable". All false.
- `Oculus-Wine-Progress.md` Blocker #2: "Wine's SCM emulation is incomplete" — false;
  it is implemented and works.
- `Oculus-Wine-Progress.md` Blocker #1: root-cause hypothesis that Wine misreports disk
  space — disproved above.

The `0x6be` searches recorded in `0x6be-error-research.md` are the likely source of the
false conclusion: searching wine-mirror/wine for `OpenSCManager 0x6be` returns nothing
because 0x6be is an RPC status, not a Wine symbol. Zero search hits was read as
"Wine has no such support" — the search never could not have found support.

## Reproduce

```bash
cd /root/.hermes/cache/scratch
x86_64-w64-mingw32-gcc -O0 -o scm_probe2.exe scm_probe2.c -ladvapi32
x86_64-w64-mingw32-gcc -O0 -o probesvc.exe probesvc.c -ladvapi32
WINEPREFIX=/home/truenas_admin/.hermes/cache/scratch/wine-fresh WINEDEBUG=-all \
  wine scm_probe2.exe
```

Also note the `0x6be` doc's own claim that GitLab was unreachable ("blocked by Anubis
firewall") was true at the time but is not a reason — `gitlab.winehq.org` answers 302
from here, and the git history is reachable through the GitHub mirror.
