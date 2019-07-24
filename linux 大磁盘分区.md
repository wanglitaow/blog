@[TOC]

# 已挂载磁盘分区
分区需先把磁盘unmount /dev/mapper/centos_ht05-home 
Q1:Error: /dev/dm-2: unrecognised disk label
mklabel gpt  

# 开始分区

``` linux
parted /dev/dm-2    查看/dev/mapper/centos_ht05-home是l文件属性，链接到dm-2,分区dm-2磁盘
(parted) mkpart                                                           
Partition name?  []? data1                                                
File system type?  [ext2]? ext4                                           
Start? 0                                                                  
End? 3584GB
(parted) mkpart
Partition name?  []? data2                                                
File system type?  [ext2]? ext4                                           
Start? 3584GB
End? 7168GB
(parted) mkpart
Partition name?  []? data3
File system type?  [ext2]? ext4
Start? 7168GB                                                           
End? -1 
(parted) p                                                                
Model: Linux device-mapper (linear) (dm)
Disk /dev/dm-2: 11.9TB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags: 

Number  Start   End     Size    File system  Name   Flags
 1      17.4kB  3584GB  3584GB               data1
 2      3584GB  7168GB  3584GB               data2
 3      7168GB  11.9TB  4745GB               data3
```

若分区错误：rm 1 
# 挂载目录

``` linux
mkdir /data1 /data2 /data3
ll -t /dev/dm-*  		发现 dm-2 dm-17 dm-28 dm-27最新
mkfs -t ext4 /dev/dm-2
mkfs -t ext4 /dev/dm-17
mkfs -t ext4 /dev/dm-27
mkfs -t ext4 /dev/dm-28
fdisk -l 查看/dev/mapper/centos_ht05-home3 /dev/mapper/centos_ht05-home2 /dev/mapper/centos_ht05-home1生成
mount /dev/mapper/centos_ht05-home1 /data1
mount /dev/mapper/centos_ht05-home2 /data2
mount /dev/mapper/centos_ht05-home3 /data3
```

详情见：
https://github.com/OneJane/blog