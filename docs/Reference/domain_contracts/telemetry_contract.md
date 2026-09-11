# Telemetry Domain Contract

**Deployment module**: Telemetry | **CLI identifier**: `telemetry`

## Upstream domain contract

Telemetry consumes the Orchestrator inventory contract to resolve the service
Kubernetes virtual IP and the Slurm node groups used by enabled collectors.
The source currently expects an explicitly selected or staged inventory; it
does not automatically transfer the Orchestrator output into the Telemetry
project.

### `orchestrator_inventory.yaml`

**Producer location**:
`$OMNIA_DATA_PATH/orchestrator/output/$OMNIA_PROJECT_NAME/orchestrator_inventory.yaml`

#### Structure

The inventory follows the standard Ansible inventory hierarchy. Telemetry
uses `kube_vip_group` to resolve the service Kubernetes virtual IP and the
Slurm groups to identify collector targets. Other groups can be present.

```yaml
all:
  children:
    kube_vip_group:
      hosts:
        kube-vip:
          ansible_host: "192.0.2.10"
          ansible_user: "root"
    service_kube_control_plane_first_x86_64:
      hosts:
        service-kube-control-plane-1:
          ansible_host: "192.0.2.11"
          bmc_ip: "198.51.100.11"
          service_tag: "ABC1234"
          group_name: "grp1"
    slurm_control_node:
      hosts:
        slurm-control-1:
          ansible_host: "192.0.2.20"
          bmc_ip: "198.51.100.20"
          service_tag: "DEF5678"
          group_name: "grp2"
    slurm_node:
      hosts:
        slurm-node-1:
          ansible_host: "192.0.2.21"
          bmc_ip: "198.51.100.21"
          service_tag: "GHI9012"
          group_name: "grp2"
```

### `bmc_group_data.csv`

**Producer location**:
`$OMNIA_DATA_PATH/orchestrator/output/$OMNIA_PROJECT_NAME/bmc_group_data.csv`

**Required when**: iDRAC telemetry is enabled.

#### Structure

The first row defines the required columns. Each subsequent row maps one BMC
address to its functional group and, when applicable, its parent service tag.

```csv
BMC_IP,GROUP_NAME,PARENT
198.51.100.20,grp2,
198.51.100.21,grp2,DEF5678
```

Reference these generated files through `cluster_inventory` and, when iDRAC
telemetry is enabled,
`idrac_telemetry_configurations.bmc_group_data_path` in
`telemetry_config.yml`. The generated Orchestrator output is authoritative;
review it before using it as Telemetry input.

## Output contract

### Deployment and cleanup status

Telemetry writes one authoritative status file:

```text
$OMNIA_DATA_PATH/telemetry/output/$OMNIA_PROJECT_NAME/telemetry_status.yml
```

When `TELEMETRY_DATA_PATH` is set, the output is written below that root
instead.

| Field | Purpose |
|---|---|
| `domain` | Records `telemetry`. |
| `type` | Identifies a `deploy` or `cleanup` result. |
| `project_name` | Active project. |
| `overall_status` | `success`, `failed`, or `partial`. |
| `generated_at` | Generation timestamp. |
| `namespace` | Kubernetes namespace, normally `telemetry`. |
| `kube_vip` | Kubernetes control-plane VIP used by the workflow. |
| `packages` | Deployment package mode and repository URL. |
| `sinks` | Kafka, VictoriaMetrics, and VictoriaLogs results. |
| `sources` | Per-source metrics and, where supported, logs results. |
| `bridges` | Vector-LDMS and Vector-OME results. |
| `deploy_unreachable_nodes.ldms` | LDMS nodes skipped during deployment because they were unreachable. |

Component deployment values are `deployed`, `failed`, or `skipped`.

The root deployment currently calls the sink playbook without a derived sink
selection, so Kafka, VictoriaMetrics, and VictoriaLogs are deployed by
default. Status values report the state evaluated by the deployment workflow;
they do not prove end-to-end ingestion. In particular, PowerScale and UFM log
status reflects VLAgent availability, and the current VAST summary does not
consume its source-specific component check. Verify actual records in the
selected sink.

Cleanup rewrites the same file with `type: cleanup` and adds
`cleanup_components`, `cleanup_unreachable_nodes`, and a `volumes` block.
The `Delete_volume` or `delete_volume` boolean extra variable controls
whether persistent volume claims are deleted or preserved. The cleanup
workflow preserves `telemetry_status.yml` as the last-known result.

### Connection exports

| Tag | Output |
|---|---|
| `external_kafka` | `<TELEMETRY_DATA_PATH>/output/<project>/external_kafka/external_kafka_connect_details.yml`, `ca.crt`, `user.crt`, and `user.key`. |
| `external_victoria` | `<TELEMETRY_DATA_PATH>/output/<project>/external_victoria/external_victoria_connect_details.yml` and `ca.crt` when TLS is enabled. The YAML contains available VictoriaMetrics, VictoriaLogs, and VLAgent endpoints. |

These utilities fail when their required deployment or endpoint state is not
available instead of presenting an incomplete export as valid.

The domain supports setup, validation, precheck, deployment, cleanup, and the
two external connection exports. The `upgrade` and `rollback` operations are
placeholders in the current source and do not perform lifecycle changes.

## iDRAC MySQL runtime contract

MySQL is an implementation component of the Telemetry domain; it is not an
independent deployment domain.

| Resource | Runtime contract |
|---|---|
| StatefulSet | `idrac-telemetry` in the `telemetry` namespace, with one replica. |
| Container | `mysqldb`, using the MySQL image selected by `images.idrac.mysql`; the current default is `docker.io/library/mysql:9.7.2`. |
| Service | Internal headless service `idrac-telemetry-service`, with ports 3306 and 33060. It is not exported as a customer-facing MySQL endpoint. |
| Database | `idrac_telemetrydb`; the `services` table stores the iDRAC service inventory consumed by the receiver. |
| Secret | `mysqldb-credentials`, generated by the Telemetry credential workflow. |
| Persistent storage | `mysqldb-pvc-idrac-telemetry-0`, requested as ReadWriteOnce using `mysqldb_storage`. |
| Recovery initialization | The `cleanup-mysql-locks` init container removes stale `.sock` and `.pid` files after an ungraceful shutdown. |

Disabling iDRAC metrics scales the StatefulSet to zero replicas and preserves
the MySQL PVC. `cleanup_idrac` removes the source resources but also preserves
the PVC by default. Passing `Delete_volume=true` deletes the PVC and permanently
removes the stored service inventory.

The `services.auth` column contains authentication data used by the receiver.
Operational verification must not print or publish this column.

## Related documentation

- [Telemetry](../../HowTo/Telemetry/index.md)
- [Deploy the Telemetry stack](../../HowTo/Telemetry/deploy_telemetry.md)
- [Export Kafka connection details](../../HowTo/Telemetry/configure_external_kafka.md)
- [Export VictoriaMetrics connection details](../../HowTo/Telemetry/configure_external_victoria.md)
- [Export VictoriaLogs connection details](../../HowTo/Telemetry/configure_external_victoria_logs.md)
- [Telemetry configuration](../Configuration/telemetry_config.md)
