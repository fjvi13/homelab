# VM Inventory

### PVE 1
| Type        | ID  | Name   | NICs                     | Comment                           |
|-------------|-----|--------|--------------------------|-----------------------------------|
| VM          | 200 | K3s-cp | net0: vmbr0, VLAN Tag 10 | K3s Master Node Central Cluster   |
| VM          | 201 | K3s-w1 | net0: vmbr0, VLAN Tag 10 | K3s Worker Node 1 Central Cluster |

### PVE 2
| Type        | ID  | Name                    | NICs                                           | Comment                           |
|-------------|-----|-------------------------|------------------------------------------------|-----------------------------------|
| LXC         | 110 | webserver               | net0: vmbr0                                    |                                   |
| VM          | 101 | k3s-worker-0            | net0: vmbr0, VLAN Tag 10                       | K3s Master Node Remote Cluster    |
| VM          | 102 | k3s-master-0            | net0: vmbr0, VLAN Tag 10                       | K3s Worker Node 1 Remote Cluster  |
| VM          | 130 | api-project             | net0: vmbr0, VLAN Tag 10                       |                                   |
| VM          | 202 | K3s-w2                  | net0: vmbr0, VLAN Tag 10                       | K3s Worker Node 2 Central Cluster |
| VM          | 300 | vpn-server              | net0: vmbr0, VLAN Tag 20                       | NordVPN Meshnet Bastion Host      |
| VM          | 310 | wg-server               | net0: vmbr0, VLAN Tag 20                       | Wireguard Bastion Host            |
| VM          | 915 | pfsense                 | net0: vmbr0, VLAN Tag 1 <br> net1: vmbr0, VLAN Tag 10 <br> net2: vmbr0, Tag VLAN 20 <br> net3: vmbr0, Tag VLAN 2 | pfSense Router |
| VM Template | 808 | ubuntu-2404-template-v5 | net0: vmbr0, VLAN Tag 10                       |                                   |