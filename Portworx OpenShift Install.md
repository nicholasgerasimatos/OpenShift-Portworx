# OpenShift - Pure Storage Validated Design Implementation Guide



**Purpose of this Document:**
This comprehensive guide provides detailed instructions and best practices for planning, implementing, operating, and maintaining a Cisco Validated Design (CVD) for FlashStack with Red Hat OpenShift Container Platform on bare metal. It covers Cisco UCS infrastructure, Pure Storage integration, OpenShift installation, configuration, and ongoing management.

**Target Audience:**
This document is intended for experienced infrastructure engineers, OpenShift administrators, solution architects, and technical personnel responsible for deploying and managing FlashStack solutions with OpenShift Container Platform. A foundational understanding of Cisco UCS, Pure Storage, Red Hat OpenShift, networking, and virtualization concepts is assumed.

**Disclaimer:**
This guide provides recommendations and procedures based on a validated design. However, every environment is unique. Users should always refer to the latest official documentation from Cisco, Pure Storage, and Red Hat. Thoroughly test all configurations in a non-production environment before implementing in production.

## Table of Contents

- [Introduction](#-introduction)
- [Solution Overview](#-solution-overview)
- [Planning and Prerequisites](#-planning-and-prerequisites)
- [Physical Infrastructure Implementation](#-physical-infrastructure-implementation)
- [OpenShift Installation and Configuration](#-openshift-installation-and-configuration)
- [Storage Integration and Configuration (Pure Storage CSI)](#-storage-integration-and-configuration-pure-storage-csi)
- [OpenShift Virtualization Implementation](#-openshift-virtualization-implementation)
- [Security Hardening](#-security-hardening)
- [Day 2 Operations](#-day-2-operations)
- [Performance Tuning](#-performance-tuning)
- [Backup and Disaster Recovery](#-backup-and-disaster-recovery)
- [Upgrades and Lifecycle Management](#-upgrades-and-lifecycle-management)
- [Automation and CI/CD Integration](#-automation-and-cicd-integration)
- [Troubleshooting](#-troubleshooting-expanded)
- [Advanced Use Cases](#-advanced-use-cases)
- [Reference Architecture](#-reference-architecture)
- [Conclusion and Next Steps](#-conclusion-and-next-steps)
- [Glossary](#-glossary)
- [Appendix (Optional - Placeholder)](#-appendix-optional---placeholder)

## Introduction
Red Hat OpenShift Container Platform provides a robust, enterprise-grade platform for developing, deploying, and managing containerized applications. When deployed on FlashStack, a converged infrastructure solution from Cisco and Pure Storage, organizations benefit from a pre-validated, high-performance, and scalable foundation. This guide details the steps to successfully implement such a solution.

## Solution Overview

### FlashStack Architecture

FlashStack is a defined set of hardware and software that serves as an integrated foundation for both virtualized and non-virtualized solutions. This CVD implements Red Hat OpenShift Container Platform on FlashStack using:

-   **Compute**: Cisco UCS B-Series or C-Series Servers managed by Cisco Intersight
-   **Network**: Cisco Nexus switching for Ethernet and Cisco MDS switches for FC connectivity
-   **Storage**: Pure Storage FlashArray//X or FlashArray//C
-   **Platform**: Red Hat OpenShift Container Platform 4.10 or newer
-   **Management**: Cisco Intersight with optional Terraform Cloud Integration

`[DIAGRAM: FlashStack for OpenShift - High-Level Architecture. Showcasing UCS, Nexus, MDS, Pure Storage, and OpenShift Layers]`
*(Placeholder: Replace with actual architecture diagram)*

### Solution Topology

The solution integrates multiple components in a validated architecture:
`[DIAGRAM: Detailed Solution Topology - Replace ASCII art with a proper diagram showing user access, load balancers, OCP control/worker/infra nodes, and connections to UCS, Network, Storage, and Management Stack components.]`
*(Original ASCII art was good for concept, but needs graphical replacement)*

### Key Benefits

-   **Simplicity**: Pre-validated architecture reduces deployment time and risk.
-   **Scalability**: Grow compute, network, and storage independently as needed.
-   **Performance**: All-flash storage with high-speed networking delivers consistent sub-millisecond latency.
-   **Automation**: End-to-end automation opportunities with Terraform and Ansible.
-   **Container and VM Support**: Run both containerized and virtualized workloads with OpenShift Virtualization.
-   **Security**: Integrated security at all layers with validated configurations.
-   **Resiliency**: Designed for no single point of failure with redundant components.
-   **Operational Efficiency**: Unified management plane opportunities with comprehensive monitoring.

### Hardware Requirements

*Always consult the official Red Hat OpenShift, Cisco UCS, and Pure Storage Hardware Compatibility Lists (HCLs) for the latest supported configurations.*

| Component                     | Minimum Specification                               | Recommended Specification                             | Enterprise Scale                                           |
| ----------------------------- | --------------------------------------------------- | ----------------------------------------------------- | ---------------------------------------------------------- |
| Compute (Control Plane Nodes) | 3× Cisco UCS B200 M6 or C220/C240 M6 (8c/64GB RAM)  | 3× Cisco UCS B200 M6 or C220/C240 M6 (16c/128GB RAM) | 3-5× Cisco UCS B200 M6 or C220/C240 M6 (32c+/256GB+ RAM)   |
| Compute (Worker Nodes)        | 3× Cisco UCS B200 M6 or C220/C240 M6 (8c/32GB RAM)   | 6× Cisco UCS B200 M6 or C220/C240 M6 (32c/128GB RAM)  | 9+× Cisco UCS B200 M6 or C220/C240 M6 (64c+/512GB+ RAM) |
| Network (Ethernet)            | 2× Cisco UCS 6454 FIs, 2× Cisco Nexus 9336C-FX2     | 2× Cisco UCS 6454 FIs, 2× Cisco Nexus 9336C-FX2       | 2× Cisco UCS 6454 FIs, 4× Cisco Nexus 9336C-FX2 (or better) |
| Network (FC, if used)         | 2× Cisco MDS 9132T or 9148T Switches                | 2× Cisco MDS 9148T or higher                          | 2× Cisco MDS 9396T or Director-class                       |
| Storage                       | 1× Pure Storage FlashArray//X70                     | 1× Pure Storage FlashArray//X90                       | 2× Pure Storage FlashArray//X90 (ActiveCluster or higher)  |
| Cabling                       | 25/100Gbps for data, 32Gbps for FC (if used)        | 25/100Gbps for data, 32Gbps for FC (if used)        | 100Gbps+ for data, 32/64Gbps for FC (if used)            |
| Memory (per Control Node)     | 64GB                                                | 128GB                                                 | 256GB+                                                     |
| Memory (per Worker Node)      | 32GB (workload dependent)                           | 128GB (workload dependent)                            | 512GB+ (workload dependent)                                |
| CPU (per Node)                | 2× Intel Xeon Silver/Gold (min 8 cores per socket)  | 2× Intel Xeon Gold 63xx / Platinum 83xx per node    | 2× Intel Xeon Platinum 83xx per node (or higher core count)|
| Disk (per Control Node etcd)  | 150GB NVMe/SSD (low latency critical)               | 250GB NVMe/SSD (low latency critical)                 | 500GB+ NVMe (low latency critical)                         |

### Software Requirements

| Component                       | Minimum Version         | Recommended Version     | Notes                                                                     |
| ------------------------------- | ----------------------- | ----------------------- | ------------------------------------------------------------------------- |
| Cisco Intersight                | Base                    | Infrastructure Service  | Advantage tier recommended for full automation and advanced features.     |
| UCS Manager / FI Firmware       | 4.2(1f)                 | Latest stable 4.2(x)+   | Consult Cisco HCL for OCP. Establish a firmware baseline.                 |
| Pure Storage Purity OS          | 6.1.x                   | Latest stable 6.3.x+    | Required for latest storage features and CSI driver compatibility.        |
| Pure Storage Plugin for vSphere | 4.x (if vSphere used)   | 5.0.0+ (if vSphere used)| For vSphere integration if managing VMs outside OCP Virtualization.       |
| OpenShift Container Platform    | 4.10                    | 4.13 or later           | For newest platform capabilities and support lifecycle.                   |
| OpenShift Virtualization        | 4.10                    | 4.13 or later           | Aligns with OCP version. For VM workloads.                              |
| OpenShift Data Foundation (ODF) | 4.10 (optional)         | 4.13+ (optional)        | Optional for additional internal storage services (e.g., for registry).   |
| Red Hat Enterprise Linux CoreOS | Tied to OpenShift ver.  | Tied to OpenShift ver.  | Base OS for OpenShift nodes, version managed by OCP.                      |
| Ansible                         | 2.9+ (optional)         | Latest stable (optional)| Recommended for configuration management and automation tasks.            |

## Planning and Prerequisites

Before implementation, consider these key design decisions and ensure all prerequisites are met.

### Design Considerations

#### Networking Architecture
`[DIAGRAM: Network VLAN Layout - Showing VLANs for Management, PXE, OCP Node Communication, OCP Machine Network, Storage (iSCSI/NFS), Application Ingress/Egress, and OOB Management across Nexus, FIs, and Host NICs.]`

| Network Type          | Purpose                                          | VLAN Considerations                      | Bandwidth Requirements          | MTU   | Notes                                                      |
| --------------------- | ------------------------------------------------ | ---------------------------------------- | ------------------------------- | ----- | ---------------------------------------------------------- |
| OOB Management        | UCSM, FI Mgmt, Server CIMC/IMM, Switch Mgmt      | Isolated, Secure                         | 1 Gbps                          | 1500  | Out-of-band access                                         |
| UCS Inband Management | Intersight connectivity, KVM, vMedia             | Routable if Intersight SaaS is used      | 1-10 Gbps                       | 1500  |                                                            |
| PXE/Provisioning      | OpenShift node OS deployment (IPI/Assisted)      | DHCP Required                            | 1-10 Gbps                       | 1500  | Needed if using IPI/Assisted Installer with DHCP.          |
| OCP Machine Network   | OpenShift Node IP addresses                      | Routable to DNS/NTP, Pull Secret access  | 10-25+ Gbps                     | 1500  | This is the primary network for the nodes themselves.      |
| OCP Cluster Network   | Pod-to-Pod communication (Overlay)               | Internal to OCP (e.g., 10.128.0.0/14)    | Handled by node NICs            | 1500+ | MTU can be higher if underlying network supports. OVNK=1500. |
| OCP Service Network   | ClusterIP services (Overlay)                     | Internal to OCP (e.g., 172.30.0.0/16)    | Handled by node NICs            | N/A   |                                                            |
| Storage (iSCSI/NFS)   | Access to Pure Storage (if not FC)               | Dedicated, Non-Routable (best practice)  | 25+ Gbps (dedicated per path)   | 9000  | Jumbo Frames highly recommended for performance.           |
| Application Ingress   | External access to applications via Ingress/LB   | Routable, potentially multiple segments  | 25-100+ Gbps                    | 1500  | Exposed to users/other services.                           |
| Application Egress    | Pod outbound traffic                             | Routable                                 | As needed by applications       | 1500  |                                                            |
| OCP API Network       | Access to OpenShift API (internal & external)    | Routable                                 | 10-25+ Gbps                     | 1500  | Usually shares Machine Network or App Ingress.             |

#### Storage Design Decisions

1.  **Protocol Selection**
    *   **Fibre Channel (FC)**: Highest performance, physical isolation, traditional SAN approach. Recommended for latency-sensitive workloads.
    *   **iSCSI**: Good performance, uses existing Ethernet infrastructure. Requires careful network design (dedicated VLANs, Jumbo Frames, QoS).
    *   **NFS**: Simpler for file sharing, suitable for some OpenShift workloads (e.g., ReadWriteMany PVCs if Pure File Services or an external NFS server is used with Pure). The Pure CSI driver can provision NFS exports if FlashArray File Services are enabled.

    > **Recommendation**: Use FC for most critical workloads requiring lowest latency. iSCSI is a strong alternative. Use NFS for RWX requirements where appropriate.

2.  **Volume Layout (on Pure Storage FlashArray)**
    *   The Pure CSI driver will dynamically provision volumes. However, consider:
        *   **Protection Groups:** Group volumes for consistent snapshot/replication policies (e.g., OCP platform volumes, application-specific groups).
        *   **QoS Policies:** Define QoS policies on the FlashArray (e.g., `gold`, `silver`, `bronze`) and map them to OpenShift StorageClasses.
    *   For OpenShift platform components that might need persistent storage *before* CSI is fully active (less common with modern IPI), manual provisioning might be needed but aim to use CSI.

3.  **Performance Tiers / StorageClasses**
    *   Define multiple OpenShift StorageClasses mapping to different Pure Storage capabilities (e.g., backend protocol, QoS, replication).

### Implementation Checklist

#### Physical Infrastructure Readiness
-   [ ] Rack and power equipment according to data center specifications.
-   [ ] Configure physical networking (cables, ToR switches, FC switches if applicable).
-   [ ] Verify connectivity between all physical components (ping, link lights).
-   [ ] Validate power redundancy and cooling requirements.
-   [ ] Establish firmware baselines for all hardware components (FIs, servers, NICs, HBAs, switches).

#### Network Prerequisites
-   [ ] **DNS Configuration:**
    -   [ ] Forward and reverse DNS records for all planned OpenShift nodes (control plane, workers).
    -   [ ] Wildcard DNS record for OpenShift applications (e.g., `*.apps.clustername.domain.com`).
    -   [ ] DNS record for OpenShift API (e.g., `api.clustername.domain.com`).
    -   [ ] DNS record for OpenShift API Internal (e.g., `api-int.clustername.domain.com`).
    -   [ ] If using load balancers, DNS records for VIPs.
-   [ ] **NTP Configuration:**
    -   [ ] Reliable NTP sources identified and accessible by all nodes.
-   [ ] **DHCP/PXE (if using IPI/Assisted Installer with DHCP):**
    -   [ ] DHCP server configured with scopes for the OCP Machine Network.
    -   [ ] Static DHCP reservations for nodes are highly recommended.
    -   [ ] PXE boot services (TFTP, HTTP) configured if not using iPXE from ISO.
-   [ ] **Firewall Configuration:**
    -   [ ] Review [OpenShift networking requirements documentation](https://docs.openshift.com/container-platform/latest/installing/installing_platform_agnostic/installing-platform-agnostic.html#installation-network-user-infra_installing-platform-agnostic) for required ports.
    -   [ ] Ensure necessary ports are open for internal cluster communication, external access, and access to external services (registry, IdP, NTP, DNS).
-   [ ] **Load Balancers (if used for API/Ingress):**
    -   [ ] External load balancer(s) provisioned and configured for API (TCP/6443, TCP/22623) and Ingress (TCP/80, TCP/443).

#### Pure Storage Preparation
-   [ ] Latest recommended Purity OS version installed.
-   [ ] Network ports configured for the chosen protocol (FC zoning done, or iSCSI/NFS IP interfaces configured).
-   [ ] Management access configured (IP, DNS, NTP).
-   [ ] Pure1 cloud-assisted support and monitoring connected and configured.
-   [ ] API user created on FlashArray with appropriate permissions for CSI driver integration.
-   [ ] (Optional) Pre-create host groups or define host personalities if needed, though CSI often manages this.
-   [ ] (Optional) Define QoS policies on the array.

#### Operational Prerequisites
-   [ ] Red Hat subscription activated and entitlements available.
-   [ ] **Pull Secret:** Downloaded from Red Hat Cloud Console (`pull-secret.txt`).
-   [ ] SSH keys (public/private pair) generated for node access during installation/debugging.
-   [ ] (Optional) Bastion host/Installation host provisioned with network access to planned OCP networks and internet access for downloads.
-   [ ] Licensing for all components verified (Red Hat, Cisco Intersight tiers, etc.).

### Network Flow Planning
`[DIAGRAM: Detailed Network Flow Diagram - Illustrating traffic paths for: User to Apps, User to API/Console, Pod-to-Pod, Pod-to-Service, Node-to-Storage, Node-to-External Services, API to Nodes, etc. Replace ASCII with proper diagram.]`

### System Sizing Guidelines (OpenShift Nodes)

#### Control Plane Nodes (Minimum 3)
*   **CPU:** 8+ cores (16+ recommended)
*   **Memory:** 64GB+ (128GB+ recommended)
*   **Disk (OS):** 120GB SSD/NVMe
*   **Disk (etcd):** 150GB+ **low-latency NVMe/SSD** (dedicated if possible, 250GB+ recommended)
*   **Network:** Redundant 10Gbps+ (25Gbps+ recommended)

#### Worker Nodes (Minimum 3, scales with workload)
*   **CPU:** 8+ cores (32-64+ recommended, workload dependent)
*   **Memory:** 32GB+ (128-512GB+ recommended, workload dependent)
*   **Disk (OS):** 120GB SSD/NVMe
*   **Network:** Redundant 10Gbps+ (25Gbps+ recommended)

> **Note**: For production environments, consider dedicated infrastructure nodes for OpenShift services like the router, registry, monitoring, and logging to isolate them from application workloads. Sizing for these would be similar to worker nodes but tailored to the service.

## Physical Infrastructure Implementation

This section details the setup of Cisco UCS, networking, and Pure Storage components.

### Cisco UCS and Network Deployment
`[DIAGRAM: UCS Service Profile Policy Structure - Illustrating how policies like BIOS, Boot, vNIC/vHBA, QoS, etc., combine into Service Profile Templates.]`

#### 1. Cisco Intersight Setup
1.  Create or use an existing Cisco Intersight account.
2.  Claim your Fabric Interconnects, and optionally C-Series servers (if managed by Intersight).
3.  Deploy and configure Intersight Assist virtual appliances if using the SaaS model and managing devices that require it (e.g., UCS Manager, Pure Storage for some integrations).
4.  Ensure device connectors on FIs and servers can reach Intersight.

#### 2. Network Configuration (Nexus Switches, MDS Switches)
1.  **Nexus Switches (Ethernet):**
    *   Configure VLANs as per "Networking Architecture" table (Management, PXE, OCP Machine, Storage, Application, etc.).
    *   Configure trunk ports (allowing necessary VLANs) towards Cisco UCS Fabric Interconnects.
    *   Configure PortChannels (vPC recommended) for uplinks to FIs for redundancy and bandwidth.
    *   If using iSCSI/NFS, configure MTU 9000 on storage VLANs/interfaces end-to-end.
    *   Implement appropriate STP (Spanning Tree Protocol) settings.
2.  **MDS Switches (Fibre Channel - if used):**
    *   Configure VSANs for storage traffic.
    *   Connect MDS uplinks to Pure Storage FC ports and Cisco UCS FI FC uplink ports.
    *   Configure zoning: Create zones containing UCS vHBA WWPNs and Pure Storage Array Target WWPNs. Use single initiator-to-multiple-targets zoning.
    *   Ensure NPIV is enabled on MDS ports connecting to FIs.

#### 3. Configure Fabric Interconnects (via UCS Manager or Intersight)
1.  **Initial Setup:** Configure FI cluster IP, management IPs, NTP, DNS.
2.  **Port Configuration:**
    *   Configure Server Ports (connecting to UCS chassis IOMs or C-Series server NICs).
    *   Configure Uplink Ports (Ethernet to Nexus, FC to MDS).
    *   Create PortChannels for Ethernet and FC uplinks.
3.  **VLANs/VSANs:**
    *   Define VLANs on FIs matching those on Nexus switches. Assign VLANs to vNICs/vNIC templates.
    *   Define VSANs on FIs matching those on MDS switches. Assign VSANs to vHBAs/vHBA templates.
4.  **Enable NPIV** on FIs if using Fibre Channel.

#### 4. Create UCS Policies (Using Intersight or UCS Manager)
Create the following policies as building blocks for Service Profiles. Examples are illustrative.

1.  **BIOS Policy (`ocp-bios-policy`):**
    *   **Processor:** Consistent with "System Sizing Guidelines".
    *   **CPU Performance:** `Enterprise` or `High Throughput`.
    *   **CPU Power Management:** `Performance`.
    *   **Processor C State, C1E, C6 Report:** `Disabled`.
    *   **NUMA Group Size Optimization:** `Clustered` or `Flat` (test for workload).
    *   **Virtualization (Intel VT-x/AMD-V, VT-d/IOMMU):** `Enabled`.
    *   **SR-IOV (if planned for OpenShift Virtualization NICs):** `Enabled`.
    *   **Hyper-Threading:** `Enabled`.
    *   **Secure Boot (Optional):** `Enabled` (requires OS/Bootloader support).
2.  **Boot Order Policy (`ocp-boot-policy`):**
    *   **Boot Mode:** `UEFI` (Required for OpenShift 4.x).
    *   **Boot Order:**
        1.  `PXE vNIC` (for IPI/Assisted deployment; specify the vNIC used for provisioning).
        2.  `Local Disk` (or `SAN LUN` if SAN booting OS).
        3.  (Optional) `Virtual Media` for recovery.
    *   **Secure Boot:** `Disabled` or `Enabled` (ensure consistency with BIOS policy and OS).
3.  **Adapter Policy (Ethernet - `ocp-eth-adapter`):**
    *   **Interrupt Mode:** `MSI-X`.
    *   **Receive Side Scaling (RSS):** `Enabled`.
    *   **Transmit/Receive Queues/Ring Sizes:** Adjust based on Cisco recommendations for high-speed NICs (e.g., for 25/100Gbps VICs).
    *   **Jumbo Frames (MTU 9000):** Enable if storage or specific application networks require it.
4.  **Adapter Policy (FC - `ocp-fc-adapter` - if used):**
    *   Default settings are often sufficient. Adjust Transmit/Receive Credits if specific issues arise.
5.  **Network Control Policy (`ocp-net-control`):**
    *   **CDP (Cisco Discovery Protocol):** `Enabled`.
    *   **LLDP (Link Layer Discovery Protocol):** `Enabled` (Transmit and Receive).
6.  **Power Control Policy (`ocp-power-control`):**
    *   **Power Capping:** `No Cap` or appropriate setting.
7.  **Local Disk Configuration Policy (`ocp-raid1-os`):**
    *   Configure RAID-1 for OS disks (typically two local M.2 or SSDs).
8.  **Scrub Policy (`ocp-scrub-policy`):**
    *   Disk Scrub: `Enabled`. BIOS Settings Scrub: `Enabled`.
9.  **Maintenance Policy (`ocp-maint-policy`):**
    *   **Reboot Policy:** `User Ack` (prevents unexpected reboots).

#### 5. Create UCS vNIC/vHBA Templates and Pools
1.  **MAC Address Pools:** Create pools for vNICs (e.g., `ocp-mac-pool-a`, `ocp-mac-pool-b`).
2.  **WWPN Pools (if FC):** Create pools for vHBAs (e.g., `ocp-wwpn-pool-a`, `ocp-wwpn-pool-b`). Ensure WWNN pool is also created.
3.  **IP Pools (for KVM/Management IPs on blades/racks):** e.g., `ocp-mgmt-ip-pool`.
4.  **UUID Suffix Pool:** e.g., `ocp-uuid-pool`.
5.  **vNIC Templates:**
    *   **`ocp-fabric-a-vnic`**: Fabric A, uses MAC Pool A, assign OCP Machine Network VLAN (native or tagged), other necessary VLANs (tagged), MTU (1500 or 9000), Adapter Policy, QoS Policy.
    *   **`ocp-fabric-b-vnic`**: Fabric B, uses MAC Pool B, same VLANs/MTU as Fabric A vNIC.
    *   (Optional) **`ocp-iscsi-a-vnic` / `ocp-iscsi-b-vnic`**: If using iSCSI, dedicated vNICs on separate fabrics for storage VLAN, MTU 9000.
6.  **vHBA Templates (if FC):**
    *   **`ocp-fabric-a-vhba`**: Fabric A, uses WWPN Pool A, assign appropriate VSAN.
    *   **`ocp-fabric-b-vhba`**: Fabric B, uses WWPN Pool B, assign appropriate VSAN.

#### 6. Create UCS Service Profile Templates
Create templates for different node roles (Control Plane, Worker).
1.  **`ocp-control-plane-spt`**:
    *   Assign policies: BIOS, Boot, Network Control, Power, RAID, etc.
    *   Add vNICs from templates (e.g., `ocp-fabric-a-vnic`, `ocp-fabric-b-vnic`).
    *   Add vHBAs from templates (if FC).
    *   Set appropriate CPU/Memory reservations if UCSM allows (less common, OpenShift manages this).
2.  **`ocp-worker-spt`**:
    *   Similar to control plane, but resource needs may differ.
    *   If using OpenShift Virtualization with SR-IOV, ensure vNICs intended for SR-IOV are configured appropriately in the template.

#### 7. Create and Associate Service Profiles
1.  Derive Service Profiles from the templates for each physical server.
2.  Assign unique identities (UUID, MACs, WWPNs) from pools.
3.  Associate Service Profiles with physical servers.
4.  **Validation:**
    *   Verify server boots and profiles apply correctly.
    *   Check vNIC/vHBA status, MAC/WWPN assignment.
    *   Verify connectivity on configured VLANs/VSANs from the server (e.g., using KVM and OS tools if an OS is temporarily booted).

### Pure Storage Configuration

#### 1. FlashArray Initial Setup
1.  Configure management interfaces (IP, netmask, gateway).
2.  Set up DNS, NTP, and SMTP/Syslog alerting.
3.  Connect to Pure1 for cloud-based analytics, support, and monitoring.
4.  Upgrade Purity OS to the latest recommended version compatible with your OpenShift version and CSI driver.
5.  Enable FlashArray File Services if NFS provisioning via CSI is planned.

#### 2. Configure Host Connectivity
1.  **Create an API User for CSI:**
    *   On the FlashArray, create a dedicated user (e.g., `openshift-csi`) with `array_admin` role (or more restricted if your Purity version supports finer-grained API roles for CSI).
    *   Generate an API token for this user. This token will be used in the Kubernetes secret for the CSI driver.
2.  **For iSCSI Storage:**
    *   Configure iSCSI network interfaces on the FlashArray (IPs, VLANs, MTU 9000).
    *   On OpenShift nodes, the iSCSI initiator software will be configured by RHCOS. The CSI driver will handle discovery and login.
    *   Ensure network path from OCP nodes to FlashArray iSCSI IPs is clear.
3.  **For Fibre Channel Storage:**
    *   Connect FlashArray FC ports to MDS switches.
    *   Ensure zoning is correctly configured on MDS switches (UCS vHBA WWPNs <-> FlashArray Target WWPNs).
    *   The CSI driver will identify hosts based on their WWPNs.
4.  **Host Groups (Optional but Recommended):**
    *   Create a host group on the FlashArray (e.g., `openshift-cluster`).
    *   The CSI driver will typically add hosts to this group if specified, or you can pre-add them if WWPNs/IQNs are known.

#### 3. Configure Volumes (Primarily Handled by CSI)
*   Most volumes will be dynamically provisioned by the Pure Storage CSI driver.
*   For volumes needed by OpenShift platform services *before* CSI might be fully operational (e.g., if not using dynamic provisioning for internal registry on some setups), you might pre-create them:
    *   `ocp-registry-vol` (if using dedicated PV for internal registry)
    *   `ocp-monitoring-vol` (if Prometheus needs dedicated PVs not from default SC)
    *   `ocp-logging-vol` (if logging stack needs dedicated PVs not from default SC)
    *   Connect these pre-created volumes to the `openshift-cluster` host group.
*   **Snapshot and Replication Policies (Protection Groups):**
    *   Define Protection Groups on the FlashArray for different data categories (e.g., `ocp-app-prod-pg`, `ocp-etcd-backups-pg`).
    *   Configure snapshot schedules (e.g., hourly, daily, weekly) with desired retention.
    *   If using asynchronous or synchronous replication to another FlashArray for DR, configure it within these Protection Groups.

## OpenShift Installation and Configuration

This section describes installing OpenShift using different methods. **Installer Provisioned Infrastructure (IPI) on Bare Metal is the primary focus for this CVD.**

### Deployment Options

1.  **Installer Provisioned Infrastructure (IPI) - Bare Metal:** Red Hat's `openshift-install` tool automates most of the infrastructure provisioning, including OS deployment via PXE or virtual media. **This is the recommended method for FlashStack bare metal.**
2.  **Assisted Installer:** A GUI-driven installation via Red Hat Cloud Console, helpful for smaller or simpler deployments. It still provisions the OS.
3.  **User Provisioned Infrastructure (UPI):** You provision and configure all infrastructure, including OS installation, networking, and load balancers, before running `openshift-install`. More complex but offers maximum control.

### OpenShift Deployment Using IPI (Bare Metal - Preferred)

This method uses `openshift-install` to provision RHCOS onto the bare metal UCS servers.

#### 1. Prepare Installation Host (Bastion)
*   Provision a RHEL 8/9 (or compatible Linux) system with network access to:
    *   The planned OpenShift Machine Network.
    *   Internet (for downloading OpenShift installer, images, etc.).
    *   DNS and NTP servers.
*   Install required tools:
    ```bash
    [bastion]$ sudo dnf install -y podman git openshift-clients jq httpd wget
    ```
*   Ensure the bastion host has at least 4 CPU, 16GB RAM, 100GB disk.

#### 2. Download OpenShift Installer and Client Tools
```bash
[bastion]$ LATEST_OCP_VERSION="4.13.latest" # Check latest stable version
[bastion]$ MIRROR_URL="https
[bastion]$ curl -k ${MIRROR_URL}/pub/openshift-v4/clients/ocp/${LATEST_OCP_VERSION}/openshift-install-linux.tar.gz -o openshift-install.tar.gz
[bastion]$ curl -k ${MIRROR_URL}/pub/openshift-v4/clients/ocp/${LATEST_OCP_VERSION}/openshift-client-linux.tar.gz -o openshift-client.tar.gz
[bastion]$ tar -xzf openshift-install.tar.gz
[bastion]$ tar -xzf openshift-client.tar.gz
[bastion]$ sudo mv oc kubectl openshift-install /usr/local/bin/
```

#### 3. Configure DHCP and PXE on Bastion (or dedicated server)
For IPI bare metal, the installer needs a way to boot nodes and assign IPs.
*   **DHCP Server (`dnsmasq` example on bastion):**
    *   Install `dnsmasq`: `sudo dnf install -y dnsmasq`
    *   Configure `/etc/dnsmasq.conf`:
        ```conf
        interface=ethX # Interface connected to OCP Machine Network
        dhcp-range=192.168.1.100,192.168.1.200,255.255.255.0,12h # Example range
        dhcp-option=option:router,192.168.1.1 # Gateway for OCP Machine Network
        dhcp-option=option:dns-server,192.168.10.5,8.8.8.8 # DNS servers
        enable-tftp
        tftp-root=/var/lib/tftpboot
        dhcp-boot=pxelinux.0 # For legacy PXE
        # For UEFI PXE, typically:
        # dhcp-match=set:efi-x86_64,option:client-arch,7 # x64 UEFI
        # dhcp-match=set:efi-x86_64,option:client-arch,9 # x64 UEFI
        # dhcp-boot=tag:efi-x86_64,shimx64.efi # Or your UEFI boot file

        # Static assignments (RECOMMENDED)
        # dhcp-host=AA:BB:CC:DD:EE:01,ocp-master-0,192.168.1.10,infinite
        # dhcp-host=AA:BB:CC:DD:EE:02,ocp-master-1,192.168.1.11,infinite
        # ... and so on for all nodes
        ```
    *   Start and enable `dnsmasq`: `sudo systemctl enable --now dnsmasq`
*   **HTTP Server (for Ignition files and RHCOS images):**
    *   `sudo systemctl enable --now httpd`
    *   Ensure firewall allows HTTP/TFTP/DHCP.

#### 4. Create Installation Directory and Configuration File
```bash
[bastion]$ mkdir ~/ocp-flashstack-install
[bastion]$ cd ~/ocp-flashstack-install
[bastion]$ cp <path_to_your_pull_secret.txt> .
[bastion]$ ssh-keygen -t rsa -b 4096 -N '' -f ./id_rsa # For node access
```
Create `install-config.yaml`:
```yaml
apiVersion: v1
baseDomain: "example.com" # Your base domain
compute:
- architecture: amd64
  hyperthreading: Enabled
  name: worker
  replicas: 3 # Minimum 3 worker nodes
  # platform: {} # For baremetal IPI, this is usually empty
  # If using specific hardware profiles from Metal3 (less common for simple IPI)
  # platform:
  #   baremetal:
  #     hwProfile: worker-profile
controlPlane:
  architecture: amd64
  hyperthreading: Enabled
  name: master
  replicas: 3
  # platform: {}
  # platform:
  #   baremetal:
  #     hwProfile: control-plane-profile
metadata:
  name: "flashstack-ocp" # Your cluster name
networking:
  clusterNetwork:
  - cidr: 10.128.0.0/14 # Pod network
    hostPrefix: 23
  machineNetwork: # Network where OCP nodes get their IPs
  - cidr: "192.168.1.0/24" # MUST match your DHCP/Static IP range
  networkType: "OVNKubernetes" # Recommended, or "OpenShiftSDN"
  serviceNetwork:
  - "172.30.0.0/16" # Service network
platform:
  baremetal: # This section is key for IPI baremetal
    apiVIP: "192.168.1.5" # Virtual IP for OpenShift API (must be in machineNetwork)
    ingressVIP: "192.168.1.6" # Virtual IP for Ingress (must be in machineNetwork)
    # If using external load balancer, these VIPs are managed externally
    # and you might use 'none' platform with UPI or more manual IPI steps.
    # For true IPI with integrated MetalLB/Keepalived, these are used.

    hosts: # Define your bare metal hosts
      - name: ocp-master-0
        role: master # or control-plane
        bmc:
          address: "idrac-virtualmedia+http://<bastion_ip>:8000/master-0.iso" # For virtual media boot
          # address: "ipmi://<cimc_ip_master0>" # For IPMI power management
          # username: "<cimc_user>"
          # password: "<cimc_password>"
        bootMACAddress: "AA:BB:CC:DD:EE:01"
        # networkConfig: # For static IP assignment within install-config (optional, DHCP is simpler)
        #   interfaces:
        #     - name: eno1
        #       type: ethernet
        #       state: up
        #       macAddress: "AA:BB:CC:DD:EE:01"
        #       ipv4:
        #         enabled: true
        #         address:
        #           - ip: 192.168.1.10
        #             prefix-length: 24
        #         dhcp: false
      - name: ocp-master-1
        # ... (similar for other masters and workers) ...
      - name: ocp-worker-0
        role: worker
        bmc:
          address: "idrac-virtualmedia+http://<bastion_ip>:8000/worker-0.iso"
        bootMACAddress: "AA:BB:CC:DD:FF:01"
        # ...
    # Provisioning Network (if different from machineNetwork and DHCP is on it)
    # provisioningNetwork: "Disabled" # or "Managed" or "Unmanaged"
    # provisioningOSDownloadURL: "http://<bastion_ip>/rhcos-live.x86_64.iso" # If hosting RHCOS images
    # bootstrapOSImage: "http://<bastion_ip>/rhcos-live-bootstrap.x86_64.iso" # If hosting RHCOS images

fips: false # Set to true if FIPS compliance is required
pullSecret: '<contents_of_your_pull_secret.txt>'
sshKey: '<contents_of_your_id_rsa.pub>'
# imageContentSources: # If using a disconnected registry
# proxy: # If behind a corporate proxy
# additionalTrustBundle: # If proxy or internal services use custom CAs
```
**Notes on `install-config.yaml` for IPI Baremetal:**
*   **`platform.baremetal.hosts`:** This is where you describe your UCS servers.
    *   `bmc.address`:
        *   For **virtual media boot (iPXE)**: `idrac-virtualmedia+http://<bastion_ip_or_http_server>/<node_name>.iso` (The installer will generate these ISOs if you don't specify `provisioningOSDownloadURL`). You'll need to ensure UCS servers can boot from virtual media provided by this HTTP URL.
        *   For **IPMI power management with PXE boot**: `ipmi://<cimc_ip>` and provide `username`/`password`. The installer then relies on your DHCP/PXE server to serve RHCOS.
    *   `bootMACAddress`: Crucial for PXE booting and DHCP reservations.
    *   `networkConfig`: For fully static IP assignment via Ignition (more complex than DHCP reservations).
*   `apiVIP` and `ingressVIP`: These will be managed by Keepalived on control plane nodes and MetalLB (or similar) if you're not using an external LB. They must be free IPs in your `machineNetwork`.

#### 5. Generate Manifests (and optionally customize)
```bash
[bastion]$ openshift-install create manifests --dir=./
```
You can now customize manifests in the `manifests/` and `openshift/` directories. Common customizations:
*   **`manifests/cluster-scheduler-02-config.yml`**: Set `mastersSchedulable: false` if you don't want workloads on control plane nodes (recommended for production).
*   **Adding NTP `MachineConfig`:** Create e.g., `openshift/99-master-chrony.yaml` and `openshift/99-worker-chrony.yaml` to configure custom NTP servers for nodes.
    ```yaml
    # Example: openshift/99-master-chrony.yaml
    apiVersion: machineconfiguration.openshift.io/v1
    kind: MachineConfig
    metadata:
      labels:
        machineconfiguration.openshift.io/role: master
      name: 99-master-chrony-configuration
    spec:
      config:
        ignition:
          version: 3.2.0
        storage:
          files:
            - contents:
                source: data:,server%20ntp1.example.com%20iburst%0Aserver%20ntp2.example.com%20iburst
              filesystem: root
              mode: 0644
              path: /etc/chrony.conf.d/custom-ntp.conf
    ```

#### 6. Start OpenShift Installation
This step will:
1.  Create Ignition configs.
2.  If using virtual media (`idrac-virtualmedia`), it will host ISOs for each node.
3.  Start a temporary bootstrap node.
4.  Control plane nodes will boot (via PXE or virtual media) and get configuration from the bootstrap node.
5.  Once control plane is up, bootstrap node is destroyed.
6.  Worker nodes boot and join the cluster.

```bash
[bastion]$ openshift-install create cluster --dir=./ --log-level=info # or debug
```
Monitor the installation. This can take 40-90 minutes.
Ensure UCS servers are set to boot from PXE or the virtual media provided by the installer.

#### 7. Post-Installation
*   The installer will output the `kubeadmin` password and `kubeconfig` file.
*   Export `KUBECONFIG`:
    ```bash
    [bastion]$ export KUBECONFIG=~/ocp-flashstack-install/auth/kubeconfig
    [bastion]$ oc whoami # Should be system:admin
    [bastion]$ oc get nodes
    ```
*   **Important:** Log in to the cluster as `kubeadmin` and configure an Identity Provider (LDAP, OAuth, etc.) and create your admin users. Do not use `kubeadmin` for daily operations.
    ```bash
    [bastion]$ oc create clusterrolebinding ocp-admins --clusterrole=cluster-admin --user=myadminuser
    ```

### OpenShift Deployment Using Assisted Installer
(This is an alternative to IPI, often simpler for fewer nodes or less automation)

1.  **Access Red Hat OpenShift Cluster Manager:** [console.redhat.com](https://console.redhat.com)
2.  **Create Cluster:** Navigate to OpenShift → Create Cluster → Datacenter → Assisted Installer.
3.  **Cluster Details:**
    *   Cluster name, Base domain, OpenShift version.
    *   CPU architecture: x86_64.
    *   Hosts: Select Bare Metal.
4.  **Networking:**
    *   Choose "User-Managed Networking" or "Assisted Networking."
    *   Provide Cluster Network CIDR, Service Network CIDR.
    *   Configure API VIP and Ingress VIP (these can be automatically assigned or you can provide static ones).
    *   Provide DNS servers.
5.  **Generate Discovery ISO:**
    *   Download the discovery ISO.
    *   You may need to provide SSH public key and proxy settings if applicable.
6.  **Boot Servers:**
    *   Configure Cisco UCS service profiles with virtual media mapped to the downloaded discovery ISO.
    *   Use Intersight or UCS Manager to mount the ISO and boot the servers.
7.  **Host Discovery & Configuration (in Assisted Installer UI):**
    *   Servers will appear in the UI as they boot the discovery ISO.
    *   Assign roles (control plane, worker) to each discovered host.
    *   Validate networking, NTP, and other prerequisites in the UI.
8.  **Start Installation:**
    *   Once all checks pass, click "Install cluster."
    *   Monitor progress in the UI.
9.  **Download `kubeconfig` and `kubeadmin` password** from the UI upon completion.

### Post-Installation Validation (Common for all methods)

1.  **Verify Cluster Health:**
    ```bash
    [bastion]$ oc get nodes -o wide
    [bastion]$ oc get clusterversion # Ensure Desired and Current versions match, and AVAILABLE=True
    [bastion]$ oc get co # Cluster Operators - all should be AVAILABLE=True, PROGRESSING=False, DEGRADED=False
    ```
2.  **Check Etcd Health:**
    *   SSH to a control plane node: `[bastion]$ oc debug node/<control-plane-node-name> -- chroot /host`
    *   Then on the node:
        ```bash
        [control-plane-node]# export ETCDCTL_CACERT=/etc/kubernetes/static-pod-resources/etcd-certs/ca-bundle.crt
        [control-plane-node]# export ETCDCTL_CERT=/etc/kubernetes/static-pod-resources/etcd-certs/server-crt.pem
        [control-plane-node]# export ETCDCTL_KEY=/etc/kubernetes/static-pod-resources/etcd-certs/server-key.pem
        [control-plane-node]# etcdctl --endpoints=$(oc get cm -n openshift-etcd etcd-endpoints -o jsonpath='{.data.host_endpoints_0}{","}{.data.host_endpoints_1}{","}{.data.host_endpoints_2}' | sed 's/,/ /g' | awk '{for(i=1;i<=NF;i++) print "https://"$i":2379";}' | paste -sd,) endpoint health --write-out=table
        [control-plane-node]# exit
        ```
3.  **Validate Core Services:**
    ```bash
    [bastion]$ oc get pods -n openshift-apiserver
    [bastion]$ oc get pods -n openshift-kube-scheduler
    [bastion]$ oc get pods -n openshift-kube-controller-manager
    [bastion]$ oc get pods -n openshift-ingress
    [bastion]$ oc get pods -n openshift-dns
    [bastion]$ oc get pods -n openshift-image-registry # Check if registry is configured and running
    [bastion]$ oc get pods -n openshift-monitoring # Check Prometheus, Grafana, etc.
    ```
4.  **Test Basic Functionality:**
    *   Deploy a sample application (e.g., `oc new-app httpd --name=test-httpd && oc expose svc/test-httpd`).
    *   Verify you can access the application via its route.
    *   Test storage provisioning with a sample PVC (see Pure Storage CSI section).
5.  **Configure Authentication and Authorization:** As mentioned, set up an IdP.

## Storage Integration and Configuration (Pure Storage CSI)

This section details deploying and configuring the Pure Storage CSI driver.

### 1. Deploy Pure Storage CSI Operator

1.  Log in to the OpenShift console as `kubeadmin` or a user with `cluster-admin` privileges.
2.  Navigate to **Operators → OperatorHub**.
3.  Search for "Pure Storage CSI Operator" (ensure it's the Red Hat Certified one).
4.  Click **Install**.
    *   **Update Channel:** Select a stable channel (e.g., `stable`, `latest`, check Pure's docs for recommended).
    *   **Installation Mode:** `All namespaces on the cluster (default)`.
    *   **Installed Namespace:** `openshift-operators` (default).
    *   **Update Approval:** `Automatic` or `Manual`.
5.  Click **Install** and wait for the operator to be installed successfully.

### 2. Create Pure Storage Array Credentials Secret

Create a Kubernetes secret containing the API token for the FlashArray user you created earlier.
**File: `pure-csi-secret.yaml`**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: pure-storage-csi-credentials # Or a name of your choice
  namespace: pure-storage-csi-driver # CSI driver's namespace
type: Opaque
stringData:
  # Option 1: Using arrays definition directly in secret (for one or more arrays)
  arrays: |-
    [{
        "FlashArray": {
            "ManagementEndpoint": "192.168.10.100", # IP or FQDN of FlashArray A
            "APIToken": "your-api-token-for-array-A"
        },
        "Labels": { "topology.purestorage.com/array-name": "FlashArray-A" }
    },
    {
        "FlashArray": {
            "ManagementEndpoint": "192.168.10.101", # IP or FQDN of FlashArray B (optional)
            "APIToken": "your-api-token-for-array-B"
        },
        "Labels": { "topology.purestorage.com/array-name": "FlashArray-B" }
    }]
  # Option 2: If using a values.yaml approach with Helm (less common for Operator install)
  # pure.json: |-
  #   {
  #     "FlashArrays": [
  #       {
  #         "MgmtEndPoint": "192.168.10.100",
  #         "APIToken": "your-api-token-for-array-A"
  #       }
  #       // Add more arrays if needed
  #     ],
  #     "FlashBlades": [] // If using FlashBlade
  #   }
```
**Note:** The `pure-storage-csi-driver` namespace is typically where the CSI driver components run. Check the Operator's documentation for the exact namespace. The `arrays` format is common for modern CSI driver versions.

Apply the secret:
```bash
[bastion]$ oc apply -f pure-csi-secret.yaml
```

### 3. Create Pure Storage CSI Driver Custom Resource (PureCSIDriver)
After the operator is installed, you create a `PureCSIDriver` CR to deploy the actual CSI driver components (controller, node plugins).
1.  Navigate to **Operators → Installed Operators**.
2.  Select the "Pure Storage CSI Operator".
3.  Find the "PureCSIDriver" API and click **Create Instance**.
4.  Configure the YAML:
    ```yaml
    apiVersion: operator.purestorage.com/v1alpha1 # Check API version from Operator
    kind: PureCSIDriver
    metadata:
      name: pure-csi # Name of the driver instance
      namespace: pure-storage-csi-driver # Namespace where driver components will run
    spec:
      # ClusterID is optional, used for identifying which cluster owns volumes if sharing arrays
      # clusterID: "flashstack-ocp-cluster1"
      namespace: pure-storage-csi-driver # Redundant, but often present
      # imagePullPolicy: IfNotPresent # Or Always
      # logLevel: info # Or debug

      # Define default SAN protocol (fc or iscsi)
      # This can be overridden in StorageClasses
      defaultSANTransport: fc # or "iscsi"

      # Array connection details (often taken from the secret, but can be specified here too)
      # If specified here, it might override/complement the secret
      # Ensure this matches how your chosen CSI version expects array config.
      # Modern versions prefer array config fully in the secret.
      # flasharray:
      #   credentials: pure-storage-csi-credentials # Name of the secret

      # Tolerations if your nodes have specific taints CSI components need to tolerate
      # tolerations:
      # - key: "key"
      #   operator: "Exists"
      #   effect: "NoSchedule"
    ```
    **Important:** Refer to the specific Pure Storage CSI Operator documentation for the exact structure of the `PureCSIDriver` CR, as it can change between operator versions. The trend is to simplify this CR and put more config into the Secret or ConfigMaps.

5.  Click **Create**. This will deploy the CSI controller pods and daemonsets for node plugins.
6.  Verify CSI pods are running:
    ```bash
    [bastion]$ oc get pods -n pure-storage-csi-driver # Or the namespace you used
    ```
    You should see controller pods and node pods (one per OCP node).

### 4. Create StorageClasses

Define StorageClasses to allow dynamic provisioning of storage with different characteristics.

**Example: `pure-block-prod.yaml` (Fibre Channel, High Performance)**
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: pure-block-fc-prod
  annotations:
    storageclass.kubernetes.io/is-default-class: "false" # Set one SC as default if desired
provisioner: csi.purestorage.com # Check exact provisioner name from Pure CSI docs
parameters:
  csi.storage.k8s.io/fstype: xfs # Or ext4
  backend: block
  # For FC, protocol is usually implicitly handled or specified if overriding default
  # san_protocol: fc # Optional, if defaultSANTransport in PureCSIDriver CR isn't FC

  # Pure Storage Specific Parameters (check Pure CSI docs for all options)
  # purestorage.com/qos: "gold" # If you have QoS policies on FlashArray
  # purestorage.com/protectiongroup: "ocp-prod-pg" # Assign to a protection group
  # purestorage.com/tags: "environment:prod,app:myapp"
reclaimPolicy: Delete # Or Retain (Retain keeps PV and backend volume after PVC deletion)
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer # Recommended for block, ensures pod placement before provisioning
```

**Example: `pure-block-iscsi-dev.yaml` (iSCSI, General Purpose)**
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: pure-block-iscsi-dev
provisioner: csi.purestorage.com
parameters:
  csi.storage.k8s.io/fstype: xfs
  backend: block
  san_protocol: iscsi # Explicitly state iSCSI if not default

  # purestorage.com/qos: "silver"
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
```

**Example: `pure-file-rwx.yaml` (NFS for ReadWriteMany, if FlashArray File Services are enabled)**
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: pure-file-rwx
provisioner: csi.purestorage.com
parameters:
  backend: file
  # purestorage.com/nfs_export_rules: "*(rw,no_root_squash)" # Example export rules
  # purestorage.com/qos: "bronze"
reclaimPolicy: Delete
allowVolumeExpansion: true
# volumeBindingMode: Immediate # Often used for file, but WaitForFirstConsumer is also valid
```

Apply the StorageClasses:
```bash
[bastion]$ oc apply -f pure-block-prod.yaml
[bastion]$ oc apply -f pure-block-iscsi-dev.yaml
[bastion]$ oc apply -f pure-file-rwx.yaml # If using file
[bastion]$ oc get sc # Verify they are created
```

### 5. Configure Pure Storage Snapshot Controller (VolumeSnapshotClass)
The CSI driver usually includes snapshot capabilities. Create a `VolumeSnapshotClass`.
**File: `pure-snapshotclass.yaml`**
```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: pure-block-snapshotclass # Name it meaningfully
driver: csi.purestorage.com # Must match the provisioner name in your StorageClasses
deletionPolicy: Delete # 'Delete' removes snapshot on FlashArray when VolumeSnapshot object is deleted. 'Retain' keeps it.
# parameters: # Optional driver-specific parameters for snapshots
  # purestorage.com/protectiongroup-snapshot: "true" # Example: if you want to snapshot entire PG
```
Apply it:
```bash
[bastion]$ oc apply -f pure-snapshotclass.yaml
```

### 6. Test Dynamic Provisioning and Snapshots
**Test PVC:**
```yaml
# File: test-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pure-test-pvc
  namespace: default
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
  storageClassName: pure-block-fc-prod # Use one of your created SCs
```
Apply: `[bastion]$ oc apply -f test-pvc.yaml`
Check: `[bastion]$ oc get pvc pure-test-pvc -n default` (Status should become `Bound`)
Also check on the FlashArray GUI/CLI that a volume was created.

**Test Pod using PVC:**
```yaml
# File: test-pod-pure.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pure-test-pod
  namespace: default
spec:
  containers:
  - image: nginx
    name: nginx
    volumeMounts:
    - mountPath: /usr/share/nginx/html
      name: my-pure-volume
  volumes:
  - name: my-pure-volume
    persistentVolumeClaim:
      claimName: pure-test-pvc
```
Apply: `[bastion]$ oc apply -f test-pod-pure.yaml`
Check: `[bastion]$ oc get pod pure-test-pod -n default` (Should be `Running`)

**Test Snapshot:**
```yaml
# File: test-snapshot.yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: pure-test-snapshot
  namespace: default
spec:
  volumeSnapshotClassName: pure-block-snapshotclass
  source:
    persistentVolumeClaimName: pure-test-pvc
```
Apply: `[bastion]$ oc apply -f test-snapshot.yaml`
Check: `[bastion]$ oc get volumesnapshot -n default` (READYTOUSE should be true)
Also check on FlashArray that a snapshot was created for the volume.

## OpenShift Virtualization Implementation

OpenShift Virtualization (based on KubeVirt) allows running VMs alongside containers.

#### 1. Install the OpenShift Virtualization Operator
1.  In OpenShift Console: **Operators → OperatorHub**.
2.  Search for "OpenShift Virtualization".
3.  Click **Install**.
    *   **Update channel:** Select a stable channel (e.g., `stable` or matching your OCP minor version).
    *   **Installation mode:** `All namespaces...`
    *   **Installed Namespace:** `openshift-cnv` (default).
    *   **Update approval:** `Automatic`.
4.  Wait for the operator installation to complete.

#### 2. Create the HyperConverged Custom Resource
This CR deploys and configures all necessary OpenShift Virtualization components.
1.  After operator installation, it might prompt you to "Create HyperConverged". Or navigate to **Installed Operators → OpenShift Virtualization → HyperConverged** tab → **Create HyperConverged**.
2.  A default YAML will be provided. Review and customize if needed.
    ```yaml
    apiVersion: hco.kubevirt.io/v1beta1
    kind: HyperConverged
    metadata:
      name: kubevirt-hyperconverged
      namespace: openshift-cnv
    spec:
      # featureGates: # Enable or disable specific features
      #   sriov: true # If you plan to use SR-IOV for VM NICs
      # liveMigrationConfig: # Customize live migration parameters
      #   parallelMigrationsPerCluster: 5
      #   parallelOutboundMigrationsPerNode: 2
      #   bandwidthPerMigration: "64Mi"
      # permittedHostDevices: # For device passthrough
      #   pciHostDevices:
      #   - pciDeviceSelector: "10DE:1EB8" # Example: NVIDIA GPU Vendor:Device ID
      #     resourceName: "nvidia.com/gpu"
      #     externalResourceProvider: true
      # infra: # Configure placement for infra components if needed
      #   nodePlacement:
      #     nodeSelector:
      #       kubernetes.io/hostname: "specific-infra-node"
      # workProfile: # Optimize for specific profiles if needed
      #   nodePlacement:
      #     nodeSelector:
      #       node-role.kubernetes.io/worker: "" # Default: runs on workers
    ```
3.  Click **Create**. This will deploy pods for KubeVirt, CDI (Containerized Data Importer), networking components, etc., in the `openshift-cnv` namespace.
4.  Monitor progress: `[bastion]$ oc get pods -n openshift-cnv -w`

#### 3. Configure Storage for OpenShift Virtualization
VMs require storage for their disks. Use the Pure Storage CSI.
1.  **Create a dedicated StorageClass for Virtual Machine Disks (recommended):**
    ```yaml
    # File: pure-vm-storageclass.yaml
    apiVersion: storage.k8s.io/v1
    kind: StorageClass
    metadata:
      name: pure-vm-block # Or pure-vm-fc, pure-vm-iscsi
      annotations:
        storageclass.kubernetes.io/is-default-class: "false"
    provisioner: csi.purestorage.com # Your Pure CSI provisioner
    parameters:
      csi.storage.k8s.io/fstype: xfs
      backend: block
      # Add any Pure-specific params like QoS, protection group, etc.
    reclaimPolicy: Delete # Or Retain, depending on your data persistence needs for VMs
    allowVolumeExpansion: true
    volumeBindingMode: WaitForFirstConsumer
    ```
    Apply: `[bastion]$ oc apply -f pure-vm-storageclass.yaml`

2.  **Set Default StorageClass for DataVolumes (CDI):**
    CDI imports VM images into PVCs. You can set a default StorageClass for these operations.
    *   Edit the `CDI` CR: `[bastion]$ oc edit cdi cdi -n openshift-cnv`
    *   Add/modify the `spec.config.dataVolumeTTLSeconds` or `spec.config.filesystemOverhead` if needed.
    *   To set a default storage class for CDI operations (uploads, imports):
        ```yaml
        # In the CDI CR spec:
        spec:
          config:
            # ... other CDI config ...
            defaultStorageClass: pure-vm-block # Your VM storage class
        ```
    Alternatively, users will specify the StorageClass when creating DataVolumes or VMs.

#### 4. Verify OpenShift Virtualization Deployment
1.  Check all pods in `openshift-cnv` are running: `[bastion]$ oc get pods -n openshift-cnv`
2.  Navigate to **Virtualization** in the OpenShift Web Console.
3.  Try creating a test VM:
    *   Use one of the provided RHEL/Windows templates or import your own image.
    *   Ensure it can boot, get an IP, and disk I/O works.
    *   Test live migration between worker nodes.

#### 5. Configure VM Networks (NetworkAttachmentDefinitions - NADs)
VMs can connect to default pod networks or dedicated L2 networks using NADs.
**Example: Bridged Network NAD (connecting VMs to an existing host network/VLAN)**
This requires the underlying worker nodes to have a bridge configured (e.g., `br1`) that is connected to the desired VLAN (e.g., VLAN 50). You might need a `MachineConfig` to create this bridge on RHCOS nodes.
```yaml
# File: vm-bridge-vlan50-nad.yaml
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: vm-vlan50-bridge
  namespace: default # Or the namespace where VMs will use this
spec:
  config: '{
    "cniVersion": "0.3.1",
    "name": "vm-vlan50-bridge",
    "type": "cnv-bridge", # KubeVirt bridge CNI
    "bridge": "br50",    # Name of the bridge on the host
    "vlan": 50,          # Optional: if bridge is a trunk, CNI can tag traffic
    "ipam": {            # Optional: if you want CNI to assign IPs from a range
      "type": "static"   # Or use "dhcp", or an external IPAM like "whereabouts"
    }
    # For whereabouts IPAM:
    # "ipam": {
    #   "type": "whereabouts",
    #   "range": "192.168.50.0/24",
    #   "gateway": "192.168.50.1"
    # }
  }'
```
Apply: `[bastion]$ oc apply -f vm-bridge-vlan50-nad.yaml -n default`

**Using SR-IOV for high-performance VM networking:**
*   Requires SR-IOV capable NICs on UCS servers, BIOS/Firmware configuration, and OCP SR-IOV Network Operator setup.
*   Create SR-IOV network policies and then SR-IOV NADs. This is an advanced topic beyond this overview.

When creating a VM, you can then attach an interface using this NAD.

## Security Hardening

This section covers comprehensive security hardening recommendations.
`[DIAGRAM: Security Layers - Illustrating security controls at Infrastructure (UCS, Pure), Host (RHCOS), OCP Platform, Network, and Application layers.]`

*(Content from your original "Security Hardening" section was very good and detailed. I will integrate it here, with minor adjustments for flow and any new context from previous sections. I'll assume the detailed YAMLs and commands you provided are to be kept.)*

### Platform Security Recommendations

#### OpenShift Container Platform Security

1.  **Authentication and Authorization**
    *   Configure enterprise identity provider integration (LDAP example provided was good).
        *   Ensure the `ldap-secret` is created in `openshift-config` namespace.
    *   Implement strong Role-Based Access Control (RBAC) policies (group examples provided were good).
        *   Follow the principle of least privilege.
2.  **Network Security**
    *   Implement `NetworkPolicy` objects to control pod-to-pod communication (example provided was good).
    *   Utilize OVN-Kubernetes advanced features like egress firewall, egress IP.
    *   Consider IPsec for inter-node pod traffic encryption if required (command provided was good, but note performance impact).
3.  **Image Security & Vulnerability Management**
    *   Integrate with Red Hat Quay or another image registry with scanning capabilities (Clair, Trivy).
        *   (Quay Operator subscription example was good - check for latest stable channel).
    *   Use image signing and enforce policies to only run signed/trusted images.
    *   Install and configure the **Compliance Operator** for security scanning and remediation against profiles like CIS Benchmarks, NIST.
        *   (Compliance Operator subscription and ScanSetting examples were good - check for latest stable channel).
4.  **Secrets Management**
    *   Use built-in Kubernetes Secrets for sensitive data.
    *   Encrypt etcd data (default in OCP 4.x).
    *   For more advanced secrets management, integrate with external providers like HashiCorp Vault.
        *   (Vault Operator subscription example was good - check for community/stable channel).
5.  **API Server Hardening**
    *   Configure API server audit logging.
    *   Restrict access to the API server using network policies or firewalls.
    *   Use `APIServer` CR to configure settings like `tlsSecurityProfile`.
6.  **Node Security**
    *   Use `MachineConfig` objects to apply host-level security settings (e.g., disabling unused services, custom SSHD config).
    *   Ensure regular updates to RHCOS via OCP update channels.

### Infrastructure Security

#### Cisco UCS Security
1.  **Access Control:**
    *   Implement RBAC in UCS Manager and/or Intersight.
    *   Use strong passwords and multi-factor authentication (MFA) for admin access.
    *   Use dedicated service accounts with least privilege for automation.
2.  **Network Security:**
    *   Segment management networks (CIMC, UCSM, Intersight) from data/production networks.
    *   Use VLANs and ACLs to restrict access.
    *   Disable unused ports on Fabric Interconnects.
    *   Enable secure management protocols (HTTPS, SSHv2) and disable insecure ones (Telnet, HTTP).
3.  **Firmware Management:** Regularly update UCS firmware to patched versions.
4.  **Secure Boot:** Enable Secure Boot in BIOS policies and ensure RHCOS supports it if used.

#### Pure Storage Security
1.  **Authentication and Authorization:**
    *   Integrate FlashArray management with directory services (LDAP/AD) for centralized authentication.
    *   Implement RBAC for Pure Storage management roles.
    *   Use strong API tokens with least privilege for CSI and other integrations; rotate them regularly.
2.  **Data Security:**
    *   Data-at-Rest Encryption (DARE) is always on by default.
    *   Configure secure data reduction (deduplication, compression).
    *   Use volume encryption/protection for sensitive workloads if finer-grained control beyond array-level encryption is needed.
3.  **Network Hardening:**
    *   **FC:** Use VSANs and proper zoning to isolate storage traffic.
    *   **iSCSI:** Use dedicated VLANs, CHAP authentication, and IP ACLs on the array.
    *   **Management:** Secure management interfaces with ACLs and use HTTPS.
    *   Disable unused network services/ports on the array.
4.  **Auditing:** Enable audit logging on the FlashArray and forward logs to a SIEM.
5.  **Purity OS Updates:** Keep Purity OS updated with latest security patches.

### Host-Level Security (RHCOS)

1.  **RHCOS Hardening (Managed by MCO):**
    *   RHCOS is immutable and managed by the Machine Config Operator (MCO).
    *   Apply security-related `MachineConfig` objects (e.g., for SSH hardening, custom sysctl params).
        *   (The `99-worker-ssh-hardening` MachineConfig example was good).
    *   Ensure `selinux=enforcing` (default).
2.  **Container Runtime Security:**
    *   OpenShift uses CRI-O, which is hardened.
    *   Use Security Context Constraints (SCCs) to limit pod/container privileges.
        *   Prioritize using restrictive SCCs (e.g., `restricted-v2`).
        *   (The custom `restricted-scc` example was very good and detailed, illustrating key restrictions).
    *   Avoid running privileged containers unless absolutely necessary.
    *   Define Pod Security Admission (PSA) policies (replacing PodSecurityPolicies in newer Kubernetes). OpenShift 4.11+ integrates PSA.

### Security Monitoring and Auditing

1.  **Log Management:**
    *   Deploy the OpenShift Logging stack (Elasticsearch, Fluentd, Kibana).
        *   (OpenShift Logging Operator subscription example was good).
    *   Configure log forwarding to an external SIEM (Splunk, QRadar, ELK).
        *   (The `ClusterLogForwarder` example for syslog and Elasticsearch was excellent).
        *   Ensure audit logs (Kubernetes API audit, node auditd) are collected.
2.  **Security Information and Event Management (SIEM) Integration:**
    *   Forward logs from OCP, UCS, Pure Storage, and network devices to a central SIEM for correlation and alerting.
3.  **Intrusion Detection/Prevention Systems (IDS/IPS):**
    *   Consider IDS/IPS at the network perimeter and potentially for inter-pod traffic inspection if required (e.g., using OVN-Kubernetes features or third-party CNIs/tools).
4.  **Runtime Security Monitoring:**
    *   Tools like Falco, Aqua Security, Sysdig Secure can provide container runtime threat detection.

### Security Compliance Validation

1.  **Regular Security Audits:**
    *   Conduct periodic internal and external security audits.
    *   Perform penetration testing against the platform and deployed applications.
2.  **Compliance Reporting:**
    *   Use the Compliance Operator to generate reports against security benchmarks (CIS, NIST, PCI-DSS).
3.  **Documentation and Procedures:**
    *   Maintain up-to-date security architecture documentation.
    *   Develop and test Incident Response (IR) plans.
    *   Define security escalation paths.

## Day 2 Operations

Effective Day 2 operations are crucial for maintaining a healthy, performant, and reliable OpenShift on FlashStack environment.
*(Content from your original "Day 2 Operations" section was also very good. I will integrate and slightly expand.)*

### Monitoring and Observability

#### OpenShift Monitoring Stack (Prometheus & Grafana)
1.  **Core Platform Monitoring:** Enabled by default, monitors cluster components.
2.  **User Workload Monitoring:**
    *   Enable it to monitor your own applications:
        *   (The `cluster-monitoring-config` ConfigMap example to enable `enableUserWorkload` and configure persistent storage for Prometheus/Alertmanager/Grafana using Pure StorageClasses was excellent).
3.  **Custom Alerting Rules:**
    *   Create `PrometheusRule` CRs for custom alerts specific to your applications or FlashStack components (e.g., Pure Storage metrics if an exporter is available and scraped).
        *   (The `PureStorageVolumeHighLatency` and `PureStorageHighUtilization` alert examples were very good. Note: These rely on having a Prometheus exporter for Pure Storage that exposes these metrics, which might be a separate deployment).
4.  **Grafana Dashboards:**
    *   Use built-in dashboards.
    *   Create/import custom Grafana dashboards for FlashStack components or applications.
        *   (The ConfigMap example for injecting a custom Pure Storage dashboard was good).
        *   Explore community dashboards or dashboards from Pure Storage (if available).
5.  **Pure1 Monitoring:** Leverage Pure1 for array-level health, performance, and capacity analytics.

#### Centralized Logging (OpenShift Logging Stack)
1.  **Deploy OpenShift Logging Stack (EFK/Loki):**
    *   Uses Elasticsearch/Fluentd/Kibana (EFK) or Loki/Fluentd/Grafana.
    *   (The `ClusterLogging` CR example for EFK with Pure Storage SC was excellent).
    *   Configure storage persistence for Elasticsearch/Loki using Pure Storage (RWX with Pure File for multiple ES pods, or RWO for single instances/Loki).
2.  **Log Retention Policies:**
    *   Configure Elasticsearch Index Management (ILM) policies or Loki retention policies to manage log storage.
        *   (The `IndexManagement` CR example for different log types was very detailed and useful).
3.  **Kibana/Grafana for Log Exploration:** Use these tools to search, visualize, and analyze logs.
4.  **Log Forwarding:** As covered in Security, forward logs to an external SIEM.

### Capacity Management

1.  **Cluster Resource Monitoring:**
    *   Use `oc adm top nodes` and `oc adm top pods`.
    *   Monitor Prometheus metrics for CPU, memory, disk, network usage across nodes and for pods/namespaces.
    *   Set up alerts for high resource utilization.
2.  **Storage Capacity (Pure Storage):**
    *   Monitor FlashArray capacity via Pure1, Purity GUI/CLI, or custom scripts using the Pure Storage REST API.
        *   (The bash script example for querying Pure capacity was good. For production, use secure credential handling).
    *   Track PVC usage per namespace/application.
    *   Set `ResourceQuota` objects in namespaces to limit storage requests.
        *   (The `ResourceQuota` example with Pure SCs was good).
3.  **OpenShift Quotas and LimitRanges:**
    *   Define `ResourceQuota` (for CPU, memory, PVC count, storage capacity) and `LimitRange` (default/max/min requests/limits for pods/containers) per project/namespace.
4.  **Cost Management (Optional):**
    *   Install and configure the **Koku Metrics Operator** or OpenShift Cost Management (if subscribed) to track resource consumption and associate costs.
        *   (Koku Metrics Operator subscription example was good).

### Troubleshooting (See dedicated Troubleshooting section for more detail)

#### Common Issues and Quick Checks
1.  **Node Health:** `oc get nodes`, `oc describe node <node>`, check `kubelet` and `crio` services via `oc debug node/...`.
2.  **Pod Issues:** `oc get pods -n <ns>`, `oc describe pod <pod> -n <ns>`, `oc logs <pod> -n <ns>`, `oc exec -it <pod> -n <ns> -- bash`.
3.  **Storage (PVCs):** `oc get pvc -n <ns>`, `oc describe pvc <pvc> -n <ns>`, check Pure CSI driver logs (`oc logs -n pure-storage-csi-driver -l app=pure-csi-controller` or similar).
4.  **Network:** Test connectivity from pods, check `NetworkPolicy`, Ingress/Route status.

### Standard Maintenance Procedures

1.  **Node Maintenance (Cord MCO Upgrade Management):**
    *   When performing maintenance on a node (e.g., hardware replacement managed outside OCP):
        ```bash
        [bastion]$ oc adm cordon <node-name>
        [bastion]$ oc adm drain <node-name> --ignore-daemonsets --delete-emptydir-data
        # Perform maintenance on the UCS server
        [bastion]$ oc adm uncordon <node-name>
        ```
    *   For MCO-driven updates (like OS changes), the MCO handles draining and uncordoning.
2.  **Cluster Updates (OCP):** See "Upgrades and Lifecycle Management" section.
3.  **Etcd Backup and Restore:** See "Backup and Disaster Recovery" section.
4.  **Certificate Management:**
    *   Most core OpenShift certificates auto-rotate.
    *   Monitor certificate expiration:
        ```bash
        [bastion]$ oc -n openshift-kube-apiserver-operator get secret kube-apiserver-to-kubelet-signer -o jsonpath='{.metadata.annotations.auth\.openshift\.io/certificate-not-after}'
        # And check other important certs.
        ```
    *   Refer to Red Hat documentation for manual rotation procedures if needed for specific certificates.
5.  **Reviewing `must-gather`:** Periodically run `oc adm must-gather` to collect diagnostic data and become familiar with its contents for faster troubleshooting when issues arise.

## Performance Tuning

Optimizing performance involves tuning at multiple layers of the FlashStack and OpenShift stack.

### OpenShift Platform Tuning

1.  **CPU Manager for Guaranteed Pods:**
    *   For latency-sensitive, CPU-bound workloads, enable the `CPUManager` policy (`static`) in the Kubelet configuration (via `KubeletConfig` CR). This dedicates cores to specific pods.
    *   Pods must request integer CPUs and have `Guaranteed` QoS class.
2.  **Memory Manager for Guaranteed Pods:**
    *   Enable `MemoryManager` policy (`static`) via `KubeletConfig` to ensure guaranteed memory allocation and NUMA alignment for `Guaranteed` QoS pods.
3.  **HugePages:**
    *   Configure HugePages (2MB or 1GB) on worker nodes for applications that benefit (e.g., databases, JVMs).
    *   Request HugePages in pod specs. Managed via `MachineConfig` and pod resource requests.
4.  **Performance Profile Creator (Operator):**
    *   Use the Performance Addon Operator to create `PerformanceProfile` CRs. This automates tuning of nodes for low-latency workloads by configuring:
        *   `tuned` profiles (e.g., `realtime-virtual-host`).
        *   CPU affinity, HugePages, disabling irqbalance.
        *   Reserving cores for housekeeping.
5.  **Topology Manager:**
    *   Enable `TopologyManager` policy (e.g., `single-numa-node`) via `KubeletConfig` to align CPU, memory, and device (e.g., SR-IOV NICs, GPUs) allocations from the same NUMA node.
6.  **Node Tuning (Tuned Operator):**
    *   Leverage the Node Tuning Operator to apply custom `tuned` profiles for specific worker node roles or workloads.

### Network Performance

1.  **MTU Size:**
    *   For iSCSI/NFS storage networks, ensure MTU 9000 (Jumbo Frames) is configured end-to-end (UCS vNICs, FIs, Nexus switches, Pure Storage ports).
    *   For pod networks (OVN-Kubernetes), default MTU is often sufficient. If overlay traffic needs higher MTU, the underlay must support it.
2.  **SR-IOV (Single Root I/O Virtualization):**
    *   For VMs (via OpenShift Virtualization) or specific container workloads requiring direct hardware access and low-latency networking.
    *   Requires SR-IOV capable NICs (e.g., Cisco VICs), UCS configuration, and the SR-IOV Network Operator in OpenShift.
3.  **DPDK (Data Plane Development Kit):**
    *   For network-intensive applications that can leverage user-space networking. Requires DPDK-enabled applications and often SR-IOV.
4.  **Network Policies:** Be mindful that complex `NetworkPolicy` objects can add slight overhead. Optimize them where possible.
5.  **Ingress Controller Tuning:** Scale Ingress Controller pods and tune their configurations (e.g., keep-alive timeouts, worker processes) based on load.

### Storage Performance (Pure Storage)

1.  **Pure Storage QoS:**
    *   Define QoS policies (bandwidth/IOPS limits) on the FlashArray if you need to guarantee or limit performance for specific OpenShift StorageClasses or applications.
    *   Map these to `purestorage.com/qos` parameter in `StorageClass`.
2.  **Multipathing:**
    *   Ensure multipathing is correctly configured for FC and iSCSI from RHCOS nodes to the FlashArray. The Pure CSI driver and RHCOS defaults usually handle this well.
3.  **Volume Layout:** For specific high-performance databases, consult database vendor best practices for volume layout (e.g., separate volumes for data, logs, tempdb) and map these to distinct PVCs using appropriate StorageClasses.
4.  **Snapshot Overhead:** While Pure Storage snapshots are highly efficient, very frequent snapshots of highly transactional volumes can have a minor performance impact. Schedule appropriately.

### Application-Level Tuning

1.  **JVM Tuning:** For Java applications, tune heap size (`-Xms`, `-Xmx`), garbage collection algorithms, and enable HugePages if beneficial.
2.  **Database Tuning:** Optimize database configuration, indexing, query plans, and connection pooling.
3.  **Connection Pooling:** Use connection pooling for applications connecting to databases or other backend services.
4.  **Resource Requests and Limits:** Set appropriate CPU/memory requests and limits for your application pods to ensure proper scheduling and prevent resource contention. Requesting resources allows Kubernetes to make better scheduling decisions.

### Monitoring Performance
*   Use OpenShift Monitoring (Prometheus/Grafana) to track key performance indicators (KPIs) for nodes, pods, storage (CSI metrics), and network.
*   Use Pure1 and FlashArray native tools to monitor storage performance (latency, IOPS, bandwidth, queue depth).
*   Use `oc adm top nodes/pods` for quick resource usage checks.

## Backup and Disaster Recovery

A robust BCDR strategy is critical for protecting your OpenShift cluster and applications.

### 1. Etcd Backup

Etcd stores the state of your OpenShift cluster. Regular backups are essential.
*   **Automated Backups:** OpenShift 4.x automatically performs etcd backups on control plane nodes (typically in `/var/lib/etcd-backup/` or as managed by `cluster-backup.sh`).
*   **Manual Backup (if needed or for off-cluster storage):**
    ```bash
    [bastion]$ oc debug node/<master-node-name>
    [Debugging node ...]# chroot /host
    [master-node]# /usr/local/bin/cluster-backup.sh /home/core/backup_dir/ # Path must exist
    [master-node]# exit
    [Exiting debug container ...]$
    ```
    *   Securely copy these backups off the cluster to a separate location (e.g., Pure Storage SafeMode snapshots, offsite backup).
*   **Disaster Recovery:** Etcd backups are crucial for recovering a cluster in a catastrophic failure scenario. Follow Red Hat's official DR procedures for restoring from etcd.

### 2. Application Backup and Restore (OADP)

Use the **OpenShift API for Data Protection (OADP) Operator**, which is based on Velero.
1.  **Install OADP Operator:** From OperatorHub.
2.  **Configure Backup Storage Location:**
    *   This is where backup data and metadata will be stored. Options include:
        *   S3-compatible object storage (e.g., AWS S3, MinIO, Ceph RGW).
        *   Pure Storage FlashBlade S3 (if available).
        *   NFS (less common for OADP's primary target, but possible for some Velero plugins).
    *   Create a `BackupStorageLocation` CR.
3.  **Configure Volume Snapshot Location (for CSI):**
    *   OADP leverages the CSI snapshot capabilities for PVs.
    *   Create a `VolumeSnapshotLocation` CR pointing to your Pure Storage CSI driver.
4.  **Backup CRs:**
    *   Create `Backup` CRs to define what to back up (namespaces, labels, resources, PVs).
    *   Schedule regular backups.
    ```yaml
    # Example Backup CR
    apiVersion: velero.io/v1
    kind: Backup
    metadata:
      name: myapp-backup
      namespace: openshift-adp # OADP operator namespace
    spec:
      includedNamespaces:
        - myapp-ns
      storageLocation: default # Your BackupStorageLocation name
      volumeSnapshotLocations:
        - default-csi # Your VolumeSnapshotLocation for Pure CSI
      ttl: "720h" # 30 days
    ```
5.  **Restore CRs:**
    *   Create `Restore` CRs to restore applications to the same or a different cluster.

### 3. Storage-Level Backup and Replication (Pure Storage)

Leverage FlashArray's native capabilities:
1.  **Local Snapshots:**
    *   Highly efficient, space-saving snapshots for quick, local recovery of PV data.
    *   Configure Protection Groups on the FlashArray to group application volumes and apply consistent snapshot schedules (e.g., hourly, daily).
    *   These are distinct from CSI snapshots but can complement them.
2.  **Asynchronous Replication:**
    *   Replicate Protection Group snapshots to a secondary Pure Storage FlashArray at a DR site.
    *   Configure replication schedules (e.g., every 15 mins, hourly).
    *   RPO depends on replication frequency.
3.  **Synchronous Replication (ActiveCluster):**
    *   For zero RPO and near-zero RTO between two FlashArrays (stretched cluster or metro-DR).
    *   Requires specific network latency and configuration.
    *   Volumes appear read/write on both arrays simultaneously. OpenShift nodes connect to their local array.
4.  **Pure Storage SafeMode™ Snapshots:**
    *   Immutable snapshots protected from deletion (even by array admins) for ransomware protection.
5.  **CloudSnap™ / Cloud Block Store™:**
    *   Replicate snapshots to cloud object storage (AWS S3, Azure Blob) or Pure Cloud Block Store for offsite backups or cloud DR.

### Disaster Recovery Strategy

1.  **Define RPO (Recovery Point Objective) and RTO (Recovery Time Objective)** for different applications.
2.  **DR Scenarios:**
    *   **Application Corruption:** Restore from OADP backup or Pure Storage snapshot.
    *   **Single Cluster Failure (recoverable):** Restore etcd, potentially re-deploy OCP, restore apps via OADP.
    *   **Full Site Disaster (DR site needed):**
        *   **Cold DR:** Rebuild OCP at DR site, restore etcd, promote replicated Pure Storage volumes, restore apps via OADP.
        *   **Warm DR:** Pre-staged OCP at DR, restore etcd, promote replicated Pure volumes, restore apps.
        *   **Hot DR (e.g., with ActiveCluster):** OCP running at DR site (potentially scaled down), Pure volumes already active or quickly promotable. Applications might need to be started or re-routed. Red Hat ACM can help manage multi-cluster application deployment for DR.
3.  **DR Plan and Testing:**
    *   Document DR procedures thoroughly.
    *   Regularly test DR plans (tabletop exercises, partial failovers, full failovers if possible).

## Upgrades and Lifecycle Management

Keeping the platform and infrastructure up-to-date is vital for security, stability, and new features.

### OpenShift Container Platform Upgrades

1.  **Upgrade Channels and Paths:**
    *   OCP uses upgrade channels (e.g., `stable-4.13`, `fast-4.13`, `candidate-4.13`).
    *   The Cluster Version Operator (CVO) manages upgrades.
    *   `oc adm upgrade` shows available paths and status.
2.  **Upgrade Process:**
    *   **Review Release Notes:** Always check for known issues or prerequisites.
    *   **Backup:** Perform etcd and application backups (OADP) before major upgrades.
    *   **Initiate Upgrade:**
        ```bash
        [bastion]$ oc adm upgrade # Check available updates
        [bastion]$ oc adm upgrade --to-latest # Upgrade to latest in current channel
        [bastion]$ oc adm upgrade --to <version> # Upgrade to a specific version
        ```
        Or use the Web Console (Administrator → Cluster Settings → Cluster Operators → Update).
    *   **Monitoring:**
        *   `oc get clusterversion`
        *   `oc get co` (watch for `PROGRESSING=True`)
        *   `oc adm upgrade` (shows progress)
    *   The CVO updates control plane nodes first (one by one, draining), then worker nodes (respecting PodDisruptionBudgets).
3.  **Operator Lifecycle Management (OLM):**
    *   Operators for CSI drivers, OpenShift Virtualization, etc., are typically updated via OLM, often tied to OCP version or their own channels.
    *   Configure update approval strategy (Automatic or Manual) for operators.

### Cisco UCS Firmware Upgrades

1.  **Using Cisco Intersight (Recommended):**
    *   Create Firmware Bundles/Policies.
    *   Schedule firmware upgrades for Fabric Interconnects, IOMs, server adapters, BIOS, CIMC.
    *   Intersight can manage rolling upgrades with workload migration (if integrated with vCenter, or manual cordoning for OCP nodes).
2.  **Using UCS Manager (Standalone):**
    *   Download firmware bundles from Cisco.com.
    *   Upload to FIs.
    *   Upgrade FIs (requires cluster failover).
    *   Upgrade IOMs, Adapters, CIMC, BIOS on blades/racks (requires node reboots).
    *   Drain OCP nodes (`oc adm drain`) before rebooting for firmware updates.
3.  **Best Practices:**
    *   Follow Cisco's recommended upgrade paths and compatibility matrices.
    *   Upgrade non-production environments first.
    *   Schedule maintenance windows.

### Pure Storage Purity OS Upgrades

1.  **Non-Disruptive Upgrades (NDU):** Purity upgrades are designed to be non-disruptive to I/O.
2.  **Planning:**
    *   Check Pure Storage release notes for new features, fixes, and any prerequisites.
    *   Consult Pure Support or documentation for recommended versions.
3.  **Upgrade Process (via Purity GUI, CLI, or Pure1 Remote Upgrade):**
    *   Download the Purity upgrade package.
    *   Purity performs pre-upgrade health checks.
    *   The upgrade process updates one controller at a time, failing over I/O transparently.
4.  **Post-Upgrade:** Verify array health and connectivity from OpenShift nodes.
5.  **CSI Driver Compatibility:** Ensure your Pure Storage CSI driver version is compatible with the new Purity OS version. Usually, CSI drivers are forward/backward compatible to some extent, but it's best to check.

### Order of Operations (General Guideline)
*   Generally, it's good practice to keep infrastructure (UCS, Pure) firmware/OS relatively current before major OCP upgrades, ensuring compatibility.
*   Always check interoperability matrices from Red Hat, Cisco, and Pure Storage.
*   For OCP upgrades, ensure add-on operators (like Pure CSI, OVSV) are compatible with the target OCP version.

## Automation and CI/CD Integration

Automating infrastructure and application deployment enhances consistency, speed, and reliability.

### Infrastructure as Code (IaC)

1.  **Terraform:**
    *   **Cisco Intersight Provider:** Automate UCS server profile deployment, policies, FI configuration.
    *   **Nexus Provider (community/Cisco):** Automate switch configurations (VLANs, interfaces, vPCs).
    *   **Pure Storage Provider (community/Pure):** Automate volume provisioning, host groups, protection groups (though CSI handles most OCP volume needs).
    *   **OpenShift Installer with Terraform:** The IPI installer can output Terraform manifests for managing cluster infrastructure resources on some platforms (e.g., cloud providers). For bare metal, Terraform's role is more around the underlying UCS/network.
2.  **Ansible:**
    *   **Pre/Post OCP Installation Tasks:** Configure NTP, DNS, bastion host setup, network device configs (using NAPALM or specific modules).
    *   **Application Configuration Management:** Manage application configs deployed on OpenShift.
    *   **UCS/Pure Modules:** Ansible modules exist for managing Cisco UCS and Pure Storage.
    *   **KubeVirt Ansible Modules:** For automating VM provisioning on OpenShift Virtualization.

### Configuration Management for OpenShift
*   **`MachineConfig` Objects:** As shown earlier, for declarative RHCOS node configuration.
*   **Operators:** Package, deploy, and manage applications and their dependencies on OpenShift.
*   **Helm Charts:** Package and deploy applications.
*   **Kustomize:** Customize Kubernetes YAML manifests without templating.

### CI/CD Pipelines for Applications on OpenShift

1.  **OpenShift Pipelines (Tekton):**
    *   Cloud-native CI/CD directly within OpenShift. Define `Pipeline`, `Task`, `PipelineRun` CRs.
    *   Integrates with Git, image registries, testing tools.
2.  **Jenkins:**
    *   Deploy Jenkins on OpenShift.
    *   Use Kubernetes plugin for dynamic agent provisioning.
    *   Integrate with `oc` client and image build/push tools.
3.  **GitLab CI/CD:**
    *   Use GitLab Runners on OpenShift (Kubernetes executor).
    *   Define CI/CD stages in `.gitlab-ci.yml`.
4.  **GitHub Actions:**
    *   Use self-hosted runners on OpenShift or actions that interact with `oc` CLI.
5.  **Common Pipeline Stages:**
    *   Code Checkout (Git)
    *   Build (Maven, Gradle, npm, Go build, etc.)
    *   Unit/Integration Tests
    *   Static Code Analysis / Security Scan (SonarQube, Clair)
    *   Containerize (Dockerfile, S2I builds in OpenShift)
    *   Push to Image Registry (Quay, OpenShift Internal Registry, Harbor)
    *   Deploy to Dev/Staging OpenShift Namespace (using `oc apply`, Helm, Kustomize)
    *   Automated End-to-End Tests
    *   Promote to Production

### GitOps for Cluster and Application Configuration

Manage OpenShift cluster configuration and application deployments declaratively using Git as the source of truth.
1.  **Argo CD (Operator Available):**
    *   Pulls desired state from Git (Kubernetes manifests, Helm charts, Kustomize).
    *   Compares with live cluster state and syncs changes.
    *   Provides UI for visualizing sync status and application health.
2.  **Flux CD:**
    *   Another popular GitOps tool, part of CNCF.
    *   Uses Kustomize and Helm controllers.

**GitOps Workflow:**
1.  Developers push code changes to application Git repo.
2.  CI pipeline builds image, pushes to registry.
3.  CI pipeline updates Kubernetes manifests in a configuration Git repo (e.g., bumps image tag).
4.  GitOps tool (Argo CD/Flux) detects change in config repo.
5.  GitOps tool applies changes to the OpenShift cluster.

## Troubleshooting (Expanded)

This section expands on troubleshooting techniques.

### General Approach
1.  **Isolate the Problem:** Infrastructure, OpenShift platform, or application?
2.  **Check Events:** `oc get events -n <namespace> --sort-by='.lastTimestamp'`
3.  **Check Logs:** Pod logs, node logs, operator logs, infrastructure logs.
4.  **Check Resource Status:** `oc describe <resource> <name> -n <namespace>`
5.  **Reproduce the Issue:** If possible, in a non-production environment.

### OpenShift Platform Issues

1.  **Cluster Operators (CO):**
    *   `oc get co`: Identify degraded operators.
    *   `oc describe co <operator-name>`: Get status and reasons.
    *   Check logs of the operator's pods (usually in `openshift-<operator-name>` namespace).
2.  **Nodes Not Ready:**
    *   `oc get nodes`: Check status.
    *   `oc describe node <node-name>`: Look at conditions, taints, capacity.
    *   **Debug Node:** `oc debug node/<node-name>` then `chroot /host`.
        *   Check `systemctl status kubelet`, `journalctl -u kubelet -f`.
        *   Check `systemctl status crio`, `journalctl -u crio -f`.
        *   Check network connectivity from the node (DNS, API server, other nodes).
        *   Check disk space (`df -h`), memory (`free -m`), CPU load (`top`).
3.  **API Server Issues:**
    *   Check `oc get pods -n openshift-kube-apiserver`.
    *   Logs: `oc logs -n openshift-kube-apiserver -l app=openshift-kube-apiserver`.
    *   Check etcd health (as shown previously).
4.  **DNS Issues:**
    *   Check `oc get pods -n openshift-dns -l dns.operator.openshift.io/daemonset-dns=default`.
    *   Test DNS resolution from a pod: `oc run -it --rm dns-test --image=busybox -- nslookup <service-name>.<namespace>.svc.cluster.local`.
    *   Test external DNS from a pod.
5.  **Ingress/Routing Issues:**
    *   `oc get routes -n <namespace>`, `oc describe route <route-name> -n <namespace>`.
    *   Check Ingress Controller pods: `oc get pods -n openshift-ingress -l ingresscontroller.operator.openshift.io/deployment-ingresscontroller=default`.
    *   Logs of Ingress Controller pods.

### Storage Issues (Pure Storage CSI)
1.  **PVC Stuck in Pending:**
    *   `oc describe pvc <pvc-name> -n <namespace>`: Look for events.
    *   No suitable PV found? StorageClass exists? Provisioner correct?
    *   Pure CSI controller logs: `oc logs -n pure-storage-csi-driver -l app=pure-csi-controller -c pure-csi-driver` (check container name).
    *   Pure CSI node plugin logs (on the node where pod is scheduled): `oc logs -n pure-storage-csi-driver <csi-node-pod-name> -c pure-csi-driver`.
2.  **Volume Attachment/Mount Issues:**
    *   `oc describe pod <pod-name> -n <namespace>`: Look for mount errors.
    *   Kubelet logs on the node: `journalctl -u kubelet` (filter for pod/volume).
    *   Pure CSI node plugin logs.
    *   Connectivity from node to Pure Storage (FC zoning, iSCSI ping/login).
        *   FC: `systool -c fc_host -v` (on node via `oc debug node/...`).
        *   iSCSI: `iscsiadm -m session` (on node via `oc debug node/...`).
3.  **Performance Issues:**
    *   Check latency/IOPS/bandwidth on Pure FlashArray and from within the pod (e.g., `fio`).
    *   Network latency between node and array.
    *   Ensure multipathing is active.

### Network Troubleshooting Tools
*   **`oc exec -it <pod> ...`:** Run network tools from within a pod.
*   **`oc debug node/<node-name>`:** Access node's network namespace.
*   **`tcpdump` / `tshark`:** Capture traffic on nodes or within pods.
*   **`mtr` / `traceroute`:** Trace network paths.
*   **`ip` / `ss` / `netstat`:** Inspect network interfaces, connections.
*   **OVN-Kubernetes tools:** `ovn-nbctl`, `ovn-sbctl` (run from `ovnkube-master` pods or control plane nodes) for advanced overlay debugging. Example: `oc rsh -n openshift-ovn-kubernetes ovnkube-master-<xxxxx> ovn-nbctl show`.

### `oc adm must-gather`
The most comprehensive tool for collecting diagnostic information for Red Hat Support.
```bash
[bastion]$ oc adm must-gather --image=<registry>/<must-gather-image>:<tag> --dest-dir=./must-gather-output
# Example:
[bastion]$ oc adm must-gather --dest-dir=./must-gather-output
# This creates a .tar.gz file and a directory with logs and resource definitions.
```
*   Add specific gatherers for components like etcd, network, virtualization, storage (if available).
    `[bastion]$ oc adm must-gather --image=registry.redhat.io/openshift4/ose-csi-driver-must-gather-rhel8 --dest-dir=csi-must-gather`

## Advanced Use Cases

FlashStack with OpenShift can support a variety of advanced workloads:

1.  **AI/ML Workloads:**
    *   **GPU Integration:** Utilize NVIDIA GPUs on UCS C-Series servers.
    *   Install NVIDIA GPU Operator on OpenShift for driver deployment and device plugin.
    *   Use OpenShift Virtualization for GPU passthrough to VMs.
    *   Leverage high-throughput, low-latency Pure Storage for training data and model storage.
    *   OpenDataHub.io or Kubeflow for MLOps pipelines.
2.  **High-Performance Databases:**
    *   Deploy stateful databases (PostgreSQL, MySQL, MongoDB, etc.) using Operators.
    *   Utilize Pure Storage CSI for persistent, high-performance block storage.
    *   Apply performance tuning techniques (HugePages, CPU/Memory manager, dedicated nodes).
    *   Pure Storage snapshots for rapid database cloning/refresh for dev/test.
3.  **Multi-Cluster Management (Red Hat Advanced Cluster Management - ACM):**
    *   Deploy ACM to manage multiple OpenShift clusters (on-prem, cloud) from a single console.
    *   Application lifecycle management, policy enforcement, observability across clusters.
    *   Useful for DR scenarios or distributed environments.
4.  **Serverless Applications (OpenShift Serverless - Knative):**
    *   Install OpenShift Serverless Operator.
    *   Deploy applications as Knative Services for scale-to-zero and event-driven capabilities.
5.  **Service Mesh (OpenShift Service Mesh - Istio):**
    *   Install OpenShift Service Mesh Operator.
    *   Manage microservice communication with traffic management, security (mTLS), and observability.
6.  **Edge Computing:**
    *   Deploy smaller OpenShift footprints (e.g., 3-node compact clusters, Single Node OpenShift - SNO) on smaller FlashStack configurations or other UCS hardware at edge locations.
    *   Manage with ACM.
7.  **Big Data Analytics (Spark, Presto):**
    *   Run Spark jobs on OpenShift, potentially using S3-compatible storage on FlashBlade or object storage for data lakes, with FlashArray for hot data/shuffle.

## Reference Architecture

`[DIAGRAM: Consolidated Reference Architecture Diagram - A clear, detailed diagram showing the validated component layout, key network paths, and integration points. This should be the primary visual anchor for the CVD.]`

*   **Official CVD Document:** Link to the official Cisco Validated Design document for FlashStack with OpenShift (if this guide is supplementary or a deep-dive).
*   **Bill of Materials (BoM) Example:**
    *   Provide a sample BoM for a typical configuration (e.g., 3 control plane, 3 worker nodes).
    *   Include Cisco UCS servers (model, CPU, RAM, NICs, HBAs), Fabric Interconnects, Nexus/MDS switches, Pure Storage FlashArray model, Purity version, licenses (OCP, Intersight, etc.).
    *   *(This would be a detailed table in a real document).*

## Conclusion and Next Steps

This guide has provided a comprehensive walkthrough for implementing Red Hat OpenShift Container Platform on a FlashStack infrastructure. By following these validated steps and best practices, organizations can build a robust, scalable, and high-performance platform for their modern applications.

**Key Takeaways:**
*   Thorough planning and prerequisite validation are crucial for success.
*   Leverage automation (IPI, Intersight, Ansible, Terraform) to streamline deployment and management.
*   Integrate Pure Storage CSI for dynamic, feature-rich persistent storage.
*   Implement strong security measures across all layers.
*   Establish robust Day 2 operational practices for monitoring, maintenance, and BCDR.

**Next Steps:**
*   Consult official vendor documentation for the latest details and version-specific instructions.
*   Begin with a Proof of Concept (PoC) or development environment to validate configurations.
*   Engage with Cisco, Pure Storage, and Red Hat professional services or partners for complex deployments or specialized requirements.
*   Continuously review and optimize your OpenShift on FlashStack environment as your needs evolve.

## Glossary

*   **ACM:** Red Hat Advanced Cluster Management
*   **API:** Application Programming Interface
*   **BCDR:** Backup and Disaster Recovery
*   **BMC:** Baseboard Management Controller (e.g., Cisco CIMC)
*   **BoM:** Bill of Materials
*   **CDI:** Containerized Data Importer (for OpenShift Virtualization)
*   **CI/CD:** Continuous Integration / Continuous Delivery (or Deployment)
*   **CIMC:** Cisco Integrated Management Controller
*   **CSI:** Container Storage Interface
*   **CVO:** Cluster Version Operator (OpenShift)
*   **CVD:** Cisco Validated Design
*   **DHCP:** Dynamic Host Configuration Protocol
*   **DNS:** Domain Name System
*   **DPDK:** Data Plane Development Kit
*   **DR:** Disaster Recovery
*   **FC:** Fibre Channel
*   **FI:** Fabric Interconnect (Cisco UCS)
*   **HCL:** Hardware Compatibility List
*   **HCO:** HyperConverged Operator (for OpenShift Virtualization)
*   **IaC:** Infrastructure as Code
*   **IdP:** Identity Provider
*   **IOM:** I/O Module (Cisco UCS Chassis)
*   **IPMI:** Intelligent Platform Management Interface
*   **IPI:** Installer Provisioned Infrastructure (OpenShift)
*   **KubeVirt:** The upstream project for OpenShift Virtualization
*   **LB:** Load Balancer
*   **LLDP:** Link Layer Discovery Protocol
*   **MAC:** Media Access Control
*   **MCO:** Machine Config Operator (OpenShift)
*   **MDS:** Multilayer Director Switch (Cisco SAN switches)
*   **MFA:** Multi-Factor Authentication
*   **MTU:** Maximum Transmission Unit
*   **NAD:** NetworkAttachmentDefinition
*   **NDU:** Non-Disruptive Upgrade
*   **NFS:** Network File System
*   **NIC:** Network Interface Card
*   **NPIV:** N_Port ID Virtualization
*   **NTP:** Network Time Protocol
*   **NUMA:** Non-Uniform Memory Access
*   **OADP:** OpenShift API for Data Protection (Velero-based)
*   **OCP:** OpenShift Container Platform
*   **ODF:** OpenShift Data Foundation (formerly OpenShift Container Storage)
*   **OLM:** Operator Lifecycle Management (OpenShift)
*   **OOB:** Out-of-Band
*   **OVN:** Open Virtual Network
*   **PSA:** Pod Security Admission
*   **PV:** PersistentVolume
*   **PVC:** PersistentVolumeClaim
*   **PXE:** Preboot Execution Environment
*   **QoS:** Quality of Service
*   **RBAC:** Role-Based Access Control
*   **RHCOS:** Red Hat Enterprise Linux CoreOS
*   **RPO:** Recovery Point Objective
*   **RTO:** Recovery Time Objective
*   **S2I:** Source-to-Image (OpenShift build strategy)
*   **SAN:** Storage Area Network
*   **SCC:** Security Context Constraints (OpenShift)
*   **SDK:** Software Development Kit
*   **SIEM:** Security Information and Event Management
*   **SNO:** Single Node OpenShift
*   **SPT:** Service Profile Template (Cisco UCS)
*   **SR-IOV:** Single Root I/O Virtualization
*   **SSD:** Solid State Drive
*   **SSH:** Secure Shell
*   **ToR:** Top-of-Rack (switch)
*   **UCS:** Unified Computing System (Cisco)
*   **UPI:** User Provisioned Infrastructure (OpenShift)
*   **VIP:** Virtual IP Address
*   **VLAN:** Virtual Local Area Network
*   **VM:** Virtual Machine
*   **vNIC:** virtual Network Interface Card
*   **vHBA:** virtual Host Bus Adapter
*   **vPC:** virtual PortChannel (Cisco Nexus)
*   **VSAN:** Virtual Storage Area Network (Cisco MDS)
*   **WWN:** World Wide Name (for FC)
*   **WWPN:** World Wide Port Name

## Appendix (Optional - Placeholder)
*   Detailed Bill of Materials for different scale points.
*   Full sample `install-config.yaml` for a common scenario.
*   Complete `MachineConfig` examples for advanced node tuning.
*   Extensive troubleshooting command cheat sheet.
*   Network cabling diagrams.
