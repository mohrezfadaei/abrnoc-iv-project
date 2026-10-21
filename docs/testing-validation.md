# Testing and Validation

## Ansible Role Idempotency Test

### Setup

Test environment: Debian 12 (Bookworm) VM with 4 vCPUs, 8GB RAM, 40GB disk, nested virtualization enabled.

```bash
vagrant up debian12-test
ansible-playbook -i inventory/hosts site.yml --limit test-host
```

### First Run

The first run installs all packages, configures services, deploys templates. Expected output: a mix of `changed` and `ok` tasks.

```
PLAY RECAP
test-host : ok=52  changed=41  unreachable=0  failed=0  skipped=2  rescued=0  ignored=0
```

### Second Run (Idempotency Proof)

The second run should report zero changes:

```
PLAY RECAP
test-host : ok=52  changed=0  unreachable=0  failed=0  skipped=2  rescued=0  ignored=0
```

Key things that had to be addressed for idempotency:
- The `transparent_hugepages` write to `/sys` uses `changed_when: false` because sysfs writes always "change" the value even if it's already set
- Package installations use `state: present` not `state: latest` to avoid unnecessary upgrades
- Template deploys only trigger handlers if the rendered content actually differs
- The `unarchive` and `get_url` tasks for exporters use `changed_when: false` since the binary install step is what determines change

### Tasks Skipped

- `remove iptables` — skipped when iptables is not installed (conditional on package presence)
- `remove default nat network` — skipped when `libvirt_default_network: true`

## Alert Rule Testing

### Test 1: Firing HighDiskLatency

To simulate high disk latency, use `stress-ng` to create I/O pressure and a fio workload on a slow loop device:

```bash
dd if=/dev/zero of=/tmp/slowdisk.img bs=1M count=512
losetup /dev/loop0 /tmp/slowdisk.img

# add artificial latency via dm-delay
dmsetup create delayed --table "0 $(blockdev --getsz /dev/loop0) delay /dev/loop0 0 200"

# generate I/O against the delayed device
fio --name=lattest --filename=/dev/mapper/delayed --rw=randwrite --bs=4k \
    --size=100M --runtime=600 --time_based --direct=1
```

After 5 minutes with 200ms delay, the `HighDiskLatency` alert fires:

```
Alertmanager UI shows:
  Alert: HighDiskLatency
  State: firing
  Instance: test-host:9100
  Device: dm-0
  Summary: Disk latency above 100ms on test-host:9100
  Since: 2026-09-15T14:23:00Z
```

Cleanup:
```bash
dmsetup remove delayed
losetup -d /dev/loop0
rm /tmp/slowdisk.img
```

### Test 2: Firing MemoryPressureCritical

Allocate memory until the system hits 95%:

```bash
# check current available memory
free -m

# allocate enough to cross the 95% threshold
stress-ng --vm 1 --vm-bytes 7200M --vm-keep --timeout 300s
```

After 2 minutes above 95%, the alert fires:

```
Alertmanager UI shows:
  Alert: MemoryPressureCritical
  State: firing
  Instance: test-host:9100
  Summary: Memory usage above 95% on test-host:9100
  Since: 2026-09-15T14:31:00Z
```

The alert resolved within 1 minute after stress-ng terminated.

### Test 3: Firing RAIDDegraded

On the test host with a software RAID1 array:

```bash
mdadm --create /dev/md127 --level=1 --raid-devices=2 /dev/vdb1 /dev/vdc1
mdadm /dev/md127 --fail /dev/vdc1
```

The `RAIDDegraded` alert fires within 30 seconds:

```
Alertmanager UI shows:
  Alert: RAIDDegraded
  State: firing
  Instance: test-host:9100
  Device: md127
  Summary: RAID array md127 degraded on test-host:9100
```

Re-add the disk to clear:
```bash
mdadm /dev/md127 --remove /dev/vdc1
mdadm /dev/md127 --add /dev/vdc1
```

## Issues Encountered

1. **libvirt_exporter binary name mismatch**: The Tinkoff/libvirt-exporter release tarball names the binary `libvirt-exporter` (with a hyphen), but the systemd unit referenced `libvirt_exporter` (underscore). Fixed by updating the `copy` task source path to match the tarball layout.

2. **node_exporter textfile directory**: The systemd unit configures `--collector.textfile.directory=/var/lib/node_exporter/textfile`, but the role didn't create this directory. Added a task to create it with correct ownership. Without it, node_exporter logs warnings on every scrape.

3. **bridge-nf-call sysctl requires br_netfilter**: The `net.bridge.bridge-nf-call-iptables` sysctl fails if the `br_netfilter` kernel module isn't loaded first. Moved the module loading task before sysctl configuration in the execution order.

4. **chrony vs systemd-timesyncd conflict**: On a clean Debian 12, systemd-timesyncd is the default NTP client. Installing chrony doesn't automatically disable it. Added an explicit task to stop and disable systemd-timesyncd to avoid two NTP daemons competing.

5. **libvirt default network removal**: `community.libvirt.virt_net` fails if the default network was never defined (fresh install without it). Added `failed_when: false` to handle this gracefully.
