# Log Analysis: vh-fra-017 Journal Excerpt

## Root Cause

The root cause is a hardware failure of NVMe drive `nvme1`. The drive's controller became unresponsive, failed its reset sequence, and was removed from the system by the kernel. Everything that follows is a downstream consequence.

## Causal Chain

Here's the sequence broken down step by step:

### 1. NVMe I/O Timeouts (03:13:58)

```
kernel: nvme nvme1: I/O 388 QID 6 timeout, aborting
kernel: nvme nvme1: I/O 389 QID 6 timeout, aborting
kernel: nvme nvme1: I/O 390 QID 6 timeout, aborting
kernel: nvme nvme1: I/O 12 QID 0 timeout, aborting
```

Multiple I/O commands timed out on queue 6 and the admin queue (QID 0). The admin queue timeout is significant — it means even management commands to the drive are failing, not just data I/O. This is a hardware-level failure, not a transient load issue.

### 2. Controller Reset Failure (03:14:28 — 03:14:59)

```
kernel: nvme nvme1: I/O 388 QID 6 timeout, reset controller
kernel: nvme nvme1: Abort status: 0x371
kernel: nvme nvme1: Device not ready; aborting reset, CSTS=0x1
kernel: nvme nvme1: Disabling device after reset failure
kernel: nvme nvme1: Removing after probe failure status: -19
```

30 seconds after the initial timeout, the kernel attempted a controller reset. The abort returned status 0x371 (multiple error bits set). The drive's Controller Status register (CSTS) remained 0x1 (not ready after reset). The kernel gave up and removed the device entirely. Status -19 is `ENODEV` — the device no longer exists as far as the kernel is concerned.

### 3. RAID Degradation (03:14:59)

```
kernel: nvme1n1: detected capacity change from 7501476528 to 0
kernel: blk_update_request: I/O error, dev nvme1n1, sector 4128768 op 0x1:(WRITE)
kernel: md/raid1:md0: Disk failure on nvme1n1p2, disabling device.
kernel: md/raid1:md0: Operation continuing on 1 devices.
```

The NVMe namespace reported 0 capacity (device gone). Pending writes immediately failed. The md RAID1 array detected the failure and kicked `nvme1n1p2` from the mirror. The array continued operating on one device — this is by design, but now there's no redundancy.

### 4. Device-Mapper Thin Pool Failure (03:14:59)

```
kernel: device-mapper: thin: 253:2: metadata operation 'dm_thin_insert_block' failed: error = -5
kernel: device-mapper: thin: 253:2: aborting current metadata transaction
kernel: device-mapper: thin: 253:2: switching pool to read-only mode
```

Error -5 is `EIO`. The thin provisioning pool's metadata update failed (it was trying to allocate a block). This could mean the metadata device was on the failed NVMe, or the I/O storm from the failure corrupted an in-flight transaction. The pool switched to read-only as a safety measure — this is correct behavior, it prevents metadata corruption from spreading.

### 5. EXT4 Filesystem Failures (03:15:00)

```
kernel: EXT4-fs error (device dm-7): ext4_journal_check_start:83: Detected aborted journal
kernel: EXT4-fs (dm-7): Remounting filesystem read-only
kernel: EXT4-fs error (device dm-12): ext4_journal_check_start:83: Detected aborted journal
kernel: EXT4-fs (dm-12): Remounting filesystem read-only
```

Two thin volumes (dm-7 and dm-12, which are VM disk images) had their EXT4 journals aborted because the underlying thin pool went read-only. These filesystems remounted read-only. Any VM writing to these disks can no longer commit data.

### 6. VM Failures (03:15:02 — 03:15:03)

```
libvirtd[1418]: qemu unexpectedly closed the monitor: I/O error on nvme1n1
libvirtd[1418]: Domain id=41 name='vm-2810447' terminated abnormally (crashed)
libvirtd[1418]: Domain id=73 name='vm-2810552' paused (I/O error: werror=stop)
libvirtd[1418]: Domain id=118 name='vm-2810931' paused (I/O error: werror=stop)
```

Three distinct failure modes appeared:
- `vm-2810447` crashed because QEMU's monitor socket got an I/O error (its virtual disk was directly on the failed NVMe)
- `vm-2810552` and `vm-2810931` paused because they were configured with `werror=stop` — QEMU pauses the VM instead of crashing it on write error. This is the safer configuration and these VMs could theoretically be resumed if the storage comes back.

