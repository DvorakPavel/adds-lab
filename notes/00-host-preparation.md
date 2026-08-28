# Task 0 – Host preparation

**Date:**
2026-08-28

## Goal

Bring the physical host to a state where Oracle VirtualBox can run Windows Server 2025 guests before
any ISO is downloaded or any VM is built. Two things to establish:

1. The host CPU exposes every instruction Server 2025 requires — SSE4.2 and POPCNT are new hard
   requirements relative to Server 2022, and Setup refuses to install without them.
2. Understand the Hyper-V / VBS state on the host, since a running hypervisor makes VirtualBox fall
   back from native hardware virtualisation to the slower Windows Hypervisor Platform backend.

## Skill-area mapping

Maps to **none** of the five published APL-1008 skill areas. The assessment assumes a working host
and a running forest already exist. This is Phase 0 groundwork, documented because the lab cannot
be built without it and because a portfolio reviewer should see the environment was set up
deliberately.

## Host state — recorded

| Item | Value |
|---|---|
| Host | ASUS TUF GAMING B550-PLUS, Ryzen 7 5700X (8C/16T), 32 GB |
| OS | Windows 11 Pro, build 26200, UEFI, Secure Boot On |
| CPU meets Server 2025 reqs | Yes — Zen 3 supports SSE4.2, POPCNT, NPT (SLAT) |
| Hypervisor present (start) | True |
| Hypervisor present (after remediation) | **Still True** — see below |
| VBS status | Running (status 2), not UEFI-locked, no policy forcing it |
| VirtualBox version | 7.2.8 r173730 |
| `C:` free | 446 GB |
| `E:` free | 803 GB |

## What I did

### 1. CPU feature check

Server 2025 processor requirements per Microsoft's hardware requirements page: x64, NX/DEP,
CMPXCHG16b, LAHF/SAHF, PrefetchW, SLAT (Intel EPT / AMD NPT), **SSE4.2**, **POPCNT**.

```powershell
Get-CimInstance Win32_Processor |
  Select-Object Name, SecondLevelAddressTranslationExtensions, VirtualizationFirmwareEnabled
```

The Ryzen 7 5700X is Zen 3, which supports SSE4.2, POPCNT and NPT — so the CPU meets the
requirements with room to spare. Note the WMI query reported
`SecondLevelAddressTranslationExtensions : False` while a hypervisor was running; that is a
reporting artifact of the running hypervisor masking the CPU virtualisation bits, not an actual lack
of SLAT. (Coreinfo `-f` is the authority for SSE4.2/POPCNT and reads them even with a hypervisor
present, since those are plain instruction flags, not virtualisation features.)

### 2. Detect a running hypervisor

```powershell
Get-CimInstance Win32_ComputerSystem | Select-Object HypervisorPresent
```

Returned `True`. `msinfo32` System Summary confirmed *"A hypervisor has been detected. Features
required for Hyper-V will not be displayed."* — captured as the before-state screenshot.

### 3. Establish what was starting the hypervisor

Windows Features (`optionalfeatures`) showed **Hyper-V, Virtual Machine Platform, Windows Hypervisor
Platform, WSL and Windows Sandbox all unchecked** — so no optional feature was the cause.

`msinfo32` and `Win32_DeviceGuard` showed the real cause: **Virtualization-Based Security (VBS) was
running**, with *App Control for Business policy: Enforced*. Memory Integrity (HVCI) was the only VBS
service running initially.

### 4. Remediation attempted (documented in full because it did not fully succeed)

Turned off Memory Integrity (Core isolation → Off) and unchecked Virtual Machine Platform, then
rebooted. `HypervisorPresent` was still `True`. Diagnosis showed VBS itself was still running
(`VirtualizationBasedSecurityStatus : 2`) even though HVCI was now off — because VBS launches
through a **separate** boot switch from the hypervisor.

