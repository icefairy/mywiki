# 修复Centos7的SELINUX错误

> 问题原因：修改selinux配置时使用了错误的配置没有检查到直接重启，就会导致服务器无法启动（停留在启动进度条加载的地方）
>
> 解决办法：强制重启的同时，通过云平台的vnc功能，在grub启动菜单处按E进入编辑，在linux16开头那行的最末尾 加入` 空格selinux=0` 按Ctrl+x执行启动，进入系统后修复错误的配置文件，再`reboot`重启测试，能正常进入系统，此时修复完成。

正确的selinux设置例子如下:

```bash
cat /etc/selinux/config 
#开启
SELINUX=enforcing
SELINUXTYPE=targeted 
#关闭
SELINUX=disabled
SELINUXTYPE=targeted 
```

