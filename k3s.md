## Steps to add a new node to the K3s cluster
1- Create a new VM (see blablabla)  
2- Install LVM: 
  - sudo apt install -y lvm2  
  - sudo systemctl status lvm2-monitor.service

3- Create the dedicated PV and VG:
  - sudo pvcreate /dev/sda3
  - sudo vgcreate k8s-vg /dev/sdb
