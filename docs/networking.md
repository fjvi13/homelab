# Networking
Overview of the homelab network topology, VLANs, and device configurations.

<img src="./assets/images/homelab-L3-arquitecture-diagram.png" width="750" alt="L3 Architecture Diagram">

*L3 Architecture Diagram*

## Network List

### VLAN 1 (Default): Home Network
*192.168.1.0/24*  

This is the home LAN network used by guests for internet access. It is also accessible via Wi-Fi through the *ISP Home Gateway AP-Router*.

**Relevant IPs**
- 192.168.1.1: ISP Home Gateway AP-Router
  - Internet Default Gateway
  - DHCP Server
  - ISP Home Gateway MGMT UI

- 192.168.1.11: pfSense Router 
  - Gateway to the rest of the Homelab networks.

### VLAN 2: MGMT Network
*172.16.1.0/24* 

This is the Homelab MGMT network.  

**Relevant IPs**
- 172.16.1.1: pfSense Router
  - MGMT Network Default Gateway
  - DHCP Server
  - PfSense MGMT UI
- 172.16.1.2: tp-link Switch MGMT UI
- 172.16.1.4: PVE1 MGMT UI
- 172.16.1.5: PVE2 MGMT UI

### VLAN 10: Lab Network
*10.0.1.0/24* 

It is the main Homelab network where all Proxmox VMs and LXCs are deployed by default.  

**Relevant IPs**
- 10.0.1.1: pfSense Router
  - Lab Network Default Gateway
  - DHCP Server

- 10.0.1.5 | 10.0.1.4 | 10.0.1.6: K3s Central Cluster Nodes (Master | Worker 1 | Worker 2)
- 10.0.1.15 | 10.0.1.16: K3s Remote Cluster Nodes (Master | Worker 1)

### VLAN 20: DMZ Network
*10.0.0.0/28*  

Demilitarized Zone Network hosting the VPN bastion servers, which provide secure access to services in the Lab network. 

**Relevant IPs**
- 10.0.0.1: pfSense Router
  - DMZ Network Default Gateway
  - DHCP Server

- 10.0.0.2: WG Server
  - Wireguard Bastion Host

- 10.0.0.3: NordVPN Server
  - NordVPN Meshnet Bastion Host

### VLAN 30: IoT Network 
> [!WARNING]
> Work in progress

## Configuration
<img src="./assets/images/homelab-L2-arquitecture-diagram.png" width="750" alt="L2 Architecture Diagram">

*L2 Architecture Diagram with some relevant VMs*

### TP-Link Switch Configuration

**IP Address Setting**  
TP-Link Switch MGMT Portal IP Configuration

| DHCP Setting | IP Address    |
|--------------|---------------|
| Disabled     | 172.16.1.2/24 |

**802.1Q VLAN**
| VLAN ID | VLAN Name | Untagged Ports | Tagged Ports |
|---------|-----------|----------------|--------------|
| 1       | Default   | 1              | 2-3          |
| 2       | MGMT      | 8              | 2-3          |
| 10      | Lab       |                | 2-3          |
| 20      | DMZ       |                | 2-3          |
| 30      | IoT       | WIP                           |


**802.1Q PVID Setting**
| Port    | PVID |
|---------|------|
| Port 1  | 1    |
| Port 2  | 1    |
| Port 3  | 1    |
| Port 4  | 1    |
| Port 5  | 1    |
| Port 6  | 1    |
| Port 7  | 1    |
| Port 8  | 2    |

### PVE 1 - Network Configuration
<img src="./assets/images/pve1-network-config.png" width="750" alt="PVE 1 Network Configuration">

### PVE 2 - Network Configuration
<img src="./assets/images/pve2-network-config.png" width="750" alt="PVE 2 Network Configuration">

> [!IMPORTANT]
> By default, use the web UI to configure Proxmox networking. If you lose access to it, SSH into the host and manually edit the `/etc/network/interfaces` file. Then, reload the host for the changes to take affect.

### ISP Home Gateway AP-Router Configuration
**Interfaces**
| Name      | Static IPv4 Address |
|-----------|---------------------|
| WAN       | -                   | 
| LAN/WLAN  | 192.168.1.1/24      |

**DHCP Server Address Pool**
| Interface | Subnet         | Address Pool Range         |
|-----------|----------------|----------------------------|
| LAN       | 192.168.1.0/24 | 192.168.1.11-192.168.1.254 |

