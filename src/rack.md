# Homelab Documentation

## Overview

This homelab is managed through Tailscale for secure remote access. The setup consists of a 20U rack with various servers and networking equipment optimized for virtualization, storage, and Kubernetes workloads.

## Physical Infrastructure

### Rack Setup
- **Size**: 20U rack
- **Management**: All hosts managed through Tailscale
- **Power**: Standard rack power distribution

## Network Equipment

### Primary Switch
- **Model**: Netgear JGS524e v2
- **Type**: 24-port Gigabit Ethernet switch
- **Purpose**: Main network connectivity for all rack equipment

## Servers

### Leviathan (Threadripper Server)
- **Hardware**: AMD Threadripper-based server
- **Hypervisor**: Proxmox VE
- **Primary Purpose**: Kubernetes cluster hosting via Talos VMs
- **Status**: Active

#### Virtual Machines
- **seko**: Arch Linux VM running on Leviathan

### SuperMicro FatTwin Server
**Model**: SuperMicro F627R3-R72B+ 4U FatTwin Server 4-Node X9DRFR  
**CPU Sockets**: 8x LGA2011 (2 per node)  
**CPU Support**: Intel E5-2600v2 series  

#### Node Configuration

##### Tower (Node 1)
- **Hostname**: tower
- **OS**: Unraid
- **Purpose**: NFS storage server
- **Services**: 
  - NFS storage for cluster
  - MinIO Docker container for S3-compatible storage
- **Status**: Active

##### Melusine (Node 2) 
- **Hostname**: melusine
- **OS**: Proxmox VE (installed but not running)
- **Purpose**: Virtualization host (planned)
- **Status**: Installed, not active

##### Node 3
- **Hostname**: node3
- **OS**: Proxmox VE (planned)
- **Status**: Hardware issue - CPU needs reseating
- **Notes**: Will install Proxmox once hardware is fixed

##### Node 4
- **Hostname**: node4
- **OS**: Proxmox VE (planned)
- **Status**: Not yet configured

## Storage Configuration

### Direct HDD Passthrough
For optimal storage performance on the SuperMicro nodes, direct HDD passthrough is configured via the SuperMicro RAID controller. This allows VMs to have direct access to physical drives without virtualization overhead.

**Implementation**: Following specialized configuration for SuperMicro hardware to bypass RAID virtualization layer.

## Raspberry Pi Cluster
- **Count**: 3 units
- **Status**: Currently powered off
- **Purpose**: Available for lightweight services or testing

## Network Architecture

### Current Network Challenges
- IP address exhaustion on home network
- Need for better network segmentation
- Kubernetes cluster isolation requirements

### Planned VLAN Segmentation
See [Network Design & VLAN Strategy](./network-design.md) for detailed network planning.

**Planned VLANs:**
- **VLAN 1**: Management (Proxmox, switch management)
- **VLAN 10**: Kubernetes cluster (10.10.0.0/16)
- **VLAN 20**: Storage traffic (NFS, MinIO)
- **VLAN 30**: General services and VMs
- **VLAN 40**: IoT devices and RPi cluster
- **VLAN 99**: Guest network isolation

### Management Network
All management interfaces accessible via Tailscale on VLAN 1:
- Leviathan Proxmox GUI
- Tower Unraid GUI  
- Melusine Proxmox GUI (when active)
- Node 3 Proxmox GUI (planned)
- Node 4 Proxmox GUI (planned)

### Service Networks
- **NFS**: Tower on dedicated storage VLAN
- **S3 Storage**: MinIO on storage VLAN
- **Kubernetes**: Isolated on dedicated VLAN with API access from management

## Current Workloads

### Active Services
1. **Kubernetes Cluster**: Running on Talos VMs (Leviathan)
2. **NFS Storage**: Unraid on Tower
3. **S3-Compatible Storage**: MinIO on Tower
4. **Development Environment**: Arch VM (seko) on Leviathan

### Planned Expansions
1. Additional Proxmox nodes (Melusine, Node 3, Node 4)
2. Expanded virtualization capacity
3. Potential Raspberry Pi cluster activation

## Maintenance Notes

### Pending Tasks
- [ ] Reseat CPU on Node 3
- [ ] Complete Node 4 initial setup
- [ ] Activate Melusine Proxmox installation
- [ ] Consider Raspberry Pi cluster utilization

### Hardware Considerations
- SuperMicro FatTwin provides excellent density for virtualization
- Direct storage passthrough optimizes I/O performance
- Tailscale provides secure remote management without VPN complexity
