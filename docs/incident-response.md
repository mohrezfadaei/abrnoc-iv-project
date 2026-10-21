# Incident Response: Compute Node Failure with 300 VMs

## Scenario

A compute node hosting 300 customer VMs becomes unreachable. The server still has power but storage latency was climbing for several minutes before the host dropped off the network.

## First 15 Minutes

**T+0:00** — Alert fires: `HostDown` on vh-fra-017 plus preceding `HighDiskLatency` and `RAIDDegraded` alerts that fired minutes earlier.

**T+0:01** — Acknowledge the page. Open a dedicated incident channel. Post initial status: "Investigating loss of connectivity to vh-fra-017. ~300 VMs affected."

**T+0:02** — Check the obvious from a different host in the same rack:
- `ping vh-fra-017` and `ping` its IPMI/BMC address
- If IPMI responds, the server is physically alive — the OS or network stack is the problem
- Check the upstream switch port via switch CLI: is the port up/down, any CRC errors?

**T+0:04** — Pull Prometheus and Grafana history for the last 30 minutes:
- Disk latency graphs (was it one device or all?)
- RAID status (md0 state changes)
- Memory usage and OOM events
- Network drop counters and interface state
- Load average
- VM count and state changes

The monitoring data tells the story before we even touch the box: NVMe failure preceded everything.

**T+0:07** — If IPMI is reachable, open a remote console (iKVM/SOL). Look at what the kernel is doing:
- Is there a kernel panic on screen?
- Is it stuck in D-state I/O wait?
- Any NVMe errors visible in the console buffer?

**T+0:10** — Query the control plane database for the full VM inventory on this host. Get a list of all 300 VMs, their customer IDs, and their storage locations. This list drives the recovery.

**T+0:12** — Assess scope: are other hosts in the same rack affected? Check their metrics. If yes, this could be a rack-level power or network event. If only this host, it's isolated hardware failure.

**T+0:14** — Make the first customer-facing status page update: "We are aware of connectivity issues affecting a subset of VMs in [region]. Our engineering team is actively investigating. We will provide updates every 15 minutes."

## Determining the Fault Domain

The approach is process of elimination, starting with the most likely cause given the symptom pattern (storage latency climbing before network loss):

**Storage:**
- The strongest signal. Climbing latency followed by host death is textbook NVMe failure.
- From IPMI console: check `dmesg` for NVMe I/O timeout messages, controller resets, device removal
- Check `cat /proc/mdstat` — is the RAID degraded?
- Check `smartctl -a /dev/nvme0n1` and `nvme1` for media errors, critical warnings, temperature
- If the drive completely failed, the dm-thin pool would go read-only, EXT4 journals would abort, VMs would crash or pause, dirty pages would accumulate in memory, OOM killer would fire

**Network:**
- Check from IPMI console: `ip link show`, `ethtool enp65s0f0`
- Check the switch side: port counters, SFP diagnostics
- Look for ixgbe driver messages in dmesg — TX queue timeouts would indicate a kernel-level network problem, not cable/switch
- If network interfaces show frames dropped and TX watchdog timeouts, but these started *after* storage issues, the network failure is a secondary symptom (kernel threads blocked on I/O, no CPU time for network processing)

**Kernel:**
- Check for kernel panic, hung tasks, D-state processes
- `cat /proc/loadavg` — if readable, shows whether load exploded
- Check if the kernel is responsive at all via IPMI SysRq: `Alt+SysRq+w` to dump blocked tasks
- A kernel that's alive but unresponsive to network traffic is usually stuck in I/O wait from storage failure

**Hardware (broader):**
- Check IPMI event log for hardware events: PSU failures, fan failures, memory ECC errors
- Check motherboard error LEDs if datacenter hands-on access is available
- If IPMI itself is unreachable, dispatch datacenter staff to visually inspect

**In this scenario**, the causal chain is almost certainly: NVMe hardware failure → RAID degradation → thin pool failure → filesystem read-only → VM crashes → memory pressure from dirty page writeback → OOM kills → kernel threads blocked → network driver starved of resources → network watchdog timeout → host unreachable.

## Conditions for Rebooting the Host

A reboot is not the first choice. It's appropriate when:

1. **The kernel is not recoverable** — stuck in a state where SysRq commands don't respond, or the console shows a hard panic
2. **The failed NVMe drive is isolated to a single RAID member** and the other drive is healthy — after reboot, the system comes up on the surviving RAID member with degraded md0
3. **All VMs on the failed storage are already dead** — rebooting doesn't make things worse
4. **We need the host back to verify RAID state and assess data integrity** — you can't do this on a completely hung kernel

