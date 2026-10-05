# Wine SCM, measured 2026-10-05

This refutes `0x6be-error-research.md` and Blocker #2 in `Oculus-Wine-Progress.md`.

Everything below came from compiling probes and running them against local Wine. Nothing
is quoted from documentation. Commands are reproducible.

## Environment

Wine source sits at `/home/truenas_admin/.hermes/cache/scratch/wine-src`, VERSION 11.19.
The binary I actually ran is `/usr/bin/wine`, wine-10.0 (Debian 10.0~repack-6). Prefix
is `WINEPREFIX=/home/truenas_admin/.hermes/cache/scratch/wine-fresh`. Compiler is
`x86_64-w64-mingw32-gcc`.

The source I read and the binary I ran are different versions. Both ship a working SCM,
so the conclusion covers both, including the 11.3 build from the log.

## services.exe exists

The original claim was that `services.exe` doesn't exist under Wine. Three independent
checks say otherwise.

```
$ grep -rn "MODULE" programs/services/Makefile.in
MODULE    = services.exe

$ ls -la /home/truenas_admin/.wine/drive_c/windows/system32/services.exe
-rwxr-xr-x 1 root root 456986 Sep 27 12:48 .../system32/services.exe

$ ps -ef | grep services.exe
root  109405  1  0 13:09 ?  00:00:00 C:\windows\system32\services.exe
```

Wine ships and starts a real `services.exe`. wineboot launches it detached at
`programs/wineboot/wineboot.c:1471`, and it signals readiness with the
`SVCCTL_STARTED_EVENT` event at `programs/services/services.c:1281`.

The oldest commit touching `programs/services/services.c` is dated 2012-02-08. So no
modern Wine is missing this feature.

## Wine implements the SCM calls the client needs

`dlls/advapi32/service.c` is the client side. It binds to `services.exe` over the
`ncacn_np` transport at endpoint `\\pipe\svcctl`, defined in `include/wine/svcctl.idl`
lines 32 to 35. `OpenSCManagerW` itself lives at `dlls/sechost/service.c:270`.

The server side implements the three RPC entry points the Oculus launcher calls.

```
$ grep -nE "svcctl_(OpenSCManager|CreateService|StartService)W" programs/services/rpc.c
284:  svcctl_OpenSCManagerW(
668:  svcctl_CreateServiceW(
1276: svcctl_StartServiceW(
```

## Measured proof of a running service

`scm_probe.c` makes the same call OVRServiceLauncher makes.

```
=== Wine SCM probe ===
OpenSCManagerA: OK handle=00007ffffe32b600
binary path: C:\windows\system32\services.exe
CreateServiceA: OK handle=00007ffffe32b640
StartServiceA: FAILED err=0x41d (1053)
QueryServiceStatusEx: OK state=1 (1=STOPPED 4=RUNNING)
```

Both OpenSCManager and CreateService succeed. Error 0x6be never appears.

The 1053 here is my own test's fault. I pointed `lpBinaryPathName` at `services.exe`,
which is not a service. The second probe uses a real service binary, `probesvc.c`, built
around `StartServiceCtrlDispatcher`.

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

State 4 is SERVICE_RUNNING, with a real PID. Wine spawned the process, dispatched
`ServiceMain`, and the status handler reported RUNNING.

### Negative control

A green result means nothing without one. Killing `services.exe` with `kill -9` breaks
every later `wine` call in that prefix, because wineboot only starts it during prefix
init.

```
$ kill -9 109405 && wine scm_probe.exe
Application could not be started, or no application associated with the specified file.
ShellExecuteEx failed: File not found.
```

So the pass above is not a no-op that always succeeds.

## Where 1726 comes from

`RPC_S_CALL_FAILED` is raised by the RPC layer when the `\\pipe\svcctl` binding cannot be
made. In practice this means `services.exe` is not running in that prefix. It does not
mean Wine refuses to implement services. `dlls/rpcrt4/cproxy.c:341` re-raises the
server's failure code, and `dlls/advapi32/service.c:518` logs
`Couldn't connect to services.exe` on the same path.

