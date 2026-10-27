```
                   ___     _     ___   _____   _  _     _     _  __
                  / __|   /_\   | _ \ |_   _| | || |   /_\   | |/ /
                  \__ \  / _ \  |   /   | |   | __ |  / _ \  | ' <
                   |___/ /_/ \_\ |_|_\   |_|   |_||_| /_/ \_\ |_|\_\

                   __  __    _  __  ___   ___   _  _   ___   _
                   \ \/ /   | |/ / | __| | _ \ | \| | | __| | |
                    >  <    | ' <  | _|  |   / | .` | | _|  | |__
                   /_/\_\   |_|\_\ |___| |_|_\ |_|\_| |___| |____|

                                 dev by sarthak </>
```

# SARTHAK X KERNEL v1.1

Kernel-assisted process tooling for Windows x64.

One `SARTHAKxKERNEL.exe`. Kernel driver, injector, monitor and loader — no extra files, no manual steps.

## What it is

SARTHAK X KERNEL is a complete kernel toolkit that deploys from a single installer.

The driver is unsigned and loads through a BYOVD DSE patch. Once it is running, everything else talks to it over a device interface: memory read, memory write, remote allocation, DLL injection, shellcode injection and process monitoring.

No test-signing. No signing certificate. No dependency install. Everything the tool needs is already inside the exe.

## Features

Deployment
- Single-file delivery — driver, agent, injector and loader all embedded in one SARTHAKxKERNEL.exe
- Encrypted payload stream — nothing extractable from the installer with a normal archiver
- Silent install flow: welcome, check, accept, progress
- Auto-cleanup — the loader and the vulnerable driver are removed from disk once the driver is up
- Custom uninstaller, no visible Add/Remove entry

Kernel layer
- Unsigned WDM driver with Ps / Cm / Ob callback registration
- Kernel-side memory primitives: allocate, write, read, free in a remote process
- PID protection with handle stripping
- Event ring buffer for lineage and process events
- Live stats over IOCTL

Injector
- Three paths: CreateRemoteThread, APC queue and driver-assisted shellcode mapping
- DLL injection by path, raw shellcode by file
- Remote command execution inside the target's context
- Arbitrary read and write by address, with hex-dump output
- Self-test mode that validates the alloc, write, read, free round trip
- Interactive shell or fully scripted CLI
- x86 / x64 / WOW64 target detection

Loader
- KDU -dse path in-process — temporarily disables Driver Signature Enforcement
- Registers and starts the kernel service through the SCM
- --no-service mode for DSE patching only
- Provider metadata via drv64.dll, driver image supplied at run time

Hardening
- Anti-debug: PEB, remote debugger, NtQueryInformationProcess (port, object, flags)
- Live image header scrub — PE layout removed from RAM after startup
- CFG + High Entropy ASLR + NX + /GS
- No PDB, no build-path strings in the shipped binary

## What makes it different

1. It is one file.
   Most kernel tooling ships as driver plus loader plus injector plus a README full of sc create commands. This is SARTHAKxKERNEL.exe, click, done.

2. No test-signing, no signing certificate.
   DSE is patched at install time, so the driver loads on a stock machine the moment signing checks are off.

3. The loader cleans up after itself.
   The BYOVD image and the DSE patcher do not stay on disk.

4. Two tools in one.
   The same driver backs a defensive monitor (watch, protect, stats) and an offensive injector.

5. Real CLI.
   Every action is scriptable. No GUI-only workflows.

## Requirements

- Windows 10 or Windows 11, x64
- Administrator rights
- Secure Boot OFF
- HVCI / Memory Integrity OFF
- Vulnerable Driver Blocklist OFF
- No pending reboot

If HVCI is enabled, setup aborts on its own. Turn it off, reboot, then run the installer again.

## Install

1. Run SARTHAKxKERNEL.exe as administrator
2. Accept the confirmation
3. Wait for the progress bar
4. Launch the tool from the finish page

After setup, the tool lives at `<install dir>\dxcap.exe`.

To remove it, run `<install dir>\uninstall.exe`.

## Usage

Interactive:

    dxcap.exe

The banner and CONNECTED appear, then target selection.

Command line:

    dxcap.exe --list
    dxcap.exe --pid 1234 --dll C:\path\payload.dll
    dxcap.exe --name chrome.exe --apc payload.bin
    dxcap.exe --pid 1234 --shellcode payload.bin
    dxcap.exe --pid 1234 --exec "calc.exe"
    dxcap.exe --pid 1234 --read 0x7ff000 64 dump.bin
    dxcap.exe --pid 1234 --write 0x7ff000 "90 90 CC"
    dxcap.exe --selftest

Flags:

    --pid <N>                    target process id
    --name <IMAGE.EXE>           resolve target by image name
    --dll <PATH>                 inject a DLL
    --shellcode <FILE>           inject raw shellcode
    --apc <FILE>                 inject shellcode via APC
    --exec <CMD>                 run a command in target context
    --read <ADDR> <SIZE> [OUT]   read remote memory
    --write <ADDR> <HEXBYTES>    write remote memory
    --method <thread|apc>        force injection method
    --list                       list processes
    --selftest                   driver alloc/write/read/free check
    --verbose                    verbose output

## Agent (monitor mode)

    winhost.exe watch
    winhost.exe stats
    winhost.exe protect <pid> [flags]
    winhost.exe unprotect <pid>

## Loader (manual use)

    wnpconn.exe --target <driver.sys> --byovd <provider.sys>
    wnpconn.exe --target <driver.sys> --byovd <provider.sys> --no-service

The --byovd filename must match a KDU provider name.
Supported examples: NalDrv.sys, RTCore64.sys, Gdrv.sys, iqvw64e.sys.

## How it works

1. SARTHAKxKERNEL.exe decrypts its embedded payloads into the target directory
2. The loader patches DSE in memory through the BYOVD provider driver
3. The loader registers and starts the kernel service through the SCM
4. The loader and the provider image are deleted from disk
5. The injector opens the device and sends IOCTLs

The driver stays loaded until the next reboot. DSE comes back after a reboot, so the service has to be started again by re-running the installer.

## FAQ

Does it need test-signing mode?
No. The loader patches DSE in memory, then starts the driver through the SCM. Test-signing and bcdedit changes are not part of the flow.

Do I need a signing certificate?
No. The driver is unsigned by design. The loader is what makes it loadable.

Does it survive a reboot?
The service is demand-start. DSE comes back after a reboot, so re-run the installer to bring the driver up again.

Why does setup refuse to continue?
One of these is on: Secure Boot, HVCI / Memory Integrity, or the Vulnerable Driver Blocklist. Turn it off, reboot, run the installer again.

Does it work with HVCI on?
No. HVCI blocks unsigned kernel code after load. It has to be off.

Will antivirus flag it?
Unsigned kernel driver plus a BYOVD loader is a detection magnet. Every build gets flagged eventually. That is the nature of the category, not a bug.

How do I remove it?
The bundled uninstaller stops and deletes the service and removes the target directory. It is not listed in Add/Remove Programs.

Which Windows builds are tested?
Windows 10 21H2+ and Windows 11 22H2+ on x64.

## Changelog

v1.1.0
- Rebranded to SARTHAK X KERNEL across installer, binary and metadata
- New installer flow: welcome, check, accept, progress
- Install progress details hidden, progress bar only
- Custom logo: welcome panel, header banner, app icon
- Payloads encrypted inside the installer
- Installer removes the loader and the vulnerable driver after the driver is up
- Injector hardening: anti-debug, header scrub, CFG, High Entropy ASLR
- Removed debug symbols and build-path strings from the binary

v1.0.0
- Kernel driver with Ps/Cm/Ob callbacks
- Kernel-assisted read/write/alloc/free
- Thread, APC and driver-assisted injection paths
- Process-lineage monitor with PID protection
- KDU-based DSE loader

## Tags

#windows #kernel #driver #reverseengineering #byovd #injection #lowlevel #cpp #securityresearch #processmonitor

dev by sarthak </>