### 7. Metrics Exporter Death (03:15:04)

```
vm-metrics-exporter[2214]: FATAL: cannot open /var/lib/libvirt/images/.metrics/state.db: Read-only file system
systemd[1]: vm-metrics-exporter.service: Failed with result 'exit-code'.
```

The metrics exporter tried to write to its state database, which is on the now read-only filesystem. It crashed and systemd started restart attempts. This is a downstream casualty with no impact on VMs.

### 8. Hung Tasks and OOM (03:15:31 — 03:16:12)

```
kernel: INFO: task jbd2/dm-7-8:3391 blocked for more than 120 seconds.
kernel: qemu-system-x86 invoked oom-killer
kernel: dirty:6118432 writeback:2941117
kernel: Out of memory: Killed process 88213 (qemu-system-x86) total-vm:17371512kB
libvirtd[1418]: Domain id=204 name='vm-2811207' terminated abnormally (crashed)
```

The JBD2 journal thread for dm-7 was stuck in D-state (uninterruptible sleep) for over 120 seconds trying to write. Meanwhile, dirty pages accumulated in memory (6.1M dirty pages + 2.9M in writeback, roughly ~35GB of dirty data). The kernel couldn't reclaim memory because it couldn't flush dirty pages to a dead storage device. This triggered the OOM killer, which chose a QEMU process (`vm-2811207`, 16.8GB RSS) — killing a customer VM.

### 9. Network Degradation (03:16:44 — 03:18:02)

```
kernel: ixgbe: RX ring 0 dropped 3 frames
kernel: ixgbe: RX ring 0 dropped 27 frames
kernel: ixgbe: RX ring 0 dropped 412 frames
kernel: ixgbe: RX ring 0 dropped 2201 frames
kernel: NETDEV WATCHDOG: enp65s0f0 (ixgbe): transmit queue 4 timed out
kernel: ixgbe: Reset adapter
```

Network packet drops escalated rapidly (3 → 27 → 412 → 2201). The cause: the CPU was overloaded with I/O wait and OOM handling, so the ixgbe driver couldn't service its RX ring buffers in time. The TX watchdog timer fired because a transmit queue had been stalled — the kernel couldn't process outgoing packets either. The driver attempted an adapter reset at 03:18:02, and this is approximately when the host became unreachable.

The network failure is purely secondary. The NIC hardware was fine — the ixgbe driver just couldn't get CPU time to process packets because every CPU was stuck in I/O wait or OOM handling.

## What's Noise

- **CRON entry (03:11:52)**: `vm-snapshot-rotate.sh` — a scheduled snapshot rotation. Unrelated to the failure but worth checking whether it puts additional I/O pressure. It ran about 2 minutes before the NVMe timeouts started. Unlikely to be causal, but if this snapshot script was writing heavily to the same NVMe, it could have been the final stressor on an already-dying drive.

- **SSH session (03:12:40 — 03:12:47)**: Ansible connected, ran something, disconnected. A 7-second session is probably a health check or fact gather. Not related to the failure.

- **vm-metrics-exporter restart loops (03:15:09, 03:15:14)**: Systemd restarting the exporter that can't write to a read-only filesystem. Pure noise — it will keep failing until the filesystem is writable again.

- **node_exporter diskstats warning (03:17:03)**: `read /sys/block/nvme1n1/stat: no such file or directory` — the device was removed from the system, so the diskstats collector can't find it. Expected downstream symptom.

## Timeline Summary

| Time | Event | Type |
|------|-------|------|
| 03:11:52 | Snapshot rotation cron runs | Noise |
| 03:12:40 | Ansible SSH session | Noise |
| 03:13:58 | NVMe1 I/O timeouts begin | Root cause |
| 03:14:28 | NVMe1 controller reset attempted | Root cause |
| 03:14:59 | NVMe1 removed, RAID degraded, thin pool fails | Direct consequence |
| 03:15:00 | EXT4 journals abort, filesystems go read-only | Downstream |
| 03:15:02 | VMs crash and pause | Downstream |
| 03:15:04 | Metrics exporter dies | Noise |
| 03:15:31 | Hung tasks detected (120s blocked) | Downstream |
| 03:16:12 | OOM killer fires, kills a VM | Downstream |
| 03:16:44 | Network packet drops escalate | Downstream |
| 03:18:02 | NIC TX watchdog fires, adapter reset, host unreachable | Downstream |
