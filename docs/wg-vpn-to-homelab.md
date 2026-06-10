**Firewall NAT**
| Interface | Protocol | Dest. Address | Dest. Ports | NAT IP   |	NAT Ports | Description   |
|-----------|----------|---------------|-------------|----------|-----------|---------------|
| WAN       | UPD      | 192.168.1.11  | 51820       | 10.0.0.2 | 51820     | wg-server nat |


sudo /opt/homebrew/bin/bash wg-quick up wg0
sudo /opt/homebrew/bin/bash wg-quick down wg0