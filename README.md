#### Migrate Logical voulmes and volumes groups from one disk to another disk in the same machine without losing data or rebooting the machine.  I'm here migrating the LVM from partition /dev/xvdb1 to /dev/xvdc1. 

#### Creating a new partition in new disk /dev/xvdc

```
[root@ip-172-31-16-207 lvm]# lsblk
NAME               MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
xvda               202:0    0   8G  0 disk 
└─xvda1            202:1    0   8G  0 part /
xvdb               202:16   0   8G  0 disk 
└─xvdb1            202:17   0   4G  0 part 
  └─vg00-lv_backup 253:0    0   3G  0 lvm  /backup
xvdc               202:32   0   8G  0 disk 

[root@ip-172-31-16-207 lvm]# fdisk /dev/xvdc

Welcome to fdisk (util-linux 2.30.2).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

Device does not contain a recognized partition table.
Created a new DOS disklabel with disk identifier 0x64a3ae29.

Command (m for help): n
Partition type
   p   primary (0 primary, 0 extended, 4 free)
   e   extended (container for logical partitions)
Select (default p): p
Partition number (1-4, default 1): 
First sector (2048-16777215, default 2048): 
Last sector, +sectors or +size{K,M,G,T,P} (2048-16777215, default 16777215): +4G

Created a new partition 1 of type 'Linux' and of size 4 GiB.

Command (m for help): w
The partition table has been altered.
Calling ioctl() to re-read partition table.
Syncing disks.

[root@ip-172-31-16-207 lvm]# lsblk
NAME               MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
xvda               202:0    0   8G  0 disk 
└─xvda1            202:1    0   8G  0 part /
xvdb               202:16   0   8G  0 disk 
└─xvdb1            202:17   0   4G  0 part 
  └─vg00-lv_backup 253:0    0   3G  0 lvm  /backup
xvdc               202:32   0   8G  0 disk 
└─xvdc1            202:33   0   4G  0 part 
```


#### To add additional physical volumes to an existing volume group, use the vgextend command. The vgextend command increases a volume group's capacity by adding one or more free physical volumes.


```
[root@ip-172-31-16-207 lvm]# vgextend vg00 /dev/xvdc1
  Physical volume "/dev/xvdc1" successfully created.
    Volume group "vg00" successfully extended
    
[root@ip-172-31-16-207 lvm]# vgs
  VG   #PV #LV #SN Attr   VSize VFree
  vg00   2   1   0 wz--n- 7.99g 4.99g
  
[root@ip-172-31-16-207 lvm]# vgdisplay
  --- Volume group ---
  VG Name               vg00
  System ID             
  Format                lvm2
  Metadata Areas        2
  Metadata Sequence No  5
  VG Access             read/write
  VG Status             resizable
  MAX LV                0
  Cur LV                1
  Open LV               1
  Max PV                0
  Cur PV                2
  Act PV                2
  VG Size               7.99 GiB
  PE Size               4.00 MiB
  Total PE              2046
  Alloc PE / Size       768 / 3.00 GiB
  Free  PE / Size       1278 / 4.99 GiB
  VG UUID               PLB0Vy-RTTz-aWsg-Tc7t-fdLB-Oy0j-FUA9zk
  ```
  
  #### Moving the LVM named as lv_backup from partition /dev/xvdb1  to /dev/xvdc1
  
  ```
  
  
  [root@ip-172-31-16-207 lvm]# lsblk
NAME               MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
xvda               202:0    0   8G  0 disk 
└─xvda1            202:1    0   8G  0 part /
xvdb               202:16   0   8G  0 disk 
└─xvdb1            202:17   0   4G  0 part 
  └─vg00-lv_backup 253:0    0   3G  0 lvm  /backup
xvdc               202:32   0   8G  0 disk 
└─xvdc1            202:33   0   4G  0 part 


[root@ip-172-31-16-207 lvm]# pvmove -n lv_backup /dev/xvdb1  /dev/xvdc1
  /dev/xvdb1: Moved: 0.91%
  /dev/xvdb1: Moved: 62.50%
  /dev/xvdb1: Moved: 100.00%

  
  [root@ip-172-31-16-207 lvm]# lsblk
NAME               MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
xvda               202:0    0   8G  0 disk 
└─xvda1            202:1    0   8G  0 part /
xvdb               202:16   0   8G  0 disk 
└─xvdb1            202:17   0   4G  0 part 
xvdc               202:32   0   8G  0 disk 
└─xvdc1            202:33   0   4G  0 part 
  └─vg00-lv_backup 253:0    0   3G  0 lvm  /backup
```


#### We can still see the volume group vg00 in partition /dev/sdb1. 

```
  
[root@ip-172-31-16-207 backup]# pvs
  PV         VG   Fmt  Attr PSize  PFree   
  /dev/sdb1  vg00 lvm2 a--  <4.00g   <4.00g
  /dev/sdc1  vg00 lvm2 a--  <4.00g 1020.00m

```

#### To remove the /dev/sdb1 from volume group vg00

```
   
  
  [root@ip-172-31-16-207 backup]# vgreduce vg00 /dev/sdb1
  Removed "/dev/sdb1" from volume group "vg00"
  
[root@ip-172-31-16-207 backup]# pvs
  PV         VG   Fmt  Attr PSize  PFree   
  /dev/sdb1       lvm2 ---   4.00g    4.00g
  /dev/sdc1  vg00 lvm2 a--  <4.00g 1020.00m
  
```

  
  #### Note - We should not adpot this method if the root partition / root file systems data is present in the partition. Never use vgreduce command after the migration of root partition / root file systems.
  
  
  
  ```

  [root@ip-172-31-16-207 backup]# vgs -o+devices
  VG   #PV #LV #SN Attr   VSize  VFree    Devices     
  vg00   1   1   0 wz--n- <4.00g 1020.00m /dev/sdc1(0)

  ```
  
  
  



