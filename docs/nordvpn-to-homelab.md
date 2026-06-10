# VPN to Homelab - NordVPN Meshnet

This is the **admin bastion**: the path I use to reach the homelab from outside the LAN with **full access** to every internal resource. Once the NordVPN Meshnet overlay is up, I SSH into the bastion VM and from there I can reach any host on any homelab network.

This is deliberately *not* the path used by other clients, collaborators or services — for that, see [wg-vpn-to-homelab.md](./wg-vpn-to-homelab.md). NordVPN Meshnet is admin-only and intentionally over-privileged in the DMZ firewall rules (see [networking.md](./networking.md)).

VPN: **NordVPN Meshnet**  
VM bastion: **vpn-server** (DMZ, `10.0.0.3`)  
Access protocol to the bastion: **SSH**  

## How to Enable VPN to Homelab
To enable VPN to Homelab we must generate the NordVPN Meshnet.   

1. Inside the **vpn-server** VM (see [VM inventory](./vm-inventory.md)), run the following commands:

   ```shell script
   labuser@vpn-server:~$ sudo apt-get update
   labuser@vpn-server:~$ sh <(curl -sSf https://downloads.nordcdn.com/apps/linux/install.sh)
   labuser@vpn-server:~$ sudo usermod -aG nordvpn $USER
   labuser@vpn-server:~$ sudo reboot
   labuser@vpn-server:~$ nordvpn login --token e9f2ab936332fbd0b0f8a6b35a8a8ef606018d90bae4ace3387fd193ef357791
   Welcome to NordVPN! You can now connect to the VPN by using 'nordvpn connect'.

   NOTE: By default, all users who are members of the 'nordvpn' group have permission to control the NordVPN application.
   To limit access exclusively to the root user, remove all users from the 'nordvpn' group.

   labuser@vpn-server:~$ nordvpn set meshnet on
   Meshnet is set to 'enabled' successfully.

   labuser@vpn-server:~$ nordvpn meshnet peer list
   This device:
   Hostname: fjvi13-everest.nord
   Nickname: -
   IP: 100.72.151.12
   Public Key: cK0z5uF+D8Jl4x5JYfAErPwUeqQbrO1N5LjbcOedP3M=
   OS: linux
   Distribution: Ubuntu

   Local Peers:
   Hostname: fjvi13-himalayas.nord
   Nickname: -
   Status: disconnected
   IP: 100.64.123.125
   Public Key: Oc+pdJbVkne77Sq87MDTTn5IqdCKO3da4bqvrtuZfxg=
   OS: windows
   Distribution: Windows 10 Home 64-bit (10.0, Build 22000)
   Allow Incoming Traffic: enabled
   Allow Routing: disabled
   Allow Local Network Access: disabled
   Allow Sending Files: enabled
   Allows Incoming Traffic: enabled
   Allows Routing: disabled
   Allows Local Network Access: disabled
   Allows Sending Files: enabled
   Accept Fileshare Automatically: disabled

   Hostname: fjvi13-alps.nord
   Nickname: -
   Status: disconnected
   IP: 100.64.61.167
   Public Key: z1N+cYYnuMJtvDLywA9iXUBZmn5ksvqL+g0qndVuZHU=
   OS: macos
   Distribution: 26.1.0
   Allow Incoming Traffic: enabled
   Allow Routing: disabled
   Allow Local Network Access: disabled
   Allow Sending Files: enabled
   Allows Incoming Traffic: enabled
   Allows Routing: disabled
   Allows Local Network Access: disabled
   Allows Sending Files: enabled
   Accept Fileshare Automatically: disabled

   Hostname: fjvi13-andes.nord
   Nickname: -
   Status: connected
   IP: 100.98.96.164
   Public Key: IItKH2vt+R29MVJ0CAz0AuXqbBSEBMHhZNfMxABi3ww=
   OS: macos
   Distribution: 26.4.0
   Allow Incoming Traffic: enabled
   Allow Routing: disabled
   Allow Local Network Access: disabled
   Allow Sending Files: enabled
   Allows Incoming Traffic: enabled
   Allows Routing: disabled
   Allows Local Network Access: disabled
   Allows Sending Files: enabled
   Accept Fileshare Automatically: disabled


   External Peers:
   [no peers]

   labuser@vpn-server:~$ ip -c a
   1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
      link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
      inet 127.0.0.1/8 scope host lo
         valid_lft forever preferred_lft forever
      inet6 ::1/128 scope host noprefixroute 
         valid_lft forever preferred_lft forever
   2: ens18: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group 57841 qlen 1000
      link/ether bc:24:11:ad:be:56 brd ff:ff:ff:ff:ff:ff
      altname enp0s18
      inet 192.168.1.130/24 metric 100 brd 192.168.1.255 scope global dynamic ens18
         valid_lft 76886sec preferred_lft 76886sec
      inet6 fe80::be24:11ff:fead:be56/64 scope link 
         valid_lft forever preferred_lft forever
   3: nordlynx: <POINTOPOINT,UP,LOWER_UP> mtu 1420 qdisc noqueue state UNKNOWN group default qlen 1000
      link/none 
      inet 100.72.151.12/10 scope global nordlynx
         valid_lft forever preferred_lft forever
   ```
   > [!NOTE]
   > Get the token from the NordVPN web portal.

2. Go to the NordVPN app and click in *Link external device* to add a peer to the Meshnet.

## How to remotely access a Kubernetes cluster located in the Homelab
Inside a client machine (peer member of the Meshnet), do the following:

1. Go to the NordVPN app and connect to the Meshnet.  
2. Set Kubeconfig *k3s-tunnel.yaml*.  
3. Run `ssh -L 6443:<k3s-cp_ip>:6443 labuser@<vpn-server_ip>`.  
4. Test any `kubectl` command.  

> [!NOTE]
> `k3s-tunnel.yaml` is a copy of the cluster's default kubeconfig with a single change: the `server` field points to `https://127.0.0.1:6443` instead of `https://<k3s-cp_ip>:6443`. This makes `kubectl` talk to the local end of the SSH tunnel created in step 3, which forwards the traffic to the real K3s API server through the bastion.

## How to remotely access to a service located in the Homelab
Inside a client machine (peer member of the Meshnet), do the following:

**Option A**  
1. Go to the NordVPN app and connect to the Meshnet.  
2. Add the following entry to `/etc/hosts`: `127.0.0.1 <service_name>`.   
3. Run `ssh -L 80:<service_host_ip>:80 labuser@<vpn-server_ip>`.  
4. Try connection: `curl http://<service_name>`.

**Option B**  
1. Go to the NordVPN app and connect to the Meshnet.  
2. Add the following entry to `/etc/hosts`: `<service_host_ip> <service_name>`.  
3. Run `ssh -D 8080 labuser@<vpn-server_ip>`.  
4. Download the *FoxyProxy* Google Chrome extension and add SOCKS4 Proxy (*Hostname: localhotst port:8080*).  
5. Try `http://<service_name>` connection in Google Chrome.
