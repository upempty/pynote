- allows kernel code to export information to user processes via an in-memory filesystem.  
  e.g. device tree info show to user space via sysfs.
```
[root@iZbp1evissossq564e432eZ ~]# tree /sys/class/net
/sys/class/net
├── eth0 -> ../../devices/pci0000:00/0000:00:05.0/virtio2/net/eth0
└── lo -> ../../devices/virtual/net/lo

```
