## How to Build a Template in Proxmox
There are several methods to create a template in Proxmox. In this section, I describe my preferred one.

1- Inside the local disk of the node where we are going to create the Template, click on ISO Images and Download from URL:
  - URL: https://nl.releases.ubuntu.com/releases/24.04.3/ubuntu-24.04.3-live-server-amd64.iso (for example)
  - Click on Query URL and Download.
 
2- Create a new VM:
> [!NOTE]  
> Use these steps as a general guide (they may not be suitable for every situation).  
> Any parameters not mentioned are either arbitrary or will remain at their default values.
  - OS: ISO Image: ubuntu-24.04.3-live-server-amd64.iso 
  - System: Qemu Agent ☑️ | SCSI Controller: VirtIO SCSI
  - Disks: Disk size (GiB): 16 | Discard: ☑️ | IO thread: ☑️ 

> [!WARNING]
> Disk size can be increased later, but not decreased (at least not as easily).

  - CPU: Type: x86-64-v2-AES
  - Memory: Memory (MiB): 1024
  - Network: Bridge: vmbr0 | VLAN Tag: 10 (Homelab Network) 
  - Click Finish

3- Go to the Hardware tab of the new VM: 
  - Add the *Cloud Init Drive*: Storage: local-lvm

4- Go to the Cloud-Init tab and configure it as you wish (Do not forget to click on Regenerage Image when you are done).

5- Start the VM and go through the Ubuntu installation process (just accept the default options if unsure).

6- Run the following commands:
  - sudo apt dist-upgrade
  - sudo apt update

7- Empty out machine ID
  - sudo cloud-init clean --machine-id

8- Run the following commands (just in case):
  - sudo apt clean
  - sudo apt autoremove

9- Poweroff the VM

10- Convert the VM to a Template

11- Remove the attachment to the ISO:
  - Click on Hardware tab - Remove CD/DVD Drive

## How to Create a New VM from a Template in Proxmox
1- Clone the Template:
  - Mode: Full Clone
  - Target Source: local-lvm

2- Now, you can do SSH to the VM. Finally, install and start QEMU Guest Agent (ensure that it is enabled in the Options tab):
  - sudo apt install qemu-guest-agent
  - sudo systemctl start qemu-guest-agent

### Reference
- [Proxmox VE - How to build an Ubuntu 22.04 Template (Updated Method)](https://www.youtube.com/watch?v=MJgIm03Jxdo)