Applied both documented boot switches ([BCDEdit /set](https://learn.microsoft.com/en-us/windows-hardware/drivers/devtest/bcdedit--set)),
plus the VBS registry value, and confirmed they were written before rebooting:

```powershell
bcdedit /set hypervisorlaunchtype off
bcdedit /set vsmlaunchtype off
reg add "HKLM\SYSTEM\CurrentControlSet\Control\DeviceGuard" /v EnableVirtualizationBasedSecurity /t REG_DWORD /d 0 /f
bcdedit /enum "{current}" | Select-String "hypervisorlaunchtype|vsmlaunchtype"
# both returned: Off
```

After reboot, `HypervisorPresent` was **still `True`** and VBS was **still running**.

Read-only diagnosis established the boundary:

```powershell
Get-ItemProperty "HKLM:\SYSTEM\CurrentControlSet\Control\DeviceGuard" | Format-List
Get-ItemProperty "HKLM:\SOFTWARE\Policies\Microsoft\Windows\DeviceGuard" -ErrorAction SilentlyContinue | Format-List
```

- `EnableVirtualizationBasedSecurity : 0` — the setting applied correctly.
- The `\SOFTWARE\Policies\...\DeviceGuard` key does **not exist** — no Group Policy is forcing VBS.
- `msinfo32` *Required Security Properties* is **blank** — VBS is **not UEFI-locked**.

### 5. Decision

VBS is enabled by default on this Windows 11 build (26200) and resists the standard, documented
switches, with no policy override and no UEFI lock. The only remaining removal paths (removing the
active App Control / WDAC policy, or `DISABLE-LSA-ISO` EFI edits) carry a real risk of an unbootable
system — a documented failure mode — for a benefit that may not be needed. **Decision: leave VBS in
place and proceed.** VirtualBox 7.2 runs with the Hyper-V backend (paravirtualisation, the "green
turtle"); actual performance is to be measured on the first VM rather than assumed.

### VirtualBox and disks

```powershell
& "C:\Program Files\Oracle\VirtualBox\VBoxManage.exe" --version   # 7.2.8 r173730
Get-Volume | Where-Object DriveLetter -in 'C','E' |
  Select-Object DriveLetter, FileSystem, @{n='FreeGB';e={[math]::Round($_.SizeRemaining/1GB,1)}}
```

`C:` 446 GB free, `E:` 803 GB free — ample for the golden-image + linked-clone strategy. Working
folder confirmed at `C:\Users\Pavel\Desktop\Future Project AD DS` with `LAB-EXCHANGE\` present.

## Result

Host is ready to build the golden image (Task 01):

- CPU meets Server 2025 requirements (Zen 3: SSE4.2, POPCNT, NPT).
- VirtualBox 7.2.8 installed; hardware virtualisation (AMD-V) enabled in firmware.
- Ample disk on `C:` and `E:`.
- VBS remains enabled and could not be cleanly removed; the lab will run on VirtualBox's Hyper-V
  backend. Performance to be evaluated on the first VM; revisit only if it proves unusable.

## Lessons learned

- SSE4.2 and POPCNT are hard install gates in Server 2025, not advisories. On a Zen 3 / recent Intel
  CPU they are always present, but the check belongs before downloading the ISO, not after a failed
  install.
- The hypervisor and VBS launch through **two separate** boot switches: `hypervisorlaunchtype` and
  `vsmlaunchtype`. Turning off Memory Integrity (an HVCI *consumer*) does not turn off VBS itself —
  that was the mistake that cost two reboots. Both switches, plus the VBS registry value, are the
  documented set.
- On Windows 11 24H2/25H2 (build 26xxx), VBS is enabled by default and can resist even the correct
  switches. Before escalating, check two read-only things: the `\SOFTWARE\Policies\...\DeviceGuard`
  key (is a policy forcing it?) and `msinfo32` *Required Security Properties* (is it UEFI-locked?).
  Neither was set here — which is exactly what says "stop escalating," because the remaining paths
  are the ones that brick a machine.
- "VirtualBox can't run while Hyper-V is on" is out of date. It runs on the Hyper-V backend, slower.
  Whether that penalty actually matters is an empirical question answered by measuring, not by
  assuming — a correction to my own initial framing of this as a hard blocker.
- **Under assessment conditions:** the lab hands you a working host, so none of this is tested. The
  transferable skill is diagnosing *why* a hypervisor is present — feature vs. VBS vs. policy vs.
  UEFI lock — because each has a different, and differently risky, remedy.

## References

- Windows Server hardware requirements — https://learn.microsoft.com/en-us/windows-server/get-started/hardware-requirements
- BCDEdit /set (hypervisorlaunchtype, vsmlaunchtype) — https://learn.microsoft.com/en-us/windows-hardware/drivers/devtest/bcdedit--set
- Virtualisation applications do not work together with Hyper-V — https://learn.microsoft.com/en-us/troubleshoot/windows-client/application-management/virtualization-apps-not-work-with-hyper-v
- Enable virtualisation-based protection of code integrity — https://learn.microsoft.com/en-us/windows/security/hardware-security/enable-virtualization-based-protection-of-code-integrity
- Coreinfo (Sysinternals) — https://learn.microsoft.com/en-us/sysinternals/downloads/coreinfo

## Evidence

Evidence for this task is stored in:

```text
evidence/task-00-host-preparation/
```

Transcripts: `task00-transcript.txt` (part 1) plus the post-reboot verification sessions
(`part2`–`part4`), to be consolidated. No secrets in any part — no redaction required.
