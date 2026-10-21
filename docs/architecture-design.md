# HA Cluster Architecture Design

## Solution Choice: Ceph + libvirt

I'm going with Ceph over LINSTOR/DRBD for this environment for the following reasons:

- We have "several hundred VMs per region" across multiple hosts. Ceph scales horizontally and doesn't require pairing hosts into replication groups the way DRBD does.
- Ceph decouples storage from compute. When a compute node dies, its VMs can be restarted on any other node because the storage is accessible cluster-wide via RBD.
- DRBD works well for two-node HA pairs, but managing hundreds of DRBD resources across a fleet gets operationally expensive. Ceph manages replication at the cluster level.
- Ceph integrates natively with libvirt/QEMU via librbd — no need for iSCSI or NFS layers.

The tradeoff: Ceph has higher operational complexity and requires dedicated hardware (monitors, OSDs). DRBD is simpler per-pair but doesn't solve the "restart VMs anywhere" problem.

## Cluster Topology

```
                    ┌─────────────────────────────────────────────┐
                    │              Ceph Cluster                   │
                    │                                             │
    ┌───────────┐   │   ┌──────┐  ┌──────┐  ┌──────┐            │
    │  Compute  │   │   │ MON  │  │ MON  │  │ MON  │            │
    │  vh-001   │───┼──▶│  01  │  │  02  │  │  03  │            │
    │  (VMs)    │   │   └──────┘  └──────┘  └──────┘            │
    └───────────┘   │                                             │
    ┌───────────┐   │   ┌──────────────────────────────────┐     │
    │  Compute  │   │   │        OSD Nodes                  │     │
    │  vh-002   │───┼──▶│  osd-01  osd-02  osd-03  osd-04 │     │
    │  (VMs)    │   │   │  osd-05  osd-06  osd-07  osd-08 │     │
    └───────────┘   │   │  (NVMe drives, 2-4 OSDs/node)    │     │
    ┌───────────┐   │   └──────────────────────────────────┘     │
    │  Compute  │   │                                             │
    │  vh-003   │───┼──▶  Network: 2x 25GbE bonded per node      │
    │  (VMs)    │   │    (dedicated storage VLAN)                 │
    └───────────┘   │                                             │
                    └─────────────────────────────────────────────┘
```

**Node roles:**
- 3 monitor/manager nodes (can be colocated on compute hosts if needed, but dedicated is better)
- N OSD nodes with NVMe drives — each NVMe is one OSD
- Compute nodes run only VMs + Ceph client (librbd)

**Network:**
- Dedicated storage network (25GbE or 100GbE bonded), separate from the VM traffic network
- Ceph public network for client ↔ OSD traffic
- Ceph cluster network for OSD ↔ OSD replication (separate from public to avoid replication traffic competing with client I/O)

## RPO and RTO Targets

| Scenario | RPO | RTO | Notes |
|----------|-----|-----|-------|
| Single OSD failure | 0 | < 1 minute | VMs continue running, Ceph handles it transparently |
| Single compute node failure | 0 | 5–10 minutes | VMs restart on other nodes with the same RBD volumes |
| Full rack failure | 0 | 10–15 minutes | CRUSH rules ensure replicas are in different racks |
| Ceph pool corruption (operator error) | 24 hours | 2–4 hours | Restore from daily backup |
| Region-level disaster | 24 hours | 4–8 hours | Restore from off-site backup |

RPO = 0 for hardware failures because Ceph replicates synchronously. The 24h RPO for backup scenarios assumes daily incremental backups.

## Split-Brain Prevention and Quorum

**Ceph monitors:**
- Deploy an odd number (3 or 5) of monitor daemons across separate failure domains
- Ceph uses Paxos for consensus — a majority of monitors must agree on the cluster map
- With 3 monitors, the cluster tolerates 1 monitor failure. With 5, it tolerates 2.
- Monitors should be spread across racks (ideally 3 different racks)

**Preventing split-brain on compute:**
- If a compute node loses connectivity to the Ceph cluster, its RBD clients will block I/O (timeout configurable) and then error out. This is correct behavior — writing to a disconnected node could cause data corruption.
- Configure `rbd_cache_max_dirty = 0` or a low value to minimize data at risk in client cache during a partition.
- QEMU with `werror=stop` pauses the VM on write errors rather than crashing it. When connectivity is restored, the VM can be resumed.

