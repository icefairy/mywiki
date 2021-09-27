K3S作为一种轻量化的k8s方案在简单的环境中非常便于部署

`curl -sfL http://rancher-mirror.cnrancher.com/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn sh -`

就这么一行命令就可以安装成功，然后就可以愉快的使用`kubectl` 之类的命令啦。