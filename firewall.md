# Proxmox Firewall
This file provides details about Proxmox Firewall and the default configuration applied.


## Documentation
- https://pve.proxmox.com/wiki/Firewall
- https://www.youtube.com/watch?v=DNsLLrCgK0U&list=PLT98CRl2KxKHnlbYhtABg6cF50bYa8Ulo&index=13

### Tips
- Firewall rules at VM, Container, or Node level will only work after enabling the firewall at the Datacenter level.
- Be sure to create a firewall rule allowing access to the Proxmox GUI (port 8006) before enabling the Datacenter firewall otherwise, you willl lock yourself out. If this happens, you must SSH directly into a Proxmox server and manually disable the firewall by editing `/etc/pve/firewall/cluster.fw`.
- Proxmox Firewall has no effect on the connections targeting the public IP provided by NordVPN.

## Default Configuration
### Datacenter Level
Input Policy **DROP**  
Output Policy **ACCEPT**  
Forward Policy **ACCEPT**  

*Rules*  
in ACCEPT vmbr0 tcp 8006 nolog

### Node Level

### VM/Container Level

**vpn-server**  
Input Policy **DROP**  
Output Policy **DROP**  

*Rules* 
out ACCEPT tcp 6443 nolog - This entry allows reaching the Kubernetes API Server. 
out ACCEPT HTTPS nolog   
out ACCEPT HTTP nolog   