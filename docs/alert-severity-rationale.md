# Alert Severity and Paging Policy

## Page Immediately (24/7)

These alerts represent active customer impact or imminent data loss. Waiting until business hours risks compounding damage.

| Alert | Why |
|-------|-----|
| HostDown | 300 VMs offline. Every minute counts. |
| RAIDDegraded | Running without redundancy. A second drive failure means data loss. |
| HighDiskLatency (>100ms, 5m) | At 100ms+ latency for 5 minutes, VMs are unusable. Likely drive failure in progress. |
| MemoryPressureCritical (>95%) | OOM killer will fire imminently, killing VM processes. |
| OOMKillDetected | A VM was just killed. Customer is already impacted. |
| VMCrashed | A customer VM terminated unexpectedly. |
| VMPausedIOError | A VM is frozen, customer's workload is halted. |
| CriticalLoadAverage (>4x CPUs) | The host is thrashing. VMs are unresponsive. |
| NetworkPacketLossCritical (>100/s) | Network is effectively down for VMs on this host. |
| NetworkInterfaceDown | Connectivity lost for all VMs on this interface. |
| DiskSpaceCritical (<5%) | VM provisioning will fail and running VMs may crash on next write. |
| NVMeCriticalWarning (>70C) | Thermal throttling or shutdown imminent. |

## Business Hours Only

These are degradations that need attention but aren't actively impacting customers yet. Waking someone up at 3 AM for a warning that can safely wait until morning adds fatigue without reducing risk.

| Alert | Why |
|-------|-----|
| DiskLatencyWarning (>20ms, 10m) | Elevated but not yet causing VM issues. Investigate at desk, not at 3 AM. |
| MemoryPressureHigh (>90%) | Concerning but not critical. Time to plan VM migration, not a 3 AM task. |
| HighLoadAverage (>2x CPUs, 15m) | Sustained but not at crisis level. Review next morning. |
| NetworkPacketLoss (>10/s) | Minor drops, probably transient. Worth investigating but not urgent. |
| NetworkReceiveErrors | Low rate, could be cabling or SFP degradation. Fix during maintenance window. |
| RAIDSyncInProgress | Expected after a disk replacement. Monitor but don't wake anyone. |
| VMDown (shutoff >5m) | Could be intentional (customer powered it off). Check during business hours. |
| LibvirtExporterDown | We lose metrics visibility, not VM availability. Fix when convenient. |
| DiskSpaceLow (<10%) | Time to clean up or provision more space, but not yet critical. |
| HighVMCount (>350) | Capacity planning signal, not an emergency. |

## Inhibition Rules

To prevent alert storms during cascading failures:

- `HostDown` inhibits all other alerts for that instance — if the host is unreachable, there's no point paging for disk latency, memory, etc. on the same box.
- `RAIDDegraded` inhibits `HighDiskLatency` for the same instance — the RAID failure already explains the latency spike.
