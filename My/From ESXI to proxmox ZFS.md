---
tags:
  - proxmox
  - esxi
  - zfs
  - virtualization
  - my-config
date: 2026-01-13
---

# Migration from ESXI to Proxmox ZFS

Монтируем sshfs сервера esxi на proxmox
 ```
 sshfs  root@192.168.172.2:/vmfs/volumes /mnt/esxi/
 ```

Конвертируем и кладем временно в /var/tmp
```
qemu-img convert -p -O raw /mnt/esxi/for_common_VMs/ubuntu_ans/ubuntu_ans-flat.vmdk /var/tmp/vm-101-disk-0.raw
```

Создаем виртуалку без диска
```
qm create 101 --name "ansible" --memory 4096 --cores 4 --net0 virtio,bridge=vmbr0
```

Импортируем сконвертированный диск и добавляем к виртуалке
```
 qm importdisk 101 /var/tmp/vm-101-disk-0.raw ZFS-Storage
```
 
Потом заходим в gui, открываем hardware ВМ hardware, там будет этот диск и его можно добавить. 
Profit.

У меня ВМ запустилась, но пришлось в ней перенастроить сеть, так как в убунту в netplan было другое имя сетевого адаптера