**DHCP Server Static Mappings**
| IP Address    | Name           |
|---------------|----------------|
| 192.168.1.11  | pfsense-router |
| 192.168.1.22  | admin-pc       |

### pfSense Configuration
**Interfaces**
| Name | Config Type | Static IPv4 Address |
|------|-------------|---------------------|
| HOME | DHCP        | -                   |
| MGMT | Static IPv4 | 172.16.1.1/24       |
| LAB  | Static IPv4 | 10.0.1.1/24         |
| DMZ  | Static IPv4 | 10.0.0.1/28         |
| IoT  | WIP                               |

**DHCP Server Address Pool**
| Interface | Subnet        | Address Pool Range       |
|-----------|---------------|--------------------------|
| MGMT      | 172.16.1.0/24 | 172.16.1.20-172.16.1.254 |
| DMZ       | 10.0.0.0/28   | 10.0.0.8-10.0.0.14       |
| LAB       | 10.0.1.0/24   | 10.0.1.20-10.0.1.254     |

**DHCP Server Static Mappings**
| IP Address | Hostname     | Description                          |
|------------|--------------|--------------------------------------|
| 10.0.0.2   | wg-server    | Wireguard Bastion Host               |
| 10.0.0.3   | vpn-server   | NordVPN Meshnet Bastion Host         |
| 10.0.1.15  | k3s-master-0 | K3s Master Node Remote Cluster       |
| 10.0.1.16  | k3s-worker-0 | K3s Worker Node 1 Remote Cluster     |
| 10.0.1.5   | k3s-cp       | K3s Master Node Central Cluster      |
| 10.0.1.4   | k3s-w1       | K3s Worker Node 1 Central Cluster    |
| 10.0.1.6   | k3s-w2       | K3s Worker Node 2 Central Cluster    |

**Firewall Rules**
> [!IMPORTANT]
> 1. **Rules are evaluated top-to-bottom.** First match wins.
> 2. **pfSense can only filter traffic that crosses a subnet boundary.** Hosts on the same VLAN/subnet talk to each other directly at Layer 2 (ARP + MAC-to-MAC through the switch) — those packets never reach pfSense, so no firewall rule can match them. To restrict intra-subnet communication, use host-based firewalls (`ufw`, `nftables`) on the endpoints, or split the hosts into separate VLANs so the traffic has to be routed.

***HOME Interface***

| # | Action | Protocol | Source       | Destination          | Port        | Description                                                   |
|---|--------|----------|--------------|----------------------|-------------|---------------------------------------------------------------|
| 1 | Pass   | TCP      | HOME subnets | MGMT address         | 443 (HTTPS) | Allow UI on MGMT IP (172.16.1.1) only                         |
| 2 | Block  | TCP      | any          | This Firewall (self) | 443 (HTTPS) | Block pfSense admin plane — UI reserved for MGMT (172.16.1.1) |
| 3 | Pass   | any      | `Admin_IP`   | any                  | any         | All permissions for Admin IP                                  |
| 4 | Pass   | UDP      | any          | 10.0.0.2             | 51820       | NAT wg-server                                                 |

> [!NOTE]
> `Admin_IP` is a pfSense alias set to `192.168.1.22` — the admin workstation on the HOME network.

***MGMT Interface***

| # | Action | Protocol  | Source | Destination          | Port         | Description                                    |
|---|--------|-----------|--------|----------------------|--------------|------------------------------------------------|
| 1 | Pass   | TCP       | any    | MGMT address         | 443 (HTTPS)  | Anti-Lockout Rule                              |
| 2 | Pass   | ICMP any  | any    | any                  | any          | Allow ping & traceroute for diagnostics        |
| 3 | Pass   | UDP       | any    | This Firewall (self) | 53 (DNS)     | Allow DNS resolution through pfSense           |
| 4 | Pass   | UDP       | any    | any                  | 123 (NTP)    | Allow NTP for time sync                        |
| 5 | Pass   | any       | any    | LAB subnets          | any          | Manage LAB Network                             |
| 6 | Pass   | any       | any    | DMZ subnets          | any          | Manage DMZ Network                             |
| 7 | Pass   | TCP       | any    | any                  | `HTTP_HTTPS` | Internet egress, HTTPS/HTTP only *(disabled)*  |

