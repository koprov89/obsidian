---
tags:
  - checkpoint
  - proxmox
  - virtualization
  - my-config
date: 2026-01-13
---

# Checkpoint Proxmox Cluster Configuration

node1
	pvecm create *name*
node2
	pvecm add *node1 ip*
scp ~/Downloads/Check_Point_R81.20_T634.iso proxmox_torus_2:/var/lib/vz/template/iso
##### /etc/pve/corosync.conf
```
logging {
  debug: off
  to_syslog: yes
}

nodelist {
  node {
    name: checkpoint-01
    nodeid: 1
    quorum_votes: 1
    ring0_addr: 217.182.94.93
  }
  node {
    name: checkpoint-02
    nodeid: 2
    quorum_votes: 1
    ring0_addr: 217.182.94.136
  }
}

quorum {
  provider: corosync_votequorum
  two_node: 1
}

totem {
  cluster_name: CP-torus
  config_version: 2
  interface {
    linknumber: 0
  }
  ip_version: ipv4-6
  link_mode: passive
  secauth: on
  version: 2
}
```

##### /etc/pve/firewall/cluster.fw
```
[OPTIONS]

enable: 1

[IPSET cluster]

217.182.94.136
217.182.94.93

[IPSET whitelist]

212.98.168.98
10.0.0.0/8
172.16.0.0/12
192.168.0.0/16

[RULES]

IN ACCEPT -source +dc/cluster -log nolog
IN ACCEPT -source +dc/whitelist -log nolog
```
##### /etc/network/interfaces
```
auto lo
iface lo inet loopback

iface enp8s0f0np0 inet manual

iface enp8s0f1np1 inet manual

auto vmbr0
iface vmbr0 inet static
        address 217.182.94.136/32
        gateway 100.64.0.1
        bridge-ports enp8s0f0np0
        bridge-stp off
        bridge-fd 0
        hwaddress 34:5a:60:00:64:3c

auto private
iface private inet manual
        bridge-ports enp8s0f1np1
        bridge-stp off
        bridge-fd 0
        bridge-vlan-aware yes
        bridge-vids 2-4094

auto private.666
iface private.666 inet static
        address 172.25.0.2/24
        up ip route add 172.16.0.0/12 via 172.25.0.250
        up ip route add 192.168.0.0/16 via 172.25.0.250
        up ip route add 10.0.0.0/8 via 172.25.0.250

```

##### /etc/pve/qemu-server/100.conf
```
boot: order=scsi0;ide2
cores: 6
cpu: x86-64-v2-AES
ide2: local:iso/Check_Point_R81.20_T634.iso,media=cdrom,size=4267296K
memory: 16384
meta: creation-qemu=9.2.0,ctime=1749746778
name: cp-torus-01
net0: virtio=BC:24:11:DD:EE:98,bridge=private
net1: virtio=BC:24:11:9C:71:C1,bridge=private
numa: 0
onboot: 1
ostype: l26
scsi0: local:100/vm-100-disk-0.qcow2,iothread=1,size=200G
scsihw: virtio-scsi-single
smbios1: uuid=80f8323c-134b-4dc1-b226-ce92cf1e25e6
sockets: 1
vmgenid: d9a1eabf-64a8-4f50-8870-51c11933bb53
```