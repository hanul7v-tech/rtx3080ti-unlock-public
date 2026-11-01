# Intel Z390 dual-GPU observations and read-only UEFI diagnostic plan

Status: **Windows display verified; UEFI ownership unverified.** Source: user-reported physical Windows test, 2026-11-01. No EFI diagnostics executed.

## Windows observations (not UEFI proof)
- CPU: Intel Core i7-9700K; board: ASRock Z390 Extreme4; BIOS P4.20.
- Intel UHD 630: Present=True, Status=OK; driver 27.20.100.9415; reported resolution 2560x1600; secondary monitor connected to motherboard HDMI.
- EVGA RTX 3080 Ti: Present=True, Status=OK; driver 610.88; reported resolution 2560x1440; main monitor connected to discrete GPU.
- Both displays show Windows; pointer and windows move between them.
- Secure Boot=False (Windows observation); Windows Boot Manager EFI path was observed, but the exact path, device and partition identifiers were not supplied.
- **No claim** about UEFI GOP initialization, ConOut routing, active firmware console, ESP association or boot entry resolution follows from Windows display behavior.

## Read-only UEFI diagnostic design (not executed)
1. A separately reviewed EFI diagnostic, launched manually without creating/changing Boot#### or BootOrder, should use UEFI Boot Services only for enumeration.
2. Read the ConsoleOutHandle and the ConOut/ConOutDev variables via GetVariable; log raw device paths and EFI_SIMPLE_TEXT_OUTPUT_PROTOCOL presence. Do not call SetVariable, ResetSystem, ConnectController, DisconnectController or any reset/FLR routine.
3. Enumerate handles supporting EFI_GRAPHICS_OUTPUT_PROTOCOL; read Mode->Info and FrameBufferBase/Size without SetMode. Map each GOP handle to PCI location/device identity using EFI_PCI_IO_PROTOCOL GetLocation and read-only PCI config reads, and record the device path. Distinguish Intel UHD 630 and NVIDIA GA102 by vendor/device IDs, not display labels.
4. Compare the ConOut device path instances and ConsoleOutHandle path to the GOP device paths. A matching parent PCI device path suggests routing, but GOP handle presence alone is **not** proof of active console use. Explicitly report ambiguous or unmatched paths as UNKNOWN.
5. Log whether the RTX 3080 Ti GOP exists and whether any ConOut path or active console handle resolves to its device path. Do not assume the Windows primary monitor identifies the UEFI console.
6. Capture text output only to a manually selected non-ESP destination if independently approved; otherwise use on-screen output or in-memory inspection. Do not write to ESP, NVRAM or firmware.

## Windows Boot Manager / ESP relationship (read-only)
- In elevated Windows PowerShell/CMD, capture `bcdedit /enum firmware /v` and `bcdedit /enum {bootmgr} /v` (read only). Do not invoke /set, /create, /delete, /displayorder or /bootsequence.
- Use `Get-Disk`, `Get-Partition` and `Get-Volume` to identify candidate GPT EFI System Partitions and their disk/partition identities; do not assign drive letters, mount, format or alter partitions.
- Correlate firmware Windows Boot Manager device path (HD partition GUID + File path) with the GPT ESP unique partition GUID, then verify the EFI executable path only through a separately approved read-only filesystem access method. A BCD device reference by itself is not proof of the firmware's active ESP.
- Preserve raw outputs with machine-specific GUIDs redacted before publishing. Missing exact paths/IDs => association **UNVERIFIED**.

## Safety gates and unresolved risks
- No GPU FLR/reset, no firmware modifications, no VBIOS flashing, no EFI boot entry/order changes, no GPU Unlock execution.
- GOP/ConOut ownership unknown until pre-OS enumeration is actually run.
- Windows Intel output does not guarantee Intel GOP is firmware ConOut; NVIDIA GOP could still be initialized or selected.
- Secure Boot=False is observed in Windows, not independent proof of firmware policy or ability to launch an unsigned EFI image.
- UEFI console output may be absent, multiplexed or changed by firmware boot phase; mapping requires full device paths and PCI locations.
- ESP GUID, Boot#### target and active Windows Boot Manager association not supplied.
- Any new EFI diagnostic must undergo static/offline tests and code review before manual execution. No physical execution authorized by this document.

## Offline regression checks (design only)
- Parse sample multi-instance ConOut paths, unmatched paths, and Intel-only/NVIDIA-only/both-GOP inventories.
- Confirm ambiguous mappings return UNKNOWN rather than VERIFIED.
- Static scan of diagnostic source for forbidden calls: SetVariable, SetMode, ResetSystem, FLR, ConnectController, DisconnectController and write-capable filesystem operations.
- Preserve all existing offline tests unchanged; run the repository's documented test command before merging. No test execution claimed here.
