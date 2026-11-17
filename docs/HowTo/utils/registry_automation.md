# Ingest OCI Boot Images from a Registry

## Overview

Use the Utils `oci_image_ingest` workflow to pull OCI boot images from a Harbor
registry, convert them to squashfs boot artifacts, and publish them to S3 so
that OpenCHAMI can boot nodes from them. The workflow runs from the Omnia
Infrastructure Manager (OIM) and is an alternative to building images with
Image Build Manager.

!!! warning "Draft - pending verification"

    This page was drafted from Story ER-UTIL-001-documentation. The
    `oci_image_ingest` implementation is not part of the reviewed source
    revision. Every item marked **[VERIFY]** must be confirmed against the
    implemented utility before this page is published.

## Prerequisites

- Complete the [OIM setup](../main/setup_oim.md).
- Initialize Utils with `./omnia.sh -i utils`.
- A reachable Harbor registry that holds the OCI boot images. **[VERIFY]**
- The `regctl` and `squashfs-tools` packages and the `s3cmd` client installed
  on the OIM. **[VERIFY]**
- S3 storage that is reachable from the OIM and from the OpenCHAMI boot
  services. **[VERIFY]**

## Configure the ingestion

The configuration file is:

```text
$OMNIA_DATA_PATH/utils/input/$OMNIA_PROJECT_NAME/oci_image_ingest_config.yml
```

Use this file to define the Harbor connection and the images to ingest.
**[VERIFY]** The parameter names, types, defaults, and an annotated example
must be taken from the implemented input schema; they are intentionally not
listed here until the schema is available.

## Set up credentials

Registry and S3 credentials are stored in:

```text
$OMNIA_DATA_PATH/utils/input/$OMNIA_PROJECT_NAME/oci_image_ingest_credentials.yml
```

Encrypt this file with Ansible Vault. Do not store credentials in clear text
and do not pass them on the command line. **[VERIFY]** the credential keys and
the encryption command against the implemented utility.

## Run the ingestion

=== "Using omnia.sh (recommended)"

    ```bash title="Run on: OIM"
    cd <OMNIA_SOURCE_PATH>/src/main
    ./omnia.sh --run utils --tags oci_image_ingest
    ```

=== "Using ansible-playbook"

    ```bash title="Run on: OIM"
    source /opt/omnia/activate-omnia.sh
    cd <OMNIA_SOURCE_PATH>/src/utils/playbooks
    export ANSIBLE_CONFIG=../ansible.cfg
    ansible-playbook utils.yml --tags oci_image_ingest
    ```

When the credentials file is Vault-encrypted, supply the Vault password as
required by your Omnia setup. **[VERIFY]**

## Verify the result

After a successful run, confirm that the boot artifacts for each ingested image
are present in the configured S3 bucket. **[VERIFY]** the artifact names and
layout.

## Use the ingested images

The workflow creates a `build_status.yml` file in the Utils output folder:

```text
$OMNIA_DATA_PATH/utils/output/$OMNIA_PROJECT_NAME/build_status.yml
```

**[VERIFY]** the exact file path. The Orchestrator reads `build_status.yml`
from the directory set by `image_build_manager_output_path` in
`orchestrator_config.yml`. Make the ingested images available to the
Orchestrator in one of the following ways:

- Copy `build_status.yml` to the directory that
  `image_build_manager_output_path` points to.
- Set `image_build_manager_output_path` in `orchestrator_config.yml` to the
  Utils output folder that contains the file.

!!! note

    Copying the file overwrites a `build_status.yml` that Image Build Manager
    created in the destination directory. Back up the existing file first.

## Limitations

- A functional group is served either by Image Build Manager or by OCI
  ingestion, not by both.
- An Image Build Manager cleanup removes ingested releases. To recover, run
  the `oci_image_ingest` workflow again.
- Full coexistence of Image Build Manager and OCI ingestion for the same
  functional group is tracked as GH-5788.

## Troubleshooting

!!! note

    **[VERIFY]** Replace the entries below with the actual error messages
    produced by the implemented utility.

- **The registry cannot be reached**: Confirm DNS or IP connectivity, the
  Harbor address and port, and certificate trust from the OIM.
- **Authentication fails**: Confirm the values in
  `oci_image_ingest_credentials.yml` and that the file decrypts with the Vault
  password you supplied.
- **A required tool is missing**: Install `regctl`, `squashfs-tools`, and
  `s3cmd` on the OIM and rerun the workflow.
- **The S3 upload fails**: Confirm S3 reachability, bucket permissions, and
  free space at the S3 destination.
- **Nodes do not boot the ingested image**: Confirm that `build_status.yml` is
  in the directory set by `image_build_manager_output_path` and that the image
  belongs to a functional group that is not also served by Image Build
  Manager.
- **Ingested images disappear after a cleanup**: An Image Build Manager cleanup
  removed them. Rerun `oci_image_ingest`.

## Related documentation

- [Utilities](index.md)
- [Utils Domain Contract](../../Reference/domain_contracts/utils_contract.md)
- [Image Build Manager Contract](../../Reference/domain_contracts/image_build_manager_contract.md)
