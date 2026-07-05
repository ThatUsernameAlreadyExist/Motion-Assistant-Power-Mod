# Motion Assistant Hyper-V / WSL2 Mitigation Proposal

This is a source-level design patch for upstream review. It is not an applyable Git diff because the public Motion Assistant Power Mod repository currently exposes binary releases rather than the main application source.

## Patch Goal

Prevent hard host freezes when Motion Assistant runs concurrently with heavy Hyper-V / WSL2 / Docker workloads.

The patch should not remove TDP controls. It should reduce risk by:

- Detecting active virtualization workloads.
- Backing off Ring-0 / MSR polling.
- Debouncing repeated TDP writes.
- Falling back to safer telemetry where possible.

## Proposed Components

### `VirtualizationGuard`

Responsibilities:

- Detect Hyper-V presence.
- Detect active WSL2 / Docker / VM processes.
- Return a conservative policy state for polling and TDP writes.

Pseudo-code:

```csharp
public sealed class VirtualizationGuard
{
    private static readonly string[] HypervisorProcessNames =
    {
        "vmmem",
        "vmmemWSL",
        "wsl",
        "wslhost",
        "vmcompute",
        "vmms",
        "Docker Desktop",
        "com.docker.backend"
    };

    public bool IsHypervisorLikelyActive()
    {
        if (Environment.GetEnvironmentVariable("WSL_INTEROP") != null)
            return true;

        return Process.GetProcesses()
            .Any(process => HypervisorProcessNames.Contains(
                process.ProcessName,
                StringComparer.OrdinalIgnoreCase));
    }
}
```

### `PowerPollingPolicy`

Responsibilities:

- Centralize polling intervals.
- Decide when WinRing0 / MSR / RyzenAdj reads are allowed.
- Decide when TDP writes are allowed.

Pseudo-code:

```csharp
public sealed class PowerPollingPolicy
{
    public int NormalPollMs { get; init; } = 1000;
    public int HypervisorSafePollMs { get; init; } = 8000;
    public int MinimumTdpWriteSpacingMs { get; init; } = 30000;

    private DateTime _lastTdpWriteUtc = DateTime.MinValue;

    public int GetPollIntervalMs(bool hypervisorActive)
    {
        return hypervisorActive ? HypervisorSafePollMs : NormalPollMs;
    }

    public bool CanWriteTdp(bool hypervisorActive, bool explicitUserAction)
    {
        if (explicitUserAction)
            return true;

        var elapsedMs = (DateTime.UtcNow - _lastTdpWriteUtc).TotalMilliseconds;
        if (elapsedMs < MinimumTdpWriteSpacingMs)
            return false;

        return !hypervisorActive;
    }

    public void MarkTdpWrite()
    {
        _lastTdpWriteUtc = DateTime.UtcNow;
    }
}
```

### Poll Loop Integration

Current risk pattern:

```text
timer tick -> read MSR / low-level sensor -> maybe write TDP -> repeat at high frequency
```

Proposed pattern:

```text
timer tick
  -> detect virtualization state
  -> select poll interval
  -> use safe telemetry if virtualization active
  -> only write TDP on state change, explicit user action, AC/DC transition, or resume
  -> debounce all automatic writes
```

Pseudo-code:

```csharp
private async Task PowerMonitorLoop(CancellationToken token)
{
    while (!token.IsCancellationRequested)
    {
        var hypervisorActive = _virtualizationGuard.IsHypervisorLikelyActive();
        var pollMs = _powerPollingPolicy.GetPollIntervalMs(hypervisorActive);

        if (hypervisorActive)
        {
            _ui.ShowVirtualizationSafeModeBanner();
            await ReadSafeTelemetryOnlyAsync(token);
        }
        else
        {
            await ReadFullHardwareTelemetryAsync(token);
        }

        if (_tdpState.HasChanged &&
            _powerPollingPolicy.CanWriteTdp(hypervisorActive, explicitUserAction: false))
        {
            await ApplyTdpAsync(_tdpState.TargetWatts, token);
            _powerPollingPolicy.MarkTdpWrite();
        }

        await Task.Delay(pollMs, token);
    }
}
```

### TDP Reapply Loop Integration

If Motion Assistant has a periodic TDP reapply feature, change it from unconditional periodic writes to state-driven writes:

```csharp
private bool ShouldReapplyTdp(bool hypervisorActive, TdpState current, TdpState desired)
{
    if (current.Equals(desired))
        return false;

    if (hypervisorActive && !desired.ChangeWasExplicitUserAction)
        return false;

    return _powerPollingPolicy.CanWriteTdp(
        hypervisorActive,
        desired.ChangeWasExplicitUserAction);
}
```

## UI / Settings

Add one Advanced setting:

```text
VM / Developer Safe Mode
```

Default:

```text
Auto
```

Options:

```text
Auto       Detect Hyper-V / WSL2 and back off polling automatically.
Always On Force conservative polling.
Off        Preserve current behavior.
```

## Logging

Add explicit log records:

```text
[PowerPolicy] Hyper-V/WSL2 detected: entering conservative polling mode.
[PowerPolicy] Skipping automatic TDP reapply because hypervisor workload is active.
[PowerPolicy] Manual TDP write allowed by explicit user action.
```

## Acceptance Criteria

- Heavy WSL2 build can run for 60 minutes with Motion Assistant open.
- No host hard-freeze.
- No repeated sub-second TDP writes while `vmmem` / WSL2 is active.
- Manual user-selected TDP writes still work.
- The UI tells the user why polling changed.

## Notes For Maintainers

The problem is not specific to one user workflow. Any sustained Hyper-V workload can increase interrupt scheduling pressure. Combining that with high-frequency Ring-0 polling or repeated MSR/power writes creates a plausible hard-freeze condition.

The conservative fix is to treat Hyper-V/WSL2 as a first-class runtime state and reduce low-level hardware access while it is active.
