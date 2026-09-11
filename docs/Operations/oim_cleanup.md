# OIM Cleanup

Use the domain cleanup workflows to remove services and artifacts owned by
each Omnia domain. After all required domain cleanups complete, use the main
cleanup command to remove the shared Omnia execution environment.

Cleanup is performed directly on the OIM. In the domain-based architecture,
each domain owns and exposes its cleanup workflow.

!!! danger

    Cleanup is destructive. Depending on the selected domain and options, it
    can remove containers, repositories, images, credentials, cluster
    configuration, telemetry workloads, persistent data, and shared-storage
    content. Back up all required data before continuing.

## When to Use OIM Cleanup

- To reset a failed or experimental deployment.
- To remove selected Omnia domains before redeployment.
- To return a lab or test OIM to a clean state.
- To remove the shared Omnia environment after domain services are removed.

## Prerequisites

- Log in to the OIM as a user with the privileges required by the selected
  cleanup workflows.
- Use the same `OMNIA_DATA_PATH` and `OMNIA_PROJECT_NAME` that were used for
  deployment.
- Stop or drain workloads that use the services being removed.
- Back up project inputs, credentials, repository content, images, telemetry
  data, databases, and shared-storage data that must be retained.
- Confirm that the Omnia virtual environment is available. Run the main
  cleanup command only after the domain cleanup commands complete.

## Cleanup Scope by Domain

| Domain | Cleanup scope |
|---|---|
| `build_stream` | Removes GitLab and BuildStreaM services, the watcher, PostgreSQL service, NFS artifacts, and BuildStreaM credentials. PostgreSQL data is preserved by default. |
| `telemetry` | Removes enabled telemetry sources and sinks. Persistent volumes, including the iDRAC MySQL PVC, are preserved by default. |
| `orchestrator` | Removes enabled OpenCHAMI, OpenLDAP, Slurm, Kubernetes, storage-mount, and generated Orchestrator resources. Credentials are removed by default. |
| `discovery` | Empties the current project's Discovery output directory while preserving the directory, and removes Discovery credentials by default. |
| `image_build_manager` | Removes MinIO, the registry, build output, domain data, logs, and Image Build Manager credentials. |
| `repo_manager` | Removes the Pulp deployment, Pulp data, CLI configuration, repository integration, and logs. Credential removal is selected interactively unless explicitly configured. |
| `utils` | The general cleanup removes cluster-log collection directories and temporary unattended-OS-installation artifacts. OIM log backups require the separate `cleanup_backup_oim_logs` tag. Credential removal is selected interactively when applicable. |

## Steps

### 1. Clean up deployed domains

Run cleanup in reverse dependency order. Run only the domains that were
initialized or deployed in the environment.

```bash title="Run on: OIM"
cd <OMNIA_SOURCE_PATH>/src/main

./omnia.sh --run build_stream --tags cleanup
./omnia.sh --run telemetry --tags cleanup
./omnia.sh --run orchestrator --tags cleanup
./omnia.sh --run discovery --tags cleanup
./omnia.sh --run image_build_manager --tags cleanup
./omnia.sh --run repo_manager --tags cleanup
./omnia.sh --run utils --tags cleanup
```

See [Clean Up Utils](../HowTo/utils/cleanup_utils.md) before running the Utils
command; its log cleanup removes every collection run directory after checking
archive age. The command does not remove OIM log backups. Preserve required
backups, then run the dedicated cleanup described in that guide when needed.

Stop and resolve any failure before continuing to the next domain. Do not run
the main cleanup while domain playbooks still need the shared virtual
environment.

!!! warning

    The current BuildStreaM entry point contains an unresolved static import
    for its upgrade placeholder. Until that source issue is corrected, the
    top-level BuildStreaM playbook can fail during parsing before the
    `cleanup` tag runs.

!!! warning

    Discovery cleanup removes every artifact from the current project's output
    directory and removes the Discovery credential file and Vault key by
    default. Copy any mapping required by Orchestrator before cleanup. See
    [Clean up Discovery data](../HowTo/discovery/index.md#clean-up-discovery-data)
    for credential-preservation and credentials-only commands.

### 2. Select optional destructive behavior

Telemetry preserves persistent volumes by default. Delete them only when a
complete telemetry data reset is intended:

```bash title="Run on: OIM"
./omnia.sh --run telemetry --tags cleanup -e Delete_volume=true
```

To remove only iDRAC Telemetry resources, use `--tags cleanup_idrac`. Its MySQL
PVC `mysqldb-pvc-idrac-telemetry-0` is also preserved unless
`Delete_volume=true` is supplied.

Orchestrator removes its encrypted credentials and Vault key during full
cleanup by default. To preserve them, run:

```bash title="Run on: OIM"
./omnia.sh --run orchestrator --tags cleanup -e cleanup_credentials=false
```

Slurm and Kubernetes shared-data deletion is selected independently during
full cleanup. Review
[Clean Up Orchestrator](../HowTo/orchestrator/cleanup_orchestrator.md) before
running the command.

Repository Manager and Utils can prompt before removing credentials. Review
each prompt carefully and select the option that matches the redeployment
plan.

### 3. Remove the shared Omnia environment

After all required domain cleanups succeed, remove the virtual environment,
installed environment files, CLI, activation script, and dependency cache:

```bash title="Run on: OIM"
./omnia.sh --cleanup
```

This command preserves domain input, output, and log data under
`$OMNIA_DATA_PATH`.

For an explicitly approved full reset, remove the shared environment and the
complete Omnia data root:

```bash title="Run on: OIM"
./omnia.sh --cleanup --all
```

Type exactly `yes` when prompted. This operation removes all data under
`$OMNIA_DATA_PATH` and cannot be undone.

## Verification

- Confirm that every selected domain cleanup has a successful Ansible recap.
- Verify that the intended services and containers are absent.
- Confirm whether credentials and persistent data were preserved or removed
  as selected.
- After main cleanup, confirm that the shared virtual environment and installed
  Omnia environment files are absent.

## Next Steps

- Run `./omnia.sh --setup-venv` to recreate the shared environment.
- Initialize and deploy only the required domains in dependency order.
- See [Clean Up Orchestrator](../HowTo/orchestrator/cleanup_orchestrator.md) for
  component-level Orchestrator cleanup.
- See [Pulp Cleanup](pulp_cleanup.md) for selective repository cleanup.
