# SeaweedFS Ansible Deployment

This repository deploys a SeaweedFS cluster with Ansible, including:

- SeaweedFS masters
- volume servers
- filer servers
- S3 gateways
- Grafana Alloy log shipping for SeaweedFS service logs
- generated Prometheus scrape config for SeaweedFS metrics
- importable Grafana dashboard for SeaweedFS observability
- HAProxy for filer and S3 access
- optional post-deploy smoke and HA/replication tests

The playbook is inventory-driven. A host can belong to multiple groups, so the same node can run more than one SeaweedFS role.

## Current inventory layout

The current example inventory in [inventory.ini](/Users/renjith/Downloads/seaweedfs-ansible-full/inventory.ini:1) defines:

- `swfs-n1` (`10.5.93.212`): master, volume, haproxy
- `swfs-n2` (`10.5.93.213`): master, volume, filer, s3
- `swfs-n3` (`10.5.93.214`): master, volume, filer, s3
- `swfs-n4` (`10.5.93.215`): volume

Inventory groups:

- `seaweedfs_nodes`: all SeaweedFS nodes
- `masters`: SeaweedFS master servers
- `volumes`: volume servers
- `filers`: filer servers
- `s3`: S3 gateway servers
- `haproxy`: HAProxy nodes

## What the playbook does

The main playbook is [site.yml](/Users/renjith/Downloads/seaweedfs-ansible-full/site.yml:1). It runs in these stages:

1. Validates the inventory layout.
2. Installs SeaweedFS binaries and configures master, volume, filer, and S3 services on `seaweedfs_nodes`.
3. Installs Grafana Alloy on `seaweedfs_nodes` to ship SeaweedFS logs to Loki.
4. Generates a Prometheus scrape config under `monitoring/prometheus/`.
5. Installs and configures HAProxy on nodes in the `haproxy` group.
6. Optionally runs an end-to-end smoke test from a master node.
7. Optionally runs an HA and replication test from a master node.

## Prerequisites

- Ansible control host with SSH access to all target nodes
- `root` SSH access, or equivalent privilege escalation
- Python 3 available on managed nodes at `/usr/bin/python3`
- systemd-based Linux hosts
- A reachable SeaweedFS package server hosting tarballs under:
  `{{ seaweedfs_download_base_url }}/{{ seaweedfs_version }}/linux_amd64.tar.gz`
- MySQL reachable from filer nodes

Important notes:

- The playbook includes Debian APT proxy configuration and disables stale `file:///cdrom` package sources automatically.
- Service user and group bootstrap use `raw` commands for compatibility with older Ansible environments.

## Key configuration

Most settings live in [group_vars/all.yml](/Users/renjith/Downloads/seaweedfs-ansible-full/group_vars/all.yml:1).

### Core SeaweedFS settings

- `seaweedfs_version`: SeaweedFS version to deploy
- `seaweedfs_download_base_url`: base URL used to build the tarball download path
- `seaweedfs_install_dir`: extracted SeaweedFS directory on target nodes
- `seaweedfs_bin`: installed `weed` binary path
- `seaweedfs_user` / `seaweedfs_group`: service account

### Data and log paths

- `seaweedfs_master_data_dir`
- `seaweedfs_volume_data_dir`
- `seaweedfs_volume_idx_dir`
- `seaweedfs_filer_data_dir`
- `seaweedfs_log_dir`

### Ports

- `seaweedfs_master_port`
- `seaweedfs_master_grpc_port`
- `seaweedfs_master_metrics_port`
- `seaweedfs_volume_port`
- `seaweedfs_volume_grpc_port`
- `seaweedfs_volume_metrics_port`
- `seaweedfs_filer_port`
- `seaweedfs_filer_grpc_port`
- `seaweedfs_filer_metrics_port`
- `seaweedfs_s3_port`
- `seaweedfs_s3_grpc_port`
- `seaweedfs_s3_metrics_port`

### Observability

- `grafana_alloy_enabled`
- `grafana_alloy_version`
- `grafana_alloy_download_url`
- `grafana_alloy_loki_url`
- `grafana_alloy_loki_tenant_id`
- `grafana_alloy_loki_basic_auth_username`
- `grafana_alloy_loki_basic_auth_password`
- `seaweedfs_observability_cluster`
- `seaweedfs_prometheus_scrape_interval`

### Replication and filer backend

- `seaweedfs_default_replication`
- `seaweedfs_db_backend`
- `seaweedfs_db_host`
- `seaweedfs_db_port`
- `seaweedfs_db_name`
- `seaweedfs_db_user`
- `seaweedfs_db_password`
- `seaweedfs_filer_group`

### HAProxy

- `haproxy_filer_bind_ip`
- `haproxy_filer_port`
- `haproxy_s3_bind_ip`
- `haproxy_s3_port`

### S3 configuration

- `seaweedfs_s3_config_enabled`
- `seaweedfs_s3_access_key`
- `seaweedfs_s3_secret_key`

When enabled, the playbook renders `/etc/seaweedfs/s3.conf` on nodes in the `s3` group using [templates/s3.conf.j2](/Users/renjith/Downloads/seaweedfs-ansible-full/templates/s3.conf.j2:1).

### Optional test toggles

- `seaweedfs_run_smoke_test`
- `seaweedfs_run_ha_replication_test`
- `seaweedfs_test_replication`
- `seaweedfs_expected_replica_count`

## Inventory configuration

Each host in `seaweedfs_nodes` should define:

