# LVM分区扩容

对于已有LVM分区模式的机器，新增一块硬盘需要将其扩容到已有的`/data`分区，大致操作步骤如下：

+ `vgdisplay`查看现有的分区组名称“VG Name”
+ `fdisk -l `找到新增的磁盘例如`/dev/vde`
+ `pvcreate /dev/vde`创建卷
+ `pvdisplay`查看所有的卷
+ `df -h`确定/data的卷名，例如`/dev/mapper/deb10--vg-data`
+ `vgextend deb10-vg /dev/vde`将vde加到逻辑卷中
+ `lvextend -L +100G /dev/mapper/deb10--vg-data` 给`/data`扩容100GB容量
+ `resize2fs /dev/mapper/deb10--vg-data`刷新磁盘容量（如果是xfs格式的话使用`xfs_growfs /dev/mapper/deb10--vg-data` )
+ `df -h`查看新的磁盘分区容量信息即可看到变动


## 全新创建lvm

```
#先找到要创建物理卷的磁盘比如/dev/vdc
fdisk -l
#创建物理卷
pvcreate /dev/vdc
#创建物理卷组
vgcreate vgname /dev/vdc
#查看卷组列表
vgs
#创建卷
lvcreate -n wtdata1 -L 100 vgname
#扩容量
lvextend -L +499G /dev/vgname/wtdata1
#格式化
mkfs.ext4 /dev/vgname/wtdata1
#创建挂载点
mkdir /data
#挂载
mount /dev/vgname/wtdata1 /data
```