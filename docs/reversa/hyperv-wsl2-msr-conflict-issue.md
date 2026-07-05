# Motion Assistant Hyper-V / WSL2 MSR Conflict Report

Upstream target:
- https://github.com/ThatUsernameAlreadyExist/Motion-Assistant-Power-Mod/issues
- GPD support channel if the official Motion Assistant maintainer prefers non-GitHub reports

Suggested title:

```text
[Bug/Architecture] Hard host freezes under WSL2/Hyper-V heavy workloads when Motion Assistant power polling is active
```

## Summary

When Motion Assistant is running with hardware monitoring, Auto-TDP, or repeated TDP reapply behavior enabled, the Windows host can hard-freeze during heavy WSL2 / Hyper-V workloads.

This is a full machine lockup: no recoverable UI, no clean BSOD, and often no crash dump. The only recovery is a physical power reset.

## Environment

- Device class: Windows handheld / mini-PC using Motion Assistant-style power controls
- Workload: WSL2 Ubuntu compiling very large Android / kernel trees
- Hypervisor: Hyper-V / WSL2 active, `vmmem` under sustained load
- Motion Assistant backend evidence:
  - `WinRing0x64.sys`
  - `MotionAssistant.sys`
  - `WinIo64.sys`
  - `msr-cmd.exe`
  - `ryzenadj.exe`

## Local Reproduction

1. Enable WSL2 / Hyper-V on Windows.
2. Start Motion Assistant with hardware monitoring, Auto-TDP, or TDP reapply behavior active.
3. Start a heavy WSL2 workload, for example an Android ROM build or kernel build.
4. Wait under sustained CPU and I/O load.
5. The host eventually hard-freezes and requires a physical reset.

## Local Static Intake Evidence

Reversa host-only intake classified the installed power stack as:

```text
WINDOWS_RING0_POWER_BACKEND_CONFLICT_MUTATION_BLOCKED
```

No Windows binaries were executed during this intake. It only hashed and parsed the installed assets.

Observed collision surface:

```text
inpoutx64.dll
msr-cmd.exe
ryzenadj.exe
winring0x64.dll
winring0x64.sys
MotionAssistant.sys
WinIo64.sys
```

Auto TDP asset summary:

```text
file_count: 31
backend_asset_count: 7
risk_line_count: 2557
critical_risk_line_count: 107
high_risk_line_count: 203
```

Motion Assistant asset summary:

```text
file_count: 119
backend_asset_count: 10
risk_line_count: 15
critical_risk_line_count: 0
high_risk_line_count: 0
```

Key point: the risk is not that Motion Assistant is malicious. The risk is that multiple Ring-0 / low-level power backends can poll or mutate CPU power/MSR state while Hyper-V is scheduling a very heavy virtualized workload.

## Why This Matters

Developers and power users increasingly use handhelds and mini-PCs as real workstations. WSL2, Docker Desktop, Android ROM builds, Linux kernel builds, and local AI tooling all create sustained Hyper-V activity.

Motion Assistant already targets the exact same users who are likely to use these devices for performance-heavy work. A safety rail here would prevent data loss and hard resets.

## Supporting Public Signals

- The Motion Assistant Power Mod repository is public and distributes Motion Assistant binary replacements, but it does not appear to expose the main application source.
- Public Motion Assistant documentation notes that MotionAssistant relies on WinRing0 for TDP, monitoring, and fan control.
- Recent public changelog text already mentions a fix for a potential system freeze when setting TDP repeatedly. This report narrows a related freeze class to Hyper-V / WSL2 heavy workloads.

## Proposed Mitigation

### 1. Detect Hyper-V / WSL2 and expose a visible safety mode

On startup and before entering any high-frequency polling loop:

- Detect Hyper-V presence.
- Detect WSL2/Docker activity via `vmmem`, `vmmemWSL`, `wslhost.exe`, `wsl.exe`, or Hyper-V services.
- If active, show a warning and switch to a conservative polling profile by default.

Suggested UI copy:

```text
Hyper-V / WSL2 activity detected.
High-frequency Ring-0 power polling may cause host instability during heavy VM workloads.
Motion Assistant has switched to conservative polling. You can override this in Advanced settings.
```

### 2. Add dynamic polling backoff

When a hypervisor workload is active:

- Stop sub-second MSR polling.
- Increase polling interval to 5-10 seconds.
- Debounce repeated TDP writes.
- Reapply TDP only on state change, AC/DC transition, resume, or explicit user request.

Suggested defaults:

```text
normal_poll_ms: 500-1000
hypervisor_safe_poll_ms: 5000-10000
minimum_tdp_write_spacing_ms: 30000
```

### 3. Separate read telemetry from write mutation

Use the lowest-risk telemetry source for read-only display when Hyper-V is active:

- Windows performance counters
- WMI / ACPI where available
- Vendor SDK APIs if available

Reserve WinRing0 / MSR / RyzenAdj calls for explicit TDP write operations, not continuous monitoring.

### 4. Add a global "VM / Developer Safe Mode"

Recommended behavior:

- Disable repeated TDP reapply loops.
- Disable aggressive CPU frequency lock refresh.
- Disable high-frequency MSR reads.
- Preserve manual one-shot TDP set if the user explicitly requests it.
- Keep fan curves and UI telemetry alive through safer APIs where possible.

## Proposed Acceptance Test

1. Enable WSL2.
2. Start Motion Assistant with VM-safe mode enabled.
3. Run a heavy WSL2 build for at least 60 minutes.
4. Confirm:
   - No host hard-freeze.
   - No rapid TDP rewrite loop.
   - Motion Assistant UI remains responsive.
   - TDP writes are debounced and logged.

## Minimal Diagnostic Logs Requested From Users

If maintainers want reproducible bug reports, request:

- Motion Assistant version.
- Device model and CPU.
- Windows build.
- Whether Hyper-V / WSL2 / Docker Desktop is enabled.
- Whether Auto-TDP, hardware monitoring, CPU frequency lock, and TDP reapply loop are enabled.
- Poll interval / monitoring interval if visible.
- Event Viewer entries after reboot.

## Closing

This is likely a concurrency and polling-policy bug, not a simple UI issue. A Hyper-V-aware polling backoff would protect developers and power users without removing Motion Assistant's core power-control features.
