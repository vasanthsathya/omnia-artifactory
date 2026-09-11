# Networking Requirements

This section outlines the key networking requirements for the components used by Omnia to deploy HPC clusters. For more information about the supported devices and software, see [Support Matrix](../index.md#support-matrix).

## Networking

- Ensure admin and BMC switches are configured and reachable.

### Discovery connectivity

- Allow the OIM to reach Dell OpenManage Enterprise (OME) over HTTPS on TCP
  port 443.
- Configure every target BMC/iDRAC interface with network connectivity and
  ensure that OME can reach, discover, and manage it before running Discovery.
- Discovery obtains BMC/iDRAC management and NIC inventory through OME; it does
  not connect directly from the OIM to the target BMC/iDRAC interfaces.

See [Network connectivity
requirements](../../HowTo/discovery/discover_nodes.md#network-connectivity-requirements)
for the required OME permissions and connectivity boundaries.

## InfiniBand

- Before deploying Omnia on clusters using InfiniBand (IB) networking, ensure that the Subnet Manager (SM) service is enabled and running on the InfiniBand switch or host.

!!! note

    Failure to meet this prerequisite may result in InfiniBand ports on hosts remaining in the Initializing state and prevent IB communication between nodes.

!!! info

    - [Networking Config](../Configuration/network_spec.md) -- Networking configuration.
    - [Configure Cluster DNS](../../HowTo/orchestrator/configure_cluster_dns.md) -- Cluster DNS configuration.
    - [Network Topology](../../Overview/network_topologies.md) -- Network topology configuration.

















