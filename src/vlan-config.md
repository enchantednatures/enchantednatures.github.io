# VLAN Configuration Guide

## Netgear JGS524e v2 Switch Configuration

### Initial Setup
1. Connect to switch web interface (default: 192.168.0.239)
2. Login with default credentials
3. Update firmware if needed
4. Change default admin password

### VLAN Creation

#### Step 1: Create VLANs
Navigate to **Switching → VLAN → 802.1Q → VLAN Configuration**

Create the following VLANs:
```
VLAN ID: 10, Name: Kubernetes
VLAN ID: 20, Name: Storage  
VLAN ID: 30, Name: Services
VLAN ID: 40, Name: IoT
VLAN ID: 99, Name: Guest
```

#### Step 2: Configure VLAN Membership
Navigate to **Switching → VLAN → 802.1Q → VLAN Membership**

**Port Configuration:**
```
Ports 1-4 (Leviathan): Tagged on VLANs 1,10,20,30
Ports 5-8 (SuperMicro): Tagged on VLANs 1,20,30
Ports 9-11 (RPi): Untagged on VLAN 40
Port 12 (Uplink): Tagged on all VLANs
Port 24 (Management): Untagged on VLAN 1
```

#### Step 3: Configure Port VLAN ID (PVID)
Navigate to **Switching → VLAN → 802.1Q → Port PVID Configuration**

```
Ports 1-8: PVID 1 (Management default)
Ports 9-11: PVID 40 (IoT devices)
Port 12: PVID 1 (Uplink)
Remaining ports: PVID 1 (Default)
```

## Router/Firewall Configuration

### VLAN Interfaces
Configure the following interfaces on your router:

```bash
# Management VLAN (existing)
interface vlan1
  ip address 192.168.1.1/24
  
# Kubernetes VLAN  
interface vlan10
  ip address 10.10.0.1/16
  
# Storage VLAN
interface vlan20
  ip address 10.20.0.1/24
  
# Services VLAN
interface vlan30
  ip address 10.30.0.1/24
  
# IoT VLAN
interface vlan40
  ip address 10.40.0.1/24
  
# Guest VLAN
interface vlan99
  ip address 10.99.0.1/24
```

### DHCP Configuration
Set up DHCP scopes for each VLAN:

```bash
# Kubernetes VLAN DHCP
dhcp pool kubernetes
  network 10.10.0.0/16
  default-router 10.10.0.1
  dns-server 10.10.0.1
  range 10.10.1.100 10.10.1.200

# Storage VLAN DHCP  
dhcp pool storage
  network 10.20.0.0/24
  default-router 10.20.0.1
  dns-server 10.20.0.1
  range 10.20.0.100 10.20.0.200

# Services VLAN DHCP
dhcp pool services
  network 10.30.0.0/24
  default-router 10.30.0.1
  dns-server 10.30.0.1
  range 10.30.0.100 10.30.0.200

# IoT VLAN DHCP
dhcp pool iot
  network 10.40.0.0/24
  default-router 10.40.0.1
  dns-server 10.40.0.1
  range 10.40.0.100 10.40.0.200

# Guest VLAN DHCP
dhcp pool guest
  network 10.99.0.0/24
  default-router 10.99.0.1
  dns-server 8.8.8.8
  range 10.99.0.100 10.99.0.200
```

## Proxmox VLAN Configuration

### Leviathan Network Setup
Configure network bridges for each VLAN:

```bash
# Edit /etc/network/interfaces
auto vmbr0
iface vmbr0 inet static
    address 192.168.1.10/24
    gateway 192.168.1.1
    bridge-ports eno1
    bridge-stp off
    bridge-fd 0

# Kubernetes VLAN bridge
auto vmbr10  
iface vmbr10 inet static
    address 10.10.0.10/16
    bridge-ports eno1.10
    bridge-stp off
    bridge-fd 0
    bridge-vlan-aware yes

# Storage VLAN bridge
auto vmbr20
iface vmbr20 inet static
    address 10.20.0.10/24
    bridge-ports eno1.20
    bridge-stp off
    bridge-fd 0

# Services VLAN bridge  
auto vmbr30
iface vmbr30 inet static
    address 10.30.0.10/24
    bridge-ports eno1.30
    bridge-stp off
    bridge-fd 0
```

### SuperMicro Nodes Network Setup
Configure each node with appropriate VLAN interfaces:

