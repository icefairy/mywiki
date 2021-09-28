## 在线安装

K3S作为一种轻量化的k8s方案在简单的环境中非常便于部署

`curl -sfL http://rancher-mirror.cnrancher.com/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn sh -`

就这么一行命令就可以安装成功，然后就可以愉快的使用`kubectl` 之类的命令啦。


## 离线安装

离线环境的升级可以通过以下步骤完成：

* 从[K3s GitHub Release](https://github.com/rancher/k3s/releases)页面下载要升级到的 K3s 版本。将 tar 文件放在每个节点的`/var/lib/rancher/k3s/agent/images/`目录下。删除旧的 tar 文件。

* 复制并替换每个节点上`/usr/local/bin`中的旧 K3s 二进制文件。复制[https://get.k3s.io](https://get.k3s.io/) 的安装脚本（因为它可能在上次发布后发生了变化）。再次运行脚本。

* 重启 K3s 服务。