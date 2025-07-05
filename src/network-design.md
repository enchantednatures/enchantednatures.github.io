# Network Design & VLAN Strategy

## Current Network Challenges
- Running out of IP addresses on home network
- Need better network segmentation for security
- Kubernetes cluster requires isolated network with management access

## Switch Capabilities
**Netgear JGS524e v2 Plus Switch**
- 24 Gigabit Ethernet ports
- 802.1Q VLAN support
- Port-based and tagged VLAN configuration
- Web-based management interface

## VLAN Segmentation Strategy

### VLAN Design Overview

| VLAN ID | Name | Purpose | Subnet | Notes |
|---------|------|---------|---------|-------|
| 1 | Default/Management | Proxmox hosts, switch management | 192.168.1.0/24 | Native VLAN |
| 10 | Kubernetes | K8s cluster nodes and services | 10.10.0.0/16 | Isolated cluster network |
| 20 | Storage | NFS, MinIO, storage traffic | 10.20.0.0/24 | High-bandwidth storage |
| 30 | Services | General services, VMs | 10.30.0.0/24 | Application workloads |
| 40 | IoT/Devices | Future IoT devices, RPi cluster | 10.40.0.0/24 | Restricted internet access |
| 99 | Guest | Guest network isolation | 10.99.0.0/24 | Internet only, no LAN access |

### VLAN Access Requirements

#### Management VLAN (1) - 192.168.1.0/24
**Purpose**: Infrastructure management and inter-VLAN routing
**Hosts**:
- Leviathan Proxmox management interface
- Tower Unraid management interface  
- Melusine Proxmox management interface
- Node 3/4 Proxmox management interfaces
- Switch management interface
- Router/firewall management

**Access Rules**:
- Full access to all VLANs for management
- Tailscale endpoints terminate here
- SSH/HTTPS management protocols

#### Kubernetes VLAN (10) - 10.10.0.0/16
**Purpose**: Kubernetes cluster isolation with large address space
**Hosts**:
- Talos VMs on Leviathan
- Kubernetes API server (accessible from Management VLAN)
- Pod networks (CNI-managed subnets)
- LoadBalancer services

**Access Rules**:
- Management VLAN can access K8s API server (port 6443)
- Storage VLAN access for persistent volumes
- Outbound internet access
- No direct access from other VLANs except management

#### Storage VLAN (20) - 10.20.0.0/24
**Purpose**: High-performance storage traffic isolation
**Hosts**:
- Tower NFS services
- Tower MinIO S3 services
- Storage-specific interfaces on Proxmox hosts
- Backup services

**Access Rules**:
- Kubernetes VLAN access for PV storage
- Services VLAN access for VM storage
- Management VLAN access for administration
- High QoS priority for storage traffic

#### Services VLAN (30) - 10.30.0.0/24
**Purpose**: General application and VM workloads
**Hosts**:
- seko Arch VM
- Future application VMs
- Development environments
- Non-critical services

**Access Rules**:
- Storage VLAN access for data
- Management VLAN access for administration
- Outbound internet access
- Limited inter-service communication

## Network Topology

```
Internet
    |
[Router/Firewall] - VLAN routing & firewall rules
    |
[Netgear JGS524e v2] - 802.1Q VLAN switch
    |
    ├── Port 1-4: Leviathan (trunk: 1,10,20,30)
    ├── Port 5-8: SuperMicro FatTwin (trunk: 1,20,30)
    │   ├── Tower: VLAN 1,20 (management + storage)
    │   ├── Melusine: VLAN 1,30 (management + services)
    │   ├── Node 3: VLAN 1,30 (management + services)  
    │   └── Node 4: VLAN 1,30 (management + services)
    ├── Port 9-11: Raspberry Pi (access: VLAN 40)
    ├── Port 12: Uplink to Router (trunk: all VLANs)
    └── Port 13-24: Available for expansion
```

## Implementation Plan

### Phase 1: Infrastructure Preparation
1. **Router Configuration**
   - Configure VLAN interfaces and routing
   - Set up firewall rules between VLANs
   - Configure DHCP scopes for each VLAN

2. **Switch Configuration**
   - Create VLANs 10, 20, 30, 40, 99
   - Configure trunk ports for servers
   - Set up access ports for devices

### Phase 2: Server Network Configuration
1. **Leviathan (Proxmox)**
   - Configure VLAN interfaces on Proxmox host
   - Bridge configuration for VM networks
   - Migrate Talos VMs to VLAN 10

2. **SuperMicro Nodes**
   - Configure management interfaces on VLAN 1
   - Set up storage interfaces on VLAN 20
   - Configure service interfaces on VLAN 30

### Phase 3: Service Migration
1. **Kubernetes Cluster**
   - Migrate to VLAN 10 network
   - Update API server accessibility
   - Reconfigure storage connections

2. **Storage Services**
   - Move NFS/MinIO to VLAN 20
   - Update client configurations
   - Test performance improvements

## Security Considerations

### Inter-VLAN Firewall Rules
```
Management (1) → All VLANs: Allow (administrative access)
Kubernetes (10) → Storage (20): Allow NFS/S3 ports
Kubernetes (10) → Internet: Allow outbound
Services (30) → Storage (20): Allow NFS/S3 ports  
Services (30) → Internet: Allow outbound
IoT (40) → Internet: Allow outbound only
Guest (99) → Internet: Allow outbound only
All other inter-VLAN: Deny
```

### Port Security
- Enable port security on access ports
- MAC address learning limits
- DHCP snooping where supported
- Storm control for broadcast traffic

## Monitoring & Troubleshooting

### Network Monitoring
- SNMP monitoring of switch ports
- VLAN traffic analysis
- Inter-VLAN routing metrics
- Bandwidth utilization per VLAN

### Troubleshooting Tools
- VLAN membership verification
- Trunk port configuration validation
- Inter-VLAN connectivity testing
- Performance baseline measurements

## Future Expansion

### Additional VLANs
- **VLAN 50**: DMZ for public services
- **VLAN 60**: Backup network isolation
- **VLAN 70**: Lab/testing environment

### Advanced Features
- VLAN QoS prioritization
- Link aggregation for high-bandwidth hosts
- VLAN-aware monitoring and alerting
- Automated VLAN provisioning