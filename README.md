# AbrNOC DevOps Task: KVM Virtualization Host

Provisioning automation, monitoring, and operational documentation for a KVM-based VPS platform running on Debian.

## Repository Structure

```
.
├── ansible/
│   ├── ansible.cfg
│   ├── inventory/hosts
│   ├── site.yml
│   └── roles/kvm_host/
│       ├── defaults/main.yml
│       ├── handlers/main.yml
│       ├── meta/main.yml
│       ├── tasks/
│       │   ├── main.yml
│       │   ├── users.yml
│       │   ├── firewall.yml
│       │   ├── ntp.yml
│       │   ├── kernel.yml
│       │   ├── logging.yml
│       │   ├── packages.yml
│       │   ├── libvirt.yml
│       │   ├── networking.yml
│       │   └── monitoring.yml
│       └── templates/
├── monitoring/
│   ├── prometheus/
│   │   ├── prometheus.yml
│   │   └── rules/
│   │       ├── host_alerts.yml
│   │       └── vm_alerts.yml
│   └── alertmanager/
│       └── alertmanager.yml
├── docs/
│   ├── incident-response.md
│   ├── architecture-design.md
│   ├── log-analysis.md
│   ├── alert-severity-rationale.md
│   └── testing-validation.md
├── Vagrantfile
└── README.md
```

## Quick Start

### Prerequisites

- Ansible 2.14+
- Target: Debian 12 (Bookworm) with hardware virtualization support
- Python 3 on the target host
- Ansible collections: `ansible.posix`, `community.general`, `community.libvirt`

### Install Dependencies

```bash
ansible-galaxy collection install ansible.posix community.general community.libvirt
```

### Configure Inventory

Edit `ansible/inventory/hosts` with your target hosts:

```ini
[kvm_hosts]
vh-fra-001 ansible_host=10.40.1.1
vh-fra-002 ansible_host=10.40.1.2

[kvm_hosts:vars]
ansible_user=ansible
ansible_python_interpreter=/usr/bin/python3
```

### Run the Playbook

```bash
cd ansible
ansible-playbook site.yml
```

For a specific host:
```bash
ansible-playbook site.yml --limit vh-fra-001
```

For specific tasks only:
```bash
ansible-playbook site.yml --tags "firewall,ssh"
```

### Verify Idempotency

Run the playbook a second time. The output should show `changed=0`:

```bash
ansible-playbook site.yml
```

## Configuration

All tunables are in `ansible/roles/kvm_host/defaults/main.yml`. Key settings:

| Variable | Default | Description |
|----------|---------|-------------|
| `ssh_port` | 22 | SSH listen port |
| `ssh_permit_root_login` | no | Disable root SSH |
| `ntp_servers` | DE pool | NTP server list |
| `vm_swappiness` | 1 | Minimize swap usage for VM hosts |
| `kernel_panic_timeout` | 10 | Auto-reboot 10s after kernel panic |
| `libvirt_pool_path` | /var/lib/libvirt/images | VM disk storage path |
| `bridge_interface` | br0 | Bridge name for VM networking |
| `node_exporter_version` | 1.8.2 | Prometheus node exporter version |

Override any variable in your inventory or a group_vars file:

```yaml
# group_vars/kvm_hosts.yml
ssh_port: 2222
ntp_servers:
  - ntp1.internal.example.com
  - ntp2.internal.example.com
```

## Monitoring Setup

### Prometheus

Copy the Prometheus configuration to your Prometheus server:

```bash
cp monitoring/prometheus/prometheus.yml /etc/prometheus/prometheus.yml
cp monitoring/prometheus/rules/*.yml /etc/prometheus/rules/
systemctl reload prometheus
```

Validate alert rules:

```bash
promtool check rules monitoring/prometheus/rules/*.yml
```

### Alertmanager

Update the placeholder values in `monitoring/alertmanager/alertmanager.yml` (Slack webhook URLs, PagerDuty service keys) and deploy:

```bash
cp monitoring/alertmanager/alertmanager.yml /etc/alertmanager/alertmanager.yml
amtool check-config monitoring/alertmanager/alertmanager.yml
systemctl reload alertmanager
```

## Testing with Vagrant

```bash
vagrant up debian12-test
vagrant provision debian12-test    # run playbook
vagrant provision debian12-test    # second run for idempotency
```

The Vagrantfile provisions a Debian 12 VM with nested virtualization and two extra disks for RAID testing.

## Documentation

- [Incident Response Plan](docs/incident-response.md) — runbook for a compute node failure affecting 300 VMs
- [HA Architecture Design](docs/architecture-design.md) — Ceph-based HA cluster design with RPO/RTO targets, CRUSH rules, and maintenance procedures
- [Log Analysis](docs/log-analysis.md) — root cause analysis of the vh-fra-017 journal excerpt
- [Alert Severity Rationale](docs/alert-severity-rationale.md) — which alerts page at 3 AM and which wait
- [Testing and Validation](docs/testing-validation.md) — idempotency test results, alert firing demos, issues found

## Assumptions and Trade-offs

### What I Built

- Full Ansible role for Debian KVM host provisioning, covering all items listed in the task: users, SSH hardening, firewall (nftables), NTP (chrony), kernel tuning, logging (journald + rsyslog), packages, libvirt/KVM, bridge networking, and monitoring exporters
- Prometheus alert rules for all six categories: disk latency, RAID degradation, high load, memory pressure, VM failure, and network packet loss
- Alertmanager config with severity-based routing and inhibition rules
- Written deliverables for incident response, architecture design, and log analysis

### What I Cut

- **Terraform integration**: The task mentions "Terraform where applicable" but the primary ask is Ansible provisioning. Terraform would come in for VM lifecycle management (creating/destroying VMs via the control plane), not host provisioning. I'd add a Terraform provider for libvirt to manage VM definitions declaratively if given more time.

- **Automated VM failover scripts**: The incident response document describes the process but I didn't write the actual failover automation. In production, this would be a control plane feature (detect host death via monitoring, query DB for VM list, restart VMs on hosts with capacity, update DNS/routing).

- **Ceph deployment automation**: The architecture design describes the Ceph cluster in detail but doesn't include Ansible playbooks for deploying it. Ceph deployment is a substantial project on its own (cephadm or ceph-ansible), and the task prioritizes depth over breadth.

- **SMART monitoring integration**: Mentioned in the incident response as a prevention measure. Would implement as a cron job running `smartctl --json` and writing to the node_exporter textfile directory, exposing NVMe health attributes as Prometheus metrics.

- **Grafana dashboards**: The monitoring stack uses Prometheus and Alertmanager, but I didn't create Grafana dashboard JSON files. The standard node-exporter-full dashboard from Grafana Labs covers most host metrics. A custom dashboard for VM state and per-host VM counts would be the next addition.

### What I'd Do With More Time

1. Write a Molecule test suite for the Ansible role with Docker or Vagrant backends
2. Add a `smartctl` textfile collector and NVMe-specific alert rules based on SMART attributes
3. Build the Ceph deployment playbook with cephadm integration
4. Write the automated VM failover script that the control plane would trigger
5. Create Grafana dashboards for host and VM monitoring
6. Add a Terraform module for VM lifecycle management
7. Implement backup automation: RBD snapshot + export-diff pipeline with retention management
8. Write an Ansible playbook for the Prometheus/Alertmanager/Grafana stack deployment