> [!NOTE]
> 1. Rule 1 is the **manual anti-lockout rule** for the MGMT interface. Without it, applying a deny-all on this interface would lock the admin out of the pfSense UI.
> 2. Rules 5 and 6 give MGMT clients full administrative reach into LAB and DMZ.
> 3. Rule 7 is disabled — MGMT hosts deliberately have **no internet access**. If a Proxmox/pfSense update needs to fetch something, enable this rule temporarily.

***LAB Interface***

| # | Action | Protocol | Source        | Destination          | Port         | Description                                                   |
|---|--------|----------|---------------|----------------------|--------------|---------------------------------------------------------------|
| 1 | Pass   | UDP      | any           | This Firewall (self) | 53 (DNS)     | Allow DNS resolution through pfSense                          |
| 2 | Pass   | UDP      | any           | any                  | 123 (NTP)    | Allow NTP for time sync                                       |
| 3 | Block  | any      | any           | `_private4_`         | any          | Block RFC1918 destinations (10/8, 172.16/12, 192.168/16)      |
| 4 | Block  | TCP      | any           | This Firewall (self) | 443 (HTTPS)  | Block pfSense admin plane — UI reserved for MGMT (172.16.1.1) |
| 5 | Pass   | TCP      | any           | any                  | `HTTP_HTTPS` | Internet egress, HTTPS/HTTP only                              |
| 6 | Pass   | any      | LAB subnets   | any                  | any          | Default allow LAN to any rule *(disabled)*                    |

> [!NOTE]
> 1. `_private4_` is a pfSense built-in alias matching all RFC1918 private IPv4 ranges.
> 2. `HTTP_HTTPS` is a custom port alias grouping TCP 80 and 443.

***DMZ Interface***

| # | Action | Protocol | Source     | Destination          | Port         | Description                                                   |
|---|--------|----------|------------|----------------------|--------------|---------------------------------------------------------------|
| 1 | Pass   | UDP      | any        | This Firewall (self) | 53 (DNS)     | Allow DNS resolution through pfSense                          |
| 2 | Pass   | UDP      | any        | any                  | 123 (NTP)    | Allow NTP for time sync                                       |
| 3 | Pass   | TCP      | `10.0.0.3` | MGMT address         | 443 (HTTPS)  | Allow UI on MGMT IP (172.16.1.1) for NordVPN Bastion Admin    |
| 4 | Block  | TCP      | any        | This Firewall (self) | 443 (HTTPS)  | Block pfSense admin plane — UI reserved for MGMT (172.16.1.1) |
| 5 | Pass   | TCP      | any        | any                  | `LAB_SVCS`   | Bastion → LAB on broker services (SSH, HTTPS, K3s API)        |
| 6 | Pass   | any      | `10.0.0.3` | any                  | any          | Allow all from NordVPN Bastion Admin                          |
| 7 | Block  | any      | any        | `_private4_`         | any          | Block RFC1918 destinations (10/8, 172.16/12, 192.168/16)      |
| 8 | Pass   | TCP      | any        | any                  | `HTTP_HTTPS` | Internet egress, HTTPS/HTTP only                              |
| 9 | Pass   | any      | any        | any                  | any          | Default allow DMZ to any rule *(disabled)*                    |

> [!NOTE]
> 1. `LAB_SVCS` is a custom port alias grouping the broker-service ports the bastions need into LAB (SSH 22, HTTPS 443, K3s API 6443).
> 2. `10.0.0.3` is the NordVPN Meshnet bastion — rules 3 and 6 give it elevated access for admin tasks. The WireGuard bastion (`10.0.0.2`) is intentionally **not** included; only the broker-services rule (5) applies to it.
> 3. The rationale behind these rules is developed in detail in [wg-vpn-to-homelab.md](./wg-vpn-to-homelab.md) and [nordvpn-to-homelab.md](./nordvpn-to-homelab.md).

**DNS Server**

pfSense's built-in DNS Resolver is the authoritative source for internal homelab names.

***Host Overrides***

| Host       | Parent Domain | IP Address | Description           |
|------------|---------------|------------|-----------------------|
| argocd     | lab           | 10.0.1.5   | ArgoCD Service        |
| compactor  | lab           | 10.0.1.5   | Thanos Compactor GUI  |
| grafana    | lab           | 10.0.1.5   | Grafana Service       |
| kibana     | lab           | 10.0.1.5   | Kibana GUI            |
| python     | lab           | 10.0.1.15  | Python App            |
| query      | lab           | 10.0.1.5   | Thanos Querier GUI    |


### VMs Network Configuration
See the full [VM inventory](./vm-inventory.md).
