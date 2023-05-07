# 

* 查看当前内核版本

```bash
uname -r
#3.10.0-514.el7.x86_64
uname -a
#Linux k8s-master 3.10.0-514.el7.x86_64 #1 SMP Tue Nov 22 16:42:41 UTC 2016 x86_64 x86_64 x86_64 GNU/Linux
cat /etc/redhat−release
#CentOSLinuxrelease7.3.1611(Core)
```

* 更新yum源仓库

```bash
yum -y update
#启用 ELRepo 仓库 ELRepo 仓库是基于社区的用于企业级 Linux 仓库，提供对 RedHat Enterprise (RHEL) 和 其他基于 RHEL的 Linux 发行版（CentOS、Scientific、Fedora 等）的支持。 ELRepo 聚焦于和硬件相关的软件包，包括文件系统驱动、显卡驱动、网络驱动、声卡驱动和摄像头驱动等。
#导入ELRepo仓库的公共密钥
rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
rpm -Uvh http://www.elrepo.org/elrepo-release-7.0-3.el7.elrepo.noarch.rpm
```

* 查看可用的系统内核包 可以看到4.4和4.18两个版本

```bash
yum --disablerepo="*" --enablerepo="elrepo-kernel" search kernel
list available Loaded plugins: fastestmirror Loading mirror speeds from cached hostfile elrepo-kernel: mirrors.tuna.tsinghua.edu.cn elrepo-kernel | 2.9 kB 00:00:00
elrepo-kernel/primary_db | 1.8 MB 00:00:03
Available Packages kernel-lt.x86_64 4.4.155-1.el7.elrepo elrepo-kernel kernel-lt-devel.x86_64 4.4.155-1.el7.elrepo elrepo-kernel kernel-lt-doc.noarch 4.4.155-1.el7.elrepo elrepo-kernel kernel-lt-headers.x86_64 4.4.155-1.el7.elrepo elrepo-kernel kernel-lt-tools.x86_64 4.4.155-1.el7.elrepo elrepo-kernel kernel-lt-tools-libs.x86_64 4.4.155-1.el7.elrepo elrepo-kernel kernel-lt-tools-libs-devel.x86_64 4.4.155-1.el7.elrepo elrepo-kernel kernel-ml.x86_64 4.18.7-1.el7.elrepo elrepo-kernel kernel-ml-devel.x86_64 4.18.7-1.el7.elrepo elrepo-kernel kernel-ml-doc.noarch 4.18.7-1.el7.elrepo elrepo-kernel kernel-ml-headers.x86_64 4.18.7-1.el7.elrepo elrepo-kernel kernel-ml-tools.x86_64 4.18.7-1.el7.elrepo elrepo-kernel kernel-ml-tools-libs.x86_64 4.18.7-1.el7.elrepo elrepo-kernel kernel-ml-tools-libs-devel.x86_64 4.18.7-1.el7.elrepo elrepo-kernel perf.x86_64 4.18.7-1.el7.elrepo elrepo-kernel python-perf.x86_64 4.18.7-1.el7.elrepo elrepo-
```

* 安装最新版本内核`yum --enablerepo=elrepo-kernel install kernel-lt kernel-lt-devel -y` --enablerepo 选项开启 CentOS 系统上的指定仓库。默认开启的是 elrepo，这里用 elrepo-kernel 替换。
* 设置 grub2 内核安装好后，需要设置为默认启动选项并重启后才会生效

查看系统上的所有可用内核：
修改grub配置文件
vi /etc/default/grub
GRUB_DEFAULT=0   
grub2-mkconfig -o /boot/grub2/grub.cfg  //update
shutdown -r now
如果系统是UEFI启动，修改Grub2配置文件，重启不生效，需要如下方法修改grub配置文件
vim /boot/efi/EFI/centos/grubenv
修改如下
saved_entry=CentOS Linux (5.4.240-1.el7.elrepo.x86_64) 7 (Core)
刷新重启

grub2-mkconfig -o /boot/efi/EFI/centos/grub.cfg  
shutdown -r now


设置新的内核为grub2的默认版本 服务器上存在4 个内核，我们要使用 4.18 这个版本，可以通过 grub2-set-default 0 命令或编辑 /etc/default/grub 文件来设置

通过 `grub2-set-default 0 `命令设置 其中 0 是上面查询出来的可用内核

* 重启`reboot`
* 验证 uname -r 显示4.18.7-1.el7.elrepo.x86_64
* 删除旧内核（可选） 查看系统中全部的内核：

```bash
rpm −qa|grep kernel
kernel−3.10.0−514.el7.x86_64
kernel−ml−4.18.7−1.el7.elrepo.x86_64
kernel−tools−libs−3.10.0−862.11.6.el7.x86_64
kernel−tools−3.10.0−862.11.6.el7.x86_64
kernel−3.10.0−862.11.6.el7.x86_64
```

方法1、yum remove删除旧内核的RPM包 yum remove kernel-3.10.0-514.el7.x86_64
kernel-tools-libs-3.10.0-862.11.6.el7.x86_64
kernel-tools-3.10.0-862.11.6.el7.x86_64
kernel-3.10.0-862.11.6.el7.x86_64