**Network partitions:**
- With 3 mons in racks A, B, C: if rack A is isolated, racks B+C still have quorum (2/3 mons). VMs in rack A pause (can't reach Ceph), VMs in B+C continue.
- Use separate network paths for monitor traffic and OSD traffic to reduce the chance of a partition affecting both.

## Failure Domains and Storage Replication

**CRUSH hierarchy:**

```
root default
├── rack rack01
│   ├── host osd-01
│   │   ├── osd.0  (nvme0)
│   │   └── osd.1  (nvme1)
│   └── host osd-02
│       ├── osd.2
│       └── osd.3
├── rack rack02
│   ├── host osd-03
│   │   ├── osd.4
│   │   └── osd.5
│   └── host osd-04
│       ├── osd.6
│       └── osd.7
└── rack rack03
    ├── host osd-05
    │   ├── osd.8
    │   └── osd.9
    └── host osd-06
        ├── osd.10
        └── osd.11
```

**Replication policy:**
- Pool type: replicated, size=3, min_size=2
- CRUSH rule: replicate across racks (`chooseleaf firstn 0 type rack`)
- This means each object has 3 copies, each on a different rack
- A single rack loss (fire, PDU failure, switch death) loses at most 1 copy — the pool remains fully available with min_size=2
- A single host loss loses at most 1 copy

**For the VM disk pool specifically:**

```
ceph osd pool create vm-disks 256 256 replicated
ceph osd pool set vm-disks size 3
ceph osd pool set vm-disks min_size 2
ceph osd crush rule create-replicated replicated_rack default rack
ceph osd pool set vm-disks crush_rule replicated_rack
```

## Backup vs. Replication

These serve different purposes and both are required:

**Replication (Ceph, synchronous):**
- Protects against hardware failure: disk death, node loss, rack loss
- RPO = 0 (synchronous)
- Does NOT protect against: accidental deletion, corruption bugs, ransomware, operator error
- If someone runs `rbd rm vm-disk-123`, all 3 replicas are deleted

**Backup (daily snapshots, async):**
- Protects against data corruption, accidental deletion, and disasters
- Uses RBD snapshots + rbd export-diff for incremental backups
- Backup target: separate Ceph cluster in a different datacenter, or S3-compatible object storage
- Retention: 7 daily, 4 weekly, 3 monthly
- RPO = 24 hours (or less with more frequent snapshots for premium customers)

**Backup implementation:**

```
rbd snap create vm-disks/vm-12345@daily-2026-09-09
rbd export-diff --from-snap daily-2026-09-08 vm-disks/vm-12345@daily-2026-09-09 - | \
  zstd | s3cmd put - s3://backups/vm-12345/incr-2026-09-09.diff.zst
```

Run this nightly per VM, staggered across the fleet to avoid I/O spikes.

## Capacity Overhead

**Replication overhead:**
- 3x replication means 1TB of VM data requires 3TB of raw storage
- Usable capacity = raw capacity / 3
- For 100TB usable: provision 300TB raw NVMe

**System overhead:**
- Ceph uses ~1-2% of OSD space for internal metadata (BlueStore DB + WAL)
- BlueStore WAL and DB on the same NVMe is fine for all-NVMe clusters (no need for separate SSD)
- Reserve 5% of each OSD for recovery headroom — Ceph needs free space to rebalance

**Monitor resources:**
- 3 monitor nodes, each with 16-32GB RAM for the monmap/OSDmap
- SSD storage for the mon database (not NVMe needed, just reliable)

**Network overhead:**
- 3x replication means each write generates 3x the network traffic
- A write-heavy workload doing 500MB/s of writes produces 1.5GB/s of replication traffic
- This is why a dedicated 25GbE+ storage network is mandatory

**Total capacity planning formula:**
```
Raw NVMe needed = (desired usable capacity * replication factor) / (1 - 0.05 - 0.02)
                = usable * 3 / 0.93
                ≈ usable * 3.23
```

For 100TB usable: ~323TB raw, spread across enough hosts to keep each host under 80% utilization for recovery headroom.

## Maintenance Procedures

**Adding a new OSD:**

1. Install the server, configure network, install Ceph packages
2. `ceph orch daemon add osd <host>:<device>` (or manual `ceph-volume lvm create`)
3. Ceph automatically starts rebalancing data onto the new OSD
4. Monitor with `ceph -w` — wait until rebalance completes before adding more

**Removing an OSD (planned):**

1. Set `noout` flag if removing multiple OSDs: `ceph osd set noout`
2. Reweight to 0: `ceph osd crush reweight osd.N 0`
3. Wait for data to drain (Ceph migrates all PGs off this OSD)
4. Stop the OSD: `systemctl stop ceph-osd@N`
5. Remove: `ceph osd out N && ceph osd purge N`
6. Clear `noout` if it was set

**Replacing a failed OSD:**

1. Ceph auto-marks the OSD as `down` and starts recovery after the `mon_osd_down_out_interval` (default 600s, can lower for NVMe)
2. Replace the physical drive
3. Create a new OSD on the new drive
4. Recovery happens automatically. Monitor with `ceph -s` and watch the recovery I/O.
5. Set `osd_recovery_max_active` and `osd_recovery_sleep` to throttle recovery if it impacts client I/O

**Rolling kernel/OS upgrades:**

1. Set `noout` on the host being upgraded: `ceph osd set-group noout <host>`
2. Migrate VMs off the compute node (if converged): `virsh migrate --live`
3. Stop all OSDs on the host
4. Upgrade the OS, reboot
5. Start OSDs, clear `noout`
6. Wait for peering to complete before moving to the next host

**Pool maintenance:**

- Scrubbing: Ceph deep-scrubs each PG weekly by default, comparing object checksums across replicas. Don't disable this.
- Schedule deep scrubs during low-traffic hours: set `osd_scrub_begin_hour` and `osd_scrub_end_hour`
- Monitor scrub errors: `ceph health detail` shows inconsistent PGs

**Monitoring the cluster:**

- `ceph -s` for overall health
- `ceph osd tree` for OSD layout and status
- `ceph osd perf` for OSD latencies
- Alert on: `HEALTH_WARN`, `HEALTH_ERR`, OSD down, PG stuck, near-full OSDs, slow requests
- Integrate with Prometheus via `ceph-mgr prometheus` module