- `ansible_host`
- `seaweedfs_rack`
- `seaweedfs_data_center`

Example:

```ini
[seaweedfs_nodes]
swfs-n1 ansible_host=10.5.93.212 seaweedfs_rack=rack1 seaweedfs_data_center=dc1
swfs-n2 ansible_host=10.5.93.213 seaweedfs_rack=rack2 seaweedfs_data_center=dc1
swfs-n3 ansible_host=10.5.93.214 seaweedfs_rack=rack3 seaweedfs_data_center=dc1
swfs-n4 ansible_host=10.5.93.215 seaweedfs_rack=rack4 seaweedfs_data_center=dc1
```

Then place the host in the appropriate role groups:

```ini
[masters]
swfs-n1
swfs-n2
swfs-n3

[volumes]
swfs-n1
swfs-n2
swfs-n3
swfs-n4

[filers]
swfs-n2
swfs-n3

[s3]
swfs-n2
swfs-n3

[haproxy]
swfs-n1
```

## How to run

### Full deployment

```bash
ansible-playbook -i inventory.ini site.yml
```

### Run only HAProxy

```bash
ansible-playbook -i inventory.ini site.yml --limit haproxy
```

### Enable smoke test for a run

```bash
ansible-playbook -i inventory.ini site.yml -e seaweedfs_run_smoke_test=true
```

### Enable HA and replication test for a run

```bash
ansible-playbook -i inventory.ini site.yml -e seaweedfs_run_ha_replication_test=true
```

### Enable both tests

```bash
ansible-playbook -i inventory.ini site.yml \
  -e seaweedfs_run_smoke_test=true \
  -e seaweedfs_run_ha_replication_test=true
```

## What the tests do

### Smoke test

The smoke test runs once from a master node and:

- checks master cluster status
- checks topology and expected rack count
- uploads a test file through the filer VIP
- reads the file back through HAProxy

### HA and replication test

The HA test runs once from a master node and:

- uploads a file through HAProxy with a replication override
- verifies the file can be read
- stops the first filer temporarily
- reads the file again through HAProxy to confirm failover
- starts the filer again
- checks topology to confirm the expected number of volume nodes contain data

## Observability outputs

The playbook now enables SeaweedFS Prometheus pull metrics by adding `-metricsPort` to each SeaweedFS systemd unit:

- masters: `9325`
- volume servers: `9326`
- filers: `9327`
- S3 gateways: `9328`

SeaweedFS log shipping is handled by Grafana Alloy on each storage node. The default Loki push URL is:

- `http://10.5.93.216:3100/loki/api/v1/push`

If your Loki needs auth or tenant headers, set the related Alloy variables in [group_vars/all.yml](/Users/renjith/Documents/projects/seaweedfs-ansible-playbook/group_vars/all.yml:1).

### Prometheus scrape config

After running the playbook, import the generated scrape config from:

- [monitoring/prometheus/seaweedfs-scrape-config.yml](/Users/renjith/Documents/projects/seaweedfs-ansible-playbook/monitoring/prometheus/seaweedfs-scrape-config.yml:1)

Add that `scrape_configs` content into your Prometheus config on `http://127.0.0.1:9090/prometheus`, then reload Prometheus.

### Grafana dashboard

Import this dashboard JSON into Grafana:

- [monitoring/dashboards/seaweedfs-observability-dashboard.json](/Users/renjith/Documents/projects/seaweedfs-ansible-playbook/monitoring/dashboards/seaweedfs-observability-dashboard.json:1)

During import, map:

- `DS_PROMETHEUS` to your Prometheus datasource
- `DS_LOKI` to your Loki datasource

## Files in this repository

- [site.yml](/Users/renjith/Downloads/seaweedfs-ansible-full/site.yml:1): main playbook
- [inventory.ini](/Users/renjith/Downloads/seaweedfs-ansible-full/inventory.ini:1): inventory and host grouping
- [group_vars/all.yml](/Users/renjith/Downloads/seaweedfs-ansible-full/group_vars/all.yml:1): shared configuration
- [templates/seaweedfs-master.service.j2](/Users/renjith/Downloads/seaweedfs-ansible-full/templates/seaweedfs-master.service.j2:1): master service unit
- [templates/seaweedfs-volume.service.j2](/Users/renjith/Downloads/seaweedfs-ansible-full/templates/seaweedfs-volume.service.j2:1): volume service unit
- [templates/seaweedfs-filer.service.j2](/Users/renjith/Downloads/seaweedfs-ansible-full/templates/seaweedfs-filer.service.j2:1): filer service unit
- [templates/seaweedfs-s3.service.j2](/Users/renjith/Downloads/seaweedfs-ansible-full/templates/seaweedfs-s3.service.j2:1): S3 service unit
- [templates/haproxy.cfg.j2](/Users/renjith/Downloads/seaweedfs-ansible-full/templates/haproxy.cfg.j2:1): HAProxy config
- [templates/s3.conf.j2](/Users/renjith/Downloads/seaweedfs-ansible-full/templates/s3.conf.j2:1): S3 JSON config

## Operational notes

- Change `seaweedfs_db_password` to an Ansible Vault secret before production use.
- Update `seaweedfs_s3_access_key` and `seaweedfs_s3_secret_key` before exposing S3 publicly.
- If your package mirror changes, update `apt_proxy_url`, `apt_https_proxy`, and `seaweedfs_download_base_url`.
- HAProxy currently uses TCP health checks for filer and S3 backends.