**Tower (Node 1) - Storage Focus:**
```bash
# Management interface
auto eno1
iface eno1 inet static
    address 192.168.1.11/24
    gateway 192.168.1.1

# Storage interface  
auto eno1.20
iface eno1.20 inet static
    address 10.20.0.11/24
```

**Melusine (Node 2) - Services:**
```bash
# Management interface
auto eno1
iface eno1 inet static
    address 192.168.1.12/24
    gateway 192.168.1.1

# Services interface
auto eno1.30  
iface eno1.30 inet static
    address 10.30.0.12/24
```

## Kubernetes Cluster Migration

### Talos Configuration Update
Update Talos configuration to use new VLAN network:

```yaml
# talos-config.yaml
machine:
  network:
    interfaces:
      - interface: eth0
        addresses:
          - 10.10.0.20/16
        routes:
          - network: 0.0.0.0/0
            gateway: 10.10.0.1

cluster:
  network:
    podSubnets:
      - 10.244.0.0/16
    serviceSubnets:
      - 10.96.0.0/12
```

### API Server Access
Ensure Kubernetes API server is accessible from management VLAN:

```bash
# Firewall rule to allow management access to K8s API
iptables -A FORWARD -s 192.168.1.0/24 -d 10.10.0.0/16 -p tcp --dport 6443 -j ACCEPT
```

## Firewall Rules

### Inter-VLAN Access Control
```bash
# Management to all VLANs (administrative access)
iptables -A FORWARD -s 192.168.1.0/24 -j ACCEPT

# Kubernetes to Storage (NFS, S3)
iptables -A FORWARD -s 10.10.0.0/16 -d 10.20.0.0/24 -p tcp --dport 2049 -j ACCEPT  # NFS
iptables -A FORWARD -s 10.10.0.0/16 -d 10.20.0.0/24 -p tcp --dport 9000 -j ACCEPT  # MinIO

# Services to Storage
iptables -A FORWARD -s 10.30.0.0/24 -d 10.20.0.0/24 -p tcp --dport 2049 -j ACCEPT  # NFS
iptables -A FORWARD -s 10.30.0.0/24 -d 10.20.0.0/24 -p tcp --dport 9000 -j ACCEPT  # MinIO

# Allow outbound internet for Kubernetes and Services
iptables -A FORWARD -s 10.10.0.0/16 -o wan0 -j ACCEPT
iptables -A FORWARD -s 10.30.0.0/24 -o wan0 -j ACCEPT

# IoT and Guest internet only
iptables -A FORWARD -s 10.40.0.0/24 -o wan0 -j ACCEPT
iptables -A FORWARD -s 10.99.0.0/24 -o wan0 -j ACCEPT

# Deny all other inter-VLAN traffic
iptables -A FORWARD -j DROP
```

## Testing & Validation

### Connectivity Tests
```bash
# Test VLAN connectivity
ping 10.10.0.1  # Kubernetes gateway
ping 10.20.0.1  # Storage gateway  
ping 10.30.0.1  # Services gateway

# Test inter-VLAN access
# From management VLAN, test K8s API access
curl -k https://10.10.0.20:6443

# Test storage access from K8s VLAN
showmount -e 10.20.0.11  # NFS exports
```

### VLAN Verification
```bash
# Verify VLAN membership on switch
show vlan brief

# Check port VLAN assignments  
show interfaces switchport

# Verify trunk port configuration
show interfaces trunk
```

## Troubleshooting

### Common Issues
1. **No inter-VLAN connectivity**: Check router VLAN interfaces and routing
2. **DHCP not working**: Verify DHCP relay configuration
3. **Trunk ports not passing traffic**: Check VLAN membership and tagging
4. **API server unreachable**: Verify firewall rules for management access

### Diagnostic Commands
```bash
# Switch diagnostics
show vlan
show mac address-table
show interfaces status

# Linux VLAN diagnostics  
ip link show
ip addr show
cat /proc/net/vlan/config
```

## Migration Checklist

- [ ] Configure VLANs on switch
- [ ] Set up router VLAN interfaces and DHCP
- [ ] Configure Proxmox host networking
- [ ] Update VM network configurations
- [ ] Migrate Kubernetes cluster to new VLAN
- [ ] Update storage service network configs
- [ ] Configure firewall rules
- [ ] Test connectivity between VLANs
- [ ] Update documentation with final IP assignments
- [ ] Monitor network performance post-migration