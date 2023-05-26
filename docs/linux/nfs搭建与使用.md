# nfs搭建与使用

+ 安装nfs-utils
+ 修改/etc/exports 增加一行或者多行 `/data/nfsroot 172.16.11.24(rw,sync,no_root_squash)`
+ 执行`exportfs -r`刷新
+ 重启nfs服务`systemctl restart nfs`
+ 重启rpcbind服务`systemctl restart rpcbind` 
+ 确保nfs和rpcbind服务处于开机自启状态`systemctl enable nfs` 和`systemctl enable rpcbind`
+ 客户端只需要依赖rpcbind服务
+ 客户端加载，例如放到/etc/fstab中`172.16.11.6:/data/nfsroot    /ycc    nfs rw,exec,noatime,nofail   0   0` 然后执行mount -a 进行加载之后用`df -h`查看效果即可
+ 客户端可以通过执行`showmount -e 服务器端ip`来查看能否访问到服务器上的资源
+ 注意如果提供网络相关错误的话 请在服务器端放行nfs服务端口和rpcbind的端口 或者直接清空防火墙规则