So the launcher's 0x6be says the SCM was not up when the launcher ran. Likely causes in a
Bottles Flatpak bottle:

1. `services.exe` died or never started in that bottle. Bottles created before Wine
   shipped a working `services.exe` are suspects, and so is a Flatpak runtime that omits
   it.
2. The bottle's `services.exe` crashed on startup, with
   `Unexpected termination of services.exe` in the wineboot log.
3. The launcher ran before the prefix finished initialising.

One command settles it inside the affected bottle.

```
WINEPREFIX=<bottle> wineserver -k && wineboot --init && ps -ef | grep services.exe
```

No output after `wineboot --init` means that is your bug, and it is a bottle problem, not
a Wine limitation.

## Machine name is not the trigger

`machine_probe.exe` passes five machine-name variants to OpenSCManager.

```
machine=NULL (local)               OK
machine="" (empty)                 OK
machine="." (current)              OK
machine=localhost                  OK
machine=COMPUTERNAME               OK
```

None produce 1726, so a non-local machine name does not explain the reported error.

## Disk space reporting is correct too

Blocker #1 assumed Wine misreports free space. `disk_probe.c` measures what the installer
would see.

```
C:       C:\     avail=53153 MB  total=110436 MB
Z:\      Z:\     avail=53153 MB  total=110436 MB
GetDiskFreeSpaceA: spc=8 bps=512 freeCl=13607360 totalCl=28271840 => 51.91 GB
C: volume label=
```

Those are the real host numbers, 51.9 GB free of 107 GB. There is no `WINEFSIZE` or
`DRIVE_C_SIZE` override to reach for either, since neither exists in Wine.

```
$ grep -rn "WINEFSIZE\|DRIVE_C_SIZE" --include=*.c --include=*.h --include=*.in .
(no output)
```

An installer's "insufficient disk space" is therefore about the volume it checked, not a
Wine reporting bug. Under Bottles Flatpak the likely culprits are a small tmpfs or an
install target that maps outside the bottle. Worth knowing that `/tmp` on this host is a
7.8 GB tmpfs, which is tight for an Oculus-sized install.

## What has to change in the existing docs

Both files state as fact things this measurement refutes.

`0x6be-error-research.md` claims services.exe doesn't exist, calls it a fundamental
limitation of Wine, says no workaround exists, and calls the problem unresolvable.
Blocker #2 in `Oculus-Wine-Progress.md` says Wine's SCM emulation is incomplete. It is
implemented and it works. Blocker #1 blames Wine's disk space reporting, which the probe
above disproves.

I think the false conclusion came from the search recorded in `0x6be-error-research.md`.
Searching wine-mirror/wine for `OpenSCManager 0x6be` returns nothing, because 0x6be is an
RPC status and not a Wine symbol. That search could never have found support. Zero hits
got read as Wine having no such support.

## Reproduce

```bash
cd /root/.hermes/cache/scratch
x86_64-w64-mingw32-gcc -O0 -o probesvc.exe probesvc.c -ladvapi32
x86_64-w64-mingw32-gcc -O0 -o scm_probe2.exe scm_probe2.c -ladvapi32
WINEPREFIX=/home/truenas_admin/.hermes/cache/scratch/wine-fresh WINEDEBUG=-all \
  wine scm_probe2.exe
```

One more correction to the old research. It recorded GitLab as blocked by an Anubis
firewall and treated that as a dead end. `gitlab.winehq.org` answers 302 from here, and
the git history is readable through the GitHub mirror. An unreachable authority plus a
token search returning zero is a conclusion with no measurement behind it.

## Limits of this evidence

I did not have the actual Oculus binaries, so I cannot say what OVRService specifically
hits. The probes prove the SCM layer works, not that Meta's client is satisfied by it.