Before rebooting:
- Try `SysRq+s` (sync), `SysRq+u` (remount read-only), then `SysRq+b` (reboot) — this is safer than hard power cycle
- If SysRq is unresponsive, use IPMI to do a graceful shutdown first, then hard reset only if that hangs
- Document the console output and screenshot it before losing the state

Do NOT reboot if:
- Some VMs are still running and serving traffic on the healthy storage — rebooting would kill those too
- We haven't yet captured enough diagnostic data and the console is still readable

## Recovering the 300 VMs

**Phase 1: Assessment (first 30 minutes post-reboot)**

If the host comes back up:
- Check RAID status: `cat /proc/mdstat`. If md0 is running on 1 device, we have the surviving mirror.
- Check dm-thin pool: `dmsetup status`. If the thin pool is in error state, we need manual recovery.
- List VMs: `virsh list --all`. Categorize them:
  - Whose storage was on the failed NVMe only (no mirror) — worst case, data loss
  - Whose storage was on the RAID — can potentially be recovered
  - Which VMs had `werror=stop` (paused) vs `crashed`

**Phase 2: Recover what's recoverable**

For VMs on the surviving RAID member:
- Run filesystem checks on the LVs: `fsck` the thin volumes
- If clean, start the VMs: `virsh start <domain>`
- If filesystem corruption is found, restore from the most recent backup

For VMs on the failed NVMe only (if any):
- If there's no mirror or backup, the data is lost from this host
- Restore from backup: pull the latest VM snapshot from the backup system
- The RPO depends on backup frequency — if daily, up to 24h of data could be lost

**Phase 3: Migrate to healthy hosts**

Even if VMs can be restarted on this host, don't keep 300 VMs on a degraded RAID:
- Live-migrate VMs to other hosts with capacity: `virsh migrate --live <domain> qemu+ssh://vh-fra-XXX/system`
- Prioritize VMs based on customer tier and current outage duration
- Batch migrations to avoid overloading the network (20-30 VMs at a time)

**Phase 4: Validate**

- Confirm each VM is reachable from the customer's perspective
- Check application-level health where possible (HTTP probes, DNS, etc.)
- Update the control plane database with the new host assignments

## Preventing Recurrence

**Short term:**
- Replace the failed NVMe drive and rebuild the RAID
- Run extended SMART tests on the surviving drive — if the drives are the same batch and age, the second one is at elevated risk
- Check all other hosts for similar NVMe health indicators: `smartctl -a` across the fleet, look for media errors, wear percentage, critical temperature events

**Medium term:**
- Improve monitoring: add alerts for NVMe SMART attributes (media errors, available spare below threshold, critical warning bits) — these are leading indicators that fire before I/O timeouts
- Add a `smartctl` textfile collector for node_exporter that exposes per-drive health as Prometheus metrics
- Set up predictive disk replacement: if spare capacity drops below 10% or media errors appear, schedule proactive replacement during a maintenance window instead of waiting for failure

**Long term:**
- Move from local RAID1 to a distributed storage solution (Ceph or LINSTOR) — if a node dies, VMs can be restarted on another node immediately because the storage is replicated across multiple servers, not tied to local drives
- Implement automated VM failover: the control plane detects host death and automatically restarts VMs on other hosts
- Reduce VM density per host: 300 VMs on one host means one failure has a massive blast radius. With distributed storage, this matters less since recovery is automatic, but with local storage, consider a cap of 150-200 per host

## Communication

**Internal:**

| When | What | To Whom |
|------|------|---------|
| T+0 | "vh-fra-017 unreachable, ~300 VMs impacted, investigating" | Incident channel, on-call team |
| T+5 | "Initial assessment: NVMe failure suspected based on preceding storage latency alerts. Attempting IPMI console access." | Incident channel |
| T+15 | "Confirmed: NVMe1 failed, RAID degraded, host kernel hung. Planning controlled reboot." | Incident channel, engineering lead |
| T+30 | "Host rebooted. Assessing VM recoverability. Beginning VM recovery." | Incident channel |
| T+60 | Progress update with recovery percentage | Incident channel |
| Post-recovery | Full incident timeline, root cause confirmed | Post-mortem meeting |

**Customer-facing (status page):**

| When | What |
|------|------|
| T+5 | "We are investigating connectivity issues affecting some VMs in [region]." |
| T+15 | "We have identified the cause: a hardware failure on one of our compute nodes. Our team is working on recovering affected VMs." |
| T+30 | "Recovery is in progress. Some VMs are being restored and will come back online over the next 30-60 minutes." |
| T+60 | "Most VMs have been recovered. We are working on the remaining [N] VMs." |
| Recovery complete | "All affected VMs have been restored. We will publish a post-mortem within 48 hours." |
| Post-mortem (48h) | Public-facing incident report with root cause, timeline, and prevention measures |

Customers with premium SLA should also get direct email or phone notification from their account manager at T+15 and at resolution.
