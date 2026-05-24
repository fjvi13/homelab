# Homelab Documentation

Personal homelab built around two Proxmox VE nodes, a VLAN-segmented network managed by pfSense, and a K3s Kubernetes cluster — with secure remote access via WireGuard and NordVPN Meshnet. This repository captures every piece of it: hardware, network design, VM inventory, and setup procedures.



## Quick Links

- 🔧 [Hardware](./docs/hardware.md)
- 🌐 [Networking](./docs/networking.md)
- 💻 [VM Inventory](./docs/vm-inventory.md)
- 📘 [Tutorials](./tutorials/)

## Structure

```
homelab-documentation/
├── docs/
│   ├── hardware.md           # Hardware specs, diagrams, and budget
│   ├── networking.md         # Network topology, VLANs, and device configurations
│   ├── vm-inventory.md       # Proxmox VM and LXC inventory
│   ├── wg-vpn-to-homelab.md  # WireGuard VPN setup guide
│   ├── nordvpn-to-homelab.md # NordVPN Meshnet access guide
│   ├── faq.md                # Frequently asked questions
│   ├── pending_tasks.md      # Ongoing and planned work
│   └── assets/
│       └── images/           # Diagrams and screenshots
├── tutorials/
│   ├── k3s.md                # K3s cluster setup
│   └── vms.md                # VM provisioning
└── README.md
```
